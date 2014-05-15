---
layout: post
title: 伟哥教你破解--Java线程谜团
description: Java 线程一直是一个令人头痛的话题，今天我给大家带来一个综合教程，从线程方法/线程同步/线程互斥/线程池/生产者消费者 等多个方面全方位解析Java线程问题。
category: project
---
##Thread类--关键方法
Thread类的关键方法有：  

- yield()   ：让出cpu使用权，但是马上进入可执行状态，给其他线程执行机会、让同等优先权的线程运行（但并不保证当前线程会被JVM再次调度、使该线程重新进入Running状态），如果没有同等优先权的线程，那么yield()方法将不会起作用;  
- sleep(time) ：交出cpu使用权，进入限时等待状态，过了time之后重新进入可执行状态。 作用：保持对象锁，让出CPU，调用目的是不让当前线程独自霸占该进程所获取的CPU资源，以留一定的时间给其他线程执行的机会；  
- start()  ：线程启动，变为可执行状态，也叫就绪状态;  
- run() ：进入执行状态;  
- stop() :结束，进入死亡状态；  
- interrupt() ：interrupt对于一个线程，标示着它应该停下它正在做的事情来做些其他的。具体的是由程序员来决定一个线程如何回应interrupt，一般都是让这个线程终止；  
- join()：“等待该线程终止”，这里需要理解的就是该线程是指的主线程等待子线程的终止。在很多情况下，主线程生成并起动了子线程，如果子线程里要进行大量的耗时的运算，主线程往往将于子线程之前结束，但是如果主线程处理完其他的事务后，需要用到子线程的处理结果，也就是主线程需要等待子线程执行完成之后再结束，这个时候就要用到join()方法了。使用方法：在当前线程A中，有一个B线程的示例线程bt，则在A中执行bt.join()之后，只能等bt执行完才能执行A线程剩下的代码；  
- join(time):跟join()类似，只不过限定了一个时间；  
- currentThread():当前正在执行的线程，使用方法如Thread.currentThread().getName()。  


##Thread状态
java中线程主要有5类状态，其中等待/睡眠/阻塞 我在这里归为一类状态，都为等待或者阻塞状态。  

- 新生状态：刚new出来，还没有run的时候。 
- 就绪状态：执行了start之后，位于就绪队列中，等待分配cpu执行。 
- 执行状态：获得到cpu使用权，进入执行状态。 
- 等待/睡眠/阻塞状态：调用join()/sleep()/wait()或者资源被占用，此时进入阻塞状态。 
- 死亡状态：就是死了。。。 

##线程状态转化图
![线程状态图](/images/projectImage/thread-status.png)

>Java中的多线程是一种抢占机制而不是分时机制。抢占机制指的是有多个线程处于可运行状态，但是只允许一个线程在运行，他们通过竞争的方式抢占CPU。

##线程间的关系
其实Java中的线程间就存在两大类关系：线程互斥 和 线程同步通信，前者通过synchronized解决，后者通过wait()/notify()/notifyAll()来解决。  

1. **线程间互斥**:利用synchronized关键字即可实现，synchronized向jvm申请了一个针对内存对象的锁。  
2. **线程间互相唤醒**: 此时就需要object.wait()和object.notify()了。  

**其中生产者/消费者模式 就融合了这两种情况，我们一会分析到。**

##关于Object类中的wait/notify
线程间的同步要用到wait/notify方法，他们在object类中。Java中每个object对象都有一个互斥锁，线程获取到后就形成了互斥操作，wait/notify操作必须在这个互斥的同步区（synchronized块，以object对象为锁）内执行。
object中的方法：  

- wait()：立即释放对象锁，然后进入等待状态。
- wait(time)：当time时间过去后，就恢复可执行状态了。
- notify()：使得一个waiting状态的线程进入可执行状态。
- notifyAll()：唤醒所以waiting状态的线程。

Obj.wait()，与Obj.notify()必须要与synchronized(Obj)一起使用，也就是wait,与notify是针对已经获取了Obj锁的线程进行操作：  

- 从语法角度来说就是Obj.wait(),Obj.notify必须在synchronized(Obj){...}语句块内。  
- 从功能上来说wait就是说线程在获取对象锁后，主动释放对象锁，同时本线程休眠。直到有其它线程调用对象的notify()唤醒该线程，才能继续获取对象锁，并继续执行。相应的notify()就是对对象锁的唤醒操作。**但有一点需要注意的是notify()调用后，并不是马上就释放对象锁的，而是在相应的synchronized(){}语句块执行结束，自动释放锁后，JVM会在wait()对象锁的线程中随机选取一线程，赋予其对象锁，唤醒线程，继续执行。**这样就提供了在线程间同步、唤醒的操作。

>从上面分析来看，wait()和notify()方法都是释放锁，一个是立即释放锁，另一个是等执行完synchronized代码块后在释放锁，同时后者会通知该互斥锁上的一个wait线程进入可执行状态，notifyAll（）的话就是所有该互斥锁上的进程都被唤醒进入可执行状态。

##sleep() wait()的比较  
  
>  Thread.sleep()与Object.wait()二者都可以暂停当前线程，释放CPU控制权，主要的区别在于Object.wait()在释放CPU同时，释放了对象锁的控制,而sleep则会一直保持它的锁。

##生产者消费者实例
生产者：  
	
    static class Producer implements Runnable { 
            private int start; 
            private int end; 

            Producer(int start, int end) { 
                this.start = start; 
                this.end = end; 
            } 

            @Override 
            public void run() { 
                for (int i = start; i < end; i ++) { 
                    synchronized (basket) { 
                        try { 
                            while (basket.size() == capacity) { 
                                basket.wait(); 
                            } 
                            String p = " PRO" + i; 
                            System.out.println(Thread.currentThread().getName() + p); 
                            basket.add(p); 
                            basket.notifyAll(); 
                            Thread.yield(); // 让出当前线程的执行权,有利于看出交替线程运行的效果 
                        } catch (InterruptedException e) { 
                            e.printStackTrace(); 
                            break; 
                        } 
                    } 
                } 
            } 
        } 

消费者：  

     static class Customer implements Runnable { 
            @Override 
            public void run() { 
                while (true) { 
                    synchronized (basket) { 
                        try{ 
                            while (basket.size() == 0) { 
                                basket.wait(); 
                            } 
                            System.out.println(Thread.currentThread().getName() + basket.remove(0)); 
                            basket.notifyAll(); 
                        } catch (InterruptedException e) { 
                            System.out.println(Thread.currentThread().getName() + "退出"); 
                            break; 
                        } 
                    } 
                } 
            } 
        } 
        

各位客官注意看wait和notifyAll的用法。

##线程池
---
###为什么要用线程池:  
>1.减少了创建和销毁线程的次数，每个工作线程都可以被重复利用，可执行多个任务。  
>2.可以根据系统的承受能力，调整线程池中工作线线程的数目，防止因为消耗过多的内存，而把服务器累趴下。  

###Java线程池类介绍
Java里面线程池的顶级接口是Executor，但是严格意义上讲Executor并不是一个线程池，而只是一个执行线程的工具。真正的线程池接口是ExecutorService。下面这张图完整描述了线程池的类体系结构：  
![线程池](/images/projectImage/java-threadPool.png)  

Java线程池中几个比较重要的类：  

- ExecutorService： 真正的线程池接口。
- ScheduledExecutorService: 能和Timer/TimerTask类似，解决那些需要任务重复执行的问题。
- ThreadPoolExecutor : ExecutorService的默认实现。

线程池类为 java.util.concurrent.ThreadPoolExecutor，常用构造方法为：  

    ThreadPoolExecutor(int corePoolSize, int maximumPoolSize,long keepAliveTime, TimeUnit unit,BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler) 
    

- corePoolSize： 线程池维护线程的最少数量
- maximumPoolSize：线程池维护线程的最大数量
- keepAliveTime： 线程池维护线程所允许的空闲时间
- unit： 线程池维护线程所允许的空闲时间的单位
- workQueue： 线程池所使用的缓冲队列
- handler： 线程池对拒绝任务的处理策略

当一个任务通过execute(Runnable)方法欲添加到线程池时：  

1. 如果运行的线程少于 corePoolSize，则 Executor 始终首选添加新的线程，而不进行排队。（如果当前运行的线程小于corePoolSize，则任务根本不会存放，添加到queue中，而是直接抄家伙（thread）开始运行）  
2. 如果运行的线程等于或多于 corePoolSize，则 Executor 始终首选将请求加入队列，而不添加新的线程。  
3. 如果无法将请求加入队列，则创建新的线程，除非创建此线程超出 maximumPoolSize，在这种情况下，任务将被拒绝。  
 
      public void execute(Runnable command) {
            if (command == null)
                throw new NullPointerException();
            /*
             * Proceed in 3 steps:
             *
             * 1. If fewer than corePoolSize threads are running, try to
             * start a new thread with the given command as its first
             * task. The call to addWorker atomically checks runState and
             * workerCount, and so prevents false alarms that would add
             * threads when it shouldn't, by returning false.
             *
             * 2. If a task can be successfully queued, then we still need
             * to double-check whether we should have added a thread
             * (because existing ones died since last checking) or that
             * the pool shut down since entry into this method. So we
             * recheck state and if necessary roll back the enqueuing if
             * stopped, or start a new thread if there are none.
             *
             * 3. If we cannot queue task, then we try to add a new
             * thread. If it fails, we know we are shut down or saturated
             * and so reject the task.
             */
            int c = ctl.get();
            if (workerCountOf(c) < corePoolSize) {
                if (addWorker(command, true))
                    return;
                c = ctl.get();
            }
            if (isRunning(c) && workQueue.offer(command)) {
                int recheck = ctl.get();
                if (! isRunning(recheck) && remove(command))
                    reject(command);
                else if (workerCountOf(recheck) == 0)
                    addWorker(null, false);
            }
            else if (!addWorker(command, false))
                reject(command);
        }

**分析可得：当新任务到来时，线程池处理优先级：corePollSize>任务队列>maximumPoolSize。**

###现成的Java线程池工厂
配置一个线程池是比较复杂的，尤其是对于线程池的原理不是很清楚的情况下，很有可能配置的线程池不是较优的，因此在**Executors**类（上图中，图片右下角这个类）里面提供了一些静态工厂，生成一些常用的线程池：  

- **newSingleThreadExecutor**：创建一个单线程的线程池。这个线程池只有一个线程在工作，也就是相当于单线程串行执行所有任务。如果这个唯一的线程因为异常结束，那么会有一个新的线程来替代它。此线程池保证所有任务的执行顺序按照任务的提交顺序执行。  
- **newFixedThreadPool**：创建固定大小的线程池。每次提交一个任务就创建一个线程，直到线程达到线程池的最大大小。线程池的大小一旦达到最大值就会保持不变，如果某个线程因为执行异常而结束，那么线程池会补充一个新线程。
- **newCachedThreadPool**：创建一个可缓存的线程池。如果线程池的大小超过了处理任务所需要的线程，那么就会回收部分空闲（60秒不执行任务）的线程，当任务数增加时，此线程池又可以智能的添加新线程来处理任务。此线程池不会对线程池大小做限制，线程池大小完全依赖于操作系统（或者说JVM）能够创建的最大线程大小。  
- **newScheduledThreadPool**：创建一个大小无限的线程池。此线程池支持定时以及周期性执行任务的需求。

因为这些方法在Executors类中都是静态方法，所以可以直接通过如下的方法获得到。

	ThreadPoolExecutor threadPool=Executors.newCachedThreadPool();

这差不多就是Java线程里面的全部内容了，大概有个全方位的了解，用于以后的精学，建议配合Jdk源码进行更深入的学习。