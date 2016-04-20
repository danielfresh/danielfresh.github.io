---
layout:     post
title:      "qemu-ga & windows"
subtitle:   "qemu-ga windows下的安装及监控开发"
date:       2016-04-20 00:00:00
author:     "Daniel"
header-img: "img/post-bg-re-vs-ng2.jpg"
header-mask: 0.3
catalog:    true
tags:
    - qemu-ga
    - 监控
---

## windows安装qemu-ga

### 虚拟机配置里添加virtio serial端口

virsh edit instance-name

devices里添加下面这段配置，

	<channel type='unix'>
		<source mode='bind' path='/var/lib/libvirt/qemu/org.qemu.guest_agent.0.instance-name.sock'/>
		<target type='virtio' name='org.qemu.guest_agent.0'/>
		<address type='virtio-serial' controller='0' bus='0' port='1'/>
	</channel>

### 安装Qemu Guest Agent服务

wget https://fedorapeople.org/groups/virt/virtio-win/virtio-win.repo -O /etc/yum.repos.d/virtio-win.repo

yum install virtio-win

安装virtio-win包，virtio-win里包含windows virtio设备驱动，及qemu-ga的安装包。
![](http://danielfresh.github.io/img/virtio-iso.jpg)

将msi安装文件和virtio-win.iso拷贝到windows下。

virtio-win.iso包含各个windows版本下的virtio serial驱动，安装对应版本的驱动。
![](http://danielfresh.github.io/img/win-dev.jpg)

安装对应版本qemu-ga.msi，至此，qga安装完毕。



## qga功能扩展

###     搭建编译环境：

- rpm -ivh epel-release

- 下载 Microsoft VSS SDK: [http://www.microsoft.com/en-us/download/details.aspx?id=23490](http://www.microsoft.com/en-us/download/details.aspx?id=23490)

- 提取Microsoft VSS SDK的源文件（qemu/scripts下提供了extract-vsssdk-headers，也可以在windows下解压后拷贝到linux上）

- wget [http://wiki.qemu-project.org/download/qemu-2.6.0-rc1.tar.bz2](http://wiki.qemu-project.org/download/qemu-2.6.0-rc1.tar.bz2)

- yum install -y mingw64-pixman

- yum install -y mingw64-glib2

- yum install -y mingw64-gmp

- yum install -y mingw64-SDL

- yum install -y mingw64-pkg-config

- ./configure --enable-guest-agent --with-vss-sdk=/vss_path --cross-prefix=x86_64-w64-mingw32-

- make qemu-ga.exe

###    添加功能示例（添加内存使用率检测）    

编辑qapi-schema.json，添加自定义结构体及命令声明。

	##  结构体声明
	# @GuestMemInfo
	#
	# Information about Memory usage.
	#
	# @total: total size of Memory
	#
	# @usage: Memory usage
	#
	# Since 2.6
	##
	{ 'struct': 'GuestMemInfo',
	  'data': { 'total': 'int', 'usage': 'int' } }
	          
	##  命令声明
	# @guest-get-mem-usage:
	#
	# Get the memory utilization rate.
	#
	# Returns: @GuestMemInfo on success.
	#
	# Since 2.6
	##
	{ 'command': 'guest-get-mem-usage',
	  'returns': 'GuestMemInfo' }

在commands-win32.c中添加命令实现：

	GuestMemInfo *qmp_guest_get_mem_usage(Error **errp)
	{
	  MEMORYSTATUS ms;
	  GuestMemInfo *meminfo = g_new0(GuestMemInfo, 1);
	  GlobalMemoryStatus(&ms);
	  meminfo->usage = (int64_t)ms.dwMemoryLoad;
	  meminfo->total = (int64_t)ms.dwTotalPhys;
	  return meminfo;
	}

在commands-posix.c中添加命令实现(直接返回QERR_UNSUPPORTED错误)：

	GuestMemInfo *qmp_guest_get_mem_usage(Error **errp)
	{
	  error_setg(errp, QERR_UNSUPPORTED);
	}

重新编译qemu-ga.exe，替换C:\Program Files\qemu-ga下的qemu-ga.exe，重启服务即可。（dll依赖，在/usr/x86_64-w64-mingw32/sys-root/mingw/bin/下可找到。）

![](http://danielfresh.github.io/img/result.jpg)



参考文献：

http://wiki.qemu.org/Features/QAPI/GuestAgent

http://wiki.libvirt.org/page/Qemu_guest_agent

http://fedoraproject.org/wiki/Windows_Virtio_Drivers

http://lists.gnu.org/archive/html/qemu-discuss/2014-11/msg00027.html

