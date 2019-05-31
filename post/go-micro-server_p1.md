---
title: "micro框架源码详解 Server Package篇（一）"
date: 2019-05-20T15:34:20+08:00
draft: false
---

在go-micro，server包用于构建应用服务，为其他的服务请求提供接口。注意，本次解析针对release v1.1.0版本，[点击查看源码](https://github.com/micro/go-micro/tree/v1.1.0/server)。  

## Server接口定义

```
// Server is a simple micro server abstraction
type Server interface {
	Options() Options
	Init(...Option) error
	Handle(Handler) error
	NewHandler(interface{}, ...HandlerOption) Handler
	NewSubscriber(string, interface{}, ...SubscriberOption) Subscriber
	Subscribe(Subscriber) error
	Start() error
	Stop() error
	String() string
}
```

* Options:返回Server配置，该结构存储Server所有的业务逻辑信息，如命名空间、版本、地址等基本信息还有注册中心组件、路由组件、编解码器组件等。
* Init:初始化操作。
* Handle:注册绑定一个请求处理器。（handler是处理请求的实体，从代码上来讲是由object包装转化而来，如果object下有n个方法，那么这些方法都可以认为是这个handler的方法，通过转化绑定提供对外（该服务外）服务。如protobuf定义的service就可以认为是object实现的一个interface，经过protoc生成代码后就是一个interface，service下的rpc方法就是interface的方法，object在再去实现这些个方法，最后调用`Server.NewHandler`生成handler。下文中的这些处理器都称为handler）。
* NewHandler:将一个object包装成一个处理器（handler），一般与Handle一起使用。
* NewSubscriber:将一个object或者一个函数包装成一个订阅者接口，一般与Subscribe一起使用。
* Subscribe:注册绑定一个订阅者。
* Start:服务开始。
* Stop:服务结束。
* String:返回服务类型

Server的具体实现逻辑在这里无法体现，需要配合[DefaultServer](https://github.com/micro/go-micro/blob/v1.1.0/server/server.go#L122:2)来看具体的接口实现。  

## DefaultServer实现

DefualtServer的由newRpcServer生成，结构对象为
```
type rpcServer struct {
	router *router
	exit   chan chan error

	sync.RWMutex
	opts        Options
	handlers    map[string]Handler
	subscribers map[*subscriber][]broker.Subscriber
	// used for first registration
	registered bool
	// graceful exit
	wg sync.WaitGroup
}
```
* router:为路由组件（下称router），主要用于请求服务的匹配与服务调用执行前预处理。
* exit:退出信号，Server退出信号。
* sync.RWMutex:操作读写锁。
* opts:Server配置，与Server接口中Options()返回一致
* handlers:服务处理器列表，包括方法名(micro api意义上的)，被包装的对象，对应的endpoint组(注册中心意义上)以及元数据，在服务注册时有重要作用。
* subscribers:消息订阅组件列表，数据结构为以订阅器object指针为key，value为以该object生成的`borker.Subscriber`接口。主要用于接受订阅消息处理。在服务注册时有重要作用。
* registered:注册标志为，表示该服务是否已经注册到注册中心。
* wg:当前服务请求等待器，当服务需要退出会用于等待当前所有请求完成。

下面将看一下`rpcServer`（即DefaultServer）对于Micro Server接口结构接口的实现  

### Init()
初始化部分比较简单，接受依赖注入。

### Start()
流程如下:  

* 1.注册debug handler。
* 2.获取传输组件（下称transport）信息，传输层开始监听。
* 3.异步消息组件开始建立连接。
* 4.新建goroutinue，利用transport处理接入连接，处理函数为`ServeConn`。
* 5.新建goroutinue，根据配置做服务定时注册策略，并等待退出信号，做好退出处理。

在该函数中，其他标准micro组件的功能这里不做解析，这里都是基本连接操作。这里主要关注`ServeConn`函数的实现，该函数（下称处理函数）为服务所有请求的处理入口。  

处理函数会不断向transport获取消息，当拿到一条完整的消息时，重新根据transport的header重新构建上下文(context)的header，并添加transport的远程地址与本地地址。  

随后根据该消息的header获取确定消息的编码类型，生成相应的编码解码器，构建处理器（处理的当前请求的业务函数，与上文说到的handler有关）的请求结构与返回结构，通过router获取到匹配的相依服务的处理器，最后带入context和构建的参数让处理器完成这次请求。  

router在次过程中服务与方法的匹配，确认当前服务是否有请求的方法，最后返回相应的处理器。关于router，会有其他的组件也会用到，以后再讲。  

### Stop()
停止服务部分也比较简单，就是往exit发送退出信号。  

### NewHandler()、Handle()
请求处理器相关部分，主要逻辑在router中，这里不细说。唯一注意的每次调用`Handle`时都会往`rpcServer.handlers`服务处理器列表中添加handler的信息，用于服务注册。  

### NewSubscriber()、Subscribe（）
订阅器相关部分，主要逻辑在[subscribe](https://github.com/micro/go-micro/blob/v1.1.0/server/subscriber.go)，与[broker组件](https://github.com/micro/go-micro/tree/v1.1.0/broker)有联系，但由并不是broker，这里也不细说。主要完成的功能就是完成相关数据结构转换，每次调用`Subscribe`时都会往`rpcServer.subscribers`消息订阅组件添加信息，用于服务注册。  

在调用`NewSubscriber`函数时，第二个参数是订阅处理器，这个参数可以是一个函数类型，也可以是一个函数类型，也可以是一个object。 如果是一个函数类型，则有订阅消息时回调;如果是一个object，则有订阅消息时都会回调该object的所有方法，同时object在服务注册时，会将该object的所有方法都会独立注册成一个订阅器。  

当然，并不是所有的函数（object方法）都能成为订阅器，只有形如`func(context.Context, interface{}) error`的函数或object方法才能成为订阅器，否则在调用`Subscribe`时会因为参数错误而返回`error`。  

### String()
返回服务的实现名称，rpc

### Options()
返回服务的配置信息

### 那些非Server接口函数

#### newRpcServer()
该函数生成一个`rpcServer`的object是[Server.NewServer](https://github.com/micro/go-micro/blob/v1.1.0/server/server.go#L140:1)所默认调用的方法，也是`DefaultServer`的默认调用方法。该方法主要定义了Options中很多默认的组件，如订阅器、注册中心、传输组件、debug组件、监听地址、服务名、id、版本，当然这些组件也可以通过传入的Option参数来自定义所需修改的组件。当完成组件定义后，剩下的就是对一些变量的空指针初始化，最后返回object。  

#### Register()
该接口实现了在调用注册组件前的数据构建和填充，最终将信息注册到注册中心（下称registry）。构造过程为：  

* 1.根据配置获取服务的本地ip和服务端口，获取ip的方式为先获取本机所有的网卡网络，优先返回私网ip，如果过没有私网ip才返回公网ip。  
* 2.根据工单获取的id端口和Options配置信息构造一个[registry.Node](https://github.com/micro/go-micro/blob/v1.1.0/registry/service.go#L11:6)。  
* 3.分别遍历`rpcServer.handlers`和`rpcServer.subscribers`的key构建数组，分别排序，然后分别取出数组元素中的[registry.Endpoint](https://github.com/micro/go-micro/blob/v1.1.0/registry/service.go#L18:6)结构，将两个数组合并成一个新的Endpoint数组。  
* 4.将2中构建的Node与3中的Endpoint和配置信息，再构建成一个[registry.Service](https://github.com/micro/go-micro/blob/v1.1.0/registry/service.go#L3:6)，调用注册组件将服务注册到registry，并修改`rpcServer.registered`的状态。  
* 5.重新遍历`rpcServer.subscribers`，将订阅器信息绑定到[borker](https://github.com/micro/go-micro/blob/v1.1.0/broker/broker.go#L64:4)中，只有通过这一步的绑定，broker订阅时才能真正调用回调处理订阅。  

#### Deregister()
该接口实现了服务在registry的注销。其过程与`Register`基本一致，都是先获取服务的ip端口信息，构建`registry.Node`与`registry.Service`，注销无需构建registry.EndPoint，然后调用registry注销该服务，解除`rpcServer.subscribers`与broker的绑定关系，修改`rpcServer.registered`的状态。  

## 总结
总体来看，server担当的多个组件的逻辑处理过程，通过适当的接口暴露（Handle、Subscribe）来添加业务的拓展能力。但这些组件调用逻辑都仅限于`rpcServer`，如果过需要自定义server，则需要重新考虑server与其他组件（如broker、registry等）的依赖绑定关系，重新编写组合逻辑。
