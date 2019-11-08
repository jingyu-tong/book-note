# Redis事件库
![Redis事件循环](/assets/Redis事件循环.png)
* 事件循环
  通过stop控制循环是否停止，beforesleep为事件循环处理之前执行的函数，然后是时间处理函数aeProcessEvents。
![aeProcessEvents声明](/assets/aeProcessEvents声明.png)
* 事件处理
  首先关注最重要的aeProcessEvents，flags表示处理事件的类型，在aeMain中，flags调用为AE_ALL_EVENTS，表示处理所有类型的事件。
  ![时间事件](/assets/时间事件_nqeu1irrc.png)
  * 阻塞时间确定(通过定时器)
    Redis的定时器是通过链表组成的，首先搜索最近到期的定时器(O(n) for list)，然后根据当前时间和到期时间的差值来确定IO复用等待的时间(<0,说明已经到期，则不阻塞)。如果没有定时器事件，则根据flags设定IO复用时间，如果为AE_DONT_WAIT，则不阻塞，否则阻塞到有事件发生。
  ![IO复用](/assets/IO复用.png)
  * IO复用
    接下来，通过IO复用接口以及tvp设置的阻塞事件，获取待处理事件。这里IO复用通过aeApiPoll形式的统一接口进行封装，配合宏定义，使得能够跨平台的同时，能够使用平台下最好的复用技术。
    ![epoll复用](/assets/epoll复用_nlqv266p5.png)
    这里以epoll为例，epoll被唤醒后，对evenLoop中的fired进行设置唤醒描述符和事件。
