---
title: 【转载】并发情况下HashMap为什么会引起死循环
date: 2019-10-21 16:10:54
tags: Java
---

今天研读Java并发容器和框架时，看到为什么要使用ConcurrentHashMap时，其中有一个原因是：线程不安全的HashMap, HashMap在并发执行put操作时会引起死循环，是因为多线程会导致HashMap的Entry链表形成环形数据结构，查找时会陷入死循环。纠起原因看了其他的博客，都比较抽象，所以这里以图形的方式展示一下

（1）当往HashMap中添加元素时，会引起HashMap容器的扩容，原理不再解释，直接附源代码，如下：

```java
	/**
    *
    * 往表中添加元素，如果插入元素之后，表长度不够，便会调用resize方法扩容
    */
   void addEntry(int hash, K key, V value, int bucketIndex) {
Entry<K,V> e = table[bucketIndex];
       table[bucketIndex] = new Entry<K,V>(hash, key, value, e);
       if (size++ >= threshold)
           resize(2 * table.length);
   }

   /**
    * resize()方法如下，重要的是transfer方法，把旧表中的元素添加到新表中
    */
   void resize(int newCapacity) {
       Entry[] oldTable = table;
       int oldCapacity = oldTable.length;
       if (oldCapacity == MAXIMUM_CAPACITY) {
           threshold = Integer.MAX_VALUE;
           return;
       }

       Entry[] newTable = new Entry[newCapacity];
       transfer(newTable);
       table = newTable;
       threshold = (int)(newCapacity * loadFactor);
   }

```

（2）参考上面的代码，便引入到了transfer方法，（引入重点）这就是HashMap并发时，会引起死循环的根本原因所在，下面结合transfer的源代码，说明一下产生死循环的原理，先列transfer代码（这是里JDK7的源码），如下：

```java
	/**
     * Transfers all entries from current table to newTable.
     */
    void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        for (Entry<K,V> e : table) {
            while(null != e) {
                Entry<K,V> next = e.next;//(1)
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];//3的next=null
                newTable[i] = e;// newTable[i]=3
                e = next;//e=3
            } // while

        }
    }

```

假如原来是是3,7,8

第一轮

```java
//当前e=3
Entry<K,V> next = e.next;//3的next=7,所以next=7
e.next = newTable[i];//3的next=newTable[i]=null  因为是第一轮,newTable里面还是空的
newTable[i] = e;// newTable[i]=3
e = next;//e=7
```

目前newTable[i]上的链表是`[3]`

第二轮

```java
//当前e=7
Entry<K,V> next = e.next;//7的next=8,所以next=8
e.next = newTable[i];//7的next=3 注意7的next=3所以相当于是7->3，相比原来反序了
newTable[i] = e;// newTable[i]=7(这里把7放到了newTable[i]这个位置的链表的首位,而7的指针指向3)
e = next;//e=8
```

目前newTable[i]上的链表是`[7]->[3]`

第三轮

```java
//当前e=8
Entry<K,V> next = e.next;//8的next=null,所以next=null,最后一轮了,因为后面null肯定通不过判断
e.next = newTable[i];//8的next=7 注意8的next=7所以相当于是8->7，相比原来反序了,而原来7指向3,所以目前是8>7>3
newTable[i] = e;// newTable[i]=8(这里把8放到了newTable[i]这个位置的链表的首位,而8的指针指向7,而7又是指向3的)
e = next;//e=null,这个位置结束
```

可以看到,进行扩容操作之后,在旧表中key值相同的数据块在新表中数据块的连接方式会**逆向**

3）假设：

```java
Map<Integer> map = new HashMap<Integer>(2);  // 只能放置两个元素，其中的threshold为1（表中只填充一个元素时），即插入元素为1时就扩容（由addEntry方法中得知）
//放置2个元素 3 和 7，若要再放置元素8（经hash映射后不等于1）时，会引起扩容

```

假设放置结果图如下:

![](1.png)

现在有两个线程A和B，都要执行put操作，即向表中添加元素，即线程A和线程B都会看到上面图的状态快照

执行顺序如下：



执行一：  线程A执行到transfer函数中（1）处挂起（transfer函数代码中有标注）。此时在线程A的栈中

```java
e = 3
next = 7
```

 执行二：线程B执行 transfer函数中的while循环，即会把原来的table变成新一table（线程B自己的栈中），再写入到内存中。如下图（假设两个元素在新的hash函数下也会映射到同一个位置）

![](2.png)



执行三： 线程A解挂，接着执行（**看到的仍是旧表**），即从transfer代码（1）处接着执行，当前的 e = 3, next = 7, 上面已经描述。

​

1 处理元素 3 ， 将 3 放入 线程A自己栈的新table中（新table是处于线程A自己栈中，是线程私有的，不被线程2的影响），处理3后的图如下：

![](3.png)

【重点在这步】 2 线程A再复制元素 7 ，当前 e = 7 ,**而next值由于线程 B 修改了它的引用，所以next 为 3** (本来7后面应该是null,但是由于线程B处理时由于hashmap扩容机制是倒序,7指向了3)，处理后的新表如下图

![](4.png)

3 由于上面取到的next = 3, 接着while循环，即当前处理的结点为3， next就为null ，退出while循环，执行完while循环后，新表中的内容如下图：

![](5.png)

4 当操作完成，执行查找时，会陷入死循环！


原文  https://blog.csdn.net/zhuqiuhui/article/details/51849692

