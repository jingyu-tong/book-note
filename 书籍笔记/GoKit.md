# Go kit 
是用于在 Go 中构建微服务的编程工具包。

kit架构由三部分构成
* Transport layer：通信层，这里可以用各种不同的通信方式，如 HTTP REST 接口或者 gRPC 接口（这是个很大的优点，方便切换成任何通信协议）
* Endpoint layer：终端层，类似于 Controller，里面主要实现各种接口的 handler，负责 req／resp 格式的转换（同时也是被吐槽繁杂的原因所在）
* Service layer：服务层，也就是实现业务逻辑的地方；

# Go kit demo

## service层

首先是service层，这部分主要实现业务逻辑，这个demo主要提供了三个业务，接口如下，实现部分略
```go
type Service interface {
	Status(ctx context.Context) (string, error)
	Get(ctx context.Context) (string, error)
	Validate(ctx context.Context, date string) (bool, error)
}
```

## Transport层

transport层顾名思义，存放的是传输相关的代码，我们以上文的Get服务为例，transport层需要提供get服务req/resp的编解码功能
```go
//采用http传输json格式数据

//解码很简单，因为demo的get服务实际上什么req信息都不需要
//只是返回时间
func decodeGetRequest(ctx context.Context, r *http.Request) (interface{}, error) {
	var req getRequest
	return req, nil
}

//编码，这里所有方法都一样，编码成json格式即可
func encodeResponse(ctx context.Context, w http.ResponseWriter, response interface{}) error {
	return json.NewEncoder(w).Encode(response)
}
```

## EndPoint层

这部分相当于是承上启下的作用，讲传输以及服务融合在一起，共同实现一个完整的程序

这层的主要功能由EndPoint函数类型提供，声明如下：
```go
type Endpoint func(ctx context.Context, request interface{}) (response interface{}, err error)
```
可以看出，这个函数对象传入请求，然后返回响应。

同样以get方法为例，我们在这层实现了一个创建get服务对应的EndPoint的方法`MakeGetEndpoint`
```go
// 创建节点的流程如下：
// 转换为getRequest格式(这里不需要，直接_忽略掉了)
// 调用Service提供的服务
// 返回getRepose格式的响应
func MakeGetEndpoint(srv Service) endpoint.Endpoint {
	return func(ctx context.Context, request interface{}) (interface{}, error) {
		_ = request.(getRequest) 
		d, err := srv.Get(ctx)
		if err != nil {
			return getResponse{d, err.Error()}, nil
		}
		return getResponse{d, ""}, nil
	}
}
```

## 构建服务

最后，利用kit提供的http里的NewServer方法，创建一个新的服务，即可把编解码部分融入架构，完成服务。
```go
r.Methods("GET").Path("/get").Handler(httptransport.NewServer(
    endpoints.GetEndpoint,
    decodeGetRequest,
    encodeResponse,
))
```



