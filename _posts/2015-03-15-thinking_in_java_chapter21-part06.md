---
layout: post
title: Java编程思想 - 第二十一章、并发（六）
categories: Java
tags: Java编程思想
---

###21.7 新类库中的构件

Java SE5的 **`java.util.concurrent`** 包中引入了大量设计用来解决并发问题的新类。学习这些“工具类”就可以专注于自己想要实现的功能而不是线程的同步、死锁等一堆令人头疼的细节。这一小节内容非常多，建议的学习方法是：

1. 首先看目录，了解这一小节主要讲的是哪几种构件
2. 通过构件的名字猜猜它们想实现的功能，然后通过查询文档总结一下每个构件的特点，适用的场景
3. 尝试着去寻找项目中涉及到的点，然后具体学习这个构件的知识，之后用新构件重新实现这一块作为巩固

嗯，上面总结了一下学习这个小节的步骤（其实是因为太多了。。。。。我不想全看 T_T），那么我们就把目录摘出来看看吧。

###一、前言

下面是21.7小节的目录。嗯，发现一共是7个构件，现在从**文档**出发，逐个浏览一下（Mac 下有 Dash 这样的神器真是幸福啊，hiahiahiahia~）

* 21.7 新类库中的构件
    * 21.7.1 CountDownLatch
    * 21.7.2 CyclicBarrier
    * 21.7.3 DelayQueue
    * 21.7.4 PriorityBlockingQueue
    * 21.7.5 使用 ScheduledExecutor 的温室控制器
    * 21.7.6 Semaphore
    * 21.7.7 Exchanger

下面我们先简单的“望文生义”一下，然后再逐个击破：）

* CountDownLatch：名字直译为——倒计时锁。官方文档的描述是 A synchronization aid that allows one or more threads to wait until a set of operations being performed in other threads completes.[一个线程同步辅助工具，可以让一个或多个线程等待直到其它线程的任务全部完成才会被唤醒。]
* CyclicBarrier：和上面那个功能相似，只是上面的倒计时数值不能被重置，只能递减到0停止；而 CyclicBarrier 可以在倒计时数减为0之后重用（还是原来的值）
* DelayQueue：无界的 BlockingQueue（前面生产者-消费者讲过哦），用于放置实现了 Delayed interface 的对象，其中的对象只能在到期时才能在队列中取走。**这种队列是有序的，即队头对象的延期到期的时间最长。**
* PriorityBlockingQueue：优先队列的 BlockingQueue，具有可阻塞的读取操作。其实就是 BlockingQueue 的优先队列实现
* 使用 ScheduledExecutor 的温室控制器：
* Semaphore：正常的锁（concurrent.Lock 或者 synchronized）在任何时刻都只能允许一个任务访问资源，而 **Semaphore （计数信号量）允许 N 个任务同时访问这个资源。**（是不是有池子的感觉嘞？？）
* Exchanger：两个任务之间交换对象的栅栏。意思是各自拥有对象，离开栅栏时，就拥有对方持有的对象了。典型就是一个任务生产对象，一个任务消费对象。（值得思考，为啥要交换？我直接用一个容器或者 BlockingQueue 完全可以解耦啊，这个到底用在哪里？）

###二、代码来了

下面给每个构件都写个小例子，然后总结一下它们产生的原因和最佳使用场景。go go go!!

####1. CountDownLatch

文档也太详细了吧：

> A synchronization aid that allows one or more threads to wait until a set of operations being performed in other threads completes.
> 
> A CountDownLatch is initialized with a given count. The `await` methods block until the current count reaches zero due to invocations of the `countDown()` method, after which all waiting threads are released and any subsequent invocations of `await` return immediately. **This is a one-shot phenomenon -- the count cannot be reset. If you need a version that resets the count, consider using a CyclicBarrier【和 CyclicBarrier 的区别】**.
> 
> A CountDownLatch is a versatile(多功能的) synchronization tool and can be used for a number of purposes. A CountDownLatch initialized with a count of one serves as a simple on/off latch, or gate: all threads invoking `await` wait at the gate until it is opened by a thread invoking `countDown()`. A CountDownLatch initialized to N can be used to make one thread wait until N threads have completed some action, or some action has been completed N times.【这里是使用场景：count=1为开关；count=N 重复 N 次】
> 
> A useful property of a CountDownLatch is that it doesn't require that threads calling countDown wait for the count to reach zero before proceeding, it simply prevents any thread from proceeding past an `await` until all threads could pass.

同时文档提供了演示代码：

{% highlight java linenos %}
Here is a pair of classes in which a group of worker threads use two countdown latches:

1. The first is a start signal that prevents any worker from proceeding until the driver is ready for them to proceed;
2. The second is a completion signal that allows the driver to wait until all workers have completed.
 
class Driver { // ...
   void main() throws InterruptedException {
     CountDownLatch startSignal = new CountDownLatch(1);
     CountDownLatch doneSignal = new CountDownLatch(N);

     for (int i = 0; i < N; ++i) // create and start threads
       new Thread(new Worker(startSignal, doneSignal)).start();

     doSomethingElse();            // don't let run yet
     startSignal.countDown();      // let all threads proceed
     doSomethingElse();
     doneSignal.await();           // wait for all to finish
   }
 }

 class Worker implements Runnable {
   private final CountDownLatch startSignal;
   private final CountDownLatch doneSignal;
   Worker(CountDownLatch startSignal, CountDownLatch doneSignal) {
      this.startSignal = startSignal;
      this.doneSignal = doneSignal;
   }
   public void run() {
      try {
        startSignal.await();
        doWork();
        doneSignal.countDown();
      } catch (InterruptedException ex) {} // return;
   }

   void doWork() { ... }
 }
{% endhighlight java %}

文档已经够清晰了，这里就不多废话了。

####2. CyclicBarrier

直译为循环栅栏，通过它可以让**一组线程全部到达某个状态后再同时执行，也就是说假如有5个线程协作完成一个任务，那么只有当每个线程都完成了各自的任务（都到达终点），才能继续运行（开始领奖）**。循环的意思是当所有等待线程都被释放（也就是所有线程完成各自的任务，整个程序开始继续执行）以后，CyclicBarrier 可以被重用。而上面的 CountDownLatch 只能用一次。

这个的文档也非常详细：

> A synchronization aid that allows a set of threads to all wait for each other to reach a common barrier point[各个线程互相等待，CountDownLatch 是一个等待其他多个线程]. CyclicBarriers are useful in programs involving a fixed sized party of threads that must occasionally wait for each other[使用场景：固定数目的线程互相等待]. The barrier is called cyclic because it can be re-used after the waiting threads are released[为什么叫做循环是因为释放后可以重用].
> 
> A CyclicBarrier supports an optional Runnable command that is run once per barrier point, after the last thread in the party arrives, but before any threads are released[一个可选的参数是 Runnable，在所有线程到达 Barrier 后，但是在释放（让他们继续运行）之前。]. This barrier action is useful for updating shared-state before any of the parties continue[为什么要有这个 Runnable ？可以在所有线程继续运行前更新共享状态等].
> 
> If the barrier action does not rely on the parties being suspended when it is executed, then any of the threads in the party could execute that action when it is released. To facilitate this, each invocation of await() returns the arrival index of that thread at the barrier. You can then choose which thread should execute the barrier action, for example:

		if (barrier.await() == 0) { 
			// log the completion of this iteration
		}
	

> The CyclicBarrier uses an all-or-none breakage model for failed synchronization attempts: If a thread leaves a barrier point prematurely(提前) because of interruption, failure, or timeout, all other threads waiting at that barrier point will also leave abnormally via BrokenBarrierException (or InterruptedException if they too were interrupted at about the same time).[要么全体成功，要么全体失败。和 Future 类似]
> 
> Memory consistency effects: Actions in a thread prior to calling await() happen-before actions that are part of the barrier action, which in turn happen-before actions following a successful return from the corresponding await() in other threads.

举一个例子，现在有一个比赛：

> 每组 5 个人，要把 A 处的球运输到10米外的 B 处，每个人可以拿2个。等10个球全部被运输到 B 处后，这个组就算是完成了任务。那么，只要5个人有一个没有完成，就不能算是完成任务。符合上面 CyclicBarrier 的应用场景。下面是这个例子的代码：

{% highlight java linenos %}
package concurrency;

import java.util.Random;
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

class Player implements Runnable {

    private CyclicBarrier cyclicBarrier;
    private Random rand = new Random(47);

    public Player(CyclicBarrier cyclicBarrier) {
        this.cyclicBarrier = cyclicBarrier;
    }

    @Override
    public void run() {
        try {
            // 运输2个球
            TimeUnit.SECONDS.sleep(rand.nextInt(5));

            // 等待其他队友完成
            System.out.println(Thread.currentThread() + " 完成任务！等待队友 ing...");
            cyclicBarrier.await();
        } catch (BrokenBarrierException e) {
            System.out.println("BrokenBarrierException " + e);
        } catch (InterruptedException e) {
            System.out.println("InterruptedException " + e);
        }
    }
}

public class CyclicBarrierGame {
    public static void main(String[] args) {
        //定义 CyclicBarrier
        CyclicBarrier cyclicBarrier = new CyclicBarrier(5, new Runnable() {
            @Override
            public void run() {
                //全部完成
                System.out.println("\n全部完成！ 举手报告 ing...\n");
                System.out.println("谁来执行 Barrier 的 Runnable？(猜测是最后一个) : " + Thread.currentThread());
            }
        });
        
        ExecutorService exec = Executors.newCachedThreadPool();
        for(int i = 0; i < 5; i++) {
            exec.execute(new Player(cyclicBarrier));
        }

        /* 下面是为了验证 CyclicBarrier 能够重用
         try {
            TimeUnit.SECONDS.sleep(5);
            for(int i = 0; i < 5; i++) {
            	exec.execute(new Player(cyclicBarrier));
        	}
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
        */
        
        exec.shutdown();
    }
}/*output:
Thread[pool-1-thread-3,5,main] 完成任务！等待队友 ing...
Thread[pool-1-thread-2,5,main] 完成任务！等待队友 ing...
Thread[pool-1-thread-5,5,main] 完成任务！等待队友 ing...
Thread[pool-1-thread-1,5,main] 完成任务！等待队友 ing...
Thread[pool-1-thread-4,5,main] 完成任务！等待队友 ing...

全部完成！ 举手报告 ing...

谁来执行 Barrier 的 Runnable？(猜测是最后一个) : Thread[pool-1-thread-4,5,main]
*/
{% endhighlight java %}

####3. DelayQueue

DelayQueue 就是一个无界队列，是用 PriorityQueue 实现的 BlockingQueue，如果要使用 DelayQueue，其中的元素必须实现 Delayed 接口，Delayed 接口有2个方法需要重写：compareTo()和 getDelay()方法。因为使用的是优先队列，所以需要确定元素之间的优先级，那么重写 compareTo()就很明显了，又为了满足 DelayQueue 的特性（每次队头是延期到期时间最长的元素），那么就需要知道元素的到期时间，而这个时间就是通过 getDelay()获取的。

* 延迟到期时间最长：这个刚看的时候还挺迷糊的，现在知道了。就是到期之后保存时间最长的元素。比如2个元素，在10:00:00这个时间点都到期了，但是 A 元素到期后保存时间为2分钟，B 元素到期后保存时间为1分钟，那么优先级最高的肯定是 A 元素了（本质来说，这个 order 是通过小顶堆维护的，所以获取延迟到期时间最长元素的时间复杂度为 O(lgN)）。

写了一个例子，但是因为输出有点问题，就看了一下 DelayQueue 的源码，发现里面的实现是委托给 PriorityQueue 的，于是写了篇文章跟了下 PriorityQueue 的基本操作（ 也是 DelayQueue 的基本操作），结合文档和源码和我给的例子，应该就非常 easy 了：[PriorityQueue 源码剖析](../priorityqueue)

####4. PriorityBlockingQueue

哈哈，前面刚看完 PriorityQueue 的源码，这里就遇到了 PriorityBlockingQueue，其实 PriorityBlockingQueue就是用 PriorityQueue 实现的 BlockingQueue，所以没啥可说的。写了个例子低空掠过：

{% highlight java linenos %}
package concurrency;

import java.util.Random;
import java.util.concurrent.PriorityBlockingQueue;

class Leader implements Comparable {
    private String name;
    private int degree;

    public Leader(String name, int degree) {
        this.name = name;
        this.degree = degree;
    }

    @Override
    public int compareTo(Object o) {
        Leader leader = (Leader) o;
        return leader.degree - this.degree;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getDegree() {
        return degree;
    }

    public void setDegree(int degree) {
        this.degree = degree;
    }

}

public class WhoGoFirst {

    // 通过随机数给领导分级别
    private static PriorityBlockingQueue<Leader> leaders = new PriorityBlockingQueue<Leader>();

    public static void watchFilm(Leader leader) {
        leaders.add(leader);
    }

    public static void goFirst(PriorityBlockingQueue<Leader> leaders) {
        try {
            while (!leaders.isEmpty()) {
                Leader leader = leaders.take();
                System.out.println("级别： " + leader.getDegree() + "的 " + leader.getName() + " 正在撤离...");
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        Random random = new Random();
        for (int i = 1; i <= 10; i++) {
            watchFilm(new Leader("leader " + i, random.nextInt(10)));
        }

        System.out.println("所有领导已经就坐，开始播放电影：速度与激情7...");

        System.out.println("着火了！！！");

        goFirst(leaders);

    }
}/*output:
所有领导已经就坐，开始播放电影：速度与激情7...
着火了！！！
级别： 8的 leader 3 正在撤离...
级别： 7的 leader 8 正在撤离...
级别： 6的 leader 4 正在撤离...
级别： 6的 leader 9 正在撤离...
级别： 6的 leader 2 正在撤离...
级别： 5的 leader 5 正在撤离...
级别： 4的 leader 6 正在撤离...
级别： 4的 leader 7 正在撤离...
级别： 2的 leader 10 正在撤离...
级别： 0的 leader 1 正在撤离...
*/
{% endhighlight java %}

####5. ScheduledExcutor

这个小节讲的是定时触发任务，知道 crontab 的应该都不陌生。看完以后我 google 了一下，发现几个类似功能的类，先知道有这几个东西，用到了再具体看文档吧。

* Timer：单线程轮询任务列表，效率较低
* ScheduledExcutor：并行执行
* JCrontab：借鉴了 crontab 的语法，其区别在于 command 不再是 unix/linux 的命令，而是一个 Java 类。如果该类带参数，例如`com.ibm.scheduler.JCronTask2#run`，则定期执行 run 方法；如果该类不带参数，则默认执行 main 方法。此外，还可以传参数给 main 方法或者构造函数，例如`com.ibm.scheduler.JCronTask2#run Hello World`表示传两个参数 Hello 和 World 给构造函数
* Quartz：Spring 就是用的这个执行定时任务的

给个随便搜到的资料：[几种任务调度的 Java 实现方法与比较](https://www.ibm.com/developerworks/cn/java/j-lo-taskschedule/)

对于举例子的 ScheduledThreadPoolExecutor，大概看下源码，本质是使用 DelayWorkQueue 实现的 BlockingQueue。其中 DelayWorkQueue 和 DelayQueue 类似，不过没有复用 DelayQueue 中用到的 PriorityQueue，而是自己捯饬了一个新的小（大）顶堆。看来concurrent 也不是100%完美的代码呀，哈哈哈。

{% highlight java linenos %}
public class ScheduledThreadPoolExecutorTest {
    public static void main(String[] args) {
        ScheduledExecutorService executorService = new ScheduledThreadPoolExecutor(1);
        BusinessTask task = new BusinessTask();
        //1秒后开始执行任务，以后每隔2秒执行一次
        executorService.scheduleWithFixedDelay(task, 1000, 2000,TimeUnit.MILLISECONDS);
    }

    private static class BusinessTask implements Runnable{
        @Override
        public void run() { 
            System.out.println("任务开始...");
            // doBusiness();
            System.out.println("任务结束...");
        }
    }
}
{% endhighlight java %}

嗯，这个例子虽然简单，但是我想说几点：

1. ScheduleAtFixedRate 是基于固定时间间隔进行任务调度，ScheduleWithFixedDelay 取决于每次任务执行的时间长短，是基于不固定时间间隔进行任务调度：
	1. scheduleWithFixedDelay()方法：每次执行时间为上一次任务结束起向后推一个时间间隔，即每次执行时间为：initialDelay, initialDelay+executeTime+delay, initialDelay+2*executeTime+2*delay
	2. scheduleWithFixedRate()方法：每次执行时间为上一次任务开始起向后推一个时间间隔，即每次执行时间为 :initialDelay, initialDelay+period, initialDelay+2*period, …
2. 有可能上面的程序执行了一段时间后，会发现不再执行了，去查看日志，可能是doBusiness()方法中抛出了异常。但是为什么doBusiness()抛出异常就会中止定时任务的执行呢？看文档就知道了：
	> Creates and executes a periodic action that becomes enabled first after the given initial delay, and subsequently with the given delay between the termination of one execution and the commencement of the next. If any execution of the task encounters an exception, subsequent executions are suppressed. Otherwise, the task will only terminate via cancellation or termination of the executor.
	>
	> 简单翻译就是：
	>
	> 创建并执行一个在给定初始延迟后首次启用的定期操作，随后，在每一次执行终止和下一次执行开始之间都存在给定的延迟。如果任务的任一执行遇到异常，就会取消后续执行。否则，只能通过执行程序的取消或终止方法来终止该任务。

所以上面的例子应该改成下面这样：

{% highlight java linenos %}
public class ScheduledThreadPoolExecutorTest {
    public static void main(String[] args) {
        ScheduledExecutorService executorService = new ScheduledThreadPoolExecutor(1);
        BusinessTask task = new BusinessTask();
        //1秒后开始执行任务，以后每隔2秒执行一次
        executorService.scheduleWithFixedDelay(task, 1000, 2000,TimeUnit.MILLISECONDS);
    }

    private static class BusinessTask implements Runnable{
        @Override
        public void run() { 
            //捕获所有的异常，保证定时任务能够继续执行
            try{
                System.out.println("任务开始...");
                // doBusiness();
                System.out.println("任务结束...");
            }catch (Throwable e) {
                // solve the exception problem
            }
        }
    }
}
{% endhighlight java %}

####6. Semaphore

Semaphore 是一个计数信号量，平常的锁（来自 concurrent.locks 或者内建的 synchronized)再任何时刻都只能允许一个任务访问一项资源，但是 Semaphore 允许 N 个任务同时访问这个资源。你还可以将信号量看做是在向外分发使用资源的“许可证”，尽管实际上没有使用任何许可证对象。

总结来说，一般的锁是保证一个资源只能被一个任务访问；Semaphore 是保证一堆资源可以同时有多个任务访问。举个例子，现在有一个厕所，5个坑位，如果使用 synchronized 的话，同步厕所就只能让1个人进入，浪费了4个坑位；稍微往前一步是使用 BlockingQueue（如果你用 synchronized 来同步5个坑位就很复杂多了），再往前一步，concurrent 提供了 Semaphore ，它通过 acquire()和 release()来保证资源的分发使用。

下面我们通过实现一个资源池来说明，具体的场景完全在例子中注释：）

{% highlight java linenos %}
package concurrency;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

// 抽象成一个资源池
class Pool<T> {
    private int size;
    private List<T> items = new ArrayList<T>();
    private volatile boolean[] checkedOut;
    private Semaphore available;

    // 要放入资源池的资源数目，如果请求线程数目大于资源池资源数目，就需要阻塞等待
    public Pool(Class<T> classObject, int size) {
        this.size = size;
        checkedOut = new boolean[size];
        /**
         * 第二个参数的含义:<br>
         * 
         * true: 代表的是公平竞争<br>
         * 没有第二个参数或者false：代表随机选中等待许可证的线程
         */
        available = new Semaphore(size, true);
        // Load pool with objects that can be checked out:
        for (int i = 0; i < size; i++) {
            try {
                // Assumes a default constructor:
                items.add(classObject.newInstance());
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }
    }

    public T checkOut() throws InterruptedException {
        available.acquire();
        return getItem();
    }

    public void checkIn(T x) {
        if (releaseItem(x))
            available.release();
    }

    private synchronized T getItem() {
        for (int i = 0; i < size; i++)
            if (!checkedOut[i]) {
                checkedOut[i] = true;
                return items.get(i);
            }
        return null;
    }

    // 回收资源
    private synchronized boolean releaseItem(T item) {
        int index = items.indexOf(item);
        // not in the list
        if (index == -1)
            return false;
        if (checkedOut[index]) {
            checkedOut[index] = false;
            return true;
        }
        return false; // wasn't checked out
    }

}

class Fat {
    private volatile double d; // 阻止指令优化
    private static int counter = 0;
    private final int id = counter++;

    public Fat() {
        // Expensive, interruptible operation
        for (int i = 1; i < 10000; i++) {
            d += (Math.PI + Math.E) / (double) i;
        }
    }

    public void operation() {
        System.out.println(this);
    }

    public String toString() {
        return "Fat id: " + id;
    }

}

class CheckoutTask<T> implements Runnable {
    private static int counter = 0;
    private final int id = counter++;
    private Pool<T> pool;

    public CheckoutTask(Pool<T> pool) {
        this.pool = pool;
    }

    @Override
    public void run() {
        try {
            T item = pool.checkOut();
            System.out.println(this + "checked out " + item);
            TimeUnit.SECONDS.sleep(1);
            System.out.println(this + "checked in " + item);
            pool.checkIn(item);
        } catch (InterruptedException e) {
            // 终止
        }
    }

    @Override
    public String toString() {
        return "checkoutTask " + id + " ";
    }
}

public class SemaphoreDemo {
    final static int SIZE = 25;

    public static void main(String[] args) throws Exception {
        // 创建一个 Fat 的资源池，大小为25
        final Pool<Fat> pool = new Pool<Fat>(Fat.class, SIZE);

        // 启动25个线程开始玩，从25个资源池 checkout
        ExecutorService exec = Executors.newCachedThreadPool();
        for (int i = 0; i < SIZE; i++) {
            exec.execute(new CheckoutTask<Fat>(pool));
        }
        System.out.println("All CheckoutTasks created");

        // 然后用主线程把所有的 Fat 全灌到 list 中了，资源池为空
        List<Fat> list = new ArrayList<Fat>();
        for (int i = 0; i < SIZE; i++) {
            Fat fat = pool.checkOut();
            System.out.println(i + ": main() thread checked out ");
            fat.operation();
            list.add(fat);
        }

        // 还记得 Future 会阻塞吗？因为主线程把25个资源全 checkout 了，所以再 checkOut 肯定阻塞了
        Future<?> blocked = exec.submit(new Runnable() {
            @Override
            public void run() {
                try {
                    // Semaphore prevents additional checkout,
                    // so call is blocked:
                    pool.checkOut();
                } catch (InterruptedException e) {
                    System.out.println("checkOut() Interrupted");
                }
            }
        });
        // 因为 Semaphore 是阻塞的，所以10s 还拿不到的情况下，就取消 blocked 线程的工作
        TimeUnit.SECONDS.sleep(10);
        blocked.cancel(true); // Break out of blocked call
        System.out.println("\n\n\nChecking in objects in " + list);
        for (Fat f : list) {
            pool.checkIn(f);
        }
        for (Fat f : list) {
            pool.checkIn(f); // Second checkIn ignored
        }
        exec.shutdown();
    }

}/*output:
checkoutTask 1 checked out Fat id: 1
checkoutTask 4 checked out Fat id: 4
checkoutTask 3 checked out Fat id: 3
checkoutTask 5 checked out Fat id: 5
checkoutTask 2 checked out Fat id: 2
checkoutTask 6 checked out Fat id: 6
checkoutTask 0 checked out Fat id: 0
checkoutTask 7 checked out Fat id: 7
checkoutTask 8 checked out Fat id: 8
checkoutTask 9 checked out Fat id: 9
checkoutTask 10 checked out Fat id: 10
checkoutTask 11 checked out Fat id: 11
checkoutTask 12 checked out Fat id: 12
checkoutTask 13 checked out Fat id: 13
checkoutTask 14 checked out Fat id: 14
checkoutTask 15 checked out Fat id: 15
checkoutTask 16 checked out Fat id: 16
checkoutTask 17 checked out Fat id: 17
checkoutTask 18 checked out Fat id: 18
checkoutTask 19 checked out Fat id: 19
checkoutTask 20 checked out Fat id: 20
checkoutTask 21 checked out Fat id: 21
checkoutTask 22 checked out Fat id: 22
checkoutTask 23 checked out Fat id: 23
All CheckoutTasks created
checkoutTask 24 checked out Fat id: 24
checkoutTask 4 checked in Fat id: 4
checkoutTask 2 checked in Fat id: 2
checkoutTask 0 checked in Fat id: 0
checkoutTask 3 checked in Fat id: 3
checkoutTask 7 checked in Fat id: 7
checkoutTask 9 checked in Fat id: 9
checkoutTask 5 checked in Fat id: 5
checkoutTask 11 checked in Fat id: 11
checkoutTask 12 checked in Fat id: 12
checkoutTask 13 checked in Fat id: 13
checkoutTask 14 checked in Fat id: 14
checkoutTask 15 checked in Fat id: 15
checkoutTask 1 checked in Fat id: 1
checkoutTask 17 checked in Fat id: 17
checkoutTask 16 checked in Fat id: 16
checkoutTask 10 checked in Fat id: 10
checkoutTask 8 checked in Fat id: 8
0: main() thread checked out 
Fat id: 0
1: main() thread checked out 
Fat id: 1
2: main() thread checked out 
Fat id: 2
3: main() thread checked out 
Fat id: 3
4: main() thread checked out 
Fat id: 4
5: main() thread checked out 
Fat id: 5
6: main() thread checked out 
Fat id: 7
7: main() thread checked out 
Fat id: 8
8: main() thread checked out 
Fat id: 9
9: main() thread checked out 
Fat id: 10
10: main() thread checked out 
Fat id: 11
11: main() thread checked out 
Fat id: 12
12: main() thread checked out 
Fat id: 13
13: main() thread checked out 
Fat id: 14
14: main() thread checked out 
Fat id: 15
15: main() thread checked out 
Fat id: 16
16: main() thread checked out 
Fat id: 17
checkoutTask 6 checked in Fat id: 6
checkoutTask 18 checked in Fat id: 18
checkoutTask 20 checked in Fat id: 20
checkoutTask 21 checked in Fat id: 21
checkoutTask 22 checked in Fat id: 22
checkoutTask 23 checked in Fat id: 23
checkoutTask 24 checked in Fat id: 24
17: main() thread checked out 
Fat id: 6
18: main() thread checked out 
Fat id: 18
19: main() thread checked out 
Fat id: 20
20: main() thread checked out 
Fat id: 21
checkoutTask 19 checked in Fat id: 19
21: main() thread checked out 
Fat id: 22
22: main() thread checked out 
Fat id: 19
23: main() thread checked out 
Fat id: 23
24: main() thread checked out 
Fat id: 24
checkOut() Interrupted



Checking in objects in [Fat id: 0, Fat id: 1, Fat id: 2, Fat id: 3, Fat id: 4, Fat id: 5, Fat id: 7, Fat id: 8, Fat id: 9, Fat id: 10, Fat id: 11, Fat id: 12, Fat id: 13, Fat id: 14, Fat id: 15, Fat i
d: 16, Fat id: 17, Fat id: 6, Fat id: 18, Fat id: 20, Fat id: 21, Fat id: 22, Fat id: 19, Fat id: 23, Fat id: 24]
*/
{% endhighlight java %}

####7. Exchanger

终于来到21.7小节的最后一个构件了！！！！

这个构件很简单，是为了让**两个任务交换对象，当两个任务进入 Exchanger 提供的“栅栏”时，他们各自拥有一个对象，当它们离开时，都拥有了之前由对方拥有的对象**。为什么要有这么个东西呢？考虑下面的场景：

> 一个任务在创建对象，这些对象的生产/销毁代价都非常高。上面 Semaphore 的例子还算靠谱，因为我用完了资源并没有销毁，直接还给资源池了，然后立马可以被复用。但是如果两个线程需要知晓对方的工作状态信息，就可以用 Exchanger 交换各自的工作状态。

更具体的使用场景还没有仔细想，大概搜了下都是缓存交换（一个读，一个写），比如这个：[java.util.concurrent.Exchanger应用范例与原理浅析](http://lixuanbin.iteye.com/blog/2166772)。我只是简单写了个 demo 备忘，用到的时候知道有这个东西，其他的再看文档吧：

{% highlight java linenos %}
package concurrency;

import java.util.ArrayList;
import java.util.List;
import java.util.Random;
import java.util.concurrent.Exchanger;

public class ExchangerTest {

    public static void main(String[] args) {
        Exchanger<List<Integer>> exchanger = new Exchanger<>();
        new Consumer(exchanger).start();
        new Producer(exchanger).start();
    }

}

class Producer extends Thread {
    List<Integer> list = new ArrayList<>();
    Exchanger<List<Integer>> exchanger = null;

    public Producer(Exchanger<List<Integer>> exchanger) {
        super();
        this.exchanger = exchanger;
    }

    @Override
    public void run() {
        System.out.println("this is Producer");
        Random rand = new Random();
        for (int i = 0; i < 1; i++) {
            list.clear();
            list.add(rand.nextInt(10000));
            list.add(rand.nextInt(10000));
            list.add(rand.nextInt(10000));
            list.add(rand.nextInt(10000));
            list.add(rand.nextInt(10000));
            try {
                System.out.println("producer exchanger...");
                list = exchanger.exchange(list);
                System.out.println("Producer is done");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

class Consumer extends Thread {
    List<Integer> list = new ArrayList<>();
    Exchanger<List<Integer>> exchanger = null;

    public Consumer(Exchanger<List<Integer>> exchanger) {
        super();
        this.exchanger = exchanger;
    }

    @Override
    public void run() {
        System.out.println("this is Consumer");
        for (int i = 0; i < 1; i++) {
            try {
                System.out.println("consumer blocking...");
                list = exchanger.exchange(list);
                System.out.println("consumer is done");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.print(list.get(0) + ", ");
            System.out.print(list.get(1) + ", ");
            System.out.print(list.get(2) + ", ");
            System.out.print(list.get(3) + ", ");
            System.out.println(list.get(4) + ", ");
        }
    }
}/*output:
this is Consumer
consumer blocking...
this is Producer
producer exchanger...
Producer is done
consumer is done
7481, 9360, 6010, 4630, 4338, 
*/
{% endhighlight java %}



