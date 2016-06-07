---
layout: post_layout
title: Android 内存溢出和内存泄漏的总结
time: 2016年03月28日
location: 北京
pulished: true
excerpt_separator: "~~~"
---

对于这两点，几乎在阅读很多周刊啊，个人博客啊等，有事没事都会碰都一些文章，内存溢出 在各个文章中的讲解都差不多，不过关于内存泄漏，我觉得有必要把我遇到过的各种情况分类讲解一下，俗话说好记性不如烂笔头，记下来，以后看。

关于内存溢出，朋友们可以参考如下两篇文章，将的很详细：

[Java 内存溢出（java.lang.OutOfMemoryError）的常见情况和处理方式总结][1]

[Java中OutOfMemoryError(内存溢出)的三种情况及解决办法 ][2]

本来还要打算总结一下JVM内存管理以及垃圾回收机制，不过网上有很将的很好的，我就不照葫芦画瓢了，我也是通过他们的博客进行认知和学习的，推荐几篇：

[Java之美[从菜鸟到高手演变]之JVM内存管理及垃圾回收][3]

[java虚拟机内存管理机制 系列文章][4]

那么接下来我想对java 以及Android 常见的几种内存泄漏做一个总结。~~~

## JAVA中自由的内存泄露的情况

Android 使用java 进行开发，所以我们首先不得不重视一下java中自有的几种会出现内存泄露的情况：
 **java 集合类导致的内存泄漏**

java 集合类导致的内存泄漏通常有两种形式：
1.数组中无用部分得不到及时销毁

2.静态集合类引起内存泄露

摘取一个经典的例子：

    Static Vector v = new Vector(10); 
    for (int i = 1; i<100; i++) 
    { 
    Object o = new Object(); 
    v.add(o); 
    o = null; 
    }// 

该例中，循环申请Object 对象，并将所申请的对象放入一个Vector 中，如果仅仅释放引用本身（o=null），那么Vector 仍然引用该对象，所以这个对象对GC 来说是不可回收的。因此，如果对象加入到Vector 后，还必须从Vector 中删除，最简单的方法就是将Vector对象设置为null。

**缓存是java内存泄漏的另一个来源**

缓存一种用来快速查找已经执行过的操作结果的数据结构。因此，如果一个操作执行需要比较多的资源并会多次被使用，通常做法是把常用的输入数据的操作结果进行缓存，以便在下次调用该操作时使用缓存的数据。缓存通常都是以动态方式实现的，如果缓存设置不正确而大量使用缓存的话则会出现内存溢出的后果，因此需要将所使用的内存容量与检索数据的速度加以平衡。

**使用监听器和其他回调造成的内存泄漏**

在java 编程中，我们都需要和监听器打交道，通常一个应用当中会用到很多监听器，我们会调用一个控件的诸如addXXXListener()等方法来增加监听器，但往往在释放对象的时候却没有记住去删除这些监听器，从而增加了内存泄漏的机会。回调的原理类似于监听器，造成内存泄漏的原因在于我们销毁持有回调的对象是，没有销毁回调对象。

**PS**以上三种情况之所讲解简介，是因为我觉得读者有必有阅读一些《Effective java》中的第二章 第6,7条关于内存泄漏以及终结方法的使用可以再这里下载： <a href="/assets/files/Effective-Java-chapter-2-tips-6-and-7.pdf">《Effective java：chapter 2,tips 6,7》</a>  。以及阅读《Thinking in JAVA》中第十一章，第10节的关于WeakHashMap的讲解。

**耗时的连接**

在没有接触Android之前，就已经和内训泄漏打了个招呼，比如最常用的数据库连接（dataSourse.getConnection()），网络连接(socket)和io连接，除非其显式的调用了其close（）方法将其连接关闭，否则是不会自动被GC 回收的。对于Resultset 和Statement 对象可以不进行显式回收，但Connection 一定要显式回收，因为Connection 在任何时候都无法自动回收，而Connection一旦回收，Resultset 和Statement 对象就会立即为NULL。但是如果使用连接池，情况就不一样了，除了要显式地关闭连接，还必须显式地关闭Resultset Statement 对象（关闭其中一个，另外一个也会关闭），否则就会造成大量的Statement 对象无法释放，从而引起内存泄漏。这种情况下一般都会在try里面去的连接，在finally里面释放连接。

**不管不顾的非静态内部类/匿名内部类**

我们知道，在类内部创建一个非静态内部类，并创建了非静态的对象时，这个对象默认是持有外部类对象的引用的，如果这个内部类对象执行大量的异步操作，比如其中包含一个工作的线程，那么及时当我们外部类对象试图销毁时，由于我们的内部类对象的工作一直在继续没有停止，他会一直持有外部类引用导致外部类对象也无法及时销毁。
另一种情况创建了一个该类的静态对象，我们知道静态变量的生命周期和我们整个进程的生命周期一样长，耐活，所以还是会发生上述情况即及时外部对象试图去销毁，由于内部对象生命周期长并不打算销毁，导致外部对象也无法销毁。

上述情况我们可以将内部类抽离出去形成单独一个类，并让其持有此前外部对象的的弱引用，这一点后续仍然会讲到。

**单例模式造成的内存泄漏**

所谓单例模式造成内存泄漏，首先明白单例模式所创建的对象在整个进程中只保存一个对象，这很正常，因为我们已经不止一次使用了淡了模式，不正常的是，当我们使用单例模式时游在该单例中持有了其他非静态对象的引用，使得这些对象无法被销毁。
解决办法是，对这些被引用的对象加以关注，不再使用时应认为的将其销毁，当然这里的销毁可能有很多情况，归根到底是把这些对象与单例模式的对象的关系解绑。

## Android 中常见的内存泄漏场景

Android 常见的内存泄漏的场景基本也囊括了java中常见的内存泄漏的场景，而他特有的其他场景大多与context有关，我主要说一下以下几中情况。

**1.错误的单例模式**

看以下例子：

      public class Singleton {
      private static Singleton instance;
      private Context mContext;

      private Singleton(Context context) {
          this.mContext = context;
      }

      public static Singleton getInstance(Context context) {
          if (instance == null) {
              instance = new Singleton(context);
          }
          return instance;
      }
  }

instance作为静态对象，其生命周期要长于普通的对象（参数 context），和这个应用进程相同，当某个Activity 试图去getInstance（this）时，instance 以后会一直保有该activity的引用，即使该activity 有朝一日试图去销毁，那也不行，因为还有对象在引用它，这导致了内存泄漏。

**2.错误的静态变量**

在Activity中极为不建议使用声明带有静态性质的view，比如以下代码：

    public class MainActivity extends Activity {
    private static ImageView iv;

    @Override
    protected void onCreate(Bundle saveInstanceState) {
        super.onCreate(saveInstanceState);
        setContentView(R.layout.activity_main);
        iv = new ImageView(this);
        iv.setImageDrawable(getResources().getDrawable(R.drawable.ic_launcher););
    }
}

在这段代码中iv多次引用了this，而自己又是一个静态的view支持他将一直保有当前activity的引用直到进程结束，造成内存泄漏。

**3.非静态handler匿名内部类造成的内存泄漏**

handler 能够潜在的造成内存泄漏的情况，人尽皆知，概括一下就是，handler并不能保证messagequeue中的消息能够及时的的被处理，尤其是在当前activity finish() 掉之前，一旦handler被阻塞，而当前activity试图去销毁时，由于handler持有当前activity的引用，所以造成了activity的内存泄漏，区别于 java中非静态内部类，及时此时handler 变量即使非静态的，仍然可能造成内存泄漏。
handler 被阻塞说的有点不正确，确切的说应该是handler中待处理的runnable 对象，或是handler 执行postDelayed() 而无法及时执行任务时，导致messageQueue 阻塞，而导致messageQueue 持有message的引用，message持有handler的应用，handler 持有activity的引用。

解决办法：
1.如果是handler正在执行runnable对象，而runnable对象自己由于某些耗时操作比如读取disk，或作一些复杂的计算，而阻塞，那么应该用正确的方法来终止这个线程。

2.如果是执行postDelayed() 而阻塞，可以通过随时对handler可能保有的message的观察，在必要时remocallbacks() 来停止handler的后续工作。
 
3.以上两个方法都是观察handler的的任务执行的情况，在恰当的时机切断其与activity的关联，而另一种方法则是改变handler的生命周期，即将要使用的handler定义为静态的类，并让其持有外部avtivity的弱引用，如不持有此引用，handler无法访问activity中的成员，因为静态内部类不可直接访问外部类的非静态变量，方式如下：

    static class MyHandler extends Handler {
    WeakReference<Activity > mActivityReference;

    MyHandler(Activity activity) {
        mActivityReference= new WeakReference<Activity>(activity);
    }

    @Override
    public void handleMessage(Message msg) {
        final Activity activity = mActivityReference.get();
        if (activity != null) {
           activity. mImageView.setImageBitmap(mBitmap);
        }
    }
}
其实，造成内存

**非静态AsyncTask匿名内部类造成的内存泄漏**

至于非静态AsyncTask匿名内部类造成的内存泄漏我遇到的有两种情况：

1.AsyncTask持有外部activity成员变量的引用，一旦AsyncTask的执行收到阻塞，那么被引用的成员变也无法被回收，而成员变量一直引用当前activity的引用，所以导致了内存泄漏。
比如下列代码：

   private TextView mTextview;

    new AsyncTask<...> {

    @Override

    protected void onPostExecute(Objecto) {

    mTextview.setText("text");

    }

    }.execute();

2.如果AsyncTask被声明为Activity的非静态的内部类，那么AsyncTask会保留一个对创建了AsyncTask的Activity的引用。如果Activity已经被销毁，AsyncTask的后台线程还在执行，它将继续在内存里保留这个引用，导致Activity无法被回收，引起内存泄露，这种情况类似于上边所说的handler，归根到底是因为AsyncTask个activity的生命周期不同。

对于上述两种情况 ：第一种情况在官方文档中正经给出过这样一个例子，其中讲到了AsyncTask 使用ImageView的方法：

    class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {     
      private final WeakReference<ImageView> imageViewReference;  
      private int data = 0; 
      public BitmapWorkerTask(ImageView imageView) {   
      		// Use a WeakReference to ensure the ImageView can be garbage collected
            imageViewReference = new WeakReference<ImageView>(imageView);    
      }     
      // Decode image in background.    
      @Override 
      protected Bitmap doInBackground(Integer... params) {   
        data = params[0];  
        return decodeSampledBitmapFromResource(getResources(), data, 100, 100));  
      }     
      // Once complete, see if ImageView is still around and set bitmap. 
      @Override   
      protected void onPostExecute(Bitmap bitmap) {     
        if (imageViewReference != null && bitmap != null) {     
            final ImageView imageView = imageViewReference.get();  
                if (imageView != null) {       
                imageView.setImageBitmap(bitmap);        
                }      
            }   
        } 
    }
看一看，是不是可以通过这种方法来解决情况一中所说的问题。

对于情况二，我们有两种方法可以解决该问题：

方法一，导致activity不能正常销毁是因为AsyncTask被阻塞，并持有activity的引用，那么我们可以在activity onDistory()中使用AsyncTask.cancel(true) 来终止AsyncTask，并且在doInBackground函数中时刻检测 isCancelled()，若为ture 则停止正在进行的工作（大多数情况下是一个循环）。

    mAsyncTask = new AsyncTask<Object, Void, Boolean>() {
            @Override
            protected void onPreExecute() {
                super.onPreExecute();
            }

            @Override
            protected Boolean doInBackground(Object... params) {
                // do something in backfround
                // 长时间的耗时
                while (true) {
                    if (isCancel())
                        break;
                    today++;
                    if (today > 100000)
                        break;
                }
                return true;
            }

            @Override
            protected void onPostExecute(Boolean result) {
                super.onPostExecute(result);
                if (result) {
                    // success do something
                } else {
                    // error
                }
            }

            @Override
            /*
            *重写该方法，进行一些善后处理
            */
            protected void onCancelled() {
                super.onCancelled();
            }
        };

方法二，可以类比handler的第三种解决方法，

    class mAsyncTask extends AsyncTask<Object, Void, Boolean>() {
    WeakReference<Activity > mActivityReference;
   		public mAsyncTask（Activity activity）{
        mActivityReference= new WeakReference<Activity>(activity);
        }
            @Override
            protected void onPreExecute() {
                super.onPreExecute();
            }

            @Override
            protected Boolean doInBackground(Object... params) {
                // do something in backfround
                // 长时间的耗时
                while (true) {
                    if (mActivityReference.get()==null)
                        break;
                    today++;
                    if (today > 100000)
                        break;
                }
                return true;
            }

            @Override
            protected void onPostExecute(Boolean result) {
                super.onPostExecute(result);
                if (result) {
                    // success do something
                } else {
                    // error
                }
            }
        }

**绑定而无解绑**

这种情况一般指的是broadcastReceiver和contentObserver，注册后没有解注册，我相信刚学习android的同学都遇到过系统提示的关于解绑的错误，这个注意点就好，不多说。

**忘记了回收与重复利用**

回想一下TypedArray、Bitmap、Parcel是不是都存在用完以后需要回收的情况，对于这一点，开发工具有时会提示我们需要recycle，这三者都占用了较大的内存，不及时会后导致内存泄漏。

另外想想哪里需要重复利用呢，其实这种场景很多，最经典肯定要属 Adapter 中重复利用viewHolder，不过时下RecycleView ‘大行其道’，已经为我们包装的很好，不需要人为的考虑这一点，使我们把重心放到效果上，很贴心。


关于Android中的内存泄漏的情况，我就先总结这么多，写完了其实也发现解决handler 和 Asyntask的内存泄漏讲的并不是很清楚，其实解决这两种情况的内存泄漏，我个人认为最重要的就是正确的停止handler或是asynctask任务，防止由于两者的阻塞使得activity不能正常销毁，这个我会以后总结一下的。

  [1]: http://outofmemory.cn/c/java-outOfMemoryError
  [2]: http://blog.sina.com.cn/s/blog_701c951f0100n1sp.html
  [3]: http://blog.csdn.net/zhangerqing/article/details/8214365
  [4]: http://blog.csdn.net/lengyuhong/article/details/5953544
