# NSQ

首先说说为什么要用消息队列呢，应用不采用消息队列也是可以在各个系统中进行通信的，用消息队列主要有几个好处：
* 异步处理及解耦  
    采用消息队列，可以把业务流程中的非紧要部分进行解耦，后续新增业务可以通过订阅消息队列进行扩展。此外，进行拆分后达到了异步处理的效果，缩短了服务的时间
* 流量削峰()
    类似秒杀（大秒）等场景下，某一时间可能会产生大量的请求，使用消息队列能够为后端处理请求提供一定的缓冲区，保证后端服务的稳定性。

# NSQ架构

![NSQ架构d](https://www.liwenzhou.com/images/Go/nsq/nsq4.png)

nsq架构如图，主要由三部分构成，nsqlookupd,nsqd以及nsqadmin
* nsqd是一个守护进程，它接收、排队并向客户端发送消息，在非集群模式下，producer和consumer可以直接通过nsqd相连
* nsqlookupd同样是守护进程，维护所有nsqd状态，它能为消费者查找特定topic下的nsqd提供了运行时的自动发现服务。
* nsqadmin是一个实时监控集群状态、执行各种管理任务的Web管理平台。 

topic和channel：  
用户通过topic以及topic下的channel进行收发消息。topic和channel不是预先配置的。topic在首次使用时创建，方法是将其发布到指定topic，或者订阅指定topic上的channel。channel是通过订阅指定的channel在第一次使用时创建的。  
需要注意，每个channel都会得到一份topic消息的拷贝，但是订阅了统一channel的多个consumer只有一个会随机的获取到消息

# 生产者

在我们开启并配置了nsq的三个进程后，就可以利用go进行消息队列的收发了，如下是典型的生产者代码，具体见注释
```go
// NSQ Producer Demo

var producer *nsq.Producer //生产者实例

// 初始化生产者
func initProducer(str string) (err error) {
	config := nsq.NewConfig() //返回一个默认设置，用于设置producer
	producer, err = nsq.NewProducer(str, config) //nsqd地址，默认设置
	if err != nil {
		fmt.Printf("create producer failed, err:%v\n", err)
		return err
	}
	return nil
}

func main() {
	nsqAddress := "127.0.0.1:4150"
	err := initProducer(nsqAddress)
	if err != nil {
		fmt.Printf("init producer failed, err:%v\n", err)
		return
	}

	reader := bufio.NewReader(os.Stdin) // 从标准输入读取
	for {
		data, err := reader.ReadString('\n') //用回车分割
		if err != nil {
			fmt.Printf("read string from stdin failed, err:%v\n", err)
			continue
		}
		data = strings.TrimSpace(data)
		if strings.ToUpper(data) == "Q" { // 输入Q退出
			break
		}
		// 向 'topic_demo' publish 数据，string要转为[]byte类型
		err = producer.Publish("topic_demo", []byte(data)) //
		if err != nil {
			fmt.Printf("publish msg to nsq failed, err:%v\n", err)
			continue
		}
	}
}
```

# 消费者

消费者代码也很简单，可以注册收到消息的handler，具体如下
```go
// MyHandler 是一个消费者类型
type MyHandler struct {
	Title string
}

// HandleMessage 是需要实现的处理消息的方法
func (m *MyHandler) HandleMessage(msg *nsq.Message) (err error) {
	fmt.Printf("%s recv from %v, msg:%v\n", m.Title, msg.NSQDAddress, string(msg.Body))
	return
}

// 初始化消费者
func initConsumer(topic string, channel string, address string) (err error) {
	config := nsq.NewConfig() //同上，产生一个默认设置
	config.LookupdPollInterval = 15 * time.Second //重连时间
	c, err := nsq.NewConsumer(topic, channel, config) //订阅的topic和channel
	if err != nil {
		fmt.Printf("create consumer failed, err:%v\n", err)
		return
	}
	consumer := &MyHandler{
		Title: "小鲸鱼",
	}
	c.AddHandler(consumer) //设置handler

	// if err := c.ConnectToNSQD(address); err != nil { // 直接连NSQD
	if err := c.ConnectToNSQLookupd(address); err != nil { // 通过lookupd查询
		return err
	}
	return nil

}

func main() {
	err := initConsumer("topic_demo", "first", "127.0.0.1:4161")
	if err != nil {
		fmt.Printf("init consumer failed, err:%v\n", err)
		return
	}
	c := make(chan os.Signal)        // 定义一个信号的通道
	signal.Notify(c, syscall.SIGINT) // 转发键盘中断信号到c
	<-c                              // 这里阻塞，直到捕获SIGIN(ctrl + c)退出
}

```

在消费者中，会在启新的goroutine handlerLoop，在循环中调用handler.HandleMessage，所以不用担心主线程阻塞的问题
