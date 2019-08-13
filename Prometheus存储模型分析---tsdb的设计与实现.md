## Prometheus存储模型分析---tsdb的设计与实现


Prometheus是时下最为流行的开源监控解决方案，我们可以很轻松地以Prometheus为核心快速构建一套包含监控指标的抓取，存储，查询以及告警的完整监控系统。单个的Prometheus实例就能实现每秒上百万的采样，同时支持对于采集数据的快速查询，而对于Kubernetes这类抓取对象变更频繁的环境，Prometheus也是最好的选择。显然，这些优秀特性的实现都离不开一个设计优良的时序数据库的支撑。本文就将对Prometheus内置的时序数据库tsdb的设计与实现进行剖析，从架构设计以及代码层面理解它何以支持Prometheus强大读写表现。

### 1. 时序数据概述

Prometheus读写的是时序数据，与一般的数据对象相比，时序数据有其特殊性，tsdb对此进行了大量针对性的设计与优化。因此理解时序数据是理解Prometheus存储模型的第一步。通常，它由如下所示的标识和采样数据两部组成：

```bash
标识 -> {(t0, v0), (t1, v1), (t2, v2), (t3, v3)...}
```

1. 标识用于区分各个不同的监控指标，在Prometheus中通常用指标名+一系列的labels唯一地标识一个时间序列。如下为Prometheus抓取的一条时间序列，其中`http_request_total`为指标名，表示HTTP请求的总数，它有`path`和`method`两个label，用于表示各种请求的路径和方法。

```bash
http_request_total{path="/", method="GET"} -> {(t0, v1), (t1, v1)...}
```
事实上指标名最后也是作为一个特殊的label被存储的，它的key为`__name__`，如下所示。最终Prometheus存储在数据库中的时间序列标识就是一堆labels。我们将这堆labels称为`series`。

```bash
{__name__="http_request_total", path="/", method="GET"}
```

2. 采样数据则由诸多的采样点（Prometheus将采样点称为sample）构成，t0, t1, t2...表示样本采集时的时间，v0, v1, v2...则表示指标在采集时刻对应的值。t0, t1, t2...是单调递增的且相邻sample的时间间隔相同，对于Prometheus默认为15s。而且一般相邻sample的指标值v并不会相差太多。基于采样数据的上述特性，我们能有效地将其压缩存储。Prometheus对于采样数据压缩算法的实现，参考的是Facebook的时序数据库[Gorilla](http://www.vldb.org/pvldb/vol8/p1816-teller.pdf)，通过该算法，16字节的sample平均只需要1.37个字节的存储空间。

### 2. 架构设计

