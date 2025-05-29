---
title: "Java ConcurrentHashMap"
author: "爱吃芒果"
description:
date: "2025-05-28T16:46:29+08:00"
image:
math:
license:
hidden: false
comments: true
draft: false
tags:
categories:
---

这篇是对[javadoop](https://www.javadoop.com/post/hashmap)对concurrentHashMap非常棒的源码解析的学习。

## Java7 HashMap

HashMap是一个非并发安全的hashmap，使用链表数组实现，逻辑比较简单。

要求容量始终为$2^n$，这样可以利用位运算计算下标，`index = hash & (length - 1)`

每次扩容为原先的2倍，这样迁移旧数据时，会将位置`table[i]`中的链表的所有节点，分拆到新的数组中的`newTable[i]`和`newTable[i + oldLenght]`位置。

## Java7 ConcurrentHashMap

ConcurrentHashMap由一个Segment数组实现，Segment通过继承ReentrantLock来进行加锁，所以每次需要加锁的操作锁住的就是一个Segment，有些地方将Segment称为分段锁，这样只要保证每个Segment都是线程安全的，就实现了全局的线程安全。

concurrencyLevel: 默认为16，也就是说ConcurrentHashMap有16个Segment，所以理论上，最多可以同时支持16个线程并发写，只要它们的操作分别分布在不同的Segment上，这个值可以在初始化的时候设置为其他值，但是一旦初始化后，它是不可以扩容的。

假设concurrentcyLevel为16，那么hash值的高4位被用于找到对应的Segment。

Segment内部是有数组+链表组成的

put操作需要对Segment加独占锁，内部操作类似于HashMap

get操作完全没有加锁，完全由代码实现保证不会发生问题。

## Java8 HashMap

Java8对HashMap进行了一些修改，引入了红黑树，我们可以通过hash快速定位到数组中的具体下表，但之后需要遍历整个链表寻找我们需要的键值对，时间复杂度取决于链表的长度，为`O(n)`。为了降低这部分的开销，在java8中，当链表中的元素达到了8个时，会将链表转换为红黑树，此时查找的时间复杂度为 `O(logn)`。

Java7中使用Entry来代表每个HashMap的数据节点，Java8中使用Node，基本没有区别，都是key, value, hash和next这四个属性，不过，Node只能用于链表的情况，红黑树的情况需要使用TreeNode。

## Java8 ConcurrentHashMap

```java
    /**
     * The array of bins. Lazily initialized upon first insertion.
     * Size is always a power of two. Accessed directly by iterators.
     */
    transient volatile Node<K,V>[] table;

    /**
     * The next table to use; non-null only while resizing.
     */
    private transient volatile Node<K,V>[] nextTable;
```

ConcurrentHashMap底层也是一个数组，每个元素要么是链表，要么是红黑树。

### put过程分析

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
  	// 得到hash值
    int hash = spread(key.hashCode());
  	// 记录相应链表的长度
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
      	// 如果数组为空，进行数组初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
      	// 查找该hash对应的数组位置处的元素
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
          	// 如果数组该位置为空，用一次cas操作将新值放入其中，如果cas失败，进入下一个循环
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
      	// 当前位置正在扩容，后面再解释，先略过
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
          	// f是该位置的头节点，而且不为空
            V oldVal = null;
          	// 获取数组该位置头结点的监视器锁
            synchronized (f) {
              	// 获取锁之后重新判断一下当前位置的节点是否已经改变，如果已经改变，进入下一个循环
                if (tabAt(tab, i) == f) {
                  	// 头结点的hash值大于0，说明是链表
                    if (fh >= 0) {
                      	// 用于累加，记录链表的长度
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                          	// 如果发现了相等的key，判断是否需要进行值覆盖，退出循环
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                          	// 到了链表的最末端，将这个新值放到链表的最后面
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) { // 红黑树
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
              	// 判断是否要将链表转换为红黑树，临界值和HashMap一样，也是8
                if (binCount >= TREEIFY_THRESHOLD)
                  	// 这个方法和HashMap中稍微有一点点不同，那就是它不是一定会进行红黑树转换
                  	// 如果当前数据的长度小于64，那么会选择进行数组扩容，而不是转换为红黑树
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

可以看到，ConcurrentHashMap在进行重要操作时，会对数组中的元素进行加锁，这样保证了加锁的粒度适合，避免过粗导致并发性能下降。

### 初始化数组 initTable

```java
/**
 * Initializes table, using the size recorded in sizeCtl.
 */
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
      	// 其他线程已经在初始化数组了，spin等待
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
      	// CAS一下，将sizeCtl设置为-1，表示抢到了锁
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                  	// DEFAULT_CAPACITY默认初始容量为16
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                  	// 初始化数组，长度为16或者初始化时提供的长度
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                  	// 将新的数组赋值给table，table是volatile
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
              	// 设置sizeCtl为sc
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

初始化一个合适大小的数组，然后会设置sizeCtl。

初始化方法中的并发问题是通过对sizeCtl进行一个CAS操作来控制的。

### 链表转红黑树： treeifyBin

```java
/**
 * Replaces all linked nodes in bin at given index unless table is
 * too small, in which case resizes instead.
 */
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
      	// MIN_TREEIFY_CAPACITY为64
      	// 所以，如果数组长度小于64的时候，其实也就是32或者16或者更小的时候，会进行数组扩容
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
          	// 数组扩容方法，稍后介绍
            tryPresize(n << 1);
      	// 当前位置是链表
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
          	// 加锁
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                  	// 遍历链表，建立一颗红黑树
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                  	// 将红黑树设置到数组相应位置
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```

### 扩容： tryPreSize

```java
/**
 * Tries to presize table to accommodate the given number of elements.
 *
 * @param size number of elements (doesn't need to be perfectly accurate)
 */
private final void tryPresize(int size) {
  	// c: size的1.5倍，再加1，再往上去最近的2的n次方
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
        tableSizeFor(size + (size >>> 1) + 1);
    int sc;
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;
      	// 初始化数组，和之前类似
        if (tab == null || (n = tab.length) == 0) {
            n = (sc > c) ? sc : c;
            if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
            }
        }
      	// 容量足够，跳出循环
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        else if (tab == table) {
            int rs = resizeStamp(n);
						// https://bugs.java.com/bugdatabase/view_bug.do?bug_id=8215409 sc < 0永远不会成立，所以这段代码不起作用，在jdk11中已经去掉
            if (sc < 0) {
                Node<K,V>[] nt;
              	// 是否允许当前线程参与扩容
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
              	// CAS将sizeCtl加1，然后执行transfer方法，此时nextTab不为null
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
          	// 将sizectl设置为负数，并且具有resize戳
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
        }
    }
}
```



```java
/**
 * The maximum number of threads that can help resize.
 * Must fit in 32 - RESIZE_STAMP_BITS bits.
 */
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
/**
 * The number of bits used for generation stamp in sizeCtl.
 * Must be at least 6 for 32bit arrays.
 */
private static int RESIZE_STAMP_BITS = 16;
/**
 * Returns the stamp bits for resizing a table of size n.
 * Must be negative when shifted left by RESIZE_STAMP_SHIFT.
 */
static final int resizeStamp(int n) {
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```

### 数据迁移： transfer

将原来的tab数组中的元素迁移到新的nextTab数组中。

之前提到的tryPresize方法中调用transfer不涉及多线程，但transfer方法可以在其他地方被调用。典型地，我们之前在说put方法的时候已经说过了，helpTransfer方法中会调用transfer方法。

此方法支持多线程执行，外围调用此方法时，会保证第一个发起数据迁移的线程，nextTab为null，之后再调用此方法的时候，nextTab不会为null。

原数组长度为n，所有我们有n个迁移任务，让每个线程每次负责一个小任务是最简单的，每做完一个任务再检测是否有其他没做完的任务，帮助迁移旧可以了，而Doug Lea使用了一个stride，简单理解就是步长，每个线程每次负责迁移其中的一部分，如每次迁移16个小任务。所以我们就需要一个全局的调度者来安排哪个线程执行哪几个任务，这个就是属性transferIndex的作用。

第一个发起数据迁移的线程会将transferIndex指向原数组最后的位置，然后从后往前的stride个任务属于第一个线程，然后将transferIndex指向新的位置，再往前的stride个任务属于第二个线程，以此类推，当然，这里说的第二个线程不是真的一定指代了第二个线程，也可以是同一个线程，其实就是将一个大的迁移任务分为一个个任务包。

之前提到，原数组i位置的键值对会被分配到新数组i位置和新数组i + oldLength位置，这样每个迁移小任务相互之前不存在资源竞争。

```java
/**
 * Moves and/or copies the nodes in each bin to new table. See
 * above for explanation.
 */
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
  	// stide在单核下直接等于n，多核模式下为 n >>> 3 / NCPU，最小值为16
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
  	// 如果nextTab为null，先进行一次初始化
  	// 之前提过，外围会保证第一个发起迁移的线程调用此方法时，参数nextTab为null
  	// 之后参与迁移的线程调用此方法时，nextTab不为null
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
          	// 容量翻倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
      	// 赋值给nextTable属性
        nextTable = nextTab;
      	// transfer属性用于控制迁移的位置
        transferIndex = n;
    }
    int nextn = nextTab.length;
  	// ForwardingNode翻译过来就是正在迁移的Node
  	// 这个构造方法会生成一个Node, key, value和next都是null，关键是hash为MOVED
  	// 后面我们会看到，原数组中位置i处的节点完成迁移工作后，将会将位置i处设置为这个ForwardingNode，用来告诉其他线程该位置已经处理过了
  	// 所以它其实相当于一个标志
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
  	// advance指的是做完了一个位置的迁移工作，可以准备做下一个位置的了
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
  	// i是位置索引，bound是边界，注意是从后往前
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
      	// advance为true表示可以进行下一个位置的迁移了
      	// 简单理解结局：i指向了transferIndex, bound指向了transferIndex - stride
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
          	// 将transferIndex赋值给nextIndex
          	// 这里transferIndex一旦小于等于0，说明原数组的所有位置都有相应的线程去处理了
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            else if (U.compareAndSetInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
              	// nextBound是这次迁移任务的边界，注意是从后往前
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
              	// 所有迁移操作都已经完成
                nextTable = null;
              	// 将新的nextTab赋值给table属性，完成迁移
                table = nextTab;
              	// 重新计算sizeCtrl: n为原数组长度，所以sizeCtrl得出的值将是新数组长度的0.75倍
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
          	// 之前我们说过，sizeCtrl在迁移前会设置为 (rs << RESIZE_STAMP_SHIFT) + 2
          	// 然后，每有一个线程参与迁移就会将sizeCtrl + 1
          	// 这里使用CAS操作对sizeCtrl进行减一，表示做完了属于自己的任务
            if (U.compareAndSetInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
              	// 任务结束，方法退出
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
              	// 所有的迁移任务都已经做完
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
      	// 如果位置i处是空的，没有任何节点，那么放入刚才初始化的ForwardingNode节点
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
      	// 该位置处是一个ForwardingNode，代表该位置已经迁移过了
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
          	// 对数组该位置处的节点加锁，开始处理数组该位置处的迁移工作
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                  	// 头节点的hash大于0，说明是链表的Node节点
                    if (fh >= 0) {
                      	// 这里展开解释一下runBits
                      	// 假设一个key的hash值为 0010
                      	// table数组的长度总是2^n，这里假定为4,也就是 0100
                      	// 所以这个对象会被放入 i = hash & (length - 1)位置（效果等价于 hash % length)，当然这是因为length一定为2^n
                      	// 还有哪些对象可能被放入这个数组位置呢，需要满足 hash & 0011 = 0010
                      	// 显然低两位要保持一致，不能改变，其余高位可以任意
                      	// 现在发生了两倍扩容，i位置都需要迁移到新的位置 新的位置通过 hash & 0111决定
                      	// 原先数组位置中元素的hash值倒数第三位可能为0，也可能为1，也就是这里的runBits，所以元素会被分布到不同的位置
                      	// 如果为0，和原先一样，分布到i为止
                      	// 如果是1，分布到 i + oldLength位置，oldLength表示原先的数组长度
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                      	// 这个循环实际上是一个优化，从前往后遍历单向链表，找到最后的一段链表，
                      	// 链表段的所有元素根据hash值将添加到新数组的同一个位置
                      	// 这个优化去掉，不会对功能有任何影响
                      	// 比较坏的情况就是每次lastRun都是链表中最后一个元素或者很靠后的元素，那么这次遍历就有点浪费了
                      	// 不过Doug Lea也说了，根据统计，如果使用默认的阈值，大约只有1/6的节点需要克隆
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                     		// 迁移完成，设置对应位置的元素，advance设置为true，表示该位置已经迁移完毕
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {
                      	// 红黑树的迁移
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                      	// 如果一分为二后，节点数小于8，那么将红黑树转换回链表
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```

### get过程分析

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
      	// 判断头节点是否就是查找的节点
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
      	// 如果头节点的hash小于0，说明正在扩容，或者该位置是红黑树
        else if (eh < 0)
          	// 参考 ForwardingNode.find(int h, Object k) 和 TreeBin.find(int h, Object k)
            return (p = e.find(h, key)) != null ? p.val : null;
      	// 遍历链表
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

可以看到get操作的实现基本上是无锁的。

```java
/**
 * A node inserted at head of bins during transfer operations.
 */
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        super(MOVED, null, null, null);
        this.nextTable = tab;
    }

    Node<K,V> find(int h, Object k) {
        // loop to avoid arbitrarily deep recursion on forwarding nodes
        outer: for (Node<K,V>[] tab = nextTable;;) {
            Node<K,V> e; int n;
            if (k == null || tab == null || (n = tab.length) == 0 ||
                (e = tabAt(tab, (n - 1) & h)) == null)
                return null;
            for (;;) {
                int eh; K ek;
                if ((eh = e.hash) == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
                if (eh < 0) {
                    if (e instanceof ForwardingNode) {
                        tab = ((ForwardingNode<K,V>)e).nextTable;
                        continue outer;
                    }
                    else
                        return e.find(h, k);
                }
                if ((e = e.next) == null)
                    return null;
            }
        }
    }
}
```

