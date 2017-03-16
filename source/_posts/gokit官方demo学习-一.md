---
title: gokit官方demo学习(一)
date: 2016-11-17 22:43:03
tags: 编程
---
## 示例一:stringsvc1
> 这个示例是一个非常简单的go kit 服务，实现了2个功能。
> 一个是字符串转大写，一个是字符串计算长度。

### 项目组成部分
#### 业务逻辑部分
首先定义了一个StringService的接口, 接口中定义了2个函数:
```go
type StringService interface {
       Uppercase(string) (string, error)
       Count(string) int
}
```
然后完成接口功能的实现:
```go
type stringService struct{}

func (stringService) Uppercase(s string) (string, error) {
	if s == "" {
		return "", ErrEmpty
	}
	return strings.ToUpper(s), nil
}

func (stringService) Count(s string) int {
	return len(s)
}
```
其实这个demo的业务逻辑就是上面这一点. 其他部分是go-kit框架需要实现的内容

#### 请求和响应
> 在 Go kit 中，主要的消息模式是 RPC。
> 接口（ interface ）的每一个方法都会被模型化为远程过程调用（RPC）。
> 对于每一个方法，我们都定义了请求和响应的结构体，捕获输入、输出各自的所有参数。

下面就是将这个示例中有道的两个响应和请求的结果体进行定义:
```go
type uppercaseRequest struct {
	S string `json:"s"`
}

type uppercaseResponse struct {
	V   string `json:"v"`
	Err string `json:"err,omitempty"` // errors don't define JSON marshaling
}

type countRequest struct {
	S string `json:"s"`
}

type countResponse struct {
	V int `json:"v"`
}
```
#### endpoint
> gokit通过endpoint实现很多功能
> 一个endpoint代表一个rpc,也就是我们服务接口中的一个函数.
> 我们需要写一个adapter(适配器)把我们的方法转换成endpoint

下面是两个endpoint,对应的就是我们之前业务代码中实现的两个函数:
```go
func makeUppercaseEndpoint(svc StringService) endpoint.Endpoint {
	return func(ctx context.Context, request interface{}) (interface{}, error) {
		req := request.(uppercaseRequest)
		v, err := svc.Uppercase(req.S)
		if err != nil {
			return uppercaseResponse{v, err.Error()}, nil
		}
		return uppercaseResponse{v, ""}, nil
	}
}

func makeCountEndpoint(svc StringService) endpoint.Endpoint {
	return func(ctx context.Context, request interface{}) (interface{}, error) {
		req := request.(countRequest)
		v := svc.Count(req.S)
		return countResponse{v}, nil
	}
}
```
#### 传输(Transports)
> 我们需要把服务暴露给外部访问。
> gokit支持多种的传输方式。
> 这个demo中使用的是基于http的json格式, 在gokit的transport/http包中

下面是代码示例: 
```go
func main() {
	ctx := context.Background()
	svc := stringService{}

	uppercaseHandler := httptransport.NewServer(
		ctx,
		makeUppercaseEndpoint(svc),
		decodeUppercaseRequest,
		encodeResponse,
	)

	countHandler := httptransport.NewServer(
		ctx,
		makeCountEndpoint(svc),
		decodeCountRequest,
		encodeResponse,
	)

	http.Handle("/uppercase", uppercaseHandler)
	http.Handle("/count", countHandler)
	log.Fatal(http.ListenAndServe(":8080", nil))
}

func decodeUppercaseRequest(_ context.Context, r *http.Request) (interface{}, error) {
	var request uppercaseRequest
	if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
		return nil, err
	}
	return request, nil
}

func decodeCountRequest(_ context.Context, r *http.Request) (interface{}, error) {
	var request countRequest
	if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
		return nil, err
	}
	return request, nil
}

func encodeResponse(_ context.Context, w http.ResponseWriter, response interface{}) error {
	return json.NewEncoder(w).Encode(response)
}
```

[点击查看官方原文](http://gokit.io/examples/stringsvc.html#first-principles)