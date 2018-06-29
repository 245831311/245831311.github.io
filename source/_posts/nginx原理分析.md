---
title: nginx原理分析
date: 2017-12-05 12:00:00
categories: 
- nginx总结
tags:
- 总结
---
**导语:**
>本章主要记录一下我对nginx原理的理解，以及相关的重点与难点

### 一.nginx优点

#### IO多路复用（重点）
前一章中提及到了nginx能支撑高并发的一个主要原因就是使用到io多路复用的这种技术。

1.  什么是io多路复用？
定义上：获取并监听多个fd(文件描述符)，通过单个线程内完成IO的读写,复用指的是线程
例子:某快递公司快递员（<font color="red">处理线程</font>），寄件（<font color="red">IO网络请求</font>）
    * 例子一：快递员不知道哪家需要寄件，逐间逐户询问是否需要寄件，这样的话就会造成耗费时间过长，万一某一户出现某些情况，寄件人还未回来，就会一直阻塞。这就是一个典型的单线程阻塞例子。
    * 例子二：快递公司觉得一个快递员收件太慢了，就聘请了多个快递员去收件。这样会造成一个问题就是收件速度确实比以前快，但这也会使快递公司成本变得高，造成不必要的人力浪费。这个就是多线程+阻塞的例子。
    * 例子三：寄件方准本好需要的寄件，然后打电话通知快递员某户寄件已准备好，快递员就上门收取。这样就能用一个快递员高效的收纳寄件。这就是一个io多路复用的例子，打电话通知寄件已准备好其实可以看作io事件准备就绪主动上报的一个过程，程序机制就会对已准备好的事件做一系列的io读写操作。

1. nginx的io多路复用模型有哪些？
nginx中提供了多种io多路复用模型的机制如select,poll,epoll等。目前，linux2.6以上才能使用epoll机制。本质上,select与epoll都是属于非阻塞的同步IO.下面，我简单介绍一下select与epoll的区别。
    - select介绍
    ![selectIO模型图](/images/nginx/select.jpg)
    1. 进程通过select调用，将fd集合从用户态全部复制到内核态当中，这个过程中会阻塞进程。
    2. select会遍历的fd集合，如果存在准备就绪的fd，就会返回可读的指示并将对应连接放入监听队列中(表示监听IO数据)。在第二次遍历如果发现存在已经就绪的数据，则会调用recvfrom的方法,进程会将数据从内核态复制到用户态中并进行对应读写事件处理。处理完毕则会移除对应监听队列的socket。
    3. 在select遍历完所有的fd都没有发现可读的fd后,则会进入休眠期。当有发现新的可读写资源，则会唤醒select进程。如果超过一定超时时间，则会重新被唤醒。重新遍历fd集合。
        <span><font color="red">
        select总结:
            -  每次select调用都需要遍历fd集合。
            -  监视的文件描述符的数量存在最大限制，可以调节，但是随着增大也会导致用户态与内核态之间的复制带来的资源开销（如内存）呈线性增大
            -  每次都要将整个集合的数据在用户态与内核到之间的互相复制。
        </font></span>
        
    - epoll介绍
    epoll的核心主要是三个函数:epoll_create(创建epoll句柄)，epoll_ctl(管理epoll注册监听事件，增删查改等)，epoll_wait(等待执行事件)
    4. 首先调用epoll_create创建一个epoll句柄对象，epoll含有一个红黑树和一个就绪队列。
    5. 当调用epoll_ctl创建注册监听事件时候，会为fd注册一个回调函数，并将fd从用户态复制到内核态。如果fd就绪后，会通过回掉函数将fd放入就绪队列中。
    6. epoll_wait则会扫描就绪队列并执行相应事件,并把数据返回给用户。与select类似都是苏醒与睡眠交替进行，但是select是不停遍历整个fd集合，而epoll_wait只是遍历就绪列表就可以了，仅判断是否为空就OK拉。
> 总结：  
简单类比一下，其实select模型与epoll模型好比如在茶楼结账时候，服务员只告诉老板有人结账了，你过去收。此时老板并不知道具体要结账的是哪一桌，就一个一个去问（遍历fd集合是否有就绪），这是select模型做的；而epoll则是服务员具体告诉老板是那几桌要结账（利用回调函数，将就绪事件直接放到epoll队列中），老板就能直接过去结账。

1. io多路复用的优缺点?

| 优点 | 缺点 |
| :---: | :---: |
|单线程，消耗少|不适宜处理少量的请求数|
|能支撑大并发量的请求|
|非阻塞|

#### cpu的亲和性
能够将nginx上的worker绑定到某一特定的cpu上，这样大大避免了进程中缓存的丢失以及切换带来的资源消耗。而且，nginx中还会将某些字符串转换成特定的int，再进行比较，减少了指令条数（如nginx在解释请求头中的method信息时候,会将其值（post|get）转换成int类型进行比较）

#### 轻量级
nginx只保留核心功能模块，而且并不多，可进行相应扩展。
nginx模块化开发，比较易读

### 二.nginx 工作原理
![nginx工作图解](/images/nginx/worker.jpg)
- master进程工作范围
    -  master负责接收外界的信号并分发给worker各个进程。worker进程会通过抢排斥锁accept_mutex来抢占资源（各个worker抢占资源是有一个算法，能均衡每个worker抢占到的资源）
    -  master管理和监控着各个worker的运行状态。
    -  master加载配置文件(如执行重启，关闭操作)。如重启操作，首先重新加载配置文件，fork出新的worker并通知旧的worker停止接收资源，等到旧worker已经完成了剩下的请求后就退出。
- worker进程工作范围
    -  每个抢到互斥锁的worker进程会进行读取请求、解释请求、处理请求、产生数据、返回客户端的步骤。

### 三.nginx中对connection流程
<center>

```flow
st=>start: 开始
step1=>operation: 客户端与nginx通过三次握手tcp连接,并创建好socket
step2=>operation: worker抢占锁并将connection封装到ngx_connection_t实体中
step3=>operation: 将socket设置好读写事件
step4=>operation: 与其他server创建连接与创建
ngx_connection_t实体并设置好读写事件

step5=>operation: 执行读写事件，完成后nginx与客户端交换数据
step6=>operation: 释放ngx_connection_t,nginx或客户端主动关掉连接
cond=>condition: 是否有其他server(如upstream等)
e=>end: 结束
st->step1
step1->step2
step2->step3->cond
cond(yes)->step4
cond(no)->step5
step4->step5->step6->e
```
</center>

### 四.总结
以上是经过我这一段时间对nginx的了解作出的总结，可能会有一些偏差或者不够完善的地方。在今后会不停地去反复完善，希望能将nginx理解得更为透彻。

### 五.参考
1.  [io模式与IO](https://www.cnblogs.com/zingp/p/6863170.html)
2.  [select模型的说明](https://www.jianshu.com/p/edb9ddd51c3d)
3.  [nginx从入门到精通](http://tengine.taobao.org/book/)

