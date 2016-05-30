---
layout: post_layout
title: JAVA 并发开发的一点总结
time: 2015年12月25日
location: 北京
pulished: true
excerpt_separator: "Syntax"
---
在公司项目最近逐渐的少了，（项目少了，公司不景气了）但是还是每天坚持阅读其他大神们的博客文章，以及一些高质量的源码，由于本人的工作比较接近Android frameworks 层的东东，所以对源码的阅读总是会带着愉悦的心情来阅读。最近心血来潮，决定总结一篇关于java并发开发的一点知识，恕在下愚钝，能力有限，写出的博客质量有待提高，主要还是供自己理解和运用，如果哪位小伙伴能够指点一二，不胜感激。
JAVA 并发操作，我目前遇到的可能有以下几种方式：

 1. synchronized 关键字；
 2. ThreadPoolExecutor 线程池 及其扩展；
 3. 信号量的使用（Semaphore）；
 4. volatile关键字的使用；
 5. ThreadLocal 使用；
 6. CountDownLathch 类型使用；
 7. CopyOnWriteAarrayList<T> 等使用；
 8. Collections.synchronizedList 等使用；
 9. Object 对象自带的 wait()、notify()、notifyAll() 的使用；
 10. blockingquene 的使用；

## 一， synchronized 关键字
synchronized 关键字，代表这个方法加锁,相当于不管哪一个线程A每次运行到这个方法时,都要检查有没有其它正在用这个方法的线程B（或者C D等）,有的话要等正在使用这个方法的线程B（或者C D）运行完这个方法后再运行此线程A,没有的话,直接运行它包括两种用法：synchronized 方法和 synchronized 块。
### synchronized 方法

    public synchronized void accessObj(int newObj);

synchronized 方法控制对类成员变量的访问：每个类实例对应一把锁，每个 synchronized 方法都必须获得调用该方法的类实例的锁方能执行，否则所属线程阻塞，方法一旦执行，就独占该锁，直到从该方法返回时才将锁释放，此后被阻塞的线程方能获得该锁，重新进入可执行状态。这种机制确保了同一时刻对于每一个类实例，其所有声明为 synchronized 的成员函数中至多只有一个处于可执行状态（因为至多只有一个能够获得该类实例对应的锁），从而有效避免了类成员变量的访问冲突（只要所有可能访问类成员变量的方法均被声明为 synchronized）。　　在 Java 中，不光是类实例，每一个类也对应一把锁，这样我们也可将类的静态成员函数声明为 synchronized ，以控制其对类的静态成员变量的访问。　　synchronized 方法的缺陷：若将一个大的方法声明为synchronized 将会大大影响效率，典型地，若将线程类的方法 run()声明为 synchronized ，由于在线程的整个生命期内它一直在运行，因此将导致它对本类任何 synchronized 方法的调用都永远不会成功。当然我们可以通过将访问类成员变量的代码放到专门的方法中，将其声明为 synchronized ，并在主方法中调用来解决这一问题，但是 Java 为我们提供了更好的解决办法，那就是 synchronized 块。

### synchronized 块

    public void method3(SomeObject so){
      synchronized(so)
      {
         //允许访问代码块
      }
    }
synchronized 块是这样一个代码块，其中的代码必须获得对象 syncObject （如前所述，可以是类实例或类）的锁方能执行，具体机制同前所述。由于可以针对任意代码块，且可任意指定上锁的对象，故灵活性较高。

### 对synchronized (this)的一些理解

一、当两个并发线程访问同一个对象object中的这个synchronized(this)同步代码块时，一个时间内只能有一个线程得到执行。另一个线程必须等待当前线程执行完这个代码块以后才能执行该代码块。　　
二、当一个线程访问object的一个synchronized(this)同步代码块时，其他线程对object中所有其它synchronized(this)同步代码块的访问将被阻塞。　　
三、然而，当一个线程访问object的一个synchronized(this)同步代码块时，另一个线程仍然可以访问该object中的除synchronized(this)同步代码块以外的部分。　
四、当一个线程访问object的一个synchronized(this)同步代码块时，它就获得了这个object的对象锁。结果，其它线程对该object对象所有同步代码部分的访问都被暂时阻塞。

### 与静态关键字static一起使用

    Class test{
    public synchronized static void methodA()   // 同步的static 函数
    {
        //…
    }
    public void methodB()
    {
       synchronized(test.class)   // class literal(类名称字面常量)
    }
}
在上例代码中methodA 和 methodB 都实现了对类的多控制，这两种方法实现的效果相同，不过笔者有一个疑问就是如果在methodB中将test.class 替换成obj.class obj为当前类实例，两者效果是否相同，有何区别。
需要注意：
A: synchronized static是某个类的范围，synchronized static cSync{}防止多个线程同时访问这个    类中的synchronized static 方法。它可以对类的所有对象实例起作用。
B: synchronized 是某实例的范围，synchronized isSync(){}防止多个线程同时访问这个实例中的synchronized 方法。
C:synchronized methods(){} 与synchronized（this）{}之间没有什么区别，只是 synchronized methods(){} 便于阅读理解，而synchronized（this）{}可以更精确的控制冲突限制访问区域，有时候表现更高效率。
D:synchronized关键字是不能继承的,我想这一点也是很值得注意的，继承时子类的覆盖方法必须显示定义成synchronized

[下一篇：ThreadPoolExecutor 线程池 及其扩展]
