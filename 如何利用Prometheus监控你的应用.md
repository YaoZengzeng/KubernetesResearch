## 如何利用Prometheus监控你的应用

Prometheus作为一套完整的开源监控接近方案，因为其诸多强大的特性以及生态的开放性，俨然已经成为了监控领域的事实标准并在全球范围内得到了广泛的部署应用。那么应该如何利用Prometheus对我们的应用形成有效的监控呢？事实上，作为应用我们仅仅需要以符合Prometheus标准的方式暴露监控数据即可，后续对于监控数据的采集，处理，保存都将由Prometheus自动完成。

一般来说，Prometheus监控对象有两种：如果用户对应用的代码有定制化能力，Prometheus提供了各种语言的SDK，用户能够方便地将其集成至应用中，从而对应用的内部状态进行有效监控并将数据以符合Prometheus标准的格式对外暴露；对于MySQL，Nginx等应用，一方面定制化代码难度颇大，另一方面它们已经以某种格式对外暴露了监控数据，对于此类应用，我们需要一个中间组件，利用此类应用的接口获取原始监控数据并转化成符合Prometheus标准的格式对外暴露，此类中间组件，我们称之为Exporter，社区中已经有大量现成的Exporter可以直接使用。

本文将以一个由Golang编写的HTTP Server为例，借此说明如何利用Prometheus的Golang SDK，逐步添加各种监控指标并以标准的方式暴露应用的内部状态。后续的内容将分为基础、进阶两个部分，通常第一部分的内容就足以满足大多数需求，但是若想要获得更多的定制化能力，那么第二部分的内容可以提供很好的参考。

### 1. 基础

Promethues提供的各种语言的SDK包其实已经非常完善了，如果希望某个Golang程序能够被Prometheus监控，我们需要做的仅仅是引入`client_golang`这个包并添加几行代码而已，代码实例如下：

```golang
package main

import (
        "net/http"

        "github.com/prometheus/client_golang/prometheus/promhttp"
)

func main() {
        http.Handle("/metrics", promhttp.Handler())
        http.ListenAndServe(":8080", nil)
}
```

可以看到，上述代码中，我们仅仅启动了HTTP Server并将`client_golang`提供的一个默认的HTTP Handler注册到了路径`/metrics`上。我们可以试着运行该程序并对接口进行访问，结果如下：

```bash
$ curl http://127.0.0.1:8080/metrics
...
# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
go_goroutines 7
# HELP go_info Information about the Go environment.
# TYPE go_info gauge
go_info{version="go1.12.1"} 1
# HELP go_memstats_alloc_bytes Number of bytes allocated and still in use.
# TYPE go_memstats_alloc_bytes gauge
go_memstats_alloc_bytes 418912
...
```

Prometheus Golang SDK提供的默认Handler会自动注册一系列用于监控Golang运行时以及应用进程相关信息的监控指标。所以，在我们未注册任何自定义指标的情况下，依然暴露了一系列的指标。指标暴露的格式也非常统一，首行以`# HELP`开头用于说明该指标的用途，次行以`# TYPE`开头用于说明指标的类型，后续几行则是指标的具体内容。对于Prometheus来说，只要提供抓取对象的地址以及访问路径（一般为`/metrics`），它就能对如上所示的监控数据进行抓取。而对于用户来说，则只需要自定义指标并按照程序的运行情况对相应的指标值进行修改即可。中间对于监控数据的聚合暴露，Prometheus的SDK会自动帮你处理。下面，我们将结合具体的需求试着添加各种类型的自定义监控指标。

对于一个HTTP Server来说，了解当前请求的接收速率是非常重要的。Prometheus支持一种称为Counter的数据类型，这一类型本质上就是一个只能单调递增的计数器。如果我们定义一个Counter表示累积接收的HTTP请求的数目，那么最近一段时间内该Counter的增长量其实就是接收速率。另外，Prometheus中定义了一种自定义查询语句PromQL，能够方便地对样本的监控数据进行统计分析，包括对于Counter类型的数据求速率。因此，经过修改后的程序如下：

```golang
package main

import (
	"net/http"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
	http_request_total = promauto.NewCounter(
		prometheus.CounterOpts{
			Name:	"http_request_total",
			Help:	"The total number of processed http requests",
		},
	)
)

func main() {
	http.HandleFunc("/", func(http.ResponseWriter, *http.Request){
		http_request_total.Inc()
	})

	http.Handle("/metrics", promhttp.Handler())
	http.ListenAndServe(":8080", nil)
}
```

我们利用`promauto`包提供的`NewCounter`方法定义了一个Counter类型的监控指标，只需要填充名字以及帮助信息，该指标就创建完成了。需要注意的是，Counter类型数据的名字要尽量以`_total`作为后缀。否则当Prometheus与其他系统集成时，可能会出现指标无法识别的问题。每当有请求访问根目录时，该指标就会调用`Inc()`方法加一，当然，我们也可以调用`Add()`方法累加任意的非负数。

再次运行修改后的程序，先对根路径进行多次访问，再对`/metrics`路径进行访问，可以看到新定义的指标已经成功暴露了：

```golang
$ curl http://127.0.0.1:8080/metrics | grep http_request_total
# HELP http_request_total The total number of processed http requests
# TYPE http_request_total counter
http_request_total 5
```

监控累积的请求处理显然还是不够的，通常我们还想知道当前正在处理的请求的数量。Prometheus中的Gauge类型数据，与Counter不同，它既能增大也能变小。将正在处理的请求数量定义为Gauge类型是合适的。因此，我们新增的代码块如下：

```golang
...
var (
	...
	http_request_in_flight = promauto.NewGauge(
		prometheus.GaugeOpts{
			Name:	"http_request_in_flight",
			Help:	"Current number of http requests in flight",
		},
	)
)
...
http.HandleFunc("/", func(http.ResponseWriter, *http.Request){
	http_request_in_flight.Inc()
	defer http_request_in_flight.Dec()
	http_request_total.Inc()
})
...
```

Gauge和Counter类型的数据操作起来的差别并不大，唯一的区别是Gauge支持`Dec()`或者`Sub()`方法减小指标的值。

对于一个网络服务来说，能够知道它的平均时延是重要的，不过很多时候我们更想知道响应时间的分布状况。Prometheus中的Histogram类型就对此类需求提供了很好的支持。具体到需要新增的代码如下：

```golang
...
var (
	...
	http_request_duration_seconds = promauto.NewHistogram(
		prometheus.HistogramOpts{
			Name:		"http_request_duration_seconds",
			Help:		"Histogram of lantencies for HTTP requests",
			// Buckets:	[]float64{.1, .2, .4, 1, 3, 8, 20, 60, 120},
		},
	)
)
...
http.HandleFunc("/", func(http.ResponseWriter, *http.Request){
	now := time.Now()

	http_request_in_flight.Inc()
	defer http_request_in_flight.Dec()
	http_request_total.Inc()
	
	time.Sleep(time.Duration(rand.Intn(1000)) * time.Millisecond)

	http_request_duration_seconds.Observe(time.Since(now).Seconds())
})
...
```

在访问了若干次上述HTTP Server的根路径之后，从`/metrics`路径得到的响应如下：

```bash
# HELP http_request_duration_seconds Histogram of lantencies for HTTP requests
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.005"} 0
http_request_duration_seconds_bucket{le="0.01"} 0
http_request_duration_seconds_bucket{le="0.025"} 0
http_request_duration_seconds_bucket{le="0.05"} 0
http_request_duration_seconds_bucket{le="0.1"} 3
http_request_duration_seconds_bucket{le="0.25"} 3
http_request_duration_seconds_bucket{le="0.5"} 5
http_request_duration_seconds_bucket{le="1"} 8
http_request_duration_seconds_bucket{le="2.5"} 8
http_request_duration_seconds_bucket{le="5"} 8
http_request_duration_seconds_bucket{le="10"} 8
http_request_duration_seconds_bucket{le="+Inf"} 8
http_request_duration_seconds_sum 3.238809838
http_request_duration_seconds_count 8
```
Histogram类型暴露的监控数据要比Counter和Gauge复杂得多，最后以`_sum`和`_count`开头的指标分别表示总的响应时间以及对于响应时间的计数。而它们之上的若干行表示：时延在0.005秒内的响应数目，0.01秒内的响应次数，0.025秒内的响应次数...最后的`+Inf`表示响应时间无穷大的响应次数，它的值和`_count`的值是相等的。显然，Histogram类型的监控数据很好地呈现了数据的分布状态。当然，Histogram默认的边界设置，例如0.005,0.01这类数值一般是用来衡量一个网络服务的时延的。对于具体的应用场景，我们也可以对它们进行自定义，类似于上述代码中被注释掉的那一行（最后的`+Inf`会自动添加）。

与Histogram类似，Prometheus中定义了一种类型Summary，从另一个角度描绘了数据的分布状况。对于响应时延，我们可能想知道它们的中位数是多少？九分位数又是多少？对于Summary类型数据的定义及使用如下：

```golang
...
var (
	...
	http_request_summary_seconds = promauto.NewSummary(
		prometheus.SummaryOpts{
			Name:	"http_request_summary_seconds",
			Help:	"Summary of lantencies for HTTP requests",
			// Objectives: map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001, 0.999, 0.0001},
		},
	)
)
...
http.HandleFunc("/", func(http.ResponseWriter, *http.Request){
	now := time.Now()

	http_request_in_flight.Inc()
	defer http_request_in_flight.Dec()
	http_request_total.Inc()

	time.Sleep(time.Duration(rand.Intn(1000)) * time.Millisecond)

	http_request_duration_seconds.Observe(time.Since(now).Seconds())
	http_request_summary_seconds.Observe(time.Since(now).Seconds())
})
...
```

Summary的定义和使用与Histogram是类似的，最终我们得到的结果如下：

```bash
$ curl http://127.0.0.1:8080/metrics | grep http_request_summary
# HELP http_request_summary_seconds Summary of lantencies for HTTP requests
# TYPE http_request_summary_seconds summary
http_request_summary_seconds{quantile="0.5"} 0.31810446
http_request_summary_seconds{quantile="0.9"} 0.887116164
http_request_summary_seconds{quantile="0.99"} 0.887116164
http_request_summary_seconds_sum 3.2388269649999994
http_request_summary_seconds_count 8
```

同样，`_sum`和`_count`分别表示请求的总时延以及请求的数目，与Histogram不同的是，Summary其余的部分分别表示，响应时间的中位数是0.31810446秒，九分位数位0.887116164等等。我们也可以根据具体的需求对Summary呈现的分位数进行自定义，如上述程序中被注释的Objectives字段。令人疑惑的是，它是一个map类型，其中的key表示的是分位数，而value表示的则是误差。例如，上述的0.31810446秒是分布在响应数据的0.45~0.55之间的，而并非完美地落在0.5。

事实上，上述的Counter，Gauge，Histogram，Summary就是Prometheus能够支持的全部监控数据类型了（其实还有一种类型Untyped，表示未知类型）。一般使用最多的是Counter和Gauge这两种基本类型，结合PromQL对基础监控数据强大的分析处理能力，我们就能获取极其丰富的监控信息。

不过，有的时候，我们可能希望从更多的特征维度去衡量一个指标。例如，对于接收到的HTTP请求的数目，我们可能希望知道具体到每个路径接收到的请求数目。假设当前能够访问`/`和`/foo`目录，显然定义两个不同的Counter，比如http_request_root_total和http_request_foo_total，并不是一个很好的方法。一方面扩展性比较差：如果定义更多的访问路径就需要创建更多新的监控指标，同时，我们定义的特征维度往往不止一个，可能我们想知道某个路径且返回码为XXX的请求数目是多少，这种方法就无能为力了；另一方面，PromQL也无法很好地对这些指标进行聚合分析。

Prometheus对于此类问题的方法是为指标的每个特征维度定义一个label，一个label本质上就是一组键值对。一个指标可以和多个label相关联，而一个指标和一组具体的label可以唯一确定一条时间序列。对于上述分别统计每条路径的请求数目的问题，标准的Prometheus的解决方法如下：

```golang
...
var (
	http_request_total = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Name:	"http_request_total",
			Help:	"The total number of processed http requests",
		},
		[]string{"path"},
	)
...
	http.HandleFunc("/", func(http.ResponseWriter, *http.Request){
		...
		http_request_total.WithLabelValues("root").Inc()
		...
	})

	http.HandleFunc("/foo", func(http.ResponseWriter, *http.Request){
		...
		http_request_total.WithLabelValues("foo").Inc()
		...
	})
)
```

此处以Counter类型的数据举例，对于其他另外三种数据类型的操作是完全相同的。此处我们在调用`NewCounterVec`方法定义指标时，我们定义了一个名为`path`的label，在`/`和`/foo`的Handler中，`WithLabelValues`方法分别指定了label的值为`root`和`foo`，如果该值对应的时间序列不存在，则该方法会新建一个，之后的操作和普通的Counter指标没有任何不同。而最终通过`/metrics`暴露的结果如下：

```bash
$ curl http://127.0.0.1:8080/metrics | grep http_request_total
# HELP http_request_total The total number of processed http requests
# TYPE http_request_total counter
http_request_total{path="foo"} 9
http_request_total{path="root"} 5
```

可以看到，此时指标`http_request_total`对应两条时间序列，分别表示path为`foo`和`root`时的请求数目。那么如果我们反过来想统计，各个路径的请求总和呢？我们是否需要定义个path的值为`total`，用来表示总体的计数情况？显然是不必的，PromQL能够轻松地对一个指标的各个维度的数据进行聚合，通过如下语句查询Prometheus就能获得请求总和：

```bash
sum(http_request_total)
```

label在Prometheus中是一个简单而强大的工具，理论上，Prometheus没有限制一个指标能够关联的label的数目。但是，label的数目也并不是越多越好，因为每增加一个label，用户在使用PromQL的时候就需要额外考虑一个label的配置。一般来说，我们要求添加了一个label之后，对于指标的求和以及求均值都是有意义的。

### 2. 进阶

基于上文所描述的内容，我们就能很好地在自己的应用程序里面定义各种监控指标并且保证它能被Prometheus接收处理了。但是有的时候我们可能需要更强的定制化能力，尽管使用高度封装的API确实很方便，不过它附加的一些东西可能不是我们想要的，比如默认的Handler提供的Golang运行时相关以及进程相关的一些监控指标。另外，当我们自己编写Exporter的时候，该如何利用已有的组件，将应用原生的监控指标转化为符合Prometheus标准的指标。为了解决上述问题，我们有必要对Prometheus SDK内部的实现机理了解地更为深刻一些。

在Prometheus SDK中，Register和Collector是两个核心对象。Collector里面可以包含一个或者多个Metric，它事实上是一个Golang中的interface，提供如下两个方法：

```golang
type Collector interface {
	Describe(chan<- *Desc)
	
	Collect(chan<- Metric)
}
```

简单地说，Describe方法通过channel能够提供该Collector中每个Metric的描述信息，Collect方法则通过channel提供了其中每个Metric的具体数据。单单定义Collector还是不够的，我们还需要将其注册到某个Registry中，Registry会调用它的Describe方法保证新添加的Metric和之前已经存在的Metric并不冲突。而Registry则需要和具体的Handler相关联，这样当用户访问`/metrics`路径时，Handler中的Registry会调用已经注册的各个Collector的Collect方法，获取指标数据并返回。

在上文中，我们定义一个指标如此方便，根本原因是`promauto`为我们做了大量的封装，例如，对于我们使用的`promauto.NewCounter`方法，其具体实现如下：

```golang
http_request_total = promauto.NewCounterVec(
	prometheus.CounterOpts{
		Name:	"http_request_total",
		Help:	"The total number of processed http requests",
	},
	[]string{"path"},
)
---
// client_golang/prometheus/promauto/auto.go
func NewCounterVec(opts prometheus.CounterOpts, labelNames []string) *prometheus.CounterVec {
	c := prometheus.NewCounterVec(opts, labelNames)
	prometheus.MustRegister(c)
	return c
}
---
// client_golang/prometheus/counter.go
func NewCounterVec(opts CounterOpts, labelNames []string) *CounterVec {
	desc := NewDesc(
		BuildFQName(opts.Namespace, opts.Subsystem, opts.Name),
		opts.Help,
		labelNames,
		opts.ConstLabels,
	)
	return &CounterVec{
		metricVec: newMetricVec(desc, func(lvs ...string) Metric {
			if len(lvs) != len(desc.variableLabels) {
				panic(makeInconsistentCardinalityError(desc.fqName, desc.variableLabels, lvs))
			}
			result := &counter{desc: desc, labelPairs: makeLabelPairs(desc, lvs)}
			result.init(result) // Init self-collection.
			return result
		}),
	}
}
```

一个Counter（或者CounterVec，即包含label的Counter）其实就是一个Collector的具体实现，它的Describe方法提供的描述信息，无非就是指标的名字，帮助信息以及定义的Label的名字。`promauto`在对它完成定义之后，还调用`prometheus.MustRegister(c)`进行了注册。事实上，prometheus默认提供了一个Default Registry，`prometheus.MustRegister`会将Collector直接注册到Default Registry中。如果我们直接使用了`promhttp.Handler()`来处理`/metrics`路径的请求，它会直接将Default Registry和Handler相关联并且向Default Registry注册Golang Collector和Process Collector。所以，假设我们不需要这些自动注入的监控指标，只要构造自己的Handler就可以。

当然，Registry和Collector也都是能自定义的，特别在编写Exporter的时候，我们往往会将所有的指标定义在一个Collector中，根据访问应用原生监控接口的结果对所需的指标进行填充并返回结果。基于上述对于Prometheus SDK的实现机制的理解，我们可以实现一个最简单的Exporter框架如下所示：

```golang
package main

import (
	"net/http"
	"math/rand"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

type Exporter struct {
	up	*prometheus.Desc
}

func NewExporter() *Exporter {
	namespace := "exporter"
	up := prometheus.NewDesc(prometheus.BuildFQName(namespace, "", "up"), "If scrape target is healthy", nil, nil)
	return &Exporter{
		up:	up,
	}
}

func (e *Exporter) Describe(ch chan<- *prometheus.Desc) {
	ch <- e.up
}

func (e *Exporter) Scrape() (up float64) {
	// Scrape raw monitoring data from target, may need to do some data format conversion here
	rand.Seed(time.Now().UnixNano())
	return float64(rand.Intn(2))
}

func (e *Exporter) Collect(ch chan<- prometheus.Metric) {
	up := e.Scrape()
	ch <- prometheus.MustNewConstMetric(e.up, prometheus.GaugeValue, up)
}

func main() {
	registry := prometheus.NewRegistry()

	exporter := NewExporter()

	registry.Register(exporter)

	http.Handle("/metrics", promhttp.HandlerFor(registry, promhttp.HandlerOpts{}))
	http.ListenAndServe(":8080", nil)
}
```

在这个Exporter的最简实现中，我们创建了新的Registry，手动对exporter这个Collector完成了注册并且基于这个Registry自己构建了一个Handler并且与`/metrics`相关联。在初始exporter的时候，我们仅仅需要调用`NewDesc()`方法填充需要监控的指标的描述信息。当用户访问`/metrics`路径时，经过完整的调用链，最后在进行Collect的时候，我们才会对应用的原生监控接口进行访问，获取监控数据。在真实的Exporter实现中，该步骤应该在`Scrape()`方法中完成。最后，根据返回的原生监控数据，利用`MustNewConstMetric()`构造出我们所需的Metric，返回给channel即可。访问该Exporter的`/metrics`得到的结果如下：

```bash
$ curl http://127.0.0.1:8080/metrics
# HELP exporter_up If scrape target is healthy
# TYPE exporter_up gauge
exporter_up 1
```

### 3. 总结

经过本文的分析，可以发现，利用Prometheus SDK将应用程序进行简单的二次开发，它就能被Prometheus有效地监控，从而享受整个Prometheus监控生态带来的便利。同时，Prometheus SDK也提供了多层次的抽象，通常情况下，高度封装的API就能快速地满足我们的需求。至于更多的定制化需求，Prometheus SDK也有很多底层的，更为灵活的API可供使用。

本文中的示例代码以及如何对应用进行Prometheus化二次开发和编写Exporter的详细规范，参见参考文献中的相关内容。

### 参考文献

* [Prometheus client_golang Source Code](https://github.com/prometheus/client_golang)
* [Prometheus Doc: Writing Exporters](https://prometheus.io/docs/instrumenting/writing_exporters/)
* [Prometheus Doc: Instrumentation](https://prometheus.io/docs/practices/instrumentation/)
* [示例代码](https://github.com/YaoZengzeng/practice/tree/master/promdemo)

