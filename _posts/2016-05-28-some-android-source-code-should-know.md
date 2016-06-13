---
layout: post_layout
title: 解析Android攻城狮几个必须要读的源码
time: 2016年05月28日
location: 北京
pulished: true
excerpt_separator: "Syntax"
---
本来还不想开这个专题，因为前边的java并发开发的的专题系列还没有总结完。不过后来又想，如果从多个角度贯穿java、 android 的知识线索，可能会在其中发现更多的交叉点，获得更多的启发，果不其然，这俩天重温了一下几个源码，感觉收获颇多（主要是读了忘，忘了还需要在读），接下来一阵子，我会好好的解读一下以下几个知识点及源码，当然其中有的相当有难度，还是要借助其他大神的文章来完成（Android 3.0以后版本）。

AsyncTask

LruCache

Handler

Binder

EventBus

#  一，AsyncTask源码解析

作为Android 最常使用的后台加载数据使用方法，Asynctask几乎是每个Android攻城狮都需要会使用，也应该明白原理的api，解析AsyncTask的源码的话题似乎先的很陈旧，因为网上关于这样的文章很多，不过个人认为大神解析时站在了比较高的高度上，而我想写是因为想从一个小白的角度来解析它，尽可能讲的全面而逻辑清晰。

先看一下官方文档给出的使用例子：

    private class DownloadFilesTask extends AsyncTask<URL, Integer, Long> {
      protected Long doInBackground(URL... urls) {
          int count = urls.length;
        long totalSize = 0;
         for (int i = 0; i < count; i++) {
              totalSize += Downloader.downloadFile(urls[i]);
              publishProgress((int) ((i / (float) count) * 100));
              // Escape early if cancel() is called
             if (isCancelled()) break;
          }
          return totalSize;
      }
 
      protected void onProgressUpdate(Integer... progress) {
          setProgressPercent(progress[0]);
      }
 
      protected void onPostExecute(Long result) {
          showDialog("Downloaded " + result + " bytes");
      }
 	 }
     //使用方法
	new DownloadFilesTask().execute(url1, url2, url3);

使用AsyncTask 都会一般使用execute() 来触发执行，那我们就从execute()来作为讲解的入口，

    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }

调用execute() 会调用executeOnExecutor(sDefaultExecutor, params)方法，好，看一下这个方法，

    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }

mStatus 作为被volatile修饰的枚举类型，初始值为Status.PENDING 意思为挂起，还没执行，进入函数executeOnExecutor	先判断mStatus的值，从代码可以看出当状态为RUNNING或FINISHED时，代码都会抛出IllegalStateException，虽然异常内容不同，当时可明确一点，AsyncTask只能被执行一次！假如当前状态还是PENDING 则会将状态置为RUNNING，接着执行我们的重在函数onPreExecute()，做一切执行前的准备工作。接着执行 mWorker.mParams = params， exec.execute(mFuture)，而 mWorker、exec、mFuture有时什么东东呢？哈哈，在么你继承AsyncTask时，有时候我们需要重新声明构造函数，那么我们必须在构造函数中指定执行super() 因为mWorker、mFuture 的初始化工作就是在积累的构造函数中完成的，我们看一下：

    public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                return postResult(doInBackground(mParams));
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occured while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }

mWorker 是一个WorkerRunnable 抽象类对象，而WorkerRunnable实现了Callable接口，Callable接口中有个类似于Runnable 中run() 方法的接口函数call() 接口函数，其中可以编写我们需要执行后台操作的代码。mTaskInvoked 为带有原子操作属性的boolean类型，这样可以再多个线程试图访问mTaskInvoked时，保持mTaskInvoked 值的唯一性，它的用法类似于关键字 volatile的使用。 当执行mTaskInvoked.set(true)时，其他任何地方试图更改或者读取mTaskInvoked值都会等待当前操作完成。Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND); 设置当前任务执行的优先级为后台，这样当多个线程并发后很多无关紧要的线程分配的CPU时间将会减少，有利于主线程的处理。接下来是极为关键的一句话return postResult(doInBackground(mParams)) 他不仅执行了我们重载的的方法doInBackground(mParams)，而且将doInBackground(mParams)返回节后作为参数传递给postResult，而参数为result，result为何类型？使我们自定义为AsyncTask<Params, Progress, Result>时，传入的类型。看一下postResult函数：

    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }

它将当前实例和result封装成为一个AsyncTaskResult，并把这个AsyncTaskResult作为message的obj，随message发送给了一个handler那么这个handler是怎么来的？看代码：

    private static Handler getHandler() {
        synchronized (AsyncTask.class) {
            if (sHandler == null) {
                sHandler = new InternalHandler();
            }
            return sHandler;
        }
    }

    private static class InternalHandler extends Handler {
        public InternalHandler() {
            super(Looper.getMainLooper());
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }

看到没，InternalHandle 通过Looper.getMainLooper()拿到当前主线程的looper来说初始化，那么你从中悟出了什么呢？哈哈，对的，既然我们需要使用主线程的looper那么我们当前的AsyncTask只能在主线程中初始化，还有一点愿意 InternalHandler 是一个静态内部类，它被所有的当前进程的AsyncTask公用，当AsyncTask类第一次被加载时，InternalHandler就会被初始化，所以你不能在子线程中加载AsyncTask类，否则其他任何地方定义的AsyncTask都将不能使用。
再看InternalHandler是如何处理message的呢？很明显根据message的种类来做了两种不同的事情，这两件事正是我们再熟悉不过的：第一个finish() 方法：

    private void finish(Result result) {
            if (isCancelled()) {
                onCancelled(result);
            } else {
                onPostExecute(result);
            }
            mStatus = Status.FINISHED;
        }
这个方法又会做一个判断，判断isCancelled()，为true则执行onCancelled(result)，否则继续执行onPostExecute(result)，这也恰恰说明了当我们试图去停止一个asynctask的时候，无法再它执行doInBackgroung()的时候打断它，而只能在做完doInBackgroung()后，而决定是否执行onPostExecute(result)。判断并执行完后在将mStatus 置为FINISHED。

第二个方法为onProgressUpdate(），这个正是我们试图去更新当前任务执行进度的时候多做的操作，那么这个message是从哪了发送过来的呢，这个正是我们在执行doInBackgroung()时，根据当前执行进度认为执行的代码publishProgress(Progress),

当mFuture 出事话的done函数中会再次试图去执行postResultIfNotInvoked(Result result)，不过这里他会判断mTaskInvoked.get() 的值，避免重复执行postResult(result)。

以上见了那么多，好像还没有讲到AsyncTask的精髓，马上就到了，executeOnExecutor() 函数我们接着分析
 exec.execute(mFuture)，exec是谁，exec是sDefaultExecutor，而sDefaultExecutor是谁呢，sDefaultExecutor 是SERIAL_EXECUTOR，什么？为什么要走这么绕，代码这么混乱？哈哈，不急，显然看上去代码有些冗余，但是你还没看到一个函数：

     /** @hide */
    public static void setDefaultExecutor(Executor exec) {
        sDefaultExecutor = exec;
    }

虽然这个方法是隐藏的，但是我们可以通过反射来重新设置我们自定义的Executor，我不能确定我们重新设计的Executor会不会比原生的好，但是系统还是为我们提供了定制的接口，可以自己尝试一下。
 记下来我们就要看非常重要的以个内部类了：SerialExecutor

     private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }

看到这个函数，我首先注意到了它是一个静态的内部类，它属于这个类以及这个类多以的实例，如果你在子类中不重写该类，那么它将为所有asynctask共用。我们分析一下：
首先定义了一个ArrayDeque 类对象，它的元素为Runnable，ArrayDeque类，你可以把它理解为一个链表，他能很好的执行元素的增删操作，不过一般它支持对链表的头以及尾部操作，使用它你可以很轻松的实现栈模式或是队列模式，而在这段代码中很，把ArrayDeque 作为一个队列使用， mTasks.offer()和mTasks.poll()正是作为这个“队列”的“插队”、“出队”的操作，那么我们在想想，队列的特点是什么，有序性，即FIFO.根据这一点，也恰巧说明了AsyncTask通过execute执行时，会是任务串行执行（一个asynctask执行完后另一个才执行）。而我们在看 execute()、scheduleNext()函数，他们都由关键字synchronized修饰，而在一个静态内部类当中，这样的方法在同一个时刻只能被一个当前类实例执行，换句话说他保证了mTasks操作的并发性，使多个asynctask可以申请操作mTasks而不会出错。读到这是不是已经被android精湛的设计深深吸引和折服。而在scheduleNext()函数中通过 THREAD_POOL_EXECUTOR.execute(mActive)，来达到串行执行任务的目的，mActive其实就是一FutureTask<Result>，FutureTask实现了Runnable接口，而它的run方法调用了mworker的call()方法，所以在线程池中执行的任务时call()函数中的代码。THREAD_POOL_EXECUTOR使用了自己定制的线程池，这里不赘述，可以参考我之前的文章：[ThreadPoolExecutor 线程池及其扩展][1]

解读完源码，我觉得还有几点需要加以注意和强调：

a. 源码中一共有两个线程 SERIAL_EXECUTOR、THREAD_POOL_EXECUTOR，但是他们完美地执行着自己的工作：SERIAL_EXECUTOR，负责将不断加入到线程池的队列排队，使他们串行执行，THREAD_POOL_EXECUTOR 则由SERIAL_EXECUTOR调用 进行真正的任务执行;

b. AsyncTask 必须在主线程中初始化并加载，负责多有的AsyncTask将不能使用；

c.AsyncTask 非要串行执行吗，错！如果你读过AsyncTask 源码中的注释你会发现下面的话：

        When first introduced, AsyncTasks were executed serially on a single background
     * thread. Starting with {@link android.os.Build.VERSION_CODES#DONUT}, this was changed
     * to a pool of threads allowing multiple tasks to operate in parallel. Starting with
     * {@link android.os.Build.VERSION_CODES#HONEYCOMB}, tasks are executed on a single
     * thread to avoid common application errors caused by parallel execution.</p>
     * <p>If you truly want parallel execution, you can invoke
     * {@link #executeOnExecutor(java.util.concurrent.Executor, Object[])} with
     * {@link #THREAD_POOL_EXECUTOR}.

大致意思是AsyncTask 的发展经历三个阶段：串行->并行->串行。并行时可能会发生意想不到的并发错误。而现阶段 我们不仅可以串行执行任务还可以并行，那就是调用 executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR,params);

c.Executor 、FutureTask 、Callable  三者组合是一个很好的并发开发的模式，他们既能执行任务又能获取形成执行后的结果，我们完全可以再自己的代码中试着去使用他们，

d.开发我们中我们可以学习static 加 synchronized 关键字来开发某些具有专门用途的的工具，在AsyncTask中它借助这种模式实现了将多个线程排队串行执行的任务，那么我们可以开发别的工具类，比如以下代码：

e.并发开发在一些相对复杂的程序中往往会使用，而关于并发开的方法也有很多种，在AsyncTask中 包括线程池，原子操作，volatile 都是提供了很好的并发性，这一点也是值得我们日常开发中去使用的，关于这一点我也正在歇一歇系列博客：[JAVA 并发开发的一点总结][2]

f.在《Effective JAVA》中曾建议尽量考虑使用枚举类型代替“int 枚举模式”，举个例子：
	
    public static final int STATUS_RUNNING = 0;
	public static final int STATUS_PENDING = 1;
	public static final int STATUS_FINISH = 2;
	
代码可以优化为：

        public enum Status {
        
            PENDING,

            RUNNING,

            FINISHED,
        }
   具体关于把这部分不再本博客的讲解之内，我只是提醒一下  ( T T) .    
        
  [1]: https://liujianyue.github.io/2016/05/06/ThreadPoolExecutor.html
  [2]: https://liujianyue.github.io/2016/05/01/java-concurrent-programming.html