---
layout: post_layout
title: IntentService 源码详解
time: 2016年03月02日
location: 北京
pulished: true
excerpt_separator: "~~~"
---

本来想说一下AsyncTask和intenservice，HandlerThread的，不过真不敢说三者能说的面面俱到，所以还是先说一下Intenservice。看一下官方文档对IntentServie的解释：IntentService 是一个继承自Service，可以异步处理消息，用户通过startService(Intent) 来发送请求，服务会开启，并且其使用子线程去处理每一个Intent，并且执行完毕后会自动停止。换句话说，IntentService 提供了一种使用后台线程来处理操作的方式，它允许在不影响不阻塞UI线程的情况下去处理耗时操作。并且IntentService也不会受到大所属UI生命周期事件的影响，所以它适合在可能中断异步任务的情况中一直运行。
IntentService 存在如下限制：

·它不能跟UI直接交互。为了把执行结果通知给UI，你需要把activity实例传给它，或者通过LocalBroadcastManager来发送广播；

·任务串行执行。如果一个IntentService正在执行一个任务，而此时你有发送给它一个请求，那么第二个请求将会等到第一个请求执行完才会执行。

·运行在IntentSerive中的操作不能被人为打断。

~~~ 以上是关于IntentService的一些概念性问题，关于他的使用我相信大部分开发人员都使用过或是很轻松能够学会使用，那么我们就从源码的角度分析一下IntentService的实现原理。

我们定义一个继承自IntentService的类：

      public class RSSPullService extends IntentService {  
          public RSSPullService() {
              // Class name will be the thread name.
              super(RSSPullService.class.getName());

              // Intent should be redelivered if the process gets killed before
              // completing the job.
              setIntentRedelivery(true/false);
          }

        @Override 
        protected void onHandleIntent(Intent workIntent) {  
          // Gets data from the incoming Intent  
          String dataString = workIntent.getDataString();  
          ...    
          // Do work here, based on the contents of dataString    
          ...  
        }
      }

因为IntentServie继承自Service，所以别忘记在AndroidManifest.xml中生命哦，接着我们看一下使用，其实和正常service使用几无区别：

     mServiceIntent = new Intent(getActivity(), RSSPullService.class);  
     mServiceIntent.setData(Uri.parse(dataUrl));
     // Starts the IntentService 
     getActivity().startService(mServiceIntent);

重写 onHandleIntent 来处理我们的Intent，我想大家都这么使用，但是他有事怎么样来异步处理我们的任务的呢？
当我们启动一个service的使用，如果这个service还没有启动，它首先会由系统创建一个实例，所以会调用到它的构造函数，这个步骤相当繁琐，也不是我们必须要了解的，关于service的创建过程，可以学习一下老罗的文章：[Android系统在新进程中启动自定义服务过程（startService）的原理分析][1]。看一下IntentService 构造函数：

    /**
     * Creates an IntentService.  Invoked by your subclass's constructor.
     *
     * @param name Used to name the worker thread, important only for debugging.
     */
    public IntentService(String name) {
        super();
        mName = name;
    }

你会纳闷，这是个有参构造函数，我们不应该调用service的有参构造函数初始化一个实例啊。对，你说的没错，但是Intentservice是一个抽象类，我们需要继承它去自定义一个service，而在自定义的service中，我们可以声明一个无参构造函数，去调用父类的有参函数，正如RSSPullService的无参构造函数一样。而Intentservice的有参构造函数中只做了一件事，就是初始化mName，它使用来为以后的一个HandlerThread初始化用的。所以说我们最好在自定义的IntentService中调用super(name),否则mName为null；至于setIntentRedelivery(true/false)一会再说。

系统调用完无参构造函数后，会调用oncreate 进行一些初始化，这里的初始化和和无参数构造函数一样只调用一次，看一下这个函数：

    @Override
    public void onCreate() {
        // TODO: It would be nice to have an option to hold a partial wakelock
        // during processing, and to have a static startService(Context, Intent)
        // method that would launch the service & hand off a wakelock.

        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }

看到oncreate的代码，我们至少明白，它使用了Handler，看一下ServiceHandler源码：

     private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }
    
它使用了带有looper的构造函数进行初始化，说明我们需要为它传进一个looper，而handlerMessage中则直接调用了需要们重写的onHandlerIntent() 方法。那我们需要看到死怎么样获得的一个looper呢，回到oncreate，可以看到，它使用了HandlerThread，HandlerThread 继承自Thread，拥有一个含参构造函数，参数为string类型：

    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }

这个name就是对intentservice初始化时那么包装，mPriority是这个线程的优先级，其实我们要看的不是构造函数二是接下来的run函数：

    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }

我们之前讲过looper，执行Looper.prepare()后，它对当前实例进行了synchronized锁，即只有一个当前实例只能同时访问为一个带有该锁的代码段，代码段中获得mlooper，并notifyAll()，为什么要notifyAll()？回到oncreate函数，执行mServiceLooper = thread.getLooper()， thread.getLooper()代码如下：

    /**
     * This method returns the Looper associated with this thread. If this thread not been started
     * or for any reason is isAlive() returns false, this method will return null. If this thread 
     * has been started, this method will block until the looper has been initialized.  
     * @return The looper.
     */
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        
        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }

我们可以看到，getLooper方法中也持有了当前实例的锁，当执行run方法时，他已经执行到锁代码，并循环执行wait()，表明释放该对象锁，供run方法使用，所以当被通知notifyall时，它就又有机会执行while循环，一旦发现符合条件则跳出循环，返回looper，读到这我内心感叹一句--精妙精妙，太精妙！run方法和getLooper方法是典型并发同步的例子，可以学习一下。拿到looper后我们就可以在当前service中初始化handler了。那么handler是如何获取消息的呢？要获取消息那就要有地方发送消息，在哪？哈哈，在onstart 中，而onstart是被onstartCommond 在比较的新的api中替换掉的api，而在onstartCommond中也恰好调用到了onstart函数，看一下onstart，onstartCommond函数：

    @Override
    public void onStart(Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }

    /**
     * You should not override this method for your IntentService. Instead,
     * override {@link #onHandleIntent}, which the system calls when the IntentService
     * receives a start request.
     * @see android.app.Service#onStartCommand
     */
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }

每当我们试图去启动当前的intentservice的时候，都会执行onStartCommand，其中第三个参数startId表示当前是第几次启动该service。看onstart中使用mServiceHandler发送了消息，消息中对startID和intent进行了封装。如果你细心你还会发现在ServiceHandler的onHandleMessage该函数中还有一句stopSelf(msg.arg1)，这句话试图去关闭当前的service，为啥要关闭呢，因为intentservice的特点或者说优点就是执行完当前所有任务后即关闭service，省去人为关闭的麻烦，节省资源。onHandleIntent执行完之后就会调用stopSelf(msg.arg1)终止当前线程吗？也不一定，为什么这么说，因为stopSelf(msg.arg1)中参数ID是用来判断是否应该停止当前service，假如当前的ID和最后一次onStartCommond中id相同则会终止，否则的话说明仍然有消息没有处理完，如果使用stopSelf()，那不管有没有消息处理完，service都会被终止。那为什么说Intentservice是异步执行任务的呢？我们已经知道handler对消息的处理一步进行的，所以intentservice处理任务也是异步（messgequeue 排队机制），因为它使用了handler处理消息。

之前讲[如果还不清楚 Handler、Looper、Message 三者的关系，那 ^O^][2]我们提到过，在子线程中如果不再使用handler，我们应该人为的去调用Looper.quit()，而在intentservice中也恰好说明这一点：

    @Override
    public void onDestroy() {
        mServiceLooper.quit();
    }
还应该说一下setIntentRedelivery(true/false)，看一下他的实现：

    public void setIntentRedelivery(boolean enabled) {
        mRedelivery = enabled;
    }
    
很简单，为mRedelivery赋值，而mRedelivery是做什么的呢？我们都知道，service可以实现自启动的功能，即当我们的service被系统kill掉，它会根据onstartCommond中返回值来决定下一次service被系统kill掉时是否应该自启，以及自启的一些细节问题，以下是关于我们最常使用的三个返回值的详细介绍：
START_NOT_STICKY
 如果返回START_NOT_STICKY，表示当Service运行的进程被Android系统强制杀掉之后，不会重新创建该Service，当然如果在其被杀掉之后一段时间又调用了startService，那么该Service又将被实例化。那什么情境下返回该值比较恰当呢？如果我们某个Service执行的工作被中断几次无关紧要或者对Android内存紧张的情况下需要被杀掉且不会立即重新创建这种行为也可接受，那么我们便可将 onStartCommand的返回值设置为START_NOT_STICKY。举个例子，某个Service需要定时从服务器获取最新数据：通过一个定时器每隔指定的N分钟让定时器启动Service去获取服务端的最新数据。当执行到Service的onStartCommand时，在该方法内再规划一个N分钟后的定时器用于再次启动该Service并开辟一个新的线程去执行网络操作。假设Service在从服务器获取最新数据的过程中被Android系统强制杀掉，Service不会再重新创建，这也没关系，因为再过N分钟定时器就会再次启动该Service并重新获取数据。

START_STICKY
 如果返回START_STICKY，表示Service运行的进程被Android系统强制杀掉之后，Android系统会将该Service依然设置为started状态（即运行状态），但是不再保存onStartCommand方法传入的intent对象，然后Android系统会尝试再次重新创建该Service，并执行onStartCommand回调方法，但是onStartCommand回调方法的Intent参数为null，也就是onStartCommand方法虽然会执行但是获取不到intent信息。如果你的Service可以在任意时刻运行或结束都没什么问题，而且不需要intent信息，那么就可以在onStartCommand方法中返回START_STICKY，比如一个用来播放背景音乐功能的Service就适合返回该值。

START_REDELIVER_INTENT
 如果返回START_REDELIVER_INTENT，表示Service运行的进程被Android系统强制杀掉之后，与返回START_STICKY的情况类似，Android系统会将再次重新创建该Service，并执行onStartCommand回调方法，但是不同的是，Android系统会再次将Service在被杀掉之前最后一次传入onStartCommand方法中的Intent再次保留下来并再次传入到重新创建后的Service的onStartCommand方法中，这样我们就能读取到intent参数。只要返回START_REDELIVER_INTENT，那么onStartCommand重的intent一定不是null。如果我们的Service需要依赖具体的Intent才能运行（需要从Intent中读取相关数据信息等），并且在强制销毁后有必要重新创建运行，那么这样的Service就适合返回START_REDELIVER_INTENT。

在intetnservice中，setIntentRedelivery(true/false)需要跟情况而定，intentservice处理的intent之间有内在的联系，或者异步影响其他时间，那么就算setIntentRedelivery(true)，恐怕最后也不说我们想要的结果。如果系统杀掉了service，而我们更希望intentservice希望再一次执行所有的任务，那就没有必要在setIntentRedelivery(true)了。

好了，先总结到这儿，我们阅读Android的源码不仅学习实现原理，而且也应该学习它的代码规范，读了这么多Android系统的源码，我早已经被它的规范性折服，这也是我以后学了努力的方向，fighting！


  [1]: http://blog.csdn.net/luoshengyang/article/details/6677029
  [2]: https://liujianyue.github.io/2016/01/10/Handler-Looper-MessageQueue.html