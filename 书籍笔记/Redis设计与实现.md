<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Redis设计与实现](#redis设计与实现)
	- [数据结构](#数据结构)
		- [1.SDS(简单动态字符串)](#1sds简单动态字符串)
		- [2.链表](#2链表)
		- [3.字典](#3字典)
		- [4.跳跃表](#4跳跃表)
		- [5.整数集合](#5整数集合)
		- [6.压缩列表](#6压缩列表)
	- [对象](#对象)
		- [1.字符串对象](#1字符串对象)
		- [2.列表对象](#2列表对象)
		- [3.哈希对象](#3哈希对象)
		- [4.集合对象](#4集合对象)
		- [5.有序集合](#5有序集合)
	- [数据库](#数据库)
		- [1.过期建删除策略](#1过期建删除策略)
	- [RDB持久化](#rdb持久化)
		- [1.RDB文件创建与载入](#1rdb文件创建与载入)
		- [2.RDB文件结构](#2rdb文件结构)
		- [3.AOF持久化](#3aof持久化)
	- [事件](#事件)
- [Redis实战](#redis实战)
	- [1.为什么要用 redis](#1为什么要用-redis)
	- [2.redis 为什么这么快](#2redis-为什么这么快)
	- [3.redis 删除策略](#3redis-删除策略)
	- [4.数据库和缓存双写方案](#4数据库和缓存双写方案)
	- [pipeline 和 mget](#pipeline-和-mget)

<!-- /TOC -->
# Redis设计与实现
用来记录Redis设计与实现相关的一些笔记，后续有空阅读Redis源码的话，相关的想法也记录在其中

## 数据结构
### 1.SDS(简单动态字符串)
封装了字符串长度、缓冲区空闲大小信息，有自己的空间分配策略。
* 空间预分配
增长后未使用空间 = max(1M, new_len)，通过预分配策略，可以减少内存分配次数
并不立即回收空闲的空间，为可能增长做优化，但是也提供了API用于空闲的空间
### 2.链表
封装了listNode，通过list来持有的双向链表
### 3.字典
字典，是一种用于保存键值对(key-value pair)的抽象数据结构。
Redis采用hashmap实现字典，解决冲突用的是链地址法。
* sizemask
hash表大小为2^n，hash表内部维护了一个sizemask = size - 1，这样在之后算出key，可以和sizemask做与运算，等价于取余，但是更快。
* rehash
为了应对hash表负载因子过小(空间利用率太低)或过高(容易发生冲突)，会进行rehash:
  * 如果是扩展，ht[1]的size大于等于ht[0].used*2的2^n
  * 收缩，为大于等于ht[0].used的2^n

在rehash过程中，会把标志位rehashidx置为0,采用的策略为渐进式rehash，分多次、渐进的转移。此时，各种操作会在两个hash表中进行，并且新添加的只会添加到新的表中。
### 4.跳跃表
因为在大部分情况调表的效率可以和平衡树媲美，并且实现上更为简单，所以Redis使用调表来代替平衡树，Redis中的最高层数为32。
每个跳表节点有个分值用来排序，还有个obj对象指向一个字符串对象

**跳表时间复杂度**
跳表的层数 log1/p(n)
时间复杂度为 log(n)

### 5.整数集合
intset是集合键的底层实现之一，当集合只有整数元素，并且数量不多，Redis就会使用intset。
intset里的`encoding`用来表示编码方式，可以用来进行升级操作。
* 升级(upgrade)
当新添加的元素类型长于所有现有的所有元素，会将数组扩容，并将所有元素同一为新元素的类型。新元素由于最长，，所以不是最大就是最小。
* 降级
不支持降级操作

这种操作使得数组可以被随意的添加任意类型的整型，并且，能够尽可能的节约内存。
### 6.压缩列表
当列表建和哈希键为小整数值或短字符串时，会采用ziplist作为实现。
压缩列表由字节数、等参数和节点(entry)构成。
* 压缩列表节点
可以保存一个字符数组或一个整数值,由previous_entry_length,encoding和content三部分组成。
  * previous_entry_length
  记录之前节点的长度，可能为1个字节或者5个字节(0xFE为首字节)。压缩列表可以通过这个变量，寻访到前一节点的地址。
  * encoding
  记录了content保存的数据类型以及长度
* 连锁更新
在某一介于250-253字节大小的entry1(pre此时为1个字节)前插入一个大于254长度的entry0，那么entry1此时长度增加，会导致接下来的节点都有可能不断的需要更新，因此称为连锁更新。
## 对象
Redis基于此前介绍的数据结构，构架了一个对象系统，包括字符串对象、列表对象、哈希对象、集合对象和有序对象。
采用此种方法，可以对不同对象分配不同执行指令，并且能够通过不同实现来优化效率。
此外实现了基于引用计数(类似智能指针)的内存回收机制。
### 1.字符串对象
字符串编码方式可以为int、raw和emstr。
* int
当保存对象为一个整数值，并且能用long表示时，将编码方式变为int，并将void* ptr 指针变为long类型的。
* raw
当保存对象为字符串，并且长度大于39字节，用SDS保存
* embstr
用于优化短字符串，申请一块大内存，同时装入字符串对象和SDS。这样申请和释放都只需要一次就能完成，而且处于同一块内存，有利于发挥缓存的优势。embstr为只读的。

当int通过APPEND添加字符串值后，会变为raw编码，当embstr修改后，即更改为raw。
### 2.列表对象
列表对象的编码方式为ziplist或者linkedlist。如果为linkedlist，那么每个链表的元素为字符串对象。
ziplist采用压缩列表(数组链表)作为底层实现，linkedlist采用链表作为实现。
当字符串元素长度小于64Byete，或者元素个数小于512个，采用ziplist，否则采用likedlist。
### 3.哈希对象
编码方式可以为ziplist或者hashtable。
* ziplist
采用压缩列表，按照keyi valuei的顺序储存
* hashtable
采用字典实现，key和value均为字符串对象

当key和value都小于64字节，或者key-value对小于512，采用ziplist，否则为hashtable。
### 4.集合对象
编码方式为instset或者hashtable。
### 5.有序集合
编码方式为ziplist或者skiplist。
* ziplist
按分值高低进行顺序存储
* skiplist
使用zset作为底层实现，包含了一个字典和一个跳跃表。

skiplist设计时，采用跳跃表来达到范围操作的目的，使用字典来使得获取对象分值为O(1)时间。
## 数据库
### 1.过期建删除策略
* 定时删除
创建timer，在timer到期时，对建进行删除操作。这种策略对内存最友好，因为可以保证到期键尽快的被删除，但是有可能在CPU时间紧张的情况下，浪费时间在删除键中。
* 惰性删除
只有在取出过期键时，才会删除。这种方案是CPU友好，但是内存不友好的。
* 定期删除
每隔一段时间进行一次删除过期建的操作，通过设定一段时间，来减少对CPU的影响，并通过定期删除，减少内存占用问题，但是选定定期的时长需要合适。

Redis选用惰性删除+定期删除，定期策略通过每次随机抽取若干个键，并删除其中的过期建完成。通过两种策略配合，在CPU和内存中达到平衡。
## RDB持久化
Redis是内存数据库，一旦服务器进程退出，那么所有状态都消失不见。为了解决这一问题，Redis提供了RDB持久化功能，可以将内存数据写入硬盘中。
### 1.RDB文件创建与载入
SAVE命令阻塞Redis服务器进程，直到RDB文件创建完毕，而BGSAVE会派生出子进程进行创建RDB文件，而父进程继续响应请求。在保存时，过期键不会被保存。
在启动时会自动载入RDB文件，如果此时服务器是主服务器模式，那么过期的会被忽略，如果是从服务器，所有都会被载入。并且由于AOF更新频率比RDB高，如果服务器开启了AOF持久化的功能，会优先使用AOF文件还原。
在执行BGSAVE期间，SAVE和BGSAVE都会被拒绝(为了避免竞争)。同时BGSAVE和BGREWRITEAOF也不会一起执行(他们会同时执行大量写磁盘操作)。
### 2.RDB文件结构
![RDB文件结构](/assets/RDB文件结构.png)
RDB文件以"REDIS"五个字符开头，程序可以通过这个，快速检查载入的是否为RDB文件。
db_version为4字节，标识了RDB的版本号。
databases包含了任意多个数据库，以及数据库中的键值对。
EOF常量长一个字节，标识正文结束。
check_sum是校验和，检测文件是否受损。
![RDB数据库结构](/assets/RDB数据库结构.png)
SELECT常量为一个字节，这意味着接下来读入的是数据库号码。
db_num保存数据库号码。
key_value_pairs保存键值对，由下图形式构成。
![pairs](/assets/pairs.png)
### 3.AOF持久化
AOF是将Redis命令请求进行保存，启动时，通过载入AOF文件命令来恢复数据库状态。
AOF持久化通过命令追加、文件写入和文件同步三部分实现：
* 命令追加
在执行完一个命令后，服务器会以协议格式将执行的命令追加到aof_buf缓冲区的末尾。
* AOF文件的写入和同步
在每个事件循环结束后，考虑是否需要将aof_buf缓冲区的内容写入并同步到AOF文件

AOF数据还原通过构建一个fake client，通过不断执行AOF里的指令，直到结束。

随着执行命令的增多，AOF文件会越来越大，为了解决文件过大的问题，Redis提供了文件重写的功能，通过创建一个新的AOF来替代现有的AOF文件(具体的，是扫描当前数据库，通过直接添加现有键值对的方式，即可代替过去历史的所有操作，记录当前数据库状态)。

为了避免重写AOF造成的阻塞，Redis采用子进程进行写(不采用线程是想在不使用锁的情况下，数据安全)。为了解决在重写过程中数据库的变化，Redis设计了一个AOF重写缓冲区，当子进程结束后，父进程会将AOF重写缓冲区的内容写入新的AOF文件。
## 事件
Redis也是基于Reactor模式，当产生响应事件后，通过文件事件分派器(dispatcher)，分发给各个事件处理器。

# Redis实战
[redis阿里面经](https://juejin.im/post/5c9a67ac6fb9a070cb24bf34)

## 1.为什么要用 redis

使用 redis 通常出于两个原因：
* 性能：对于执行时间较久，且变动不频繁的数据，可以缓存到 redis 中，使得请求得以快速响应。
* 并发：在大并发情况下，所有请求直接访问数据库，容易把数据库打挂了。

## 2.redis 为什么这么快

* 纯内存操作，自然比 mysql 存到磁盘上快很多
* IO 多路复用机制
* 由于瓶颈通常不再 cpu，而受限于网络等，因此采用了单线程方案，避免了上下文切换

## 3.redis 删除策略

* 过期键删除
	定时删除+过期删除，如果采用定时器删除，虽然对内存有好，但是需要维护定时器，对 CPU 反而不太有好。  

	定期删除，redis默认每个100ms检查，是否有过期的key,有过期key则删除。需要说明的是，redis不是每个100ms将所有的key检查一次，而是随机抽取进行检查(如果每隔100ms,全部key进行检查，redis岂不是卡死)。因此，如果只采用定期删除策略，会导致很多key到时间没有删除。 于是，惰性删除派上用场。也就是说在你获取某个key的时候，redis会检查一下，这个key如果设置了过期时间那么是否过期了？如果过期了此时就会删除。
* 缓存淘汰策略
	超过内存大小限制的话，会有逐出策略，一般采用 lru。

## 4.数据库和缓存双写方案

1. 先更新数据库，再更新缓存(非常不推荐)

	感觉这基本是最坑的方案了。。。因为
	同时有请求A和请求B进行更新操作，那么会出现
	（1）线程A更新了数据库
	（2）线程B更新了数据库
	（3）线程B更新了缓存
	（4）线程A更新了缓存
	数据库里最后的是 B 的数据，缓存是 A 的数据，缓存里的是脏数据。	
2. 先删除缓存，再更新数据库

该方案会导致不一致的原因是。同时有一个请求A进行更新操作，另一个请求B进行查询操作。那么会出现如下情形:
（1）请求A进行写操作，删除缓存
（2）请求B查询发现缓存不存在
（3）请求B去数据库查询得到旧值
（4）请求B将旧值写入缓存
（5）请求A将新值写入数据库
上述情况就会导致不一致的情形出现。而且，如果不采用给缓存设置过期时间策略，该数据永远都是脏数据。(使用从库，也会有这个问题)

可以采用双删策略解决：
（1）先淘汰缓存
（2）再写数据库（这两步和原来一样）
（3）休眠一定时间秒，再次淘汰缓存
这么做，可以将一定时间内所造成的缓存脏数据，再次删除。

3. 先更新数据库，再删除缓存

假设这会有两个请求，一个请求A做查询操作，一个请求B做更新操作，那么会有如下情形产生
（1）缓存刚好失效
（2）请求A查询数据库，得一个旧值
（3）请求B将新值写入数据库
（4）请求B删除缓存
（5）请求A将查到的旧值写入缓存
ok，如果发生上述情况，确实是会发生脏数据。

但是要求写数据库的时间比读操作更短，才能先删除，再读入旧的缓存，但是数据库一般来说读的速度快于写。

但是，如果删除缓存操作失败的话，那么就会出现脏数据的情况了，可以添加重试机制来解决，但是这种方案在业务代码里加入了大量的额外操作。
![消息队列重试策略](https://images.cnblogs.com/cnblogs_com/rjzheng/1202350/o_update1.png)

4. 缓存穿透

缓存穿透是指查询一个一定不存在的数据，由于缓存是不命中时被动写的，并且出于容错考虑，如果从存储层查不到数据则不写入缓存，这将导致这个不存在的数据每次请求都要到存储层去查询，失去了缓存的意义。在流量大时，可能DB就挂掉了，要是有人利用不存在的key频繁攻击我们的应用，这就是漏洞。

缓存穿透可以会在接口层增加校验，比如用户鉴权校验，参数做校验，不合法的参数直接代码Return，比如：id 做基础校验，id <=0的直接拦截等。

还可以用布隆过滤器做检验，因为可以明确的判断出不再数据库中，不存在可以直接返回

5. 缓存雪崩

缓存雪崩是指在我们设置缓存时采用了相同的过期时间，导致缓存在某一时刻同时失效，请求全部转发到DB，DB瞬时压力过重雪崩。

给缓存的失效时间，加上一个随机值，避免集体失效。

6. 缓存击穿

对于一些设置了过期时间的key，如果这些key可能会在某些时间点被超高并发地访问，是一种非常“热点”的数据。这个时候，需要考虑一个问题：缓存被“击穿”的问题，这个和缓存雪崩的区别在于这里针对某一key缓存，前者则是很多key。

使用互斥锁，当缓存失效时，先去获得锁，再更新缓存，这样后续线程发现缓存已经更新，就使用缓存了。

## pipeline 和 mget

1. pipeline

pipeline 和浏览器管线化是一个意思，都是说客户端缓存一部分命令，然后一起发送，服务端按序依次返回，可以节省往返时间

2. mget

发送类似 pipeline，proxy 缓存多个 key 的结果，最后一起返回

比起 pipeline 更进一步节省 RTT，但是 proxy 需要缓存，且 mget 延时是最后一个 key 的回复时间，前面的 key 平白进行了等待。