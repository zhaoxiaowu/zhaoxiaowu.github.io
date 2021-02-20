---
layout: post
category: JDK源码
---
## ArrayList源码分析

## 一、简述

我们知道，数据结构中有两种存储结构，分别是：顺序存储结构（线性表）、链式存储结构（链表），在java中，对这两种结构分别进行实现的类有：

- 顺序存储结构：ArrayList、Stack
- 链式存储结构：LinkedList、Queue

## 二、分析

### List

在分析ArrayList之前，我们先来看看集合的接口——List。

```
public interface List<E> extends Collection<E> {
    boolean add(E e);
    void add(int index, E element);
    boolean remove(Object o);
    E remove(int index);
    E set(int index, E element);
    E get(int index);
	...
}
```

在List这个接口中，提供了对集合进行操作的增、删、改、查方法，我们知道，ArrayList和LinkedList都实现了List接口，但它们的底层实现分别是线性表与链表，所以，对应的增、删、改、查方法肯定也不一样，下面的分析也将从这几个方法入手。

### ArrayList

#### 1、成员变量

在ArrayList的源码中，成员变量并不多，下面就截出其中几个重要的变量进行说明。

```
public class ArrayList<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, java.io.Serializable{
	...
    private static final int DEFAULT_CAPACITY = 10;
    private static final Object[] EMPTY_ELEMENTDATA = {};
	private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    transient Object[] elementData;
    private int size;
	...
}
```

- DEFAULT_CAPACITY：默认的数组长度
- EMPTY_ELEMENTDATA：默认的空数组
- DEFAULTCAPACITY_EMPTY_ELEMENTDATA：默认的空数组（与EMPTY_ELEMENTDATA有点区别，在不同的构造函数中用到）
- elementData：真正用于存放数据的数组
- size：数组元素个数

> ArrayList的底层实现是数组，数组必定是有限长度的，ArrayList中默认的数组大小是10。

#### 2、构造函数

```
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

这个构造函数是开发最常用的，可以看到，它仅仅只是让elementData等于一个空数组（DEFAULTCAPACITY_EMPTY_ELEMENTDATA）而已。

```
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}
```

这个构造函数可以指定初始化数组的长度，当initialCapacity大于0时，为elementData创建一个长度为initialCapacity的Object数组；当initialCapacity等于0时，则让elementData等于一个空数组（EMPTY_ELEMENTDATA）。

#### 3、增

##### 1）add(E e)

```
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```

在前面已经提到了，size是一个成员变量，表示ArrayList中的元素个数。在这个方法中，先调用了ensureCapacityInternal()方法确保数组有足够的容量，再对将元素添加到elementData数组中。下面就来看看ensureCapacityInternal()方法是如何确保数组有足够的容量的。

```
private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }

    ensureExplicitCapacity(minCapacity);
}
```

该方法结合ensureExplicitCapacity()方法，总的来说就是计算并扩大最小的容器体积。

> 这里就用到了DEFAULTCAPACITY_EMPTY_ELEMENTDATA这个空数组，如果此时elementData与DEFAULTCAPACITY_EMPTY_ELEMENTDATA相等，说明开发者使用的是无参构造函数创建了集合，而且是添加第一个元素，此时的容器大小为0。

```
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```

当minCapacity - elementData.length > 0时，说明当前数组（容器）的空间不够了，需要扩容，所以调用grow()方法。

> modCount只是一个计数变量而已，源码中有很多地方出现，无须理会。

```
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

grow()方法是用来给ArrayList集合进行扩容的，它计算出新的容器大小（即newCapacity），并确保了newCapacity不会比minCapacity小，最后调用Arrays.copyOf()创建一个新的数组并将数据拷贝到新数组中，最后让elementData进行引用。

> oldCapacity >> 1 等价于 oldCapacity / 2，也就是说ArrayList默认的扩容大小是当前数组大小的一半。

##### 2）add(int index, E element)

```
public void add(int index, E element) {
    rangeCheckForAdd(index);

    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    elementData[index] = element;
    size++;
}
```

经过对add(E e)方法进行分析，这个增加方法就很容易理解了，它先确保容器有足够大的空间，没有就扩容，然后将elementData数组中从index位置开始的所有元素往后"移动"1位，再对数组的index位置进行元素赋值，最后将记录集合中元素个数的size变量加1。

#### 4、删

##### 1）remove(int index)

```
public E remove(int index) {
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```

numMoved表示在执行删除操作时数组需要移动的元素个数，将elementData数组中从index后一位开始的所有元素（即numMoved个元素）往前"移动"1位（这样一移动，index位置的元素会被后面的元素覆盖，间接起到了删除元素的作用），然后把最后的那个元素置空，同时将size变量减1。

##### 2）remove(Object o)

```
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```

该方法的操作与remove(int index)基本一致，这里就不再说明了。（fastRemove()方法的代码不是可以复用么。。。）

#### 5、查 & 改

ArrayList的修改和获取元素的方法相当简单，就是对elementData数组进行简单操作罢了，这里就列出源码看看就好了。

##### 1）get(int index)

```
public E get(int index) {
    rangeCheck(index);
    return elementData(index);
}

E elementData(int index) {
    return (E) elementData[index];
}
```

##### 2）set(int index, E element)

```
public E set(int index, E element) {
    rangeCheck(index);

    E oldValue = elementData(index);
    elementData[index] = element;
    return oldValue;
}
```

## 三、总结

1. ArrayList底层实现是数组。
2. 当使用无参数构造函数创建ArrayList对象时，ArrayList对象中的数组初始长度为0（是一个空数组）。  添加元素后小于10，则扩容为10
3. ArrayList的扩容策略是每次都增加当前数组长度的一半（非固定分配）。
4. ArrayList的扩容方式是直接创建一个新的数组，并将数据拷贝到新数组中。

[ArrayList源码分析](https://juejin.im/post/6844903509633007629)