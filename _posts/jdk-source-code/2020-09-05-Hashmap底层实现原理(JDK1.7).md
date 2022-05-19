---
layout: post
category: JDK源码
---
Java集合三大体系——List、Set、Map，而Set是基于Map实现的.在Map中HashMap作为其中常用类

HashMap是链表散列的数据结构，其容量是16，负载因子0.75，当大于容量*负载因子会进行2倍扩容，put操作是将key的hashcode值进行一次hash计算，key的equals方法找到键值对进行替换返回被旧数据，若没有找到会插入到链表中

HashMap线程不安全.当面试官听到这些以后第一个问题为什么容量是16，15、14不行吗？为什么2倍扩容？为什么HashMap建议不可变对象用Key？自己当时思考得不够深入，还没问到ConcurrentHashMap我就已经心慌了...下面我来聊一聊我对HashMap的看法

### Map接口

**HashMap**: key不可重复，也是无序的 可以为null

**LinkedHashMap**: 这是一个「HashMap + 双向链表」的结构，落脚点是 HashMap。所以既拥有 HashMap 的所有特性还能有顺序。

**TreeMap**: 是有序的，本质是用二叉搜索树来实现的。

### HashMap简介

先看下HashMap的继承关系:

![image-20200820001319340](https://raw.githubusercontent.com/zhaoxiaowu/blog/master/2020/image-20200820001319340.png)

### HashMap源码

**关键代码**

```
    /**
     * 默认初始容量16——必须是2的幂
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; 
    
    /**
     * HashMap存储的键值对数量
     */
    transient int size;
    
    /**
     * 默认负载因子0.75
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    
    /**
     * 扩容阈值，当size大于等于其值，会执行resize操作
     * 一般情况下threshold=capacity*loadFactor
     */
    int threshold;
    
    /**
     * Entry数组
     */
    transient Entry[] table = (Entry[]) EMPTY_TABLE;
    
    /**
     * 记录HashMap修改次数，fail-fast机制
     */
    transient int modCount;
    
    /**
     * hashSeed用于计算key的hash值，它与key的hashCode进行按位异或运算
     * hashSeed是一个与实例相关的随机值，用于解决hash冲突
     * 如果为0则禁用备用哈希算法
     */
     transient int hashSeed = 0;
```

**构造函数**

```
   /**
     * 指定容量及负载因子构造方法
     */
    public HashMap(int initialCapacity, float loadFactor) {
        //校验初始容量
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity:" +
                                               initialCapacity);
        //当初始容量超过最大容量，初始容量为最大容量
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        //校验初始负载因子    
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        //设置负载因子
        this.loadFactor = loadFactor;
        //设置扩容阈值
        threshold = initialCapacity;
        //空方法，让其子类重写例如LinkedHashMap
        init();
    }
    
    /**
     * 默认构造方法，采用默认容量16，默认负载因子0.75
     */
    public HashMap() {
        this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);
    }
    
    /**
     * 指定容量构造方法，负载因子默认0.75
     */
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

```

由以上源码可知，Hashmap 的初始容量默认是 16, 底层存储结构是数组(到这里只能看出是数组, 其实还有链表，下边看源码解释)。基本存储单元是 Entry，那 Entry 是什么呢?我们接着看 Entry 相关源码，

```
    static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        Entry<K,V> next;    // 链表后置节点
        final int hash;

        /**
         * Creates new entry.
         */
        Entry(int h, K k, V v, Entry<K,V> n) {
            value = v;
            next = n;   // 头插法: newEntry.next=e
            key = k;
            hash = h;
        }
        ...
    }
```

由 Entry 源码可知，Entry 是链表结构。综上所述，可以得出:**Hashmap 底层是基于数组和链表实现的**

为什么采用这种结构来存储元素呢？

**数组的特点：查询效率高，插入，删除效率低**。

**链表的特点：查询效率低，插入删除效率高**。



### Hashmap 中 put()过程

![image-20200819232123563](https://raw.githubusercontent.com/zhaoxiaowu/blog/master/2020/image-20200819232123563.png)y

**源码**

```
public V put(K key, V value) {
        if (key == null)
            return putForNullKey(value);
        // 根据key计算hash
        int hash = hash(key.hashCode());
        // 计算元素在数组中的位置
        int i = indexFor(hash, table.length);
        // 遍历链表，如果相同覆盖
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                //空方法，让其子类重写例如LinkedHashMap
                e.recordAccess(this);
                 //返回旧值
                return oldValue;
            }
        }
		//记录修改		
        modCount++;
        // 头插法插入元素
        addEntry(hash, key, value, i);
        return null;
    }
```

**上图中多次提到头插法，啥是 `头插法` 呢？接下来看 `addEntry` 方法**

```
    void addEntry(int hash, K key, V value, int bucketIndex) {
        //当前hashmap中的键值对数量超过扩容阈值，进行2倍扩容
        if ((size >= threshold) && (null != table[bucketIndex])) {
            //2倍扩容
            resize(2 * table.length);
            //扩容后，桶的数量增加了，重新对键进行哈希码的计算
            hash = (null != key) ? hash(key) : 0;
            //根据键的新哈希码和新的桶数量重新计算桶索引值
            bucketIndex = indexFor(hash, table.length);
        }
        //创建结点
        createEntry(hash, key, value, bucketIndex);
    }
    
    /**
     * 头插结点
     * 将原本在数组中存放的链表头置入到新的Entry之后，将新的Entry放入数组中
     */
    void createEntry(int hash, K key, V value, int bucketIndex) {
        Entry e = table[bucketIndex]; //当两个线程同时执行到这里  一个线程的赋值会被另一个覆盖掉  这是对象丢失的原因
        table[bucketIndex] = new Entry<>(hash, key, value, e);
        size++;
    }
```

结合 Entry 类的构造方法，每次插入新元素的时候，将 bucket 原链表取出，新元素的 next 指向原链表,这就是 `头插法` 。为了更加清晰的表示 Hashmap 存储结构，再绘制一张存储结构图。

![image-20200819233009617](https://raw.githubusercontent.com/zhaoxiaowu/blog/master/2020/image-20200819233009617.png)

### Hashmap 中 get()过程

![image-20200819233040546](https://raw.githubusercontent.com/zhaoxiaowu/blog/master/2020/image-20200819233040546.png)

**get()源码**

    public V get(Object key) {
        *// 获取key为null的值*
        if (key == null)
            return getForNullKey();
        *// 根据key获取hash*
        int hash = hash(key.hashCode());
        *// 遍历链表，直到找到元素*
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k)))
                return e.value;
        }
        return null;
    }
###  Hashmap的hash和indexFor

```

    final int hash(Object k) {
        // 当h不为0且键对象类型为String用此算法，1.8已删除
        int h = hashSeed;
        if (0 != h && k instanceof String) {
            return sun.misc.Hashing.stringHash32((String) k);
        }
        h ^= k.hashCode();
        //此函数确保在每个比特位置上仅以恒定倍数不同的hashCode具有有限的碰撞数量（在默认负载因子下约为8）
        h ^= (h >>> 20) ^ (h >>> 12);
        return h ^ (h >>> 7) ^ (h >>> 4);
    }  
```

根据所计算的值与数组长度计算桶位置:

    static int indexFor(int h, int length) {
        *// 根据hash与数组长度mod运算*
        return h & (length-1);
    }

由源码可知, jdk 根据 key 的 hash 值和数组长度做 mod 运算，这里用位运算代替 mod。

那么既然是取模运算为什么不直接h%length，因为其效率很低，所以采用位运算

hash 运算值是一个 int 整形值，在 java 中 int 占 4 个字节，32 位，下边通过图示来说明位运算。

![image-20200819235609427](https://raw.githubusercontent.com/zhaoxiaowu/blog/master/2020/image-20200819235609427.png)

length为16时在[0,15]区间内冲突为0，且雨露均沾分布均匀每个桶都可能会存放数据，而为15，14时不仅有冲突而且有些空间永远不会存放数据这就导致了资源浪费，并且散列就不会出现下标越界得到一个异常.

**为什么容量要为2的幂次？**

因为HashMap在算桶index时根据key的hashcode值进行hash计算获取hash值与数组length-1进行与运算，length-1的二进制位全为1，这样可以分布均匀避免冲突，所以HashMap容量要为2的幂数

**为什么不推荐用可变对象？**

若key的hash和传入key的hash相同且key的equals放回true，那么直接覆盖value.key的hash值是根据其hashcode值进行hash哈希计算得到的，那么当我们用可变对象时其hashcode值很容易会变化，那么就会带来风险找不到原来的value，所以HashMap建议使用不可变对象作为Key

### Hashmap 中 resize()过程

只要是新插入元素，即执行 addEntry()方法，在插入完成后，都会判断是否需要扩容。从 addEntry()方法可知，扩容后的容量为原来的 2 倍。

```
void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }
        // 新建数组
        Entry[] newTable = new Entry[newCapacity];
        // 数据迁移
        transfer(newTable);
        // table指向新的数组
        table = newTable;
        // 新的扩容临界值
        threshold = (int)(newCapacity * loadFactor);
    }
```

transfer方法遍历旧数组所有Entry，根据新的容量逐个重新计算索引头插保存在新数组中，扩容相当麻烦，所以如果当我们知道需要添加多少数据时最好指定容量初始化.

    /**
     * 将旧Entry数组转移到新Entry数组中去
     */
    void transfer(Entry[] newTable, boolean rehash) {
        //获取新数组的长度
        int newCapacity = newTable.length;
        for (Entry e : table) {
            while(null != e) {
                Entry next = e.next;
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                //重新计算索引
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            }
        }
    }  
transfer迁移 当数据非常大的时候会非常消耗资源，迁移过程中 其他线程新增的元素可能落在已经遍历过的哈希槽上 导致数据丢失

如果多个线程同时执行resize,每个线程都会new Entry[] 导致新表被覆盖、

### Hashmap 扩容安全问题

**高并发环境下对象会丢失的原因**

- 新增时并发复制被覆盖
- 已遍历区间 新增元素丢失
- 新表被覆盖



**多线程扩容有可能会形成环形链表**

正常扩容后：

1. 原来在oldTable[i]位置的元素，会被放到newTable[i]或者newTable[i+oldTable.length]的位置
2. 链表在复制的时候会反转

### 参考

[Java集合——HashMap（jdk1.7）](https://juejin.im/post/6844903589236703239)

[Java面试必问之Hashmap底层实现原理(JDK1.7)](https://mp.weixin.qq.com/s/ugBm-koApBRepbSQ2kiV2A)

[面试官：HashMap死循环形成的原因是什么？](https://juejin.im/post/6844904084173144071)