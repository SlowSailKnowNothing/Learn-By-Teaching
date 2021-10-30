---
title: HashMap源码解析
date: 2021-10-28 08:09:39
tags:
---

#### 引言

HashMap是java的经典类，也是面试常常问的类，因此，阅读hashmap的源码不仅可以加深我们对hashmap的了解，更能让我们学习一些设计思想和技巧。话不多说，开始看源码。

#### 源码解析

我们的源码解析以jdk1.8为主。同时可能会回顾一些jdk1.7中的hashmap的实现作为参考。

首先我们看看hashmap的几个静态变量：

```java
private static final long serialVersionUID = 362498820763181265L;
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
static final int MAXIMUM_CAPACITY = 1 << 30;
static final float DEFAULT_LOAD_FACTOR = 0.75f;
static final int TREEIFY_THRESHOLD = 8;
static final int UNTREEIFY_THRESHOLD = 6;
static final int MIN_TREEIFY_CAPACITY = 64;
```

其中第一个变量是serialVersionUID,该ID是为了序列化而定义的。在我们自己写码的时候，有时候如果一个类需要序列化，我们就简单的实现一个序列化接口，而没有去定义一个序列化ID。其实最好还是定义一个这样的ID，该ID的作用是在**反序列化期间用于验证序列化对象的发送方和接收方是否为该对象加载了与序列化兼容的类**。即序列化的时候和反序列化的时候做一个对比，如果两个ID相同才认为是相同的，这就保证了当类的版本发生变化的时候，可以相互兼容，这里有一个stackoverflow的回答可以作为自定义序列化ID作用的参考：https://stackoverflow.com/questions/25501988/can-hashmap-serialized-in-1-7-be-used-in-1-6

而我们看到不论是jdk1.7还是jdk1.8它们的序列化ID都是一样的。即即使类的版本升级了，但是由于我们的序列化ID是自己计算的，因此也可以成功反序列化。



另外几个量比如初始容量16，负载因子0.75，变成红黑树的门限值0.75，变成红黑树的hashmap软了最小值64等，都是经过权衡之后的结果。这里要注意的一点是这里常量的编码方式。这里16用1<<4来表示，是因为这样可读性更好，告诉我们是2的4次方。且编译器会自动计算该值，也不影响速度。

接下来是Node的定义如下：

```java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }
```



这里我们要注意，这里node的定义方式是静态内部类。事实上，静态类都是以内部类出现的，且静态内部类是较独立于外部类的。即使用静态内部类的场景是 **外部类需要使用内部类，而内部类无需外部资源，且内部类可以单独创建的时候。**而我们的Node实际上就是hashmap需要使用，但是本身不需要hashmap的资源，因此可以用静态内部类的方式来实现。



HashMap的源码继续实现了几个静态函数，比如下面生成hash值的函数：

```java
   static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

上面在利用object类自带的hashCode（）生成hash值之后，做了一个hash值扰动的操作。

扰动的原因及过程概况下来有如下几条：

1.hashcode生成的code范围很大，肯定不可用直接作为数组的下标。

2.数组用n-1的这种方式，将hashcode的低位取出。

3.但是受限于数组的长度，低位就那几位，信息有限。

4.于是我们希望最后生成的结果中，能保留高位的信息、

5.于是我们将hashcode进行无符号移位（>>>）,即高位右移并补0同时与自身做^运算，这样得到的结果就有低位和高位的信息。



知乎有一个非常漂亮的回答，这样清晰的写作风格是自己想要学习的：JDK 源码中 HashMap 的 hash 方法原理是什么？ - 胖君的回答 - 知乎 https://www.zhihu.com/question/20733617/answer/111577937



接下来的两个静态方法是用来比较对象的。当hashtabe的链表转换为红黑树，我们需要put一个新元素时，我们要确定put新元素的位置。如果如果该元素键的hash值小于当前节点的hash值的时候，就会作为当前节点的左节点；hash值大于当前节点hash值得时候作为当前节点的右节点。那么hash值相同的时候呢？这时还是会先尝试看是否能够通过Comparable进行比较一下两个对象（当前节点的键对象和新元素的键对象），要想看看是否能基于Comparable进行比较的话，首先要看该元素键是否实现了Comparable接口，此时就需要用到comparableClassFor方法来获取该元素键的Class，然后再通过compareComparables方法来比较两个对象的大小.

```java
    static Class<?> comparableClassFor(Object x) {
        if (x instanceof Comparable) {
            Class<?> c; Type[] ts, as; Type t; ParameterizedType p;
            if ((c = x.getClass()) == String.class) // bypass checks
                return c;
            if ((ts = c.getGenericInterfaces()) != null) {
                for (int i = 0; i < ts.length; ++i) {
                    if (((t = ts[i]) instanceof ParameterizedType) &&
                        ((p = (ParameterizedType)t).getRawType() ==
                         Comparable.class) &&
                        (as = p.getActualTypeArguments()) != null &&
                        as.length == 1 && as[0] == c) // type arg is c
                        return c;
                }
            }
        }
        return null;
    }
```

```java
    static int compareComparables(Class<?> kc, Object k, Object x) {
        return (x == null || x.getClass() != kc ? 0 :
                ((Comparable)k).compareTo(x));
    }
```

接下来是将给定的容量转换为最小的不小于该容量的二进制值的函数：

```java
 static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```



接下来我们来看根据key来获取val的函数：

```JAVA
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```

get的逻辑如下：

1.首先根据hash值取得索引值。

2.根据索引值判断第一个节点的key是否是传入的key。

3.如果数据是红黑树，那么查找红黑树。

4.如果数据是链表，那么查找链表。

5.如果没有符合条件的节点，返回null。

说完get，继续说put。putVal的代码如下：

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;//如果tab为空，那么就扩容,注意，jdk1.8中首次扩容在put的时候
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);//如果对应的索引位置为空，生成一个node
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;//如果已经有节点，并且key相同，那么就覆盖节点
            else if (p instanceof TreeNode)//如果是红黑树节点，那么调用红黑树的put方法
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);//遍历，如果有一个点为空，那么就put
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);//如果超出阈值，那么就变成树
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

接下来让我们看看hashmap的扩容操作：

```java
  final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;//如果旧容量已经比MAXIMUM_CAPACITY，直接拉满到MAX_VALUE
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)//如果oldCap两倍小于容量上线且大于初始容量
                newThr = oldThr << 1; // double threshold，门限翻倍
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;//上限不为0，但是容量为0，那么容量更改为上限值
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);//如果容量上限都为0就进行初始化
        }
        if (newThr == 0) {//如果新的门限为0，在之前有数的时候是不会执行的
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];//这里是执行扩容了，新建一个数组
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {//将旧的数组放到新的数组里面
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {//如果有数据
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
                            if ((e.hash & oldCap) == 0) {//注意，这里的oldCap没有减去1，所以这里是和高位做与操作。相当于判断高位是不是1，这里是0的清空，即索引值不变，注意移动的操作在后面进行
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



什么时候扩容？

元素超过之前的门限值的时候，就会扩容至原来的两倍。

每次扩容是翻倍，相当与原来计算的结果相比，只是多了一个bit位：

![image-20211024214148965](https://gitee.com/SlowSail/blogIMg/raw/master//img/image-20211024214148965.png)

以上面为例子，当数组的容量为16的时候，上面有两个hash值都是映射到5的位置。但是当数组的容量翻倍的时候，计算索引的公式(n-1)&hash所取到的hash的范围就扩大了一位，因此上面两个hash值一个映射到了5，一个映射到了5+16，对于是否加16，就取决于hash值的更高一位是0还是1，而这个位实际上是随机的，因此就保证了之前冲突的hash映射可以均匀地分布。于是，对于旧的数组元素，只有两种可能，一种是保持之前的索引，一种是新的索引=旧的索引+原来的长度。





#### HashMap的并发问题

在jkd1.8中，HashMap的并发问题在两种情况下可以体现。第一种情况是当两个线程都做put操作的时候。如果两个线程put的key的hash映射对应同一个索引的话，就可能出现并发问题。

我们看putVal函数的这个遍历链表的代码：

```java
 for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {//尾部
                        p.next = newNode(hash, key, value, null);//遍历，如果有一个点为空，那么就put
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);//如果超出阈值，那么就变成树
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
```

可以看到第二行，如果判断到了一个链表的尾部，就开始做newNode的操作。试想，如果两个线程都到了 第二行，且此时确实两个线程都通过了判断条件，获取了当前尾部的指针p，那么后面可能会各自新建自己的节点。

即如下所示：
![image-20211025081446984](https://gitee.com/SlowSail/blogIMg/raw/master//img/image-20211025081446984.png)

另外一种情况是同时做put和get操作的时候，如果put操作导致resize，那么就可能让原本有返回值的get操作返回了null。主要原因是在resize的时候，如下所示：

```java
for (int j = 0; j < oldCap; ++j) {//将旧的数组放到新的数组里面
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {//如果有数据
                    oldTab[j] = null;//将旧的数组置为空
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
```

在将数据转移到新数组的时候，会逐步将旧数组置为空，这就导致get得到的数据为null。
