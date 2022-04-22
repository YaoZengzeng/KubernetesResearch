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

`tcp_proxy`的初始化分为两部分，第一部分是在Filter的构造函数中，主要初始化了其中的`config_`，`cluster_manager_`，`downstream_callbacks_`以及`upstream_callback_`这几个对象，第二部分的初始化则由`Filter::initializeReadFilterCallbacks`完成，主要用于初始化`read_callbacks_`对象并且将`downstream_callbacks_`加入到downstream connection中，用于响应来自downstream connection的事件。`tcp_proxy` filter中各主要字段及其功能如下：

```c++
// 该tcp_proxy filter的配置
const ConfigSharedPtr config_;
// 根据配置获取目标cluster的信息
Upstream::ClusterManager& cluster_manager_;
// 主要用于获取downstream connection的信息
Network::ReadFilterCallbacks* read_callbacks_{};
// 对于downstream connection的事件处理函数
DownstreamCallbacks downstream_callbacks_;
// 对于upstream connection的事件处理函数，与一般的Network::ConnectionCallbacks
// 相比，主要扩展了onUpstreamData(...)方法，处理来自upstream的数据，通常情况下都是
// 直接调用tcp_proxy的onUpstreamData函数
std::shared_ptr<UpstreamCallbacks> upstream_callbacks_;
// 连接池配置
std::unique_ptr<GenericConnPool> generic_conn_pool_;
```

`tcp_proxy`同样符合一般的network filter的处理流程，当downstream connection刚刚建立时，会调用它的`onNewConnection()`函数，而有downstream的数据需要处理时，则会调用它的`onData()`函数。另外，函数`onUpstreamData()`处理来自upstream的数据。`tcp_proxy` filter的主要接口如下：

```c++
// 对新的downstream connection进行处理，如果config_中设置了downstream connection最大的
// 连接超时时间，则进行处理，之后直接调用initializeUpstreamConnection()
Network::FilterStatus Filter::onNewConnection()
// 初始化upstream connection，首先路由获取目标cluster，不过和http流量的路由处理相比，tcp流量
// 要简单得多，一般会在配置中直接指定目标cluster，从cluster_manager_获取对应的thread_local_cluster
// 之后，再经过一系列的配置，最终调用maybeTunnel()函数
Network::FilterStatus Filter::initializeUpstreamConnection()
// 获取连接池工厂，根据配置创建一个连接池，即generic_conn_pool_，最后调用
// generic_conn_pool_->newStream(*this)真正在连接池中创建/获取连接
bool Filter::maybeTunnel(Upstream::ThreadLocalCluster& cluster)
// 连接池创建upsream connection失败，则回调此函数
void Filter::onGenericPoolFailure(...)
// 连接池创建upstream connection成功，则回调此函数，对upstream_进行赋值，打开downstream
// connection的readDisable()并且调用network filter manager的onContinueReading()继续
// 对network filter进行遍历
void Filter::onGenericPoolReady(...)
// 处理来自downstream connection的数据，直接写入upstream_即可，如果它已经被赋值了的话
Network::FilterStatus Filter::onData(Buffer::Instance& data, bool end_stream)
// 处理来自upstream connection的数据，直接写入downstream connection，
// read_callbacks_->connection().write()即可
void Filter::onUpstreamData(Buffer::Instance& data, bool end_stream)
```

现在我们来关注连接池的创建，首先会构建一个名为`TcpConnPool`的对象，不过这个对象的作用更多的是功能的适配，用于衔接`tcp_proxy` filter的接口和真正的连接池实例的接口，`TcpConnPool`的主要接口如下：

```C++
// 构造函数，调用thread_local_cluster真正创建连接池并保存在conn_pool_data_中
TcpConnPool(Upstream::ThreadLocalCluster& thread_local_cluster, ...)
// 通用TCP连接池的回调函数，直接转发调用callbacks_，即tcp_proxy filter
// 的onGenericPoolFailure()函数
void onPoolFailure(ConnectionPool::PoolFailureReason reason, ...)
// 通用TCP连接池的回调函数，构建TcpUpstream并转发给tcp_proxy filter的onGenericPoolReady()
void onPoolReady(Tcp::ConnectionPool::ConnectionDataPtr&& conn_data, ...)
// 直接调用conn_pool_data_创建新的连接
void newStream(GenericConnectionPoolCallbacks& callbacks)
```

需要注意的是，为什么不让`tcp_proxy`直接适配TCP连接池的回调接口，而用`TcpConnPool`进行转发，原因是`tcp_proxy`可以配置隧道，即利用HTTP连接池代理TCP连接。因此`tcp_proxy`的接口为`onGenericPoolReady()`，通过`TcpConnPool`和`HttpConnPool`进行转换，从而既能适配TCP，也能适配HTTP连接池的回调接口。

最终`thread_local_cluster`创建类型为`Tcp::ConnPoolImpl`的连接池，事实上`Tcp::ConnPoolImpl`是对`Envoy::ConnectionPool::ConnPoolImplBase`的封装，该结构被TCP连接池和HTTP连接共用，实现连接池的通用核心逻辑。下面，我们先来看`Tcp::ConnPoolImpl`的主要接口：

```c++
// 将callbacks（即上文的TcpConnPool）封装进TcpAttachContext结构，调用newStreamImpl()函数
ConnectionPool::Cancellable* newConnection(Tcp::ConnectionPool::Callbacks& callbacks)
// 当连接池中没有可用连接时，将stream封装为pending stream，需要注意的是，pending stream保存在
// 底层的ConnPoolImplBase中
ConnectionPool::Cancellable* newPendingStream(Envoy::ConnectionPool::AttachContext& context)
// 构造ActiveTcpClient，在该构造函数中真正调用底层接口真正创建连接
Envoy::ConnectionPool::ActiveClientPtr instantiateActiveClient()
// 从context中解析出callbacks，即TcpConnPool，调用它的onPoolReady()接口
// 并且将ActiveTcpClient和Network::ClientConnection打包成TcpConnectionData
// 作为参数传入
void onPoolReady(...ActiveClient& client,...AttachContext& context)
// 直接调用TcpConnPool的onPoolFailure()接口
void onPoolFailure()
```

`ConnPoolImplBase::newStreamImpl(AttachContext& context)`真正用于串联downstream和upstream，如果连接池中有合适的连接，则调用`ConnPoolImplBase::attachStreamToClient(client, context)`将上下游进行关联，否则调用`newPendingStream(context)`将downstream的相关信息先保存起来，再调用`ConnPoolImplBase::tryCreateNewConnections()`异步创建连接。`ConnPoolImplBase`的主要字段如下：

```c++
// 一系列处于各种状态的clients
std::list<ActiveClientPtr> ready_clients_;
std::list<ActiveClientPtr> busy_clients_;
std::list<ActiveClientPtr> connecting_clients_;
// 一系列的pending streams
std::list<PendingStreamPtr> pending_streams_;
```

`ConnPoolImplBase`的主要接口如下：

```c++
// 如果ready_clients_非空，则调用attachStreamToClient将上下游进行关联，
// 否则调用newPendingStream()构建pending stream并调用tryCreateNewConnections()
// 异步创建连接
ConnectionPool::Cancellable* ConnPoolImplBase::newStreamImpl(AttachContext& context)
// 进行一系列是否需要创建新连接的判断，如果真的需要，则调用instantiateActiveClient()
ConnPoolImplBase::ConnectionResult ConnPoolImplBase::tryCreateNewConnection(...)
// 本函数在ConnPoolImplBase中是纯虚函数，对于TCP连接池则创建ActiveTcpClient对象
Envoy::ConnectionPool::ActiveClientPtr instantiateActiveClient()
// 进行client的状态以及一些监控指标的更新，最后调用onPoolReady()
void attachStreamToClient(...ActiveClient& client,AttachContext& context)
// 纯虚函数，ConnPoolImplBase不实现
void onPoolFailure(...)
void onPoolReady(...)
// 对于upstream connection的事件处理，当为Connected事件时，转换client状态并且
// 调用onUpstreamReady()函数
void ConnPoolImplBase::onConnectionEvent(ActiveClient& client, absl::string_view failure_reason, Network::ConnectionEvent event)
// 匹配pending_streams_和ready_clients_，调用attachStreamToClient()
// 如果还有多余的pending_stream_，则再调用tryCreateNewConnections()创建新连接
void ConnPoolImplBase::onUpstreamReady()
```

`ActiveTcpClient`在构造函数中调用`host->createConnection(...)`真正创建底层连接并调用`connection_->addConnectionCallbacks(*this)`在连接中注册事件处理函数。它的主要接口如下：

```C++
// 构造函数，创建底层的upstream连接并且注册事件处理函数
// 另外构造ConnReadFilter对象，将它作为read filter加入upstream connection中，
// 用于接收来自upstream connection的数据，最终调用ActiveTcpClient::onUpstreamData()函数
ActiveTcpClient::ActiveTcpClient(...ConnPoolImplBase& parent,...HostConstSharedPtr& host, uint64_t concurrent_stream_limit)
// 底层upstream连接的事件处理函数，一般为调用parent，即ConnPoolImplBase的onConnectionEvent(...)
// 函数
void ActiveTcpClient::onEvent(Network::ConnectionEvent event)
// 处理来自upstream connection的数据，如果callbacks_字段不为空，则调用它的onUpstreamData函数
void onUpstreamData(Buffer::Instance& data, bool end_stream)
```

最后，总结`tcp_proxy`创建upstream connection并进行数据传输的步骤如下：

1. 调用`TcpConnPool::newStream()`试着异步创建upstream connection
2. 当upstream connection处于ready状态时，通过层层的`onPoolReady()`回调，最终调用到`TcpConnPool::onPoolReady`，将upstream的connection以及`ActiveTcpClient`都封装至`TcpUpstream`中，其中最为重要的是将`upstream_callbacks_`注册到`ActiveTcpClient`中，从而能串联`onUpstreamData`函数对于来自upstream connection的数据的流转
3. `tcp_proxy`的`onData()`函数处理来自downstream connection的数据，最终调用`upstream_->encodeData（）`将数据转发到upstream connection
4. `tcp_proxy`的`onUpstreamData()`函数处理来自upstream connection的数据，最终调用`read_callbacks_->connection().write()`将数据转发到downstream connection



### 参考文献

* [Life of a Request](https://www.envoyproxy.io/docs/envoy/latest/intro/life_of_a_request)
* [Envoy源码注释](https://github.com/YaoZengzeng/envoy/tree/settings-comments)

