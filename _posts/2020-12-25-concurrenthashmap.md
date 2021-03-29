---
layout:     post
title:      "ConcurrentHashMap 简单研究"
subtitle:   ""
date:       2020-12-25 12:00:00
author:     "boredream"
catalog: false
header-style: text
tags:
- 

---

ConcurrentHashMap 可以理解为线程安全的 HashMap  

### 对比
#### HashTable 
不能null的key或value，线程安全的，所有方法都是synchronize的

#### HashMap
普通Hash表，支持null的key，把null直接作为hashcode=0

#### Collections.synchronizedMap 
传入map，然后将所有方法都包一层synchronized关键字。本质和HashTable差不多

### ConcurrentHashMap原理
线程安全的Hash表，不同java sdk下不同实现  
##### 1.7 segment锁
每个segment都是个bucket集合table表，每个bucket都是个HashEntry的链表。  
![hashmap1](https://github.com/boredream/boredream.github.io/blob/master/img/in-post/hashmap1.jpg?raw=true)

##### 1.8 CAS + synchronized
默认容量、转换红黑树阈值等都和1.8HashMap一样。  
基本数据单位是Node，和HashMap一样继承自Map.Entry，但val和next都是volatile。  
setValue final 且直接报错，即只读不能写入。  

### ConcurrentHashMap源码
```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {  //onlyIfAbsent:仅仅缺少的时候


    if (key == null || value == null) throw new NullPointerException(); //key ,value不允许为null
    int hash = spread(key.hashCode()); //两次hash，减少hash冲突，可以均匀分布
    int binCount = 0; //用于记录元素个数
    for (Node<K,V>[] tab = table;;) { //对这个table进行遍历
        Node<K,V> f; int n, i, fh;
        //这里就是上面构造方法没有进行初始化，在这里进行判断，为null就调用initTable进行初始化，属于懒汉模式初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        //如果i位置没有桶，就直接无锁CAS插入
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)//如果在进行扩容转移中(当前节点是forwardingNode)，则帮助扩容
            tab = helpTransfer(tab, f);  //后面会详解
        else {
            V oldVal = null;
            //如果以上条件都不满足，那就要进行加锁操作，也就是存在hash冲突，锁住链表或者红黑树的头结点
            synchronized (f) {
                if (tabAt(tab, i) == f) { //f改变再次循环
                    if (fh >= 0) { //表示该节点是链表结构(红黑树或者正在转移都为负数)
                        binCount = 1;  
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            //这里涉及到相同的key进行put就会覆盖原先的value
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent) //只有不存在的时候才插入
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {  //插入链表尾部
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {//红黑树结构
                        Node<K,V> p;
                        binCount = 2;
                        //红黑树结构旋转插入
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }


            //binCount != 0 说明向链表或者红黑树中添加或修改一个节点成功
            //binCount  == 0 说明 put 操作将一个新节点添加成为某个桶的首节点
            if (binCount != 0) { //如果链表的长度大于8时就会进行红黑树的转换
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;  //不会调用到下面的addCount
                break;
            }
        }
    }  //for循环结束
    // 只有新增1个元素的时候才会调用这个方法
    addCount(1L, binCount);//统计size，CAS更新baseCount,并且检查是否需要扩容
    return null;
}
```
1. 先spread减少hash冲突。  
2. 死循环开始处理，落在无数据位置时，再casTabAt判断，无锁加入数据。
3. 如果是扩容转移中，helpTansfer 处理之。  
4. 其它情况，即有hash冲突，且需要对当前处理的Node进行synchronized锁处理，Synchronized 代码内正常插入数据，看情况是链表or红黑树处理。
5. 如果成功添加了数据会break结束循环，否则会继续自循环操作。


##### 什么是CAS？
一种解决并发问题的锁机制 Java CAS 锁。  
使用CAS的具体方法是casTabAt
```java
casTabAt(tab, i, null, new Node<K,V>(h, key, value, null))
```
即比较tab集合对象，在i位置的字段，也就是目标位置的要处理的数据。字段的期望值应该是null（无碰撞，该位置应该是null），新值为用新数据创建的Node对象。

##### 为什么要循环？
虽然 tabAt 是直接 getObjectVolatile 保证了可见性，casTabAt 用到了CAS 技术也保证了原子性的比较。但这俩之间可能发生数据变化，所以判断casTabAt如果失败，则继续自旋循环，再次尝试。

##### 为什么在冲突时使用 synchronized 不继续用 CAS？
非冲突情况下是对单个数据操作，可以CAS。冲突时是链表or红黑树，无法使用CAS。

##### 为什么1.8改成了 CAS+synchronized？
1.7是将容器分成多个segment碎片分别锁，1.8是针对每个Node锁，颗粒度更小，效率更高。



