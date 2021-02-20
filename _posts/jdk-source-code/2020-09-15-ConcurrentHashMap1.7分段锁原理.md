---
layout: post
category: JDK源码
---
## 1. 概述

`JDK1.7` 在生产环境已逐渐被 `JDK1.8` 替代，然而一些好的思想还是需要进行学习的。比方说位图中寻找 `bit` 位的思路是不是和 `ConcurrentHashMap1.7` 有点相似？

接下来，本文基于 `OpenJDK7` 来做源码解析

Hashmap多线程会导致HashMap的Entry链表形成环形数据结构，一旦形成环形数据结构，Entry的next节点永远不为空，就会产生死循环获取Entry。

HashTable使用synchronized来保证线程安全，但在线程竞争激烈的情况下HashTable的效率非常低下。因为当一个线程访问HashTable的同步方法，其他线程也访问HashTable的同步方法时，会进入阻塞或轮询状态。如线程1使用put进行元素添加，线程2不但不能使用put方法添加元素，也不能使用get方法来获取元素，所以竞争越激烈效率越低。

## 2.原理和实现

### 分段锁技术

HashTable容器在竞争激烈的并发环境下表现出效率低下的原因，是因为所有访问HashTable的线程都必须竞争同一把锁。

那假如容器里有多把锁，每一把锁用于锁容器其中一部分数据，那么当多线程访问容器里不同数据段的数据时，线程间就不会存在锁竞争，从而可以有效的提高并发访问效率，这就是ConcurrentHashMap所使用的锁分段技术，首先将数据分成一段一段的存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问。

另外，ConcurrentHashMap可以做到读取数据不加锁，并且其内部的结构可以让其在进行写操作的时候能够将锁的粒度保持地尽量地小，不用对整个ConcurrentHashMap加锁。

### ConcurrentHashMap的内部结构

ConcurrentHashMap是由Segment数组结构和HashEntry数组结构组成。

Segment是一种可重入锁ReentrantLock，在ConcurrentHashMap里扮演锁的角色，HashEntry则用于存储键值对数据。

一个ConcurrentHashMap里包含一个Segment数组，Segment的结构和HashMap类似，是一种数组和链表结构，

每个Segment守护着一个HashEntry数组里的元素，当对HashEntry数组的数据进行修改时，必须首先获得它对应的Segment锁。

结构图如下:

![image-20200831161306914](https://gitee.com/tostringcc/blog/raw/master/2020/image-20200831161306914.png)

从上面的结构我们可以了解到，ConcurrentHashMap定位一个元素的过程需要进行两次Hash操作，第一次Hash定位到Segment，第二次Hash定位到元素所在的链表的头部，因此，这一种结构的带来的副作用是Hash的过程要比普通的HashMap要长，但是带来的好处是写操作的时候可以只对元素所在的Segment进行加锁即可，不会影响到其他的Segment，这样，在最理想的情况下，ConcurrentHashMap可以最高同时支持Segment数量大小的写操作（刚好这些写操作都非常平均地分布在所有的Segment上），所以，通过这一种结构，ConcurrentHashMap的并发能力可以大大的提高。

### ConcurrentHashMap源码分析

#### ConcurrentHashMap的成员变量

```
//默认初始化容量，这个和 HashMap中的容量是一个概念，表示的是整个 Map的容量
static final int DEFAULT_INITIAL_CAPACITY = 16;

//默认加载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

//默认的并发级别，这个参数决定了 Segment 数组的长度
static final int DEFAULT_CONCURRENCY_LEVEL = 16;

//最大的容量
static final int MAXIMUM_CAPACITY = 1 << 30;

//每个Segment中table数组的最小长度为2，且必须是2的n次幂。
//由于每个Segment是懒加载的，用的时候才会初始化，因此为了避免使用时立即调整大小，设定了最小容量2
static final int MIN_SEGMENT_TABLE_CAPACITY = 2;

//用于限制Segment数量的最大值，必须是2的n次幂
static final int MAX_SEGMENTS = 1 << 16; // slightly conservative

//在size方法和containsValue方法，会优先采用乐观的方式不加锁，直到重试次数达到2，才会对所有Segment加锁
//这个值的设定，是为了避免无限次的重试。后边size方法会详讲怎么实现乐观机制的。
static final int RETRIES_BEFORE_LOCK = 2;

//segment掩码值，用于根据元素的hash值定位所在的 Segment 下标。后边会细讲
final int segmentMask;

//偏移量  和 segmentMask 配合使用来定位 Segment 的数组下标，后边讲。
final int segmentShift;

// Segment 组成的数组，每一个 Segment 都可以看做是一个特殊的 HashMap
final Segment<K,V>[] segments;
```

#### Segment对象

```
//Segment 对象，继承自 ReentrantLock 可重入锁。
//其内部的属性和方法和 HashMap 神似，只是多了一些拓展功能。
static final class Segment<K,V> extends ReentrantLock implements Serializable {
	
	//这是在 scanAndLockForPut 方法中用到的一个参数，用于计算最大重试次数
	//获取当前可用的处理器的数量，若大于1，则返回64，否则返回1。
	static final int MAX_SCAN_RETRIES =
		Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;

	//用于表示每个Segment中的 table，是一个用HashEntry组成的数组。
	transient volatile HashEntry<K,V>[] table;

	//Segment中的元素个数，每个Segment单独计数（下边的几个参数同样的都是单独计数）
	transient int count;

	//每次 table 结构修改时，如put，remove等，此变量都会自增
	transient int modCount;

	//当前Segment扩容的阈值，同HashMap计算方法一样也是容量乘以加载因子
	//需要知道的是，每个Segment都是单独处理扩容的，互相之间不会产生影响
	transient int threshold;

	//加载因子
	final float loadFactor;

	//Segment构造函数
	Segment(float lf, int threshold, HashEntry<K,V>[] tab) {
		this.loadFactor = lf;
		this.threshold = threshold;
		this.table = tab;
	}
	
	...
	// put(),remove(),rehash() 方法都在此类定义
}
```

count用来统计该段数据的个数，它是volatile变量，它用来协调修改和读取操作，以保证读取操作能够读取到几乎最新的修改。协调方式是这样的，每次修改操作做了结构上的改变，如增加/删除节点(修改节点的值不算结构上的改变)，都要写count值，每次读取操作开始都要读取count的值。这利用了 Java 5中对volatile语义的增强，对同一个volatile变量的写和读存在happens-before关系。

modCount统计段结构改变的次数，主要是为了检测对多个段进行遍历过程中某个段是否发生改变。

threashold用来表示需要进行rehash的界限值。

table数组存储段中节点，每个数组元素是个hash链，用HashEntry表示。table也是volatile，这使得能够读取到最新的 table值而不需要同步。loadFactor表示负载因子。

#### HashEntry

Segment中的元素是以HashEntry的形式存放在链表数组中的，看一下HashEntry的结构：

```
// HashEntry，存在于每个Segment中，它就类似于HashMap中的Node，用于存储键值对的具体数据和维护单向链表的关系
static final class HashEntry<K,V> {
	//每个key通过哈希运算后的结果，用的是 Wang/Jenkins hash 的变种算法，此处不细讲，感兴趣的可自行查阅相关资料
	final int hash;
	final K key;
	//value和next都用 volatile 修饰，用于保证内存可见性和禁止指令重排序
	volatile V value;
	//指向下一个节点
	volatile HashEntry<K,V> next;

	HashEntry(int hash, K key, V value, HashEntry<K,V> next) {
		this.hash = hash;
		this.key = key;
		this.value = value;
		this.next = next;
	}
}
```

可以看到HashEntry的一个特点，**除了value以外，其他的几个变量都是final的，这意味着不能从hash链的中间或尾部添加或删除节点，因为这需要修改next 引用值，所有的节点的修改只能从头部开始(头插法)**。

对于put操作，可以一律添加到Hash链的头部。

**但是对于remove操作，可能需要从中间删除一个节点，这就需要将要删除节点的前面所有节点整个复制一遍，最后一个节点指向要删除结点的下一个结点。**。为了确保读操作能够看到最新的值，将value设置成volatile，这避免了加锁。

#### ConcurrentHashMap的初始化

```
public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
	//检验参数是否合法。值得说的是，并发级别一定要大于0，否则就没办法实现分段锁了。
	if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
		throw new IllegalArgumentException();
	//并发级别不能超过最大值
	if (concurrencyLevel > MAX_SEGMENTS)
		concurrencyLevel = MAX_SEGMENTS;
	// Find power-of-two sizes best matching arguments
	//偏移量，是为了对hash值做位移操作，计算元素所在的Segment下标，put方法详讲
	int sshift = 0;
	//用于设定最终Segment数组的长度，必须是2的n次幂
	int ssize = 1;
	//这里就是计算 sshift 和 ssize 值的过程  (1) 
	while (ssize < concurrencyLevel) {
		++sshift;
		ssize <<= 1;
	}
	this.segmentShift = 32 - sshift;
	//Segment的掩码
	this.segmentMask = ssize - 1;
	if (initialCapacity > MAXIMUM_CAPACITY)
		initialCapacity = MAXIMUM_CAPACITY;
	//c用于辅助计算cap的值   (2)
	int c = initialCapacity / ssize;
	if (c * ssize < initialCapacity)
		++c;
	// cap 用于确定某个Segment的容量，即Segment中HashEntry数组的长度
	int cap = MIN_SEGMENT_TABLE_CAPACITY;
	//(3)
	while (cap < c)
		cap <<= 1;
	// create segments and segments[0]
	//这里用 loadFactor做为加载因子，cap乘以加载因子作为扩容阈值，创建长度为cap的HashEntry数组，
	//三个参数，创建一个Segment对象，保存到S0对象中。后边在 ensureSegment 方法会用到S0作为原型对象去创建对应的Segment。
	Segment<K,V> s0 =
		new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
						 (HashEntry<K,V>[])new HashEntry[cap]);
	//创建出长度为 ssize 的一个 Segment数组
	Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
	//把S0存到Segment数组中去。在这里，我们就可以发现，此时只是创建了一个Segment数组，
	//但是并没有把数组中的每个Segment对象创建出来，仅仅创建了一个Segment用来作为原型对象。
	UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
	this.segments = ss;
}
```

上边的注释中留了 (1)(2)(3) 三个地方还没有细说。我们现在假设一组数据，把涉及到的几个变量计算出来，就能明白这些参数的含义了。

```
//假设调用了默认构造，都用的是默认参数，即 initialCapacity 和 concurrencyLevel 都是16
//(1)  sshift 和 ssize 值的计算过程为，每次循环，都会把 sshift 自增1，并且 ssize 左移一位，即乘以2，
//直到 ssize 的值大于等于 concurrencyLevel 的值 16。
sshfit=0,1,2,3,4
ssize=1,2,4,8,16
//可以看到，初始他们的值分别是0和1，最终结果是4和16
//sshfit是为了辅助计算segmentShift值，ssize是为了确定Segment数组长度。
//(2)  此时,计算c的值，
c = 16/16 = 1;
//判断 c * 16 < 16 是否为真，真的话 c 自增1，此处为false，因此 c的值为1不变。
//(3)  此时，由于c为1， cap为2 ，因此判断 cap < c 为false，最终cap为2。
//总结一下，以上三个步骤，最终都是为了确定以下几个关键参数的值，
//确定 segmentShift ，这个用于后边计算hash值的偏移量，此处即为 32-4=28，
//确定 ssize，必须是一个大于等于 concurrencyLevel 的一个2的n次幂值
//确定 cap，必须是一个大于等于2的一个2的n次幂值
//感兴趣的小伙伴，还可以用另外几组参数来计算上边的参数值，可以加深理解参数的含义。
//例如initialCapacity和concurrencyLevel分别传入10和5，或者传入33和16
```

CurrentHashMap的初始化一共有三个参数:

1. 一个initialCapacity，表示初始的容量，
2. 一个loadFactor，表示负载参数，
3. 最后一个是concurrentLevel，代表ConcurrentHashMap内部的Segment的数量，ConcurrentLevel一经指定，不可改变，后续如果ConcurrentHashMap的元素数量增加导致ConrruentHashMap需要扩容，ConcurrentHashMap不会增加Segment的数量，而只会增加Segment中链表数组的容量大小，这样的好处是扩容过程不需要对整个ConcurrentHashMap做rehash，而只需要对Segment里面的元素做一次rehash就可以了。

整个ConcurrentHashMap的初始化方法还是非常简单的，先是根据concurrentLevel来new出Segment，这里Segment的数量是不大于concurrentLevel的最大的2的指数，就是说Segment的数量永远是2的指数个，这样的好处是方便采用移位操作来进行hash，加快hash的过程。

接下来就是根据intialCapacity确定Segment的容量的大小，每一个Segment的容量大小也是2的指数，同样使为了加快hash的过程。

**这边需要特别注意一下两个变量，分别是segmentShift和segmentMask，这两个变量在后面将会起到很大的作用，假设构造函数确定了Segment的数量是2的n次方，那么segmentShift就等于32减去n，而segmentMask就等于2的n次方减一。**

当用 new ConcurrentHashMap() 无参构造函数进行初始化的，那么初始化完成后：

1. Segment 数组长度为 16，不可以扩容
2. Segment[i] 的默认大小为 2，负载因子是 0.75，得出初始阈值为 1.5，也就是以后插入第一个元素不会触发扩容，插入第二个会进行第一次扩容
3. 这里初始化了 segment[0]，其他位置还是 null
4. 当前 segmentShift 的值为 32 – 4 = 28，segmentMask 为 16 – 1 = 15，姑且把它们简单 翻译 为移位数和掩码，这两个值马上就会用到

#### hash()方法

```
 private int hash(Object k) {
        int h = hashSeed;
        //如果Key是字符串类型，则使用专门为字符串设计的Hash方法，否则使用一连串的异或操作增加hash随机性
        if ((0 != h) && (k instanceof String)) {
            return sun.misc.Hashing.stringHash32((String) k);
        }

        h ^= k.hashCode();

        // Spread bits to regularize both segment and index locations,
        // using variant of single-word Wang/Jenkins hash.
        h += (h <<  15) ^ 0xffffcd7d;
        h ^= (h >>> 10);
        h += (h <<   3);
        h ^= (h >>>  6);
        h += (h <<   2) + (h << 14);
        return h ^ (h >>> 16);
    }
```

#### 初始化Segment

ConcurrentHashMap 初始化的时候会初始化第一个槽 segment[0]，对于其他槽来说，在插入第一个值的时候进行初始化。

这里需要考虑并发，因为很可能会有多个线程同时进来初始化同一个槽 segment[k]，不过只要有一个成功了就可以。

```
private Segment<K,V> ensureSegment(int k) {
    final Segment<K,V>[] ss = this.segments;
    long u = (k << SSHIFT) + SBASE; // raw offset
    Segment<K,V> seg;
    if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
        // 这里看到为什么之前要初始化 segment[0] 了，
        // 使用当前 segment[0] 处的数组长度和负载因子来初始化 segment[k]
        // 为什么要用“当前”，因为 segment[0] 可能早就扩容过了
        Segment<K,V> proto = ss[0]; // use segment 0 as prototype
        int cap = proto.table.length;
        float lf = proto.loadFactor;
        int threshold = (int)(cap * lf);
        // 初始化 segment[k] 内部的数组
        HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
            == null) { // recheck Segment[k] 是否被其它线程初始化了
            Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
            // 使用 while 循环，内部用 CAS，当前线程成功设值或其他线程成功设值后，退出
            while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                   == null) {
                if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                    break;
            }
        }
    }
    return seg;
}
```

#### put过程分析

put 方法的总体流程是，

1. 通过哈希算法计算出当前 key 的 hash 值
2. 通过这个 hash 值找到它所对应的 Segment 数组的下标
3. 再通过 hash 值计算出它在对应 Segment 的 HashEntry数组 的下标
4. 找到合适的位置插入元素



场景：线程A和线程B同时执行相同Segment对象的put方法

```
1. 线程A执行tryLock()方法成功获取锁，则把HashEntry对象插入到相应的位置；
2. 线程B获取锁失败，则执行scanAndLockForPut()方法，在scanAndLockForPut方法中，会通过重复执行tryLock()方法尝试获取锁，在多处理器环境下，重复次数为64，单处理器重复次数为1，当执行tryLock()方法的次数超过上限时，则执行lock()方法挂起线程B；
3. 当线程A执行完插入操作时，会通过unlock()方法释放锁，接着唤醒线程B继续执行；
 
```

put 方法的过程:

1. 判断value是否为null，如果为null，直接抛出异常。注：不允许key或者value为null
2. 通过哈希算法定位到Segment（key通过一次hash运算得到一个hash值，将得到hash值向右按位移动segmentShift位，然后再与segmentMask做&运算得到segment的索引j）。
3. 使用Unsafe的方式从Segment数组中获取该索引对应的Segment对象
4. 向这个Segment对象中put值

注：对共享变量进行写入操作为了线程安全，在操作共享变量时必须得加锁，持有段锁(锁定整个segment)的情况下执行的。修改数据是不能并发进行的

判断该值的插入是否会导致该 segment 的元素个数超过阈值，以确保容量不足时能够rehash扩容，再插值。

注：rehash 扩容 segment 数组不能扩容，扩容的是 segment 数组某个位置内部的数组 HashEntry[] 扩容为原来的 2 倍。先进行扩容，再插值

查找是否存在同样一个key的结点，存在直接替换这个结点的值。否则创建一个新的结点并添加到hash链的头部，修改modCount和count的值，修改count的值一定要放在最后一步。

```
//这是Map的put方法
public V put(K key, V value) {
	Segment<K,V> s;
	//不支持value为空
	if (value == null)
		throw new NullPointerException();
	//通过 Wang/Jenkins 算法的一个变种算法，计算出当前key对应的hash值
	int hash = hash(key);
	//上边我们计算出的 segmentShift为28，因此hash值右移28位，说明此时用的是hash的高4位，
	//然后把它和掩码15进行与运算，得到的值一定是一个 0000 ~ 1111 范围内的值，即 0~15 。
	int j = (hash >>> segmentShift) & segmentMask;
	//这里是用Unsafe类的原子操作找到Segment数组中j下标的 Segment 对象
	if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
		 (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
		//初始化j下标的Segment
		s = ensureSegment(j);
	//在此Segment中添加元素
	return s.put(key, hash, value, false);
}
```

上边有一个这样的方法， UNSAFE.getObject (segments, (j << SSHIFT) + SBASE。它是为了通过Unsafe这个类，找到 j 最新的实际值。这个计算 (j << SSHIFT) + SBASE ，在后边非常常见，我们只需要知道它代表的是 j 的一个偏移量，通过偏移量，就可以得到 j 的实际值。可以类比，AQS 中的 CAS 操作。 Unsafe中的操作，都需要一个偏移量，看下图，

![image-20200903163903938](https://gitee.com/tostringcc/blog/raw/master/2020/image-20200903163903938.png)

(j << SSHIFT) + SBASE 就相当于图中的 stateOffset偏移量。只不过图中是 CAS 设置新值，而我们这里是取 j 的最新值。 后边很多这样的计算方式，就不赘述了。接着看 s.put 方法，这才是最终确定元素位置的方法。

Segment 内部是由 数组+链表 组成的。

```
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // 先获取该 segment 的独占锁
    // 每一个Segment进行put时，都会加锁
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        // segment 内部的数组
        HashEntry<K,V>[] tab = table;
        // 利用 hash 值，求应该放置的数组下标
        int index = (tab.length - 1) & hash;
        // 数组该位置处的链表的表头
        HashEntry<K,V> first = entryAt(tab, index);
        for (HashEntry<K,V> e = first;;) {
            // 如果链头不为 null
            if (e != null) {
                K k;
                //如果在该链中找到相同的key，则用新值替换旧值，并退出循环
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                //如果没有和key相同的，一直遍历到链尾，链尾的next为null，进入到else
                e = e.next;
            }
            else {
                // node 到底是不是 null，这个要看获取锁的过程，不过和这里都没有关系。
                // 如果不为 null，那就直接将它设置为链表表头；如果是null，初始化并设置为链表表头。
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                // 如果超过了该 segment 的阈值，这个 segment 需要扩容
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);
                else
                    // 没有达到阈值，将 node 放到数组 tab 的 index 位置，
                    // 其实就是将新的节点设置成原链表的表头
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        // 解锁
        unlock();
    }
    return oldValue;
}
```

#### get()方法

1. 计算 hash 值，找到 segment 数组中的具体位置,使用Unsafe获取对应的Segment
2. 根据 hash 找到数组中具体的位置
3. 从链表头开始遍历整个链表（因为Hash可能会有碰撞，所以用一个链表保存），如果找到对应的key，则返回对应的value值，否则返回null。

注：get操作不需要锁，由于其中涉及到的共享变量都使用volatile修饰，volatile可以保证内存可见性，所以不会读取到过期数据。

```
public V get(Object key) {
    Segment<K,V> s; // manually integrate access methods to reduce overhead
    HashEntry<K,V>[] tab;
    int h = hash(key);
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                 (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```

#### remove操作

Remove操作的前面一部分和前面的get和put操作一样，都是定位Segment的过程，然后再调用Segment的remove方法：

```
final V remove(Object key, int hash, Object value) {
    if (!tryLock())
        scanAndLock(key, hash);
    V oldValue = null;
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;
        HashEntry<K,V> e = entryAt(tab, index);
        HashEntry<K,V> pred = null;
        while (e != null) {
            K k;
            HashEntry<K,V> next = e.next;
            if ((k = e.key) == key || (e.hash == hash && key.equals(k))) {
                V v = e.value;
                if (value == null || value == v || value.equals(v)) {
                    if (pred == null)
                        setEntryAt(tab, index, next);
                    else
                        pred.setNext(next);
                    ++modCount;
                    --count;
                    oldValue = v;
                }
                break;
            }
            pred = e;
            e = next;
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```

首先remove操作也是确定需要删除的元素的位置，不过这里删除元素的方法不是简单地把待删除元素的前面的一个元素的next指向后面一个就完事了，前面已经说过HashEntry中的next是final的，一经赋值以后就不可修改，在定位到待删除元素的位置以后，程序就将待删除元素前面的那一些元素全部复制一遍，然后再一个一个重新接到链表上去，看一下下面这一幅图来了解这个过程：

![image-20201108210008547](https://gitee.com/tostringcc/blog/raw/master/2020/image-20201108210008547.png)

假设链表中原来的元素如上图所示，现在要删除元素3，那么删除元素3以后的链表就如下图所示：

![image-20201108210016985](https://gitee.com/tostringcc/blog/raw/master/2020/image-20201108210016985.png)

注意：图1和2的元素顺序相反了，为什么这样，不防再仔细看看源码或者再读一遍上面remove的分析过程，元素复制是从待删除元素位置起将前面的元素逐一复制的，然后再将后面的链接起来。

#### size 操作

size操作需要遍历所有的Segment才能算出整个Map的大小。先采用不加锁的方式，循环所有的Segment（通过Unsafe的getObjectVolatile()以保证原子读语义）连续计算元素的个数，最多计算3次：

1. 如果前后两次计算结果相同，则说明计算出来的元素个数是准确的；
2. 如果前后两次计算结果都不同，则给每个Segment进行加锁，再计算一次元素的个数；

注：在put,remove和clean方法里操作元素前都会将变量modCount进行加1，那么在统计size前后比较modCount是否发生变化，从而得知容器的大小是否发生变化。

## 参考：

[J.U.C 之ConcurrentHashMap(JDK1.7)](https://juejin.im/post/6844903953830772750##heading-1)

[图解ConcurrentHashMap](https://juejin.im/post/6844903520957644808)

[嘿嘿，我就知道面试官接下来要问我 ConcurrentHashMap 底层原理了，看我怎么秀他](https://www.lagou.com/lgeduarticle/119948.html)