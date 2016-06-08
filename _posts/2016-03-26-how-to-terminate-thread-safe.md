---
layout: post_layout
title: 如何安全的停止一个线程
time: 2016年03月26日
location: 北京
pulished: true
excerpt_separator: "Syntax"
---

我也是前一阵子看到了一个博客谈论停止AsyncTask 的cancel(boolean) 方法时，想起了如何能够安全的突出一个线程，是的线程退出，释放他所持的的其他对象的引用。AsyncTask 的cancel() 方法调用后，只有等到doInBackground() 执行后 才能停止执行onPostExecute()，但是万一doInBackground() 发生阻塞，或是它是一个相当耗时的操作肿么办，哈哈，所以说还得看看，如何能够安全的、迅速的停止一个线程。

网上关于这方面的文章也不少，大致分为了三种方法：

1.使用退出标志位，让线程随时判断该标记为；

2.使用stop方法强行终止线程（它和stop、suspend、resume一样，也可能发生不可预料的结果，所以depressed）。 

 3.使用interrupt方法中断线程。

## 使用退出标志位

这种方法简介高效，我们看一下使用该方法包装好点线程类：

    static class MyThread extends Thread{
		public volatile MyThread mythread;
		public int SleepTime = 1000;
		
		public MyThread(){
			mythread = (MyThread) Thread.currentThread();
		}
		public void stopIt(){
			mythread = null;
		}
		public void run() {
			super.run();
			 Thread thisThread = Thread.currentThread(); 
			while (mythread == thisThread) { 
				//do something
	            /*try { 
	                Thread.sleep(SleepTime); 
	            } catch (InterruptedException e){ 
	            } */
		}
	}
	}
    
 这个需要注意一点，mythread 使用了volatile 来修饰，目的是为了保持mythread在任何位置读写的统一，由于我们的代码相对简单，并不能感受到它的作用，但当我们的线程变得复杂后，就要考虑到线程安全问题了。后续我们可能会涉及到线程安全以及同步的总结哦。	
 
##  使用stop方法强行终止线程
使用stop()来停止一个正在运行的线程是一个不明智切危险的方法，他所造成的有时远比想象的要错综复杂，尽管他的确能快速的将一个线程扼杀。为了说明这一点，我们举一个简单的例子。

    tatic class MyThread2 extends Thread{
		public void run() {
			super.run();
			 a = 1;
			 b= 2;
		}
	}
    
例子过于简单，但是能够清晰的表示一个使用stop() 来停止潜在的危险，当我们试图去sop()这个线程的时候，run() 方法正好运行到a = 1,这一步，b = 2没有来的及执行，计入 a 和 b共同控制着某个事件的执行，那么这是讲发生你不期望的错误。使用stop()来停止线程由于已经遭到废弃，所以你只要别再使用就OK了。

## 使用interrupt方法中断线程
    
 这也是一种被广泛介绍的方法，stop() 不能使用，那么在有关线程的api中也还能查到interrupt是关于停止线程的 。然而interrupt并不是停止线程，他只是试图去中断当前线程，那么试想一下，如果一个线程现在正处于sleep() 或 wait() 状态，而我们调用了interrupt，会出现什么情况，没错，会发生异常，因为sleep() 或 wait() 是无法被中断的，那么java中提供了抢大的异常处理机制，你的程序不会因为某个地方抛出异常而crash掉，所以我们只需要在catch中处理此异常或者什么都不做即可，使用情况有两种。
 
 1，类似于使用标志位使用，大多数情况下线程使用了循环操作
 
   static class MyThread3 extends Thread{
		public volatile MyThread mythread;
		public int SleepTime = 1000;
		
		public MyThread3(){
			mythread = (MyThread) Thread.currentThread();
		}
		public void stopIt(){
			Thread smythread = mythread;
			mythread = null;
	        if (smythread != null) {
	        	smythread.interrupt();
	        }
		}
          public void run() {
              super.run();
               Thread thisThread = Thread.currentThread(); 
              while (isInterrupted()) { 
                  //do something
			}
		}
	}


2.针对于线程处于sleep() 或 wait()状态时，通过异常来停止线程

   static class MyThread3 extends Thread{
		public volatile MyThread mythread;
		public int SleepTime = 1000;
		
		public MyThread3(){
			mythread = (MyThread) Thread.currentThread();
		}
		public void stopIt(){
			Thread smythread = mythread;
			mythread = null;
	        if (smythread != null) {
	        	smythread.interrupt();
	        }
		}
		public void run() {
			super.run();
			 Thread thisThread = Thread.currentThread(); 
             	try { 
                    //do something
                        Thread.sleep(SleepTime); 
                    } catch (InterruptedException e){ 
                    	throw new RuntimeException("Interrupted",e);
	            } 
			}
		}
	}

 
**另外还有两种情况需要我们注意：**

### 处于IO阻塞状态线程的停止
    
 Java中的输入输出流并没有类似于Interrupt的机制，但是Java的InterruptableChanel接口提供了这样的机制，任何实现了InterruptableChanel接口的类的IO阻塞都是可中断的，中断时抛出ClosedByInterruptedException，也是由Thread对象调用Interrupt方法完成中断调用。IO中断后将关闭通道。
        以文件IO为例，构造一个可中断的文件输入流的代码如下：   
    
   	 new InputStreamReader(  
           Channels.newInputStream(  
                   (new FileInputStream(FileDescriptor.in)).getChannel())));   
    
 实现InterruptableChanel接口的类包括FileChannel,ServerSocketChannel, SocketChannel, Pipe.SinkChannel andPipe.SourceChannel，也就是说，原则上可以实现文件、Socket、管道的可中断IO阻塞操作。
        虽然解除IO阻塞的方法还可以直接调用IO对象的Close方法，这也会抛出IO异常。但是InterruptableChanel机制能够使处于IO阻塞的线程能够有一个和处于中断等待的线程一致的线程停止方案。
        
### 处于大数据IO读写中的线程停止
    
 处于大数据IO读写中的线程实际上处于运行状态，而不是等待或阻塞状态，因此上面的interrupt机制不适用。线程处于IO读写中可以看成是线程运行中的一种特例。停止这样的线程的办法是强行close掉io输入输出流对象，使其抛出异常，进而使线程停止。
        最好的建议是将大数据的IO读写操作放在循环中进行，这样可以在每次循环中都有线程停止的时机，这也就将问题转化为如何停止正在运行中的线程的问题了。   
    
 后两种情况参考的博客 [如何终止java线程][1] 。
    


  [1]: http://blog.csdn.net/anhuidelinger/article/details/11746365