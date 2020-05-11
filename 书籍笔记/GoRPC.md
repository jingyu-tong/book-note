# RPC和Protobuf
RPC是Remote Procedure Call的缩写，简单来说就是调用一个远处的函数，可能是同一个机器另一个进程的函数，甚至是不同机器，不同语言的调用。

因为RPC设计复杂的通信，而protobuf由于支持多种语言，因此非常合适作为RPC的语言

# RPC入门

## RPC demo

```go
// 服务端
type HelloService struct{}

func (h *HelloService) Hello(request string, reply *string) error {
	*reply = "hello " + request
	return nil
}

func main() {
	//注册远程调用
	rpc.RegisterName("HelloService", &HelloService{})
	//Listen & Accept
	l, err := net.Listen("tcp", ":9090")
	if err != nil {
		return
	}
	conn, err := l.Accept()
	//执行对应调用
	rpc.ServeConn(conn) //根据conn消息进行调用
}

// 客户端
func main() {
	client, err := rpc.Dial("tcp", ":9090") //连接服务端
	if err != nil {
		return
	}

	//发起RPC
	var reply string
	client.Call("HelloService.Hello", "jingyu", &reply)
	fmt.Println(reply)
}
```
上面的demo简单的利用go语言实现了rpc，但是，rpc标准库默认采用了go语言的编码，这使得跨语言调用很不容易，我们可以采用序列化的方式解决这个问题

## 序列化

### JSON序列化

通过JSON格式序列化，可以实现跨语言的RPC，并且官方自带了net/rpc/jsonrpc扩展，将上述server段调用`ServeConn`改为`rpc.ServeCodec(jsonrpc.NewServeCodec(conn))`可以改为JSON编码的服务端。

### protobuf

虽然利用JSON可以方便的实现序列化，从而实现跨语言通信，但是JSON方便的编码方式，也带来了效率不高的问题。幸运的是，我可以采用google提供的protobuf序列化方式，来实现高效的序列化。



