## Prometheus告警模型分析

Prometheus作为时下最为流行的开源监控系统，其庞大的生态体系：包括针对各种传统应用的Exporter，完整的二次开发工具链，与Kubernetes等主流平台的高度亲和以及由此带来的强大的自发现能力，使得我们通过简单的配置就能获取大量的监控指标且包含的维度及其丰富。一方面，如此多样的指标极大地提高了集群的可观测性，配合Grafana等Dashboard就能让我们实时了解集群各个维度的状态；另一方面，基于监控数据进行实时地告警也是在可观测性得到满足之后必然要实现的需求。当然，Prometheus社区已经很好地解决了这个问题，本文也将对Prometheus的告警模型进行详细的叙述。

### 1. 概述

如果对Prometheus项目有所了解的话，可以发现，Prometheus一个非常重要的原则就是尽量让设计保持简洁并且用简洁的设计满足绝大多数场景的需求。同时让项目保持良好的扩展性，针对极端场景，可以拼接Prometheus生态的一些外围组件来对已有能力进行增强，从而满足要求。对于告警也是类似的，基于Prometheus的告警系统的整体架构下图所示：

![arch](./pic/alertmanager/arch.jpg)

告警系统整体被解耦为两部分：

1. Prometheus Server会读取一系列的告警规则并基于采集的监控数据定期对这些规则进行评估，一旦满足触发条件就会生成相应的告警实例发送至AlertManager
2. AlertManager是一个独立于Prometheus Server运行的HTTP Server，它负责接受来自Client端的告警实例并对这些实例进行聚合（aggregation），静默（silence），抑制（inhibit）等高级操作并且支持Email，Slack等多种通知平台对告警进行通知。对于AlertManager来说，它并不在乎告警实例是否是由Prometheus Server发出的，因此我们只要构造出符合要求的告警实例并发送至Alertmanager，它就能无差别地进行处理。

### 2. Alert Rules

通常来说，Prometheus的告警规则都会以文件的形式保存在磁盘中，我们需要在配置文件中指定这些规则文件的位置供Prometheus Server启动时读取：

```yaml
rule_files:
  - /etc/prometheus/rules/*.yaml
```

一般一个规则文件的内容如下：

```yaml
groups:
- name: example
  rules:
  - alert: HighRequestLoad
    expr: rate(http_request_total{pod="p1"}[5m]) > 1000
    for: 1m
    labels:
      severity: warning
    annotations:
      info: High Request Load
```

在一个规则文件中可以指定若干个group，每个group内可以指定多条告警规则。一般来说，一个group中的告警规则之间会存在某种逻辑上的联系，但即使它们毫无关联，对后续的流程也不会有任何影响。而一条告警规则中包含的字段及其含义如下：

1. `alert`: 告警名称
2. `expr`: 告警的触发条件，本质上是一条promQL查询表达式，Prometheus Server会定期（一般为15s）对该表达式进行查询，若能够得到相应的时间序列，则告警被触发
3. `for`: 告警持续触发的时间，因为数据可能存在毛刺，Prometheus并不会因为在`expr`第一次满足的时候就生成告警实例发送到AlertManager。比如上面的例子意为名为"p1"的Pod，每秒接受的HTTP请求的数目超过1000时触发告警且持续时间为一分钟，若告警规则每15s评估一次，则表示只有在连续四次评估该Pod的负载都超过1000QPS的情况下，才会真正生成告警实例。
4. `labels`: 用于附加到告警实例中的标签，Prometheus会将此处的标签和评估`expr`得到的时间序列的标签进行合并作为告警实例的标签。告警实例中的标签构成了该实例的唯一标识。事实上，告警名称最后也会包含在告警实例的label中，且key为"alertname"。
5. `annotations`: 用于附加到告警实例中的额外信息，Prometheus会将此处的annotations作为告警实例的annotations，一般annotations用于指定告警详情等较为次要的信息

需要注意的是，一条告警规则并不只会生成一类告警实例，例如对于上面的例子，可能有如下多条时间序列满足告警的触发条件，即n1和n2这两个namespace下名为p1的pod的QPS都持续超过了1000：

```
http_request_total{namespace="n1", pod="p1"}
http_request_total{namespace="n2", pod="p1"}
```

最终生成的两类告警实例为：

```
# 此处只显示实例的label
{alertname="HighRequestLoad", severity="warning", namespace="n1", pod="p1"}
{alertname="HighRequestLoad", severity="warning", namespace="n2", pod="p1"}
```

因此，例如在K8S场景下，由于Pod具有易失性，我们完全可以利用强大的promQL语句，定义一条Deployment层面的告警，只要其中任何的Pod满足触发条件，都会产生对应的告警实例。

### 3. 在Kubernetes下操作Alert Rules

初一看，Prometheus这种将所有告警规则一股脑写入文件中的方式貌似很简单，事实上，这的确简化了Prometheus本身的设计实现难度。但是，真正在生产环境中，尤其是当把Prometheus Server以Pod的形式部署在Kubernetes集群中时，对告警规则的增删改差操作将变得异常繁琐。特别地，在Kubernetes环境中，显然我们只能将若干告警规则文件包含在ConfigMap中并挂载到Prometheus所在Pod的指定目录中，如果要进行增删改操作，最直观的方法就是整体加载该ConfigMap并在修改后重新写入。

所幸，对此社区早已准备了一套完整的解决方案。我们知道，在Kubernetes体系下，管理复杂有状态应用最常用的方式就是为其编写一个专门的Operator。[Prometheus Operator](https://github.com/coreos/prometheus-operator)作为社区最早实现的Operator之一，大大简化了Prometheus的配置部署流程。Prometheus Operator将Prometheus相关的概念都抽象为了CRD。与本文相关的主要是`Prometheus`和`PrometheusRule`这两个CRD。

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
spec:
  ruleSelector:
    matchLabels:
      role: alert-rules
---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    role: alert-rules
  name: 
spec:
  groups:
  - name: example
    rules:
    - alert: HighRequestLoad
      expr: rate(http_request_total{pod="p1"}[5m]) > 1000
      for: 1m
      labels:
        severity: none
      annotations:
        info: High Request Load
```

上面展示的就是近乎最简的`Prometheus`和`PrometheusRule`资源对象。当上述yaml文件被提交至Kubernetes APIServer之后，Prometheus Operator会马上同步到并根据`Prometheus`的配置生成一个StatefulSet用于运行Prometheus Server实例，同时将`Prometheus`中的配置写入Server的配置文件中。对于`PrometheusRule`，我们可以发现它的内容与上面的告警规则文件是基本一致的。Prometheus Operator会依据`PrometheusRule`的内容生成相应的ConfigMap并将其以Volume的形式挂载到Prometheus Server所在Pod的对应目录中。

那么Operator是如何将`Prometheus`和`PrometheusRule`关联在一起的呢？类似于Service通过Selector字段指定关联的Pod。`Prometheus`也通过ruleSelector字段指定了一组label，Operator会将任何包含这些label的`PrometheusRule`都整合到一个ConfigMap(若超出单个ConfigMap的限制，则生成多个)并挂载到`Prometheus`对应的StatefulSet的各个Pod实例中。因此，在Prometheus Operator的帮助下，对于Prometheus告警规则进行增删改查的难度已经退化到对Kubernetes资源对象的CRUD操作，整个过程中最为繁琐的部分已经完全被Operator自动化了。事实上，Prometheus Server的高级配置乃至AlertManager的部署都可以通过Prometheus Operator提供的CRD轻松实现，因为与本文关联不大，所以不再赘述了。

最后，虽然Operator能保证对于`PrometheusRule`的增删改查能及时反映到相应的ConfigMap中，而Kubernetes本身则保证了ConfigMap的修改也最终能同步到相应Pod的挂载文件中，但是Prometheus Server并不会监听告警规则文件的变更。因此，我们需要以Sidecar的形式将[ConfigMap Reloader](https://github.com/jimmidyson/configmap-reload)部署在Prometheus Server所在的Pod内。由它来监听告警规则所在ConfigMap的变更，一旦监听到变化，它就会调用Prometheus Server提供的Reload接口，触发Prometheus对于配置的重新加载。
