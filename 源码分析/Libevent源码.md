<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Libevent源码阅读笔记](#libevent源码阅读笔记)
	- [1.简单例子](#1简单例子)
	- [2.事件Event](#2事件event)
		- [2.1Event结构](#21event结构)
		- [2.2Event相关设置](#22event相关设置)

<!-- /TOC -->
# Libevent源码阅读笔记
libevent是一个轻量级的基于Reactor的高性能网络库，今天开始阅读libevent源码，希望自己能以比较系统的角度阅读，学习设计思想方面的问题，不要太过纠结于细节(当然，好的技巧肯定也需要学习)。采用的版本是1.4，一方面是网上资料比较多，另一方面也考虑早期的可能会更精简一些。

## 1.简单例子
  ```c
  #include <stdio.h>
  //使用libevent库所需头文件  
  #include <event.h>  

  struct event ev; //事件实例
  struct timeval tv; //超时时间

  void on_time(int sock,short event,void *arg)  
  {  
      printf("hello world\n");  


      // 事件执行后,默认就被删除,所以需要重新添加  
      event_add(&ev, &tv);  
  }  

  int main()  
  {  
      //  实例化一个Reactor
      struct event_base* base = event_init();   

      //  设置定时器回调函数，绑定到current_base，这里只有一个
      evtimer_set(&ev, on_time, NULL);  

      //1s运行一次func函数
      tv.tv_sec = 1;  
      tv.tv_usec = 0;  

      //添加到事件循环中
      event_add(&ev, &tv);  

      //事件循环，没有事件会return
      event_base_dispatch(base);

      return 0;  
  }  
  ```
libevent采用Reactor模式，上述简单的例子描绘了libevent简单的使用流程，首先要实例化一个Reactor，然后设置回调，并且绑定到Reactor中，接着添加事件到事件队列，然后开始事件循环。效果如下，每个1s触发一次事件。
![libeventdemo](/assets/libeventdemo.png)
## 2.事件Event
### 2.1Event结构
Event的定义如下：
```c
struct event {
	TAILQ_ENTRY (event) ev_next; //该节点在注册IO事件中的位置
	TAILQ_ENTRY (event) ev_active_next; //激活链表中的位置
	TAILQ_ENTRY (event) ev_signal_next; //信号事件位置
	unsigned int min_heap_idx;	//小根堆中的索引

	struct event_base *ev_base; //属于的Reactor

	int ev_fd; //绑定的文件描述符，或者信号
	short ev_events; //关注的事件
	short ev_ncalls; //执行回调的次数，通常为1
	short *ev_pncalls;	//指向ncalls，或为NULL

	struct timeval ev_timeout; //超时值

	int ev_pri;		/* smaller numbers are higher priority */

	void (*ev_callback)(int, short, void *arg); //回调
	void *ev_arg; //给回调传入的参数

	int ev_res;		//当前激活的事件
	int ev_flags; //标识event的状态
};
```
具体见注释，有几点需要注意：
* 事件类型
  ```c
  #define EV_TIMEOUT	0x01
  #define EV_READ		0x02
  #define EV_WRITE	0x04
  #define EV_SIGNAL	0x08
  #define EV_PERSIST	0x10	/* Persistant event */
  ```
用每一位来表示一个事件，可以方便的开关某一位，并且可以通过|操作设置多个类型。注意，信号和IO不能同时设置。
* 链表
  三个队列都是用同一类链表结构实现的，定义如下：
  ```c
  #define	_TAILQ_ENTRY(type, qual)					\
  struct {								\
  	qual type *tqe_next;		/* next element */		\
  	qual type *qual *tqe_prev;	/* address of previous next element */\
  }
  #define TAILQ_ENTRY(type)	_TAILQ_ENTRY(struct type,)
  ```
  可以看到，这个链表定义next节点指向的是下一个节点，但是prev节点是一个二级指针。看到这里对这部分感到很困惑，不理解为什么要用一个二级指针指向前驱。
  ![TAILQ](/assets/TAILQ.png)
  经过查阅资料，TAILQ队列是FreeBSD内核中的一种队列数据结构，没有像Linux那样，只是将list挂接到某个struct中，因此TAILQ是含有对象信息的。为了方便管理，TAILQ建立了相应抽象的头结点，只含有first和last节点。那么如果我们采用一级指针的方式，第一个元素的prev就没有地方可以指向(因为头节点和元素不是一个类型，这会让我们操作头元素可能需要特殊处理)
  ```c
  #define	TAILQ_NEXT(elm, field)		((elm)->field.tqe_next)
  #define	TAILQ_PREV(elm, headname, field) \
	(*(((struct headname *)((elm)->field.tqe_prev))->tqh_last))
  ```
  因为这种特殊的结构，TAILQ访问后继和前驱的效率相差很大，后继只要找到filed的next就行。而前驱需要找到当前节点前驱的前驱的next再解引用，要复杂的多。
### 2.2Event相关设置
通过event_set可以设置event对象
```c
void
event_set(struct event *ev, int fd, short events,
	  void (*callback)(int, short, void *), void *arg)
{
	/* Take the current base - caller needs to set the real base later */
	ev->ev_base = current_base;

	ev->ev_callback = callback;
	ev->ev_arg = arg;
	ev->ev_fd = fd;
	ev->ev_events = events;
	ev->ev_res = 0;
	ev->ev_flags = EVLIST_INIT;
	ev->ev_ncalls = 0;
	ev->ev_pncalls = NULL;

	min_heap_elem_init(ev);

	/* by default, we put new events into the middle priority */
	if(current_base)
		ev->ev_pri = current_base->nactivequeues/2;
}
```
可以看到，even_set是把event注册到current_base中取的，可以通过event_base_set重新进行注册。在demo中，我们只有一个Reactor实例，所以设不设置都可以。
## 3.Reactor框架
### 3.1event_base结构
在libevent中，Reactor表现为event_base结构体，声明如下：
```c
struct event_base {
	const struct eventop *evsel; //复用方式封装，select, epoll等
	void *evbase; //复用采用方式封装的对象
	int event_count; //事件总数
	int event_count_active; //激活事件数

	int event_gotterm;		/* Set to terminate loop */
	int event_break;		/* Set to terminate loop immediately */

	/* active event management */
	struct event_list **activequeues; //事件优先级队列
	int nactivequeues; //

	/* signal handling info */
	struct evsignal_info sig; //信号相关

	struct event_list eventqueue; //事件列表，保存所有事件
	struct timeval event_tv; //

	struct min_heap timeheap; //定时器小根堆

	struct timeval tv_cache;
};
```
具体见注释，evsel指向的是eventop形成的数组，数组中封装了多种复用方式，包括select,epoll等。

### 3.2初始化event_base
跟demo中一样，程序需要通过调用event_init来，当中调用event_base_new创建event_base对象，下面列出event_base_new的重要片段:
```c
if ((base = calloc(1, sizeof(struct event_base))) == NULL) //从堆中申请一个event_base结构
		event_err(1, "%s: calloc", __func__);
gettime(base, &base->event_tv); //获取时间
min_heap_ctor(&base->timeheap); //初始化定时器堆
TAILQ_INIT(&base->eventqueue); //初始化事件队列

//早到最前的eventops，并进行初始化
base->evbase = NULL;
for (i = 0; eventops[i] && !base->evbase; i++) {
	base->evsel = eventops[i];

	base->evbase = base->evsel->init(base);
}
```
具体就是申请一个新的event_base，然后对时间、队列等进行初始化，最重要的，在eventops中挑选出第一个有效的，进行初始化。
### 3.3接口
Reactor需要提供事件注册，注销的接口，然后根据事件复用接口进行时间循环，当事件就绪时，执行回调函数。
* 注册事件
```c
int
event_add(struct event *ev, const struct timeval *tv)
{
	struct event_base *base = ev->ev_base; //要注册到的reactor
	//reactor采用的复用操作，evsel配合evbase
	const struct eventop *evsel = base->evsel;
	void *evbase = base->evbase;
	int res = 0;

	//新的timer事件，对堆进行一些操作
	if (tv != NULL && !(ev->ev_flags & EVLIST_TIMEOUT)) {
		if (min_heap_reserve(&base->timeheap,
			1 + min_heap_size(&base->timeheap)) == -1)
			return (-1);  /* ENOMEM == errno */
	}

	//新的event，调用evsel_add接口进行注册
	if ((ev->ev_events & (EV_READ|EV_WRITE|EV_SIGNAL)) &&
	    !(ev->ev_flags & (EVLIST_INSERTED|EVLIST_ACTIVE))) {
		res = evsel->add(evbase, ev);
		if (res != -1) //注册成功，添加到已注册链表
			event_queue_insert(base, ev, EVLIST_INSERTED);
	}

	//添加定时器事件
	if (res != -1 && tv != NULL) {
		struct timeval now;

		//已经添加过了，删除旧的
		if (ev->ev_flags & EVLIST_TIMEOUT)
			event_queue_remove(base, ev, EVLIST_TIMEOUT);

		//已经激活，从激活列表删除
		if ((ev->ev_flags & EVLIST_ACTIVE) &&
		    (ev->ev_res & EV_TIMEOUT)) {
			if (ev->ev_ncalls && ev->ev_pncalls) {
				/* Abort loop */
				*ev->ev_pncalls = 0;
			}

			event_queue_remove(base, ev, EVLIST_ACTIVE);
		}

		//计算定时事件，并插入到小根堆中
		gettime(base, &now);
		evutil_timeradd(&now, tv, &ev->ev_timeout);

	}

	return (res);
}
```
这里有一些对timer事件的处理，因为注册新的timer事件需要申请新的空间，而这是有可能失败的，这里很巧妙的先进行空间申请，成功的话再进行后序处理，将整个时间注册封装为一个类似原子操作的行为，如果定时器注册失败，那么也就不将事件添加到队列中。
* 删除事件
```c
int
event_del(struct event *ev)
{
	struct event_base *base;
	const struct eventop *evsel;
	void *evbase;

	//没有ev_base，也就意味着没有被注册
	if (ev->ev_base == NULL)
		return (-1);

	//取得ev注册的event_base和eventop指针
	base = ev->ev_base;
	evsel = base->evsel;
	evbase = base->evbase;

	//设置回调次数为0
	if (ev->ev_ncalls && ev->ev_pncalls) {
		/* Abort loop */
		*ev->ev_pncalls = 0;
	}

	//从对应队列中删除
	if (ev->ev_flags & EVLIST_TIMEOUT)
		event_queue_remove(base, ev, EVLIST_TIMEOUT);

	if (ev->ev_flags & EVLIST_ACTIVE)
		event_queue_remove(base, ev, EVLIST_ACTIVE);

	//如果是IO或Signal，需要调用IO复用的删除函数
	if (ev->ev_flags & EVLIST_INSERTED) {
		event_queue_remove(base, ev, EVLIST_INSERTED);
		return (evsel->del(evbase, ev));
	}

	return (0);
}
```
## 4. 事件循环
![eventloop](/assets/eventloop.png)
事件循环流程如图，可以看到libevent很好的将IO，Signal以及定时器融合在了一起。
* IO  
对于IO事件，可以很自然的利用描述符和IO复用机制进行配合，没有什么好说的。
* 定时器  
对于定时器事件，libevent通过小根堆进行组织，然后根据小根堆根节点来设置最长的等待时间，通过这种方式和IO复用机制进行配合。这种方式由于poll和epoll只能支持到ms精度，所以定时精度有限，但是定时器的添加删除等操作均能自己定制，比timerfd效率更高。
* 信号  
对于信号本身肯定是不能和IO复用机制进行合作的，但是如果我们能在产生信号后，唤醒IO复用机制，就能将信号和IO复用机制进行联合。
pipe和socketpair都能做到这一点，只要在信号处理函数中，对写端进行写，然后IO复用对读端进行监听即可，步骤如下：
	* epoll进行初始化时，会注册socketpair读fd的事件
	* 注册信号时，会设置信号的回调函数evsignal_handler，在回调中会写入以通知epoll，同时会将信号加入注册队列
	* 当事件发生，调用信号回调，此时epoll会被唤醒
	* 唤醒后，遍历链表，查找signal事件，并将发生事件添加到激活列表
## 5.统一IO复用
首先，通过eventop封装了所有操作的函数指针
```c
struct eventop {
     const char *name;
     void *(*init)(struct event_base *); // 初始化
     int (*add)(void *, struct event *); // 注册事件
     int (*del)(void *, struct event *); // 删除事件
     int (*dispatch)(struct event_base *, void *, struct timeval *); // 事件分发
     void (*dealloc)(struct event_base *, void *); // 注销，释放资源
     /* set if we need to reinitialize the event base */
     int need_reinit;
10};
```
然后，通过一个全局数组eventops储存所有支持的eventop对象，如下
```c
/* In order of preference */
static const struct eventop *eventops[] = {
#ifdef HAVE_EVENT_PORTS
	&evportops,
#endif
#ifdef HAVE_WORKING_KQUEUE
	&kqops,
#endif
#ifdef HAVE_EPOLL
	&epollops,
#endif
#ifdef HAVE_DEVPOLL
	&devpollops,
#endif
#ifdef HAVE_POLL
	&pollops,
#endif
#ifdef HAVE_SELECT
	&selectops,
#endif
#ifdef WIN32
	&win32ops,
#endif
	NULL
};
```
这样，所有支持的操作会按照期望的顺序排列成一个数组。然后，在evenbase初始化时，根据ops进行挑选，并设置。


如果是c++的话，我们可以封装一个eventop的抽象类，然后派生出各个可能用的接口，利用多态，我们可以统一的采用基类指针进行操作。
