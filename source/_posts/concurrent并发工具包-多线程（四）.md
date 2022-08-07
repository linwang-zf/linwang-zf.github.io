---
title: concurrent并发工具包 --- 多线程（四）
author: linWang
date: 2022-08-07 19:34:03
tags: 并发工具包下用法
categories: Java并发
---

>   本文介绍了concurrent下常见的一些工具类的用法

<!--more-->

# CountDownLatch(倒计时器)

在多线程协作完成业务功能时，有时候**需要等待其他多个线程完成任务之后，主线程才能继续往下执行业务功能**，在这种的业务场景下，通常可以使用

1.  Thread 类的 join 方法，让主线程等待被 join 的线程执行完之后，主线程才能继续往下执行。
2.  使用线程间消息通信机制也可以完成。
3.  使用Executor体系中的 FutureTask,主线程调用其他线程 get() 方法

其实，java 并发工具类中为我们提供了类似“倒计时”这样的工具类，可以十分方便的完成所说的这种业务场景。

## 方法

![image-20200229224934309](00831rSTgy1gcdnpux2e4j30ug09imyh.jpg)



```java
await() throws InterruptedException //调用该方法的线程等到构造方法传入的 N 减到 0 的时候，才能继续往下执行；
await(long timeout, TimeUnit unit) //与上面的 await 方法功能一致，只不过这里有了时间限制，调用该方法的线程等到指定的 timeout 时间后，不管 N 是否减至为 0，都会继续往下执行；
countDown() //使 CountDownLatch 初始值 N 减 1；
long getCount() //获取当前 CountDownLatch 维护的值；
CountDownLatch() //参数为倒计时数量
```

为了能够理解 CountDownLatch，举一个很通俗的例子，运动员进行跑步比赛时，假设有 6 个运动员参与比赛，裁判员在终点会为这 6 个运动员分别计时，可以想象每当一个运动员到达终点的时候，对于裁判员来说就少了一个计时任务。直到所有运动员都到达终点了，裁判员的任务也才完成。这 6 个运动员可以类比成 6 个线程，当线程调用 CountDownLatch.countDown 方法时就会对计数器的值减一，直到计数器的值为 0 的时候，裁判员（调用 await 方法的线程）才能继续往下执行。

## Demo

```java
package 多线程.并发工具包;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class CountDownLatchDemo {
    private static CountDownLatch startSignal = new CountDownLatch(1);
    //用来表示裁判员需要维护的是6个运动员
    private static CountDownLatch endSignal = new CountDownLatch(6);

    public static void main(String[] args) throws InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(6);
        for (int i = 0; i < 6; i++) {
            executorService.execute(() -> {
                try {
                    System.out.println(Thread.currentThread().getName() + " 运动员等待裁判员响哨！！！");
                    startSignal.await();
                    System.out.println(Thread.currentThread().getName() + "正在全力冲刺");
                    System.out.println(Thread.currentThread().getName() + "  到达终点");
                    endSignal.countDown();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }
        TimeUnit.SECONDS.sleep(3);
        System.out.println("裁判员发号施令啦！！！");
        startSignal.countDown();
        endSignal.await();
        System.out.println("所有运动员到达终点，比赛结束！");
        executorService.shutdown();
    }
}
```

![image-20200229225217749](00831rSTgy1gcdnsp6oh2j30im0mwgpc.jpg)



# CyclicBarrier(循环栅栏)

CyclicBarrier 也是一种多线程并发控制的实用工具，和 CountDownLatch 一样具有等待计数的功能，但是相比于 CountDownLatch 功能更加强大。

循环栅栏，也就是有一个临界点，所有人到了这个临界点，才可以一起继续工作，然后再等下一次所有人都到这个临界点，继续工作…如此循环…

## Demo

```java
package com.alibaba.global.concurrent;

import java.util.concurrent.*;

/**
 * @author :  zhufeng
 * @time :  2022/8/7 12:28
 * @description : 三个线程交替输出ABC，如果需要按照ABC的顺序输出，可以通过Thread.sleep或者CountDownLatch来实现
 **/
public class CyclicBarrierDemo {

    public static void main(String[] args) {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(3, () -> System.out.println("线程A、B、C已经执行完一轮"));

        ExecutorService executorService = Executors.newFixedThreadPool(3);
        executorService.execute(() -> {
            while (true) {
                try {
                    System.out.println("A");
                    cyclicBarrier.await();
                } catch (InterruptedException | BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }
        });
        executorService.execute(() -> {
            while (true) {
                try {
                    System.out.println("B");
                    cyclicBarrier.await();
                } catch (InterruptedException | BrokenBarrierException e) {
                    e.printStackTrace();
                }finally {
                }
            }
        });
        executorService.execute(() -> {
            while (true) {
                try {
                    System.out.println("C");
                    cyclicBarrier.await();
                } catch (InterruptedException | BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }
        });

    }
}
```

# CountDownLatch 与 CyclicBarrier 的比较

CountDownLatch 与 CyclicBarrier 都是用于控制并发的工具类，都可以理解成维护的就是一个计数器，但是这两者还是各有不同侧重点的：

-   CountDownLatch 一般用于**某个线程 A 等待若干个其他线程执行完任务**之后，它才执行，最重要的是有一个主心骨，主线程是核心，他等所有人干完；而 CyclicBarrier 一般用于**一组线程互相等待至某个状态**，然后这一组线程再同时执行；CountDownLatch 强调一个线程等多个线程完成某件事情。CyclicBarrier 是多个线程互等，等大家都完成，再携手共进。所以两者 await 的地方不一样，前者是对主线程进行 await，而后者则是所有一起工作的进行 await。
-   CountDownLatch 方法比较少，操作比较简单，而 CyclicBarrier 提供的方法更多，比如能够通过 getNumberWaiting()，isBroken()这些方法获取当前多个线程的状态，**并且 CyclicBarrier 的构造方法可以传入 barrierAction**，指定当所有线程都到达时执行的业务功能。

# Semaphore(信号量)

信号量，看源码就是基于 AQS 实现的，类似于***共享锁\***，只不过这个共享是有限制的共享，它能够控制共享的线程数量。Semaphore 就相当于一个许可证，线程需要先通过 acquire 方法获取该许可证，该线程才能继续往下执行，否则只能在该方法出阻塞等待。当执行完业务功能后，需要通过`release()`方法将许可证归还，以便其他线程能够获得许可证继续执行。

Semaphore 可以用于做流量控制，特别是**公共资源有限的应用场景，比如数据库连接**。假如有多个线程读取数据后，需要将数据保存在数据库中，而可用的最大数据库连接只有 10 个，这时候就需要使用 Semaphore 来控制能够并发访问到数据库连接资源的线程个数最多只有 10 个。在限制资源使用的应用场景下，Semaphore 是特别合适的。

![image-20200229234039915](00831rSTgy1gcdp7doykcj30u00u4wjf.jpg)

>   这里的队列就是就是很简单的用 ArrayList 进行存储，存储那些等待许可证的线程，没有阻塞然后加入队列然后唤醒这么麻烦…

```java
package 多线程.并发工具包;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

public class SemaphoreDemo {
    //表示老师只有5支笔
    private static Semaphore semaphore = new Semaphore(5);

    public static void main(String[] args) {

        //表示10个学生
        ExecutorService service = Executors.newFixedThreadPool(10);
        for (int i = 0; i < 10; i++) {
            service.execute(() -> {
                try {
                    semaphore.acquire();
                    System.out.println(Thread.currentThread().getName() + "  同学获取到笔，填表格");
                    TimeUnit.SECONDS.sleep(3);
                    semaphore.release();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }
        service.shutdown();
    }
}
```

# Exchanger (交换器)

可以直接在某个临界点，两个线程交换数据，不需要使用 wait/notify 或者是 condition 机制。

它提供了一个交换的同步点，在这个同步点两个线程能够交换数据。具体交换数据是通过 exchange 方法来实现的，如果一个线程先执行 exchange 方法，那么它会同步等待另一个线程也执行 exchange 方法，这个时候两个线程就都达到了同步点，两个线程就可以交换数据。

```java
//当一个线程执行该方法的时候，会等待另一个线程也执行该方法，因此两个线程就都达到了同步点
//将数据交换给另一个线程，同时返回获取的数据
V exchange(V x) throws InterruptedException
//同上一个方法功能基本一样，只不过这个方法同步等待的时候，增加了超时时间
V exchange(V x, long timeout, TimeUnit unit)
throws InterruptedException, TimeoutException
copypackage 多线程.并发工具包;

import java.util.concurrent.Exchanger;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class ExchangerDemo {
    private static Exchanger<String> exchanger = new Exchanger();
    public static void main(String[] args) {

        //代表男生和女生
        ExecutorService service = Executors.newFixedThreadPool(2);

        service.execute(() -> {
            try {
                //男生对女生说的话
                String girl = exchanger.exchange("我其实暗恋你很久了......");
                System.out.println("女孩儿说：" + girl);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        service.execute(() -> {
            try {
                System.out.println("女生慢慢的从教室你走出来......");
                TimeUnit.SECONDS.sleep(3);
                //男生对女生说的话
                String boy = exchanger.exchange("我也很喜欢你......");
                System.out.println("男孩儿说：" + boy);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

    }

}
copy女生慢慢的从教室你走出来......
男孩儿说：我其实暗恋你很久了......
女孩儿说：我也很喜欢你......
```

两句话交换了，这个真的牛逼！
