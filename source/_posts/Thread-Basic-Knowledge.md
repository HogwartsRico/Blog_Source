---
title: 多线程基础知识
date: 2020-11-20 17:31:59
tags: Java
---

# 进程和线程

谈到多线程，就得先讲进程和线程的概念。

**进程**  

进程可以理解为受操作系统管理的基本运行单元。360浏览器是一个进程、WPS也是一个进程，正在操作系统中运行的".exe"都可以理解为一个进程  

![](5.png) 



**线程**

进程中独立运行的子任务就是一个线程。像QQ.exe运行的时候就有很多子任务在运行，比如聊天线程、好友视频线程、下载文件线程等等。

 

**为什么要使用多线程**

如果使用得当，线程可以有效地降低程序的开发和维护等成本，同时提升复杂应用程序的性能。具体说，线程的优势有：

**1、发挥多处理器的强大能力**

现在，多处理器系统正日益盛行，并且价格不断降低，即时在低端服务器和中断桌面系统中，通常也会采用多个处理器，这种趋势还在进一步加快，因为通过提高时钟频率来提升性能已变得越来越困难，处理器生产厂商都开始转而在单个芯片上放置多个处理器核。试想，如果只有单个线程，双核处理器系统上程序只能使用一半的CPU资源，拥有100个处理器的系统上将有99%的资源无法使用。多线程程序则可以同时在多个处理器上执行，如果设计正确，多线程程序可以通过提高处理器资源的利用率来提升系统吞吐率。

**2、在单处理器系统上获得更高的吞吐率**

如果程序是单线程的，那么当程序等待某个同步I/O操作完成时，处理器将处于空闲状态。而在多线程程序中，如果一个线程在等待I/O操作完成，另一个线程可以继续运行，使得程序能在I/O阻塞期间继续运行。

**3、建模的简单性**

通过使用线程，可以将复杂并且异步的工作流进一步分解为一组简单并且同步的工作流，每个工作流在一个单独的线程中运行，并在特定的同步位置进行交互。我们可以通过一些现有框架来实现上述目标，例如Servlet和RMI，框架负责解决一些细节问题，例如请求管理、线程创建、负载平衡，并在正确的时候将请求分发给正确的应用程序组件。编写Servlet的开发人员不需要了解多少请求在同一时刻要被处理，也不需要了解套接字的输入流或输出流是否被阻塞，当调用Servlet的service方法来响应Web请求时，可以以同步的方式来处理这个请求，就好像它是一个单线程程序。

**4、异步事件的简化处理**

服务器应用程序在接受多个来自远程客户端的套接字连接请求时，如果为每个连接都分配其各自的线程并且使用同步I/O，那么就会降低这类程序的开发难度。如果某个应用程序对套接字执行读操作而此时还没有数据到来，那么这个读操作将一直阻塞，直到有数据到达。在单线程应用程序中，这不仅意味着在处理请求的过程中将停顿，而且还意味着在这个线程被阻塞期间，对所有请求的处理都将停顿。为了避免这个问题，单线程服务器应用程序必须使用非阻塞I/O，但是这种I/O的复杂性要远远高于同步I/O，并且很容易出错。然而，如果每个请求都拥有自己的处理线程，那么在处理某个请求时发生的阻塞将不会影响其他请求的处理。



# 线程生命周期 

在开始所有的线程方法介绍前，先来看下线程的几种状态.这样在讲下面的线程方法时，可以直到线程当前是进入到哪种状态了。  

Java中的线程的生命周期大体可分为5种状态。  

1. **新建(NEW)**：新创建了一个线程对象。

2. **可运行(RUNNABLE)**：线程对象创建后，其他线程(比如main线程）调用了该对象的start()方法。该状态的线程位于可运行线程池中，等待被线程调度选中，获取cpu 的使用权 。

3. **运行(RUNNING)**：可运行状态(runnable)的线程获得了cpu 时间片（timeslice） ，执行程序代码。
4. **阻塞(BLOCKED)**：阻塞状态是指线程因为某种原因放弃了cpu 使用权，也即让出了cpu timeslice，暂时停止运行。直到线程进入可运行(runnable)状态，才有机会再次获得cpu timeslice 转到运行(running)状态。阻塞的情况分三种： 

> (一). 等待阻塞：运行(running)的线程执行wait()方法，JVM会把该线程放入等待队列(waitting queue)中。
> (二). 同步阻塞：运行(running)的线程在获取对象的同步锁时，若该同步锁被别的线程占用，则JVM会把该线程放入锁池(lock pool)中。
> (三). 其他阻塞：运行(running)的线程执行Thread.sleep(long ms)或t.join()方法，或者发出了I/O请求时，JVM会把该线程置为阻塞状态。当sleep()状态超时、join()等待线程终止或者超时、或者I/O处理完毕时，线程重新转入可运行(runnable)状态。

5. **死亡(DEAD)**：线程run()、main() 方法执行结束，或者因异常退出了run()方法，则该线程结束生命周期。死亡的线程不可再次复生。



![](1.png) 

# Thread中的实例方法

从Thread类中的实例方法和类方法的角度讲解Thread中的方法，这种区分的角度也有助于理解多线程中的方法。实例方法，只和实例线程（也就是new出来的线程）本身挂钩，和当前运行的是哪个线程无关。看下Thread类中的实例方法：

## start()  

**start()方法的作用讲得直白点就是通知"线程规划器"**，此线程可以运行了，**正在等待CPU调用线程对象得run()方法**，产生一个**异步执行**的效果。 

**调用start()方法的顺序不代表线程启动的顺序，线程启动顺序具有不确定性**。

## run()

线程开始执行，虚拟机调用的是线程run()方法中的内容。 

**如果只有run()没有start()，Thread实例run()方法里面的内容是没有任何异步效果的**，全部被main函数执行。换句话说，**只有run()而不调用start()启动线程是没有任何意义的** 

## isAlive()

测试线程是否处于活动状态，只要线程启动且没有终止，方法返回的就是true。 

## getId()

在一个Java应用中，有一个long型的全局唯一的线程ID生成器**threadSeqNumber**，每new出来一个线程都会把这个自增一次，并赋予线程的tid属性



## getName()

我们new一个线程的时候，可以指定该线程的名字，也可以不指定。如果指定，那么线程的名字就是我们自己指定的，getName()返回的也是开发者指定的线程的名字；如果不指定，那么Thread中有一个**int型全局唯一的线程初始号生成器threadInitNum**，Java先把threadInitNum自增，然后以"Thread-threadInitNum"的方式来命名新生成的线程

## getPriority()和setPriority(int newPriority)

这两个方法用于获取和设置线程的优先级，**优先级高的CPU得到的CPU资源比较多**，设置优先级有助于帮"线程规划器"确定下一次选择哪一个线程优先执行。换句话说，**两个在等待CPU的线程，优先级高的线程越容易被CPU选择执行**。 

**线程默认优先级为5，如果不手动指定，那么线程优先级具有继承性，比如线程A启动线程B，那么线程B的优先级和线程A的优先级相同** 

MyThread1.java

```java
public class MyThread1 extends Thread{
    public void run() {
        long beginTime = System.currentTimeMillis();
        for (int j = 0; j < 100000; j++){}
        long endTime = System.currentTimeMillis();
        System.out.println("我是高优先级的 use time = " +   (endTime - beginTime));
    }
}
```

MyThread2.java

```
public class MyThread2 extends Thread{
    public void run() {
        long beginTime = System.currentTimeMillis();
        for (int j = 0; j < 100000; j++){}
        long endTime = System.currentTimeMillis();
        System.out.println("我是低优先级的 use time = " +   (endTime - beginTime));
    }
}
```

main

```java
 public static void main(String[] args) {
        for (int i = 0; i < 5; i++) {
            MyThread1 t1 = new MyThread1();
            t1.setPriority(6);
            t1.start();
            MyThread2 t2 = new MyThread2();
            t2.setPriority(4);
            t2.start();
        }
    }
```



![](6.png) 

从这个例子我们得出结论：**CPU会尽量将执行资源让给优先级比较高的线程**。

## isDaemon、setDaemon

讲解两个方法前，首先要知道理解一个概念。Java中有两种线程，一种是用户线程，一种是守护线程。守护线程是一种特殊的线程，它的作用是为其他线程的运行提供便利的服务，最典型的应用便是GC线程。如果进程中不存在非守护线程了，那么守护线程自动销毁，因为没有存在的必要，为别人服务，结果服务的对象都没了，当然就销毁了。理解了这个概念后，看一下例子：

过setDaemon(true)来设置线程为“守护线程”；将一个用户线程设置为守护线程的方式是在线程启动用线程对象的setDaemon方法。

MyThread.java

```java

public class MyThread extends Thread{
    private int i = 0;
    public void run() {
        try {
            while (true) {
                i++;
                System.out.println("i = " + i);
                Thread.sleep(1000);
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

main

```java
public static void main(String[] args) {
        try {
            MyThread mt = new MyThread();
            mt.setDaemon(true);
            mt.start();
            Thread.sleep(5000);
            System.out.println("我离开非守护线程就停止执行了，再也不打印了！");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```

将MyThread线程设置为守护线程，main函数都运行完了，自然也没有存在的必要了，就自动销毁了，因此也就没有再往下打印数字。

关于守护线程，有一个细节注意下，**setDaemon(true)必须在线程start()之前**  

## interrupt

Thread类的interrupt()方法无法中断线程。

```java
public static void main(String[] args) {
            MyThread t = new MyThread();
            t.start();
            t.interrupt();
    }

    static class MyThread extends  Thread{
        @Override
        public void run() {
            for (int i = 0; i < 500000; i++) {
                System.out.println("i = " + (i + 1));
            }
        }
    }
```

看结果还是打印到了50000。也就是说，尽管调用了interrupt()方法，但是线程并没有停止。

interrupt()方法的作用实际上是：**在线程受到阻塞时抛出一个中断信号，这样线程就得以退出阻塞状态**。换句话说，**没有被阻塞的线程，调用interrupt()方法是不起作用的**。关于这个会在之后讲中断机制的时候详细介绍



### interrupt()打断wait()

之前有说过，interrupt()方法的作用不是中断线程，而是在线程阻塞的时候给线程一个中断标识，表示该线程中断。wait()就是"阻塞的一种场景"，看一下用interrupt()打断wait()的例子：

ThreadDomain.java

```java
public class ThreadDomain {
    public void testMethod(Object lock) {
        try {
            synchronized (lock) {
                System.out.println("开始wait");
                lock.wait();
                System.out.println("结束wait()");
            }
        } catch (InterruptedException e) {
            System.out.println("wait被interrupt()打断了！");
            e.printStackTrace();
        }
    }
}
```



MyThread.java

```java
public class MyThread extends Thread{
    private Object lock;
    public MyThread(Object lock) {
        this.lock = lock;
    }
    public void run() {
        ThreadDomain td = new ThreadDomain();
        td.testMethod(lock);
    }
}
```



main

```java
 public static void main(String[] args) throws InterruptedException {
        Object lock = new Object();
        MyThread mt = new MyThread(lock);
        mt.start();
        Thread.sleep(5000);
        mt.interrupt();
    }
```

![](4.png) 

## join()

join()方法的作用是等待线程销毁。join()方法反应的是一个很现实的问题，比如main线程的执行时间是1s，子线程的执行时间是10s，但是主线程依赖子线程执行完的结果，这时怎么办？可以像生产者/消费者模型一样，搞一个缓冲区，子线程执行完把数据放在缓冲区中，通知main线程，main线程去拿，这样就不会浪费main线程的时间了。另外一种方法，就是join()了。当然也可以用 CountDownLatch 或 CyclicBarrier。后面会介绍。 but先看一下例子：

MyThread

```java
public class MyThread extends Thread{
    public void run() {
        try {
            int secondValue = (int)(Math.random() * 10000);
            System.out.println(secondValue);
            Thread.sleep(secondValue);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```



main

```java
 public static void main(String[] args) throws InterruptedException {
        MyThread mt = new MyThread();
        mt.start();
        mt.join();
        System.out.println("我想等别人执行完毕之后我再执行，我做到了");
    }
```





![](8.png) 





join()方法会使调用join()方法的线程所在的线程（也就是main线程）无限阻塞，直到调用join()方法的线程销毁为止，此例中main线程就会无限期阻塞直到mt的run()方法执行完毕。

join()方法的一个重点是要区分出和sleep()方法的区别。**join(2000)也是可以的，表示调用join()方法所在的线程最多等待2000ms**，两者的区别在于：  

**sleep(2000)不释放锁，join(2000)释放锁**，因为join()方法内部使用的是wait()，因此会释放锁。 





# Thread类中的静态方法





































# wait/notify/notifyall

线程本身是操作系统中独立的个体，但是线程与线程之间不是独立的个体，因为它们彼此之间要相互通信和协作。

想像一个场景，A线程做int型变量i的累加操作，B线程等待i到了10000就打印出i，怎么处理？一个办法就是，B线程while(i == 10000)，这样两个线程之间就有了通信，B线程不断通过轮训来检测i == 10000这个条件。

这样可以实现我们的需求，但是也带来了问题：CPU把资源浪费了B线程的轮询操作上，因为while操作并不释放CPU资源，导致了CPU会一直在这个线程中做判断操作。如果可以把这些轮询的时间释放出来，给别的线程用，就好了。

 

在Object对象中有三个方法wait()、notify()、notifyAll()，既然是Object中的方法，那每个对象自然都是有的。如果不接触多线程的话，这两个方法是不太常见的。下面看一下前两个方法：

## wait() 

wait()的作用是使当前执行代码的线程进行等待（线程进入等待阻塞），将当前线程置入"预执行队列"中，并且wait()所在的代码处停止执行，直到接到通知或被中断。**在调用wait()之前，线程必须获得该对象的锁，因此只能在同步方法/同步代码块中调用wait()方法**。 

## notify()

notify()的作用是，如果有多个线程等待，那么线程规划器**随机**挑选出一个wait的线程，对其发出通知notify()，并使它等待获取该对象的对象锁注意"等待获取该对象的对象锁"，这意味着，即使收到了通知，wait的线程也不会马上获取对象锁，必须等待notify()方法的线程释放锁才可以。**和wait()一样，notify()也要在同步方法/同步代码块中调用**。

总结起来就是，**wait()使线程停止运行，notify()使停止运行的线程继续运行**。  

PS:如果想要不随机唤醒线程，可使用Lock，以后会介绍 

示例: 

```java
public class MyThread1 extends Thread{
    private Object lock;
    public MyThread1(Object lock) {
        this.lock = lock;
    }
    @Override
    public void run() {
        try {
            synchronized (lock) {
                System.out.println("thread1开始wait time:" + System.currentTimeMillis());
                lock.wait();
                System.out.println("thread1结束 time:" + System.currentTimeMillis());
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```



```java
public class MyThread2 extends Thread{
    private Object lock;
    public MyThread2(Object lock) {
        this.lock = lock;
    }
    public void run() {
        synchronized (lock) {
            System.out.println("thread2开始 notify time:" + System.currentTimeMillis());
            lock.notify();
            System.out.println("thread2结束notify time:" + System.currentTimeMillis());
        }
    }
}
```

写个main函数，Thread.sleep(3000)也是为了保证t1先运行，这样才能看到wait()和notify()的效果： 

```java
public static void main(String[] args) throws InterruptedException {
        Object lock = new Object();
        MyThread1 t1 = new MyThread1(lock);
        t1.start();
        System.out.println("睡眠3秒后thread2进行notify");
        Thread.sleep(3000);
        MyThread2 t2 = new MyThread2(lock);
        t2.start();
    }
```

```
睡眠3秒后thread2进行notify
thread1开始wait time:1605852468080
thread2开始 notify time:1605852471083
thread2结束notify time:1605852471083
thread1结束 time:1605852471083
```

**wait()之后代码一直暂停，notify()之后代码才开始运行**。  

wait()方法可以使调用该线程的方法**释放共享资源的锁**，然后从运行状态退出，进入等待队列，直到再次被唤醒。



notify()方法可以随机唤醒等待队列中等待同一共享资源的一个线程，**并使得该线程退出等待状态，进入可运行状态** （进入可运行状态，并不意味着马上运行。需要等当前线程运行完之后）

notifyAll()方法可以使所有正在等待队列中等待同一共享资源的**全部线程**从等待状态退出，进入可运行状态 

最后，如果`wait()`方法和`notify()/notifyAll()`方法不在同步方法/同步代码块中被调用，那么虚拟机会抛出java.lang.IllegalMonitorStateException，注意一下。  



## notifyall

MyThread1和MyThread2代码就不贴了，和上面的一样

NotifyThread.java

```java
public class NotifyThread extends Thread{
    private Object lock;
    public NotifyThread(Object lock) {
        this.lock = lock;
    }
    public void run() {
        synchronized (lock) {
            System.out.println("thread开始notify time:" + System.currentTimeMillis());
            lock.notifyAll();
            System.out.println("thread结束notify time:" + System.currentTimeMillis());
        }
    }
}
```



```java
  public static void main(String[] args) throws InterruptedException {
        Object lock = new Object();
        MyThread1 t1 = new MyThread1(lock);
        MyThread2 t2 = new MyThread2(lock);
        t1.start();
        t2.start();
        System.out.println("睡眠3秒");
        Thread.sleep(3000);
        NotifyThread nt  = new NotifyThread(lock);
        nt.start();
    }
```



![](2.png) 

可以看到notifyall确实是唤醒所有wait中的线程, 利用Object对象的notifyAll()方法可以唤醒处于同一监视器下的所有处于wait的线程 。至于唤醒的顺序，就和线程启动的顺序一样，是虚拟机随机的。



## wait()释放锁

多线程的学习中，任何地方都要关注"锁"，wait()和notify()也是这样。wait()方法是释放锁的,notify是不释放锁的。写一个例子来证明一下：

ThreadDomain.java

```java
public class ThreadDomain {
    public void testMethod(Object lock) {
        try {
            synchronized (lock) {
                System.out.println(Thread.currentThread().getName() + "开始wait");
                lock.wait();
                System.out.println(Thread.currentThread().getName() + "结束wait");
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

MyThread.java

```java
public class MyThread extends Thread{
    private Object lock;
    public MyThread(Object lock) {
        this.lock = lock;
    }
    public void run() {
        ThreadDomain td = new ThreadDomain();
        td.testMethod(lock);
    }
}
```

main函数调用下

```java
public static void main(String[] args) {
        Object lock = new Object();
        MyThread t1 = new MyThread(lock);
        MyThread t2 = new MyThread(lock);
        t1.start();
        t2.start();
    }
```

![](3.png) 

如果wait()方法不释放锁，那么Thread-1根本不会进入同步代码块打印的，所以，证明完毕。 



## notify()不释放锁

```java
public class ThreadDomain {
    public void testMethod(Object lock) {
        try {
            synchronized (lock) {
                System.out.println("开始wait, ThreadName = " + Thread.currentThread().getName());
                lock.wait();
                System.out.println("结束wait, ThreadName = " + Thread.currentThread().getName());
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public void synNotifyMethod(Object lock) {
        try {
            synchronized (lock) {
                System.out.println("开始notify,并睡五秒, ThreadName = " + Thread.currentThread().getName());
                lock.notify();
                Thread.sleep(5000);
                System.out.println("结束notify, ThreadName = " + Thread.currentThread().getName());
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

MyThread1.java

```java
public class MyThread1 extends Thread{
    private Object lock;

    public MyThread1(Object lock) {
        this.lock = lock;
    }

    public void run() {
        ThreadDomain td = new ThreadDomain();
        td.testMethod(lock);
    }
}
```

MyThread2.java

```
public class MyThread2 extends Thread{
    private Object lock;

    public MyThread2(Object lock) {
        this.lock = lock;
    }

    public void run() {
        ThreadDomain td = new ThreadDomain();
        td.synNotifyMethod(lock);
    }
}
```

main

````java
  public static void main(String[] args) {
        Object lock = new Object();
        MyThread1 t1 = new MyThread1(lock);//调的是wait
        t1.start();
        MyThread2 t2 = new MyThread2(lock);//调的是notify
        t2.start();
        MyThread2 t2_ = new MyThread2(lock);//调的是notify,用2个线程来notify来证明，第一个notify是否释放了锁,如果释放了，第二个notify可以立即执行.否则就证明了notify就不会释放锁的
        t2_.start();
    }
````



结果:

```
开始wait, ThreadName = Thread-0
开始notify,并睡五秒, ThreadName = Thread-1
......这里阻塞了5秒......
结束notify, ThreadName = Thread-1
结束wait, ThreadName = Thread-0
开始notify,并睡五秒, ThreadName = Thread-2
结束notify, ThreadName = Thread-2
```



如果notify()方法释放锁，那么在Thread-1调用notify()方法后Thread.sleep(5000)必定应该有其他线程可以进入同步代码块了，但是实际上没有，必须等到Thread-1把代码执行完。所以，证明完毕。





## wait/sleep的区别 

Thread有一个sleep()静态方法，它也能使线程暂停一段时间。sleep与wait的不同点是：**sleep并不释放锁**，并且sleep的暂停和wait暂停是不一样的。obj.wait会使线程进入obj对象的等待集合中并等待唤醒。 

但是**wait()和sleep()都可以通过interrupt()方法打断线程的暂停状态**，从而使线程立刻抛出InterruptedException。 

如果线程A希望立即结束线程B，则可以对线程B对应的Thread实例调用interrupt方法。如果此刻线程B正在wait/sleep/join，则线程B会立刻抛出InterruptedException，在catch() {} 中直接return即可安全地结束线程。 

需要注意的是，InterruptedException是线程自己从内部抛出的，并不是interrupt()方法抛出的。对某一线程调用interrupt()时，如果该线程正在执行普通的代码，那么该线程根本就不会抛出InterruptedException。但是，一旦该线程进入到wait()/sleep 

sleep是线程类（Thread）的方法，导致此线程暂停执行指定时间，**给执行机会给其他线程，但是监控状态依然保持，到时后会自动恢复 。调用sleep不会释放对象锁**。  

wait是Object类的方法，对此对象调用wait方法导致本线程放弃对象锁，**进入等待此对象的等待锁定池，只有针对此对象发出notify方 法（或notifyAll）后本线程才进入对象锁定池准备获得对象锁进入运行状态**。







