# 令人头疼的问题

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
    private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
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
核心的扩容方法grow，是按1.5倍扩容的，并保证扩容满足minCapacity：
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
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true); // 内部的扰动函数使得碰撞的可能性更低，见 https://www.zhihu.com/question/20733617/answer/111577937
    }
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
容量设计为2次幂是基于算法效率上的考虑，如使用e.hash & (newCap - 1)计算下标。

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
同时还实现了Deque（音同deck，双向队列）接口，其中Deque还声明了栈的相关操作。

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
            EventListener.class.notifyAll(); // 需要特别注意notify信号可能会错过，因为一个可能的调度是notify时其它线程还尚未wait（尚未synchronized）
            System.in.read();
        }
    }
```

#### WAITING
Object.wait后获取到锁未被notify、Thread.join、LockSupport.park时进入的状态。
对于ReentrantLock.lock、ReentrantLock.newCondition.await，由于底层调用的是LockSupport.park，所以线程状态不是BLOCKED而是WAITING或TIMED_WAITING。

Tips：notify是"不可靠的"，被notify的对象可能会错过notify信号。

#### TIMED_WAITING
同上，不过带时间参数，Thread.sleep时也是。

### Java内存模型
和分布式系统一样，在并发处理器下，不同的模型会有不同的一致性级别。
简单地说，JMM是一种模型，抽象出了Java程序员与JVM实现之间的契约。
> The Java Memory Model describes what behaviors are legal in multithreaded code, and how threads may interact through memory. It describes the relationship between variables in a program and the low-level details of storing and retrieving them to and from memory or registers in a real computer system. It does this in a way that can be implemented correctly using a wide variety of hardware and a wide variety of compiler optimizations.
> Java includes several language constructs, including volatile, final, and synchronized, which are intended to help the programmer describe a program's concurrency requirements to the compiler. The Java Memory Model defines the behavior of volatile and synchronized, and, more importantly, ensures that a correctly synchronized Java program runs correctly on all processor architectures.

具体可见http://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html

中文版可以参考程晓明的《深入理解Java内存模型》。

从Java应用程序员的角度看的话，最重要的是Happens-before规则：
If one action happens-before another, then the first is visible to and ordered before the second.
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
而Singleton.class锁保护了第二个 instance == null 的判断，同时volatile修饰instance变量是必要的：
这里的synchronized只保护引用，不保护对象的构造，volatile避免未初始化的Singleton被引用。

### 死锁
当使用互斥锁来实现同步时，通常会遇到死锁的问题。死锁的四个条件：
- 互斥
- 持有并等待
- 非抢占
- 循环等待

#### 解决死锁

死锁防止，破坏上面四个条件之一：
- 如使用ThreadLocal或String这样的不可变对象
- 开始前一次性申请所有资源的锁
- 无法申请时主动/被动释放资源
- 资源排序，按序申请

死锁避免，运行时避免死锁，如银行家算法

死锁检测和解除

### 锁优化
指HotSpot对synchronized的优化。
由于这些优化是与JVM实现相关的，大部分相关的原理需要查阅JVM的源代码（C++）。

包括：
- 锁粗化 避免频繁的锁开销
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

#### synchronized （HotSpot）实现
同步块：monitorenter、monitorexit指令
修饰方法：方法修饰符ACC_SYNCHRONIZED
虚拟机中C++实现的ObjectMonitor对象，Monitor（管程、监视器）是一种高级同步原语，为了更容易编写正确的程序。

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

#### 常见基于AQS的工具
- ReentrantLock（CyclicBarrier）
- Semaphore（互斥锁是一个特殊的信号量，二元信号量）
- CountDownLatch
- ReentrantReadWriteLock

### 线程池
见 https://liuweiqiang.me/2021/11/17/java-thread-source.html

在Spring异步任务中会需要配置线程池。

### ThreadLocal
见 https://liuweiqiang.me/2020/09/08/qs&tree.html

在请求的生命周期内进行context的访问时用到、Spring @Transactional也用来保存事务的上下文。

## JVM
JVM与Java语言其实没有太大的关系，但它们都和class字节码有着重要的联系。

### JVM规范
JVM规范描述的是一种抽象化的虚拟机的行为，而不是任何一种广泛使用的虚拟机实现。
JVM实现可以自由地决定不在规范中描述的细节，如运行时的数据区如何布局，选用哪种垃圾收集算法，优化等。
详见《Java虚拟机规范（Java SE 8版）》

### 运行时数据区
即运行时会用到的数据的区域，其中一些区域与虚拟机进程的生命周期绑定，另外一些与线程的绑定。
- pc寄存器 其值可以用三元表达式表达（运行方法是native? undefined: 字节码指令地址），与线程绑定
- Java虚拟机栈（线程）
- Java堆（进程） 需要垃圾回收
- 方法区（进程） 内含运行时常量池（对象类型的只包含引用，对象数据在堆中），可以不进行垃圾回收<br>
  运行时常量池通过类或接口的class字节码常量池构造
- 本地方法栈

> 运行时数据区是一种概念模型，具体的Java虚拟机并不一定要完全照着概念模型的定义来进行设计，可能会通过一些更高效率的等价方式去实现它。

字符串常量池在哪个区域值得商榷，本人自己测试String.valueOf(i++).intern()同时调节MaxMetaspaceSize是不会有MetaSpace的OOM的（Zulu OpenJDK）。

以下方法也没看出intern时哪个区域的使用大小有变化（jconsole）
```text
    public static void main(String[] args) throws Exception {
        int size = 11116384;
        Set<String> set = new HashSet<>(size + 1);
        int count = 0;
        String zero = String.valueOf(size - 1);
        set.add(zero);
        String copy = String.valueOf(size - 1);
        if (zero != copy.intern() && copy == zero.intern()) {
            System.out.println("aaa");
        }
        for (int i = size; i < 2 * size; i++) {
            set.add(String.valueOf(i));
        }
        for (String s: set) {
            if (s == s.intern()) {
                count++;
            }
        }
        System.out.println(set.size());
        System.out.println(count);
    }
```

#### HotSpot内存参数
- -Xss Java虚拟机栈大小
- -Xms、-Xmx Java堆最小最大大小
  - -Xmn 新生代大小，即将将-XX:NewSize与-XX:MaxNewSize设为一致
- -XX:MaxMetaspaceSize 元空间（方法区的HotSpot实现）大小，默认无上限
- -MaxDirectMemorySize 直接内存，不属于运行时数据区，New IO中的DirectByteBuffer对象会用到，直接内存受此参数与本地内存的限制

#### HotSpot对象的创建
1. 类加载检查
2. 分配内存 有两种安全的分配方式：CAS、TLAB（UseTLAB参数控制）
3. 初始化零值（虚拟机的初始化）
4. 设置对象头
5. 执行构造器方法（Java的初始化）

#### 堆问题排查
- HotSpot参数【-XX:+HeapDumpOnOutOfMemoryError】获取OOM时HeapDump文件以进一步分析
- 或如果是内存泄漏还没溢出，在通过系统工具找到占用大的进程后，使用jmap获取HeapDump文件（jps -l 查看pid）
  > jmap -dump:<dump-options> <pid>

获取HeapDump文件后可通过jhat工具的histo图、rootset等方式分析
> jhat -port 1099 dump.test

在打开资源时，良好的习惯是使用try-with-resources或finally释放。

#### 其它内存问题
栈问题一般都比较容易排查。

元空间（方法区）在OOM时也会提示MetaSpace的OOM，通常是因为：
- 大量的CGLib
- 大量的JSP
- 大量的OSGi

运行时还可以通过HotSpot参数NativeMemoryTracking（生产一般不用）来先排查是哪部分区域有问题。
部分参数可以通过jinfo运行时修改。

### GC
垃圾收集与内存分配有着密切的关系，在选择用哪种分配和回收算法时通常需要综合考虑。

#### 为什么需要GC
因为Java的内存管理系统是自动的，解放了Java应用开发人员。

#### GC的过程

##### 根节点枚举、可达性分析
在分配内存的时候，如遇容量不足（不同区域的不足会进行不同程度的回收），虚拟机首先需要暂停所有的用户线程（Stop-The-World）枚举出根节点，然后从GC Roots开始分析对象的可达性，以便进一步对不可达的对象进行回收。
期间会穿插用户线程的并发，Stop-The-World用来保证被标记的对象不会再被引用。
有两种方式降低这种停顿：
- 增量更新
- 原始快照

典型的GC Roots包括：
- 虚拟机栈中引用的对象，线程还在使用这些对象，所以不能回收
- 方法区中类静态属性引用的对象
- 被同步锁持有的对象

对于分代回收等算法，因为只回收一部分区域，其它区域可能会引用，所以需要把这些关联的【粒度适中的】区域也加到GC Roots。

##### 垃圾收集算法
较为经典的有：
- 分代收集 因此HotSpot还将堆分成了新生代、老年代，新生代又分Eden（伊甸园）、Survivor 1、Survivor 2
- 标记清除 适用于不需要大量清除的情况，如老年代
- 标记复制 与标记清除相反
- 标记整理 与清除算法类似，但需要移动对象，移动则内存回收时会更复杂，不移动则内存分配时会更复杂，结果是移动的话吞吐量大、不占CPU时间，不移动时停顿少

#### 引用类型
这里涉及一些 https://liuweiqiang.me/2020/09/08/qs&tree.html

#### 垃圾收集器选型
一般需要综合停顿时间、吞吐量、多处理器利用，一般交易服务器使用ParNew+CMS实现低停顿时间。

### 类加载
类加载有三个阶段：
1. 加载 加载的结果是得到Class<?>，其中数组类是由虚拟机创建的，不存在数组类的外部class字节码
2. 链接 与C语言类似，链接使得类与类之间在运行时能相互作用，链接又可以细分
   - 验证
   - 准备 为静态变量分配变量并初始化零值
   - 解析
3. 初始化 运行类的初始化方法<cinit>

jvms（Java虚拟机规范）规定了两种类加载器：
- 引导类加载器 通常是加载<JAVA_HOME>/lib中的规定的类
- 用户自定义的类加载器 需继承java.lang.Classloader

在通常的虚拟机实现（JDK 8）中，是按双亲委派的模型实现的：
1. 引导类加载器
2. 扩展类加载器 Java实现，父加载器是null，自动使用引导类加载器作为父加载器
3. 应用程序类加载器 Java实现，加载ClassPath
> 如果一个类加载器收到了类加载的请求，它不会自己去尝试加载这个类，
> 而是把这个请求委派给父加载器去完成，每一个层次的类加载器都是如此，
> 因此所有的加载请求最终都应该传送到最顶层的启动类加载器中，只有当
> 父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需
> 的类）时，子加载器才会尝试自己去完成加载。

这里的父子指的是委派关系上的父子，而不是继承关系上的父子。

使用双亲委派的一个好处是，保证Java程序的稳定运行，避免类重复加载造成的混乱。

### 问题排查及调优
需要具体问题具体分析。

#### 应用执行耗时分析
|方案|优势|劣势
|:---:|:---:|:---:
|System.currentTimeMillis()|简单|代码侵入且繁琐
|StopWatch|简单且更加方便|代码侵入且繁琐
|AOP|较少侵入|私有和静态方法无法拦截
|Java Agent、Attach API（Arthas trace）|无侵入|开发门槛较高，需要注意多Agent之间的执行顺序、冲突

#### 重放+Debug
在开发阶段最常用手段，线上一般不允许使用。

如果远程服务支持，可以远程Debug。

#### 日志分析异常
包括应用日志、中间件日志，测试阶段常用，线上可能会屏蔽低级别日志。

应用问题排查要求对应用日志打印得较全。个人经验是打印方法所有的输入，如入参、DAO的返回等。

有时会漏打，通过JVM的一些API（Java Agent、Attach API）可以实现一些运行时增强，如BTrace、Arthas。

#### jstack查看线程状态
在遇到Java应用阻塞时常用。

##### CPU高占用
```shell
ps -emT | more # 查看进程与线程
ps -eT Huk -pcpu | more # 按CPU占用排序查看线程
```
查看进程与线程CPU占用，jstack <PID> 然后通过16进制的SPID（小写）找到相应的线程分析。

还可以通过专业工具的火焰图分析。

#### JDK内存分析工具
分析OOM与内存泄漏时常用，见上《运行时数据区》。

#### GC日志
同样也是分析内存问题时常用，需JVM参数
- -XX:+PrintGCDetails 或简化版-verbose:gc
- -XX:+PrintGCTimeStamps
- -XX:+PrintGCDateStamps
- -XX:+UseGCLogFileRotation
- -XX:NumberOfGCLogFiles=1
- -XX:GCLogFileSize=1M
- -Xloggc:<file>
- -XX:+PrintHeapAtGC

获得GC日志后可以通过gceasy.io等分析工具分析。也可结合jstat。

新生代频繁GC，可以考虑加大新生代的大小。

## TCP
RFC 793

### 分层
TCP的分法，有四层：
1. 链路层
2. 网络层 IP
3. 传输层 TCP、UDP
4. 应用层 HTTP、FTP、TELNET

OSI分了七层：
1. 物理层
2. 数据链路层
3. 网络层
4. 传输层
5. 会话层
6. 展示层
7. 应用层

### 可靠性
这个说法不一，比起实现可靠，或许捋清楚不可靠的原因更加值得关注，这方面在《自顶向下方法》这本书中描述得比较清楚：
1. 数据包乱序
2. 数据包丢失
3. 数据包错误

### 三次握手与四次挥手
由于是面向连接的，握手双方必须交换初始的控制信息（ISN等），因此至少需要前两次握手。
而TCP又是可靠的，对于控制信息包（SYN）也是如此（因而SYN也消耗一个序号），所以ACK是必须的，因而通信双方至少个需要一个ACK。
由于服务方的SYN与ACK可以通过同一包返回，因此握手是三次。

在特殊情况下，握手可以是四次的，比如双方同时打开。

对于SYN的可靠传输，带来一个异常处理上的好处是，ACK接收方（连接发起方）可以识别出错误的包，当ACK所确认的序号，未记录在ACK接收方时，便是一个错误。

对于挥手，由于需要允许半关闭的情况，挥手的ACK由底层内核的TCP完成，而第二次FIN一般是由应用层根据应用逻辑发起关闭。

### 关闭
发起关闭的一方，即执行主动关闭，在包处理完成后，进入TIME_WAIT（2MSL）状态，这样在另一方FIN发送失败或者主动关闭方ACK发送失败时能够重发FIN及ACK。
同时，TIME_WAIT使得连接变得松弛，不会有旧的包进入新的连接造成错误。

被动关闭的一方，即执行被动关闭，半关闭时的状态是CLOSE_WAIT。

### HTTP
由 请求行（请求）/状态行（响应） + 头部 + 空行 + 消息体 组成

状态码：
- 1XX 信息性状态码，表示继续处理
- 2XX 成功
- 3XX 重定向
- 4XX 客户端错误
- 5XX 服务端错误

请求方法：
- GET
- POST
- PUT
- DELETE
- PATCH
- HEAD
- OPTIONS
- CONNECT
- TRACE

#### GET vs. POST
|-|GET|POST
|:---:|:---:|:---:
|缓存|可以|一般不支持
|服务器状态|无变化|有变化
|请求参数|在URL和HEAD，因而长度的编码有限制|在BODY
|多次请求|无副作用|有副作用
|其它浏览器行为|-|-

#### HTTPS
见 https://liuweiqiang.me/2021/12/23/implements-ssl-client.html

## 微服务栈
这里也有一些说明 https://liuweiqiang.me/2022/08/29/ddd-&-hibernate.html

常用的框架及中间件（可以参考Spring Cloud）：
- 网关
- RPC/REST
- 消息队列
- 数据库（包括RDBMS和NoSQL）
- 注册中心、配置中心
- 分布式事务 https://liuweiqiang.me/2022/12/28/consistency.html
- 监控跟踪

### 分布式锁
如果Redis客户端是Redisson，建议直接用封装好的。

基本思路是通过唯一约束来实现，包括Redis的key、RDBMS的唯一索引。
加锁时插入，解锁时删除。

### 限流
由于通常的分布式锁并无fencing token这样的保护，这样的分布式锁只能用来降低对资源的竞争而不能避免。

比较成熟的方案是通过限流或消息队列来缓解。

#### 算法
- 固定窗口计数器
- 滑动窗口计数器 减小突发流量击穿的概率，滑动间隔越短概率越小
- 漏桶 请求以固定速率被消费，桶满了就限流，可能会无故地增加延迟
- 令牌桶 桶空了就限流，启动时最好初始化成满的

### MySQL

#### 架构
![架构](https://github.com/lvv9/lvv9.github.io/blob/master/pic/mysql-architecture.png?raw=true)

#### 事务
https://liuweiqiang.me/2019/01/28/database-note.html & https://liuweiqiang.me/2022/12/28/consistency.html

#### 索引
- B+树索引（包括主键） https://liuweiqiang.me/2020/09/08/qs&tree.html
- hash索引

#### Oracle
|-|MySQL|Oracle
|:---:|:---:|:---:
|默认端口|3306|1521
隔离级别|通常意义上的4种|只有2（Read Committed、Serializable）+1（Read Only）种

#### 优化

##### 慢查询日志
对于MySQL的分析，其中一种是通过自带的慢查询日志（slow_query_log）来查看记录的慢查询。

##### Explain
查看执行计划。

##### 数据库优化
包括但不限于
- 根据需要创建索引，利用排好序的冗余信息来获得更好的查询速度
- 避免索引失效，如避免左模糊、对谓词所含的列使用函数、谓词中的操作无法使用索引
- 反范式，增加冗余
- 减少数据量，如覆盖索引、只select必要的列、分页等
- 减少网络IO，如在程序循环外一次性读取而不是循环中读取、batchInsert代替循环insert
- 使用预编译SQL而不是动态SQL
- 读写分离，使用主从复制分离读请求和写请求，也存在一些现实问题见DDIA
- 分库分表，微服务中更多的是垂直分库，水平分叫sharding，shard后涉及到分流（路由）、再平衡、事务的问题

##### Index Condition Pushdown
> Without ICP, the storage engine traverses the index to locate rows in the base table and returns them to the MySQL server which evaluates the WHERE condition for the rows.
> With ICP enabled, and if parts of the WHERE condition can be evaluated by using only columns from the index, the MySQL server pushes this part of the WHERE condition down to the storage engine.
> The storage engine then evaluates the pushed index condition by using the index entry and only if this is satisfied is the row read from the table.
> 
> EXPLAIN output shows Using index condition in the Extra column when Index Condition Pushdown is used.
> 
> Index Condition Pushdown is enabled by default.

#### drop vs. truncate vs. delete
|-|drop|truncate|delete
|:---:|:---:|:---:|:---:
|类型|DDL|DDL|DML
|回滚|不可以|不可以|可以
|元数据|删除|重置|不变
|速度|快|较快｜较慢
|触发器|不触发|不触发|触发

### Redis

#### 数据类型
- STRING 字符串，RedisObject封装，底层类型之一的SDS更安全、长度函数复杂度低、支持二进制、空间预分配减少了修改时内存分配的花销
- LIST 列表，类似于Java的List
- SET 集合，类似于Java的Set
- HASH 哈希，类似于Java的HashMap
- ZSET 有序集合

此外还有BitMap（Java中的BitSet）：
> When key does not exist, a new string value is created.
> The string is grown to make sure it can hold a bit at offset.
> The offset argument is required to be greater than or equal to 0, and smaller than 2^32 (this limits bitmaps to 512MB).
> When the string at key is grown, added bits are set to 0.

HyperLogLog（提供不精确的去重计数）。

#### RedisObject（数据类型底层实现）
```text
Redis objects can be encoded in different ways:

Strings can be encoded as:

raw, normal string encoding.
int, strings representing integers in a 64-bit signed interval, encoded in this way to save space.
embstr, an embedded string, which is an object where the internal simple dynamic string, sds, is an unmodifiable string allocated in the same chuck as the object itself. embstr can be strings with lengths up to the hardcoded limit of OBJ_ENCODING_EMBSTR_SIZE_LIMIT or 44 bytes.

Lists can be encoded as ziplist or linkedlist. The ziplist is the special representation that is used to save space for small lists.

Sets can be encoded as intset or hashtable. The intset is a special encoding used for small sets composed solely of integers.

Hashes can be encoded as ziplist or hashtable. The ziplist is a special encoding used for small hashes.

Sorted Sets can be encoded as ziplist or skiplist format. As for the List type small sorted sets can be specially encoded using ziplist, while the skiplist encoding is the one that works with sorted sets of any size.

All the specially encoded types are automatically converted to the general type once you perform an operation that makes it impossible for Redis to retain the space saving encoding.
```

#### 多线程
得益于内存的利用，Redis使用单线程即可高效地执行命令：
> Redis is, mostly, a single-threaded server from the POV of commands execution.

为了获得更高的性能，除了IO多路复用，Redis 6增加了IO线程来处理IO，命令执行保持使用原来的主线程处理。

#### 持久化
- RDB（全量）
  - save命令 阻塞主进程
  - bgsave 创建子进程后主进程解除阻塞，手动bgsave命令执行或根据配置自动执行
- AOF（增量） 三种刷盘策略
  - appendfsync always
  - appendfsync everysec
  - appendfsync no
- 混合

#### 部署
- 单机 可用性低
- 主从 从节点发送PSYNC，主节点发送RDB及增量，不能自动故障转移
- 哨兵 并发依旧不高，容量依旧不大，机器利用率低
- 集群 至少3主3从

#### 集群
```text
CLUSTER NODES
```
查看集群节点信息

```text
Every Redis Cluster node requires two open TCP connections: a Redis TCP port used to serve clients, e.g., 6379, and second port known as the cluster bus port. By default, the cluster bus port is set by adding 10000 to the data port (e.g., 16379); however, you can override this in the cluster-port configuration.

Cluster bus is a node-to-node communication channel that uses a binary protocol, which is more suited to exchanging information between nodes due to little bandwidth and processing time. Nodes use the cluster bus for failure detection, configuration updates, failover authorization, and so forth. Clients should never try to communicate with the cluster bus port, but rather use the Redis command port. However, make sure you open both ports in your firewall, otherwise Redis cluster nodes won't be able to communicate.
```

由于分了三个主节点，每个主节点负责一部分数据，因此同样涉及分流（路由）、再平衡、事务的问题。

既然用了Redis，关系型数据库的事务特性几乎无法使用了。

Redis再平衡的做法是：
固定分16384个分区，如果集群中添加了一个新节点，该新节点可以从每个现有的节点上匀走几个分区。如果从集群中删除节点，则采取相反的均衡措施。
选中的整个分区会在节点之间迁移，但分区的总数量仍维持不变，也不会改变关键字到分区的映射关系（CRC16后对16383取模）。这里唯一要调整的是分区与节点的对应关系。
考虑到节点间通过网络传输数据总是需要些时间，这样调整可以逐步完成，在此期间，旧的分区仍然可以接收读写请求。

Redis服务端只支持有限的路由服务：
- MOVED重定向 客户端在遇到MOVED后应更新映射关系
- ASK重定向 仅在迁移过程中出现

#### 应用
点击量计数、排行榜、分布式缓存、分布式锁、集群共享会话等。

在应用于缓存时，需要考虑以下情况不适用：
- 频繁修改、写多于读
- 强一致

对于应用配置、业务参数等更新不频繁而又需要较强的一致性（主要是原子性）时，可以考虑增加版本号，将数据转换为不可变数据，继而使用缓存。

#### 内存管理
达到maxmemory时的驱逐策略：
- noeviction
- allkeys-lru
- allkeys-lfu
- volatile-lru
- volatile-lfu
- allkeys-random
- volatile-random
- volatile-ttl Removes keys with expire field set to true and the shortest remaining time-to-live (TTL) value.

> Redis reclaims expired keys in two ways: upon access when those keys are
> found to be expired, and also in background, in what is called the
> "active expire key".

#### 缓存一致性
如果需要非常强的一致性，只能通过共识算法解决。

退而求其次，用小概率不一致的做法：
- Cache Aside 先更新数据库，后失效缓存。对于因缓存失效失败的问题，可以1.不操作自动过期；2.提供操作后台强制失效。

#### 缓存失效问题
- 缓存穿透 Cache Penetration 缓存null结果而不是缓存数据
- 缓存击穿 Cache Breakdown 临时降低并发
- 缓存雪崩 Cache Avalanche 随机化过期时间

#### lua
Redis的命令执行模型是单线程的，但在应用组合多命令时没有原子性，这时可以通过lua来完成。

### ZooKeeper

#### 应用
- 分布式锁
- 分布式序列
- 配置中心

#### 节点类型
Zookeeper的数据模型是由树形的多个znode节点组成的。节点的类型包括从两个维度组合包括：
- 临时节点
- 永久节点
- 临时顺序节点
- 永久顺序节点

节点可以注册watcher，以利用ZK的通知机制。

#### ZAB
https://zookeeper.apache.org/doc/r3.8.1/zookeeperInternals.html

##### 角色
- Leader 负责事务的提交
- Follower 转发事务请求，同步状态，处理读请求，参与提议与选举
- Observer 转发事务请求，同步状态，处理读请求，不参与提议与选举

##### 工作状态
- Active messaging 集群工作在这种状态时，Leader按（改进的）两阶段提交的方式进行：
  1. Leader按事务ID——zxid（epoch+count组成）顺序提议（写请求）
  2. Follower按序回复，过半通过后Leader发送COMMIT
- Leader activation 启动、恢复时进入这种状态进行Leader选举，选举后Leader需先将状态同步到Follower。
  选举选出服务器最新事务ID中的epoch任期最大的，epoch有相同的则选出事务ID中的count最大的，接着是机器ID最大的。
  如果Leader在其任内出现任期大于它的事务，且这个Leader任内zxid最大的那部分提议没有被任一参与投票的Follower COMMIT，说明已有新的Leader，那么这部分过期的提议会被放弃。

### 消息队列
一般应用于：
- 异步 有助于实现响应式的风格
- 削峰
- 解藕 实现观察者模式

#### RocketMQ

##### 部署模型
- Producer 消息生产者
- Consumer 消息消费者
- Broker 消息代理，接收生产者发送的消息，发送给消费者，存在主从（还可以多主）：
  - ASYNC_MASTER 异步指与SLAVE的交互
  - SYNC_MASTER 可靠
  - SLAVE

  Broker刷盘：
  - SYNC_FLUSH 可靠
  - ASYNC_FLUSH 性能
- NameServer 为生产者和消费者提供服务发现，无状态

##### 消息模型
- 主题 与Kafka的Topic类似
- 队列 对应Kafka的分区，顺序投递时需要实现MessageQueueSelector
- 消费者组 同Kafka，一个消费者组订阅主题的全部内容，组内消费者均衡分配队列（RocketMQ集群模式，也就是通常意义上的队列模型，不同的消费者组可以当作发布订阅模型）。
  还有广播模式，组内消费者处理所有队列的消息（一般不用）
- 标签Tag RocketMQ中用来在业务上归类的标识，使用Tag（二级分类）可以实现对Topic（一级分类）中的消息进行过滤，官方文档说明了【什么时候该用Topic，什么时候该用Tag？】
- 生产者组 用在运维与事务消息上：
  ```此外，需要注意的是事务消息的生产组名称ProducerGroupName不能随意设置。事务消息有回查机制，回查时Broker端如果发现原始生产者已经崩溃，则会联系同一生产者组的其他生产者实例回查本地事务执行情况以Commit或Rollback半事务消息。```

##### 消费模式
- Push 消费者实现Listener的接口，将消费方法注册
- Pull

##### 特色消息
- 顺序消息 顺序消息的代价太高，建议在应用层保证，不要依赖中间件的顺序性保证。 https://liuweiqiang.me/2022/03/16/kafka-in-action.html
  RocketMQ的顺序保证只能保证同一生产者的顺序性，且需要满足：
  - 生产者顺序投递至同一队列（即同一分区），且需要同步发送，在同步发送成功前不能发送下一消息
  - Broker故障时消息不乱序，这种配置会影响可用性
  - 消费者顺序消费
- 延时消息 setDelayTimeLevel()设置，在Broker内部（内部SCHEDULE_TOPIC_XXXX主题）暂存后投递到真正的Topic当中。
- 事务消息 RocketMQ通过生产者检查本地事务状态与"回查"的方式来实现事务消息：
  ```在断网或者是生产者应用重启的特殊情况下，若服务端未收到发送者提交的二次确认结果，或服务端收到的二次确认结果为Unknown未知状态，经过固定时间后，服务端将对消息生产者即生产者集群中任一生产者实例发起消息回查。```
  没有实现共识，理论上是不构成原子提交的，但也是业界比较常用的做法了。
  事务消息的内部主题是RMQ_SYS_TRANS_HALF_TOPIC。

##### 可靠消息
依旧从应用层保证，可以参考TCP。
简单版：生产者数据与消息在事务数据库同步提交，生产者定时任务将消息发送，失败则重试。
在得到消费者明确的成功返回（或退而求其次Broker的返回，同时Broker配置高可靠）后删除消息或改变消息状态（或者消费者调生产者服务）。
消费者实现幂等。

##### 积压
基本思路是让消费速度大于生产速度：
- 消费者扩容

如果是临时的：
- 增加队列容量

##### 高并发、高吞吐实现
- 顺序读写
- mmap实现的零拷贝
- 可选的异步刷盘
- 可选的生产者异步发送
- 可选的生产者批量发送
- 批量消费
- 分区（即队列）

### Dubbo

#### 架构
- 服务提供者
- 服务消费者 利用代理模式实现实现类，并通过观察者模式监听注册中心实现服务提供者变更
- 注册中心 有：
  - ZooKeeper
  - Nacos
  - Redis
  - Multicast
- 监控中心 消费方、提供方通过责任链模式在过滤器中收集信息并发送到监控中心

服务提供者将自身信息注册到注册中心（ZK的临时节点，可通过ZK自带的命令行工具zkCli查看/dubbo子节点），客户端运行时到注册中心取服务提供者信息，并缓存本地，因此注册中心挂掉后不影响运行中的客户端。

#### 负载均衡算法
- 加权随机
- 最小活跃数 请求开始活跃数增加，请求结束活跃数减少，高性能节点活跃数下降得快
- 一致性哈希 对比一般的哈希算法（哈希取模），一致性哈希避免了扩\缩容发生过多数据迁移的问题，但负载并不均匀（采用虚拟节点解决）
- 加权轮询

#### 容错
- Failover 自动切换并重试，成功后返回成功结果
- Failfast 快速失败，抛出异常
- Failsafe 失败安全，忽略异常，返回默认结果
- Failback 失败自动恢复，先返回默认结果，使用另外的线程定时重试
- Forking 并发调用，返回成功的
- Broadcast 逐个调用，任意一台报错则报错
- 其它略

#### RPC协议
默认的RPC协议为Dubbo，使用netty服务端和hessian2序列化。

其它协议还有gRPC、RMI等。

#### Dubbo vs. Spring Cloud OpenFeign
|Dubbo|Feign
|:---:|:---:
|RPC风格，耦合较高|RESTful
|还包括了负载均衡等|需要其它组件配合
|倾向于自定义协议，更容易获得高性能|HTTP

### Spring

#### IoC
控制反转将对象创建的控制反转了，由IoC容器控制对象的创建。

依赖注入（DI）是实现控制反转的一种方式。

#### 注解
- org.springframework.beans.factory.annotation.Autowired
- javax.annotation.Resource

#### Spring AOP vs. AspectJ
- Spring AOP基于代理运行时增强
- AspectJ基于字节码操作在编译前后或加载前增强

#### Spring Boot Starter自动装配
Spring Boot的SPI机制，通过扫描ClassLoader中Jar的META-INF/spring.factories元数据【org.springframework.boot.autoconfigure.EnableAutoConfiguration=xxx（@Configuration）】实现。

#### Spring Boot(Cloud)配置优先级
Spring Cloud有个bootstrap.yml配置，优先级如下（见演示项目 https://github.com/lvv9/spring-cloud-config）：
1. Spring Cloud Config Server中保存的应用配置（如config-client.yml）
2. Spring Boot中的application.yml
3. Spring Cloud Config Client（即Spring Boot应用）中的bootstrap.yml

#### Circular dependency
主要解决思路是提前暴露对象，因此如果相互依赖的Bean都是通过构造器注入时就无法解决，且目前只支持单例作用域的。

二三级缓存用于解决代理带来的问题，暴露早期代理对象，同时保证早期暴露的对象与最终的对象（一级缓存里的）一致：https://juejin.cn/post/6985337310472568839

否则会抛：
```text
org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'serviceA': Bean with name 'serviceA' has been injected into other beans [serviceB] in its raw version as part of a circular reference, but has eventually been wrapped. This means that said other beans do not use the final version of the bean. This is often the result of over-eager type matching - consider using 'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.
```

## Versioning

### 语义化版本
https://semver.org/lang/zh-CN/

版本格式：主版本号.次版本号.修订号，版本号递增规则如下：
1. 主版本号：当你做了不兼容的 API 修改，
2. 次版本号：当你做了向下兼容的功能性新增，
3. 修订号：当你做了向下兼容的问题修正。