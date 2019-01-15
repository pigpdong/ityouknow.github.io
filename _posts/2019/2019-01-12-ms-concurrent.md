---
layout:     post
title:    轻松构建微服务之并发
no-post-nav: true
category: other
tags: [arch]
excerpt: thread lock
---

## 前言







并发
CAS 无锁队列

每个Monitor在每一个时刻只能被一个线程拥有，也就是只有一个Active Thread,而其他线程都是Waiting线程，分别存放在两个队列里，一个Entry Set,一个Waiting Set,
在Entry Set里的线程就是blocked的，从jstak的dump来看就是 waiting for monitor entry, 在Waiting Set里的状态就是 Waiting, 在jstack里dump出来就是Waint in Object.wait()

waiting是线程主动调用object.wait，sleep,join,等待某一个事情结束，例如等待object.notify等,sleep不会释放自己持有的锁，而wait会释放对象锁，join是通过thread对象的wait方法，然后等线程执行结束会执行notifyall
runnning状态的线程调用yield会变成runnable可以运行，


wait和notify必须在synchronized语句块内，这三个操作必须针对的是同一个对象的监视器，wait之后其他线程可以进入同步块，因为wait会释放监视器，如果某代码不持有对象监视器直接调用wait则会报错IllegalMonitorStateException,这个异常经常出现在synchronized语句块内去调用另外一个对象的wait方法


volatile变量：在处理线程的时候每次都要从主内存load到工作线程，让后写入工作线程，过一段时间在save到主内存，而volatile修饰的变量每次修改都会先从主内存load写入工作内存后立即save到主内存

thread.intrrupt（）只是修改了一个状态标示，而这个标示只有在wait sleep状态，可以用 thread.interrupted()循环判断来退出，

ThreadLocal 线程本地变量，每个线程维护一份，每一个Thread内部都会维护一个ThreadLocalMap变量，ThreadLocalMap本身是一个容器，一个线程内可以存储多个不同的ThreadLocal变量，所以需要一个map来存储，而这个map的key就为ThreadLocal对象本身，value为一个内部封装好的value对象，ThreadLocalMap内的entry是一个弱引用对象

Atomic相关类采用Unsafe里面的方法来根据字段在对象中得偏移量来设置值的方法，cas操作来保证原子，但是 AtomicReference可能出现对象地址相同但是属性对象不同

juc里面提供的lock，比synchronized更灵活，同步块要后加的锁先释放，而lock没有锁的释放的顺序要求，而lock还提供了trylock等无阻塞的方式，等待，以及课中断等，性能也更高，
lock和lockInterruptily区别是 线程a获取到锁后，如果线程B通过r.lockInterruptily()去获取锁，由于A还没有释放，所以B只能block等待，但是线程B此时可以调用inerrupt方法中断，退出阻塞不在争抢资源
