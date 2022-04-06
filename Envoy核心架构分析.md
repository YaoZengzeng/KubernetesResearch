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



### 3. 请求处理



### 参考文献

* [Life of a Request](https://www.envoyproxy.io/docs/envoy/latest/intro/life_of_a_request)
* [Envoy源码注释](https://github.com/YaoZengzeng/envoy/tree/settings-comments)

