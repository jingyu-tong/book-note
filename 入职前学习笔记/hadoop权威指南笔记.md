# 1. Hadoop 基础

## 1.1 基本流程

* Hadoop 基于 MapReduce，首先需要编写自己的 Mapper 类，然后编写 Reducer 类，再编写一个任务主函数，对 job 进行设定，然后利用 hadoop 指令就可以运行
* job：客户端执行的一个单元，包括输入数据，MapReduce 程序以及配置信息
* task：Hadoop 会将 job 分为多个 task 来进行执行，其中包括 map task 和 reduce task，由 YARN 进行调度，如果任务失败，会再其他节点重新调度任务。
  * 分片：将输入数据分为登场的小数据块，叫做分片。Hadoop 为每个分片都会创建一个 map 任务
  * map 任务：Hadoop 会尽量在存储了输入数据的节点上运行 map 任务，来获得最优的性能。map 任务的输出为本地硬盘，因为输出为中间结果，没必要利用 HDFS 备份，即使丢失了也可以重启一个 map 任务来再次构建
  * reduce 任务：reduce 任务没有本地化的优势，因为他的输入通常来自于所有 mapper 输出的一部分（或者全部）。如果有多个 reduce 任务，每个 map 任务会对输出进行分区，为每一个 reduce 任务进行分区。
* combiner 函数：因为数据需要在 map 和 reduce 任务节点上传输，因此想办法减少传输的数据量有助于提高系统性能。combiner 函数可以处理 mapper 输出，然后作为 reducer 的输入，通过本地的一些处理优化，来减少传输的数据。
* Hadoop Streaming：Hadoop 允许使用非 Java 语言来实现自己的 map 和 reduce 函数，Hadoop 会使用标准流来作为应用程序间的接口。

**注意：**新版本的 hadoop，根据[CLASSNAME](http://hadoop.apache.org/docs/r3.1.1/hadoop-project-dist/hadoop-common/CommandsManual.html#CLASSNAME)运行的类必须是 package 内的，或者利用 jar 指令运行 jar 包内的类，这点跟书上的指令不同

## 1.2 Hadoop 分布式文件系统(HDFS)

### 1.2.1 设计目的

* 超大文件：HDFS 适用于几百 M，几百 G 甚至几百 T 的大文件
* 流式数据访问：一次写入、多次读取是最实用的访问模式。
* 有容错机制，适用于一般的商用硬件，不适于低延迟的访问
* 仅支持单个写入者，而且写操作总是以追加的方式在末尾写入数据，不支持任意位置的更改

### 1.2.2 HDFS 概念

* 数据块：HDFS 将数据分为多个分块(chunk)，作为独立的存储单元。需要注意，数据块的大小应该很大(默认为 128M，比一般文件系统大得多)。(**块大小之所以设置的这么大，是为了减小寻址开销的占比。假设网络是 100M/s 的速率，寻址时间大约为 10ms，如果希望寻址占比大约为 1%，那么就需要 100M 的数据块**)。使用块有着一些好处
  * 可以存储超过一个磁盘容量大小的文件，甚至整个集群存储一个文件
  * 处理系统只需要关注块管理，简化管理
  * 对每个块进行复制，可以提高容错能力和可用性
* namenode 和 datanode：
  * namenode：集群管理节点，存储着文件和 Chunk 的命名空间、文件和 Chunk 的对应关系、 每个 Chunk 副本的存放地点。master 服务器不持久化 chunk 存放的地址，而是在启动时，遍历所有的 datanode 来获取最新的表格
  * datanode：工作节点，存储文件块的内容，定期向 namenode 发送自己存储的块的列表
* 容错：
  * namenode 容错：因为 namenode 存储了文件目录树以及每个文件对应的 chunk 索引，所以如果 namenode 挂掉了，即使数据都还在，也是没办法恢复出数据的。有两种机制实现 namenode 容错：
    * 备份master 节点需要持久化的文件，同过配置，使 namenode 在多个文件系统上保存持久状态
    * 运行一个辅助的 namenode，定期合并边界日志和命名空间，保存合并后的命名空间副本，在主节点失效的时候可以启动，但是难免有数据丢失
  * datanode 备份：一个 chunk 会被备份到多个副本，来避免单个失效

### 1.2.3 Hadoop 文件系统

Hadoop 有一个抽象类 FileSystem，定义了文件系统的客户端接口，具体实现除了 HDFS 外，还有着其他的系统，具体如下。

* Local：使用本地磁盘文件系统

* HDFS：使用 hdfs 方式的分布式文件系统

* webHDFS：使用 webhdfs 方案，基于 HTTP 的文件系统，提供对 HDFS 的读写访问，还有 HTTPS 版本的 Secure WebHDFS

* FTP：由 FTP 服务器支持的文件系统

  尽管 Hadoop 提供了对其他文件系统的访问，但是处理大数据时的时候，最好选择有数据本地化优化的分布式文件系统，如 HDFS

## 1.3 java 接口

### 1.3.1 读取数据

* 利用 URL 读取数据(缺点明显，只能设置一次 URLStreamHandlerFactory 实例)

  利用 URL 类可以解析 URL 的资源，也包括 hdfs 路径，利用它打开数据流，就可以简单的读取出对应 URL 中的数据，具体见注释:

  ```java
  public class HadoopTestMain {
      static { // 静态代码块，JVM 会在加载类的时候，按出现顺序执行这些静态代码块
        // 
          URL.setURLStreamHandlerFactory(new FsUrlStreamHandlerFactory()); // 设置 URL 流处理的 handler 工厂，一个 JVM 只能设置一次
      }
      public static void main(String[] args) throws Exception {
          InputStream in = null;
          in = new URL(args[0]).openStream(); //创建一个 URL 对象，并且打开流
        IOUtils.copyBytes(in, System.out, 4096, false); // 拷贝流到标准输出
      }
  }
  ```

* FileSystem

  利用 FileSystem 来打开一个文件的输入流，注意，Hadoop 利用 Hadoop Path 对象来代表文件，具体操作如下：

  ```java
  Configuration conf = new Configuration();
  FileSystem fs = FileSystem.get(URI.create(uri), conf); // 这里的 URI 主要是提供创建连接的信息，和后续的 PATH 不必要是一样的
  in = fs.open(new Path(uri));
  ```

  fs.open()返回的是 FSDataInputStream 对象，除了支持基本的读入，skip 等功能，还实现了 Seekable 和 PosisionedReadable 接口，分别支持指定绝对位置跳转和从一个偏移量读取文件一部分。

### 1.3.2 写入数据

* FSDataOutputStream create(Path f):创建对应文件，并返回一个对应的写入流。
* FSDataOutputStream append(Path f):在现有文件末尾追加数据
* FSDataOutputStream 支持获取当前位置，但是 HDFS 只允许对打开的文件顺序写入，或者在现有文件末尾追加数据！

### 1.3.3 查询文件系统

* FileStatus：调用 FileSystem::getFileStatus() 方法获取FileStatus 类型的对象，存储了文件长度、块大小、修改时间等文件元数据信息
* FileStatus[] FileSystem::listStatus() 方法可以获取目录中的内容
* boolean delete(Path f, boolean recursive)：删除数据，当 f 为非空目录时，recursive 必须为 true 才能删除，否则抛出异常

### 1.3.4 数据流

介绍一下客户端读写请求的完整交互流程：

<img src="/Users/jingyu/Documents/GitHub/book-note/assets/image-20201211191313879.png" alt="image-20201211191313879" style="zoom: 33%;" />

* 文件读取：读取流程如上图，步骤如下：

  * 通过 RPC 跟 namenode 通信，来获取文件起始块的位置
  * FSDataInputStream 对象会被创建来，客户端通过这个对象，读取文件内容。读取时，会读取距离最近的 datanode
  * 一个块读取结束，会寻找下一个块的最佳 datanode，然后继续读取，在客户端来看，一直是在读取一个连续的流
  * 如果这批块读完了，重新向 namenode 请求下一批的 datanode 地址
  * **基本就是 GFS 的流程，主要优点就是分离了 namenode 和 datanode，降低了 namenode 的传输量，将请求更多的打散到各个 datanode 中**

  <img src="/Users/jingyu/Documents/GitHub/book-note/assets/image-20201211192519060.png" alt="image-20201211192519060" style="zoom:33%;" />

* 文件写入：写入流程如上图，步骤如下：

  * 调用 RPC 在 namenode 处新建文件，并进行一些检查判断是否能够创建
  * 客户端节点在内部缓存下客户写入的数据，称为**数据队列**，挑选出适合存储的 datanode，要求 namenode 分配 chunk。
  * 客户端将包发给第 1 个 datanode，第一个存储数据包，并发送给第二个 datanode，一次类推构成数据流
  * 客户端内部维护着**确认队列**，收到所有 datanode 的确认信息后，改数据包才会从确认队列中删除，如果写入过程中有节点发生故障：
    * 关闭管线，当前确认队列的所有包都添加回数据队列的最前端，保证故障 datanode 的下游节点不会漏掉包(放回队列的本质，实际上就是重写一遍)
    * 为存储在另一正常datanode的当前数据块指定一个新的标志，并将该标志传送给namenode，以便故障datanode在恢复后可以删除存储的部分数据块（用新标志让故障的标志无效）
  * 只要写入超过 min 副本数的设置，写操作就会成功，关闭后，联系 namenode，告知写入完成。

* HDFS 系统的一致性

  * 新建一个文件后，他能在命名空间中立刻可见(因为文件状态相关，都是走的 namenode)
  * 写入的文件不保证立即可见，即使数据流已经刷新并存储(因为可能读取还没有同步的 datanode)
  * 写入超过一个块后，第一块对所有的 reader 都可见，也就是对正在写入的块不可见
  * hflush 强行刷新缓存到 datanode 的方式，返回后，所有 datanode 的内存有着所有写入的数据
  * Hsync 和 hflush 类似，但是会刷到磁盘上

## 1.4 YARN

### 1.4.1 YARN 运行机制

YARN 有四个基本组件：

* ResourceManager： 每个集群只有一个，负责集群资源的调度
* NodeManager：每个节点有一个，负责单节点的资源管理和调度
* ApplicationMaster：每个应用程序一个，管理该应用程序
* Container：对任务运行资源的抽象，可以理解为一个计算机

YARN 的工作流程如下：

* client 向 YARN 提交应用程序，ResourceManager 为该程序分配第一个 Container，并与 Container 对应的 NodeManager 通信，让其在 Container 中启动应用程序的 ApplicationMaster
* ApplicationMaster 向 ResourceManager 注册，这样用户可以通过ResourceManager查看应用程序的状态。然后 ApplicationMaster 将为各个任务申请资源，并监控到运行结束。
* ApplicationMaster 向 ResourceManager 申请和获取资源，获取后与对应的 NodeManager 通信，要求其启动人物
* NodeManager 启动任务，并且各个人物向 ApplicationMaster 汇报情况（如果有失败的话，ApplicationMaster 可以重启任务）
* 运行结束后，ApplicationMaster 向 ResourceManager 注销并关闭自己

TODO：4.1.2 没看懂

### 1.4.2 YARN 与 MapReduce1 比较

首先介绍一下 MapReduce1 的运行机制：

* jobtracker 负责调度 tasktracker 上的任务来协调运行在系统上的作业，tasktracker 将进度发送给 jobtracker，jobtracker 可以在任务失败的时候进行重新调度
* 因此，jobtracker 同时负责作业调度和任务监控。

因为 jobtracker 做了大部分的工作，因此很容易再此处出现上限（大概 4000 个节点），且容错能力不好

YARN相比，ResourceManager 只负责资源管理，而每个任务的监控，则有对应任务的 ApplicationMaster 来负责，这样一方面打散了监控任务(不同任务的监控可以运行在不同的 node 上)，另一方面也简化了核心节点 ResourceManager 的功能。

### 1.4.3 YARN 的调度

当资源受限的时候，YARN 应用显然也需要以某种方式进行调度，因此 YARN 有着几种调度器可以选择

* FIFO 调度器
  * 先入先出顺序运行应用
  * 简单易懂，不需要进行配置，但是不适合共享集群。因为大的应用会占用集群所有资源，小作业就要一直被阻塞
* 容量调度器
  * 划分出一个单独的队列来保证小队列以提交就可以启动，来减小小作业被大作业阻塞的问题
  * 问题在于小作业运行需要的容量需要被保留，因此容量调度是以降低集群利用率为代价的，也意味着大作业执行时间比 FIFO 要长
* 公平调度器
  * 在所有运行的资源中动态平衡资源，比如，第一个(大)作业启动的时候，会获得集群的所有资源，第二个(小)作业启动时，会分配到集群的一般资源，让每个左右公平共享
  * 后续的作业启动得到公平资源的时间有滞后，因为他必须等待之前作业使用的容器用完并且释放资源。当小作业结束并不再申请资源后，大作业将再次使用全部的集群资源
  * 默认情况下，使用内存来进行公平调度，但是在多种资源需要调度的时候，使用单一规则可能并不合适。(**主导资源公平性 Dominant Resourse Fairness, drf**：可以更好的解决这个问题。drf 统计每个应用主导占比的资源，然后按照主动资源的公平性进行分配)

## 1.5 Hadoop I/O 操作

TODO：先不看了

# 2. MapReduce

上一部分主要是介绍 Hadoop 体系的整体知识，这部分主要介绍 Hadoop 的主要功能，MapReduce 的开发部分。

开发 MapReduce 的流程一般为：

* 编写 Map 函数和 Reduce 函数
* 编写 Map 和 Reduce 的单元测试
* 编写驱动程序运行作业，并使用本地小数据集进行测试，出问题的话，重新修改单测和代码
* 上传到集群运行

## 2.1 配置

xml 就像 json 一样，可以指定属性和对应的值，一个设置可以由多个 xml 文件进行合并得到，且后添加的属性会覆盖之前的属性。

根据之前说的流程，经常需要改变程序测试的方式，因此可以对应三个模式编写好对应的 xml 文件，然后利用`hadoop fs -conf xml-location`指定设置(不指定，会再$HADOOP_HOME/etc/hadoop 查找配置信息，如果设置了HADOOP_CONF_DIR，则从这个位置读取)

Hadoop 自带了一些辅助类来简化命令行，GenericOptionParser 可以用来解释命令行，但通常不直接使用，而是使用 ToolRunner(ToolRunner 的 run 方法，会先调用 GenericOptionParser 来解析命令行，再把剩下的参数传递给 Tool 接口的 run 方法)，来调用一个自己实现的 Tool 接口

## 2.2 单测

因为 Map 和 Reduce 这种函数式编程的形式，所以可以很方便的进行独立的测试。MRUnit 是一个测试库，下面以 Map 为例

```java
// 新建一个 MapDriver，类型要和 Map 对应
// 指定测试的 Mapper，输入和输出后，运行测试
new MapDriver<LongWritable, Text, Text, IntWritable>() 
  .withMapper(new TemperatureMapper()) 
  .withInput(new LongWritable(0), value)
  .withOutput(new Text("1950"), new IntWritable(-11))
  .runTest();
```

这里有个地方需要注意，导包的时候需要导入 mapreduce 底下的新版 api，不能直接导入 mrunit 下的老版

```java
import org.apache.hadoop.mrunit.mapreduce.ReduceDriver;
import org.apache.hadoop.mrunit.ReduceDriver;
```

## 2.3 运行本地数据(单机模式)

通过编写驱动程序，可以先在单机模式下运行 hadoop，具体如下

```java
@Override
public int run(String[] strings) throws Exception {
  // 初始化 job
  Job job = Job.getInstance(getConf(), "MaxTemperature");
  job.setJarByClass(getClass());

  // 设置输入输出流地址
  FileInputFormat.addInputPath(job, new Path(strings[0]));
  FileOutputFormat.setOutputPath(job, new Path(strings[1]));

  // 设置 mapper 和 reducer
  job.setMapperClass(TemperatureMapper.class);
  job.setReducerClass(TemperatureReducer.class);

  // 设置输出 key 和 value 的类型，需要和 reducer 保持一致，mapper 默认和 reducer 一致，不同的话需要另外设置
  job.setOutputKeyClass(Text.class);
  job.setOutputValueClass(IntWritable.class);

  // 等待任务完成，true 则打印进度
  job.waitForCompletion(true);
  return 0;
}
```

然后利用单机模式的 conf 就可以运行，具体命令为

```shell
mvn package
hadoop jar hadoop-test.jar TemperatureDriver -conf conf/hadoop-local.xml input/ncdc/micro output && cat output/* 
```

除了用命令行的方式，也可以为 driver 程序编写单测，进行测试

## 2.4 集群运行

分布式环境中，需要将作业 jar 文件发送到集群，hadoop 会搜索设定的类路径自动找到这个作业的 jar 文件

客户端的类路径有以下几部分组成

* 作业的 jar 文件
* 做的 jar 文件的 lib 目录中的所有 jar 和 classes 目录
* HADOOP_CLASSPATH 定义的类路径

在集群上，map 和 reduce 任务在各自的 JVM 上运行，他们的类路径不受 HADOOP_CLASSPATH 控制(这个只能控制驱动程序，不能控制后面启动的任务)

任务通常会有依赖的库，有以下几种方式处理：

* 解包库然后和作业一起打包进 jar
* 将作业 jar 和 lib 目录中的库打包
* 保持库和作业 jar 分开，并通过 HADOOP_CLASSPATH 将他们添加到客户端的路径，通过 -libjars 添加到任务的类路径(**从创建的角度来说，这种方式是最简单的，因为依赖不需要在作业的 jar 中创建，而且意味着在集群中更少的文件转移**)

运行表示：

* Application ID 和 Job ID: 对于 mapreduce 来说，这两个是一样的，构造为 application/job-时间戳-应用下标(从 1 开始)
* 任务 ID：一个 job，或者说 Application，可能需要有多个 task，task id 用来表示任务，形式为 task-时间戳-应用下标-m/r-任务下标(从 0 开始)

## 2.5 日志

YARN 有一个日志聚合任务，会将已完成的应用日志，搬移到 hdfs 中，默认关闭，可以通过 yarn.log-aggresstion-enable 配置开启

每个子任务都用 log4j 产生一个日志文件(称为 syslog)，还有一个保存 stdout 文件以及一个保存 stderr 的文件。但是在 Streaming 模式下，标准输出用于 map 和 reduce 的输出，所以不会出现在标准输出日志文件中

java 中利用 api 可以写入到 syslog 中

一般而言，对于简单的任务，可以方便的转换成 MapReduce 任务，但是如果处理过程比较复杂，通常将其转换为多个 MapReduce 作业，串行的进行处理。(对于复杂问题，也可以考虑使用 hive，spark 等高级语言来进行处理)

## 2.6 MapReduce 工作机制

### 2.6.1 作业流程

Hadoop 运行中有以下几个实体：

* 客户端：提交 MapReduce 作业，相当于编写的驱动程序
* YARN ResourceManager：资源管理器，负责协调集群上的计算机资源分配
* YARN NodeManager：节点管理器，负责启动和监视集群中机器上的计算容器
* MapReduce 的 ApplicationMaster：负责协调运行 MapReduce 作业任务

MapReduce 任务运行的完整流程如下：

* 提交作业
  * 在客户端运行驱动程序，驱动程序设置并提交一个 job
  * 向 ResourceManager 申请一个应用 ID
  * 检验输入输出路径、权限等是否合法，可以读取或写入
  * 将运行作业所需要的资源(包括作业 jar，配置文件等)放入以作业 ID 命名的共享文件系统中
* 作业初始化
  * ResourceManager 将请求传递给 YARN 调度器，获取一个容器，并启动 ApplicationMaster
  * ApplicationMaster 初始化作业
  * 针对每个输入分片，创建一个 map 任务对象，以及根据设置创建多个 reduce 对象
  * 如果小作业，则直接在同一 JVM 运行
* 分配任务，向资源管理器请求容器，首先请求 Map 任务所需要的，5%map 任务完成后，才会请求 reduce
* 执行任务
  * ApplicationMaster 通过与节点管理器通信来启动容器
  * 在指定容器中运行用户定义的 map 或 reduce 函数
  * 如果使用 Streaming 方式，则会借用 stdin 和 stdout 来执行用户提供的可执行程序(通常用来支持别的语言的可执行文件)
* 进度和状态监控（因为 MapReduce 任务相对于传统的程序来说，运行时间要长的多，因此能够获取当前程序的一些情况是很重要的）
  * map 任务进度：统计的是已经处理的输入所占的比例
  * reduce 任务：完成复制1/3，完成排序1/3，剩下的为完成 reduce 任务阶段的占比
* 作业完成：job 会定时轮询 ApplicationMaster，一旦最后一个任务完成，就可以查询到作业成功

### 2.6.2 shuffle 和排序

* map 端
  * map 端的输出是不会存储到 HDFS 上的，因为没有必要，因为丢失的话，可以直接重启，但是为了效率，一般系统都不会直接输出到磁盘，而是采用缓冲区的方式，Hadoop 也不例外。具体来说，会先在环形缓冲区内写入数据，当达到阈值的时候，开始把内容写入磁盘。在写入磁盘前，会对数据进行排序、分区(根据分区函数和 reduce 任务个数)、和可选的 combiner函数。
  * 当执行结束，会将所有文件合并成一个已经排序、分区后的文件，如果溢出文件较多(默认是超过 2)，才会调用可选的 combiner。
* reduce 端
  * reduce 端会去各个已经完成的 map 任务中拉取自己分区的数据，这个步骤是并行的，默认有 5 个后台线程
  * 拉取之后，开始合并阶段，reduce 任务会合并所有的输出，并保证输出文件是有序的
  * 得到完整合并后的文件，就可以开始执行 reduce 任务了

# 3. Hadoop 项目

## 3.1 Hive

Hive 是一个构建在 Hadoop 基础上的数据仓库，目的是希望避免编写复杂的 MapReduce 程序，转而使用更简单的 SQL 来进行分析。

> 数据仓库和传统的数据库不同，数据库主要用于事务处理(OLTP)，而数据仓库主要用于数据分析(OLAP)
>
> 数据库的特点是：复杂表结构、读写优化、相对简单的 query，单次作用于相对于少的数据
>
> 数据仓库的特点是：简单的表结构、读优化、复杂的 read query，单次作用于大量的数据(历史数据)
>
> 那么为什么要在数据库之外，额外设计这样一套数据仓库呢？(一个是业务数据来源于多个数据源，聚合不易。另一方面，如果在数据源系统进行大量的数据分析，那么可能会危及业务运行，给系统很重的负担。最后，OLTP 对大量数据没有做优化，运行效率可能更低)

hive 由于基于 HDFS，因此最开始是不支持 update 操作的(因为 HDFS 不提供随机访问和更新)，后续版本中 hive 支持了 update等操作，会将变化储存在较小的增量文件中，由 metastore 在后再运行 MapReduce，定期将增量文件合并到基表中。

### 3.1.1 Hive 服务

hive 除了能够使用命令行环境，还可以启动其他的服务

* cli：Hive 的命令行接口，默认
* hiveserver2: 让 hive 以提供 Thrift 服务的服务器形式运行，这样就允许用不同的语言编写客户端进行访问
* hwi：hive 的 web 接口，在没有安装客户端软件的情况下，可以替代 cli
* jar：与 hadoop jar 等价，这是运行路径同时包含 hadoop 和 hive 程序最简便的方法

### 3.1.2 Metastore 元数据存储

元数据：表格的结构，权限，表存放的位置等描述数据的数据

因为元数据访问需要很好的随机访问和更新的事务支持，而像 HDFS 只支持序列访问，因此不适合存放元数据，因此 Hive 会将元数据单独存放，使用 metastore 服务，分为服务和后台数据存储两部分：

* 内嵌 metastore 配置：采用本地磁盘作为存储的 Derby 数据库实例，优点在于简单，缺点是每次只有一个内嵌 Derby 数据库可以访问某个文件，这意味着一次只能为每个 metastore 打开一个 hive 会话，启动第二个会报错
* 本地 metastore 配置：metastore 仍然和 hive 运行在同一进程，但是数据库运行在远端(如 mysql)
* 远程 metastore 配置：metastore 和 hive 运行在不同进程内

## 3.2 Spark

spark 是用于大数据处理的集群计算框架，和其他的不同，他没有基于 MapReduce，但是于 Hadoop 紧密集成，可以在 YARN 上运行，并支持 Hadoop 文件格式，及其存储后端如 HDFS。

Spark 的优势在于，他能将作业和作业之间产生的大规模的工作数据集存储在内存中(与之对比，MapReduce 的工作与工作间结果，需要存储在 HDFS 中，代价大)，另外 spark 对 DAG 引擎的支持，可以处理任意操作的流水线。

<img src="/Users/jingyu/Documents/GitHub/book-note/assets/image-20210103184724773.png" alt="image-20210103184724773" style="zoom: 50%;" />

Spark 整体流程如上图：

* 首先每个应用都有一个 Driver(驱动程序)，和若干个 job(每个 action 都会分隔出一个 job)构成，Driver会构建 SparkContext，进行资源的申请、任务的分配合监控
* 资源管理器为 Executor 分配资源，并启动 Executor 进程
* SparkContext 根据 RDD 的依赖关系，构建DAG，DAG 调度器分解成多个 stage，Task 调度器将 stage 按照RDD 分区的数量分为多个 task，Executor 申请 task 运行
* 运行结束，写入数据并释放资源

### 3.2.1 弹性分布式数据集 (Resilient Dstributed Dataset, RDD)

RDD 本质就是一个只读记录集合，每个 RDD 可以分成多个分区，每个分区就是一个数据集片段，RDD 只能通过数据创建，或者从其他 RDD 上执行转换操作得到新的 RDD。

![image-20210103191800112](/Users/jingyu/Documents/GitHub/book-note/assets/image-20210103191800112.png)

如上图所示，RDD 的动作分为 transformation 和 action 两种，transformation 是惰性的，只会构建RDD 关系，action 是立即产生输出结果的操作。

RDD 之间，有着两种依赖关系：

* 窄依赖：一个子 RDD 的一个分区，对应一个父 RDD 的一个或多个多个分区，因为从数据流向上变窄了，所以得名
* 宽依赖：一个父 RDD 分区，对应一个子 RDD 的多个分区

stage 的划分，就是根据 RDD 间依赖关系得到的，因为窄依赖可以实现流水线优化，而宽依赖无法实现，因此DAG 每次宽依赖的边作为分隔，构成一个 stage

### 3.2.2 执行器与集群管理器

spark 支持多种不同的部署方式：

* 本地模式：有一个 Executor 和 Driver 运行在同一个 JVM 中
* 独立模式：Spark 自己的一个集群管理器的简单实现，不推荐
* Mesos 模式、YARN 模式：利用其它集群管理器，进行管理，这种比较推荐，因为会综合考虑集群上运行的其他节点，统一调度，集群利用率高

## 3.3 Zookeeper

zookeeper 是分布式协调系统，他通过一个核心的文件系统和一些简单的抽象操作，可以实现多重协调数据结构和协议，如：分布式队列、分布式锁、领导选举等。

### 3.3.1  zk 实例

这部分会介绍一些 zk api，为了更好的查看zk 的变化，这里先介绍一个利用 zkCli.sh 脚本的交互方式：

* 利用zkCli.sh -server localhost:2181连接 zk
* 使用支持的命令进行交互，常用的包括 ls，create，delete 等。

下面开始在一个实例中，介绍 zk 的用法，具体使用场景为一组服务器为客户端提供服务，

* 创建组：创建一个 znode，作为父亲节点来代表服务器组，流程如下：
  * 首先调用 zookeeper 的构造函数创建一个 zk 实例，当一个 zk 实例被创建，会启动一个线程连接到 zk 服务。由于构造函数会直接返回因此在使用前需要确定已经建立连接
  * 当建立连接后，watcher 的 process 方法会被调用，我们在回调函数中通知 zk 实例可用
  * 调用 zk.create 创建 znode
  * 结束后调用 zk.close 关闭连接
* 加入组：在组节点下创建一个临时的 znode，只要这个节点存在，就表示有一个对应的服务器可以运行
* 列出组成员：调用 zk.getChildren 可以获取路径下的列表
* 删除节点：滴啊用 zk.delete 方法可以显示的删除某一个节点，无论是临时还是持久

### 3.3.2 zk 服务

zk 相关的概念和定义基本在之前[论文总结](https://github.com/JingyuTongdlut/6.824-note/blob/master/note/paper/Zookeeper.md)里都写了，这里只写论文里没有提到的一些点。

1. 集合更新

   zk 有一个 multi 的操作，可以将多个基本操作集合成一个操作单元，zk 保证这个集合成功被执行，或者同时失败。典型的例子比如更新一个图的一条边，我们需要一个同时删除两个节点对应的边来提供正确的操作。

2. watch 触发

   在读操作 exists/getChildren/getData 上，可以设置 watch，这些 watch 可以被写操作 create/delete/setData 触发，具体规则为

   * znode 被创建、删除或者数据更改时，设置在 exist 上的 watch 会被触发
   * znode 被删除、数据更新时，getData 的watch 被触发，因为 getData 要求获取时 znode 已经存在
   * znode 的子节点创建或删除，或者 znode 自己被删除，getChildren 上的 watch 会触发

3. 访问控制列表(Acquire control list, ACL)

   每个 znode 都有一个列表，ACL 依赖 zk 的客户端身份验证机制，zk 提供了几种验证方式：

   * digest：通过用户名和密码识别客户端
   * sasl：通过 kerberos 来识别客户端
   * ip：通过客户端 ip 来识别客户端

### 3.3.3 利用 zk 构建高阶应用

1. 配置服务：利用 zk 存储配置信息，其他节点通过 watch 机制监测配置信息的变更。
2. 可复原的 zk：分布式系统难以避免发生故障，java api 提供了两个异常
   * InterruptedException 异常：如果操作被中断，则会抛出这个异常
   * KeeperException 异常：如果服务器发出错误信号，或者与服务器存在通信问题抛出的异常。
     * 状态异常：操作不能应用于 znode 树，比如另一个进程正在修改znode 导致版本不一致，这种异常是可以预见的，程序员应该编写代码进行处理
     * 可恢复异常：能够在同一个 zk 回话中回复的异常，比如连接丢失引发的异常，zk 会重连其他节点
     * 不可回复异常：比如 zk 回话失效，可能是因为超时或者回话被关闭，或者身份验证失败等。 



