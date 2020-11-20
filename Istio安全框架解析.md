## Istio安全框架解析

### 1. Istio安全概述

作为服务网格的事实标准，极大地降低微服务架构下流量管理的复杂度往往是Istio最为引入注目的特性，但事实上，随着越来越多的服务从单体架构向微服务架构演进，模块间由最初的函数调用转变为进程间通信，微服务间通信的安全性，服务的访问策略控制以及如何降低大规模场景下安全配置的复杂度等问题同样亟待解决。

当然，Istio给出了一套完整的框架用于解决这些问题，与对流量管理的处理类似，这套框架不要求对应用做任何侵入式修改，并且提供了灵活的服务访问策略以及简易的配置方式，以近乎零成本的方式解决了上述问题，有效地降低了开发运维人员的心智负担。



### 2. Istio安全架构

需要注意的是，本文主要以Istio 1.3版本作为分析对象，虽然随着Istio版本的演进，服务架构以及上层API都发生了一些变化，但是无论如何，底层原理都是类似的，因此相信这些变化并不会造成太大的困扰。Istio的架构如下所示：

![arch](./pic/istio-security/arch.png)

从安全的角度来看，各个组件的作用如下：

* **Citadel**：密钥以及证书管理
* **Mixer**：鉴权以及审计
* **Proxy**：一般就是Envoy，接收Pilot下发的安全配置并且应用于其拦截代理的流量
* **Pilot**：API转换层，将Istio提供的上层的安全相关的Kubernetes资源对象转换为底层数据面Proxy对应的安全配置



### 3. Istio安全概念、实例以及实现

#### 3.1 Istio identity

在展开Istio安全框架中的各种概念、实例以及实现方法之前，有必要首先了解Istio identity这个概念。简单地说Istio identity就是一种身份标识，在服务间通信的开始阶段，通信双方需要交换包含各自身份标识的证书，这样一来，客户端就能通过校验证书中的身份标识来确认这的确是它要访问的目标服务，而服务端则在认证客户端身份的同时，还能对该用户进行授权、审计等等更为复杂的操作。本文以Kubernetes平台作为主要的讨论背景，而在Kubernetes下，Istio identity一般与Service Account对应，通信双方会以绑定的Service Account作为各自的身份标识。

#### 3.2 Authentication概念、实例及实现

对于认证，Istio提供两种方式：**Transport authentication**以及**Origin authentication**。后者基于JWT进行认证，本文不详细展开。而Transport authentication允许在不对应用源码做侵入式修改的情况下，提供服务间的双向安全认证，同时密钥以及证书的创建、分发、轮转都由系统自动完成，对用户透明，从而大大降低了安全配置管理的复杂度。

那么如何配置认证策略呢？对此，Istio提供了名为*Policy*的Kubernetes CRD。如下所示的yaml文件的内容展示了，如何开启对default这个ns下的名为details和reviews的service的认证，且要求对于reviews的认证仅限于9080端口：

```yaml
apiVersion: "authentication.istio.io/v1alpha1"
kind: "Policy"
metadata:
  name: "details-and-reviews"
spec:
targets:
 - name: details
 - name: reviews
   ports:
   - number: 9000
  peers:
  - mtls: {}
```

另外，Istio也支持Mesh范围以及Namespace范围的全局策略配置，在此不再赘述。

当上述配置生效之后，在没有开启双向认证（mTLS）的情况下，其他服务将无法直接访问details以及reviews服务。那么该如何开启mTLS呢？Istio在*DestinationRule*这个CRD中实现了该功能。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: details
spec:
  host: details
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

通过配置`DestinationRule`中的`spec.trafficPolicy.tls.mode`为`ISTIO_MUTUAL`，其他服务在于details这个服务通信时将开启mTLS。





#### 3.3 Authorization概念、实例及实现



### 4. 证书获取机制及其演进过程



### 5. 总结

