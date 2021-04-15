# 1. k8s 基础

## 1.1 k8s 架构和组件

* namespace：k8s 中可以创建多个虚拟集群，称为 namespace。如果设置时没有明确定义命名空间，则集群将在始终存在的默认 namespace 中创建
* node：k8s 运行在node 上，node 是集群中的单个机器，他可以是物理机器，也有可能是云中的虚拟机。k8s 是典型的主从结构，因此节点类型有两种：
  * master 节点：控制其他所有节点的节点，他向集群吉他节点发送消息分配工作，工作节点向主节点的 API Server 汇报（*master 节点的 API Server 组件是节点于控制平面通信的唯一端点，这是事实所所所 worker 节点和 master 节点就 pod、deployment 和其他对象状态进行通信的点*）
  * worker 节点：部署容器或 pod 时，其实是部署到 worker 节点上运行，worker 节点托管和运行一个或多个容器的资源
* pod：k8s 中的逻辑工作单位，一个 pod 允许把多个容器按指定的组合方式组合在一起创建应用程序，并将其作为一个单位来管理
* service：是一组逻辑上的 pod，他提供了单一的 ip 地址和 dns 名称，可以通过其访问服务内的所有 pod，可以方便的进行设置和负载均衡等
* ReplicationController、ReplicaSet：负责管理 pod 生命周期的组件，可以用于启动 pod 或者收到指示时杀死 pod，有助于实现指定运行的 pod 数量的状态

* kubectl：k8s 命令行工具，用于和 k8s 集群以及其中的 pod 通信

> 使用 k8s 而不直接使用 docker 的原因之一，是 k8s 能够自动扩展应用实例的数量以满足工作负载的需求

## 1.2 k8s 网络相关概念

为了和外部用户 or 程序进行安全的交互，需要设置安全规则来管控流量，k8s 中流量分为两种：

* Ingress：进入 pod 的流量
* Egress：从 pod 到集群外的出站流量
* Ingress Controller：定义入口和出口策略前，需要启动入口控制器组件，集群默认不启动

为了保证应用程序的弹性，需要在不同节点上，创建多个 pod 的副本，假设策略为 webserver-1的 pod 维持在 3 个副本，那么 ReplicationController 将会监控副本的数量，如果其中任何一个 replica 不可用，那么将自动创建一个新的系统

* 具体的，通过 deployment 定义所需状态，master 节点中的 depolyment controller 负责使当前状态不断趋向所需状态

## 1.3 k8s 监控工具

* Prometheus 监控：开源监控和报警工具，包含一个内部存储用来收集指标，并可以将数据暴露给外部解决方案
* Grafana 仪表盘：优秀的仪表盘、分析和数据可视化的工具，没有 Prometheus 全功能的数据收集能力，但是后者没有其的数据呈现页面。最好的结合是二者一起使用，Prometheus 负责数据收集和汇总，Grafana 负责数据展示。

# 2. hello k8s

