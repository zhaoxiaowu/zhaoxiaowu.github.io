---
layout: post
category: JDK源码
---
众所周知 [HashMap](https://github.com/crossoverJie/Java-Interview/blob/master/MD/HashMap.md) 是一个无序的 `Map`，因为每次根据 `key` 的 `hashcode` 映射到 `Entry` 数组上，所以遍历出来的顺序并不是写入的顺序。

因此 JDK 推出一个基于 `HashMap` 但具有顺序的 `LinkedHashMap` 来解决有排序需求的场景。

它的底层是继承于 `HashMap` 实现的，由一个双向链表所构成

`LinkedHashMap` 的排序方式有两种：

- 根据写入顺序排序。
- 根据访问顺序排序。

其中根据访问顺序排序时，每次 `get` 都会将访问的值移动到链表末尾，这样重复操作就能的到一个按照访问顺序排序的链表。

## 构造方法

LinkedHashMap提供了多个构造方法，我们先看空参的构造方法。

```java
    public LinkedHashMap() {
        // 调用HashMap的构造方法，其实就是初始化Entry[] table
        super();
        // 这里是指是否基于访问排序，默认为false
        accessOrder = false;
    }
```

首先使用super调用了父类HashMap的构造方法，其实就是根据初始容量、负载因子去初始化Entry[] table



然后把accessOrder设置为false，这就跟存储的顺序有关了，LinkedHashMap存储数据是有序的，而且分为两种：插入顺序和访问顺序。

这里accessOrder设置为false，表示不是访问顺序而是插入顺序存储的，这也是默认值，表示LinkedHashMap中存储的顺序是按照调用put方法插入的顺序进行排序的。



HashMap的构造函数中，调用了init方法，而在HashMap中init方法是空实现，但LinkedHashMap重写了该方法，所以在LinkedHashMap的构造方法里，调用了自身的init方法，init的重写实现如下：



```dart
    /**
     * Called by superclass constructors and pseudoconstructors (clone,
     * readObject) before any entries are inserted into the map.  Initializes
     * the chain.
     */
    @Override
    void init() {
        // 创建了一个hash=-1，key、value、next都为null的Entry
        header = new Entry<>(-1, null, null, null);
        // 让创建的Entry的before和after都指向自身，注意after不是之前提到的next
        // 其实就是创建了一个只有头部节点的双向链表
        header.before = header.after = header;
    }
```

这好像跟我们上一篇HashMap提到的Entry有些不一样，HashMap中静态内部类Entry是这样定义的：



```dart
    static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        Entry<K,V> next;
        int hash;
```

没有before和after属性啊！原来，LinkedHashMap有自己的静态内部类Entry，它继承了HashMap.Entry，定义如下:



```cpp
    /**
     * LinkedHashMap entry.
     */
    private static class Entry<K,V> extends HashMap.Entry<K,V> {
        // These fields comprise the doubly linked list used for iteration.
        Entry<K,V> before, after;

        Entry(int hash, K key, V value, HashMap.Entry<K,V> next) {
            super(hash, key, value, next);
        }
```

所以LinkedHashMap构造函数，主要就是调用HashMap构造函数初始化了一个Entry[] table，然后调用自身的init初始化了一个只有头结点的双向链表。完成了如下操作：

![image-20200821051341176](https://raw.githubusercontent.com/zhaoxiaowu/blog/master/2020/image-20200821051341176.png)

## put方法

LinkedHashMap没有重写put方法，所以还是调用HashMap得到put方法

我们看看LinkedHashMap的addEntry方法：

```csharp
    void addEntry(int hash, K key, V value, int bucketIndex) {
        // 调用父类的addEntry，增加一个Entry到HashMap中
        super.addEntry(hash, key, value, bucketIndex);

        // removeEldestEntry方法默认返回false，不用考虑
        Entry<K,V> eldest = header.after;
        if (removeEldestEntry(eldest)) {
            removeEntryForKey(eldest.key);
        }
    }
```

这里调用了父类HashMap的addEntry方法，如下：



```csharp
    void addEntry(int hash, K key, V value, int bucketIndex) {
        // 扩容相关
        if ((size >= threshold) && (null != table[bucketIndex])) {
            resize(2 * table.length);
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }
        // LinkedHashMap进行了重写
        createEntry(hash, key, value, bucketIndex);
    }
```

前面是扩容相关的代码，在上一篇HashMap解析中已经讲过了。这里主要看createEntry方法，LinkedHashMap进行了重写。



```csharp
   void createEntry(int hash, K key, V value, int bucketIndex) {
       HashMap.Entry<K,V> old = table[bucketIndex];
       // e就是新创建了Entry，会加入到table[bucketIndex]的表头
       Entry<K,V> e = new Entry<>(hash, key, value, old);
       table[bucketIndex] = e;
       // 把新创建的Entry，加入到双向链表中
       e.addBefore(header);
       size++;
   }
```

我们来看看LinkedHashMap.Entry的addBefore方法：



```cpp
        private void addBefore(Entry<K,V> existingEntry) {
            after  = existingEntry;
            before = existingEntry.before;
            before.after = this;
            after.before = this;
        }
```

从这里就可以看出，当put元素时，不但要把它加入到HashMap中去，还要加入到双向链表中，所以可以看出LinkedHashMap就是HashMap+双向链表，下面用图来表示逐步往LinkedHashMap中添加数据的过程，红色部分是双向链表，黑色部分是HashMap结构，header是一个Entry类型的双向链表表头，本身不存储数据。

![image-20200821052634585](https://raw.githubusercontent.com/zhaoxiaowu/blog/master/2020/image-20200821052634585.png)

跟HashMap的put类似，只不过多了把新增的Entry加入到双向列表中。

## 扩容

LinkedHashMap重写了transfer方法，数据的迁移，它的实现如下：



```java
    void transfer(HashMap.Entry[] newTable, boolean rehash) {
        // 扩容后的容量是之前的2倍
        int newCapacity = newTable.length;
        // 遍历双向链表，把所有双向链表中的Entry，重新就算hash，并加入到新的table中
        for (Entry<K,V> e = header.after; e != header; e = e.after) {
            if (rehash)
                e.hash = (e.key == null) ? 0 : hash(e.key);
            int index = indexFor(e.hash, newCapacity);
            e.next = newTable[index];
            newTable[index] = e;
        }
    }
```

可以看出，LinkedHashMap扩容时，数据的再散列和HashMap是不一样的。

HashMap是先遍历旧table，再遍历旧table中每个元素的单向链表，取得Entry以后，重新计算hash值，然后存放到新table的对应位置。

LinkedHashMap是遍历的双向链表，取得每一个Entry，然后重新计算hash值，然后存放到新table的对应位置。

从遍历的效率来说，遍历双向链表的效率要高于遍历table，因为遍历双向链表是N次（N为元素个数）；而遍历table是N+table的空余个数（N为元素个数）。

## 双向链表的重排序

HashMap的put方法中的如下代码：



```csharp
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                // 重排序
                e.recordAccess(this);
                return oldValue;
            }
        }
```

主要看e.recordAccess(this)，这个方法跟访问顺序有关，而HashMap是无序的，所以在HashMap.Entry的recordAccess方法是空实现，但是LinkedHashMap是有序的,LinkedHashMap.Entry对recordAccess方法进行了重写。



```csharp
        void recordAccess(HashMap<K,V> m) {
            LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
            // 如果LinkedHashMap的accessOrder为true，则进行重排序
            // 比如前面提到LruCache中使用到的LinkedHashMap的accessOrder属性就为true
            if (lm.accessOrder) {
                lm.modCount++;
                // 把更新的Entry从双向链表中移除
                remove();
                // 再把更新的Entry加入到双向链表的表尾
                addBefore(lm.header);
            }
        }
```

在LinkedHashMap中，只有accessOrder为true，即是访问顺序模式，才会put时对更新的Entry进行重新排序

## get方法

LinkedHashMap有对get方法进行了重写，如下：



```kotlin
    public V get(Object key) {
        // 调用genEntry得到Entry
        Entry<K,V> e = (Entry<K,V>)getEntry(key);
        if (e == null)
            return null;
        // 如果LinkedHashMap是访问顺序的，则get时，也需要重新排序
        e.recordAccess(this);
        return e.value;
    }
```

后面调用了LinkedHashMap.Entry的recordAccess方法，上面分析过put过程中这个方法，其实就是在访问顺序的LinkedHashMap进行了get操作以后，重新排序，把get的Entry移动到双向链表的表尾。

## 参考

[LinkedHashMap 底层分析](https://juejin.im/post/6844903560589606925)

[图解LinkedHashMap原理](https://www.jianshu.com/p/8f4f58b4b8ab)