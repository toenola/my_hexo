---
title: gokit官方demo学习(二)
date: 2016-11-17 22:44:01
tags: 编程
---
## 示例二:stringsvc2

这个示例主要是用中间件来实现日志和跟踪请求的

### 传输日志 
> 任何需要日志记录的组件都需要将 logger 作为依赖，就像数据库连接一样。
> 因此，我们在 main 函数中构造 logger，将它传到需要使用它的组件中。
> 我们从来不去使用一个全局的 logger。
> 我们可以直接将 logger 传入到 stringService 的实现代码中。
> 但是，还有一个更好的方式。使用 中间件 (middleware) ,也常常被称为装饰者(decorator)。
> middleware 是一个函数，它接收一个 endpoint 作为参数，并且返回一个 endpoint。

定义一个中间件变量:
```go
type Middleware func(Endpoint) Endpoint
```
补充:在go中函数也是一种变量，我们通过type定义这种变量的类型。拥有相同参数和相同返回值的函数属于同一种类型。

构造一个日志的中间件函数:
```go
func loggingMiddleware(logger log.Logger) Middleware {
	return func(next endpoint.Endpoint) endpoint.Endpoint {
		return func(ctx context.Context, request interface{}) (interface{}, error) {
			logger.Log("msg", "calling endpoint")
			defer logger.Log("msg", "called endpoint")
			return next(ctx, request)
		}
	}
}
```
并且将他写到我们的处理函数中:
```go
var uppercase endpoint.Endpoint
var countcase endpoint.Endpoint
uppercase = e.MakeUppercaseEndpoint(svc)
uppercase = middlewares.LoggingMiddleware(log.NewContext(logger).With("method", "uppercase"))(uppercase)
countcase = e.MakeCountEndpoint(svc)
countcase = middlewares.LoggingMiddleware(log.NewContext(logger).With("method", "count"))(countcase)
uppercaseHandler := httptransport.NewServer(
		ctx,
		uppercase,
		decodeUppercaseRequest,
		encodeResponse,
	)
countHandler := httptransport.NewServer(
		ctx,
		countcase,
		decodeCountRequest,
		encodeResponse,
	)
```
>事实上，这个技术非常有用，不仅仅是在日志方面，很多gokit的模块都被作为中间件实现。

### 应用日志
> 那我们如何在我们的应用中级日志呢？比如一些参数需要被带入。
> 事实上我们能够为我们的服务定义一个中间件，能够获得很好的组合效果。
> 示例中StringService被定义为一个接口，我们只需要创建一个新的类型来包装它，执行额外的日志任务。

```go
type loggingMiddleware struct {
	logger log.Logger
	next   StringService
}

func (mw loggingMiddleware) Uppercase(s string) (output string, err error) {
	defer func(begin time.Time) {
		_ = mw.logger.Log(
			"method", "uppercase",
			"input", s,
			"output", output,
			"err", err,
			"took", time.Since(begin),
		)
	}(time.Now())

	output, err = mw.next.Uppercase(s)
	return
}

func (mw loggingMiddleware) Count(s string) (n int) {
	defer func(begin time.Time) {
		_ = mw.logger.Log(
			"method", "count",
			"input", s,
			"n", n,
			"took", time.Since(begin),
		)
	}(time.Now())

	n = mw.next.Count(s)
	return
}

```
这里是在我们的业务方法之上包装了一层.next指向的就是我们真实的业务函数
> 然后在入口处调用
```go
logger := log.NewLogfmtLogger(os.Stderr)

var svc StringService
svc = stringService{}
svc = loggingMiddleware{logger, svc}
```
> 使用endpoint中间件作为传输环节，比如回路断开和频率限制。
> 在业务逻辑层领域使用服务中间件，比如日志和instrumentation(仪器?)

### 应用instrumentation
> 在go kit中instrumentation意味着用包指标来记录你的服务的运行状态，计算执行任务的次数，请求完成后记录消耗时间，以及跟踪所有正在执行的操作的数量，都是instrumentation
> 我们可以使用相同的中间价模式就像我们日志使用的那样

```go
type instrumentingMiddleware struct {
	requestCount   metrics.Counter
	requestLatency metrics.Histogram
	countResult    metrics.Histogram
	next           StringService
}

func (mw instrumentingMiddleware) Uppercase(s string) (output string, err error) {
	defer func(begin time.Time) {
		lvs := []string{"method", "uppercase", "error", fmt.Sprint(err != nil)}
		mw.requestCount.With(lvs...).Add(1)
		mw.requestLatency.With(lvs...).Observe(time.Since(begin).Seconds())
	}(time.Now())

	output, err = mw.next.Uppercase(s)
	return
}

func (mw instrumentingMiddleware) Count(s string) (n int) {
	defer func(begin time.Time) {
		lvs := []string{"method", "count", "error", "false"}
		mw.requestCount.With(lvs...).Add(1)
		mw.requestLatency.With(lvs...).Observe(time.Since(begin).Seconds())
		mw.countResult.Observe(float64(n))
	}(time.Now())

	n = mw.next.Count(s)
	return
}
```
> 然后在我们的服务中加入
```go
fieldKeys := []string{"method", "error"}
requestCount := kitprometheus.NewCounterFrom(stdprometheus.CounterOpts{
	Namespace: "my_group",
	Subsystem: "string_service",
	Name:      "request_count",
	Help:      "Number of requests received.",
}, fieldKeys)
requestLatency := kitprometheus.NewSummaryFrom(stdprometheus.SummaryOpts{
	Namespace: "my_group",
	Subsystem: "string_service",
	Name:      "request_latency_microseconds",
	Help:      "Total duration of requests in microseconds.",
}, fieldKeys)
countResult := kitprometheus.NewSummaryFrom(stdprometheus.SummaryOpts{
	Namespace: "my_group",
	Subsystem: "string_service",
	Name:      "count_result",
	Help:      "The result of each count method.",
}, []string{}) // no fields here

var svc StringService
svc = stringService{}
svc = loggingMiddleware{logger, svc}
svc = instrumentingMiddleware{requestCount, requestLatency, countResult, svc}
```
> 目前完整的服务就是stringsvc2

运行结果如下: 
```
eno-macdeMacBook-Pro:kit eno$ curl -XPOST -d'{"s":"hello world~~!!!"}' 127.0.0.1:8080/uppercase
{"v":"HELLO WORLD~~!!!"}
eno-macdeMacBook-Pro:kit eno$ curl -XPOST -d'{"s":"hello world~~!!!"}' 127.0.0.1:8080/count
{"v":16}
```
控制台打印的日志:
```
method=uppercase input="hello world~~!!!" output="HELLO WORLD~~!!!" err=null took=2.517µs
method=count input="hello world~~!!!" n=16 took=735ns
```

[点击查看官方原文](http://gokit.io/examples/stringsvc.html#first-principles)