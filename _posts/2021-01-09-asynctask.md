---
layout:     post
title:      "Java 线程池"
subtitle:   ""
date:       2021-1-19 12:00:00
author:     "boredream"
catalog: false
header-style: text
tags:
- 线程

---

### 什么是？
当需要异步线程执行的时候，不再新建一条，从一个已有的池子里获取闲置的线程，然后使用之，用完后放回池子。

### 为什么要？
无线创建线程，完成后销毁线程，浪费。线程池可以复用线程。Java线程

### 如何实现？
系统ThreadPoolExecutor类，可以自定义新建对象，也可以Executors工厂类创建fixed/singleThreadPool线程池。  
新建线程池ThreadPoolExecutor的时候，有几个关键参数：
* int corePoolSize 线程数量，及时线程在idle状态也保持。
* int maximumPoolSize 最大线程数量，
* long keepAliveTime 超过core数量、不到max数量的线程，在idle状态后保留多久存活时间
* TimeUnit unit 存活时间的单位
* BlockingQueue<Runnable> workQueue 阻塞队列类型，保存尚未执行的任务用的
* ThreadFactory threadFactory 创建线程工厂类
* RejectedExecutionHandler handler 当加入线程达到队列容量边界时，进行处理

### 具体执行？
按照不同的配置，线程池当前不同状态，新任务进来时会有不同处理
![pool1](https://github.com/boredream/boredream.github.io/blob/master/img/in-post/pool1.png?raw=true)
* < corePoolSize 时，新建个工作线程开始任务。任务结束后线程也保留。
* \> corePoolSize && < maxPoolSize 时，要进一步看阻塞队列情况：阻塞队列是什么类型？阻塞队列是否已满？

### 为什么要有阻塞队列？
如果没有，假设是高频加入且很快结束的任务，就会刚超过core数量，新建工作线程，然后任务结束小于core数量。之后超出的线程可能就要销毁，再次来任务时又会重复，在core最大值边界不停抖动。    
有了队列，一来在这个生产消费者模型中做了层缓存，新来的超出core时，先加入队列~ core里任务完了再从队列取。队列内的数量不停抖动，但工作线程一直保持core最大值不变，不会生成新的线程。二来不同的阻塞队列实现，可以提供更灵活的控制，实现不同功能需求。


### 日常使用？
Executors工厂类生成（实际都是用ThreadPollExecutor实现）

1. newFixedThreadPool(nThreads)  固定大小的线程池。
core和max数量都是nThreads，即不会有超出core不到max生成新线程的情况，固定线程池core数量。
所以也不要超时设为0，阻塞队列用的是LinkedBlockingQueue？

2. newSingleThreadExecutor 单线程池。
参数和fixed的基本一样，只是core和max固定为1。特殊的是外面包了一层
New FinalizableDelegatedExecutorService(new ThreadPoolExecutor(…))

3. newCachedTheadPoll 缓存线程池。
core为0，max为int最大值。超时时间60秒。阻塞队列SynchronousQueue。
线程池容量无限，但都不会一直持有~ 所有线程不使用时只保留60秒。

固定大小的阻塞队列是LinkedBlockingQueue。因为大小固定，所以缓存的队列要这种无限大小的。  
而缓存线程池是没有core的，所以阻塞队列里加东西是没意义的（阻塞队列里是core线程池里取来使用的），所以使用了SynchronousQueue，它是一种线程间移交的机制，不持有任何内容（容量为0），相当于一个桥，在放入一个元素时，需要有另一个等待接收元素的线程，直接移交过去。


### 主流框架源码里使用情况？
#### Retrofit
框架里如果是异步请求的，会以自定义参数新建一个ThreadPoolExecutor
* corePoolSize=0
* maxPoolSize=MAX_INTEGER
* keepAliveTime=60s
* workQueue=SynchronousQueue（同步、不公平即不满足FIFO 的队列）
* threadFactory= new Thread(name:"OkHttp Dispatcher", daemon:false)

解释：
线程池不长期持有线程core=0，但线程池容量不限制（最大值），保活60秒，无特殊需求非公平锁提升效率，Synch同步。线程创建不要守护进程？


> 参考文章：  
> https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html

