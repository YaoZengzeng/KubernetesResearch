## Envoy核心架构分析

本文将对Envoy的核心架构进行分析，主要包含如下三个组成部分：第一部分“架构分析”将对Envoy的整体架构进行宏观地介绍，第二部分“初始化”对Envoy的初始化逻辑进行了梳理，最后一部分“请求处理”则说明了当Envoy接收到一个请求之后Envoy是如何对其进行处理的。

理论上，对于架构分析来说，其实应该尽量减少对代码的引用，一方面引用的代码太多会极大地降低文章的可读性，使得文章叙述的重点不再突出，另一方面架构的稳定性显然是好于代码的，代码的频繁变更会使得文章显得“过时”。

不过由于Envoy的代码确实非常庞杂，经常会出现刚刚把逻辑理清楚，第二天就遗忘了的情况。同时，据我观察Envoy的核心代码其实已经比较稳定了，核心的数据结构基本不会产生太大的变化。因此，本文在分析过程中对Envoy中主要的数据结构进行引用，主要用于在需要进一步了解细节的时候能够快速定位代码并帮助回忆起相关的处理逻辑



### 1. 架构分析

从功能上看，Envoy本质上是一个网络代理，处理流程简单地说就是从downstream接收流量（无论是客户端直连还是iptables转发）并根据配置（无论是基于静态配置还是动态配置）进行一系列处理（即各类filters）后，基于路由选择相应的Cluster，再通过配置的负载均衡策略从Cluster中选择一个合适的Endpoint，将处理后的流量发送至目标Endpoint。

从线程模型上看，Envoy是基于事件的线程模型，包含两类线程，一类是main thread，主要用于Envoy的整体生命周期管理（初始化）、配置管理（加载初始配置，更重要的是处理基于xDS的动态配置）以及log、stats、trace等非核心处理逻辑；另一类线程是worker thread，它们真正用于处理来自downstream的请求并在处理后发往upstream，一般worker thread的数目与主机的核数相等。

需要注意的是，当接收到一个downstream connection，它就会被分配给其中一个worker thread，而它的整个生命周期也将和该worker thread相绑定，这样一来worker threads之间可以基本不用共享状态，各自独立运行，从而充分保证Envoy的处理能力能够线性扩展。

![arch](https://www.envoyproxy.io/docs/envoy/latest/_images/lor-architecture.svg)



从架构上看，Envoy主要由Listener子系统和Cluster子系统构成，分别由`ListenerManagerImpl`和`ClusterManagerImpl`两个数据结构进行承载。

Listener子系统管理所有的listeners以及所有的处理连接的worker（一个worker即一个worker thread），主要负责对来自downstream的请求进行处理，管理downstream request的生命周期，包括接收连接、分配连接到各个worker，在worker中根据匹配的配置，利用各种filters对请求进行处理等等，另外对于来自upstream的response也要进行相应的处理。

Cluster子系统负责维护所有clusters以及对应的endpoints信息，它会根据downstream request的配置为其选择一个合适的endpoints并建立相应的连接（事实上会维护一个连接池），最终将来自经过Listener子系统处理后的请求发送至upstream，完成代理。

路由真正将两个子系统串联在一起，即根据downstream request的配置为其选择合适的upstream cluster。如上图所示，对于七层流量的处理，路由是由router filter（一个http filter）完成的；如果处理的是四层流量，则路由功能由tcp_proxy filter中的路由模块实现。

### 2. 初始化



#### 建立listener

`ListenerManagerImpl`首先基于listener的配置信息构建出`ListenerImpl`对象，该对象提供了Listener的配置接口（`Network::ListenerConfig`）以及创建各种filter chain的接口（`Network::FilterChainFactory`）。`ListenerImpl`提供的主要接口如下：

```c++
// 负载均衡接口，ActiveTcpListener在构造时会通过本接口进行注册，
// 当ActiveTcpListener接收到连接时，必要时，会调用本接口，将连接转发到负载最小的ActiveTcpListener
virtual ConnectionBalancer& connectionBalancer() PURE;

// listener socket的创建接口
virtual ListenSocketFactory& listenSocketFactory() PURE;
```



之后`ListenerManagerImpl`会将`ListenerImpl`依次加入到各个worker中，最终调用的是`ConnectionHandlerImpl`的`addListener()`函数，假设添加的listener监听的TCP并且该listener之前并不存在，则会创建一个`ActiveTcpListener`对象，显然，`ActiveTcpListener`是每个worker自有的。该对象提供的主要接口如下：

```c++
// 提供给Network::TcpListenerImpl的回调函数，在接收到连接的时候被调用，
// 经过判断是否超过listener的连接上限并且负载均衡之后，创建ActiveTcpSocket对象并在其上构建
// 并且遍历listener filter chains
void onAccept(Network::ConnectionSocketPtr&& socket);
```

在`ActiveTcpListener`的构造函数中，调用dispatcher创建`TcpListenerImpl`对象，`TcpListenerImpl`真正对listener socket的事件进行监听。

### 3. 请求处理

#### 3.1 建立连接

到有连接事件到达时，回调函数`TcpListenerImpl::onSocketEvent()`获取底层的connection socket并构建`AcceptedSocketImpl`对象，并回调`ActiveTcpListener::onAccept()`函数。`onAccept()`函数在接收到连接之后会进行前置的判断，比如连接数是否超过了 listener的配置的上限，对连接进行负载均衡，找到负载最小的`ActiveTcpListener`，最后基于socket构建`ActiveTcpSocket`对象并调用`ListenerImpl.filterChainFactory().createListenerFilterChain()`在`ActiveTcpSocket`之上构建listener filter chain并调用`ActiveTcpSocket::continueFilterChain()`。`ActiveTcpSocket`的主要接口如下：

```c++
// 遍历listener filter chain，完成之后调用newConnection()
void continueFilterChain(bool success) override;

// 如果连接是经过iptables重定向的，则需要根据原先的目的地址找到真正的ActiveTcpListener，
// 调用ActiveTcpListener::onAcceptWorker()函数，将上文的步骤再执行一遍，最后还是调用本函数，
// 最终调用ActiveStreamListenerBase::newConnection()
void ActiveTcpSocket::newConnection();
```

`ActiveStreamListenerBase`其实是`ActiveTcpListener`的父类，其主要接口如下：

```c++
// 基于socket的信息，调用ListenerImpl的filterChainManager获取匹配的filter chain，
// 调用filter chain的transportSocketFactory()接口创建transport socket，
// 接着调用dispatcher的createServerConnection()，真正构建server connection，
// 即Network::ServerConnectionImpl，并在连接上构建Network filter chain，
// 最终，调用ActiveTcpListener::newActiveConnection()，
// 构建filter chain和connection之间的映射并保存在ActiveTcpListener中
void newConnection(Network::ConnectionSocketPtr&& socket, ...);
```

`Network::ServerConnectionImpl`继承自`Network::ConnectionImpl`函数，`Network::ConnectionImpl`真正对socket的读写事件进行处理。

#### 3.2 请求处理

``Network::ConnectionImpl``在构造函数中初始化了``filter_manager_``，``write_buffer_``，``read_buffer_``等对象，最主要的是建立了连接socket的处理函数``onFileEvent``，``Network::ConnectionImpl``的主要接口如下：

```C++
// 对连接socket上的Closed, Write, Read等事件进行处理
void ConnectionImpl::onFileEvent(uint32_t events)
    
// 从transport_socket_读取数据并存入read_buffer_中，最终调用ConnectionImpl::onRead()函数
// 而onRead()最终调用filter_manager_.onRead()，开始对network filters的遍历
void ConnectionImpl::onReadReady()
```

``filter_manager_``在``Network::ConnecitonImpl``的构造函数中被创建，随后在``ActiveStreamListenerBase::newConnection()``函数中调用``createNetworkFilterChain()``函数，在``filter_manager_``之上构建filter chain配置中指定的network filter。需要注意的是，在对network filter进行配置时，对于read network filter，会调用``initializeReadFilterCallbacks()``对read network filter进行初始化，对于write network filter操作类似 。构建完成之后，调用``filter_manager_.initializeReadFilters()``函数。``Network::FilterManagerImpl``的主要接口如下：

````c++
// 直接调用onContinueReading()函数
void FilterManagerImpl::onRead()
    
// 本函数分别在FilterManagerImpl::initializeReadFilters()和FilterManagerImpl::onRead()函数
// 被调用，分别用于network filter chains的初始化以及对连接接收到的数据进行处理，前者调用
// network filter的"onNewConnection()"接口，而后者调用"onData()"接口
void onContinueReading(ActiveReadFilter* filter, ReadBufferSource& buffer_source)
````

Envoy整体的请求处理框架如上所示，至于具体的处理细节会在一个个network filter中实现。Envoy既可以做七层代理也可以做四层代理，分别由``http_connection_manager``和``tcp_proxy``两个network filter实现。由于``http_connection_manager``是最为复杂的network filter实现了，因此为了说明的简单性，本文将以``tcp_proxy``为例来说明Envoy后半部分的处理过程。

#### 3.3 tcp_proxy处理流程分析



### 参考文献

* [Life of a Request](https://www.envoyproxy.io/docs/envoy/latest/intro/life_of_a_request)
* [Envoy源码注释](https://github.com/YaoZengzeng/envoy/tree/settings-comments)

