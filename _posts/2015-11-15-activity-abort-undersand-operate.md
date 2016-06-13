---
layout: post_layout
title: 关于Activity的生命周期以及异常销毁的理解
time: 2015年11月15日
location: 北京
pulished: true
excerpt_separator: "Syntax"
---

当一个ACtivity的呈现在我们面前时，开发者预期的事情是你会一直操作当前页面或者按back键返回，因为开发者不能预测到一个页面什么时候被系统调到后台，所以Android提供在这种情况下的处理机制。我们看一下下面几种情况，哪种属于activity的正常退出，哪种属于异常销毁：

a.屏幕的方向xuanzhuan
b.按下home键
c.人为或是系统调用了其他activity
d.内存不错，app被kill
e.锁屏
f.按back键

显然除了按back键以外，其他都属于异常销毁，Android为我们提供了针对异常销毁的处理机制，相信大家都很熟悉，这几乎是面试基本会问到的内容，使用onSaveInstanceState(),能够很好的处理页面的异常销毁。不过你可能很清楚这一点，但是偶一个问题，onSaveInstanceState()到底在什么时候会调用？
关于这个问题先不急于回答，在我刚学习Android的时候，遇到不懂的问题第一时间找百度，久而久之我发现我所学会除了微不足道的android技能外就剩下如何在百度的汪洋大海中寻找自己想要的代码的能力了，然并卵，后来博客中我发现大神们比较推荐的一是阅读高质量的源码，二是及时的去阅读Android官方文。我深受启发，我试着去阅读官方文档，其中英语内容并不会包含比较复杂的单词，慢慢的我发现，我之前在网上看到的代码存在很大问题，有的并不是官方文档推荐的，茅塞顿开，恍然大悟啊。为什么要说着些，因为在写这篇博客前，我又重新阅读了官方关于Activity的文档，它囊括了几乎activity生命周期的所有东西，所以总结Activity的异常销毁，我觉得有必要阅读文档，在此基础上在加上我的一些其他理解。

Activity 有三种关键的状态需要引起我们的注意：

1.The entire lifetime of an activity happens between the first call to onCreate(Bundle) through to a single final call to onDestroy(). An activity will do all setup of "global" state in onCreate(), and release all remaining resources in onDestroy(). For example, if it has a thread running in the background to download data from the network, it may create that thread in onCreate() and then stop the thread in onDestroy(). 

2.The visible lifetime of an activity happens between a call to onStart() until a corresponding call to onStop(). During this time the user can see the activity on-screen, though it may not be in the foreground and interacting with the user. Between these two methods you can maintain resources that are needed to show the activity to the user. For example, you can register a BroadcastReceiver in onStart() to monitor for changes that impact your UI, and unregister it in onStop() when the user no longer sees what you are displaying. The onStart() and onStop() methods can be called multiple times, as the activity becomes visible and hidden to the user. 

3.The foreground lifetime of an activity happens between a call to onResume() until a corresponding call to onPause(). During this time the activity is in front of all other activities and interacting with the user. An activity can frequently go between the resumed and paused states -- for example when the device goes to sleep, when an activity result is delivered, when a new intent is delivered -- so the code in these methods should be fairly lightweight. 

在activity的各个生命周期中，我就的我们需要注意以下几点：
1.onCreate()：Called when the activity is first created. This is where you should do all of your normal static set up: create views, bind data to lists, etc. This method also provides you with a Bundle containing the activity's previously frozen state, if there was one. 
Always followed by onStart().
当activity第一次被调用时执行，我们的数据加载、view初始化硬蛋在这里执行，并且我们也可使用bundle来恢复之前的状态。

2.onRestart()：Called after your activity has been stopped, prior to it being started again. 
Always followed by onStart()
activity被停止时调用，只在重启。

3.onStart()：Called when the activity is becoming visible to the user. 
Followed by onResume() if the activity comes to the foreground, or onStop() if it becomes hidden.
此时activity已经可见，但是不能与用户交互，在它调用后不一定会调用onresume()，因为有可能退出当前页面而调用onstop().

4.onResume():Called when the activity will start interacting with the user. At this point your activity is at the top of the activity stack, with user input going to it. 
Always followed by onPause().
该方法调用后，当天页面可以和用户进行交互，并且此时我们的页面有可能伴随输入法出现在当前活动栈的最顶端。

5.onpause() :Called when the system is about to start resuming a previous activity. This is typically used to commit unsaved changes to persistent data, stop animations and other things that may be consuming CPU, etc. Implementations of this method must be very quick because the next activity will not be resumed until this method returns. 
Followed by either onResume() if the activity returns back to the front, or onStop() if it becomes invisible to the user.
这句话大意是当系统试图唤醒另一activity时，将被调用。onpause典型用法是去存储一些持久化的数据，停止动画或是其他一些比较小号CPU资源的事。并且需要注意的事，最好不要字其中做一些比较耗时的操作，因为只有这个歌方法返回了，另一个actiity才会resume().当然如果此acrivity通过onresume方法又回到了前台，那么该方法仍然有可能再一次被执行。

6.onStop()：Called when the activity is no longer visible to the user, because another activity has been resumed and is covering this one. This may happen either because a new activity is being started, an existing one is being brought in front of this one, or this one is being destroyed. 
Followed by either onRestart() if this activity is coming back to interact with the user, or onDestroy() if this activity is going away.
一旦调用该方法，当前页面将不再可见，会被另一个页面覆盖掉。此时activity即有可能重新回到前台，执行ionrestart(),也有可能执行onDestory()，而彻底销毁。

7.onDestroy()：The final call you receive before your activity is destroyed. This can happen either because the activity is finishing (someone called finish() on it, or because the system is temporarily destroying this instance of the activity to save space. You can distinguish between these two scenarios with the isFinishing() method.

有两种情况将会调用该方法：一是认为退出当前活动页面，二是系统为了节省空间而临时销毁当前activity实例。但是我们可以调用 isFinishing()来判断是哪种情况，true代表前一种前一种	


8.onSaveInstanceState (Bundle outState) ：This method is called before an activity may be killed so that when it comes back some time in the future it can restore its state. For example, if activity B is launched in front of activity A, and at some point activity A is killed to reclaim resources, activity A will have a chance to save the current state of its user interface via this method so that when the user returns to activity A, the state of the user interface can be restored via onCreate(Bundle) or onRestoreInstanceState(Bundle). 

这个方法将会在一个activity被杀死但他仍有机会恢复到之前状态的时候，恢复数据时，将会调用onCreate(Bundle) 或onRestoreInstanceState(Bundle). 两者有啥区别？前者在activity第一次调用或者重新创建时都会调用，所以Bundle需要判断是否为空来甄别这两种状态，而后者一旦被调用，Bundle一定不为空，不需要判空。既然说一个activity被杀死会调用该方法，那么什么情况下activity会被杀死或比较容易被被杀死呢 ？
a.当前activity由前台转后台时；

b.屏幕旋转等异常情况，activity重建；

c.activity 按照优先级从后台到前台被杀死；

但是只有后两种情况会调用onRestoreInstance()来恢复数据。

后边还有一句：The default implementation takes care of most of the UI per-instance state for you by calling onSaveInstanceState() on each view in the hierarchy that has an id, and by saving the id of the currently focused view (all of which is restored by the default implementation of onRestoreInstanceState(Bundle)). If you override this method to save additional information not captured by each individual view, you will likely want to call through to the default implementation, otherwise be prepared to save all of the state of each view yourself.

大意是，大多数的android自带的view都已经实现了onSaveInstanceState() 方法，前提是这些view需要有id，且唯一（否则会相互覆盖），你只需要关心那些不会被系统保存的数据。其中提到了一个词语hierarchy，系统会逐级通知下层view去执行onSaveInstanceState()。

If called, this method will occur before onStop(). There are no guarantees about whether it will occur before or after onPause()。
这句话则说明了onSaveInstanceState() 调用发生在onStop()之前，但是并不能确定和onPause()调用顺序的先后关系。

而关于屏幕旋转导致activity销毁而重建，我们是不有方法是activity不重建呢，答案是肯定的，我们可以通过对屏幕旋转、语言、等一列配置来自定义对这个情况的处理，详细内容不再赘述。

现在又这样一个问题，我们知道activity异常销毁时，有重建恢复数据的机制，那么如果我们需要回复的数据很大，并且不是普通类型，那该怎么办呢？官方文档曾经给过这样一个解决方法，这也是它推荐使用的方法：



















































