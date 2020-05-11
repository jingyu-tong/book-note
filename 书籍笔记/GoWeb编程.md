# GO Web编程

## 1.Go与Web应用

Web应用主要任务是：
* 通过HTTP协议，以HTTP请求报文形式获取客户端输入
* 对请求进行处理，并执行必要操作
* 生产HTML，并以HTTP响应形式返回客户端

为了完成这些任务，Web应用被分成了处理器(handler)和模板引擎(template engine)两部分
* 处理器  
    处理器负责接收和处理客户端的请求，然后调用模板引擎，由引擎生产HTML并把数据填充至要发给客户端的报文中
* 引擎模板
    生产HTML报文

## 2.接收请求

net/http中的Server、ServerMux、Handler/HandleFunc、ResponseWriter、Header、Request和Cookie提供了对服务器的支持

### 创建服务器

使用http.ListenAndServe，传入网络地址以及负责处理请求的handler作为参数即可。网络地址默认为80，handler为nil，则默认为DefaultServeMux

此外，可以通过http.Server实例来配置Web服务器，Server结构如下：
```go
type Server struct {
    Addr           string        // 监听的地址和端口
    Handler        Handler       // 所有请求需要调用的Handler（实际上这里说是ServeMux更确切）如果为空则设置为DefaultServeMux
    ReadTimeout    time.Duration // 读的最大Timeout时间
    WriteTimeout   time.Duration // 写的最大Timeout时间
    MaxHeaderBytes int           // 请求头的最大长度
    TLSConfig      *tls.Config   // 配置TLS
}
```

### 处理器和处理器函数

在go中，一个处理器，就是一个拥有ServeHTTP方法的类，该方法的两个参数为ResponseWriter接口和指向Request的指针，具体如下：
`ServeHTTP(http.ResponseWriter, *http.Request)`

因此，我们如果声明一个含有ServeHTTP方法的类，我们可以对Serve中的Handler进行替换默认的多路复用器(**默认的多路复用器同时也是处理器**)。

如果我们希望对于不同的URL请求返回不同的相应，那么我们就需要使用多路复用器，然后通过http.Handle将处理器绑定到多路复用器中`http.Handle(url, handler)`

由于声明一个handler比较麻烦，http包中也提供了直接利用handler function进行设置的方法`http.HandleFunc(url, handler function)`

这里主要是利用了go自带的HandlerFunction可以吧带有正确签名的f，转换为带f方法的Handler，简化了设置流程

### ServeMux和DefaultServeMux

* ServeMux  
    这是一个HTTP请求多路复用器，负责接收HTTP请求，并根据URL，将请求重定向到正确的处理器
* DefaultServeMux  
    是ServeMux的一个实例，包含net/http的都可以使用，是服务器的默认handler

服务器采用的是最小惊讶原则，以/hello/there为例，会先匹配/hello/there，然后匹配/hello/，最后是/

## 3.处理请求

### Request结构

Request结构表示了客户端发送的http请求报文，Request结构是经过语法分析之后的信息，主要由以下几部分

* URL字段
* Header字段
* Body字段
* Form字段、PostForm字段、MultipartForm字段
* URL结构  
    HTTP的url请求格式为scheme://[userinfo@]host/path[?query][#fragment]

    ```go
    type URL struct {
        Scheme   string
        Opaque   string
        User     *Userinfo
        Host     string
        Path     string
        RawQuery string
        Fragment string
    }
    ```

* 请求首部(header)  
    header也是HTTP中重要的组成部分。Request结构中就有Header结构，Header本质上是一个map（map[string][]string）。将http协议的header的key-value进行映射成一个map

    获取特定首部，可以通过Header[key]获取value，是一个slice，也可以通过Header的Get方法，获取一个字符串，value的每个元素间用逗号分割

* html表单  
    解析body可以读取客户端请求的数据。而这个数据是无论是键值对还是form-data数据，都比较原始。直接读取解析还是挺麻烦的。这些body数据通常也是表单提供。因此go提供处理这些表单数据的方法。

    为了获取表单数据，首先调用ParseForm/ParseMultipartForm进行语法分析，然后访问对应的Form字段、PostForm字段或MultipartForm字段

    * Form字段获取所有表单中的key-value，如果在表单和url两地都出现的key，slice中表单值在url前面
    * PostForm只访问表单中的key  
    * MultipartForm用来解析multipart/form-data格式的数据

    总结一下，读取urlencode的编码方式，只需要ParseForm即可，读取form-data编码需要使用ParseMultipartForm方法。如果参数中既有url，又有body，From和FromValue方法都能读取。而带Post前缀的方法，只能读取body的数据内容。其中MultipartForm的数据通过r.MultipartForm.Value访问得到。

### ResponseWriter

请求和响应是http的孪生兄弟，不仅它们的报文格式类似，相关的处理和构造也类似。go构造响应的结构是ResponseWriter接口。

```go
type ResponseWriter interface {
    Header() Header //返回表示Header的map，通过这个设置相应首部
    Write([]byte) (int, error) //写入到响应主体
    WriteHeader(int) //设置HTTP响应状态码
}
```
