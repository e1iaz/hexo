---
title: HashMap
date: 2019-10-23 17:25:30
categories: "Java"
tags: ["Java", "HashMap"]
---
# HashMap

## 摘要

HashMap是使用频率最高的用于映射处理的数据类型，随着JDK版本的更新，对HahsMap底层的实现进行了优化，例如引入红黑树的数据结构和扩容的优化。根据[美团技术团队](https://zhuanlan.zhihu.com/p/21673805)和自身理解写一下这个博客  

## 简介

Java为数据结构中的映射定义了一个接口`java.util.Map`，次接口主要有四个常用的实现类，分别是`HashMap` `Hashtable` `LinkedHashMap` `TreeMap`。
(1) HahsMap：它根据键的`hashCode`值存储数据，大多数情况下可以直接定位到它的值，因而具有很快的访问速度，但遍历顺序确实不确定的。HahsMap最多只允许一条记录的键为null，允许多条记录的值为null。HahsMap非线程安全，如果要满足线程安全，可以用`Collections`的`synchronizedMap`方法使HashMap举有线程安全的能力，或者使用`ConcurrentHashMap`。  
(2) Hashtable：遗留类，很多映射的常用功能与HahsMap类似，不同的是它继承自`Dectionary`类，且线程安全，并发性不如`ConcurrentHashMap`，因为`ConcurrentHashMap`引入分段锁。不建议在新代码中使用。  
(3) LinkedHashMap：是HashMap的一个子类，保存了记录的插入顺序，在用Iterator遍历LinkedHashMap时，先得到的记录肯定是先插入的，也可以在构造时带参数，按照访问次序排序  
(4) TreeMap：实现`SortedMap`接口，能够把它保存的记录根据键排序，默认时按键值的升序排序，也可以指定排序的比较器。当用Iterator遍历TreeMap时，得到的记录时排过序的。如果使用排序的映射，建议使用TreeMap。在使用TreeMap时，key必须实现Comparable接口或者在构造TreeMap传入自定义的Comparator，否则会在运行时抛出`java.lang.ClassCastException`类型的异常  
下文我们主要结合源码，从存储结构、常用方法分析、扩容以及安全性等方面深入讲解HashMap的工作原理。  

## 内部实现

搞清HashMap，首先需要知道HashMap是什么，即它的存储结构——字段；其次要明白他能干什么，即它的功能实现——方法。

### 存储结构——字段

从结构实现来讲，HashMap时数据 + 链表 + 红黑树实现的，如下图所示  
![hashmap.png](https://i.loli.net/2019/10/23/wV1dxnYHTZ3fcq2.png)
这里需要明白两个问题：数据底层具体存储的是什么，这样的存储方式有什么优点  
(1) 从源码可以，HashMap类中有个非常重要的字段，就是Node[] table，即哈希桶数组，那么Node是什么  

```java
static class Node<K, V> implements Entry<K, V> {
    //用来定位数组索引位置
    final int hash;
    final K key;
    V value;
    //链表下一个node
    HashMap.Node<K, V> next;
    Node(int hash, K key, V value, HashMap.Node<K, V> next) {}
    public final K getKey() {}
    public final V getValue() {}
    public final String toString() {}
    public final int hashCode() {}
    public final V setValue(V newValue) {}
    public final boolean equals(Object o) {}
}
```

Node是HahsMap的内部类，实现了`Map.Entry`接口，本质是一个映射，上图中每个黑点就是一个Node对象。  
(2) HashMap就是使用哈希表来存储的。哈希表为解决冲突，可以采用开放地址法和链地址法等来解决问题，Java中HashMap采用了链地址法。链地址法就是数组加链表的组合。每个数组元素上都是一个链表结果，当数据被Hash后，得到数组下标，把数组放在对应下表元素的链表上。  
如果两个key定位到相同的位置，表示发生了Hash碰撞，当然Hash算法计算结果越分散均匀，Hash碰撞的概率越小，map的存取效率就会越高。如果哈希桶数组很大，即使较差的Hash算法也会比较分散，如果哈希桶数组很闲，即使好的Hash算法也会出现较多碰撞，所以是需要在空间成本和时间成本之间的权衡，其实就是在根据实际情况确定哈希桶数组的大小，并在此基础上设计好的hash算法减少Hash碰撞。好的Hash算法和扩容机制可以控制map使得Hash碰撞的概率又小，哈希桶数组占用空间又少。  
在了解Hash和扩容之前，先了解一下HashMap的几个字段  

``` java
int threshold;             // 所能容纳的key-value对极限 
final float loadFactor;    // 负载因子
int modCount;  
int size;
```

`Node[] table`的初始长度length默认为16，`Load factor`为负载因子默认值为0.75，threshold是hashMap所能容纳的最大数量的Node个数，`threshold = length * Load factor`。如果数组定义好长度之后，负载因子越大，所容纳的键值对个数越多  
结合负载因子的定义公式可知，threshold就是在此Load factor和length对应下允许的最大元素数量，超过这个数目就重新resize，扩容后的HashMap容量是之前的两倍。默认的负载因子0.75是对空间和时间效率的一个平衡选择，建议不要修改，出发i在时间和空间比较特殊的情况下。如果内存空间很多而对时间效率要求又高，可以减低负载因子的值；相反，如果内存空间紧张而对时间效率要求不高，可以增加负载因子的值。这个值可以大于1.  
size这个字段是HashMap中实际存在的键值对数量。modCount字段主要记录HashMap内部结构发生变化的次数，主要用于迭代的快速失败。  
> 内部结构发生变化指结构发生变化，例如put新键值对，但某个key对应的value值被覆盖不属于结构变化  

在HashMap中，哈希桶数组table的长度length大小必为2的n次方。这是一种非常规的设计，常规的设计是把桶的大小设计为素数，相对于素数导致冲突的概率小于合数。HashMap采用合数是为了在取模和扩容时做优化，同时为了减少冲突，HashMap定位哈希桶索引位置时，也加入了高位参与运算的过程。  
> 为什么常规使用素数导致冲突的概率小？  
> 首先hash的意义在于把一个大的集合A映射到小的集合B，通过取余的方式进行Hash。然后如果A的元素分布是{0, 1, 2, 3, .....}，合数素数都没问题。但是A的分布是非1步长的，如果A的元素是{0, 2, 4, 6, ....}，如果对8取模的话1, 3, 5, 7的位置就空，最坏情况就会退化成链表。

这里存在一个问题，即使负载因子和Hash算法设计再合理，也免不了会出现拉链过长的情况，一旦出现链过长，则会严重影响HashMap的性能，所以1.8之后对数据结构做了优化，引入红黑树。当链表长度太长（默认超过8），链表转换为红黑树，提高性能。

## 功能实现——方法

HashMap的内部功能实现很多，先从根据key获取哈希桶数组索引位置、put方法的详细执行、扩容过程三个具有代表性的点深入展开讲解。

### 确定哈希桶数组索引位置

不管增加、删除、查找键值对，定位到哈希桶数组的位置都是关键的第一步。HashMap定位数组索引位置直接决定了hash方法的离散性能，先看源码实现（方法一 + 方法二）  

```java
//方法一
static final int hash(Object key) {
    int h;
    // h = key.hashCode() 为第一步  取hashCode值
    // h ^ (h >>> 16) 为第二部  高位参与运算
    return key == null ? 0 : (h = key.hashCode()) ^ h >>> 16;
}
//方法二
static int indexFor(int h, int length) {  //jdk1.7的源码，jdk1.8没有这个方法，但是实现原理一样的；
     return h & (length-1);  //第三步 取模运算
}
//jdk1.8源码，原理一样，但是直接看不是很好理解
n = (tab = this.resize()).length;
tab[i = n - 1 & hash]
```

这里Hash算法本质就是三步：取key的hashCode值、高位运算、取模运算  
对于任意给定的对象，只要他的hashCode()返回值相同，那么程序调用方法一所计算得到的Hash码值总是相同。我们首先想到的就是把hash值对数组长度取模运算，这样元素的分布相对来说比较均匀。但是模运算的消耗比较大，HashMap是这样做的：调用方法二来计算该对象应该保存在table数组的哪个索引处。  
这个方法非常巧妙，它通过`h & (table.length - 1)`来得到该对象的保存位，而HashMap底层数组长度总是2的n次方，所以`h & (table.length - 1)`等价于对length取模，也就是h%length，但是&比%更高效。  
在jdk8中，优化了高位运算的算法，通过hashCode()的高16位异或低16位实现: `(h = key.hashCode()) ^ (h >>> 16)`，主要是从速度、功效、质量来考虑的，这么做可以在数组table的length比较小的时候，也能保证考虑到高低Bit都参与到Hash的计算中，同时不会有太大开销。  

### 分析HashMap的put方法

![put.png](https://i.loli.net/2019/10/23/eLYmBPtKU9ACXfD.png)

1. 判断键值对数组table[i]是否为空或者null，否则执行resize()扩容
2. 根据键值key计算hash值得到插入的数组索引i，如果table[i] == null，直接新建节点添加，转向7，如果table[i]不为空，转型3
3. 判断table[i]的首个元素时候和key一样，如果想通过直接覆盖value，否则转型4
4. 判断table[i]是否为treeNode，如果是，则直接插入键值对，否则转向5
5. 遍历table[i]，链表插入，如果插入后长度大于8，则把链表转换为红黑树  

> 美团的文章中我觉得有些问题
> 遍历table[i]，判断链表长度是否大于8，大于8的话把链表抓换为红黑树，红黑树中执行插入操作，否则进行链表的插入操作，遍历过程中发现key已经存在直接覆盖value即可

6. 根据参数判断是否覆盖原值。
7. 插入成功后，判断实际存在的键值对数量size是否超过了最大容量threshold，如果超过，进行扩容。

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    HashMap.Node[] tab;
    int n;
    if ((tab = this.table) == null || (n = tab.length) == 0) {
        n = (tab = this.resize()).length;
    }
    Object p;
    int i;
    if ((p = tab[i = n - 1 & hash]) == null) {
        tab[i] = this.newNode(hash, key, value, (HashMap.Node)null);
    } else {
        Object e;
        Object k;
        if (((HashMap.Node)p).hash == hash && ((k = ((HashMap.Node)p).key) == key || key != null && key.equals(k))) {
            e = p;
        } else if (p instanceof HashMap.TreeNode) {
            e = ((HashMap.TreeNode)p).putTreeVal(this, tab, hash, key, value);
        } else {
            int binCount = 0;
            while(true) {
                if ((e = ((HashMap.Node)p).next) == null) {
                    ((HashMap.Node)p).next = this.newNode(hash, key, value, (HashMap.Node)null);
                    if (binCount >= 7) {
                        this.treeifyBin(tab, hash);
                    }
                    break;
                }
                if (((HashMap.Node)e).hash == hash && ((k = ((HashMap.Node)e).key) == key || key != null && key.equals(k))) {
                    break;
                }
                p = e;
                ++binCount;
            }
        }
        if (e != null) {
            V oldValue = ((HashMap.Node)e).value;
            if (!onlyIfAbsent || oldValue == null) {
                ((HashMap.Node)e).value = value;
            }
            this.afterNodeAccess((HashMap.Node)e);
            return oldValue;
        }
    }
    ++this.modCount;
    if (++this.size > this.threshold) {
        this.resize();
    }
    this.afterNodeInsertion(evict);
    return null;
}
```

### 分析HashMap中的putAll方法

`putAll()`方法要讲的真的不多，都是先根据插入的map计算容量，对于进行插入。    //因为插入的map一定会在原map中出现，所以，如果插入的map容量大于阈值，就可以提前扩容至插入map的容量，这样可以减少扩容的次数。先看下1.7的源码

```java
public void putAll(Map<? extends K, ? extends V> m) {
    int numKeysToBeAdded = m.size();
    if (numKeysToBeAdded == 0)
        return;
    //如果被插入的map里面没有数据就要可以根据新map新建了
    if (table == EMPTY_TABLE) {
        inflateTable((int) Math.max(numKeysToBeAdded * loadFactor, threshold));
    }
    //扩容
    if (numKeysToBeAdded > threshold) {
        int targetCapacity = (int)(numKeysToBeAdded / loadFactor + 1);
        if (targetCapacity > MAXIMUM_CAPACITY)
            targetCapacity = MAXIMUM_CAPACITY;
        int newCapacity = table.length;
        while (newCapacity < targetCapacity)
            newCapacity <<= 1;
        if (newCapacity > table.length)
            resize(newCapacity);
    }
    for (Map.Entry<? extends K, ? extends V> e : m.entrySet())
        put(e.getKey(), e.getValue());
}
```

在扩容的时候会指定扩容到多少容量，所以可以直接扩容，这个容量是合并后最小的容量。如果两个map没有相同的key，即在put()的时候扩容。  
但是在jdk8以后我就不是很理解了。  

### 分析HashMap中的remove方法

remove()方法的的流程就是先定位，根据key求出索引位置，然后在桶内查询。这个流程在1.7和1.8一致。先看下1.7的代码  

```java
final Entry<K,V> removeEntryForKey(Object key) {
    if (size == 0) {
        return null;
    }
    int hash = (key == null) ? 0 : hash(key);
    //求出key的位置
    int i = indexFor(hash, table.length);
    Entry<K,V> prev = table[i];
    Entry<K,V> e = prev;
    //链表遍历查找，如果找到返回e，没有返回null(也是e)
    while (e != null) {
        Entry<K,V> next = e.next;
        Object k;
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k)))) {
            modCount++;
            size--;
            if (prev == e)
                table[i] = next;
            else
                prev.next = next;
            e.recordRemoval(this);
            return e;
        }
        prev = e;
        e = next;
    }
    return e;
}
```

由于在jdk8中采用了红黑树，remove会出现红黑树退化到链表的情况，这个还没太看懂。

### 扩容机制

扩容是重新计算容量，向HashMap对象不停的添加元素，而HashMap对象内部的数组无法装在更多的元素时，对象就需要扩大数组的长度，以便能装入更多的元素。当然Java的数组时无法自动扩容的，方法时使用一个新的数组代替已有容量小的数组。  
先看下jdk7的代码，这个比较好理解  

```java
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    //如果数组大小已经达到最大(2^30)，则修改阈值为(2^31 - 1)，不在扩容
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }
    Entry[] newTable = new Entry[newCapacity];
    //将数组转移到新的数组，并将table属性引用新的的数组，修改阈值
    transfer(newTable, initHashSeedAsNeeded(newCapacity));
    table = newTable;
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}

//拷贝函数，与美团的博客不一样，思路一样，不多我觉得美团的代码会好一些，因为对于旧数组职位null
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            //头插法
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}
```

在扩容拷贝的时候，jdk8做了一些优化。我们使用的是2次幂的扩展，所以元素位置要么是原位置，要么是在移动2次幂的位置  
![resize.jpg](https://i.loli.net/2019/10/23/Vurep7EZtLAcSFl.jpg)  
因此我们在扩容的时候，不需要像jdk7的实线那样重新计算hash，只需要看原来的hash值新增的那个bit是1还是0就好了，0则索引不变，1的话`原索引 ~ oldCap`。这样设计确实巧妙，省去了重新计算hash值的时间。  
还有一点，jdk在迁移的时候，如果在信标的数组索引位置相同，则链表元素会倒置，但在jdk不会，接下来看下jdk8的源码  

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        //扩容的数值计算
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
    else {
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    //计算新的resize阈值
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
        //将旧的移动到新的里
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
                        //通过判断前一位是0还是1来判断是当前位置还是新增位置
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

在扩容的时候，一个桶内如果是红黑树的话，扩容后数量会减少。所以在扩容后，如果容量会小于等于6的时候就会将红黑树退化成链表。  

```java
final Node<K,V> untreeify(HashMap<K,V> map) {
    Node<K,V> hd = null, tl = null;
    for (Node<K,V> q = this; q != null; q = q.next) {
        Node<K,V> p = map.replacementNode(q, null);
        if (tl == null)
            hd = p;
        else
            tl.next = p;
        tl = p;
    }
    return hd;
}
```

## 线程安全

在多线程使用场景中，应该尽量避免使用线程不安全的`HashMap`，而是用线程安全的`ConcurrentHashMap`。为什么说hashmap是线程不安全的呢，因为在并发环境下会造成HashMap
的死循环，成环了。  

```java
public class HashMapInfiniteLoop {  
    private static HashMap<Integer,String> map = new HashMap<Integer,String>(2，0.75f);  
    public static void main(String[] args) {  
        map.put(5， "C");  
        new Thread("Thread1") {  
            public void run() {  
                map.put(7, "B");  
                System.out.println(map);  
            };  
        }.start();  
        new Thread("Thread2") {  
            public void run() {  
                map.put(3, "A);  
                System.out.println(map);  
            };  
        }.start();        
    }  
}
```

通过设置断点让线程一和线程二同时debug到transfer方法的首行，此时两个线程已经成功添加数据。放开的thread1的断点至transfer的`Entry next = e.next`这一行；然后放开线程2的断点，让线程2进行resize。  
![threads.png](https://i.loli.net/2019/10/23/8JIKGtYA32NTPfV.png)  
此时thread1的e指向key(3)，而next指向了key(7)，其在线程而rehash后，指向线程二重组后的链表。线程以北调度回来执行，先执行`newTable[i] = e`，然后是`e = next`，导致e指向了key(7)，而下一次循环的`next = e.next`导致了next指向了key(3)。成环了。
这个原因我猜的才是是因为hashmap在1.7的时候是头插法，这样会导致当前桶的逆序，而在1.8之后使用尾插法，理论上不会成环，但还是倒是覆盖的问题。  

## JDK1.8和JDK1.7的性能对比

如果Hash算法非常好的的话，`getKey`方法的时间复杂度是O(1)，如果碰撞非常多，所有的Hash结果的都跑到一个桶，或者是一个链表，或者是红黑树，时间复杂度分别是O(n)和O(lgn)。所以jdk1.8的总体性能优于jdk1.7

## 小结

1. 扩容是一个特别消耗性能的操作，所以当程序员在使用Hashmap的时候，估算map的大小，初始化的时候给一个大致的数组，避免map进行频繁的扩容  
2. 负载因子是可以修改的，也可以大于1，但是建议不要轻易修改，除非情况非常特殊  
3. HashMap是线程不安全的，不要在并发的环境下同时操作HashMap，建议使用ConcurrentHashMap  
4. jdk1.8中引入红黑树大程度优化了HashMap的性能  

## 疑问

### putAll

在1.7的时候putAll使用的

```java
while (newCapacity < targetCapacity)
    newCapacity <<= 1;
```

进行直接扩容，而在1.8之后只要小于就先扩容一次

```java
int s = m.size();
if (s > 0) {
    if (s > threshold)
        resize();
}
```

既然下面的`putVal()`也会扩容，那么之前的扩容有什么意义呢？？

### remove

remove过程中，从红黑树退化到链表。
