- [Gin](#gin)
- [RESTful风格](#restful%e9%a3%8e%e6%a0%bc)
- [Gin渲染](#gin%e6%b8%b2%e6%9f%93)
  - [http/template](#httptemplate)
  - [Gin模板渲染](#gin%e6%a8%a1%e6%9d%bf%e6%b8%b2%e6%9f%93)
    - [HTML渲染](#html%e6%b8%b2%e6%9f%93)
    - [定义模板函数](#%e5%ae%9a%e4%b9%89%e6%a8%a1%e6%9d%bf%e5%87%bd%e6%95%b0)
  - [JSON渲染](#json%e6%b8%b2%e6%9f%93)
- [gin获取参数](#gin%e8%8e%b7%e5%8f%96%e5%8f%82%e6%95%b0)
  - [获取querystrings的参数(GET请求)](#%e8%8e%b7%e5%8f%96querystrings%e7%9a%84%e5%8f%82%e6%95%b0get%e8%af%b7%e6%b1%82)
  - [获取FORM的参数](#%e8%8e%b7%e5%8f%96form%e7%9a%84%e5%8f%82%e6%95%b0)
  - [参数绑定](#%e5%8f%82%e6%95%b0%e7%bb%91%e5%ae%9a)
- [重定向](#%e9%87%8d%e5%ae%9a%e5%90%91)
  - [http重定向](#http%e9%87%8d%e5%ae%9a%e5%90%91)
  - [路由重定向](#%e8%b7%af%e7%94%b1%e9%87%8d%e5%ae%9a%e5%90%91)
- [Gin中间件](#gin%e4%b8%ad%e9%97%b4%e4%bb%b6)
  - [为某个路由注册中间件](#%e4%b8%ba%e6%9f%90%e4%b8%aa%e8%b7%af%e7%94%b1%e6%b3%a8%e5%86%8c%e4%b8%ad%e9%97%b4%e4%bb%b6)
  - [全局注册](#%e5%85%a8%e5%b1%80%e6%b3%a8%e5%86%8c)
  - [为路由组注册](#%e4%b8%ba%e8%b7%af%e7%94%b1%e7%bb%84%e6%b3%a8%e5%86%8c)
  - [gin中间件常用方式和注意事项](#gin%e4%b8%ad%e9%97%b4%e4%bb%b6%e5%b8%b8%e7%94%a8%e6%96%b9%e5%bc%8f%e5%92%8c%e6%b3%a8%e6%84%8f%e4%ba%8b%e9%a1%b9)
    - [常用方式](#%e5%b8%b8%e7%94%a8%e6%96%b9%e5%bc%8f)
    - [注意](#%e6%b3%a8%e6%84%8f)
# Gin

Gin是一个用Go语言编写的web框架，这里只是主要的总结，具体可见[Gin中文文档](https://gin-gonic.com/zh-cn/docs/)

# RESTful风格

Gin采用的是RESTful的代码风格，REST是Representational State Transfer的简称，含义是表现层状态转换。

这个风格简单来说，就是用http的get/post/put/delete来代替在url中传递状态转换的信息

以下代码展示了对于书籍的四个操作
```go
func main() {
	r := gin.Default() //创建一个默认的路由引擎
	r.GET("/book", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "GET",
		})
	})

	r.POST("/book", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "POST",
		})
	})

	r.PUT("/book", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "PUT",
		})
	})

	r.DELETE("/book", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "DELETE",
		})
	})
}
```

# Gin渲染

在一些前后端不分离的Web架构中，通常需要在后端将一些数据渲染到HTML文档中，从而实现动态的网页（网页的布局和样式大致一样，但展示的内容并不一样）效果。

顺带了解下前后端分离  

在互联网架构中，名词解释：  
Web服务器：一般指像nginx，apache这类的服务器，他们一般只能解析静态资源。  
应用服务器：一般指像tomcat，jetty，resin这类的服务器可以解析动态资源也可以解析静态资源，但解析静态资源的能力没有web服务器好。  
一般都是只有web服务器才能被外网访问，应用服务器只能内网访问。

前后端分离指的是，前端html页面通过ajax调用后端的restuful api接口并使用json数据进行交互。这样，后端服务传送的数据量将大大降低，也不再需要去做渲染方面的事情。

## http/template

go自带的html/template实现了数据驱动的模板，它们的作用机制可以简单归纳如下：  
* 模板文件通常定义为.tmpl和.tpl为后缀（也可以使用其他的后缀），必须使用UTF8编码。
* 模板文件中使用{{和}}包裹和标识需要传入的数据。
* 传给模板这样的数据就可以通过点号（.）来访问，如果数据是复杂类型的数据，可以通过{ { .FieldName }}来访问它的字段。
* 除{{和}}包裹的内容外，其他内容均不做修改原样输出。

模板的使用有三个步骤
* 编写模板，也就是需要被部分替换的文件
* 解析模板文件，以下为常用方法
    ```go
    func (t *Template) Parse(src string) (*Template, error)  
    func ParseFiles(filenames ...string) (*Template, error)
    func ParseGlob(pattern string) (*Template, error)
    ```
* 最后是模板渲染，简单将就是用数据去填充模板
    ```go
    func (t *Template) Execute(wr io.Writer, data interface{}) error
    func (t *Template) ExecuteTemplate(wr io.Writer, name string, data interface{}) error
    ```

传递多个变量时，可以把他们放在一个map中，再进行渲染



## Gin模板渲染

### HTML渲染

跟使用标准库的步骤类似，也是先解析模板，然后在进行渲染，主要有两个函数进行加载和解析`LoadHTMLFiles`以及`LoadHTMLGlob`。然后通过`HTML`方法渲染后就可以传回给gin.Context

简单的demo如下：
```go
func main() {
	// 创建一个默认的路由引擎
	r := gin.Default()

	// 解析模板文件
	r.LoadHTMLFiles("./templates/index.tmpl")

	// GET：请求方式；/hello：请求的路径
	// 当客户端以GET方法请求/hello路径时，会执行后面的匿名函数
	r.GET("/hello", func(c *gin.Context) {
		//返回一个模板文件
		c.HTML(200, "index.tmpl", gin.H{
			"title": "jingyu",
		})
	})
	// 启动HTTP服务，默认在0.0.0.0:8080启动服务
	r.Run()
}
```

### 定义模板函数

gin框架同样支持自定义的函数，通过`SetFunMap`实现，如下：
```go
router := gin.Default()
router.SetFuncMap(template.FuncMap{
    "safe": func(str string) template.HTML{
        return template.HTML(str)
    },
})
```

## JSON渲染

在前后端分离的项目中，前后端通常通过JSON格式进行传递信息，主要有两种方式，具体如下：
```go
// method 1.传递一个map[string]interface{}，也就是gin.H格式的数据
r.GET("/json", func(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{
        "message": "Hello World!",
    })
})

// method 2.传递一个struct
r.GET("/json2", func(c *gin.Context) {
    type info struct {
        Message string `json:"message"`
    }
    i := info{"Hello World"}
    c.JSON(http.StatusOK, i)
})
```

# gin获取参数

## 获取querystrings的参数(GET请求)

query string的参数指的是在url后面，以`？`开头的key-value参数，可以对不同的参数返回不同的网页，解析方法如下
```go
// 解析请求中query的值
name := c.Query("query")

//设置默认值的解析
name := c.Default("query", "defaultname")

//ok-idiom
name, ok := c.GetQuery("query")
```

## 获取FORM的参数

在GET中传递参数受限与url长度限制，另一种获取参数的方式是在from表单中传参，具体常用的有以下几种
```go
// 解析post中form表单的值
name := c.PostForm("username")

// 带默认值的
name := c.DefaultPostForm("username", "default")

// ok-idiom
name, ok := c.GetPostForm("username")
```

## 参数绑定

前面所述一个个解析未免过于繁琐，gin提供了结构体绑定参数的方法，首先需要声明结构体，并且对每个字段声明一个form的tag，然后利用shouldbind进行绑定
```go
type Login struct {
	User     string `form:"user" json:"user" binding:"required"`
	Password string `form:"password" json:"password" binding:"required"`
}
var login Login
c.ShouldBind(&login)
```

需要注意，shouldBind可以处理query类型、json、以及form表单的格式

# 重定向

## http重定向

`c.Redirect(http.StatusMovedPermanently, "http://www.google.com/")`

## 路由重定向

```go
r.GET("/test", func(c *gin.Context) {
    //修改URI，然后继续处理，从而定向
    c.Request.URL.Path = "/test2" 
    r.HandleContext(c)
})
r.GET("/test2", func(c *gin.Context) {
    c.JSON(200, gin.H{"hello": "world"})
})
```

# Gin中间件

gin允许开发者在处理请求的过程中，加入自己的钩子函数(称为中间件)，用来处理一些公共的业务逻辑，比如登录认证，耗时统计等。

## 为某个路由注册中间件

为某个路由注册的方式很简单，只要在正式处理handler之前传入一个handlerFunction即可
```go
// 给/test2路由单独注册中间件（可注册多个）
r.GET("/test2", StatCost(), func(c *gin.Context) {
    name := c.MustGet("name").(string) // 从上下文取值
    log.Println(name)
    c.JSON(http.StatusOK, gin.H{
        "message": "Hello world!",
    })
})
```

## 全局注册

如果想对所有的请求进行验证，就要使用全局注册的方式
```go
r := gin.Default()
r.Use(StatCost) //注册全局中间件
```

## 为路由组注册

为路由组注册中间件有以下两种写法。
```go
// 1.声明路由组的时候传入
shopGroup := r.Group("/shop", StatCost())
{
    shopGroup.GET("/index", func(c *gin.Context) {...})
    ...
}

// 2.利用路由组的use方法
shopGroup := r.Group("/shop")
shopGroup.Use(StatCost())
{
    shopGroup.GET("/index", func(c *gin.Context) {...})
    ...
}
```

## gin中间件常用方式和注意事项

### 常用方式

* 通常会采用闭包的方式处理中间件，方便补货一些可能需要处理的变量，状态等
* c.Set()和c.Get()可以在同一路由不同handlerFunc之间传递参数
* c.Next()可以调用下一个handler
* c.Abort()不执行后续的handler
* 

```go
// StatCost 是一个统计耗时请求耗时的中间件
func StatCost(p ...parameter) gin.HandlerFunc {
    //some operations
    //...
	return func(c *gin.Context) {
		start := time.Now()
		c.Set("name", "小王子") // 可以通过c.Set在请求上下文中设置值，后续的处理函数能够取到该值
		// 调用该请求的剩余处理程序
		c.Next()
		// 不调用该请求的剩余处理程序
		// c.Abort()
		// 计算耗时
		cost := time.Since(start)
		log.Println(cost)
	}
}
```

### 注意

* 当在中间件或handler中启动新的goroutine时，不能使用原始的上下文（c *gin.Context），必须使用其只读副本（c.Copy()）
* gin.Default()默认使用了Logger和Recovery中间件，其中：
    * Logger中间件将日志写入gin.DefaultWriter，即使配置了GIN_MODE=release。
    * Recovery中间件会recover任何panic。如果有panic的话，会写入500响应码。

    如果不想使用上面两个默认的中间件，可以使用gin.New()新建一个没有任何默认中间件的路由。


