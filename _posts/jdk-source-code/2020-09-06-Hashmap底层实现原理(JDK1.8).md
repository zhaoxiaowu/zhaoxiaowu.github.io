---
layout: post
category: JDK源码
---
JDK1.8 中对 Hashmap 做了以下改动。

- 引入红黑树，优化数据结构
- 将链表头插法改为尾插法，解决 1.7 中多线程循环链表的 bug
- 优化 hash 算法
- resize 计算索引位置的算法改进
- 先插入后扩容

### 底层数据结构

在JDK 1.8中引入了红黑树，在链表的长度大于等于8并且hash桶的长度大于等于64的时候，会将链表进行树化。这里的树使用的数据结构是红黑树，红黑树是一个自平衡的二叉查找树，查找效率会从链表的o(n)降低为o(logn)，效率是非常大的提高。

**为什么不全部转化为二叉树？**

第一个是链表的结构比红黑树简单，所以在链表的节点不多的情况下，从整体的性能看来， 数组+链表+红黑树的结构不一定比数组+链表的结构性能高。

第二个是HashMap频繁的resize（扩容），扩容的时候需要重新计算节点的索引位置，也就是会将红黑树进行拆分和重组其实 这是很复杂的，这里涉及到红黑树的着色和旋转。



（1）允许NULL值，NULL键

（2）不要轻易改变负载因子，负载因子过高会导致链表过长，查找键值对时间复杂度就会增高，负载因子过低会导致hash桶的 数量过多，空间复杂度会增高

（3）Hash表每次会扩容长度为以前的2倍

（4）HashMap是多线程不安全的，我在JDK1.7进行多线程put操作，之后遍历，直接死循环，CPU飙到100%，在JDK 1.8中进行多线程操作会出现节点和value值丢失，为什么JDK1.7与JDK1.8多线程操作会出现很大不同，是因为JDK 1.8的作者对resize方法进行了优化不会产生链表闭环。这也是本章的重点之一，具体的细节大家可以去查阅资料。这里我就不解释太多了

（5）尽量设置HashMap的初始容量，尤其在数据量大的时候，防止多次resize

### 核心源码

```
  //默认hash桶初始长度16
  static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; 

  //hash表最大容量2的30次幂
  static final int MAXIMUM_CAPACITY = 1 << 30;

  //默认负载因子 0.75
  static final float DEFAULT_LOAD_FACTOR = 0.75f;

  //链表的数量大于等于8个并且桶的数量大于等于64时链表树化 
  static final int TREEIFY_THRESHOLD = 8;

  //hash表某个节点链表的数量小于等于6时树拆分
  static final int UNTREEIFY_THRESHOLD = 6;

  //树化时最小桶的数量
  static final int MIN_TREEIFY_CAPACITY = 64;
    //hash桶
  transient Node<K,V>[] table;                         

  //键值对的数量
  transient int size;

  //HashMap结构修改的次数
  transient int modCount;

  //扩容的阀值，当键值对的数量超过这个阀值会产生扩容
  int threshold;

  //负载因子
  final float loadFactor;

```



### Hashmap 中 put()过程

JDK1.8 中，Hashmap 将基本元素由 Entry 换成了 Node，不过查看源码后发现换汤不换药，这里没啥好说的。

![image-20200820020719939](https://gitee.com/tostringcc/blog/raw/master/2020/image-20200820020719939.png)

**put的源码**

```
public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        // 判断数组是否为空，长度是否为0，是则进行扩容数组初始化
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        // 通过hash算法找到数组下标得到数组元素，为空则新建
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            // 找到数组元素，hash相等同时key相等，则直接覆盖
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            // 该数组元素在链表长度>8后形成红黑树结构的对象,p为树结构已存在的对象
            elseif (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                // 该数组元素hash相等，key不等，同时链表长度<8.进行遍历寻找元素，有就覆盖无则新建
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        // 新建链表中数据元素，尾插法
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            // 链表长度>=8 结构转为 红黑树
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            // 新值覆盖旧值
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                // onlyIfAbsent默认false
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        // 判断是否需要扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        returnnull;
    }
```

1. 基本过程如下:

   1. 检查数组是否为空，执行 resize()扩充；在实例化 HashMap 时，并不会进行初始化数组）

   2. 通过 hash 值计算数组索引，获取该索引位的首节点。

   3. 如果首节点为 null（没发生碰撞），则创建新的数组元素，直接添加节点到该索引位(bucket)。

   4. 如果首节点不为 null（发生碰撞），那么有 3 种情况

      ① key 和首节点的 key 相同，覆盖 old value（保证 key 的唯一性）；否则执行 ② 或 ③

      ② 如果首节点是红黑树节点（TreeNode），将键值对添加到红黑树。

      ③ 如果首节点是链表，进行遍历寻找元素，有就覆盖无则新建，将键值对添加到链表。添加之后会判断链表长度是否到达 TREEIFY_THRESHOLD - 1 这个阈值，“尝试”将链表转换成红黑树。

   5. 最后判断当前元素个数是否大于 threshold，扩充数组。

### Hashmap 中 get()过程

```
public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            // 永远检查第一个node
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)  // 树查找
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&   // 遍历链表
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        returnnull;
    }
```

在 Hashmap1.8 中，无论是存元素还是取元素，都是优先判断 bucket 上第一个元素是否匹配，而在 1.7 中则是直接遍历查找。

基本过程如下:

1. 根据 key 计算 hash;
2. 检查数组是否为空，为空返回 null;
3. 根据 hash 计算 bucket 位置，如果 bucket 第一个元素是目标元素，直接返回。否则执行 4;
4. 如果 bucket 上元素大于 1 并且是树结构，则执行树查找。否则执行 5;
5. 如果是链表结构，则遍历寻找目标

### Hashmap 中 resize()过程

```
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            // 如果已达到最大容量不在扩容
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            // 通过位运算扩容到原来的两倍
            elseif ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        elseif (oldThr > 0) // initial capacity was placed in threshold
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
        // 新的扩容临界值
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    // 如果该位置元素没有next节点，将该元素放入新数组
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    elseif (e instanceof TreeNode)
                        // 树节点
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        // 链表节点。

                        // lo串的新索引位置与原先相同
                        Node<K,V> loHead = null, loTail = null;
                        // hi串的新索引位置为[原先位置j+oldCap]
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            // 原索引，oldCap是2的n次方，二进制表示只有一个1，其余是0
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    // 尾插法
                                    loTail.next = e;
                                loTail = e;
                            }
                            // 原索引+oldCap
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        // 根据hash判断该bucket上的整个链表的index还是旧数组的index，还是index+oldCap
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

JDK1.8 版本中扩容相对复杂。在 1.7 版本中，重新根据 hash 计算索引位置即可；而在 1.8 版本中分 2 种情况，下边用图例来解释。



HaspMap扩容就是就是先计算 新的hash表容量和新的容量阀值，然后初始化一个新的hash表，将旧的键值对重新映射在新的hash表里。这里实现的细节当然 没有我说的那么简单，如果在旧的hash表里涉及到红黑树，那么在映射到新的hash表中还涉及到红黑树的拆分。

在扩容的源代码中作者有一个使用很巧妙的地方，是键值对分布更均匀，不知道读者是否有看出来。在遍历原hash桶时的 一个链表时，因为扩容后长度为原hash表的2倍，假设把扩容后的hash表分为两半，分为低位和高位，如果能把原链表的键值对， 一半放在低位，一半放在高位，这样的索引效率是最高的。那看看源码里是怎样写的。大师通过e.hash & oldCap == 0来判断， 这和e.hash & (oldCap - 1) 有什么区别呢。下面我通过画图来解释一下。

因为n是2的整次幂，二进制表示除了最高位为1外，其他低位全为0，那么e.hash & oldCap 是否等于0,取决于n对应最高位 相对于e.hash那一位是0还是1，比如说n = 16，二进制为10000，第5位为1，e.hash & oldCap 是否等于0就取决于e.hash第5 位是0还是1，这就相当于有50%的概率放在新hash表低位，50%的概率放在新hash表高位。大家应该明白了e.hash & oldCap == 0的好处与作用了吧。

![image-20200820022614984](https://gitee.com/tostringcc/blog/raw/master/2020/image-20200820022614984.png)

![image-20200820022623550](https://gitee.com/tostringcc/blog/raw/master/2020/image-20200820022623550.png)

### **常见问题**

**为什么要计算hash值，而不用hashCode？**

用为通常n是很小的，而hashCode是32位，如果（n - 1）& hashCode那么当n大于2的16次方加1，也就是65537后(n - 1)的高位数据才能与hashCode的高位数据相与，当n很小是只能使用上hashCode低 16位的数据，这会产生一个问题，既键值对在hash桶中分布不均匀，导致链表过长，而把hashCode>>>16无符号右移16位让 高16位间接的与（n - 1）参加计算，从而让键值对分布均匀。降低hash碰撞。



### 参考

[Java面试必问之Hashmap底层实现原理(JDK1.8)](https://mp.weixin.qq.com/s/ugBm-koApBRepbSQ2kiV2A)

[HashMap源码分析（JDK 1.8）](https://juejin.im/post/6844903559675248647)

[源码解析：HashMap 1.8](https://juejin.im/post/6844904015088599054#heading-32)

[Java源码分析：关于 HashMap 1.8 的重大更新](https://blog.csdn.net/carson_ho/article/details/79373134)