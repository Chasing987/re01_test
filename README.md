---
title: 常见面试题：为什么HashMap不是线程安全的呢？(JDK1.7和JDK1.8角度)（看完你就能和面试官笑谈人生了）
tags: 面试常见题

---

## 常见面试题：为什么HashMap不是线程安全的呢？(JDK1.7和JDK1.8角度)（看完你就能和面试官笑谈人生了）

<!-- more -->

## 为什么HashMap不是线程安全的呢？

我们在面试的时候，总是知道HashMap是线程不安全，如果你要保证线程安全的话就是用`ConcurrentHashMap`。但是我们好像从来没怎么没从HashMap的底层原理上去分析HashMap为什么不是线程安全的。

那现在就一起来分析一下，为什么HashMap不是线程安全的吧！！！

首先，先从整体上说一说HashMap吧。HashMap的线程不安全主要体现在会造成死循环、数据丢失、以及数据覆盖这些问题上。`在JDK1.7中`，主要是体现在`死循环和数据丢失的情况`；而`在JDK1.8中`，这种现象已经得到解决，但是在1.8中仍然会出现`数据覆盖`这样的问题。接下来，分别来分析它们为什么会造成这种情况呢。

### （一）在 JDK1.7 中引发的线程不安全的原因：

JDK1.7 主要是**在扩容的时候**引发的线程不安全，主要发生在扩容函数中，即**transfer函数**。下面就是**transfer函数**的源代码：

```java
/**
 * Transfers all entries from current table to newTable. 
 */
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {

        while(null != e) {
            //（关键代码）
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        } // while  

    }
}
```

开始分析之前，我们首先来回顾一下**HashMap的扩容机制：**

HashMap默认设定的装载因子`loadFactor` 为0.75，`HashMap`的大小为length，已经装载的元素数量为`nums`，当`（nums / length）> loadFactor`时，开始扩容。JDK1.7的`HashMap`的扩容操作，是重新定位每个桶的下标，并采用**头插法**将元素迁移到新的数组中。

先创建一个散列表HashMap：`Map<Integer> map = new HashMap<Integer>(2); `，装载因子默认0.75，当插入第二个元素时，会发生扩容。
我们先在map中放入6、8两个元素。

<img src="https://tva1.sinaimg.cn/large/008eGmZEly1gorifnkegpj30kc0hqq3i.jpg" style="zoom:50%;" />

这时有两个线程都执行`put操作`，那么在此刻两个线程都对`HashMap`进行扩容，这时候就注意在上文的源码里注释为（关键代码）这一行：`Entry<K,V> next = e.next;`

假如两个线程分别为A、B两个线程。A线程在执行到`关键代码`这一行线程就被挂起，那么此刻A线程中：`e = 6; next = 8;`

接着B线程开始进行扩容，假设新的散列表中，节点6 和 节点8 还是会产生散列冲突，那么线程B的扩容过程为：

①先申请一个空间为旧散列表两倍大的空间

<img src="https://tva1.sinaimg.cn/large/008eGmZEly1goriisqkgnj30qw0nkaaw.jpg" style="zoom:38%;" />

②将节点6 迁移至新散列表

<img src="https://tva1.sinaimg.cn/large/008eGmZEly1goril4io7uj30kc0hsjrv.jpg" style="zoom:50%;" />

③ 将节点8迁移至新的散列表

<img src="https://tva1.sinaimg.cn/large/008eGmZEly1gorimloshaj30kc0ht74x.jpg" style="zoom:50%;" />

此时线程B的扩容已经完成，节点8的后继节点为节点6，节点6的后继节点为null。

此时，我们可以将新旧两个散列表做个对比：

<img src="https://tva1.sinaimg.cn/large/008eGmZEgy1gorizieecbj319k0qi40s.jpg" style="zoom:40%;" />

回顾一下，线程A的当前状态：`e = 6; next = 8;`，处于挂起状态。接着A线程取消挂起状态，接着执行（关键代码）之后的代码：将`e = 6;`节点迁移至新的散列表，并将`next = 8`的节点赋值给`e`。扩容并迁移节点6后的状态，如下图所示：

<img src="https://tva1.sinaimg.cn/large/008eGmZEgy1gorj9eu0v3j30kc0b20ta.jpg" style="zoom:90%;" />

于是第二次执行while循环时，当前待处理节点：`e = 8;`

在执行（关键代码）这一行时，由于线程B在扩容时将节点8的后继节点变为节点6，所以next不是为null，而是`next = 6;`

<img src="https://tva1.sinaimg.cn/large/008eGmZEgy1gorjaw2yz7j30kc0i2t9d.jpg" style="zoom:70%;" />

接着开始执行第三次while循环，由于节点6的后继节点为null，所以 `next = null;`，执行完第三次while循环的结果为：

<img src="https://tva1.sinaimg.cn/large/008eGmZEgy1gorjbxlgb9j30kc0hs0tg.jpg" style="zoom:70%;" />

循环结束。

从这个扩容函数`transfer函数`来看，可以看到扩容后的散列表中链表成环，如果这时候执行`get()`方法查询，就会导致死循环。并且HashMap在并发执行`put操作`时扩容，可能会导致结点丢失，会导致数据不准的情况。

### （二）JDK 1.8 引发线程不安全的原因有：

其实在JDK1.8 中已经很好地解决了JDK1.7 中的问题了，如果再看JDK1.8 的源码，你就会发现JDK1.8中摒弃了`transfer函数`的，是直接在`resize函数`中完成数据的迁移，并且在JDK1.8 中插入元素时是使用的尾插法。但是，`JDK1.8 会带来数据覆盖的线程不安全`。

那为什么JDK1.8 会带来会出现数据覆盖的现象呢？可以来看一段JDK 1.8 中的put 操作代码：

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null) // 如果没有hash碰撞则直接插入元素
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

我们看上面的源代码，发现在第六行代码是`判断是否会出现hash碰撞`，假设此时有两个线程A，B都在利用该put函数进行put操作，并且利用hash函数计算出的插入下标是相同的，当线程A执行完第六行代码后，由于时间片被耗尽而被挂起，而线程B此时得到cpu的时间片后，在该下标处插入了元素，完成了正常的插入，然后线程A获得时间片，由于之前已经进行过了hash碰撞的判断，所以线程A此时并不会再去判断hash碰撞操作了，而是直接在该位置出进行插入，这就导致了线程B插入的数据被线程A覆盖了，从而导致线程不安全了。

除此之前，还有就是代码的第38行处有个`++size`，我们这样想，还是线程A、B，这两个线程同时进行put操作时，假设当前`HashMap`的zise大小为10，当线程A执行到第38行代码时，从主内存中获得size的值为10后准备进行+1操作，但是由于时间片耗尽只好让出CPU，线程B快乐的拿到CPU还是从主内存中拿到size的值10进行+1操作，完成了put操作并将size=11写回主内存，然后线程A再次拿到CPU并继续执行(此时size的值仍为10)，当执行完put操作后，还是将size=11写回内存，此时，线程A、B都执行了一次put操作，但是size的值只增加了1，所有说还是由于数据覆盖又导致了线程不安全。

**总结：**

**HashMap的线程不安全主要是体现在下面两个方面：**

① 在JDK1.7 中，当并发执行扩容操作时会`造成环形链`和`数据丢失`的情况；

② 在JDK1.8 中，在并发执行put操作的过程中，会出现`数据覆盖`的情况。