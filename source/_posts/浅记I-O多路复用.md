---
title: 浅记I/O多路复用
date: 2022-09-15 11:24:36
tags:
- 操作系统
- I/O多路复用
categories: 操作系统
comments: true
mathjax: true
---

​	传统的I/O模型是一种1对1的模型，即单个进程只在单个文件描述符上进行I/O操作，如果读缓冲区为空或者写缓冲区满，那么调用`read()`或者`write()`时就会阻塞直到有数据可读或者有空间可写。如果一个服务端连接了多个客户端，某个客户端在进行I/O时发生了阻塞，那么别的客户端就也需要等待，这样就造成了很大的影响，所以就希望能以非阻塞的方式进行I/O，即如果我们的I/O调用不能立刻完成的话，就返回错误而不是阻塞进程去等待。很容易地就想到采用多进程的方式，但是为父进程为每个文件描述符或者每个客户端fork一个子进程的话，又需要花费很多的资源，而且进程上下文的切换涉及到用户空间的和内核空间资源的切换，维护的代价也很大。<!--more-->既然多进程模型过重的话我们呀很容易再去考虑比进程轻量许多的多线程模型，服务端每与一个客户端连接时，就会通过`pthread_create()`函数来创建线程，将`已连接socket`的文件描述符传递给线程，通过这个线程来与客户端进行通信。虽然线程所消耗的资源没有进程多，但是如果连接很多的话，频繁的线程创建销毁及上下文切换也是会造成不小的系统开销的，虽然可以通过**线程池**来维护线程的创建销毁，但是为了避免线程之间的竞争在操作的时候又需要加锁，这又会使得编程的工作变的复杂许多。

## I/O多路复用

​	基于上面的问题，就有了I/O多路复用这么一个模型出来。简单的来说，I/O多路复用就是让一个进程同时监听多个socket，也就是允许进程同时检查多个文件描述符来找出是否有可执行I/O操作的。而比较经典的I/O多路复用接口便是`select()`、`poll()`和`epoll()`，其中epoll()是Linux2.6版本之后专有的。

### select()

​	select()函数实现多路复用的方式是：将用户态里的一组文件描述符集合拷贝到内核中，在内核中检查是否有事件发生，如果发生的话就把对应事件的文件描述符标记为就绪，当整个集合遍历完并且修改完后便把这组文件描述符集合再拷贝到用户空间中，然后在用户空间中去遍历这组文件描述符来看哪些是就绪的。可以看到一个select()方法就要涉及到**2次的文件描述符集合遍历**和**2次文件描述符集合拷贝**了。select()方法函数定义如下：

```c++
int select(int nfds,fd_set *readfds,fd_set *wrtiefds,fd_set *exceptfds,struct timeval *timeout);
//readfds 是用来检测输入是否就绪的文件描述符集合。
//writefds 是用来检测输出是否就绪的文件描述符集合。
//exceptfds 是用来检测异常情况是否发生的文件描述符集合。
//nfds是比上面这3个文件描述符集合中所包含的最大文件描述符号还要大1的数。该参
//数使内核就不用去检查大于这个值的文件描述符号是否属于这些文件描述符集合。
```

​	在**每次调用select时，都需要初始化fd_set**。	

​	select返回0时代表调用超时，返回-1时代表有错误发生，返回一个正整数代表有一个或者多个文件描述符就绪，如果一个文件描述符存在于多个文件描述符集合的话就会被统计多次。还有文件描述符集合的上限通常是1024。

### poll()

​	poll()执行的任务和select是很相似的，区别在于poll()是通过一组文件描述符结构来代替select的三个集合。poll()函数的定义如下：

```c++
int poll(struct pollfd fds[],nfds_t nfds,int timeout);
//fds列出了我们需要poll()来检查的文件描述符。该参数为pollfd 结构体数组
//参数nfds 指定了数组fds 中元素的个数。数据类型nfds_t 实际为无符号整形。
```

pollfd的结构如下：

```c++
struct pollfd{
    int fd;
    short events;
    short revents;
};
```

上面的events和revents都是位掩码。调用者初始化events 来指定需要为描述符fd 做检查的事件。当poll()返回时，revents 被设定以此来表示该文件描述符上实际发生的事件。

![image-20220915122617255](https://typora-oss-pic.oss-cn-guangzhou.aliyuncs.com/typoraImg/image-20220915122617255.png)

​	poll返回正整数时表示有1 个或多个文件描述符处于就绪态了。返回值表示数组fds 中拥有非零revents 字段的pollfd 结构体数量。

​	poll和select一样，只会告诉我们**I/O操作是否会阻塞**，而不是告诉我们到底能否成功传输数据。而且poll也是跟select一样要在用户态和内核态之间拷贝文件描述符集合和线性地遍历哪个文件描述符就绪。当并发数量上去之后，所消耗的时间也会线性地增长，占用CPU的时间也就越多，而造成这两个方法性能不好的原因是它们API设计的局限性：程序重复调用这些系统调用所检查的文件描述符集合都是相同的，可是**内核并不会在每次调用成功后就记录下它们**。poll与select相比，就是没有了文件描述符集合大小的上限，不过随着要检查的文件描述符越多，poll的数据结构也就越大，也就占用更多的CPU时间。

### epoll()

​	针对select和poll的问题，epoll便解决了他们的问题。epoll提供了三个函数，这三个函数分别是`epoll_create()`、`epoll_ctl()`和`epoll_wait()`：

epoll_create:

```c++
//创建一个epoll实例，返回值是这个epoll实例的文件描述符
//Linux2.6.8版本之前该方法有一个参数size来告诉内核应该如何为内部数据结构划分初始大小
int epoll_create();
```

​	用户调用该函数时，内核会创建一个struct eventpoll内核对象，并把它关联到当前进程的已打开的文件列表中。这个eventpoll对象有这么三个成员：

![image-20220915211541276](https://typora-oss-pic.oss-cn-guangzhou.aliyuncs.com/typoraImg/image-20220915211541276.png)

epoll_ctl:

```c++
//修改由文件描述符epfd所代表的epoll实例中的兴趣列表。
//epfd：epoll实例的文件描述符
//op：执行的操作，有EPOLL_CTL_ADD、EPOLL_CTL_MOD和EPOLL_CTL_DEL
//fd：所要操作的文件描述符
int epoll_ctl(int epfd,int op,int fd,struct epoll_event *ev);
```

```c++
struct epoll_event{
    uint32_t events; //事件掩码
    epoll_data_t data; //文件描述符fd的消息
}
```

```c++
typedef union epoll_data{
    void *ptr; //指向用户自定义的数据
    int fd; //注册的文件描述符
    uint32_t u32;
    uint64_t u64;
}epoll_data_t;
```

​	调用`epoll_ctl()`函数注册socket时，以添加监视socket为例子，内核会分配一个红黑树节点，然后将所要等待的事件添加到这个socket的等待队列中，并且设置了回调函数`ep_poll_callback()`，最后将分配好的红黑树节点插入到红黑树中，也就是epoll实例的兴趣列表。上面的回调函数的主要功能是所监视的socket所等待的事件发生时，会将其在兴趣列表上对应的实例添加到就绪链表中，就绪链表不为空时就会唤醒等待着就绪链表的进程，其调用的epoll_wait也就能继续执行下去。

epoll_wait:

```c++
//阻塞等待所监视的文件描述符的事件发生，返回就绪的数目，并且将发生的事件写到evlist中
//epfd：epoll实例的文件描述符
//evlist：有关就绪态文件描述符的信息。数组evlist 的空间由调用者负责申请，所包含的元素个数在参数maxevents 中指定。
//maxevents：返回的就绪events的最大数目
int epoll_wait(int epfd,struct epoll_event *evlist,int maxevents,int timeout);
```

​	**epoll_wait是阻塞**的，当就绪链表没有数据时epoll_wait就会阻塞着去等待，直到超时或者就绪链表非空才继续执行。	就像上面说的epoll有两个存储实例的结构：兴趣列表和就绪链表。`epoll_ctl()`函数则是负责把所要监视的socket加入到兴趣列表，或者是修改其在兴趣列表里的状态，亦或者是将其从兴趣列表删除。**兴趣列表**的结构是一个**红黑树**，红黑树是一个高效率的数据结构，红黑树能让 epoll 在**查找效率、插入效率、内存开销等等多个方面比较均衡**，增删改的时间复杂度通常为O(logn)，而且这颗红黑树是维护在**内核中**的，也就是我们并不需要将其在用户态和内核态之间拷贝来拷贝去的，只需要传入待监视的socket进去就行。就绪链表是一个**双链表**，epoll_wait在检测到就绪事件后就会把最多**maxevents个**就绪事件从内核复制到用户空间中，也就是复制到所传入进来的evlist中。这样系统所遍历的返回的文件描述符都是就绪的，比select和poll遍历全部文件描述符集合高效了许多。上面的过程用一张简单的图来描述就是如下这样：

![image-20220915214425308](https://typora-oss-pic.oss-cn-guangzhou.aliyuncs.com/typoraImg/image-20220915214425308.png)

​	基于epoll的特点，**epoll**的应用场景是在**大量的连接需要监视**但只有**少数的文件描述符处于就绪状态**的情况，这是由于epoll采用的是事件触发机制，如果**大量的连接要处于就绪状态，就会大量地调用回调函数**，回调的过程中又会涉及到兴趣列表(或者说被监视的socket列表)的遍历，这样操作的话效率就不如select和poll直接遍历整个文件描述符集合来获取就绪文件描述符了。相对的说，**select和poll**的应用场景就是**少量的连接但大部分连接都是处于就绪状态**的情况。

### 事件触发

#### 边缘触发(ET——Edge Trigger)

​	当监控的socket上有事件发生时，服务端只会在epoll_wait中苏醒一次收到通知，即使进程没有read读缓冲区或者write写缓冲区，也不会再次通知，**直到该socket的下一个事件发生**。采用边缘触发时，要让程序尽可能地在对应的文件描述符上去执行I/O，比如尽可能地读取字节，所以我们就会**循环**地在文件描述符上执行I/O，如果文件描述符是阻塞的，在没有数据可读写时就会阻塞，程序无法继续进行下去，这也就是为什么**边缘触发通常配合非阻塞I/O**来使用，使文件描述符在得到通知后重复执行I/O操作直到相应系统调用以错误码**EAGAIN**或者**EWOULDBLOCK**的形式失败。

#### 水平触发(LT——Level Trigger)

​	当监控的socket有事件发生而且可以非阻塞地进行I/O调用时，服务端就会在epoll_wait中**不断地苏醒收到通知，直到内核缓冲区的数据读写完毕**。也就是说我们可以在任意时刻检查I/O状态，而且没有必要在文件描述符就绪后尽可能多地执行I/O操作，甚至不进行I/O操作。

​	边缘触发的效率是比水平触发的效率高的，因为**边缘触发下系统不会充斥着大量不关心的就绪文件描述符**，在很大程度上降低了同一个epoll事件被重复触发的次数。select和poll只支持水平触发，而epoll支持边缘触发，但是**epoll默认的触发机制是水平触发**，如果要使用边缘触发，就要在调用`epoll_ctl()`注册时在其epoll_event结构中将events字段指定为**EPOLLET**。

I/O多路复用的整理大概就是这些了，I/O多路复用涉及到的应用其实还是挺多的，就比如Redis，基于内存操作的Redis的瓶颈在网络I/O和内存大小，单线程的Redis通过I/O多路复用监听多个socket从而实现一个Redis线程处理多个I/O流的效果。如果对epoll源码感兴趣的，也可以自行去尝试阅读下，链接：https://github.com/torvalds/linux/blob/master/fs/eventpoll.c。

## 参考

> - [I/O 多路复用：select/poll/epoll](https://xiaolincoding.com/os/8_network_system/selete_poll_epoll.html)
> - [I/O多路复用的实现机制 - epoll 用法总结](https://blog.csdn.net/u010429831/article/details/118600310)
> - [IO多路复用原理&场景 ](https://www.cnblogs.com/zhangyjblogs/p/15862358.html)
> - [图解 | 深入揭秘 epoll 是如何实现 IO 多路复用的！](https://mp.weixin.qq.com/s?__biz=MjM5Njg5NDgwNA==&mid=2247484905&idx=1&sn=a74ed5d7551c4fb80a8abe057405ea5e&chksm=a6e304d291948dc4fd7fe32498daaae715adb5f84ec761c31faf7a6310f4b595f95186647f12&scene=21#wechat_redirect)
