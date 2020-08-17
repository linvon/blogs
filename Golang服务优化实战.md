---
title: "Golang服务优化实战"
date: "2020-08-17"
toc: "true"
---

最近几天给推荐业务做了一次流程优化，请求耗时的 P95 分位降低了一半，记录一下排查与优化的历程，也方便有需要的朋友参考。


## 背景

熟悉推荐的同学应该知道，推荐中的历史池、召回 Recall 和 Ranking 是比较重要的部分。同时各个模型有轻有重，数据量有大有小，并且模型之间可能还存在依赖关系。而我们的推荐同学重点负责算法和模型的研发，在工程上我们便接手来做优化。

### 串行改并行

首先想到的优化思路便是串改并，我们的服务是使用 Go 来搭建的，并发跑模型是很容易的。但改并发并不能一蹴而就，我们提到过模型之前很多是存在依赖关系的，有些模型可能会需要上一个模型的结果作为参数。所以在处理模型的串改并上，我们做了下面的事情：

- 理清模型时序依赖，使用配置文件控制哪些模型可以并发运行，哪些不能
- 在模型队列周期和每个模型运行周期做好时间 Metrics，方便直观的观察运行效率
- 重构同时走读流程代码，查看其它可异步运行的流程，拆离主流程

最终通过 Metrics 日志统计，发现模型队列平均可以优化近一半的耗时

### 使用 Jaeger 追踪请求流程

[Jaeger](https://www.jaegertracing.io/) 是一个优秀的分布式微服务事务追踪软件。即使一个请求会经过不同的微服务来完成业务，它仍然可以追踪一个完整的会话上下文。

<img src="/Users/Zuiyou/openSourceCode/blog-images/optimize/jaeger view.png" alt="39d66ffc2a28e0f0f810cda217f7add0.png" style="zoom:50%;" />

点击一个请求会话，可以进入到它的详情页

<img src="/Users/Zuiyou/openSourceCode/blog-images/optimize/jaeger detail view.png" alt="aa77e19aae4dc70cfe60134c9df2e3ed.png" style="zoom: 50%;" />

可以看到在整个请求的分阶段耗时都展现了出来，我们可以清楚的了解到整个请求过程中时间都花费在了什么地方。

对于 Go 服务来讲，应用 Jaeger 也非常简单，可以参照 [Github](https://github.com/yurishkuro/opentracing-tutorial/tree/master/go/lesson02) 上的开源实例来实现，也可以直接参阅 [官方的文档](https://www.jaegertracing.io/docs/1.16/)

以 opentracing 为例，在使用时可以设定 Operation 和 Tags，方便给这一个事务做归类，同时在 Jaeger 中我们可以直接对想看的类型做筛选

``` go
span, _ := opentracing.StartSpanFromContext(ctx, "RecallStrategies", opentracing.Tag{Key: string(ext.SpanKind), Value: "Recall"})
defer span.Finish()
```

<img src="/Users/Zuiyou/openSourceCode/blog-images/optimize/Jaeger category.jpg" alt="cffc3627459ff9044a4dc8328bf21691.jpeg" style="zoom:50%;" />


既然 Jaeger 已经可以将耗时展现的如此细致了，为什么还需要其他优化策略呢？
很遗憾，Jaeger 并不是完全稳定可靠的，由于网络或其他一些原因，[流程中的 Span 可能会丢掉](https://medium.com/jaegertracing/where-did-all-my-spans-go-a-guide-to-diagnosing-dropped-spans-in-jaeger-10d9697f8182)，导致一整个会话并不能统计完全。因此我们需要一些其他稳定可靠的手段来完善耗时统计。

### Prometheus & Grafana

如果做过服务监控的同学可能不会对这两个组件陌生。

- [Prometheus](https://prometheus.io/) 是一个开源的监控与报警系统，它本质上是一个时序数据库，会从微服务上发现并拉取含有时间序列的数据指标并做分析统计存储。上手可以参考 [官方文档](https://prometheus.io/docs/introduction/overview/)，同时也可以参考 [prometheus-book](https://yunlzheng.gitbook.io/prometheus-book/)，都是不错的教程
- [Grafana](https://grafana.com/grafana/) 是一个数据可视化仪表盘工具，用以将你收集存储的数据指标展示出来

通常情况下，Prometheus + Grafana 是很有效的搭配，在上面推荐的教程中也有提到这两种组件的组合使用方式。实际上就是在 Grafana 中利用 promQL 向 Prometheus 查询时序数据。这里再推荐一篇 [新手教程](https://kalasearch.cn/blog/grafana-with-prometheus-tutorial/)，这两个组件的搭建我们在本文中不再赘述

使用这种组合可以监控很多种我们需要的数据，比方说接口的 QPS，系统的 CPU、内存等性能指标，请求的耗时、延时等。在本次优化中，我们需要使用到的便是 Prometheus 的统计直方图功能，将我们需要统计的函数耗时记录到 Prometheus 中，利用其自身的直方图功能构建出每个函数的耗时分布直方图。

对于 Prometheus 的使用我们不再赘述，有需要的同学可以直接参考官方的 API 文档，这里以 Go 为例：

```go
import (
	"github.com/prometheus/client_golang/prometheus"
)

var elapsedHis = prometheus.NewHistogramVec(prometheus.HistogramOpts{
	Namespace: "rec",
	Name:      "rec_func_elapsed",
	Help:      "Rec Function Elapsed Time",
	Buckets:   append(prometheus.LinearBuckets(0, 10, 10), prometheus.ExponentialBuckets(100, 2, 4)...),
}, []string{"func"})

func InitProm() {
	prometheus.MustRegister(elapsedHis)
 	http.Handle("/metrics", promhttp.Handler())
	log.Println("start serving prometheus")
}

func AddFuncElapsed(funcName string, t time.Time) {
	elapsed := int64(time.Since(t)) / 1e6
	elapsedHis.WithLabelValues(funcName).Observe(float64(elapsed))
}
```

首先建立一个直方桶，桶的分槽范围我们采用的是线性增长（LinearBuckets）加指数增长（ExponentialBuckets），最终桶分布是 0 10 20 30 40 ... 100 200 400 800 这样。之后需要将建立好的桶注册，并启用 HTTP 服务，将 metrics 接口暴露给 Prometheus 用以抓取数据。

之后便可以在需要记录的函数内添加函数回调，用以统计该函数执行所耗费的时间

```go
func A() {
	defer base.AddFuncElapsed("A", time.Now())  
  
  ...
}
```

完善好服务后，访问服务的 ip:port/metrics 应该可以看到我们服务所暴露出来的指标内容。Prometheus 会抓取这些指标，这时候我们便可以在 Grafana 中绘制我们所需要的分布图。在这里我们所需要的是根据直方桶来绘制函数延时 95 分位的图，即 95% 的函数耗时均低于该所得值，用以观察绝大部分函数的耗时，个例不再做考虑。

建图后可以看到类似的效果

<img src="/Users/Zuiyou/openSourceCode/blog-images/optimize/gbef.png" alt="279313ebb2615b02cfd8801e9d2a0e20.png" style="zoom:50%;" />

图中我们可以看到所有函数的 95 分位的耗时时长，值得注意的是 callRecall 函数负责了对于 Recall 的调用，而在 Recall 中的耗时实际上比 callRecall 低了很多，甚至不到一半。然而走读代码并没有发现有其他的耗时操作，这就成为了一个比较困惑的点。为了排查除去 Recall 自身还有什么其他的耗时操作，我们就需要另一个工具了

### Go tool pprof

熟悉 Go 优化的同学应该对这个工具非常熟悉，这是 Go 自带的性能分析工具，在 Go 程序或服务上引入`"net/http/pprof"`包，程序便会自动构建 Go 运行时分析数据。

通过这个工具我们可以排查 Go 程序运行时的 CPU 分布、内存的分配情况等等，这里我们需要的是程序的 CPU 占用，来找找是否有某些计算密集操作拖慢了整个流程。

我们在程序中加入包引用，通过本地工具获取运行时数据：

```shell
go tool pprof -http=:8080 http://ip:port/debug/pprof/profile
```

该工具会将运行时数据生成可视化图表 svg 文件，使用浏览器打开后找到 Flame 可以看到如下内容

<img src="/Users/Zuiyou/openSourceCode/blog-images/optimize/cpubef.png" alt="e2d51db28b3b04b61dc52a34360711fe.png" style="zoom:50%;" />

可以看到在 callRecall 过程中对于 json 数据的 Unmarshal 占用了很长时间的 CPU，可以推测因为 Recall 获取到的数据量非常大，接口返回数据后对于数据的格式化占用了非常大的时间。实际上 Go 标准库的 JSON 库性能并不是很优秀，有经验的 Gopher 们往往会采用第三方 JSON 来加快序列化、反序列化过程。

而在本例中，对于结构庞大且复杂的 Struct 数据，使用 jsoniter 是一个非常好的选择。json-iterator 使用 [modern-go/reflect2](https://github.com/modern-go/reflect2) 来优化反射性能，因此在 Struct 的 Unmarshal 上性能非常好，在引入 jsoniter 后，我们再次查看 CPU 分布如下

<img src="/Users/Zuiyou/openSourceCode/blog-images/optimize/cpuaft.png" alt="193a4ac0e557327e13b6cbaa7da75e52.png" style="zoom:50%;" />

可见使用 jsoniter 的确提升了数倍性能

## 收工

这个时候我们再去查看 Grafana 上整个函数的耗时，发现整体调用耗时下降很多，已经在可接受范围了

<img src="/Users/Zuiyou/openSourceCode/blog-images/optimize/gafter.jpg" alt="11e97065638d6b904a13d2c3ec542ac9.jpeg" style="zoom:50%;" />

再补一张优化上线后各函数耗时的变化图，可见部分函数耗时下降明显

<img src="/Users/Zuiyou/openSourceCode/blog-images/optimize/gdown.jpg" alt="b9b27de046605fc2f9af6808a927f8eb.jpeg" style="zoom:50%;" />

最后看一下整个推荐请求的耗时在优化前后的对比，一整天下来基本所有耗时都优化了近一半的时间
优化前：
<img src="/Users/Zuiyou/openSourceCode/blog-images/optimize/recbef95.jpg" alt="8167536fd3f037eff841a7b9c5abc8a2.jpeg" style="zoom:50%;" />
优化后：
<img src="/Users/Zuiyou/openSourceCode/blog-images/optimize/recaft95.jpg" alt="8d53ed3f71a32dee256199426d6f4ac4.jpeg" style="zoom:50%;" />

