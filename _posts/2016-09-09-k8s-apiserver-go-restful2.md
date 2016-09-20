---
layout:     post
title:      "kubernetes源码解析"
subtitle:   "apiserver路由构建解析(2)"
date:       2016-09-20 08:00:00
author:     "Daniel"
header-img: "img/apiserver.png"
header-mask: 0.3
catalog:    true
tags:
    - golang
    - kubernetes
    - kube-apiserver
---

## kubernetes源码解析---- apiserver路由构建解析(2)

上文主要对go-restful这个包进行了简单的介绍，下面我们通过阅读代码来理解apiserver路由的详细构建过程。

(kubernetes代码版本：1.3.6     Commit id：ed3a29bd6aeb)



从启动位置main函数开始（kubernetes\cmd\kube-apiserver\apiserver.go）：

```go
func main() {
	rand.Seed(time.Now().UTC().UnixNano())

  	// New APIServer
	s := options.NewAPIServer()
	s.AddFlags(pflag.CommandLine)

  	// 解析命令行参数
	flag.InitFlags()
	logs.InitLogs()
	defer logs.FlushLogs()

	verflag.PrintAndExitIfRequested()

  	// 启动APIServer
	if err := app.Run(s); err != nil {
		fmt.Fprintf(os.Stderr, "%v\n", err)
		os.Exit(1)
	}
}
```

main函数里做的事情比较简单，主要是生成默认的ApiServer运行参数，解析命令行，设置log，然后调用 app.Run()方法启动服务。继续跟进这个Run()方法：

```go
func Run(s *options.APIServer) error {
	genericvalidation.VerifyEtcdServersList(s.ServerRunOptions)
	genericapiserver.DefaultAndValidateRunOptions(s.ServerRunOptions)
  
  	// master config组装
  	...
  
  	// New master
  	m, err := master.New(config)
	if err != nil {
		return err
	}

	sharedInformers.Start(wait.NeverStop)
  	// 启动master
	m.Run(s.ServerRunOptions)
	return nil
```

Run()方法代码较长，这里只贴出了其中一部分。代码上，首先组装master config，其中包括ssh tunneler配置，storageFactory配置，authenticator配置和authorizer配置等，然后根据config创建并启动master。

这里有两个与*路由构建*相关的方法：master.New(config)和m.Run(s.ServerRunOptions)。我们先来跟进New()方法：

```go
// New returns a new instance of Master from the given config.
// Certain config fields will be set to a default value if unset.
// Certain config fields must be specified, including:
//   KubeletClient
func New(c *Config) (*Master, error) {
	if c.KubeletClient == nil {
		return nil, fmt.Errorf("Master.New() called with config.KubeletClient == nil")
	}

  	// 创建并初始化GenericAPIServer
	s, err := genericapiserver.New(c.Config)
	if err != nil {
		return nil, err
	}

  	// 构造Master
	m := &Master{
		GenericAPIServer:        s,
		enableCoreControllers:   c.EnableCoreControllers,
		deleteCollectionWorkers: c.DeleteCollectionWorkers,
		tunneler:                c.Tunneler,

		disableThirdPartyControllerForTesting: c.disableThirdPartyControllerForTesting,
	}

	// Add some hardcoded storage for now.  Append to the map.
	if c.RESTStorageProviders == nil {
		c.RESTStorageProviders = map[string]RESTStorageProvider{}
	}
	c.RESTStorageProviders[appsapi.GroupName] = AppsRESTStorageProvider{}
	c.RESTStorageProviders[autoscaling.GroupName] = AutoscalingRESTStorageProvider{}
	c.RESTStorageProviders[batch.GroupName] = BatchRESTStorageProvider{}
	c.RESTStorageProviders[certificates.GroupName] = CertificatesRESTStorageProvider{}
	c.RESTStorageProviders[extensions.GroupName] = ExtensionsRESTStorageProvider{
		ResourceInterface:                     m,
		DisableThirdPartyControllerForTesting: m.disableThirdPartyControllerForTesting,
	}
	c.RESTStorageProviders[policy.GroupName] = PolicyRESTStorageProvider{}
	c.RESTStorageProviders[rbac.GroupName] = RBACRESTStorageProvider{AuthorizerRBACSuperUser: c.AuthorizerRBACSuperUser}
	c.RESTStorageProviders[authenticationv1beta1.GroupName] = AuthenticationRESTStorageProvider{Authenticator: c.Authenticator}
	c.RESTStorageProviders[authorization.GroupName] = AuthorizationRESTStorageProvider{Authorizer: c.Authorizer}
  
  	// 安装APIs
	m.InstallAPIs(c)

	// TODO: Attempt clean shutdown?
	if m.enableCoreControllers {
		m.NewBootstrapController(c.EndpointReconcilerConfig).Start()
	}

	return m, nil
}
```

首先，调用genericapiserver.New(c.Config)，创建一个genericapiserver对象。然后往master config中添加一些“hardcoded storage”，再调用m.InstallAPIs(c)安装APIs。

我们先跟进这个genericapiserver.New()方法：

```go
// New returns a new instance of GenericAPIServer from the given config.
// Certain config fields will be set to a default value if unset,
// including:
//   ServiceClusterIPRange
//   ServiceNodePortRange
//   MasterCount
//   ReadWritePort
//   PublicAddress
// Public fields:
//   Handler -- The returned GenericAPIServer has a field TopHandler which is an
//   http.Handler which handles all the endpoints provided by the GenericAPIServer,
//   including the API, the UI, and miscellaneous debugging endpoints.  All
//   these are subject to authorization and authentication.
//   InsecureHandler -- an http.Handler which handles all the same
//   endpoints as Handler, but no authorization and authentication is done.
// Public methods:
//   HandleWithAuth -- Allows caller to add an http.Handler for an endpoint
//   that uses the same authentication and authorization (if any is configured)
//   as the GenericAPIServer's built-in endpoints.
//   If the caller wants to add additional endpoints not using the GenericAPIServer's
//   auth, then the caller should create a handler for those endpoints, which delegates the
//   any unhandled paths to "Handler".
func New(c *Config) (*GenericAPIServer, error) {
	if c.Serializer == nil {
		return nil, fmt.Errorf("Genericapiserver.New() called with config.Serializer == nil")
	}
	setDefaults(c)

	s := &GenericAPIServer{
		ServiceClusterIPRange: c.ServiceClusterIPRange,
		ServiceNodePortRange:  c.ServiceNodePortRange,
		RootWebService:        new(restful.WebService),
		enableLogsSupport:     c.EnableLogsSupport,
		enableUISupport:       c.EnableUISupport,
		enableSwaggerSupport:  c.EnableSwaggerSupport,
		enableSwaggerUI:       c.EnableSwaggerUI,
		enableProfiling:       c.EnableProfiling,
		enableWatchCache:      c.EnableWatchCache,
		APIPrefix:             c.APIPrefix,
		APIGroupPrefix:        c.APIGroupPrefix,
		corsAllowedOriginList: c.CorsAllowedOriginList,
		authenticator:         c.Authenticator,
		authorizer:            c.Authorizer,
		AdmissionControl:      c.AdmissionControl,
		RequestContextMapper:  c.RequestContextMapper,
		Serializer:            c.Serializer,

		cacheTimeout:      c.CacheTimeout,
		MinRequestTimeout: time.Duration(c.MinRequestTimeout) * time.Second,

		MasterCount:          c.MasterCount,
		ExternalAddress:      c.ExternalHost,
		ClusterIP:            c.PublicAddress,
		PublicReadWritePort:  c.ReadWritePort,
		ServiceReadWriteIP:   c.ServiceReadWriteIP,
		ServiceReadWritePort: c.ServiceReadWritePort,
		ExtraServicePorts:    c.ExtraServicePorts,
		ExtraEndpointPorts:   c.ExtraEndpointPorts,

		KubernetesServiceNodePort: c.KubernetesServiceNodePort,
		apiGroupsForDiscovery:     map[string]unversioned.APIGroup{},

		enableOpenAPISupport:   c.EnableOpenAPISupport,
		openAPIInfo:            c.OpenAPIInfo,
		openAPIDefaultResponse: c.OpenAPIDefaultResponse,
	}

  	// 初始化GenericAPIServer成员变量HandlerContainer与mux
	if c.RestfulContainer != nil {
		s.mux = c.RestfulContainer.ServeMux
		s.HandlerContainer = c.RestfulContainer
	} else {
		mux := http.NewServeMux()
		s.mux = mux
		s.HandlerContainer = NewHandlerContainer(mux, c.Serializer)
	}
	// Use CurlyRouter to be able to use regular expressions in paths. Regular expressions are required in paths for example for proxy (where the path is proxy/{kind}/{name}/{*})
	s.HandlerContainer.Router(restful.CurlyRouter{})
	s.MuxHelper = &apiserver.MuxHelper{Mux: s.mux, RegisteredPaths: []string{}}

	s.init(c)

	return s, nil
}
```

New()方法创建一个GenericAPIServer对象并返回，这里初始化了GenericAPIServer结构用于构建Restful路由的两个成员变量：

1. mux ： net/http包中原生的路由构造器。

     > ServeMux is an HTTP request multiplexer.
     > It matches the URL of each incoming request against a list of registered
     > patterns and calls the handler for the pattern that most closely matches the URL.

2. HandlerContainer： 既是go-restful包中的Container类型对象(详细解释参见此系列文章1)。设置路由选择器为**CurlyRouter**。后面创建的Webservice都要加入到这个Container中。


退出这个New()方法，继续跟进Master结构的m.InstallAPIs(c)方法：

```go
func (m *Master) InstallAPIs(c *Config) {
	apiGroupsInfo := []genericapiserver.APIGroupInfo{}

	// Install v1 unless disabled.
	if c.APIResourceConfigSource.AnyResourcesForVersionEnabled(apiv1.SchemeGroupVersion) {
		// Install v1 API.
		m.initV1ResourcesStorage(c)
		apiGroupInfo := genericapiserver.APIGroupInfo{
			GroupMeta: *registered.GroupOrDie(api.GroupName),
			VersionedResourcesStorageMap: map[string]map[string]rest.Storage{
				"v1": m.v1ResourcesStorage,
			},
			IsLegacyGroup:               true,
			Scheme:                      api.Scheme,
			ParameterCodec:              api.ParameterCodec,
			NegotiatedSerializer:        api.Codecs,
			SubresourceGroupVersionKind: map[string]unversioned.GroupVersionKind{},
		}
		if autoscalingGroupVersion := (unversioned.GroupVersion{Group: "autoscaling", Version: "v1"}); registered.IsEnabledVersion(autoscalingGroupVersion) {
			apiGroupInfo.SubresourceGroupVersionKind["replicationcontrollers/scale"] = autoscalingGroupVersion.WithKind("Scale")
		}
		if policyGroupVersion := (unversioned.GroupVersion{Group: "policy", Version: "v1alpha1"}); registered.IsEnabledVersion(policyGroupVersion) {
			apiGroupInfo.SubresourceGroupVersionKind["pods/eviction"] = policyGroupVersion.WithKind("Eviction")
		}
		apiGroupsInfo = append(apiGroupsInfo, apiGroupInfo)
	}

	...

  	// 安装APIs
	if err := m.InstallAPIGroups(apiGroupsInfo); err != nil {
		glog.Fatalf("Error in registering group versions: %v", err)
	}
}
```

InstallAPIs()方法进行实际的API安装工作，这里贴出部分代码。主要关注m.initV1ResourcesStorage(c)和m.InstallAPIGroups(apiGroupsInfo)这两个方法。

m.initV1ResourcesStorage(c)初始化各种资源后端的存储配置，并存放到Master的成员变量v1ResourcesStorage中。这些后端存储其实也就是对应的Restful路由，只是这些路由还没有实装。

```go
func (m *Master) initV1ResourcesStorage(c *Config) {
	restOptions := func(resource string) generic.RESTOptions {
		return m.GetRESTOptionsOrDie(c, api.Resource(resource))
	}

  	// 生成资源对应的后端存储
	podTemplateStorage := podtemplateetcd.NewREST(restOptions("podTemplates"))

	eventStorage := eventetcd.NewREST(restOptions("events"), uint64(c.EventTTL.Seconds()))
	limitRangeStorage := limitrangeetcd.NewREST(restOptions("limitRanges"))

	resourceQuotaStorage, resourceQuotaStatusStorage := resourcequotaetcd.NewREST(restOptions("resourceQuotas"))
	secretStorage := secretetcd.NewREST(restOptions("secrets"))
	serviceAccountStorage := serviceaccountetcd.NewREST(restOptions("serviceAccounts"))
	persistentVolumeStorage, persistentVolumeStatusStorage := pvetcd.NewREST(restOptions("persistentVolumes"))
	persistentVolumeClaimStorage, persistentVolumeClaimStatusStorage := pvcetcd.NewREST(restOptions("persistentVolumeClaims"))
	configMapStorage := configmapetcd.NewREST(restOptions("configMaps"))

	namespaceStorage, namespaceStatusStorage, namespaceFinalizeStorage := namespaceetcd.NewREST(restOptions("namespaces"))
	m.namespaceRegistry = namespace.NewRegistry(namespaceStorage)

	endpointsStorage := endpointsetcd.NewREST(restOptions("endpoints"))
	m.endpointRegistry = endpoint.NewRegistry(endpointsStorage)

	nodeStorage := nodeetcd.NewStorage(restOptions("nodes"), c.KubeletClient, m.ProxyTransport)
	m.nodeRegistry = node.NewRegistry(nodeStorage.Node)

	podStorage := podetcd.NewStorage(
		restOptions("pods"),
		kubeletclient.ConnectionInfoGetter(nodeStorage.Node),
		m.ProxyTransport,
	)

	serviceRESTStorage, serviceStatusStorage := serviceetcd.NewREST(restOptions("services"))
	m.serviceRegistry = service.NewRegistry(serviceRESTStorage)

	var serviceClusterIPRegistry rangeallocation.RangeRegistry
	serviceClusterIPRange := m.ServiceClusterIPRange
	if serviceClusterIPRange == nil {
		glog.Fatalf("service clusterIPRange is nil")
		return
	}

	serviceStorageConfig, err := c.StorageFactory.NewConfig(api.Resource("services"))
	if err != nil {
		glog.Fatal(err.Error())
	}

	serviceClusterIPAllocator := ipallocator.NewAllocatorCIDRRange(serviceClusterIPRange, func(max int, rangeSpec string) allocator.Interface {
		mem := allocator.NewAllocationMap(max, rangeSpec)
		// TODO etcdallocator package to return a storage interface via the storageFactory
		etcd := etcdallocator.NewEtcd(mem, "/ranges/serviceips", api.Resource("serviceipallocations"), serviceStorageConfig)
		serviceClusterIPRegistry = etcd
		return etcd
	})
	m.serviceClusterIPAllocator = serviceClusterIPRegistry

	var serviceNodePortRegistry rangeallocation.RangeRegistry
	serviceNodePortAllocator := portallocator.NewPortAllocatorCustom(m.ServiceNodePortRange, func(max int, rangeSpec string) allocator.Interface {
		mem := allocator.NewAllocationMap(max, rangeSpec)
		// TODO etcdallocator package to return a storage interface via the storageFactory
		etcd := etcdallocator.NewEtcd(mem, "/ranges/servicenodeports", api.Resource("servicenodeportallocations"), serviceStorageConfig)
		serviceNodePortRegistry = etcd
		return etcd
	})
	m.serviceNodePortAllocator = serviceNodePortRegistry

	controllerStorage := controlleretcd.NewStorage(restOptions("replicationControllers"))

	serviceRest := service.NewStorage(m.serviceRegistry, m.endpointRegistry, serviceClusterIPAllocator, serviceNodePortAllocator, m.ProxyTransport)

  	// 将（资源：存储）的匹配关系存放到v1ResourcesStorage中
	// TODO: Factor out the core API registration
	m.v1ResourcesStorage = map[string]rest.Storage{
		"pods":             podStorage.Pod,
		"pods/attach":      podStorage.Attach,
		"pods/status":      podStorage.Status,
		"pods/log":         podStorage.Log,
		"pods/exec":        podStorage.Exec,
		"pods/portforward": podStorage.PortForward,
		"pods/proxy":       podStorage.Proxy,
		"pods/binding":     podStorage.Binding,
		"bindings":         podStorage.Binding,

		"podTemplates": podTemplateStorage,

		"replicationControllers":        controllerStorage.Controller,
		"replicationControllers/status": controllerStorage.Status,

		"services":        serviceRest.Service,
		"services/proxy":  serviceRest.Proxy,
		"services/status": serviceStatusStorage,

		"endpoints": endpointsStorage,

		"nodes":        nodeStorage.Node,
		"nodes/status": nodeStorage.Status,
		"nodes/proxy":  nodeStorage.Proxy,

		"events": eventStorage,

		"limitRanges":                   limitRangeStorage,
		"resourceQuotas":                resourceQuotaStorage,
		"resourceQuotas/status":         resourceQuotaStatusStorage,
		"namespaces":                    namespaceStorage,
		"namespaces/status":             namespaceStatusStorage,
		"namespaces/finalize":           namespaceFinalizeStorage,
		"secrets":                       secretStorage,
		"serviceAccounts":               serviceAccountStorage,
		"persistentVolumes":             persistentVolumeStorage,
		"persistentVolumes/status":      persistentVolumeStatusStorage,
		"persistentVolumeClaims":        persistentVolumeClaimStorage,
		"persistentVolumeClaims/status": persistentVolumeClaimStatusStorage,
		"configMaps":                    configMapStorage,

		"componentStatuses": componentstatus.NewStorage(func() map[string]apiserver.Server { return m.getServersToValidate(c) }),
	}
	if registered.IsEnabledVersion(unversioned.GroupVersion{Group: "autoscaling", Version: "v1"}) {
		m.v1ResourcesStorage["replicationControllers/scale"] = controllerStorage.Scale
	}
	if registered.IsEnabledVersion(unversioned.GroupVersion{Group: "policy", Version: "v1alpha1"}) {
		m.v1ResourcesStorage["pods/eviction"] = podStorage.Eviction
	}
}
```

以pod为例，首先构造pod的后端存储podStorage：

```go
podStorage := podetcd.NewStorage(
		restOptions("pods"),
		kubeletclient.ConnectionInfoGetter(nodeStorage.Node),
		m.ProxyTransport,
	)
```

然后注册pod及其子资源的后端存储到成员变量v1ResourcesStorage中：

```go
m.v1ResourcesStorage = map[string]rest.Storage{
		"pods":             podStorage.Pod,
		"pods/attach":      podStorage.Attach,
		"pods/status":      podStorage.Status,
		"pods/log":         podStorage.Log,
		"pods/exec":        podStorage.Exec,
		"pods/portforward": podStorage.PortForward,
		"pods/proxy":       podStorage.Proxy,
		"pods/binding":     podStorage.Binding,
		"bindings":         podStorage.Binding,
		...
}
```

那么APIs的实装在哪里，我们来看下m.InstallAPIGroups(apiGroupsInfo)这个方法：

```go
// Exposes the given group versions in API. Helper method to install multiple group versions at once.
func (s *GenericAPIServer) InstallAPIGroups(groupsInfo []APIGroupInfo) error {
	for _, apiGroupInfo := range groupsInfo {
		if err := s.InstallAPIGroup(&apiGroupInfo); err != nil {
			return err
		}
	}
	return nil
}
```

InstallAPIGroups()实际上是批量运行InstallAPIGroup()方法的封装，继续跟进到InstallAPIGroup()这个方法:

```go
// Exposes the given group version in API.
func (s *GenericAPIServer) InstallAPIGroup(apiGroupInfo *APIGroupInfo) error {
	apiPrefix := s.APIGroupPrefix
	if apiGroupInfo.IsLegacyGroup {
		apiPrefix = s.APIPrefix
	}

	// Install REST handlers for all the versions in this group.
	apiVersions := []string{}
	for _, groupVersion := range apiGroupInfo.GroupMeta.GroupVersions {
		apiVersions = append(apiVersions, groupVersion.Version)

		apiGroupVersion, err := s.getAPIGroupVersion(apiGroupInfo, groupVersion, apiPrefix)
		if err != nil {
			return err
		}
		if apiGroupInfo.OptionsExternalVersion != nil {
			apiGroupVersion.OptionsExternalVersion = apiGroupInfo.OptionsExternalVersion
		}

		if err := apiGroupVersion.InstallREST(s.HandlerContainer); err != nil {
			return fmt.Errorf("Unable to setup API %v: %v", apiGroupInfo, err)
		}
	}
  
	// Install the version handler.
	...
  
	apiserver.InstallServiceErrorHandler(s.Serializer, s.HandlerContainer, s.NewRequestInfoResolver(), apiVersions)
	return nil
}
```

调用getAPIGroupVersion()方法组装APIGroupVersion类型对象，APIGroupVersion用于转换存储配置(rest.Storage)到Restful后端处理(Restful Handlers)。

> APIGroupVersion is a helper for exposing rest.Storage objects as http.Handlers via go-restful

然后调用APIGroupVersion中的InstallREST()方法实装APIs：

```go
// InstallREST registers the REST handlers (storage, watch, proxy and redirect) into a restful Container.
// It is expected that the provided path root prefix will serve all operations. Root MUST NOT end
// in a slash.
func (g *APIGroupVersion) InstallREST(container *restful.Container) error {
	installer := g.newInstaller()
  	// 为每个资源新建WebService
	ws := installer.NewWebService()
  	// 安装Routes
	apiResources, registrationErrors := installer.Install(ws)
	lister := g.ResourceLister
	if lister == nil {
		lister = staticLister{apiResources}
	}
	AddSupportedResourcesWebService(g.Serializer, ws, g.GroupVersion, lister)
  	// 将WebService加入到Container中
	container.Add(ws)
	return utilerrors.NewAggregate(registrationErrors)
}
```

InstallREST()代码逻辑很简单，就是为资源新建一个WebService，然后调用installer.Install(ws)向WebService中添加资源对应的路由，最后将WebService加入到Container中。

installer.Install(ws)方法中调用APIInstaller.registerResourceHandlers()来进行Routes的安装，registerResourceHandlers()方法代码有500行左右，这里只贴出关键逻辑：

```go
func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService, proxyHandler http.Handler) (*unversioned.APIResource, error) {
  
	...

	// what verbs are supported by the storage, used to know what verbs we support per path
	creater, isCreater := storage.(rest.Creater)
	namedCreater, isNamedCreater := storage.(rest.NamedCreater)
	lister, isLister := storage.(rest.Lister)
	getter, isGetter := storage.(rest.Getter)
	getterWithOptions, isGetterWithOptions := storage.(rest.GetterWithOptions)
	deleter, isDeleter := storage.(rest.Deleter)
	gracefulDeleter, isGracefulDeleter := storage.(rest.GracefulDeleter)
	collectionDeleter, isCollectionDeleter := storage.(rest.CollectionDeleter)
	updater, isUpdater := storage.(rest.Updater)
	patcher, isPatcher := storage.(rest.Patcher)
	watcher, isWatcher := storage.(rest.Watcher)
	_, isRedirector := storage.(rest.Redirector)
	connecter, isConnecter := storage.(rest.Connecter)
	storageMeta, isMetadata := storage.(rest.StorageMetadata)
	if !isMetadata {
		storageMeta = defaultStorageMetadata{}
	}
	exporter, isExporter := storage.(rest.Exporter)
	if !isExporter {
		exporter = nil
	}

	versionedExportOptions, err := a.group.Creater.New(optionsExternalVersion.WithKind("ExportOptions"))
	if err != nil {
		return nil, err
	}

	if isNamedCreater {
		isCreater = true
	}

	...
  
	var apiResource unversioned.APIResource
	// Get the list of actions for the given scope.
	switch scope.Name() {
	case meta.RESTScopeNameRoot:
		// 组装actions
      
      	...
      
		break
	case meta.RESTScopeNameNamespace:
		// Handler for standard REST verbs (GET, PUT, POST and DELETE).
		// 组装actions
      
      	...
      
		break
	default:
		return nil, fmt.Errorf("unsupported restscope: %s", scope.Name())
	}

	// Create Routes for the actions.
	// TODO: Add status documentation using Returns()
	// Errors (see api/errors/errors.go as well as go-restful router):
	// http.StatusNotFound, http.StatusMethodNotAllowed,
	// http.StatusUnsupportedMediaType, http.StatusNotAcceptable,
	// http.StatusBadRequest, http.StatusUnauthorized, http.StatusForbidden,
	// http.StatusRequestTimeout, http.StatusConflict, http.StatusPreconditionFailed,
	// 422 (StatusUnprocessableEntity), http.StatusInternalServerError,
	// http.StatusServiceUnavailable
	// and api error codes
	// Note that if we specify a versioned Status object here, we may need to
	// create one for the tests, also
	// Success:
	// http.StatusOK, http.StatusCreated, http.StatusAccepted, http.StatusNoContent
	//
	// test/integration/auth_test.go is currently the most comprehensive status code test

	reqScope := RequestScope{
		ContextFunc:    ctxFn,
		Serializer:     a.group.Serializer,
		ParameterCodec: a.group.ParameterCodec,
		Creater:        a.group.Creater,
		Convertor:      a.group.Convertor,
		Copier:         a.group.Copier,

		// TODO: This seems wrong for cross-group subresources. It makes an assumption that a subresource and its parent are in the same group version. Revisit this.
		Resource:    a.group.GroupVersion.WithResource(resource),
		Subresource: subresource,
		Kind:        fqKindToRegister,
	}
	for _, action := range actions {
		reqScope.Namer = action.Namer
		namespaced := ""
		if apiResource.Namespaced {
			namespaced = "Namespaced"
		}
		switch action.Verb {
		case "GET": // Get a resource.
			var handler restful.RouteFunction
          	// 创建handler
			if isGetterWithOptions {
				handler = GetResourceWithOptions(getterWithOptions, reqScope)
			} else {
				handler = GetResource(getter, exporter, reqScope)
			}
			handler = metrics.InstrumentRouteFunc(action.Verb, resource, handler)
			doc := "read the specified " + kind
			if hasSubresource {
				doc = "read " + subresource + " of the specified " + kind
			}
          
          	// 设置路由属性，包括Path，Handler，Doc，Param，MIME等
			route := ws.GET(action.Path).To(handler).
				Doc(doc).
				Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed.")).
				Operation("read"+namespaced+kind+strings.Title(subresource)).
				Produces(append(storageMeta.ProducesMIMETypes(action.Verb), a.group.Serializer.SupportedMediaTypes()...)...).
				Returns(http.StatusOK, "OK", versionedObject).
				Writes(versionedObject)
          
			...
          
          	// 设置路由的参数
			addParams(route, action.Params)
          	// 将路由添加到WebService中
			ws.Route(route)
		case "LIST": // List all resources of a kind.
          
			...
          
			ws.Route(route)
		case "PUT": // Update a resource.
          
			...
          
			ws.Route(route)
		case "POST": // Create a resource.
          
			...
          
			ws.Route(route)
		case "DELETE": // Delete a resource.
          
			...
          
			ws.Route(route)
		// TODO: deprecated
		case "WATCHLIST": // Watch all resources of a kind.
          
			...
          
			ws.Route(route)
		// We add "proxy" subresource to remove the need for the generic top level prefix proxy.
		// The generic top level prefix proxy is deprecated in v1.2, and will be removed in 1.3, or 1.4 at the latest.
		// TODO: DEPRECATED in v1.2.
		case "PROXY": // Proxy requests to a resource.
			// Accept all methods as per http://issue.k8s.io/3996
			addProxyRoute(ws, "GET", a.prefix, action.Path, proxyHandler, namespaced, kind, resource, subresource, hasSubresource, action.Params)
			addProxyRoute(ws, "PUT", a.prefix, action.Path, proxyHandler, namespaced, kind, resource, subresource, hasSubresource, action.Params)
			addProxyRoute(ws, "POST", a.prefix, action.Path, proxyHandler, namespaced, kind, resource, subresource, hasSubresource, action.Params)
			addProxyRoute(ws, "DELETE", a.prefix, action.Path, proxyHandler, namespaced, kind, resource, subresource, hasSubresource, action.Params)
			addProxyRoute(ws, "HEAD", a.prefix, action.Path, proxyHandler, namespaced, kind, resource, subresource, hasSubresource, action.Params)
			addProxyRoute(ws, "OPTIONS", a.prefix, action.Path, proxyHandler, namespaced, kind, resource, subresource, hasSubresource, action.Params)
		case "CONNECT":
			for _, method := range connecter.ConnectMethods() {
              
				...
              
				ws.Route(route)
			}
		default:
			return nil, fmt.Errorf("unrecognized action verb: %s", action.Verb)
		}
		// Note: update GetAttribs() when adding a custom handler.
	}
	return &apiResource, nil
}
```

代码首先对资源的后端存储(rest.Storage)进行验证，根据存储支持的方法进行actions的初始化，然后根据actions来进行Route的构建，以Get方法为例，参见代码注释。

至此，所有路由构建的相关组件Container，WebService， Route等已经初始化完毕。

我们再回到Master.Run()方法，其实也就是GenericAPIServer.Run()方法：

```go
func (s *GenericAPIServer) Run(options *options.ServerRunOptions) {
  
	...

	if secureLocation != "" {
		handler := apiserver.TimeoutHandler(apiserver.RecoverPanics(s.Handler), longRunningTimeout)
		secureServer := &http.Server{
			Addr:           secureLocation,
			Handler:        apiserver.MaxInFlightLimit(sem, longRunningRequestCheck, handler),
			MaxHeaderBytes: 1 << 20,
			TLSConfig: &tls.Config{
				// Can't use SSLv3 because of POODLE and BEAST
				// Can't use TLSv1.0 because of POODLE and BEAST using CBC cipher
				// Can't use TLSv1.1 because of RC4 cipher usage
				MinVersion: tls.VersionTLS12,
			},
		}

		...
      
		go func() {
			defer utilruntime.HandleCrash()
			for {
				// err == systemd.SdNotifyNoSocket when not running on a systemd system
				if err := systemd.SdNotify("READY=1\n"); err != nil && err != systemd.SdNotifyNoSocket {
					glog.Errorf("Unable to send systemd daemon successful start message: %v\n", err)
				}
				if err := secureServer.ListenAndServeTLS(options.TLSCertFile, options.TLSPrivateKeyFile); err != nil {
					glog.Errorf("Unable to listen for secure (%v); will try again.", err)
				}
				time.Sleep(15 * time.Second)
			}
		}()
	} else {
		// err == systemd.SdNotifyNoSocket when not running on a systemd system
		if err := systemd.SdNotify("READY=1\n"); err != nil && err != systemd.SdNotifyNoSocket {
			glog.Errorf("Unable to send systemd daemon successful start message: %v\n", err)
		}
	}

	handler := apiserver.TimeoutHandler(apiserver.RecoverPanics(s.InsecureHandler), longRunningTimeout)
	http := &http.Server{
		Addr:           insecureLocation,
		Handler:        handler,
		MaxHeaderBytes: 1 << 20,
	}

	glog.Infof("Serving insecurely on %s", insecureLocation)
	go func() {
		defer utilruntime.HandleCrash()
		for {
			if err := http.ListenAndServe(); err != nil {
				glog.Errorf("Unable to listen for insecure (%v); will try again.", err)
			}
			time.Sleep(15 * time.Second)
		}
	}()
	select {}
}
```

可以看到Run()中通过http.ListenAndServe()方法启动了两个端口来监听请求： Localhost Port和Secure Port。

至此，整个apiserver的Restful服务已经构建完成并运行。

