---
layout:     post
title:      Java ConcurrentHashMap源码分析
subtitle:   Java ConcurrentHashMap源码分析
date:       2018-09-20
author:     XINGXING
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Java
---

>
>Java ConcurrentHashMap源码分析篇
> 

## 简介  
* ConcurrentHashMap是一个并发的散列映射表的实现，HashTable 和同步包装器包装的 HashMap同时只能有一个线程持有全局锁，导致多线程之间的并发访问是串行化的；
* ConcurrentHashMap和HashMap一样使用table存储Node，并用key的hashcode来确定table的index下标，处理hash碰撞时也是使用链表和红黑树来解决;    
* 在1.7之前ConcurrentHashMap使用的是segment锁分段的技术实现的高并发，而在1.8中ConcurrentHashMap是使用CAS+synchronized的方法来解决并发问题；

#### JDk1.7
ConcurrentHashMap采用 分段锁的机制，实现并发的更新操作，底层采用数组+链表的存储结构。  
其包含两个核心静态内部类 Segment和HashEntry。    
    
    结构:   
    Segment继承ReentrantLock用来充当锁的角色，每个Segment对象守护每个散列映射表的若干个桶。     
    HashEntry 用来封装映射表的键 / 值对；      
    每个桶是由若干个 HashEntry 对象链接起来的链表。     
一个 ConcurrentHashMap 实例中包含由若干个 Segment 对象组成的数组,如下图:   
![](https://ws4.sinaimg.cn/large/006tNc79gy1g2cxw3xfg8j30xa0u0q4q.jpg)   

##### JDK1.8  
1.8的实现已经抛弃了Segment分段锁机制，利用CAS+Synchronized来保证并发更新的安全，底层采用数组+链表+红黑树的存储结构。如下图:  
![](https://ws4.sinaimg.cn/large/006tNc79gy1g2cxx7x6mkj31bw0q0jyl.jpg)  
和HashMap类似，ConcurrentHashMap使用了一个table来存储Node，ConcurrentHashMap同样使用记录的key的hashCode来寻找记录的存储index，而处理哈希冲突的方式与HashMap也是类似的，冲突的记录将被存储在同一个位置上，形成一条链表，当链表的长度大于8的时候会将链表转化为一棵红黑树，从而将查找的复杂度从O(N)降到了O(lgN)


## 源码分析  

#### table的初始化

先说下sizeCtl变量，该变量默认为0，控制table的初始化和扩容操作；  
```java  
/**
     * Table initialization and resizing control.  When negative, the
     * table is being initialized or resized: -1 for initialization,
     * else -(1 + the number of active resizing threads).  Otherwise,
     * when table is null, holds the initial table size to use upon
     * creation, or 0 for default. After initialization, holds the
     * next element count value upon which to resize the table.
     * -1表示table正在初始化
     * -N表示table有N-1个线程正在进行table初始化操作
     * 如果table未初始化 则sizeCtl表示需要初始化的大小   
     * 如果table已经初始化则表示table的容量，默认是table大小的0.75倍； 
     */
    private transient volatile int sizeCtl;
```
table的初始化操作如下:
```java
 /**
     * Initializes table, using the size recorded in sizeCtl.
     */
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)  //如果sizeCtl<0则代表有其他线程正在进行初始化操作,即执行Thread.yield();让出cpu的执行权
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) { //否则将sizeCtl的值置为-1，代表当前线程正在初始化table
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```

table初始化完毕之后就可以执行put操作


#### put源码分析  
put操作采用CAS+synchronized实现并发插入或更新操作，代码如下：  
```java
 /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;  //f:代表hash所对应的元素Node，n:table的长度,fh:代表f节点的hash值
            if (tab == null || (n = tab.length) == 0) //判断table是否为空，如果为空则进行initTable()操作
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) { //先获取hash对应的索引i = (n - 1) & hash，然后使用tabAt(tab, i = (n - 1) & hash)获取
                                                                     //这里使用tabAt,即Unsafe.getObjectVolatile来获取索引所对应的元素来确保是从内存中获取到的元素，而不是从线程的私有copy中获取的
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)  //判断f的hash值是否为-1,如果是-1则表示当前table正在进行扩容
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) {  //否则按照链表或者红黑树来进行插入数据  在f节点进行同步
                    if (tabAt(tab, i) == f) { //使用unsafe来判断该索引上的元素是否是f,防止被其他线程修改
                        if (fh >= 0) { //f.hash如果>=0则表示是链表结构
                            binCount = 1;  //计算链表的长度
                            for (Node<K,V> e = f;; ++binCount) { //遍历链表，如果找到对应的node节点，则修改value，否则在链表尾部加入节点。  
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
                        else if (f instanceof TreeBin) {//如果f是TreeBin类型，则表示是红黑树的根节点,则在树结构上遍历元素，更新或增加节点。
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
                if (binCount != 0) {  //如果链表的长度>=8则将链表转为红黑树
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




