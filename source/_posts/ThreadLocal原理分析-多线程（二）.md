---
title: ThreadLocal原理分析 --- 多线程（二）
author: linWang
date: 2022-08-07 19:33:07
tags: Threadlocal原理
categories: Java并发
---

>   本文带大家分析下ThreadLocal的原理及常见的问题

<!--more-->

>   [主要参考了ThreadLocal的原理部分，就是这篇文章终于让我明白我想错了](http://www.jasongj.com/java/threadlocal/)
>
>   [初始化的时候，注意不要保存对象的引用，这样的话拷贝的副本也必然是对象的引用，最终都会改变对象，这样就不对了](https://my.oschina.net/clopopo/blog/149368)
>
>   [ThreadLocal 简述](https://www.jianshu.com/p/30ee77732843)
>
>   [ThreadLocal 内存泄漏问题](https://www.jianshu.com/p/dde92ec37bd1)
>
>   [ThreadLocal相关问题！非常有深度！面试之前一定要再看看！！！！！强烈推荐！！！](https://juejin.im/post/5a0e985df265da430e4ebb92)

## ThreadLocal 简介

-   ThreadLocal 与线程同步无关。

    ThreadLocal虽然提供了一种解决多线程环境下成员变量的问题，但是它并不是解决多线程共享变量的问题。对于多线程资源共享的问题，同步机制采用了**“以时间换空间”**的方式，而ThreadLocal采用了“**以空间换时间**”的方式。前者仅提供一份变量，让不同的线程排队访问，而后者为每一个线程都提供了一份变量，因此可以同时访问而互不影响。

-   正如上面所言，每个线程都会复制一份 ThreadLocal 的副本，然后线程内部进行使用，所谓的 ThreadLocal就是指 “线程局部变量”。

## ThreadLocal 原理

看源码的时候，被 ThreadLocal、ThreadLocalMap、threadLocals 绕晕了，最后终于绕明白了…其实还是自己先入为主的去思考了 ThreadLocal 的实现，最重要的是，**每一个线程都是拥有一个 ThreadLocalMap** 的，为什么要用 Map 而不使用 Pair 呢？因为每个线程可能拥有非常多个 ThreadLocal 变量。再重复一遍，ThreadLocalMap 是从属于每个线程的，每个线程自己管理自己的 map，既然说到这，我就再详细的说一下。

### 我的想法—-ThreadLocal 维护线程与实例的映射

因为我看 ThreadLocalMap 是 ThreadLocal 的 内部类，所以以为从属关系是 Map 从属于该变量，那一个可能的方案就是每一个 ThreadLocal 变量都维护一个 ThreadLocalMap，我们都知道，每个访问 ThreadLocal 变量的线程都有自己的一个“本地”实例副本。线程通过该 ThreadLocal 的 get() 方案获取实例时，只需要以线程为键，从 Map 中找出对应的实例即可。该方案如下图所示：

![ThreadLocal side Map](00831rSTgy1gcc5uxgu0yj31740jmjua.jpg)





该方案可满足上文提到的每个线程内一个独立备份的要求。每个新线程访问该 ThreadLocal 时，需要向 Map 中添加一个映射，而**每个线程结束时，应该清除该映射**。这里就有两个问题：

-   增加线程与减少线程均需要写 Map，故需保证该 Map 线程安全。虽然有几种实现线程安全 Map 的方式，但它或多或少都需要锁来保证线程的安全性。
-   **线程结束时，需要保证它所访问的所有 ThreadLocal 中对应的映射均删除，否则可能会引起内存泄漏**。（后文会介绍避免内存泄漏的方法）

其中锁的问题，应该是 JDK 未采用该方案的一个原因。

### 真正的实现—-Thread 维护ThreadLocal与实例的映射

上述方案中，出现锁的问题，原因在于多线程访问同一个 Map。如果该 Map 由 Thread 维护，从而使得每个 Thread 只访问自己的 Map，那就不存在多线程写的问题，也就不需要锁。该方案如下图所示。

![ThreadLocal side Map](00831rSTgy1gcc5yy8ppuj312a0n8gov.jpg)



该方案虽然没有锁的问题，但是由于每个线程访问某 ThreadLocal 变量后，都会在自己的 Map 内维护该 ThreadLocal 变量与具体实例的映射，如果不删除这些引用（映射），则这些 ThreadLocal 不能被回收，可能会造成内存泄漏。后文会介绍 JDK 如何解决该问题。

## ThreadLocal 使用 Demo

在这里，我们先来使用一下 ThreadLocal，再来分析其源码。

```java
package 多线程;

import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ThreadLocalTest {

}

class ThreadLocalDemo {
    private static ThreadLocal<SimpleDateFormat> sdf = new ThreadLocal<>();
    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        for (int i = 0; i < 100; i++) {
            executorService.submit(new DateUtil("2019-11-25 09:00:" + i % 60));
        }

    }

    static class DateUtil implements Runnable {
        private String date;

        public DateUtil(String date) {
            this.date = date;
        }

        @Override
        public void run() {
            if (sdf.get() == null) {
                sdf.set(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
            } else {
                try {
                    Date date = sdf.get().parse(this.date);
                    System.out.println(date);
                } catch (ParseException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

-   注意我们在这里，用的是 private static 修饰的 ThreadLocal，下文有说为何官方推荐这样修饰。
-   注意到我们这里使用的方法是 sdf.get()，sdf.set(obj)。

## ThreadLocal 源码分析

分析之前，我说一下重点，最核心的当然是 ThreadLocal 中的内部类 ThreadLocalMap，整个底层的实现基本全部依托于该内部类，同时别忘了为了 Thread 和 ThreadLocalMap 一一对应，Thread 了类中有一个成员变量—-threadLocals。

```java
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
```

接下来正式进入源码分析接阶段：

### ThreadLocal 部分方法

#### set()

```java
 public void set(T value) {
     Thread t = Thread.currentThread();
     ThreadLocalMap map = getMap(t);
     if (map != null)
         map.set(this, value);
     else
         createMap(t, value);
 }

 ThreadLocalMap getMap(Thread t) {
     return t.threadLocals;
 }

void createMap(Thread t, T firstValue) {
     t.threadLocals = new ThreadLocalMap(this, firstValue);
 }
```

逻辑如下：

1.  通过 getMap() 拿到当前线程对应的 ThreadLocals，注意这里可能有多个 ThreadLocal，所以返回的是一个map；
2.  如果 map 为空，说明该线程还没有使用过 ThreadLocal 变量，此时就调用 createMap() 新建一个 ThreadLocalMap，这里 map 又出现了，相信大家已经感受到了重要性了。
3.  如果 map 不为空，我们就把 key = 当前的 ThreadLocal，value 即为当前线程的 ThreadLocal 副本 对应的值。

#### get()

```java
public T get() {
   Thread t = Thread.currentThread();
   ThreadLocalMap map = getMap(t);
   if (map != null) {
       ThreadLocalMap.Entry e = map.getEntry(this);
       if (e != null) {
           @SuppressWarnings("unchecked")
           T result = (T)e.value;
           return result;
       }
   }
   return setInitialValue();
}
 private T setInitialValue() {
       T value = initialValue();
       Thread t = Thread.currentThread();
       ThreadLocalMap map = getMap(t);
       if (map != null)
           map.set(this, value);
       else
           createMap(t, value);
       return value;
  }
```

逻辑如下：

1.  通过当前线程拿到 map，然后拿到对应的 key 的 entry，然后返回对应的 value 值。
2.  如果 map 为空，则 调用 setInitialValue()，调用初始化 ThreadLocal 时的值。

#### remove()

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}
```

懒得解释了…还是调用的 ThreadLocalMap 中的 remove() 函数。

### ThreadLocalMap（一）

前面的方法已经铺垫很久了，所有的方法基本都是基于这个类实现的。ThreadLocal 700多行源代码，有500多行都是内部类 ThreadLocalMap。

#### 数据结构

必须要说明的是，这里的 map 不同于我们之前介绍的 HashMap、TreeMap等，这里的 map 底层采用的就是只有数组，没有用到红黑树和链表，因为这里解决冲突的方法并不是链地址法，而是开放地址法中的**线性探测**。所以这里数组的基本单元是 Entry。

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

>   Tip: 这里有一个比较特殊的类 WeakReference，也就是弱引用，我们后面会详细介绍这个，因为这个跟内存泄漏有很大的关系。

然后初始值跟 HashMap 一样，都是 16，但是其装载因子变成了 2/3，装载因子越大，越容易出现冲突的现象，但是装载因子越小，就越浪费空间，我们可以看到，这里的装载因子是比 HashMap 要小的，所以这里更不容易产生冲突，这也是采用开放地址法的原因吧。

现在来分析一下主要的函数：

#### set()

```java
private void set(ThreadLocal<?> key, Object value) {
  	//这里有一段英文注释，意思是不会像 get() 一样，直接去找，成功了就返回，没成功就再调用另外一个函数
  	// 因为 set()大概率是失败的多，所以就直接写在一个函数内了。
    Entry[] tab = table;
    int len = tab.length;
    //根据threadLocal的hashCode确定Entry应该存放的位置
    int i = key.threadLocalHashCode & (len-1);

    //采用开放地址法，hash冲突的时候使用线性探测
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        //覆盖旧Entry
        if (k == key) {
            e.value = value;
            return;
        }
        //当key为null时，说明threadLocal强引用已经被释放掉，那么就无法
        //再通过这个key获取threadLocalMap中对应的entry，这里就存在内存泄漏的可能性
        if (k == null) {
            //用当前插入的值替换掉这个key为null的“脏”entry
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    //新建entry并插入table中i处
    tab[i] = new Entry(key, value);
    int sz = ++size;
    //插入后再次清除一些key为null的“脏”entry,如果大于阈值就需要扩容
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

逻辑如下：

1.  先确定 ThreadLocal 所在的位置；
2.  然后采用线性探测法，确定位置；
3.  如果存在该 ThreadLocal，那么直接覆盖就好了；如果为 null，说明该 ThreadLocal的强引用已经消失，此时直接在该位置插入即可；如果不存在，那么就新建一个 Entry，插入，最后再次清楚 脏entry，并判断是否需要扩容。「后面会详细讲专门处理脏 entry 的这几个方法」

#### getEntry()

```java
private Entry getEntry(ThreadLocal<?> key) {
    //1. 确定在散列数组中的位置
    int i = key.threadLocalHashCode & (table.length - 1);
    //2. 根据索引i获取entry
    Entry e = table[i];
    //3. 满足条件则返回该entry
    if (e != null && e.get() == key)
        return e;
    else
        //4. 未查找到满足条件的entry，额外在做的处理
        return getEntryAfterMiss(key, i, e);
}
copyprivate Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            //找到和查询的key相同的entry则返回
            return e;
        if (k == null)
            //解决脏entry的问题
            expungeStaleEntry(i);
        else
            //继续向后环形查找
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

#### resize()

```java
/**
 * Double the capacity of the table.
 */
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    //新数组为原数组的2倍
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            //遍历过程中如果遇到脏entry的话直接另value为null,有助于value能够被回收
            if (k == null) {
                e.value = null; // Help the GC
            } else {
                //重新确定entry在新数组的位置，然后进行插入
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }
    //设置新哈希表的threshHold和size属性
    setThreshold(newLen);
    size = count;
    table = newTab;
}   
private static int nextIndex(int i, int len) {
  return ((i + 1 < len) ? i + 1 : 0);
}
```

扩容还是非常简单的，因为不涉及多线程，而且只用到数组，整体逻辑如下：

1.  new 一个原来容量两倍的数组；
2.  遇到了脏 entry，就将其 value 置为 null，方便 GC，如果 key != null，就重新计算 entry 在新数组的位置；
3.  然后插入，如果有冲突就线性探测就好了；
4.  最后更新 threshold、size、table就好啦。

#### remove()

```java
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            //将entry的key置为null
            e.clear();
            //将该entry的value也置为null
            expungeStaleEntry(i);
            return;
        }
    }
}
```

这里也很简单，就不说了，就跟上面的一样，在这疯狂用 expungeStaleEntry()！！！

### ThreadLocalMap（二）

这里单独开一小节，讲 ThreadLocalMap中是如何处理脏 entry 的，也就是 key == null 的键值对，这样会导致内存泄漏，所以必须处理。

#### cleanSomeSlots()

可以看到，核心依旧是 expungeStaleEntry()。

```java
// 清除部分 脏entry
// 启发式的扫描过期数据并擦除，启发式是这样的：
// 如果实在没有过期数据，那么这个算法的时间复杂度就是O(log s)
// 如果有过期数据，那么这个算法的时间复杂度就是O(log n)
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
      	// 如果 entry 不为空，且entry中的key指向的是 null
      	// 说明产生了 脏entry
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);	
    return removed;
}
```

整体逻辑如下：

1.  从插入的entry索引的后一位开始检测是否有 脏entry；
2.  先n == size，log n 去检测，如果发现有脏entry，立即加大n = length，然后从下一个哈希桶为 null 的索引位置 i 继续 log n 检测；
3.  最后返回是否有修改。

#### expungeStaleEntry()

```java
/**
 * Expunge a stale entry by rehashing any possibly colliding entries
 * lying between staleSlot and the next null slot.  This also expunges
 * any other stale entries encountered before the trailing null.  See
 * Knuth, Section 6.4
 *
 * @param staleSlot index of slot known to have null key
 * @return the index of the next null slot after staleSlot
 * (all between staleSlot and this slot will have been checked
 * for expunging).
 */
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    //清除当前脏entry
    // expunge entry at staleSlot
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    //2.往后环形继续查找,直到遇到table[i]==null时结束
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        //3. 如果在向后搜索过程中再次遇到脏entry，同样将其清理掉
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            //处理rehash的情况
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

该方法逻辑请看注释（第1,2,3步），主要做了这么几件事情：

1.  清理当前脏entry，即将其value引用置为null，并且将table[staleSlot]也置为null。value置为null后该value域变为不可达，在下一次gc的时候就会被回收掉，同时table[staleSlot]为null后以便于存放新的entry;
2.  从当前staleSlot位置向后环形（nextIndex）继续搜索，直到遇到哈希桶（tab[i]）为null的时候退出；
3.  若在搜索过程再次遇到脏entry，继续将其清除。

也就是说该方法，**清理掉当前脏entry后，并没有闲下来继续向后搜索，若再次遇到脏entry继续将其清理，直到哈希桶（table[i]）为null时退出**。因此方法执行完的结果为 **从当前脏entry（staleSlot）位到返回的i位，这中间所有的entry不是脏entry**。为什么是遇到null退出呢？原因是存在脏entry的前提条件是 **当前哈希桶（table[i]）不为null**,只是该entry的key域为null。如果遇到哈希桶为null,很显然它连成为脏entry的前提条件都不具备。

cleanSomeSlot方法主要有这样几点：

1.  从当前位置i处（位于i处的entry一定不是脏entry）为起点在初始小范围（log2(n)，n为哈希表已插入entry的个数size）开始向后搜索脏entry，若在整个搜索过程没有脏entry，方法结束退出
2.  如果在搜索过程中遇到脏entryt通过expungeStaleEntry方法清理掉当前脏entry，并且该方法会返回下一个哈希桶(table[i])为null的索引位置为i。这时重新令搜索起点为索引位置i，n为哈希表的长度len，再次扩大搜索范围为log2(n)继续搜索。

下面，以一个例子更清晰的来说一下，假设当前table数组的情况如下图。

![img](00831rSTgy1gcd21mceyyj30kh08cq34.jpg)



1.  如图当前n等于hash表的size即n=10，i=1,在第一趟搜索过程中通过nextIndex,i指向了索引为2的位置，此时table[2]为null，说明第一趟未发现脏entry,则第一趟结束进行第二趟的搜索。
2.  第二趟所搜先通过nextIndex方法，索引由2的位置变成了i=3,当前table[3]!=null但是该entry的key为null，说明找到了一个脏entry，**先将n置为哈希表的长度len,然后继续调用expungeStaleEntry方法**，该方法会将当前索引为3的脏entry给清除掉（令value为null，并且table[3]也为null）,但是**该方法可不想偷懒，它会继续往后环形搜索**，往后会发现索引为4,5的位置的entry同样为脏entry，索引为6的位置的entry不是脏entry保持不变，直至i=7的时候此处table[7]位null，该方法就以i=7返回。至此，第二趟搜索结束；
3.  由于在第二趟搜索中发现脏entry，n增大为数组的长度len，因此扩大搜索范围（增大循环次数）继续向后环形搜索；
4.  直到在整个搜索范围里都未发现脏entry，cleanSomeSlot方法执行结束退出。

#### replaceStaleEntry()

```java
/*
 * @param  key the key
 * @param  value the value to be associated with key
 * @param  staleSlot index of the first stale entry encountered while
 *         searching for key.
 */
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                               int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    // Back up to check for prior stale entry in current run.
    // We clean out whole runs at a time to avoid continual
    // incremental rehashing due to garbage collector freeing
    // up refs in bunches (i.e., whenever the collector runs).

    //向前找到第一个脏entry
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))
        if (e.get() == null)
1.          slotToExpunge = i;

    // Find either the key or trailing null slot of run, whichever
    // occurs first
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        // If we find key, then we need to swap it
        // with the stale entry to maintain hash table order.
        // The newly stale slot, or any other stale slot
        // encountered above it, can then be sent to expungeStaleEntry
        // to remove or rehash all of the other entries in run.
        if (k == key) {
            
            //如果在向后环形查找过程中发现key相同的entry就覆盖并且和脏entry进行交换
2.            e.value = value;
3.            tab[i] = tab[staleSlot];
4.            tab[staleSlot] = e;

            // Start expunge at preceding stale entry if it exists
            //如果在查找过程中还未发现脏entry，那么就以当前位置作为cleanSomeSlots
            //的起点
            if (slotToExpunge == staleSlot)
5.                slotToExpunge = i;
            //搜索脏entry并进行清理
6.            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        // If we didn't find stale entry on backward scan, the
        // first stale entry seen while scanning for key is the
        // first still present in the run.
        //如果向前未搜索到脏entry，则在查找过程遇到脏entry的话，后面就以此时这个位置
        //作为起点执行cleanSomeSlots
        if (k == null && slotToExpunge == staleSlot)
7.            slotToExpunge = i;
    }

    // If key not found, put new entry in stale slot
    //如果在查找过程中没有找到可以覆盖的entry，则将新的entry插入在脏entry
8.    tab[staleSlot].value = null;
9.    tab[staleSlot] = new Entry(key, value);

    // If there are any other stale entries in run, expunge them
10.    if (slotToExpunge != staleSlot)
        //执行cleanSomeSlots
11.        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

这个有点复杂，暂时就不去掌握了…只需要知道这个函数用当前插入的值替换掉这个key为null的“脏”entry，并且并不局限于如此，还把附近所有可能是脏entry都清理了，应该算是整个 ThreadLocal中最为复杂的函数了。

### ThreadLocalMap 与 HashMap 的部分区别

-   解决冲突的方式不一样。一个使用的是开放地址法，一个是链地址法；
-   默认装载因子不同。ThreadLocalMap 2/3，HashMap 0.75；
-   ThreadLocalMap 没有扰动函数，直接就是相与(也就是取模的过程)，而 HashMap 有高16位与低16位相与进行扰动；

扩容和初始容量都是一样的，初始16，然后扩容都是两倍。

## 弱引用

>   [四大引用的概念](https://blog.csdn.net/luoyeyeyu/article/details/85621663)

Q：弱引用的概念

A：四大引用。强引用、弱引用、软引用、幻想引用。弱引用通过WeakReference类实现。 **弱引用的生命周期比软引用短。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。**由于垃圾回收器是一个优先级很低的线程，因此不一定会很快回收弱引用的对象。弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java虚拟机就会把这个弱引用加入到与之关联的引用队列中。

Q：这里的key对ThreadLocal为弱引用的作用，如果使用强引用呢？

A：这里的 key 对 ThreadLocal 采用弱引用，是为了减少内存泄漏的发生。当 ThreadLocal 强引用全部消失时，Entry 中的 key 会变为 null，但此时 value 还是被 map 持有强引用，此时 entry 就是可能会造成内存泄漏，不可用但是可达，无法 gc，针对这个问题，THreadLocalMap中可以调用 set()、get()、remove()等方法都会触发 expungeStaleEntry机制，将脏 entry清除掉防止内存泄漏。如果像以前使用了强引用，那么当对象使用完 ThreadLocal 变量，ThreadLocal由于一直被 key 持有强引用，无法回收，并且该 Entry 也过期了，导致更大的内存泄漏，并且此时无法通过 expungeStaleEntry机制解决该问题。**至于造成内存泄漏的原因，根本在于 ThreadLocalMap 和 Thread 拥有同样长的生命周期。**

重要的事情说三遍！用完ThreadLocal最后一定要 remove()、remove()、remove()。

Q：Entry、value 使用弱引用，是什么效果？

A：Entry定义为弱引用：**当GC回收后，无法区分是原本就没有写入还是被回收了，后续线性探测的修补也无法完成。**

value定义为弱引用：似乎也是个不错的方法，为啥没这么做？**因为这么做和将key定义为弱引用基本没区别，仍然可以依赖弱引用机制清理，但通常在我们的使用中不会持有value的强引用，只会持有key即ThreadLocal对象的强引用，而value没有强引用的情况下会被GC回收，与我们期望的功能不符。**

## 内存泄漏

>   主要参考网址：https://juejin.im/post/5d75ebf451882576c478a102

这应该是 ThreadLocal 最难的部分了，网上查阅了几百篇文章，基本上把所有有关的 ThreadLocal 内存泄漏的文章都看了一遍…只有这一篇比较欣赏，剩下的基本上全部是在互相抄…好多地方解释的也不通…接下来希望自己能把这个讲通吧…

#### 什么是内存泄漏

对象已经没有被应用程序使用，但是垃圾回收器没办法移除它们，因为还在被引用着。
在Java中，**内存泄漏**就是存在一些被分配的对象，这些对象有下面两个特点，**首先**，这些对象是可达的，即**在有向图中，存在通路可以与其相连**；**其次**，**这些对象是无用的，即程序以后不会再使用这些对象**。如果对象满足这两个条件，这些对象就可以判定为Java中的内存泄漏，这些对象不会被GC所回收，然而它却占用内存。

#### ThreadLocal 什么时候会发生内存泄漏

我个人倾向于将其分为两类：

##### 当 ThreadLocal 变量不使用 static 修饰

当 ThreadLocal 变量不使用 static 修饰，那么 ThreadLocal 的强引用就很有可能全部消失「什么时候会消失？？ 这个翻了整个互联网都没见到…」，而 ThreadLocalMap 中的 key 对 ThreadLocal 的引用是弱引用，根据 GC 规则，ThreadLocal 变量会被回收，此时 ThreadLocaMap 对应的 key 会自动变为 null，而 此时的 Entry 和 value 又被 ThreadLocalMap 强引用，即不可能会被GC，所以此时就造成了内存泄漏，解决的方法就是显示的调用 set()、get()、remove()「当然调用 set()、get() 也不一定能解决内存泄漏问题，因为这两个方法可能在没碰到脏 entry就结束循环了」等方法，因为其可以调用 expungeStaleEntry() 将这些 key 为 null 的键值对回收。当然了，如果线程会结束，那最终也不会发生内存泄漏啦，线程死了那肯定就自动清理 map 了。

Q：回到上面那个问题，什么时候强引用会消失？

A：我参考的网址终于有了一个例子。强引用是**对应的子线程或主线程中某个对象**持有的，**对象生命周期结束**或**对象替换指向这个key的引用**后，key的强引用也就断了。对，就是对象生命周期结束，强引用就消失了。

eg：假如我现在有一个主线程，然后我调用了 A，创建了 A 的实例对象 a1，a1 调用了 ThreadLocal 变量，此时对象 a1 是有对 ThreadLocal 的强引用的，此时 ThreadLocalMap 中的 key还不会为 null ，当主线程调用了 a = null，该对象生命周期结束，此时强引用消失了，但是 ThreadLocalMap 中的 value 还是强引用，无法 GC，也就是上面的情况。

```java
public class A {
    private ThreadLocal<Context> local = new ThreadLocal<>();

    public void doSth() {
        // Context ctx = ...
        local.set(ctx);
    }
}

public static void main(String[] args){
  	A a = new A();
  	a.doSth();
  	a = null; // 或者直接 finalize();
  	// ...
  	doSometing();
}
```

##### 当 ThreadLocal 变量使用了 static 修饰

当 ThreadLocal 变量使用了 static 修饰，说明该变量的生命周期和类一样的，只要引用不改变，那么该变量会一直持有强引用（强引用上面已经讲了），那么此时 key 对 ThreadLocal 持有弱引用我个人觉得就没有任何意义了，因为是不可能触发 expungeStaleEntry() 的，因为 key 不会变为 null。那么此时是不是就不会出现内存泄漏了呢？**不，依旧会出现。**其实此时如果我们正常使用，是不会产生内存泄漏的，但是一旦我们使用的是线程池，线程使用完之后不会回收，而我们之前使用的 ThreadLocalMap 又存在很多过期的 value 需要清理，此时就算 key 是弱引用，也没有办法，因为 ThreadLocal 一直持有强引用根本不会触发 GC，此时就产生了内存泄漏。

举个例子：

1.  线程1和线程2都用了threadLocal1和threadLocal2，且设置了value
2.  线程1使用完毕归还线程池，但没有调用threadLocal1.remove()
3.  之后线程1不再使用threadLocal1了，仅使用threadLocal2
4.  线程1的threadLocalMap中仍然保存了obj1
5.  由于静态变量threadLocal1引用仍然可达，不会被回收，线程1无法触发expungeStaleEntry机制，threadLocal1对应的entry和value无法回收，造成了内存泄漏

>   所以用private static修饰之后，好处就是仅使用有限的ThreadLocal对象以节约创建对象和后续自动回收的开销，坏处是需要我们手动调用remove方法清理使用完的slot，否则会有内存泄漏问题。

## 问题

>   https://juejin.im/post/5a0e985df265da430e4ebb92#heading-2
>
>   https://zhuanlan.zhihu.com/p/91756169

Q1：ThreadLocal 跟普通局部变量有何区别，在其内部使用局部变量不是一样的吗？

A1：网上有两种说法。

1.  该实例需要在多个方法中共享，但不希望被多线程共享，此时可以使用 ThreadLocal，这样使得代码耦合度更低，无需在每个方法中都去声明一个局部变量，简而言之，就是做到线程间隔离，线程内共享。我个人觉得这种说法有些过于牵强，但是貌似网上基本都是这种说法…

2.  我觉得 ThreadLocal 相比局部变量而言，其实也就是 全局变量 和 局部变量 的区别。虽然每个线程的 ThreadLocal 副本互不影响，但是我可以通过 ThreadLocal 进行通信，这是局部变量所不具备的能力。举个例子，假设要设计这么一个程序：三个线程同时从0数到10，一旦有一个线程数到了10，三个线程都从0重新开始数，一直这么循环。用run创建局部变量的方式就没法做，因为每一个线程之间是完全独立的。

    ThreadLocal保存共享变量的副本，用于特定的使用场景。看源码就知道ThreadLocal类是基于Thread类来实现的，如果直接用继承Thread来实现上面的功能也是可以的，只不过JDK都已经给封装好了，自己能想到的思路估计跟ThreadLocal的实现思路差不多。如果场景适合使用ThreadLocal，为何不直接用呢。

    所以，线程间可通信且可以做到互不影响，我觉得这才是 ThreadLocal 真正存在的意义吧。

Q2：既然 ThreadMap 从属于每个线程，那为何不把 ThreadLocalMap 放到 Thread 类中，反而是成为了 ThreadLocal 的内部类？

A2：其实我觉得都是可以的，不管放到 Thread 类中，还是放到 ThreadLocal 中，都是可以解决问题的，但是我觉得作者这样设计，主要是因为在之前 ThreadLocalMap 是从属于 ThreadLocal 的，可能后来考虑到我上文提到的需要加锁还有线程结束带来的 ThreadLocal 的内存泄漏问题，改成了 ThreadLocalMap 从属于 Thread，为了最大限度的不改动代码，只在 Thread 中添加了一个成员变量 `ThreadLocal.ThreadLocalMap threadLocals`，并且这样设计的话，用户层面去调用 api 的时候根本不会发现 ThreadLocalMap 的存在，设计的还是非常优美的！

Q3：为何官方建议定义 ThreadLocal 时 采用 private static 修饰？

A3：可以避免重复创建TSO(Thread Specific Object，即与线程相关的变量)，如果把 ThreadLocal 声明为某个类的实例变量（而不是静态变量），那么每创建一个该类的实例就会导致一个新的TSO实例被创建。显然，这些被创建的TSO实例是没有任何意义的。既然所有的线程用到的 ThreadLocal 都是一样的，那么很明显设计成 static 是很正常的，不以对象为出发点，而以整个类为出发点就好了。

>   所以用private static修饰之后，好处就是仅使用有限的ThreadLocal对象以节约创建对象和后续自动回收的开销，坏处是需要我们手动调用remove方法清理使用完的slot，否则会有内存泄漏问题。

Q4：ThreadLocal 的用途？

A4：

1.  线程间隔离，线程内通信，解决相同变量在多线程环境下的冲突问题，例如：
    -   hibernate中的session使用；
        -   **spring中单例bean的多线程访问**。我们知道在一般情况下，只有无状态的`Bean`才可以在多线程环境下共享，在`Spring`中，绝大部分`Bean`都可以声明为`singleton`作用域。就是因为`Spring`对一些`Bean`（如`RequestContextHolder`、`TransactionSynchronizationManager`、`LocaleContextHolder`等）中非线程安全状态采用`ThreadLocal`进行处理，让它们也成为线程安全的状态，有状态的`Bean`就可以在多线程中共享了；
        -   **以及 spring 如何保证数据库事务在同一个连接下执行的**。DataSourceTransactionManager 是spring的数据源事务管理器， 它会在你调用getConnection()的时候从数据库连接池中获取一个connection， 然后将其与ThreadLocal绑定， 事务完成后解除绑定。这样就保证了事务在同一连接下完成。
2.  可以用于线程间通信，比如三个线程都用到了 ThreadLocal，只要有一个达到了目标值，我就全部设为初值，这是可以通过 ThreadLocal直接完成的。

Q5：ThreadLocal 和 Synchronized 区别？

A5：`ThreadLocal`和`synchronized`关键字都用于处理多线程并发访问变量的问题，只是二者处理问题的角度和思路不同。

1.  `ThreadLocal`是一个Java类,通过**对当前线程中的局部变量的操作来解决不同线程的变量访问的冲突问题**。所以，`ThreadLocal`提供了线程安全的共享对象机制，每个线程都拥有其副本。
2.  Java中的`synchronized`是一个保留字，它依靠JVM的锁机制来实现临界区的函数或者变量的访问中的原子性。在**同步机制**中，通过对象的锁机制保证同一时间只有一个线程访问变量。此时，被用作“锁机制”的变量时多个线程共享的。
3.  同步机制(`synchronized`关键字)采用了以“**时间换空间**”的方式，提供一份变量，让不同的线程排队访问。而`ThreadLocal`采用了“**以空间换时间**”的方式，为每一个线程都提供一份变量的副本，从而实现同时访问而互不影响。
