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
