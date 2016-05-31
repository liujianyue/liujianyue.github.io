---
layout: post_layout
title: ThreadPoolExecutor 线程池及其扩展
time: 2016年05月06日
location: 北京
pulished: true
excerpt_separator: "线程池主要由"
---
## 二， 线程池
关于线程池，我之前在网上看了很多资料，感觉将的浅显易懂还是《Java Concurrency in Practice》一书，下面对书中的一些内容进行摘录和自我理解。从字面含义来看，是指管理一组同构工作的线程的资源池。线程池是与工作队列（Work Queue）密切相关，其中在工作队列中保存了所有等待执行的任务。工作者线程（WorkerThread）的任务很简单；从工作队列中获取一个任务，执行任务，然后返回线程池并等待先一个任务。

之所以学习线程池，是因为它比“为每一个任务分配一个线程具有巨大优势”。通过重用现有的线程而不是创建新线程，可以再处理多个请求是分摊在线程创建和销毁的过程中产生的巨大开销。另外，当请求到达时，工作线程通常已经存在，因此不会犹豫等待创建线程而延迟任务的执行，从而提高响应性，同事还可以防止过多线程相互竞争资源而是应用程序耗尽内存或失败。
线程池主要由java通过ThreadPoolExecutor已经扩展好的四类方式，所以我们首先来学习ThreadPoolExecutor

### 1.ThreadPoolExecutor
线程池类为 java.util.concurrent.ThreadPoolExecutor，常用构造方法为： 

    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler)
                              
                              
#### corePoolSize： 线程池维护线程的最少数量 ,也可称为核心线程数
这些线程可能一直存活，因为这样可以减少系统创建线程的开销。

#### maximumPoolSize：线程池维护线程的最大数量 
当活动线程数量达到这个数值后，后续的新任务将会被阻塞。

#### keepAliveTime： 线程池维护线程所允许的空闲时间 
1.默认情况下，核心线程将会在线程池中一直存活，即使他们处于闲置状态。若将ThreadPoolExecutor 的allowCoreThreadTimeOut (在JDK1.6之后)属性设置为ture，那么心智的核心线程在等待新任务到来时会有超市策略，这个时间间隔就有keepAliveTime决定，当等待时间超过此值时，核心线程就会终止。

2.不管的allowCoreThreadTimeOut(在JDK1.6之后)的属性为何，非核心线程的闲置的时间超过这个值都会被收回。

#### unit： 线程池维护线程所允许的空闲时间的单位 
unit可选的参数为java.util.concurrent.TimeUnit中的几个静态属性： 
NANOSECONDS、MICROSECONDS、MILLISECONDS、SECONDS。

#### workQueue： 线程池所使用的缓冲队列 
a.一个任务通过 execute(Runnable)方法被添加到线程池，任务就是一个 Runnable类型的对象，任务的执行方法就是 Runnable类型对象的run()方法。 

b.workQueue是BlockingQueue类型，而BlockingQueue只是一个接口，它所表达的是当队列为空或者已满的时候，需要阻塞以等待生产者/消费者协同操作并唤醒线程。其有很多不同的具体实现类，各有特点。有的可以规定队列的长度，也有一些则是无界的。

c.按照Executors类中的几个工厂方法，分别使用的是：
LinkedBlockingQueue:FixedThreadPool和SingleThreadExecutor使用的是这个BlockingQueue，队列长度是无界的，适合用于提交任务相互独立无依赖的场景。
SynchronousQueue:CachedThreadPool使用的是这个BlockingQueue，通常要求线程池不设定最大的线程数，以保证提交的任务有机会执行而不被丢掉。通常这个适合任务间有依赖的场景。
当然，开发者也可以定制ThreadPoolExecutor时使用ArrayBlockingQueue有界队列。

#### threadFactory 创建新线程的线程构造器
它是一个接口类型，只有一个方法 Thread newThread(Runnable r).
#### handler： 线程池对拒绝任务的处理策略 (拒绝执行处理器)

a.这个参数不常用，当线程池由于队列已满或无法成功执行任务而不能执行新任务时，若将ThreadPoolExecutor 会调用handler 的 rejectedExecution 方法。

b.对于任务丢弃，ThreadPoolExecutor以内部类的形式实现了4个策略。分别是：
CallerRunsPolicy。提交任务的线程自己负责执行这个任务。

AbortPolicy。使Executor抛出异常，通过异常做处理。

DiscardPolicy。丢弃提交的任务。

DiscardOldestPolicy。丢弃掉队列中最早加入的任务。

在调用构造方法时，参数中未指定RejectedExecutionHandler情况下，默认采用AbortPolicy。

### 2.ThreadPoolExecutor 执行规则
1.如果此时线程池中的数量小于corePoolSize，分为两种情况：

a.如果通过execute(Runnable) 方法来添加线程，即使线程池中的线程都处于空闲状态，也要创建新的线程来处理被添加的任务；
b.如果是 threadFactory.newThread(Runnable) 来添加线程那么会直接启动一个核心线程来执行任务。

2.如果此时线程池中的数量等于 corePoolSize，但是缓冲队列 workQueue未满，那么任务被放入缓冲队列。 

3.如果此时线程池中的数量大于corePoolSize，缓冲队列workQueue满，并且线程池中的数量小于maximumPoolSize，建新的线程来处理被添加的任务。 

4.如果此时线程池中的数量大于corePoolSize，缓冲队列workQueue满，并且线程池中的数量等于maximumPoolSize，那么通过 handler所指定的策略来处理此任务。

### 3.ThreadPoolExecutor的线程池大小的确定
线程池大小的确定是一件相当繁琐的事情，可以参考:[如何合理地估算线程池大小？][1]
根据官方文档以及其他的专业书籍的建议，核心线程数以及线程池的大小在如下配置时可以实现资源最优利用率：
  
    CPU_COUNT = Runtime.getRuntime().availableProcessors();
   	CORE_POOL_SIZE = CPU_COUNT + 1;
    MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
    
    
### 4.ThreadPoolExecutor 的预有配置
java之所以人性化，是因为他已经考虑到了使用者在使用ThreadPoolExecutor 过程中对于根据实际场景而配置ThreadPoolExecutor的头疼问题，所以java为我们已经配置好了四种在实际应用场景中最常使用的配置,福利啊：

### newFixedThreadPool
newFixedThreadPool将创建一个固定长度的线程池，每当提交一个任务就创建一个线程，知道达到线程的最大数量，这个线程池的规模将不再变化（如果某个线程由于发生了未预料的Exception而结束，那么线程池会补充一个新的线程）。

### newCachedThreadPool
newCachedThreadPool 将创建一个可缓存的线程池，如果线程池的当前规模超过了需要时，那么将回收闲置的线程，而当需求增加时，则可以添加新的线程，线程池的规模不存在任何闲置。

### newSingleThreadExecutor 
newSingleThreadExecutor 是一个单线程的Executor，他创建单个工作者线程来执行任务，如果这个线程异常结束，将会创建另一个线程来替代。newSingleThreadExecutor 能确保依照任务在队列的顺序来串行执行（例如FIFO、LIFO、优先级）。

### newScheduledThreadPool 
newScheduledThreadPoo 创建了一个固定长度的线程池，而且以延迟或定时的方式来执行任务，类似于Timer。

P98 《Java Concurrency in Practice》

### 5.ThreadPoolExecutor 的扩展
我们知道ThreadPoolExecutor是ExecutorService的一个实现类，而在ExecutorService 是继承Executor而来，ExecutorService 在Executor的基础上既增加了submmit系列接口也增加了对生命周期的控制。

### submit系列接口

![submit系列接口](/assets/img/ThreadPoolExecutor-submmit.png)

### ExecutorService中，和生命周期相关的，声明了5个方法：

awaitTermination() 阻塞等待shutdown请求后所有线程终止，会有时间参数，超时和中断也会令方法调用结束

isShutdown()  通过ctl属性判断当前的状态是否不是RUNNING状态

isTerminated()  通过ctl属性判断当前的状态是否为TERMINATED状态

shutdown() 关闭Executor，不再接受提交任务

shutdownNow() 关闭Executor，不再接受提交任务，并且不再执行入队列中的任务

在ThreadPoolExecutor中他们如下定义：

    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(SHUTDOWN);
            interruptIdleWorkers();
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }

    public List shutdownNow() {
        List tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP);
            interruptWorkers();
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }
    
    public boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (;;) {
                if (runStateAtLeast(ctl.get(), TERMINATED))
                    return true;
                if (nanos <= 0)
                    return false;
                nanos = termination.awaitNanos(nanos);
            }
        } finally {
            mainLock.unlock();
        }
    }
    
    final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            if (workerCountOf(c) != 0) { // Eligible to terminate
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        terminated();
                    } finally {
                        ctl.set(ctlOf(TERMINATED, 0));
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }

### 除此之外，ThreadPoolExecutor还提供了其他扩展点供使用者扩展：
1.beforeExecute() 在每个任务执行前做的处理

2.afterExecute() 在每个任务执行后做的处理

3.terminated() 在ThreadPoolExecutor到达TERMINATED状态前所做的处理

4.finalize() 有默认实现，直接调用shutdown()，以保证线程池对象回收

5.onShutdown() 在shutdown()方法执行到最后时调用，
6.在java.util.concurrent.ScheduledThreadPoolExecutor类实现中用到了这个扩展点，做一些任务队列的清理操作

    class MyThreadPoolExecutor extends ThreadPoolExecutor{
		public MyThreadPoolExecutor(...) {
			super(...);
		}
		
		@Override
		protected void beforeExecute(Thread t, Runnable r) {
			// 在每个任务执行前做的处理
			super.beforeExecute(t, r);
		}

		@Override
		protected void afterExecute(Runnable r, Throwable t) {
			// 在每个任务执行后做的处理
			super.afterExecute(r, t);
		}

		@Override
		protected void terminated() {
			// 在ThreadPoolExecutor到达TERMINATED状态前所做的处理
			super.terminated();
		}

		@Override
		protected void finalize() {
			// 有默认实现，直接调用shutdown()，以保证线程池对象回收
			super.finalize();
		}
	}

关于JAVA线程池，需要学习的东西还有很多，笔者主要参考了《Java Concurrency in Practice》书中关于这部分的解读，感兴趣的可以看一下
<a href="/assets/files/Java-Concurrency-in-Practice-chapter-8.pdf">《Java Concurrency in Practice：chapter 8》</a>

 参考：
[线程池ThreadPoolExecutor使用简介][2]

[ThreadPoolExecutor的应用和实现分析][3]

[java中Executor、ExecutorService、ThreadPoolExecutor介绍][4]

感兴趣的同学也可以参考任玉刚《Android 开发艺术探索》关于这部分的讲解。后续可能还会接着写，完善这一部分。

[下一篇：信号量的使用（Semaphore）](https://liujianyue.github.io/2016/05/06/ThreadPoolExecutor.html)

  [1]: http://ifeve.com/how-to-calculate-threadpool-size/
  [2]: http://coach.iteye.com/blog/855850
  [3]: http://www.molotang.com/articles/514.html
  [4]: http://blog.csdn.net/linghu_java/article/details/17123057
