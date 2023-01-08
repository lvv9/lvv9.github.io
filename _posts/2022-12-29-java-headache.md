# 令人头疼的刁钻问题

## 集合

### ArrayList
ArrayList底层用Object[]实现，内部使用了两个空数组：
1. EMPTY_ELEMENTDATA
2. DEFAULTCAPACITY_EMPTY_ELEMENTDATA

源码中注释的说法是
> We distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when first element is added.

翻开1.8的源码，只有无参构造器会赋值DEFAULTCAPACITY_EMPTY_ELEMENTDATA：
```text
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```
在调用add方法时，只有DEFAULTCAPACITY_EMPTY_ELEMENTDATA是按DEFAULT_CAPACITY（10）扩容的：
```text
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
    private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
```
核心的扩容方法grow，是按1.5被扩容的，并保证扩容满足minCapacity：
```text
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```

### HashMap
HashMap底层（第一层）是Node<K,V>[]
> An instance of <tt>HashMap</tt> has two parameters that affect its
> performance: <i>initial capacity</i> and <i>load factor</i>.  The
> <i>capacity</i> is the number of buckets in the hash table, and the initial
> capacity is simply the capacity at the time the hash table is created.  The
> <i>load factor</i> is a measure of how full the hash table is allowed to
> get before its capacity is automatically increased.  When the number of
> entries in the hash table exceeds the product of the load factor and the
> current capacity, the hash table is <i>rehashed</i> (that is, internal data
> structures are rebuilt) so that the hash table has approximately twice the
> number of buckets.

HashMap核心的方法：
```text
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```
i = (n - 1) & hash，即把key值hash取模后放到桶中。
如果桶
1. 存在旧值（hash && key相等），包括桶节点、链表节点、树节点，则替换value，if (e != null) {...}
2. 否则插入并判断是否需要扩容

如果在桶链表插入过程中，binCount >= TREEIFY_THRESHOLD - 1，且capacity大于等于MIN_TREEIFY_CAPACITY（64）会进行树化（java.util.HashMap.TreeNode#treeify）。
否则capacity小于64只进行扩容，且扩容后不再进行树化操作，下一次冲突还是扩容，直到容量到达64。

红黑树put：
```text
        final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                                       int h, K k, V v) {
            Class<?> kc = null;
            boolean searched = false;
            TreeNode<K,V> root = (parent != null) ? root() : this;
            for (TreeNode<K,V> p = root;;) {
                int dir, ph; K pk;
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                    return p;
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0) {
                    if (!searched) {
                        TreeNode<K,V> q, ch;
                        searched = true;
                        if (((ch = p.left) != null &&
                             (q = ch.find(h, k, kc)) != null) ||
                            ((ch = p.right) != null &&
                             (q = ch.find(h, k, kc)) != null))
                            return q;
                    }
                    dir = tieBreakOrder(k, pk);
                }

                TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    Node<K,V> xpn = xp.next;
                    TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    xp.next = x;
                    x.parent = x.prev = xp;
                    if (xpn != null)
                        ((TreeNode<K,V>)xpn).prev = x;
                    moveRootToFront(tab, balanceInsertion(root, x));
                    return null;
                }
            }
        }
```
红黑树首先按hash排序，和前面一样，如果key相等就将旧值返回。（dir）小于时放左边，大于时放右边。
在hash相等的情况下，会通过判断是否实现了comparable比较，如果再相等再比较类名然后java.lang.System#identityHashCode。

resize初始化及扩容：
```text
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```
除去初始化的逻辑，基本就是扩大两倍（newThr = oldThr << 1），然后根据是树节点还是链表节点进行迁移。

### HashSet
HashSet实现上基本是对HashMap的包装，value为固定的 private static final Object PRESENT = new Object() 。
面试基本上是问以上两种集合，其它在笔试（算法）中比较常见。

### LinkedList
LinkedList使用双向链表实现：
```text
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```
同时还实现了这些接口：List、Deque（音同deck，双向队列），其中Deque还声明了栈的相关操作。

### ArrayDeque
用数组实现的双向队列，插入后自动扩容。

### PriorityQueue
优先队列，通过堆实现，底层也是数组Object[]。
> Priority queue represented as a balanced binary heap: the two
> children of queue[n] are queue[2*n+1] and queue[2*(n+1)].

在末尾添加元素，并与父节点比较进行调整满足堆的约束。
在头部删除元素时，将末尾元素移到头部，并比较两子节点进行调整满足堆的约束。

### TreeMap与TreeSet
TreeMap底层是红黑树，与HashMap中的红黑树类似，在增删元素后需要进行调整（改变节点颜色或旋转）以满足红黑树的约束。

红黑树的定义限制了树的高度不会大于2lg(n+1)，相对于AVL树来说，它的旋转次数少。

### ConcurrentHashMap
```text
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
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
                    if (binCount >= TREEIFY_THRESHOLD)
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
初始化initTable通过成员变量sizeCtl的CAS操作来进行同步操作。
如果桶节点为null，则CAS插入，否则加锁。

### 其它集合
- CopyOnWriteArrayList
- Vector
- ConcurrentLinkedQueue（CAS）
- BlockingQueue（ArrayBlockingQueue、LinkedBlockingQueue、PriorityBlockingQueue）

## 多线程并发

### 线程状态
运行
```text
    public static void main(String[] args) {
        while (true) {
            System.out.println("x");
        }
    }
```
用JDK自带的工具jstack可以查看到main线程
```text
"main" #1 prio=5 os_prio=31 tid=0x000000015280b000 nid=0x1603 runnable [0x000000016d696000]
   java.lang.Thread.State: RUNNABLE
	at java.io.FileOutputStream.writeBytes(Native Method)
	at java.io.FileOutputStream.write(FileOutputStream.java:326)
	at java.io.BufferedOutputStream.flushBuffer(BufferedOutputStream.java:82)
	at java.io.BufferedOutputStream.flush(BufferedOutputStream.java:140)
	- locked <0x00000006c0007de8> (a java.io.BufferedOutputStream)
	at java.io.PrintStream.write(PrintStream.java:482)
	- locked <0x00000006c00070b8> (a java.io.PrintStream)
	at sun.nio.cs.StreamEncoder.writeBytes(StreamEncoder.java:221)
	at sun.nio.cs.StreamEncoder.implFlushBuffer(StreamEncoder.java:291)
	at sun.nio.cs.StreamEncoder.flushBuffer(StreamEncoder.java:104)
	- locked <0x00000006c0007058> (a java.io.OutputStreamWriter)
	at java.io.OutputStreamWriter.flushBuffer(OutputStreamWriter.java:185)
	at java.io.PrintStream.write(PrintStream.java:527)
	- eliminated <0x00000006c00070b8> (a java.io.PrintStream)
	at java.io.PrintStream.print(PrintStream.java:669)
	at java.io.PrintStream.println(PrintStream.java:806)
	- locked <0x00000006c00070b8> (a java.io.PrintStream)
	at me.liuweiqiang.hibernate.EventListener.main(EventListener.java:61)
```
查看java.lang.Thread.State源码，有以下枚举：
- NEW
- RUNNABLE
- BLOCKED
- WAITING
- TIMED_WAITING
- TERMINATED

#### NEW
Thread state for a thread which has not yet started.

#### RUNNABLE
正常运行的线程。

#### BLOCKED
只要线程未获取到锁，就是BLOCKED状态，不管是执行同步方法/块是未获取到锁还是wait释放锁。
```text
    public static void main(String[] args) throws Exception {
        Thread thread = new Thread(() -> {
            synchronized (EventListener.class) {
                System.out.println("x");
                try {
                    EventListener.class.wait();
                } catch (InterruptedException e) { // 获取锁后才会catch
                    e.printStackTrace();
                }
            }
        });
        thread.start();
        TimeUnit.SECONDS.sleep(1);
        synchronized (EventListener.class) {
            EventListener.class.notifyAll(); // 需要特别注意notify信号可能会错过，因为一个可能的调度是notify时其它线程还尚未wait
            System.in.read();
        }
    }
```

#### WAITING
Object.wait后获取到锁未被notify、Thread.join、LockSupport.park时进入的状态。
对于ReentrantLock.lock、ReentrantLock.newCondition.await，由于底层调用的是LockSupport.park，所以线程状态不是BLOCKED而是WAITING或TIMED_WAITING。

#### TIMED_WAITING
同上，不过带时间参数，Thread.sleep时也是。

### Java内存模型
和分布式系统一样，在并发处理器下，不同的模型会有不同的一致性级别。
> The Java Memory Model describes what behaviors are legal in multithreaded code, and how threads may interact through memory. It describes the relationship between variables in a program and the low-level details of storing and retrieving them to and from memory or registers in a real computer system. It does this in a way that can be implemented correctly using a wide variety of hardware and a wide variety of compiler optimizations.
> Java includes several language constructs, including volatile, final, and synchronized, which are intended to help the programmer describe a program's concurrency requirements to the compiler. The Java Memory Model defines the behavior of volatile and synchronized, and, more importantly, ensures that a correctly synchronized Java program runs correctly on all processor architectures.

具体可见http://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html

中文版可以参考程晓明的《深入理解Java内存模型》。

从Java应用程序员的角度看的话，最重要的是Happens-before规则。
> It should be noted that the presence of a happens-before relationship between two actions does not necessarily imply that they have to take place in that order in an implementation. If the reordering produces results consistent with a legal execution, it is not illegal.

#### 懒汉单例
```java
public class Singleton {
    private volatile static Singleton instance;

    private Singleton() {
    }

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }

        return instance;
    }
}
```
单例的饿汉式写法因为一个bug变得非常经典。
第一个对 instance == null 的判断是为了避免对Singleton.class锁的无效竞争。
但是因为第一个判断不是同步的，所以多个线程可能"同时"得到true的结果。
而Singleton.class锁保护了第二个 instance == null 的判断，同时volatile修饰instance变量是必要的：避免未初始化的Singleton被引用。

### 死锁
当使用互斥锁来实现同步时，通常会遇到死锁的问题。死锁的四个条件：
- 互斥
- 持有并等待
- 非抢占
- 循环等待

### 锁优化
指HotSpot对synchronized的优化。
由于这些优化是与JVM实现相关的，大部分相关的原理需要查阅JVM的源代码（C++）。

包括：
- 锁粗化
- 锁消除
- 锁升级
- 自适应自旋锁 在自旋的基础上自适应自旋的次数
> Enable naive spinning on Java monitor before entering operating system thread synchronization code.

### CAS
即Compare-And-Swap。
几乎所有线程安全机制的底层实现都利用CAS来实现。
JVM的CAS底层是通过处理器CAS指令实现的，对于数据库CAS的实现，可以通过update set value='n' from table where version=2。

#### ABA
如果compare的值不是一个单调（递增）的值，那么就会存在value值A->B->A变化后被CAS修改的情况，如果这是一个问题，那么就需采用单调的版本号。

### AQS
AbstractQueuedSynchronizer。是众多同步器实现的抽象基类。

#### ReentrantLock
ReentrantLock(fair)中同步器FairSync的lock()会落到acquire(1)：
```text
    private volatile int state;
    private transient volatile Node head; // Node结点是等待获取资源的线程的封装
    private transient volatile Node tail; // 同时通过Node成员变量next/prev维护了一个队列

    public final void acquire(int arg) {
        if (!tryAcquire(arg) && // 获取锁成功后直接返回
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) // 否则加入队列
            selfInterrupt();
    }

        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() && // 即使这里可能会因为竞争问题倒是if判断过期了，还是会在acquireQueued()入队处理
                    compareAndSetState(0, acquires)) { // 获取到锁，所以setExclusiveOwnerThread是安全的
                    setExclusiveOwnerThread(current); // 这里还体现了公平锁的策略：只有!hasQueuedPredecessors()才去竞争锁
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) { // 可重入
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
        
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) { // 保护setTail及 pred.next = node
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
    
    private Node enq(final Node node) { // 自旋入队
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node())) // dummy哑节点
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) { // 保护setTail及 pred.next = node
                    t.next = node;
                    return t;
                }
            }
        }
    }
    
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) { // 若p == head的判断过期，自旋重新判断
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt()) // park失败会重新进入循环
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
    
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL); // 需要先SIGNAL再重新进入循环达到parkAndCheckInterrupt()避免竞争冒险
        }
        return false;
    }
    
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```

waitStatus：
```text
* Status field, taking on only the values:
*   SIGNAL:     The successor of this node is (or will soon be)
*               blocked (via park), so the current node must
*               unpark its successor when it releases or
*               cancels. To avoid races, acquire methods must
*               first indicate they need a signal,
*               then retry the atomic acquire, and then,
*               on failure, block.
*   CANCELLED:  This node is cancelled due to timeout or interrupt.
*               Nodes never leave this state. In particular,
*               a thread with cancelled node never again blocks.
*   CONDITION:  This node is currently on a condition queue.
*               It will not be used as a sync queue node
*               until transferred, at which time the status
*               will be set to 0. (Use of this value here has
*               nothing to do with the other uses of the
*               field, but simplifies mechanics.)
*   PROPAGATE:  A releaseShared should be propagated to other
*               nodes. This is set (for head node only) in
*               doReleaseShared to ensure propagation
*               continues, even if other operations have
*               since intervened.
*   0:          None of the above
```

#### synchronized vs. Lock
从上面的内容可以知道，synchronized与Lock最根本的不同在于：
synchronized是Java语言提供的特性，Java语言只规定了synchronized语义必须符合这样的，底层JVM可以根据需要进行不同的实现以及做出各种优化；
而Lock，只是JDK中的lib，从上面也可以看到基本上是用的volatile+Node队列（同步队列，如果有condition的话还包括nextWaiter变量维护的条件队列）+CAS(native)+自旋+LockSupport.park()（及unpark，底层是native）来实现的。

对比JVM实现与Lock实现：

|维度|synchronized|Lock
|:---:|:---:|:---:
|中断|不支持|支持
|超时|不支持|支持
|tryLock|不支持|支持
|公平|非公平|both
|可重入|支持|支持
|condition|一个|多个
|释放|隐式|显式

#### 常见基于AQS的工具
- ReentrantLock（CyclicBarrier）
- Semaphore（互斥锁是一个特殊的信号量，二元信号量）
- CountDownLatch
- ReentrantReadWriteLock