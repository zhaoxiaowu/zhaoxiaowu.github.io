---
layout: post
category: JDK源码
---
## 一、简述

### 继承结构

![image-20200827103550217](https://gitee.com/tostringcc/blog/raw/master/2020/image-20200827103550217.png)

### Deque

> Deque接口是“double ended queue”的缩写(通常读作“deck”)，即双端队列，支持在线性表的两端插入和删除元素.因此，我们很可以认为，Deque接口既可以当做队列，也可以当做栈。

我们可以发现**LinkList以链表结构，同时实现了队列和栈**

## 二、分析

### List

在分析LinkedList之前，还是先瞄一眼List接口，虽然前篇已经看过一遍了，但为了明确下文的分析方向，还是先把List接口中的几个增删改查方法再列一次。

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

### LinkedList

#### 1、成员变量

```
public class LinkedList<E> extends AbstractSequentialList<E> implements List<E>, Deque<E>, Cloneable, java.io.Serializable{
    transient int size = 0;
    transient Node<E> first;
    transient Node<E> last;
	...
}
```

- size：数组元素个数
- first：头节点
- last：尾节点

LinkedList的成员变量很少，就上面那3个，其中first和last都是Node类型（即节点类型），用来表示链表的头和尾，这跟ArrayList就存在着本质的区别了。

> 要注意：
>  first和last仅仅只是节点而已，跟数据元素没有关系，可以认为就是2个额外的"指针"，分别指着链表的头和尾。

#### 2、构造函数

##### 1）LinkedList

```
public LinkedList() {
}
```

LinkedList的构造函数有2个，以平时最常用的构造函数为例，发现该构造函数什么事都没做。

##### 2）Node

```
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

再来看看这个节点类型的类结构，它描述了一个带有两个箭头的数据节点，也就是说LinkedList是双向链表。

> 为什么Node这个类是静态的？答案是：这跟内存泄露有关，Node类是在LinkedList类中的，也就是一个内部类，若不使用static修饰，那么Node就是一个普通的内部类，在java中，一个普通内部类在实例化之后，默认会持有外部类的引用，这就有可能造成内存泄露。但使用static修饰过的内部类（称为静态内部类），就不会有这种问题，在Android中，有很多这样的情况，如Handler的使用。好像扯远了~

好了，那下面就看看LinkedList是怎么进行增、删、改、查的。

#### 3、增

##### 1）add(E e)

```
public boolean add(E e) {
    linkLast(e);
    return true;
}
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```

因为LinkedList是链表结构，所以每添加一个元素就是让这个元素链接到链表的尾部。
 add(E e)的核心是linkLast()方法，它对元素进行了真正添加操作，分为以下几个步骤：

1. 先让此时集合中的尾节点（即last"指针"指向的节点）赋给变量 l 。
2. 然后，创建一个新节点，结合Node的构造函数，我们可以知道，在创建新节点（newNode）的同时，newNode的prev指向了l(即之前集合中的尾节点)，变量 l 就是newNode的前驱节点了，newNode的后继节点为null。
3. 再将last指向newNode，也就是说newNode成为该链表新的末尾节点。
4. 接着，判断变量 l 是否为null，若是null，说明之前集合中没有元素（此时newNode是集合中唯一一个元素），则将first指向newNode，也就是说此时的newNode既是头节点又是尾节点（要知道，这时newNode中的prev和next均是null，但被first和last同时指向）；
    若变量 l 不是null，说明之前集合中已经存在了至少一个元素，则让之前集合中的尾节点（即变量 l ）的next指向newNode。（结合步骤2，此时的newNode与newNode的前驱节点 l 已经是相互指向了）
5. 最后，跟ArrayList一样，让记录集合长度的size加1。

通过对add(E e)方法的分析，我们也知道了，原来LinkedList中的元素就是一个个的节点（Node），而真正的数据则存放在Node之中（数据被Node的item所引用）。

##### 2）add(int index, E element)

```
public void add(int index, E element) {
    checkPositionIndex(index);

    if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index));
}
```

该add方法将添加集合元素分为2种情况，一种是在集合尾部添加，另一种是在集合中间或头部添加，因为第一种情况也是调用linkLast()方法，这里不再啰嗦，我们看看第二种情况，分析linkBefore(E e, Node succ)这个方法是怎么对元素进行添加操作的。

```
void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}
```

往LinkedList集合中间或头部添加元素分为以下几个步骤：

1. 先调用node(int index)方法得到指定位置的元素节点，也就是linkBefore()方法中的形参 succ。
2. 然后，通过succ.prevt得到succ的前一个元素pred。（此时拿到了第index个元素succ，和第index-1个元素pred）
3. 再创建一个新节点newNode，newNode的prev指向了pred，newNode的next指向了succ。（即newNode往succ和pred的中间插入，并单向与它们分别建立联系，**eg：pred ← newNode → succ**）
4. 再让succ的prev指向newNode。（succ与跟newNode建立联系了，此时succ与newNode是双向关联，**eg：pred ← newNode ⇋ succ**）。
5. 接着，判断pred是否为null，若是null，说明之前succ是集合中的第一个元素（即index值为0），现在newNode跑到了succ前面，所以只需要将first指向newNode（**eg：first ⇌ newNode ⇋ succ**）；
    若pred不为null，则将pred的next指向newNode。（这时pred也主动与newNode建立联系了，此时pred与newNode也是双向关联，**eg：pred ⇌ newNode ⇋ succ**）
6. 最后，让记录集合长度的size加1。

对于链表的操作还是有些复杂的，特别是这种双向链表，不过仔细理解下，也不是什么问题（看不懂的可以边看步骤边动手画一画）。到这里，对于LinkedList的第一个添加方法就分析完了。

###### 下面是对node(int index)方法的分析：

这也是LinkedList获取元素的核心方法，相当重要，因为后面会出现很多次，这里就顺带先分析一下了。

```
Node<E> node(int index) {
    // assert isElementIndex(index);

    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

  细看node(int index)方法中的代码逻辑，可以看到，它是通过遍历的方式，将集合中的元素一个个拿出来，再通过该元素的prev或next拿到下一个遍历的元素，经过index次循环后，最终才拿到了index对应的元素。
  跟ArrayList相比，因为ArrayList底层是数组实现，拥有下标这个特性，在获取元素时，不需要对集合进行遍历，所以查找某个元素会特别快（在数据量特别多的情况下，ArrayList和LinkedList在效率上的差别就相当明显了）。
  不过，LinkedList对元素的获取还是做了一定优化的，它对index与集合长度的一半做比较，来确定是在集合的前半段还是后半段进行查找。

#### 4、删

##### 1）remove(int index)

```
public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}
E unlink(Node<E> x) {
    // assert x != null;
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;

    if (prev == null) {
        first = next;
    } else {
        prev.next = next;
        x.prev = null;
    }

    if (next == null) {
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }

    x.item = null;
    size--;
    modCount++;
    return element;
}
```

在remove(int index)这个方法中，先通过index和node(int index)拿到了要被删除的元素x，然后调用了unlink(Node x)方法将其删除，自然，LinkedList删除元素的核心方法就是unlink(Node x)，删除操作分以下几个步骤：

1. 通过要删除的元素x拿到它的前驱节点prev和后继节点next。

   ![image-20201108205358898](https://gitee.com/tostringcc/blog/raw/master/2020/image-20201108205358898.png)

2. 若前驱节点prev为null，说明x是集合中的首个元素，直接将first指向后继节点next即可；

   ![image-20201108205413520](https://gitee.com/tostringcc/blog/raw/master/2020/image-20201108205413520.png)

   若不为null，则让前驱节点prev的next指向后继节点next，再将x的prev置空。（这时prev与x的关联就解除了，并与next建立了联系）。

   ![image-20201108205430993](https://gitee.com/tostringcc/blog/raw/master/2020/image-20201108205430993.png)

3. 若后继节点next为null，说明x是集合中的最后一个元素，直接将last指向前驱节点prev即可；（下图分别对应步骤2中的两种情况）

   ![image-20201108205438513](https://gitee.com/tostringcc/blog/raw/master/2020/image-20201108205438513.png)

   若不为null，则让后继节点next的prev指向前驱节点prev，再将x的next置空。（这时next与x的关联就解除了，并与prev建立了联系）。

   ![image-20201108205445569](https://gitee.com/tostringcc/blog/raw/master/2020/image-20201108205445569.png)

4. 最后，让记录集合长度的size减1。

##### 2）remove(Object o)

```
public boolean remove(Object o) {
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
 
```

remove(Object o)这个删除元素的方法的形参o是数据本身，而不是LinkedList集合中的元素（节点），所以需要先通过节点遍历的方式，找到o数据对应的元素，然后再调用unlink(Node x)方法将其删除，关于unlink(Node x)的分析在第一个删除方法中已经提到了，可往回再看看。

#### 5、查 & 改

LinkedList集合对数据的获取与修改均通过node(int index)方法来执行往后的操作，关于node(int index)方法的分析也已经在第一个添加方法的时候已经提过，这里也就不再啰嗦了。

##### 1）set(int index, E element)

```
public E set(int index, E element) {
    checkElementIndex(index);
    Node<E> x = node(index);
    E oldVal = x.item;
    x.item = element;
    return oldVal;
}
 
```

##### 2）get(int index)

```
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}
 
```

## 三、队列Queue

这里要顺带分析下java中的队列实现，why？因为java中队列的实现就是LinkedList，你可能会疑问，队列的英文是Queue，在java中也有对应的接口，怎么会跟LinkedList扯上关系呢？因为LinkedList实现了队列：

```
public class LinkedList<E> extends AbstractSequentialList<E> implements List<E>, Deque<E>, Cloneable, java.io.Serializable {
	...
}
 
```

代码中的Deque是Queue的一个子接口，它继承了Queue：

```
public interface Deque<E> extends Queue<E> {...}
 
```

从这两者的关系，不难得出，队列的实现方式也是链表。下面先来看看Queue的接口声明：

### 1、Queue

我们知道，队列是先进先出的，添加元素只能从队尾添加，删除元素只能从队头删除，Queue中的方法就体现了这种特性。

```
public interface Queue<E> extends Collection<E> {
	boolean offer(E e);
	E poll();
	E peek();
	...
}
 
```

- offer()：添加队尾元素
- poll()：删除队头元素
- peek()：获取队头元素

从上面这几个方法出发，来看看LinkedList是如何实现的。

### 2、LinkedList对Queue的实现

#### 1）增

```
public boolean offer(E e) {
    return add(e);
}
 
```

可以看到，在LinkedList中，队列的offer(E e)方法实际上是调用了LinkedList的add(E e)，add(E e)已经在最前面分析过了，就是在链表的尾部添加一个元素~

#### 2）删

```
public E poll() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}

private E unlinkFirst(Node<E> f) {
    // assert f == first && f != null;
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null;
    f.next = null; // help GC
    first = next;
    if (next == null)
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}
 
```

poll()方法先拿到队头元素 f ，若 f 不为null，就调用unlinkFirst(Node f)其删除。unlinkFirst(Node f)在实现上跟unlink(Node x)差不多且相对简单，这里不做过多说明。

#### 3）查

```
public E peek() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
}
 
```

peek()先通过first拿到队头元素，然后取出元素中的数据实体返回而已。

## 四：栈Stack

```
public void push(E e) {
    addFirst(e);
}

public E pop() {
    return removeFirst();
}
```

## 五：Deque的顺序存储ArrayDeque

ArrayDeque 用一个动态数组实现了栈和队列所需的所有操作。

#### 添加元素

```
    public void addFirst(E e) {
        if (e == null)
            throw new NullPointerException();
        elements[head = (head - 1) & (elements.length - 1)] = e;
        if (head == tail)
            doubleCapacity();
    }

    public void addLast(E e) {
        if (e == null)
            throw new NullPointerException();
        elements[tail] = e;
        if ( (tail = (tail + 1) & (elements.length - 1)) == head)
            doubleCapacity();
    }

    private void doubleCapacity() {
        assert head == tail;
        int p = head;
        int n = elements.length;
        int r = n - p; // number of elements to the right of p
        int newCapacity = n << 1;
        if (newCapacity < 0)
            throw new IllegalStateException("Sorry, deque too big");
        Object[] a = new Object[newCapacity];
        System.arraycopy(elements, p, a, 0, r);
        System.arraycopy(elements, 0, a, r, p);
        elements = a;
        head = 0;
        tail = n;
    } 
```

这里可以看到，无论是头部还是尾部添加新元素，当需要扩容时，会直接变化为原来的2倍。同时需要复制并移动大量的元素。

#### 删除元素

```
public E pollFirst() {
        final Object[] elements = this.elements;
        final int h = head;
        @SuppressWarnings("unchecked")
        E result = (E) elements[h];
        // Element is null if deque empty
        if (result != null) {
            elements[h] = null; // Must null out slot
            head = (h + 1) & (elements.length - 1);
        }
        return result;
    }

    public E pollLast() {
        final Object[] elements = this.elements;
        final int t = (tail - 1) & (elements.length - 1);
        @SuppressWarnings("unchecked")
        E result = (E) elements[t];
        if (result != null) {
            elements[t] = null;
            tail = t;
        }
        return result;
    } 
```

从头部和尾部删除（获取）元素，就比较方便了，修改head和tail位置即可。head是当前数组中第一个元素的位置，tail是数组中第一个空的位置。

### BlockingDeque

```
/**
 * A {@link Deque} that additionally supports blocking operations that wait
 * for the deque to become non-empty when retrieving an element, and wait for
 * space to become available in the deque when storing an element.
 * /

public interface BlockingDeque<E> extends BlockingQueue<E>, Deque<E> {
} 
```

关于Deque最后一点，BlockingDeque 在Deque 基础上又实现了**阻塞**的功能，当栈或队列为空时，不允许出栈或出队列，会保持阻塞，直到有可出栈元素出现；同理，队列满时，不允许入队，除非有元素出栈腾出了空间。常用的具体实现类是LinkedBlockingDeque，使用链式结构实现了他的阻塞功能。Android中大家非常熟悉的AsyncTask 内部的线程池队列，就是使用LinkedBlockingDeque实现，长度为128，保证了AsyncTask的串行执行。

**这里比较一下可以发现，对于栈和队列这两种特殊的数据结构，由于获取（查找）元素的位置已经被限定，因此采用顺序存储结构并没有非常大的优势，反而是在添加元素由于数组容量的问题还会带来额外的消耗；因此，在无法预先知道数据容量的情况下，使用链式结构实现栈和队列应该是更好的选择。**	

## 六、总结

1. LinkedList是基于链表实现的，并且是双向链表。
2. LinkedList中的元素就是一个个的节点，而真正的数据则存放在Node之中。
3. LinkedList通过遍历的方式获取集合中的元素，效率比ArrayList低。
4. Queue队列的实现方式也是链表，java中，LinkedList是Queue的实现。
5. 实现Deque双端队列，可以实现队列和栈的功能。

**参考：**

[数据结构-栈&队列&Deque实现比较](https://juejin.im/post/6844903505325473799)

[LinkedList与Queue源码分析](https://juejin.im/post/6844903509633024013##heading-21)

