## Kubernetes存储框架解析

针对不同的应用以及不同的场景，我们往往需要不同的存储方案以满足我们的需求，而各类存储方案的实现以及配置方法各异，直接使用通常会造成较大的心智负担。因此，Kubernetes作为容器编排领域的事实标准，设计一套通用的存储框架用以无缝接入尽量多的存储方案并能够供用户方便地使用，就显得尤为重要了。当然，这样一套框架也不是一蹴而就的，对于存储方案的集成方式大致上经历了InTree（与存储方案对接的实现直接硬编码至Kubernetes中），FlexVolume以及CSI三个阶段。本文将对最新的基于CSI的Kubernetes存储框架进行分析，包含的主要内容如下：

* 存储框架设计概述
* 存储拓扑与Pod调度
* CSI协议分析

### 1. 存储框架设计概述

对于存储来说，普通用户最关心的或者只想关心的是我要给应用挂载一个多大的盘，至于如何配置底层存储以满足需求，这对存储的使用者来说往往是一种额外的负担，例如，要学会如何配置Ceph以获取一块存储对任何初学者来说都不是一件容易的事情。因此，这也是Kubernetes设计PV/PVC/SC这一套资源对象的根本原因：

* PersistentVolumeClaim（PVC）：用户对所需存储资源的抽象描述，主要描述了所需存储资源的类型（文件系统或者块设备）、大小以及访问方式等核心诉求
* PersistentVolume（PV）：PVC与PV的关系类似于Go语言中的接口与具体实现的关系。PV不仅包含了PVC中描述的关于存储资源的抽象信息，更包含为了真正获得该资源需要的关于底层存储的配置。例如，对于基于NFS的存储，则需要在PV中配置对应的NFS服务器的IP以及路径。PV中包含了底层存储配置的相关信息，因此通常由系统管理员进行管理，对用户透明。
* StorageClass（SC）：如果用户每创建一个PVC都要由管理员创建相应的PV从而允许两者进行绑定未免太过死板。因此Kubernetes设计了StorageClass这一资源对象将两者进行联结，从而允许Kubernetes能为新创建、没有现成PV可供绑定的PVC动态创建PV并绑定。

在用户声明了PV/PVC/SC之后，Kubernetes又是做了哪些工作将声明转换为真正的存储资源供应用使用呢？同样，我们可以用三个单词进行描述，Provision/Attach/Mount，这三个操作的具体含义如下：

* Provision：调用底层的存储系统创建存储资源，最常见的就是调用底层的云服务创建某种类型的云盘
* Attach：由于使用存储资源的Pod最终会被调度到某个节点，因此对于块设备类型的存储首先需要附着到对应Pod所在的节点
* Mount：最终将存储设备挂载到Pod的数据目录，供其使用

事实上这三个动作正是对底层存储方案的抽象，任何存储方案至多只要实现这三个动作（例如基于NFS的存储就无需实现Provision/Attach）就能接入Kubernetes。至此，我们可以将Kubernetes的存储框架划分为一个三层结构，如下图所示：

![framework](./pic/storage/framework.png)

我们知道，“资源对象+控制器”是Kubernetes架构体系的基础，对于存储的实现也不例外。存储相关的控制器会持续地对集群中存储相关的资源对象进行List/Watch并根据资源对象的增删改查做出相应的调整，例如，为PVC找到合适的PV进行绑定，根据策略对PV进行回收等等。如果真正需要操纵底层的存储资源，则通过标准的Volume Plugin进行实现。最后，我们对框架的主体，即控制器部分进行说明。

![./controller](./pic/storage/controller.png)

如上图所示，存储相关的控制器主要为：PersistentVolume Controller，AttachDetach Controller，Kubelet，三者通过PV/PVC/SC/Pod/Node这五个资源对象进行交互，最终完成将用户指定的存储资源挂载到指定应用负载的任务。虽然随着整体架构的发展，各个Controller的存在形式（集成到Kube-Controller-Manager中或者以独立的方式运行）或者功能划分（Attach/Detach从Kubelet移动到AD Controller）会有所不同，但是一种存储类型要接入Kubernetes总是需要有相应的组件来完成类似的功能。下面就以上图为例，依次说明各个Controller在框架中所起的作用，一方面能够对整个框架的执行过程有一个宏观的了解，另一方面也能为后续两块内容：存储约束下的容器调度以及CSI奠定基础。

#### PV Controller

PV Controller主要包含两个逻辑处理单元：Claim Worker以及Volume Worker，它们基于标准的Kubernetes Controller模式分别对PVC和PV的Add/Update/Delete进行处理。不过PV/PVC这两个资源对象状态以及字段繁多，更由于两者之间存在绑定关系，因此无论是PV还是PVC的Add/Update/Delete都涉及大量边界条件的处理。所以下面将只对最核心的，即Unbound的PVC如何寻找合适的PV并与之绑定的过程，进行分析，从而能够对PV Controller的整体架构有着提纲挈领式的认识而不至于过度陷入细节而无法自拔（注：暂时不考虑Delay Binding的场景）。

1. Claim Worker会对PVC的Add/Update事件或者定期的全量Resync进行处理。如果PVC的annotation中不包含Key为`pv.kubernetes.io/bind-completed`的键值对，则说明PVC仍未绑定。
2. 遍历匹配所有已经存在的PV，根据AccessMode，VolumeMode，Label Selector，StorageClass以及Size等条件进行筛选，得到符合要求的PV，需要注意的是，在大小或者节点亲和性合适的情况，优先选择已经与该PVC提前绑定（pre-bound）的PV，即`Spec.Claim`字段与目标PVC完全匹配的PV。若不存在pre-bound且匹配的PV，则在剩余匹配的PV中选择Size最小的PV。
3. 若未筛选到合适的现存的PV且PVC中指定了StorageClass，则根据对应StorageClass中指定的Provisioner找到对应的Volume Plugin进行Provision，真正创建存储资源并生成相应的PV对象。其中PV的`Spec.ClaimRef`字段会用PVC对应的内容进行填充，从而保证在下一次同步过程的步骤2中，该PVC和PV能优先绑定。
4. 不论PVC通过静态查找还是动态生成找到了匹配的PV，则在两个资源对象中同时填入对方的信息，添加一些annotations以及将状态转变为`Bound`，表示二者已经绑定。否则，PVC处于`Pending`状态，等待下一次的同步。

总之，PV Controller用于对PV/PVC这两个资源对象的生命周期以及状态进行管理，简答地说，就是努力为PVC绑定最合适的PV。由于PV以及PVC之间的关联异常紧密，因此其中一个资源对象发生变化往往另一个相对的资源也需要做变更从而保持整体的一致性，这也在一定程度上增加了处理逻辑的复杂性，针对某个具体场景或者异常条件下的处理，建议直接分析源码，在此不再赘述。

#### AD Controller

AD Controller，即Attach/Detach Controller，包含如下核心组成部分：

* DesiredStateOfWorld：
* ActualStateOfWorld：
* DesiredStateOfWorldPopulator
* Reconciler：



### 参考链接

* [Kubernetes源码](https://github.com/kubernetes/kubernetes)
* [Kubernetes存储架构及插件使用](https://developer.aliyun.com/article/743613)

