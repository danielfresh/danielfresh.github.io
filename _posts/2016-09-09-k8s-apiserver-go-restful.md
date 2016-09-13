---
layout:     post
title:      "kubernetes源码解析"
subtitle:   "apiserver路由构建解析(1)"
date:       2016-09-13 08:00:00
author:     "Daniel"
header-img: "img/apiserver.png"
header-mask: 0.3
catalog:    true
tags:
    - golang
    - kubernetes
    - go-restful
    - kube-apiserver
---

## kubernetes源码解析---- apiserver路由构建解析(1)

apiserver作为k8s集群的唯一入口，内部主要实现了两个功能，一个是请求的路由和处理，简单说就是监听一个端口，把接收到的请求正确地转到相应的处理逻辑上，另一个功能就是认证及权限控制。本文主要对apiserver的路由构建过程进行解析。
apiserver使用go-restful来构建REST-style Web服务，所以我们先来了解一下这个包的相关内容，以便更好地理解apiserver的源码。

(kubernetes代码版本：1.3.6     Commit id：ed3a29bd6aeb)



### go-restful

> Package restful, a lean package for creating REST-style WebServices without magic.

[go-restful](https://github.com/emicklei/go-restful)是用于构建REST-style Web服务的一个golang包，它的来历参见[go-restful-api-design](http://ernestmicklei.com/2012/11/go-restful-api-design/)，大体意思就是一个javaer在golang中没找到顺手的REST-based服务构建包，所以就按照他在Java里常用的[JAX-RS](http://jax-rs-spec.java.net/)的设计，在golang中造了一个轮子。

#### go-restful里的几个组件和特性####

1. Route

   路由包含两种，一种是标准JSR311接口规范的实现**RouterJSR311**，一种是快速路由**CurlyRouter**。CurlyRouter支持正则表达式和动态参数，相比RouterJSR311更加轻量级，apiserver中使用的就是这种路由。

   一条Route的设定包含：请求方法(Http Method)，请求路径(URL Path)，处理方法以及可选的接受内容类型(Content-Type)，响应内容类型(Accept)等。

2. WebService

   WebService逻辑上是Route的集合，功能上主要是为一组Route统一设置包括root path，请求响应的数据类型等一些通用的属性。需要注意的是，WebService必须加入到Container中才能生效。

   ```go
   func InstallVersionHandler(mux Mux, container *restful.Container) {
   	// Set up a service to return the git code version.
   	versionWS := new(restful.WebService)
   	versionWS.Path("/version")
   	versionWS.Doc("git code version from which this is built")
   	versionWS.Route(
   		versionWS.GET("/").To(handleVersion).
   			Doc("get the code version").
   			Operation("getCodeVersion").
   			Produces(restful.MIME_JSON).
   			Consumes(restful.MIME_JSON).
   			Writes(version.Info{}))

   	container.Add(versionWS)
   }
   ```

3. Container

   Container逻辑上是WebService的集合，功能上可以实现多终端的效果。例如，下面代码中创建了两个Container，分别在不同的port上提供服务。

   ```go
   func main() {
   	ws := new(restful.WebService)
   	ws.Route(ws.GET("/hello").To(hello))
     	// ws被添加到默认的container restful.DefaultContainer中
   	restful.Add(ws)
   	go func() {
          // restful.DefaultContainer 监听在端口8080上
   		http.ListenAndServe(":8080", nil)
   	}()

   	container2 := restful.NewContainer()
   	ws2 := new(restful.WebService)
   	ws2.Route(ws2.GET("/hello").To(hello2))
     	// ws2被添加到container container2中
   	container2.Add(ws2)
     	// container2中监听在端口8081上
   	server := &http.Server{Addr: ":8081", Handler: container2}
   	log.Fatal(server.ListenAndServe())
   }

   func hello(req *restful.Request, resp *restful.Response) {
   	io.WriteString(resp, "default world")
   }

   func hello2(req *restful.Request, resp *restful.Response) {
   	io.WriteString(resp, "second world")
   }
   ```

4. Filter

   Filter用于动态的拦截请求和响应，类似于放置在相应组件前的钩子，在相应组件功能运行前捕获请求或者响应，主要用于记录log，验证，重定向等功能。go-restful中有三种类型的Filter:

   1. Container Filter

      运行在Container中所有的WebService执行之前。

      ```go
      // install a (global) filter for the default container (processed before any webservice)
      restful.Filter(globalLogging)
      ```

   2. WebService Filter

      运行在WebService中所有的Route执行之前。

      ```go
      // install a webservice filter (processed before any route)
      ws.Filter(webserviceLogging).Filter(measureTime)
      ```

   3. Route Filter

      运行在调用Route绑定的方法之前。

      ```go
      // install 2 chained route filters (processed before calling findUser)
      ws.Route(ws.GET("/{user-id}").Filter(routeLogging).Filter(NewCountFilter().routeCounter).To(findUser))
      ```



#### 使用样例####

下面代码是官方提供的例子。

```go
package main

import (
	"github.com/emicklei/go-restful"
	"log"
	"net/http"
)

type User struct {
	Id, Name string
}

type UserResource struct {
	// normally one would use DAO (data access object)
	users map[string]User
}

func (u UserResource) Register(container *restful.Container) {
  	// 创建新的WebService
	ws := new(restful.WebService)
  
  	// 设定WebService对应的路径("/users")和支持的MIME类型(restful.MIME_XML/ restful.MIME_JSON)
	ws.
		Path("/users").
		Consumes(restful.MIME_XML, restful.MIME_JSON).
		Produces(restful.MIME_JSON, restful.MIME_XML) // you can specify this per route as well

  	// 添加路由： GET /{user-id} --> u.findUser
	ws.Route(ws.GET("/{user-id}").To(u.findUser))
  
  	// 添加路由： POST / --> u.updateUser
	ws.Route(ws.POST("").To(u.updateUser))
  
  	// 添加路由： PUT /{user-id} --> u.createUser
	ws.Route(ws.PUT("/{user-id}").To(u.createUser))
  
  	// 添加路由： DELETE /{user-id} --> u.removeUser
	ws.Route(ws.DELETE("/{user-id}").To(u.removeUser))

  	// 将初始化好的WebService添加到Container中
	container.Add(ws)
}

// GET http://localhost:8080/users/1
//
func (u UserResource) findUser(request *restful.Request, response *restful.Response) {
	id := request.PathParameter("user-id")
	usr := u.users[id]
	if len(usr.Id) == 0 {
		response.AddHeader("Content-Type", "text/plain")
		response.WriteErrorString(http.StatusNotFound, "User could not be found.")
	} else {
		response.WriteEntity(usr)
	}
}

// POST http://localhost:8080/users
// <User><Id>1</Id><Name>Melissa Raspberry</Name></User>
//
func (u *UserResource) updateUser(request *restful.Request, response *restful.Response) {
	usr := new(User)
	err := request.ReadEntity(&usr)
	if err == nil {
		u.users[usr.Id] = *usr
		response.WriteEntity(usr)
	} else {
		response.AddHeader("Content-Type", "text/plain")
		response.WriteErrorString(http.StatusInternalServerError, err.Error())
	}
}

// PUT http://localhost:8080/users/1
// <User><Id>1</Id><Name>Melissa</Name></User>
//
func (u *UserResource) createUser(request *restful.Request, response *restful.Response) {
	usr := User{Id: request.PathParameter("user-id")}
	err := request.ReadEntity(&usr)
	if err == nil {
		u.users[usr.Id] = usr
		response.WriteHeader(http.StatusCreated)
		response.WriteEntity(usr)
	} else {
		response.AddHeader("Content-Type", "text/plain")
		response.WriteErrorString(http.StatusInternalServerError, err.Error())
	}
}

// DELETE http://localhost:8080/users/1
//
func (u *UserResource) removeUser(request *restful.Request, response *restful.Response) {
	id := request.PathParameter("user-id")
	delete(u.users, id)
}

func main() {
  	// 创建一个空的Container
	wsContainer := restful.NewContainer()
  
    // 设定路由为CurlyRouter
	wsContainer.Router(restful.CurlyRouter{})
  
  	// 创建自定义的Resource Handle(此处为UserResource)
	u := UserResource{map[string]User{}}
  
  	// 创建WebService，并将WebService加入到Container中
	u.Register(wsContainer)

	log.Printf("start listening on localhost:8080")
	server := &http.Server{Addr: ":8080", Handler: wsContainer}
  	
  	// 启动服务
	log.Fatal(server.ListenAndServe())
}
```

上面的示例构建Restful服务，分为几个步骤，apiserver中也是类似的：

1. 创建Container。
2. 创建自定义的Resource Handle，实现Resource相关的处理方法。
3. 创建对应于Resource的WebService，在WebService中添加相应Route，并将WebService加入到Container中。
4. 启动监听服务。



