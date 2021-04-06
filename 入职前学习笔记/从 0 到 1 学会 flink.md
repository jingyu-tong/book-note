# 1. flink runtime 核心机制

## 1.1 flink 整体架构

<img src="/Users/jingyu/Documents/GitHub/book-note/assets/image-20210113191716160.png" alt="image-20210113191716160" style="zoom: 33%;" />

flink 整体架构如上图，他底层可以通过单进程多线程方式运行，也可以通过 YARN、k8s 这种资源管理器运行，也可以在云环境运行。为了对不同运行环境进行统一封装，flink 提供了 runtime 层来提供统一的分布式作业引擎。在此基础上，提供了流处理和批处理两套API。

<img src="https://jiamaoxiang.top/2019/10/23/Flink%E8%BF%90%E8%A1%8C%E6%9E%B6%E6%9E%84%E5%89%96%E6%9E%90/Runtime.png" alt="Runtime" style="zoom:33%;" />

他采用的是 mater-slave 的结构，白框中的是 master 节点，负责整个集群中的资源和作业，右侧的两个 taskexecutor 则是 slave，负责提供具体的资源并实际执行作业。

Master 有三个组件：

* Dispatcher：负责接收用户提供的作业，并负责为这个新提交的作业拉起一个 JobManager 组件
* ResourceManager：负责资源管理，整个 flink 集群只有一个
* jobManager：管理一个作业的执行，每个作业都有一个

基于上述结构，作业执行流程为：

* 用户提交作业时，提交脚本启动一个 Client 负责作业的编译和提交。
* Client 首先编译，并判断哪些 operator 可以 chain 到一个 task 中，然后将产生的 jobGraph 提交到集群中运行
  * 类似 Standalone 的 Session 模式：AM 预先启动，Client 直接与 Dispatcher 通信并提交作业 (**这个模式下，共享 Dispatcher 和 RM，共享资源 TaskExecutor，适合小规模、时间短的任务**)
  * Per-Job 模式：AM 不会预先启动，Client 向资源管理系统(YARN/k8s 等) 申请资源来启动 AM，然后再跟 AM 中的 Dispatcher 通信，提交作业（**独享 DIspatcher 和 RM，按需要申请资源，适合长时间的大作业**）
* DIspatcher 收到 job 后，启动 JM 组件，然后向 RM 申请资源来自动作业中的具体任务
  * Session：TaskManager 已经启动了，可以直接分配资源
  * Per-Job：ResourceManager 也需要首先向外部资源管理系统申请资源来启动 TaskExecutor，然后等待 TaskExecutor 注册相应资源后再继续选择空闲资源进程分配，JobManager 收到 TaskExecutor 注册上来的 Slot 后，就可以实际提交 Task 了
* TaskExecutor 收到 task 后，启动新的线程来执行 task

## 1.2 