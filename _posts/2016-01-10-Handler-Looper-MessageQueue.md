---
layout: post_layout
title: 如果还不清楚 Handler、Looper、Message 三者的关系，那 ^O^
time: 2016年01月10日
location: 北京
pulished: true
excerpt_separator: "~~~~"
---
## 1.概述
为什么会突然捡起这老话题来总结呢，原因是最近在好多地方都看到了各个博主对Handler、Lopper、Message的总结，不写一篇都感觉不好意思呢，哈哈，其实主要是自己想熟悉并理清思路。Handler 作为Android 中重要的异步消息处理机制，它主要由Handler、Lopper、Message 构成，简述一下它们几个的概念：
**Message**
消息，其中包含了消息ID，消息处理对象以及处理的数据等，由MessageQueue统一列队，终由Handler处理。
**Handler**
处理者，负责Message的发送及处理。使用Handler时，需要实现handleMessage(Message msg)方法来对特定的Message进行处理，例如更新UI等。
**Looper**
消息队列，不断地从MessageQueue中抽取Message执行。因此，一个MessageQueue需要一个Looper。
**MessageQueue**
消息队列，用来存放Handler发送过来的消息，并按照FIFO规则执行。当然，存放Message并非实际意义的保存，而是将Message以链表的方式串联起来的，等待Looper的抽取。~~~~
## 2.讲清楚
Android系统的消息队列和消息循环都是针对具体线程的，一个线程可以存在（当然也可以不存在）一个消息队列和一个消 息循环（Looper），特定线程的消息只能分发给本线程，不能进行跨线程，跨进程通讯。想学习使用handler 和 messager 进行跨进程通信，可以参考[使用Messager 进行进程间通信][1]但是创建的工作线程默认是没有消息循环和消息队列的，如果想让该 线程具有消息队列和消息循环，需要在线程中首先调用Looper.prepare()来创建消息队列，然后调用Looper.loop()进入消息循环。 三步：

    class LooperThread extends Thread { 
      public Handler mHandler; 
   
      public void run() { 
          Looper.prepare(); //① 初始化一个Looper 实例，并放到 sThreadLocal中
   
          mHandler = new Handler() { // ② new 一个handler，此时将mHandler与与Looper、														//MessageQueue相关联
              public void handleMessage(Message msg) { 
                  // process incoming messages here 
              } 
          }; 
          Looper.loop(); //③ 开始让looper运转，也可以理解为MessageQueue开始无限循环
      } 
  }
  
  看一下 ① ② ③ 两个函数的源码：
**Looper.prepare() 源码**
    public static final void prepare() {
        if (sThreadLocal.get() != null) {//一个线程只允许调用一次prepare() 方法，否则抛出异常
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(true));
        }
**PS：sThreadLocal 是 一个ThreadLocal 变量，它可以根据不同的线程将当前数值保存在一个map中，这样做可以减小系统对于在不同线程中重复开辟一个相同变量付出的开销，关于ThreadLocal 网上有很多解释，大家可以参考，大家也可以参看一下ContentProvider 的源码，其中也用到了大量的ThreadLocal，后续我也回总结一篇关于ThreadLocal 的文章。**

**new Handler() 源码**
    public Handler() { 
    		this(null, false);  // 通常我们会调用无参构造函数，当时他还会调用有参构造函数
    }
    public Handler(Callback callback, boolean async) {  
        if (FIND_POTENTIAL_LEAKS) {  
            final Class<? extends Handler> klass = getClass();  
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&  
                    (klass.getModifiers() & Modifier.STATIC) == 0) {  
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +  
                    klass.getCanonicalName());  //注意内存泄漏哦
            }  
        }  
  
        mLooper = Looper.myLooper();  //其实在执行 sThreadLocal.get();
        if (mLooper == null) {  
            throw new RuntimeException(  
                "Can't create handler inside thread that has not called Looper.prepare()");  
        }  
        mQueue = mLooper.mQueue;  
        mCallback = callback;  
        mAsynchronous = async;  
    }  
**PS：正好 说一下callback，看一下下面的函数，msg消息执行可以分为以下几个步骤**

    /**
     * Handle system messages here.
     */
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
    
    
1.判断msg.callback 是否为空，这个callback 也就是我们平常再是方法 handler.post(Runnable r) 中的Runnable 对象 r，不为空这执行Runnable；
  2.msg.callback 为空，则判断mCallback是否为空，mCallback是Handler的一个内部接口，供Handler构造函数Handler(Callback callback) 使用，其中有一个和handler的相同的需要使用者实现的方法public boolean handleMessage(Message msg)，看一下源码你就知道怎么用了：
  
    /**
     * Callback interface you can use when instantiating a Handler to avoid
     * having to implement your own subclass of Handler.
     *
     * @param msg A {@link android.os.Message Message} object
     * @return True if no further handling is desired
     */
    public interface Callback {
        public boolean handleMessage(Message msg);
    }
  3.倘若mCallback也为空，那只好调用Handler自己的handleMessage(Message msg) 了。
   
   **Looper.loop()源码**
   
    public static void loop() {
        final Looper me = myLooper();//从该线程中取出对应的looper对象
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;//取消息队列对象...   

        //确保此线程是属于本地的，并且时刻跟踪该线程真实的标志位
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // 可能发生阻塞，取消息队列中的一个待处理消息..  
            if (msg == null) {
                // 并没有msg指示该looper应该停止，那继续循环
                return;
            }
            // This must be in a local variable, in case a UI event sets the logger
            //当前线程才必须是一个本地变量，以防止UI时间去设置日志器
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }
            msg.target.dispatchMessage(msg);

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }
            //确保在消息分发过程中，当前线程的标志位并没有被销毁
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }
            msg.recycle();
        }
}

通过代码，应该能够明白一下两点了：
a.Looper 保证了每个线程只有一个Looper 和一个MessageQueue；
b. 实际处理消息者 可以从代码： msg.target.dispatchMessage(msg) 看出，谁发送了消息，谁将最终处理消息；

好吧，上边太多，有些凌乱了吧，没事我们总结一下下：
1.首先Looper.prepare()在本线程中保存一个Looper实例，然后该实例中保存一个MessageQueue对象；因为Looper.prepare()在一个线程中只能调用一次，所以MessageQueue在一个线程中只会存在一个。
2.Looper.loop()会让当前线程进入一个无限循环，不断从MessageQueue的实例中读取消息，然后回调msg.target.dispatchMessage(msg)方法。
3、Handler的构造方法，会首先得到当前线程中保存的Looper实例，进而与Looper实例中的MessageQueuexiang相关联。
4.Handler的sendMessage方法，会给msg的target赋值为handler自身，然后加入MessageQueue中。
5.创建Handler实例时，会重写handleMessage方法，也就是msg.target.dispatchMessage(msg)最终调用的方法，当然这还有Callback供你选择。

**PS：1.你可能有一点会感到困惑，在Handler实际应用中，没有写过Looper.prepare()，Looper.loop(); 表急，Android 太贴心，已经在 Activity ActivityThread线程中对这些东东做了所有准备工作。
2.如果是我们手动在非主线程创建了Looper，那最好在我们不需要子线程handler再工作了手动去停止Looper，两种方法：quit(),QuitSafety(),两者的区别在于后者会等MssageQueue所有消息处理完在退出。**

## 3. 其他一些东东：关于非UI线程更新UI

**1.双Handler**
使用两层Handler，让子线程handler 发送消息给主线程Handler发送消息，来个例子吧：

    Handler mHandler = new Handler() {//主线程handler
		@Override
		public void handleMessage(Message msg) {
			super.handleMessage(msg);
			switch (msg.what) {
			case 0:
				//完成主界面更新,拿到数据
				String data = (String)msg.obj;
				
				updatedata();
				textView.setText(data);
				break;
			default:
				break;
			}
		}
	};
    
    private void updatedata() {
		new Thread(new Runnable(){

			@Override
			public void run() {
				//需要数据传递，用下面方法；
				Message msg =new Message();
				msg.obj = "数据";//可以是基本类型，可以是对象，可以是List、map等；
				mHandler.sendMessage(msg);
			}
			
		}).start();
	}
**PS:读者可以根据这个例子进行其他扩展，比如主线程和子线之间的交互以及循环更新某些UI 比如绘制canvas等**

**2.View.post(Runnable)/View.postDelayed(Runnable, long)**

**3.Activity.runOnUiThread(Runnable)**

以上两种你可能觉得方法独特，有种另辟蹊径的赶脚，不过你被骗了，其实这两个方法归根到底仍然是使用了Handler ，他们获得主线程的handler，并使用post(Runable).看一下源码你就知道了：


	View.java
    /*
     * <p>Causes the Runnable to be added to the message queue.
     * The runnable will be run on the user interface thread.</p>
     *
     * @param action The Runnable that will be executed.
     *
     * @return Returns true if the Runnable was successfully placed in to the
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     *
     * @see #postDelayed
     * @see #removeCallbacks
     */
    public boolean post(Runnable action) {
        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null) {
            return attachInfo.mHandler.post(action);
        }
        // Assume that post will succeed later
        ViewRootImpl.getRunQueue().post(action);
        return true;
    }
    Activity.java
    /*
     * Runs the specified action on the UI thread. If the current thread is the UI
     * thread, then the action is executed immediately. If the current thread is
     * not the UI thread, the action is posted to the event queue of the UI thread.
     *
     * @param action the action to run on the UI thread
     */
    public final void runOnUiThread(Runnable action) {
        if (Thread.currentThread() != mUiThread) {
            mHandler.post(action);
        } else {
            action.run();
        }
    }


至于Handler 以及looper、MessageQueue，现讲打这里，不过对于Handler的学习远不止于此，handler即可以帮你实现很多功能，正如Android源码自己使用那样，但也可能在使用过程中出现某些问题，比如内存泄漏等。






  [1]: http://my.oschina.net/u/262208/blog/378249