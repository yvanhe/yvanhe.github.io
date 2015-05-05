---
layout: post
title: 如何判断 Java 线程并发的安全性
categories: Java
tags: 线程安全
---

###前言

在高并发的时代中，如何写出高质量的并发程序一直是一个令人头疼的事情。现在给你一段代码，你如何判断它是否是线程安全的？又如何改进呢？在这里，我们简单介绍一下Java内部是如果保证线程安全的。一切的关键就在：

> 高效利用并发，同时也必须保证JMM三大特性的有序性。只有保证了有序性，才能在代码正确执行的前提下去求追更高的效率。而这个王牌就是：先行发生原则

想一想，如果Java中所有的同步操作都交给synchronized或者volatile来做，那将是很麻烦的。考虑你要写一个并发环境下的程序，你得花费大半的精力来实现并发的线程安全性。而这对于新手来说，几乎是不可能完成的任务。所以Java定义了一些规则，使得Java会自动判断数据是否存在竞争，线程是否安全等。通过这个先行并发原则，Java可以一揽子解决**并发环境下两个操作之间是否可能存在冲突的所有问题。**那么，什么是先行发生呢？

###一、先行发生原则

> 离散数学中曾经定义了“偏序”的概念，在Java中正是使用了这个概念。偏序通俗的来理解就是拓扑结构，一个DAG。如果操作A先行发生于B，那么B将能观察到A所做的所有事情（包括修改变量、发送消息、调用方法等）。这个可以用例子来说明：

{% highlight java linenos %}
//线程A
i = 1;

//线程B
j = i;

//线程C
i = 2;
{% endhighlight java %}

如果只考虑A和B，并且保证A先行发生于B，那么B的值是多少？答案很显然是1。依据有两个：

1. 根据先行发生的偏序关系，i的值的改变可以被j观察到
2. 在线程C修改i值之前，线程A结束之后没有其他线程会修改i。

但是如果考虑C线程，很悲桑，j的值会不确定。**因为线程B和线程C没有先行发生定义的偏序，意思就是C可以发生在B前，B后的任意位置，如果C发生在A/B之间，那么显然j的值就是2了。**所以线程B就不具备多线程安全性。为了解决这种问题，JMM定义了一些”天然的“先行发生关系来保证多线程的安全。假如所有的操作都像A/B定义好了偏序关系，那么并发就不会有任何难度了。但是为了更灵活的使用，Java只对一些场景定了先行发生原则，所以遇到这几个规则范围内的问题，我们就不需要考虑并发的各种问题，Java会自动帮我们解决。所以，**以下操作无须任何同步手段就能保证并发的成功**：

* 程序次序规则：一个线程内，代码的执行会按照程序书写的顺序
* 管程锁定原则：对同一变量的unlock操作先行发生于后来的lock操作
* volatile变量规则：对一个volatile的写操作先行发生于后来的读操作
* 线程启动原则：Thread的start()先行发生于线程内的所有动作
* 线程终止原则：线程内的所有动作都先行发生于线程的终止检测
* 线程中断原则：对线程调用interrupt()先行发生于被中断的代码检测到是否有中断发生
* 对象终结原则：一个对象的初始化操作先行发生于finalize()方法
* 传递性：A先行发生于B，B先行发生于C，那么A先行发生于C


###二、先行发生原则的应用

上面我们介绍了先行发生原则，下面我们就用实例来看看，它是如何工作的。考虑这样一段代码：

{% highlight java linenos %}
private int value = 0;

public int getValue() {
	return this.value;
}

public void setValue(int value) {
	this.value = value;
}
{% endhighlight java %}

上面是一段再普通不过的getter/setter方法了，现在假设有A/B两个线程。线程A先（时间上的先后）调用了`setValue(2)`，然后线程B调用了同一个对象的`getValue()`，那么线程B收到的返回值是多少呢？

那么，我们就用前面介绍的先行发生原则来判断一下：

* 因为A/B不是一个线程，所以无法使用程序次序原则
* 因为没有synchronized，所以不存在lock/unlock操作，无法使用管程锁定原则
* 没有volatile修饰，不能使用volatile变量原则
* 因为是两个独立的线程，所以线程启动、终止、终端原则都不能使用
* 因为不涉及对象的初始化和finalize()，所以无法使用对象终结原则
* 因为根本就没有一个先行发生原则，所以也不能使用传递性

综上所述，我们发生A/B之间不满足所有的先行发生原则。所以A/B线程的操作不是线程安全的。如果想要线程安全，必须通过程序员自己去实现。这里提供2个方法：

1. 将getter/setter方法添加synchronized修饰，使之满足管程锁定原则
2. 把value定义为volatile，因为setter中对value的修改不依赖value的原值，所以符合volatile的使用场景（一定要符合前提，如果这里方法是value++就肯定不行了），然后套用volatile变量原则就可以保证了

通过上面的例子，我们可以得出一个结论：

> 时间上先发生的操作是无法保证“先行发生”的。那如果一个操作满足”先行发生“的定义，是否就一定是时间上的先发生呢？很遗憾，这个结论也是不成立的。一个典型的例子就是提到过的**指令重排序**：

{% highlight java linenos %}
//以下操作在同一个线程内执行
int i = 1;
int j = 2;
{% endhighlight java %}

既然是同一个线程，那么肯定满足程序次序原则。所以`int i = 1;`先行发生于`int j = 2;`，但是`int j = 2;`完全可能被处理器先处理，这并不影响先行发生原则的正确性（我猜测是JVM在满足先行发生原则的基础上，会对某些无关语句进行指令重排序优化），从而我们无法在线程中感知。Java有句话是这样说的就是这个道理：

> 线程内观察指令全部是串行的，而在其他线程观察那个线程，会发现内部程序的执行是杂乱无章的。但JVM会保证结果的正确性

综合上面的解释我们就知道了：

> **时间上的先后顺序与先行发生原则之间没有太大的关系，所以我们衡量并发安全问题的时候不能受到时间顺序的干扰，一切必须以先行发生原则为准**。