---
layout: post_layout
title: JAVA 并发开发的一点总结(三)
time: 2016年06月12日
location: 北京
pulished: true
excerpt_separator: "~~~"
---
今天我们说说java中的信号量Semahpore。

## 信号量Semahpore

大学的时候我们应该都接触过信号量的有问题，比如消费者问题，哲学家就餐问题等。我在当时也是只跟着老师讲的东东，理解了一些皮毛，随着后来的工作，接触的代码以及自学的知识等，对他又有了重新及更加深刻的认识。不单单操作系统中使用了大量的信号量的东西，我们日常开发中也可借助它完成很优美的事件模型。

先看一下《Java Concurrency in Practice》一书中关于信号量的解释：技术信号量(Couting Semaphore) 用来同事控制同时访问某个特定资源的操作数量，或者同时执行某个操作的数量。技术信号量还可以用来实现某种资源池，或者对容器施加边界。而在java官方文档中是这样解释信号量的：一个计数信号量。从概念上讲，信号量维护了一个许可集。如有必要，在许可可用前会阻塞每一个 acquire()，然后再获取该许可。每个release() 添加一个许可，从而可能释放一个正在阻塞的获取者。但是，不使用实际的许可对象，Semaphore 只对可用许可的号码进行计数，并采取相应的行动。 

~~~ 获得一项前，每个线程必须从信号量获取许可，从而保证可以使用该项。该线程结束后，将项返回到池中并将许可返回到该信号量，从而允许其他线程获取该项。注意，调用 acquire() 时无法保持同步锁，因为这会阻止将项返回到池中。信号量封装所需的同步，以限制对池的访问，这同维持该池本身一致性所需的同步是分开的。 

将信号量初始化为 1，使得它在使用时最多只有一个可用的许可，从而可用作一个相互排斥的锁。这通常也称为二进制信号量，因为它只能有两种状态：一个可用的许可，或零个可用的许可。按此方式使用时，二进制信号量具有某种属性（与很多 Lock 实现不同），即可以由线程释放“锁”，而不是由所有者（因为信号量没有所有权的概念）。在某些专门的上下文（如死锁恢复）中这会很有用。 

此类的构造方法可选地接受一个公平 参数。当设置为 false 时，此类不对线程获取许可的顺序做任何保证。特别地，闯入 是允许的，也就是说可以在已经等待的线程前为调用 acquire() 的线程分配一个许可，从逻辑上说，就是新线程将自己置于等待线程队列的头部。当公平设置为 true 时，信号量保证对于任何调用获取方法的线程而言，都按照处理它们调用这些方法的顺序（即先进先出；FIFO）来选择线程、获得许可。注意，FIFO 排序必然应用到这些方法内的指定内部执行点。所以，可能某个线程先于另一个线程调用了 acquire，但是却在该线程之后到达排序点，并且从方法返回时也类似。还要注意，非同步的 tryAcquire 方法不使用公平设置，而是使用任意可用的许可。 

通常，应该将用于控制资源访问的信号量初始化为公平的，以确保所有线程都可访问资源。为其他的种类的同步控制使用信号量时，非公平排序的吞吐量优势通常要比公平考虑更为重要。 

此类还提供便捷的方法来同时 acquire 和释放多个许可。小心，在未将公平设置为 true 时使用这些方法会增加不确定延期的风险。 

内存一致性效果：线程中调用“释放”方法（比如 release()）之前的操作 happen-before 另一线程中紧跟在成功的“获取”方法（比如 acquire()）之后的操作。

我们看一下semaphore的api：

![api][1]


下面我先总结一下我所遇到两种的semaphore的使用情况：

### Semaphore与线程池的结合使用

前几天去XX驾校学习驾驶，哪的环境很好，在休息区有一个笼子，里面关了大约有十来只鹦鹉，而地下有大约五个小碗来供鸟儿们啄食，加入每一个完只能同时被一个鹦鹉使用，那么这恰好提供了一个信号量使用的场景，所以我们拿这个场景来举一个例子吧，看代码：

    public class SemephoreBird {
	private final static int BOWL_COUNT = 5;
	private final Semaphore sem = new Semaphore(BOWL_COUNT);
	private ExecutorService mExecutorService = Executors.newCachedThreadPool(); 
	private ArrayList<Bird> mList = new ArrayList<Bird>();
	class BirdEatingTask implements Runnable{
		Bird bird;
		public BirdEatingTask(Bird bird){
			this.bird = bird;
		}
		
		public void run() {
			try {
				sem.acquire();
				bird.Eat();
				sem.release();
			} catch (InterruptedException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}
			
		}
		
	}
	class Bird{
		public void Eat(){
			try {
			System.out.println("eating");
				Thread.sleep(5000);
			} catch (InterruptedException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}
		}
	}
	
	public void FeedBird(){
		for(int i = 0;i<10;i++){
			mList.add(new Bird());
		}
		for(Bird b : mList){
			mExecutorService.execute(new BirdEatingTask(b));
		}
	}
}

其中定义了一个Bird实体类，一个runnable的实现类BirdEatingTask，用来模仿鹦鹉啄食的场景，另外开辟了一个线程池，这个线程可以允许多个线程并发执行，但是每个线程并不一定能立即执行。应为他们受到了BOWL_COUNT的限制，然而最终鹦鹉们最终不管先后都会占到碗啄食，这是一个信号量和线程池的良好使用，这用情况网上提到的也很多，不再解释。

## 共享使用资源池

共享资源是一个信号量的经典使用场景，	例如数据库池。看一下例子：

        class Pool {
     private static final int MAX_AVAILABLE = 100;
     private final Semaphore available = new Semaphore(MAX_AVAILABLE, true);

     public Object getItem() throws InterruptedException {
       available.acquire();
       return getNextAvailableItem();
     }

     public void putItem(Object x) {
       if (markAsUnused(x))
         available.release();
     }

     // Not a particularly efficient data structure; just for demo

     protected Object[] items = ... whatever kinds of items being managed
     protected boolean[] used = new boolean[MAX_AVAILABLE];

     protected synchronized Object getNextAvailableItem() {
       for (int i = 0; i < MAX_AVAILABLE; ++i) {
         if (!used[i]) {
            used[i] = true;
            return items[i];
         }
       }
       return null; // not reached
     }

     protected synchronized boolean markAsUnused(Object item) {
       for (int i = 0; i < MAX_AVAILABLE; ++i) {
         if (item == items[i]) {
            if (used[i]) {
              used[i] = false;
              return true;
            } else
              return false;
         }
       }
       return false;
     }

   }

我们可以构造一个固定长度的资源池，当池为空时，请求资源将会失败，但你真的希望看到的行为是阻塞而不是失败，并且当池非空时接触阻塞。如若将Semaphore的计算值初始化为池的大小，并在从池中获取一个资源之前首先调用acquire方法获取一个 许可，在将资源返回给池之后调用release的释放许可，那么acquire将一直阻塞知道资源池不为空。

### 实现互斥体

计算信号量的一种简化形式是二值信号量，即初始值为1的Semaphore。二值信号量可以用作互斥体(mutex),b并具备不可重入的加锁语义：谁拥有这个唯一的许可，谁家拥有互斥锁。多数情况下用到这一点时一般跟初始化有关系。在我们初始化一些资源的时候, 尤其是在线程中，这些初始化的完成进度并不是我们随时都能确保完成的，所以我们需要知道初始化的结果，方便后续使用，或者释放资源。看下面的例子：

    private void init()
	{
		// loop thread
		initThread = new Thread()
		{
			@Override
			public void run()
			{
				try
				{
					// 请求一个信号量
					mSemaphore.acquire();
				} catch (InterruptedException e)
				{
				}
				Looper.prepare();

				mHander = new Handler()
				{
					@Override
					public void handleMessage(Message msg)
					{
						//do something
					}
				};
				// 释放一个信号量
				mSemaphore.release();
				Looper.loop();
			}
		};
		initThread.start();
	}

当我们试图去使用mHander时，并不能确定mHander已经初始化完成，所以可以这样做：

    try
		{
			// 请求信号量，防止mPoolThreadHander为null
			if (mHander == null)
				mSemaphore.acquire();
		} catch (InterruptedException e)
		{
		}
        mSemaphore.release();
		mHander.sendEmptyMessage(0x110);
  上边演示的历史很简单，不过现实中可能有更加复杂的使用情况，我在读Android源码的时候发现过这样一个情况：
  
    private final Semaphore mCameraOpenCloseLock = new Semaphore(1);
    
     /**
     * Open camera and start the preview.
     */
    private void openCameraAndStartPreview() {
        // Only enable HDR on the back camera
        boolean useHdr = mHdrEnabled && mCameraFacing == Facing.BACK;

        try {
            // TODO Given the current design, we cannot guarantee that one of
            // CaptureReadyCallback.onSetupFailed or onReadyForCapture will
            // be called (see below), so it's possible that
            // mCameraOpenCloseLock.release() is never called under extremely
            // rare cases.  If we leak the lock, this timeout ensures that we at
            // least crash so we don't deadlock the app.
            if (!mCameraOpenCloseLock.tryAcquire(CAMERA_OPEN_CLOSE_TIMEOUT_MILLIS, TimeUnit.MILLISECONDS)) {
                throw new RuntimeException("Time out waiting to acquire camera-open lock.");
            }
        } catch (InterruptedException e) {
            throw new RuntimeException("Interrupted while waiting to acquire camera-open lock.", e);
        }
        if (mCamera != null) {
            // If the camera is already open, do nothing.
            Log.d(TAG, "Camera already open, not re-opening.");
            mCameraOpenCloseLock.release();
            return;
        }
        mCameraManager.open(mCameraFacing, useHdr, getPictureSizeFromSettings(),
                new OpenCallback() {
                    @Override
                    public void onFailure() {
                        Log.e(TAG, "Could not open camera.");
                        mCamera = null;
                        mCameraOpenCloseLock.release();
                        mAppController.showErrorAndFinish(R.string.cannot_connect_camera);
                    }

                    @Override
                    public void onCameraClosed() {
                        mCamera = null;
                        mCameraOpenCloseLock.release();
                    }

                    @Override
                    public void onCameraOpened(final OneCamera camera) {
                        Log.d(TAG, "onCameraOpened: " + camera);
                        mCamera = camera;
                        updatePreviewBufferDimension();

                        // If the surface texture is not destroyed, it may have
                        // the last frame lingering. We need to hold off setting
                        // transform until preview is started.
                        resetDefaultBufferSize();
                        mState = ModuleState.WATCH_FOR_NEXT_FRAME_AFTER_PREVIEW_STARTED;
                        Log.d(TAG, "starting preview ...");

                        // TODO: Consider rolling these two calls into one.
                        camera.startPreview(new Surface(mPreviewTexture),
                                new CaptureReadyCallback() {
                                    @Override
                                    public void onSetupFailed() {
                                        // We must release this lock here, before posting
                                        // to the main handler since we may be blocked
                                        // in pause(), getting ready to close the camera.
                                        mCameraOpenCloseLock.release();
                                        Log.e(TAG, "Could not set up preview.");
                                        mMainHandler.post(new Runnable() {
                                           @Override
                                           public void run() {
                                               if (mCamera == null) {
                                                   Log.d(TAG, "Camera closed, aborting.");
                                                   return;
                                               }
                                               mCamera.close(null);
                                               mCamera = null;
                                               // TODO: Show an error message and exit.
                                           }
                                        });
                                    }

                                    @Override
                                    public void onReadyForCapture() {
                                        // We must release this lock here, before posting
                                        // to the main handler since we may be blocked
                                        // in pause(), getting ready to close the camera.
                                        mCameraOpenCloseLock.release();
                                        mMainHandler.post(new Runnable() {
                                           @Override
                                           public void run() {
                                               Log.d(TAG, "Ready for capture.");
                                               if (mCamera == null) {
                                                   Log.d(TAG, "Camera closed, aborting.");
                                                   return;
                                               }
                                               onPreviewStarted();
                                               // Enable zooming after preview has
                                               // started.
                                               mUI.initializeZoom(mCamera.getMaxZoom());
                                               mCamera.setFocusStateListener(CaptureModule.this);
                                               mCamera.setReadyStateChangedListener(CaptureModule.this);
                                           }
                                        });
                                    }
                                });
                    }
                }, mCameraHandler);
    }
    
上边代码试图打开camera并进行预览，但是在打开即相机初始化时将会进行复杂的工作，这是一个耗时的操作，但是假如在相机还没有初始化完毕，我们就试图去关闭相机呢，很可能会发生程序崩溃等严重错误，看一下相机关闭的代码：

     private void closeCamera() {
        try {
            mCameraOpenCloseLock.acquire();
        } catch(InterruptedException e) {
            throw new RuntimeException("Interrupted while waiting to acquire camera-open lock.", e);
        }
        try {
            if (mCamera != null) {
                mCamera.close(null);
                mCamera.setFocusStateListener(null);
                mCamera = null;
            }
        } finally {
            mCameraOpenCloseLock.release();
        }
    }
    
使用二值信号量将有效的避免程序初始化以及释放资源过程中的矛盾和错误，增强程序的健壮性。
其实在有些情况下二值信号量的的作用也可以用其他方法实现，比如说使用原子操作(Atomic)，以后会涉及到这里。
在之前的开发中我也几乎很少用到信号量的有关知识点，不过在阅读Android源码以及一些大神的代码的时候也会遇到使用情况，所谓技多不压身，学习学习也很有必要。
    
  [1]: /assets/img/easy-pure-blog.png