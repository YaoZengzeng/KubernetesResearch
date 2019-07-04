## Kubernetes Controller执行框架解析

毫无疑问，声明式API以及Controller机制是Kubernetes设计理念的基础。Controller不断从API Server同步资源对象的期望状态并且在资源对象的期望状态和实际运行状态之间进行调谐，从而实现两者的最终一致性。Kubernetes系统中的各种组件，包括Scheduler，Kubelet以及各种资源对象的Controller都以这种统一的模式运行着。

虽然Kubernetes已经内置了Deployment，StatefulSet，Job等丰富的编排对象，但是在落地过程中，面对纷繁复杂的应用场景，尤其是针对Etcd，Prometheus等配置复杂的有状态应用，现有的编排对象依然显得捉襟见肘。所幸的是，Kubernetes在v1.7引入了CRD，允许用户自定义资源对象并且可以像原生对象一样操作它们。因此对于复杂的应用，我们完全可以自定义编排对象以及相应的Controller进行配置管理。

同时，声明式API的设计也让其他系统与Kubernetes的集成变得更为容易。例如，当使用Prometheus对Kubernetes系统进行监控，并开启抓取对象的自动发现时，Prometheus本质上也是一个Controller，它会不断地从API Server同步集群中所有的Pod，Service和Node信息并从中筛选出抓取对象进行抓取。

可见，Controller在Kubernetes生态系统中无处不在。事实上，Controller的执行框架都是类似的，Kubernetes社区也贴心地将通用的代码进行了抽象，封装了在[client-go](https://github.com/kubernetes/client-go.git)这个包里。手写Controller的难度也瞬间下降到了在代码中引入client-go包并且在适当的位置实现业务逻辑这样的"体力活"。话虽如此，但是对Controller执行框架的深入理解，对于更好地编写Controller乃至更好地理解Kubernetes都将是不可或缺的，而这也是本文叙述的重点。

### 1. Controller实现概述

不论对于同步资源对象的真正意义上的Controller，还是类似于Prometheus仅仅全量获取资源对象进行服务发现的广义的Controller。从API Server持续同步资源对象并对其进行处理是所有Controller都相同的执行框架。乍一看，这个问题并不复杂，我们甚至可以写一个脚本对API Server进行轮询，每次全量获取资源对象并逐个处理，似乎也能满足要求。

当然，在现实条件下，上述朴素的方法显然是不行的。从上文的分析可知，Controller是普遍存在的，而且资源对象之间往往存在关联关系，例如Deployment和Pod，Service和Endpoint，因此一个Controller通常需要对多种资源对象进行同步。因此，如此大量的轮询操作，对于API Server来说是无法接受的。

对于这个问题，Kubernetes社区的解法是List & Watch。首先利用List接口从API Server中获取资源对象的全量数据并存储在缓存中，之后再利用Watch接口对API Server进行监听并且以增量事件的方式持续接收资源对象的变更。一方面Controller可以对这些资源对象的增量变更事件进行即时处理，另一方面，也可对缓存进行更新，保证缓存与API Server中的源数据保持最终的一致性。

落实到具体的实现，通用的Controller架构如下图所示：

![arch](./pic/kubecontroller/arch.jpeg)

上图中有一条虚线体贴地将整张架构图切割为成了两部分：图的上半部分是client-go封装的与API Server的底层交互逻辑；下半部分则是Controller的业务代码，主要是针对资源对象增量事件的处理。接下来，本文将以资源对象的增量事件从API Server传输到Controller之后，在架构中的流转作为顺序，依次说明各个组件所起的作用。

### 2. Reflector & Delta FIFO

Reflector负责对特定的资源对象进行监听并且将资源对象的所有变更推送到Delta FIFO这个队列中。首先，Reflector会利用特定资源对象的client向API Server发起List操作，获取该资源对象的全量信息并将它们推送到Delta FIFO中。List本质上就是一个HTTP请求，其Request/Response类似的形式如下：

```sh
 GET /api/v1/namespaces/test/pods
 ---
 200 OK
 Content-Type: application/json
 {
   "kind": "PodList",
   "apiVersion": "v1",
   "metadata": {"resourceVersion":"10245"},
   "items": [...]
 }
```

List操作相当于对API Server中指定资源对象的内容作了一个快照，返回的元数据当中的ResourceVersion用于标记该快照。ResourceVersion本质上是资源对象在底层数据库中的版本号，它会随着资源对象的变更而不断累加。之后我们只要持续不断地利用Watch从API Server中获取资源对象的所有变更事件用于更新Controller中的缓存，那么缓存中的数据就可以认为与API Server中的数据是一致的（当然，最终还是与etcd中的数据一致）。Watch本质上也是一个HTTP请求，不过它是一个长连接，会不断地从服务器端获取数据，具体形式如下：

```sh
GET /api/v1/namespaces/test/pods?watch=1&resourceVersion=10245
 ---
 200 OK
 Transfer-Encoding: chunked
 Content-Type: application/json
 {
   "type": "ADDED",
   "object": {"kind": "Pod", "apiVersion": "v1", "metadata": {"resourceVersion": "10596", ...}, ...}
 }
 {
   "type": "MODIFIED",
   "object": {"kind": "Pod", "apiVersion": "v1", "metadata": {"resourceVersion": "11020", ...}, ...}
 }
 ...
```

Watch会持续地返回资源对象的变更事件，"type"字段指定了事件的类型，主要类型如下：

* Added：新的资源对象实例，例如：新增一个Pod
* Modified：资源对象实例更新，例如：更新一个Pod
* Deleted：资源对象实例删除，例如：删除一个Pod

"object"字段则包含了对应资源对象实例的全部信息。Reflector维护了一个lastSyncResourceVersion字段，最初由List返回结果中的ResourcVersion初始化，之后每次从Watch接收到一个事件，都会利用该事件中包含对象的ResourceVersion更新lastSyncResourceVersion。实际上，现实世界里网络是不可能永远稳定的，Watch底层的连接也随时可能会断开。一旦，Watch断开，Reflector就会以lastSyncResourceVersion为起点重新开始对API Server进行监听，从而既不会遗漏也不会重复接受资源对象的变更事件，保证了缓存数据与API Sever中原始数据的一致性。

Delta FIFO是一个先进先出队列，其中缓存了资源对象的变更事件，每一个事件即一个如下所示的Delta结构：

```go
type Delta struct {
	Type	DeltaType	// Added, Updated, Deleted, Sync
	Object	interface{}
}
```

其中Type是事件的类型，Object则是变更发生后，资源对象的状态。

Delta FIFO也是一个典型的生产者-消费者模型，Reflector就是生产者，它通过List&Watch从API Server获取相应的事件并根据事件的类型转换为Delta结构并放入队列中。而如下所示的Delta FIFO的Pop方法就是该队列的消费者：

```
type PopProcessFunc func(interface{}) error

func (f *DeltaFIFO) Pop(process PopProcessFunc) (interface{}, error)
```

Pop方法会持续地队列进行监听，一旦队列中存在"消费品"时，Pop方法就会将其出队并交由类型为PopProcessFunc的处理函数进行处理。这里需要注意的是，Pop方法每次从队列中弹出的并不是某个资源的单个事件，而是该资源在队列期间按发生的先后排好序的所有变更事件的集合。从而对于每个资源对象实例，你只需要处理它一次并且每次处理你都能看到从上次处理它以来，发生在它身上所有的变更。

### 3. Informer

从上文的架构图可以看到，Informer这个结构处于整个架构的中心，简单地说，它有以下三个作用：

* 接收Delta FIFO的Pop方法弹出的资源对象的一系列变更事件并根据事件的类型进行处理
* 根据资源对象的变更事件更新本地缓存，即Indexer
* 将资源对象的变更事件分发给已经在Informer中注册的各个事件变更类型的Handler进行处理

如果你仔细研究过`client-go`的源码，那你一定感受过被Index支配的恐惧：`Index`，`Indexers`，`Indices`以及`IndexFunc`这几种名字看起来非常相似但是作用却完全不同的自定义类型一定让你感到晕头转向。事实上，Indexer仅仅是只是在最简单的缓存之上封装了一个索引的功能。

`client-go`实现缓存的方法很简单，就是golang原生的Map加上一个读写锁，也就是上面架构图中的Thread Safe Store。其中Map的Value是具体的资源对象实例，例如PodA，而Map的Key的形式一般为`Namespace/Name`，例如对于NamespaceA下的PodA，它的Key即为`NamespaceA/PodA`，对于类似于节点这种不属于任何Namespace的资源对象，则Key直接为`Name`。

当监听的资源对象是Pod时，获取某个Namespace之下的所有Pod，或者获取和某个Pod处于同一个Namaspace下的所有Pod等等，都是常见的需求。但是如果没有索引的话，我们每次都要遍历Map，从中筛选出符合要求的资源对象实例，这种方式显然是不能满足要求的。

因此，缓存Thread Safe Store在真正实现时，对应到具体的数据结构如下：

```go
type threadSafeMap struct {
	lock	sync.RWMutex
	items	map[string]interface{}
	
	indexers	Indexers
	indices	Indices
}


type IndexFunc func(obj interface{}) ([]string, error)
type Indexers	map[string]IndexFunc

type Index map[string]sets.String
type Indices map[string]Index
```

可以看到，在实现加锁Map的结构以外还增加了一个类型为`Indexers`和`Indices`的字段，那么这两个字段的作用又是什么呢？对于一组资源对象，我们可以从很多的维度构建索引，而`Indexers`其实是一组索引生成函数，每个键值对都代表不同的索引维度，最常见的自然是基于Namespace进行索引，此时`Indexers`的Key就是字符串`namespace`，而Value则为函数`MetaNamespaceIndexFunc`，它的参数是任意的资源对象，返回的则是该资源对象所处的Namespace。而`Indices`则是一个存储索引的结构。类似地，它的Key也是构建索引的维度，例如此处的`namespace`，而`Index`则是真正存储索引的结构，Key为某个索引值，Value则是与该索引值匹配的资源对象实例的Key。下面我们用一个例子对上述机制进行更为直观的说明。

例如，当NamespaceA下新增了一个PodA时，需要根据该事件更新缓存。`threadSafeMap`首先计算出这个Pod在`items`中存储的键值，即`NamespaceA/PodA`，接下来还需要为该Pod建立各个维度的索引，此时我们遍历`Indexers`调用各个维度的索引构造函数，例如对于`namespace`这个维度，索引构造函数得到PodA对应的索引值是NamespaceA，然后我们再用这一结果更新`Indices`，即`threadSafeMap.indices["namespace"]["NamespaceA"] = sets.String{..., "NamespaceA/PodA"}`。之后如果我们再想通过缓存获取NamespaceA下的所有Pod信息，那么最终利用的是`threadSafeMap.indices["namespace"]["NamespaceA"]`得到一系列的键值，再利用这些键值从`threadSafeMap.items`得到相关的Pod信息。索引建立之后，筛选特定的资源对象就不再需要使用遍历缓存这种原始而低效的方法了。

Controller基于的是一种面向事件的编程模型，因此Controller中业务逻辑的实现本质上是对与资源对象变更事件的处理。因此，通常需要向Informer注册针对变更事件的处理函数，分别对资源对象的增加（Add），修改（Update）以及删除（Delete）进行处理。当Informer从Delta FIFO弹出Delta时，它会根据Delta的类型，将Delta中的对象交由对应变更类型的事件处理函数进行处理。

上述即是client-go对于Controller通用逻辑的封装，概括地说，就是Reflector利用List & Watch从API Server得到关于特定资源对象的变更事件，Informer再利用这些事件更新缓存并且用注册的事件处理函数对各个事件进行处理。

client-go对Controller中的通用逻辑进行了封装，本质上，它为我们提供了两组接口：一组是特定资源对象的缓存，即上文中的Indexer，通过它可以知道对应资源对象所有实例的状态；另一组就是我们可以向Informer注册的事件处理函数，一旦某个资源对象实例发生变更，相应的事件处理函数就会被调用。

### 4. Controller中业务逻辑的实现

通过上文的分析可知，对于Controller的实现者来说，他所要做的仅仅是将业务逻辑封装在资源对象变更事件的处理函数里并向Controller注册。从编码的角度来看，Controller开发者只要实现如下的接口即可：

```golang
type ResourceEventHandler interface {
	OnAdd(obj interface{})
	OnUpdate(oldObj, newObj interface{})
	OnDelete(obj interface{})
}
// 实际向Informer注册的其实是`ResourceEventHandlerFuncs`，`ResourceEventHandlerFuncs`可以允许指定三种事件处理函数中的某几种，但依然确保能够实现接口`ResourceEventHandler`
```

每当相应的事件类型发生，对应的处理函数就会被调用。不过通常来说，用户并不会真的在事件处理函数里面实现自己的业务逻辑。例如，Kubelet本质也是一个Controller，它会对Pod进行监听并且注册相应的处理函数进行处理。当系统中新增一个Pod时，Kubelet显然不会直接在OnAdd()函数中直接就开始将这个新增的Pod实例化，因为这样的操作往往非常耗时，从而阻塞Informer对于事件的分发。

因此，事件处理函数中的逻辑往往非常简单。它会直接获得一个Key用于表示该对象，就像我们在上文的缓存中做的那样，例如NamespaceA下的PodA，它的Key就是`NamespaceA/PodA`，并将这个Key放入一个队列。同时，我们可以创建多个Goroutine从队列中获取Key，对发生变更的资源对象实例进行处理。那么如何从Key转换成具体的对象实例呢？显然缓存可以帮我们做到这一点。获取到实例之后，我们就可以从容地展开业务逻辑了。例如，对于Kubelet来说就可以用新的Pod配置去创建或者更新Pod实例了（如果在缓存中找不到Key对应的实例，则说明它已经被删除了）。可以发现，这部分叙述的内容和上文架构图虚线以下的部分是完全对应的。

### 5. 总结

Controller其根本的理念是非常简单的：实现实际状态和期望状态的最终一致性。如果API Server有着无限的并发能力，Controller的实现完全可以简单到直接通过轮询全量资源对象的目标状态，然后据此对实际状态进行调整。但实际情况是，API Server的并发能力是有限的，因此我们需要利用List & Watch进行缓存，以增量式地，面向事件的模式对资源对象的更新进行处理。所幸的是，社区对Controller实现框架中的通用部分进行了抽象封装，从而极大地减轻了Controller开发者的心智负担，可以更多地专注于业务逻辑的处理。

### 参考文献

* [client-go source code](https://github.com/kubernetes/client-go)
* [prometheus-operator source code](https://github.com/coreos/prometheus-operator)
* [sample-controller source code](https://github.com/kubernetes/sample-controller)
* [client-go under the hood](https://github.com/kubernetes/sample-controller/blob/master/docs/controller-client-go.md)
* [Kubernetes Deep Dive: Code Generation for CustomResources](https://blog.openshift.com/kubernetes-deep-dive-code-generation-customresources/)
* [Kubernetes设计理念分析](https://juejin.im/entry/5ad95f55f265da0b767d042d)

