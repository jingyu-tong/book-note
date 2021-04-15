# hello k8s

本文主要介绍怎么部署应用到 k8s 上，采用的系统为 macos11

## 1. 环境安装

首先需要安装 k8s，这里我们采用 mac 自带的 k8s，安装过程见[mac docker 及 k8s 安装](https://learnku.com/articles/42843)

> k8s 本身并不依赖 docker，事实上 k8s 除了 docker 外，还支持其他几种容器平台，这里主要是用 docker 较为方便

![image-20210415201200076](/Users/jingyu/Documents/GitHub/book-note/assets/image-20210415201200076.png)

当显示上述图片样式的时候，说明 k8s 和 docker 都已经安装完成，为了方便操作 k8s，这里安装 Kuboard，安装过程见[安装 kuboard](https://kuboard.cn/install/install-dashboard.html#%E4%B8%BA%E5%BC%80%E5%8F%91%E6%B5%8B%E8%AF%95%E4%BA%BA%E5%91%98%E6%8E%88%E6%9D%83)

![image-20210415202540926](/Users/jingyu/Documents/GitHub/book-note/assets/image-20210415202540926.png)

安装完成后，访问`http://任意一个Worker节点的IP地址:32567/`，就可以进入管理界面，如上图所示。至此，就完成了环境的安装。

## 2. 编写代码及 DockerFile

这里采用一个用 go 写的很简单的服务器进行验证，服务器代码如下：

```go
func echo(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "hello, your request is:%v\n", r.FormValue("key"))
}
func main() {
	logFile, err := os.OpenFile("./logs.log", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
	if err != nil {
		fmt.Println("open log file failed, err:", err)
		return
	}
	log.SetOutput(logFile)
	log.SetFlags(log.Llongfile | log.Lmicroseconds | log.Ldate)

	if err != nil {
		fmt.Println("listen failed, err:", err)
		return
	}
	log.Println("开始运行")
	http.HandleFunc("/", echo)
	err = http.ListenAndServe(":4500", nil)
	if err != nil {
		log.Printf("listen failed, err: %v", err)
	}
}
```

接着，要编写 Dockerfile 来定义 Docker 镜像生成的流程，这里的 Dockerfile 编写如下：

```dockerfile
FROM golang:latest

ENV GOPROXY https://goproxy.cn,direct
WORKDIR /home/echoserver
COPY . /home/echoserver
RUN go mod tidy
RUN go build .

EXPOSE 4500
ENTRYPOINT [ "./echo" ]
```

这些指令的含义分别为：

* FROM：指定基础镜像，必须有，且必须在第一条
* WORKDIR：指定工作目录，目录不存在，则会创建
* COPY source target：将构建目录上下文目录中的源，拷贝到新一层镜像内的目标路径位置
* RUN：执行指令
* EXPOSE：声明运行时容器提供服务的端口，**这只是一个声明**，应用不会因为这个声明就开启端口
* ENTRYPOINT：格式和 RUN 一样，分为两种，用于指定容器启动程序和参数
  * exec：`ENTRYPOINT "CMD"`
  * Shell：`ENTRYPOINT ["curl", "-s", "http://ip.cn"]`

因此，上面 Dockerfile 的含义就是在容器中创建工作目录，并且将当前目录的文件拷贝到容器中，然后拉取依赖并且编译，最后执行编译出的程序

最后，输入`docker build -t echo-docker .`编译容器镜像

## 3. k8s 部署

![image-20210415204934470](/Users/jingyu/Documents/GitHub/book-note/assets/image-20210415204934470.png)

首先需要选择一个命名空间（虚拟机群），这里选择 default。

<img src="/Users/jingyu/Documents/GitHub/book-note/assets/image-20210415205535830.png" alt="image-20210415205535830" style="zoom: 50%;" />

进入命名空间后，创建工作负载，填写相关内容，需要注意镜像名要和之前编译的一致，且抓取策略不能选择 always，否则将会从远端拉取镜像，但是是拉不到的。

保存后，可以看到容器已经被部署了，且正常运行，但是会发现访问 localhost:4500 是难以访问的，因为默认没有创建 service，service 提供了一层 pod 的抽象，使得一组具有某些特征的 pod 对外有一个统一的 ip，通过这个 ip 才能访问。

这个 ip 有着不同的暴露方式：

* ClusterIP（默认）：集群内部 IP 上公布服务，这类 service 只有内部集群可以访问
* NodePort：使用 NAT 在集群中的每个端口上公布服务，可以使用集群中任意节点+端口号`NodeIP:NodePort`的方式访问服务
* LoadBalancer：创建一个集群外部的负载均衡器，使用该 ip 作为服务的地址

![image-20210415213754535](/Users/jingyu/Documents/GitHub/book-note/assets/image-20210415213754535.png)

通过选取 NodePort 方式，我们可以在集群外通过任意节点 ip+节点端口对其进行访问，成功结果如图。

![image-20210415213827542](/Users/jingyu/Documents/GitHub/book-note/assets/image-20210415213827542.png)