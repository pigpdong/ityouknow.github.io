---
layout:     post
title:    轻松构建微服务之并发
no-post-nav: true
category: other
tags: [arch]
excerpt: thread lock
---

## 前言

现代的操作系统都是支持多任务分时技术的,也就是会为每个进程分配一个时间段,在这个时间段内只会有一个进程会获取到CPU的使用权限,时间段结束或者这个获取到CPU使用权的进程主动让出使用权,其他进程才可以争抢CPU使用权,如果一个进程在分配的时间段无法完成,就会将执行的上下文保持起来,下次获取到执行权需要重新加载上下文然后继续执行.
这么看来是不是不会存在并发的场景了,首先现在的机器都会有多个CPU,多个CPU上如果在执行同一个任务,就会出现并发,在同一个CPU上也会有多个任务虽然先后获取到CPU执行权,但是CPU执行的时候需要从内存或者硬盘上加载共享变量,在一个CPU周期没有结束的时候,会将修改的变量保存在上下文中,这个上下文的数据并没有写会主内存,而另外一个任务去主内存获取这个共享变量的时候就发生了并发访问,下面我分析下JVM下处理并发有哪些手段.


> 有些同学可能会有疑问,既然单个CPU在某一个时刻只有一个单任务在运行,那么在java中我起多个线程去执行是否真的比单线程快呢? 如果是计算密集型的,可能确实没有啥优势,但是如果这个任务有大量IO,需要在用户态和内核态进行转换,多线程环境下可以提高数据处理效率.


### object.wait() yield() notify() synchronized

java中每个对象都会持有一个Monitor对象,只有在多线程环境下该Monitor才会起作用,当使用synchronized修饰某个方法,或者用synchronized圈住一片代码的时候,那么运行线程将会尝试获取这段代码所需要的对象的Monitor监视器,如果获取到,就去执行方法体或者代码片段,
如果没有获取到,就会阻塞并等待其他线程释放这个对象的Monitor.

- 如果synchronized加在一个普通方法上就需要获这个方法所在的对象的锁,如下面代码中得test1方法,需要获取ThreadTest这个对象的monitor
- 如果synchronized加在一个static方法上就需要获这个方法所在的对象的类的Monitor,如下面代码中得test4方法,需要获取ThreadTest这个class对象的monitor
- 如果用synchronized圈住一个代码片段,并且用this修饰,如下面代码中的test2,跟test1效果类似,需要获取所执行的代码的所在对象的monitor
- 如果用synchronized圈住一个代码片段,并且用一个对象来修饰,如下面代码中的test3,就需要获取到lock这个对象的lock

```
public class ThreadTest {


    public synchronized void  test1(){
       // do sth ...
    }


    public  void  test2(){
        synchronized(this){
            // do sth ...
        }
    }

    private Object lock = new Object()

    public  void  test3(){
        synchronized(lock){
            // do sth ...
        }
    }

    public static synchronized void  test4(){
        // do sth ...
    }

}

```

wait和notify是每个对象都有的方法,当调用wait()方法后,线程会释放所持有的Monitor锁,然后进入等待状态,这个状态需要其他线程调用notify来唤醒,我们可以将synchronize理解为object对象的lock方法用来阻塞获取锁,
object的wait和notify方法必须在线程获取到这个对象的monitor后才能调用,也就是必须在synchronized语句块内,否则的话会抛出IllegalMonitorStateException异常.例如以下代码

```
public class ThreadTest {




    private Object lock = new Object()

    public  void  test3(){
        synchronized(lock){
            // do sth
            this.wait();
        }
    }


}

```

以下为采用object的wait和notify来实现一个线程安全的阻塞队列,可以支持多线程生产和多线程写入
```
private static PriorityQueue<Integer> quene = new PriorityQueue<Integer>();
    private  int size = 100;

    public  void  consumer(){
        synchronized (quene)
        {
            while (quene.size() == 0)
            {
                System.out.println("【消费者】：队列空了，等待生产...");
                try
                {
                    quene.wait();
                }
                catch (InterruptedException e)
                {
                    e.printStackTrace();
                }
            }
            //队列不空：消费一个产品 每次移走队首元素
            quene.poll();
            //如果消生产者等待，唤醒
            quene.notify();
            System.out.println("【消费者】消费了1个产品，当前队列中产品数量：" + quene.size());
        }
    }


    public  void  product() {
        synchronized (quene)
        {

            while (quene.size() == size)
            {
                System.out.println("【生产者】队列满了，等待消费...");
                try
                {
                    quene.wait();
                }
                catch (InterruptedException e)
                {
                    e.printStackTrace();
                }
            }
            //队列未满：生产一个产品
            quene.offer(1);
            //如果消费者处于等待，唤醒
            quene.notify();
            System.out.println("【生产者】生产一个产品，当前队列中产品数量：" + quene.size());
        }
    }
```

object.wait() 和 notify 是最原始的同步技术,现在jdk1.5以后已经提供更易用的锁的实现,以及读写锁,条件锁等,稍后我们在比较两者的区别


### 自适应锁

当一个代码被synchronized修饰时,编译而成的字节码中就会包含一个Monitorenter和多个MonitorExit,为什么有多个exit,是因为jvm需要保证在正常执行和异常执行的情况下都可以释放锁,
这样一个enter和exit的过程我们可以简单地看成一个锁计数器和一个持有该锁的线程的指针,当执行monitorenter的时候,如果目标锁对象的计数器为0则说明没有被其他线程占有,java虚拟机会将该锁对象的持有线程指向当前线程,并将计数器加1.
当目标锁对象的计数器不为0的情况下,如果锁对象的持有线程是当前线程,则将计数器加1,否则就需要等待直到持有线程释放该锁,当进入monitorexit的时候,锁对象的计数器减1,当减为0就代表该锁已经被释放了.

java线程的阻塞和唤醒都是需要操作系统来完成的,这些操作需要涉及系统调用,需要在用户态和内核太进行切换开销很大,为了尽量避免线程的昂贵的阻塞和唤醒的开销,java虚拟机会在线程被阻塞之前,以及被唤醒但是还没竞争到锁的情况下,进入自旋状态,在处理器上空跑并轮询锁是否已经释放,如果此时锁正好释放了就不需要阻塞直接获取到锁,这两种方式分别指代jvm中的重量级锁和轻量级锁,
也是悲观锁和乐观锁,在自旋状态下的锁会导致获取锁的时候没有严格按照,先到先获取锁的原则,所以破坏了锁的公平性,而我们在使用锁的时候并没有api来指明使用哪种锁,jvm默认使用的是自适应锁,自己去学习当前锁竞争的激烈程度以及释放效率,如果竞争的线程很少而且释放很快就用轻量锁,否则就用重量锁.


### 线程的状态

java中的线程有以下几种状态

- 1.NEW 一个线程创建后,还没有执行下地状态,我们 new Thread() 方法获取到得thread对象默认就是这个状态
- 2.RUNNABLE ,当调用线程的thread.start()方法后就是这个状态,表示这个线程可以被调度执行,正在等待cpu
- 3.BLOCKED 当一个线程需要等待获取Monitor的时候就会进入阻塞状态,当我们调用一个被synchronized修饰的方法的时候,就会进入这个状态,当这个歌线程获取到对应的monitor后,就会回到RUNNABLE状态
- 4.WAITING 当一个线程在获取到Monitor之后,调用object.wait(),或者调用LockSupport.park()方法的时候,会进入这个状态,这个状态下线程会让出锁并且让出CPU,当然线程的join也是通过wait和notify实现,当调用object的notify方法,会唤醒线程使其进入RUNNABLE状态,然后等待CPU调度
- 5.TIMED-WAITING 这个状态和WAITING状态类似,只是需要调用一个带超时时间的wait方法或者sleep方法,sleep方法会让出CPU但是不会让出锁
- 6.TERMINATED 线程中断,注意blocked状态下的线程无法被中断的,只有从RUNNABLE 和 WAITING,TIMED-WAITING 下才能被中断,线程的中断只是线程内部一个变量进行标记,当线程正常运行的时候,即使设置了中断状态线程还是会继续正常执行直到结束,当线程进入wait状态后,就会抛一个异常,然后线程结束

![](https://pigpdong.github.io/assets/images/2019/thread/threadstate.png)

> running状态下的线程调用yield()会让出CPU,会到runnable状态

> thread.intrrupt（）只是修改了一个状态标示，而这个标示只有在wait sleep状态下才会抛出异常然后线程终止，可以用 while(thread.interrupted())循环判断来优雅退出.

> thread.sleep() 线程会让出CPU时间,但是不会让出监视锁

> 使用thread最好设置下线程名字和线程组 thread-name,thread-group,一般都建议用ThreadFactory来创建线程

> 每个Monitor在每一个时刻只能被一个线程拥有，也就是只有一个Active Thread,而其他线程都是Waiting线程，分别存放在两个队列里，一个Entry Set,一个Waiting Set,
  在Entry Set里的线程就是blocked的，从jstak的dump出来的线程状态来看就是waiting for monitor entry, 在Waiting Set里的状态就是 Waiting, 从jstack里dump出来就是Wait in Object.wait()

![](https://pigpdong.github.io/assets/images/2019/thread/entryset.jpeg)

### ThreadLocal

线程本地私有变量,每个线程维护一份,会存在整个线程的生命周期,可以在不同上下文间传递,其内部实行原理为:每一个Thread内部维护一个ThreadLocalMap变量,ThreadLocalMap本身是一个容器，一个线程内可以存储多个不同的ThreadLocal变量，所以需要一个map来存储，而这个map的key就为ThreadLocal对象本身，value为一个内部封装好的value对象，ThreadLocalMap内的entry是一个弱引用对象

下面我们分析下threadlocal的get方法和set方法,以加深怎么使用threadlocal

```
 public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }

 public void set(T value) {
      Thread t = Thread.currentThread();
      ThreadLocalMap map = getMap(t);
      if (map != null)
             map.set(this, value);
       else
             createMap(t, value);
 }
```

- 1.先通过Thread.currentThread()获取当前线程
- 2.获取thread对象的ThreadLocalMap变量,每个thread都有这个对象
- 3.通过的ThreadLocalMap的getEntry()获取对应的Entry,key为this,也就是这个ThreadLocal对象本身
- 4.通过entry获取对应的value
- 5.set方法和这个类似,只是修改entry里value的值

```
 static class ThreadLocalMap {

        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

我们可以看到ThreadLocalMap里的entry对象是一个WeakReference,弱引用是当GC的时候没有强引用的对象将会被清理,即使这个个对象存在其他对象的弱引用也会被清理,但这个弱引用只是针对MAP里的key,也就是ThreadLocal,但是如果value是一个很大的对象,并且只要当前线程还活着就会一直不会释放value.

![](https://pigpdong.github.io/assets/images/2019/thread/entryset.jpeg)

> 如果不调用threadlocal的remove方法,为什么会发生内存泄露? 如图:每个thread中都存在一个map,map的key是一个threadlocal实例,这个key是一个弱引用指向threadlocal,当把threadlocal设置为null的时候,没有任何引用指向threadlocal,除了entry里的弱引用,所以可以被GC回收,但是value还存在当前线程的强引用,所以无法被清理,只要当前线程一直存活就会一直存在.那么在threadlocal被设置为null的时候到线程结束这段时间内就发生了内存泄露.

> 为了防止使用者,没有主动调用remove方法,threadlocal在调用get和set方法的时候会主动去检查key是否为空,如果为空就会清除掉value,就怕使用者在创建了threadlocal后又不在使用get,set方法,尤其线程池中一个线程的生命周期会很长.

> threadlocal使用案例,dubbo用来存储RpcContext,而请求的结果,上下文都封装在RpcContext中,spring-tx将同一个事物使用的connection放入threadlocal,用来保证事物内的多个sql处理使用的是同一个connection.

### volatile

在java的内存模型中,存在主内存,以及工作内存,分别对应堆和线程栈,线程读写数据的时候,需要先将变量从主内存load到线程栈空间,写完在save回去,而多线程环境下,load和save期间可能数据已经被其他线程更改,而volatile修饰的变量,每次针对该变量的读取都会触发一次load操作,写入触发一次save操作,而普通变量在线程栈内的读取只会触发一次,而写入会在不确实的时间才进行save操作.volatile不能保证对该变量的非原子操作也是线程安全的,例如i++.

### 线程池

### ScheduledExecutorService

### future

### Lock

### condition

### Atomic

### LockSupport

### CountDownLatch

### Semaphore

### 阻塞队列

### 线程安全的集合

### RingBuffer

### Disruptor


并发
CAS 无锁队列


waiting是线程主动调用object.wait，sleep,join,等待某一个事情结束，例如等待object.notify等,sleep不会释放自己持有的锁，而wait会释放对象锁，join是通过thread对象的wait方法，然后等线程执行结束会执行notifyall
runnning状态的线程调用yield会变成runnable可以运行，


wait和notify必须在synchronized语句块内，这三个操作必须针对的是同一个对象的监视器，wait之后其他线程可以进入同步块，因为wait会释放监视器，如果某代码不持有对象监视器直接调用wait则会报错IllegalMonitorStateException,这个异常经常出现在synchronized语句块内去调用另外一个对象的wait方法


volatile变量：在处理线程的时候每次都要从主内存load到工作线程，让后写入工作线程，过一段时间在save到主内存，而volatile修饰的变量每次修改都会先从主内存load写入工作内存后立即save到主内存

thread.intrrupt（）只是修改了一个状态标示，而这个标示只有在wait sleep状态，可以用 thread.interrupted()循环判断来退出，


Atomic相关类采用Unsafe里面的方法来根据字段在对象中得偏移量来设置值的方法，cas操作来保证原子，但是 AtomicReference可能出现对象地址相同但是属性对象不同

juc里面提供的lock，比synchronized更灵活，同步块要后加的锁先释放，而lock没有锁的释放的顺序要求，而lock还提供了trylock等无阻塞的方式，等待，以及课中断等，性能也更高，
lock和lockInterruptily区别是 线程a获取到锁后，如果线程B通过r.lockInterruptily()去获取锁，由于A还没有释放，所以B只能block等待，但是线程B此时可以调用inerrupt方法中断，退出阻塞不在争抢资源
