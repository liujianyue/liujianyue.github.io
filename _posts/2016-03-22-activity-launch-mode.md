---
layout: post_layout
title: 关于Activity 启动模式及你所应该知道的一切
time: 2016年03月21日
location: 北京
pulished: true
excerpt_separator: "Syntax"
---

对于启动模式，官方文档这样解释：启动模式指示一个activity应该如何被加载，有四种可以和Intent结合使用的activity标记位，从而决定当一个activity被调用时，它将如何去相应这个intent，他们分别是：
 "standard" 
"singleTop" 
"singleTask"
"singleInstance"
在Android中，承载启动模式的载体是任务栈(activity stack)，它的功能类似于函数调用中的栈，Activity进入栈的顺序代表了，activity的启动顺序，先启动的Activity的被压入栈底部。我们接下来详细讲解这四种启动模式。
## 使用方式

    <activity
            android:name=".SingleTaskActivity"
            android:label="singleTask launchMode"
            android:launchMode="singleTask">
            
## 各种模式

### standard ：默认启动模式

如果我们不对launchMode加以声明，我们使用就是standard模式。每当我们启动一个这样的一个activity A 时，都会在当前调用的栈中生成一个A，试想一下加入有十个栈调用过A，那么就将产生十个A，每个A都有各自的生命周期，onCreate、onResume 等都会被调用。关于这种默认简单好理解。

### singleTop：栈顶复用模式
如果新的activity已经位于栈顶，那么这个Activity不会被重写创建，同时它的onNewIntent方法会被调用，通过此方法可重新传递intent，达到清除之前oncreate 中信息的目的。如果栈顶不存在该Activity的实例，则情况与standard模式相同。需要注意的是这个Activity它的onCreate()方法不会被调用，因为它并没有发生改变。 
此时你是不是在想，我们为什么会需要这样模式呢，什么情况下使用这种模式呢？说到此我们不得不谈谈
onNewIntent().在当前模式下，如果Activity A 处于任务栈的顶端，也就是说之前打开过的Activity，现在处于onPause、onStop 状态的话，其他应用再发送Intent的话，执行顺序将为：
		onNewIntent，onRestart，onStart，onResume。
 总结了一下几点singleTop模式使用的场景，大家可以类比其他场景：
 
 a.加入你正处在默认app的注册界面，当信息填写完成后，你点击了“完成”按钮，但是在短暂的时间内UI并没有相应该事件，结果你重复按了一次，若果使用当前模式，则不会因为你过多点击按钮而产生多个活动页面；(当然处理按钮多次连续点击的方法很多github上游开源库，后续我也会对这方面写点东西^o^)
 
 b.当点活动页面下开启了一个后台service，用以完成后台耗时操作，完成后对当前活动页面做一些事件处理等，而在service没有完成任务时你按了home键，如果此时是singleTop模式，那么service完成任务后将会复用你当前栈顶的活动页面，并且或用onNewIntent 接受的intent 覆盖你之前的intent。
 
 使用singleTop 厘清以下几点即可：
 
 1.当前栈中已有该Activity的实例并且该实例位于栈顶时，不会新建实例，而是复用栈顶的实例，并且会将Intent对象传入，回调onNewIntent方法；
 
 2.当前栈中已有该Activity的实例但是该实例不在栈顶时，其行为和standard启动模式一样，依然会创建一个新的实例；
 
 3.当前栈中不存在该Activity的实例时，其行为同standard启动模式
 
 4.standard和singleTop启动模式都是在原任务栈中新建Activity实例，不会启动新的Task，即使你指定了taskAffinity属性(后续会讲到taskAffinity等多个属性)。
 
###  singleTask：栈内复用模式

这种模式是使用最多也是最为复杂的模式，我将它的使用情况分为以下几点概括：

检测目标Activity 属性 taskAffinity 是否为空（如果没有指定taskAffinity，则为当前应用的包名，也定为已制定）

1.假如为空，则会在系统中检测是否存在目标Activity，若有则将该栈中目标Activity以上的activity全部出栈(销毁)，如果不存在目标ACtivity，则会在当前栈栈顶创建目标Activity；

2.假如指定了属性 taskAffinity，则首先会检测此属性与当前活动activity的taskAffinity是否一致，如果一直那么处理情况和 1 相同，如果不一致，则会去检测与属性taskAffinity指示的栈是否存在，如存在对栈的处理和情况1 相同，若不存在，则会创建名为taskAffinity属性的栈，并在此栈中创建目标Activity，此战中只有一个目标activity。

而关于singleTask的使用情况我也做了一下划分：

1.系统提示型。我相信很多小伙伴在试图启动一Activity时，碰到过这种错误：

    android.util.AndroidRuntimeException: Calling startActivity from outside of an Activity context requires the FLAG_ACTIVITY_NEW_TASK flag. Is this really what you want?

什么意思？这是因为非Activity类型的Context并没有所谓的任务栈，所以待启动的Activity就找不到栈了。解决这个问题的方法就是为待启动的Activity指定FLAG_ACTIVITY_NEW_TASK标记位，这样启动的时候就为它创建一个新的任务栈，而此时Activity是以singleTask模式启动的。所有这种用Application启动Activity的方式不推荐使用，Service，BroadCastReciver同Application。

2.需要借助该模式完美我们的应用。试想一下现在有这样一种情况，当前activity的显示的数据需要随时和和我们记录的数据保持同步，不希望用户在不同的activity看到不同的版本，以至于给用户篡改数据或是由于数据版本不同导致迷惑用户的机会，此时使用singleTask模式是不二选择。那我们最常用的QQ来说，我们不希望我们的聊天界面因为我们的跳转而由多个activity显示出不同的聊天记录，尽管可以通过其他手段来避免这一弊端，不过那将变得很费事，为违反了android的初衷。

3.在Android应用程序开发的时候，从一个Activity A启动另一个Activity B并传递一些数据到新的Activity B上非常简单，但是当需要让后台运行的Activity B回到前台并传递一些数据可能就会存在一点点小问题。
首先，在默认情况下，当您通过Intent启到一个Activity的时候，就算已经存在一个相同的正在运行的Activity,系统都会创建一个新的Activity实例并显示出来。为了不让Activity实例化多次，我们使用该模式。不过在这种情况应该注意一点,带给你调用onNewIntent 时，尽管你传递进去了新的intent 但是，在整个当前活动页面所保管的intent仍然是之前调用oncreate传入的intent，所以你最好按照如下方法来做：

    public void onCreate(Bundle savedInstanceState) {
      super.onCreate(savedInstanceState);
      setContentView(R.layout.main);
      processExtraData();
	}
        protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);
         setIntent(intent);//必须要存储新的intent，否则将会返回旧的intent
        processExtraData()
	}
        private void processExtraData(){
        Intent intent = getIntent();
      //让oncreate 和 onNewIntent 调用相同的data处理函数
	}
上边还有一个小技巧，让oncreate 和 onNewIntent 调用相同的data处理函数，这样可防止activity在后台被系统销毁(异常终止)时，能够用新的intent来重新创建activity，哈哈，很有用。

### singleInstance：全局唯一模式

这个模式非常接近于singleTask，系统中只允许一个Activity的实例存在。区别在于持有这个Activity的任务中只能有一个Activity：即这个单例本身。由于栈内复用的特性，后续的请求均不会创建新的Activity实例，除非这个特殊的任务栈被销毁了。以singleInstance模式启动的Activity在整个系统中是单例的，如果在启动这样的Activiyt时，已经存在了一个实例，那么会把它所在的任务调度到前台，重用这个实例。

## 其他相关属性

### android:taskAffinity 

官网是这样说的，鸟语,没有太生僻的单词不翻译了(·_·):

		The task that the activity has an affinity for. Activities with the same affinity conceptually belong to the same task (to the same "application" from the user's perspective). The affinity of a task is determined by the affinity of its root activity. 
		The affinity determines two things — the task that the activity is re-parented to (see the allowTaskReparenting attribute) and the task that will house the activity when it is launched with the FLAG_ACTIVITY_NEW_TASK flag. 
		By default, all activities in an application have the same affinity. You can set this attribute to group them differently, and even place activities defined in different applications within the same task. To specify that the activity does not have an affinity for any task, set it to an empty string. 
		If this attribute is not set, the activity inherits the affinity set for the application (see the <application> element's taskAffinity attribute). The name of the default affinity for an application is the package name set by the <manifest> element.

### android:allowTaskReparenting 
	
这个属性用来标记一个Activity实例在当前应用退居后台后，是否能从启动它的那个task移动到有共同affinity的task，“true”表示可以移动，“false”表示它必须呆在当前应用的task中，默认值为false。如果一个这个Activity的元素没有设定此属性，设定在上的此属性会对此Activity起作用。例如在一个应用中要查看一个web页面，在启动系统浏览器Activity后，这个Activity实例和当前应用处于同一个task，当我们的应用退居后台之后用户再次从主选单中启动应用，此时这个Activity实例将会重新宿主到Browser应用的task内，在我们的应用中将不会再看到这个Activity实例，而如果此时启动Browser应用，就会发现，第一个界面就是我们刚才打开的web页面，证明了这个Activity实例确实是宿主到了Browser应用的task内。 

### android:alwaysRetainTaskState

这个属性用来标记应用的task是否保持原来的状态，“true”表示总是保持，“false”表示不能够保证，默认为“false”。此属性只对task的根Activity起作用，其他的Activity都会被忽略。 默认情况下，如果一个应用在后台呆的太久例如30分钟，用户从主选单再次选择该应用时，系统就会对该应用的task进行清理，除了根Activity，其他Activity都会被清除出栈，但是如果在根Activity中设置了此属性之后，用户再次启动应用时，仍然可以看到上一次操作的界面。
这个属性对于一些应用非常有用，例如Browser应用程序，有很多状态，比如打开很多的tab，用户不想丢失这些状态，使用这个属性就极为恰当。

### android:clearTaskOnLaunch 

这个属性用来标记是否从task清除除根Activity之外的所有的Activity，“true”表示清除，“false”表示不清除，默认为“false”。同样，这个属性也只对根Activity起作用，其他的Activity都会被忽略。 如果设置了这个属性为“true”，每次用户重新启动这个应用时，都只会看到根Activity，task中的其他Activity都会被清除出栈。如果我们的应用中引用到了其他应用的Activity，这些Activity设置了allowTaskReparenting属性为“true”，则它们会被重新宿主到有共同affinity的task中。

### android:finishOnTaskLaunch 

这个属性和android:allowReparenting属性相似，不同之处在于allowReparenting属性是重新宿主到有共同affinity的task中，而finishOnTaskLaunch属性是销毁实例。如果这个属性和android:allowReparenting都设定为“true”，则这个属性好些。

这篇博客没有使用大量的代码辅以说明，相信大家在实际项目中也会用到这些。写这篇博客供自己和大家学习交流，有不对之处还望指出，一起学习进步嘛？