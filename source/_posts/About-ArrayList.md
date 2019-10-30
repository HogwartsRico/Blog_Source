---
title: ArrayList常用方法和实现原理
date: 2015-04-08 21:30:19
tags: Java基础
---

#  ArrayList常用方法

ArrayList常用的方法和Collection基本一样 

**1.add("AA");向集合中添加一个元素**

```java
public static void main(String[] args) {
	List<String> list = new ArrayList<String>();
	list.add("AA");
}
```

**2.add(2, "CC");将指定的元素插入此列表中的指定位置。索引从0开始**

```java
public static void main(String[] args) {
	List<String> list = new ArrayList<String>();
	list.add("AA");
	list.add("BB");
	list.add("DD");
	System.out.println(list);
	list.add(2, "CC");
	System.out.println(list);
}
```

结果为：

```
[AA, BB, DD]
[AA, BB, CC, DD]

```

**3.addAll(list2);将形参中的集合元素全部添加到当前集合中的尾部。**

```java
public static void main(String[] args) {
	List<String> list = new ArrayList<String>();
	list.add("AA");
	list.add("BB");
	list.add("DD");
	List<String> list2 = new ArrayList<String>();
	list2.add("CC");
	list2.add("EE");
	list.addAll(list2);
	System.out.println(list);
}
```

结果为：

```
[AA, BB, DD, CC, EE]

```

**4.addAll(2, list2);将形参中集合元素添加到当前集合指定的位置。**

```java
public static void main(String[] args) {
	List<String> list = new ArrayList<String>();
	list.add("AA");
	list.add("BB");
	list.add("DD");
	List<String> list2 = new ArrayList<String>();
	list2.add("CC");
	list2.add("EE");
	list.addAll(2, list2);
	System.out.println(list);
}
```

结果为：

```
[AA, BB, CC, EE, DD]

```

**5.clear();清空当前集合中所有元素**

```
list.clear();

```

注意：以下方法和Collection一样，都依赖元素对象的equals方法。更多的时候都需要重写equals方法。

 **6.contains("aa");返回当前 元素是否包含某一个对象。当前放回false。**

```java
public static void main(String[] args) {
	List<String> list = new ArrayList<String>();
	list.add("AA");
	list.add("BB");
	list.add("DD");
	boolean contains = list.contains("aa");
	System.out.println(contains);
}
```

**7.get(1);获取当前集合中指定位置的元素，这里返回BB**

```java
public static void main(String[] args) {
	List<String> list = new ArrayList<String>();
	list.add("AA");
	list.add("BB");
	list.add("DD");
	String string = list.get(1);
	System.out.println(string);
}
```

**8.indexOf("BB");返回当前集合中首次出现形参对象的位置，如果集合中不存在就返回-1.**

```java
public static void main(String[] args) {
	List<String> list = new ArrayList<String>();
	list.add("AA");
	list.add("BB");
	list.add("BB");
	int indexOf = list.indexOf("BB");
	System.out.println(indexOf);
}
```

简单的方法就查询JDK API文档吧，下面作简要的说明：

1.`size();`返回当前集合元素个数.   

2.`isEmpty();`判断集合是否为空，返回布尔类型的结果。  

3.`lastIndexOf(Object o);`返回集合中最后一次出现形参元素的索引，不存在就返回-1。 

4.`toArray();`将集合转换为数组   

5.`set(int index,E element);`用指定元素替代集合中指定位置的元素。   

6.`remove(Object o);`  **移除集合中首次出现的元素**。   

7.`remove(int index);`移除集合中指定位置的元素。  

# ArrayList实现原理

ArrayList是List接口的可变数组的实现，允许包括null在内的所有元素，既然是数组，那么该类肯定会存在改变存储列表的数组大小的方法。每一个ArrayList实例都有一个容量，该容量是用来存储列表元素的数组的大小，**他总是等于列表的大小**，随着往ArrayList中添加元素，这个容量也会相应的总动增长，**自动增长就会带来数据向新数组的重新拷贝**。  

## 1.底层使用数组实现

```java
 private transient Object[] elementData; 
```

关于关键字**transient**: 

一个对象只要实现了Serilizable接口，这个对象就可以被序列化。 

不需要序列化的属性前添加关键字transient，序列化对象的时候，这个属性就不会序列化

## 2.构造方法：

```java
	/**
     * Constructs an empty list with an initial capacity of ten.
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```

其中 

```java
 private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
```

可以看出它的默认数组为长度为0。而在之前JDK1,6中，无参数构造器代码是初始长度为10。 
JDK6代码这样的：

```java
 public ArrayList() {
    this(10);
}
 public ArrayList(int initialCapacity) {//可以指定初始大小
    super();
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Capacity: "+ initialCapacity);
    this.elementData = new Object[initialCapacity];
}
 public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    size = elementData.length;
    // c.toArray might (incorrectly) not return Object[] (see 6260652)
    if (elementData.getClass() != Object[].class)
        elementData = Arrays.copyOf(elementData, size, Object[].class);
}
```

 

## 3.如何实现存储和扩容的？

### 3.1  使用set(int index, E element) ;方法，用指定的元素替代指定位置的元素

```java
public E set(int index, E element) {
    rangeCheck(index);
    E oldValue = elementData(index);
    elementData[index] = element;
    return oldValue;
}
```

###  3.2  add(E e);如何扩容的？ 

将指定的元素添加到集合尾部

```java
	public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
```

代码中关于size含义的源码注释  

` The size of the ArrayList (the number of elements it contains).` 意思就是目前ArrayList中实际包含的元素数量  不一定等于elementData的长度 一般小于等于elementData的长度 ,elementData长度会大于等于size 

往下看就知道了 

ArrayList中的`size()`方法返回的值就是size  

```java
	/**
     * Returns the number of elements in this list.
     *
     * @return the number of elements in this list
     */
    public int size() {
        return size;
    }
```



ensureCapacityInternal方法名的英文大致是“确保内部容量”，size表示的是执行添加之前的实际元素个数，并非ArrayList的容量，容量是数组elementData的长度。ensureCapacityInternal该方法通过将现有的元素个数数组的容量比较。看如果需要扩容，则扩容。  

下面具体看 ensureCapacityInternal(size + 1);

```java
//如何判断和扩容的
private void ensureCapacityInternal(int minCapacity) {
      //如果目前实际存储数组 是空数组，则最小需要容量为  默认容量(10)和0+1=1之间最大值,也就是10,其中DEFAULT_CAPACITY=10
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity) {
      //当第一次插入值时，由于minCapacity 一定大于等于10 而elementData.length是0  所以需要扩容
	  //如果不是第一次插入则是size+1  
        modCount++;
        //如果数组（elementData)的长度小于最小需要的容量（minCapacity）就扩容,如果还够用则不扩容
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

    /**
     * Default initial capacity.
     */
    private static final int DEFAULT_CAPACITY = 10;
```



```java
	/*
    *增加容量，以确保它至少能容纳
    *由最小容量参数指定的元素数。
    * @param mincapacity所需的最小容量
    */
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
//>>位运算，右移动一位。 整体相当于newCapacity =oldCapacity + 0.5 * oldCapacity  即旧elementData数组长度的1.5倍 jdk1.7采用位运算比以前的计算方式更快
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
       //jdk1.7这里增加了对元素个数的最大个数判断,jdk1.7以前是没有最大值判断的，MAX_ARRAY_SIZE 为int最大值减去8  最大的容量之所以设为Integer.MAX_VALUE - 8，在定义上方的注释中已经说了，大概是一些JVM实现时，会在数组的前面放一些额外的数据，再加上数组中的数据大小，有可能超过一次能申请的整块内存的大小上限，出现OutOfMemoryError。
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // 最重要的复制元素方法  把原来数组的数据复制到新的数组中。调用了Arrays的copyOf方法。内部是System的arraycopy方法，由于是native方法，所以效率较高。
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```

如果新的容量大于数组的最大值MAX_ARRAY_SIZE 

这时，又会进入另一个方法hugeCapacity

```java
	private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```

会对minCapacity 和 MAX_ARRAY_SIZE进行比较，minCapacity 大的话，就将Integer.MAX_VALUE 作为新数组的大小，否则将MAX_ARRAY_SIZE作为数组的大小。 



通过分析可以发现，ArrayList的扩容会产生一个新的数组，将原来数组的值复制到新的数组中。会消耗一定的资源。**所以我们初始化ArrayList时，最好可以估算一个初始的大小**。 而且就算你之后往ArrayList中添加了超过数量的元素也不会报错，会自动扩容   

经过打印输出，我们可以看到

**向数组中添加第一个元素时，数组容量为10.**  

**向数组中添加到第10个元素时，数组容量仍为10.**  

**向数组中添加到第11个元素时，数组容量扩为15.** 

**向数组中添加到第16个元素时，数组容量扩为22.**  

对比执行时间要使用控制变量法，先后执行所使用的时间会有不同。

如：

```java
public class Test {
	private static void test(int n) {
        long start = System.currentTimeMillis();
        List<Integer> list = new ArrayList<Integer>(n);
        for (int i = 0; i < n; i++) {
            list.add(i);
        }
        long end = System.currentTimeMillis();
        System.out.println(end-start);
    }
	private static void test0(int n) {
        long start = System.currentTimeMillis();
        List<Integer> list2 = new ArrayList<Integer>();
        for (int i = 0; i < n; i++) {
            list2.add(i);
        }
        long end = System.currentTimeMillis();
        System.out.println(end-start);
    }
	 public static void main(String[] args) {
		 	test(10000000);
		 	test0(10000000);
	    }
}
```

输出 : 

2412
3898

明显可以看出初始化长度比没有初始化长度节省时间。



###  3.3 add(int index, E element)将指定的元素插入此列表中的指定位置

```java
public void add(int index, E element) {  
    if (index > size || index < 0)  
        throw new IndexOutOfBoundsException("Index: "+index+", Size: "+size);  
    // 如果数组长度不足，将进行扩容。  
    ensureCapacity(size+1);  // Increments modCount!!  
    // 将 elementData中从Index位置开始、长度为size-index的元素，  
    // 拷贝到从下标为index+1位置开始的新的elementData数组中。  
    // 即 将当前位于该位置的元素以及所有后续元素右移一个位置。  
    System.arraycopy(elementData, index, elementData, index + 1, size - index);  
    elementData[index] = element;  
    size++;  
}  
```



System.arraycopy

```java
 public static native void arraycopy(Object src,  int  srcPos, Object dest, int destPos,  int length);
```

参数注释

 * @param      src      the source array. 源数组
 * @param      srcPos   starting position in the source array. 源数组的位置
 * @param      dest     the destination array.目标数组
 * @param      destPos  starting position in the destination data.目标数组的位置
 * @param      length   the number of array elements to be copied. 数量

## 4.如何读取元素？

```java
  public E get(int index) {
    rangeCheck(index);
    return elementData(index);
}
```

## 5.删除元素

```java
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
```

注意：从集合中移除一个元素，如果这个元素不是最后一个元素，那么这个元素的后面元素会向向左移动一位。

 ```java
	/*
     * Private remove method that skips bounds checking and does not
     * return the value removed.
     */
    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
          //将当前位于该位置的元素以及所有后续元素左移一个位置。
          //将index后边的对象往前复制一位，并将数组中的最后一位元素设置为null 不置为null的话最后2个元素值是一模一样的
            System.arraycopy(elementData, index+1, elementData, index, numMoved);
        elementData[--size] = null; // clear to let GC do its work 让GC来回收 
    }
 ```



##  6.手动调整底层数组的容量为列表实际元素大小的方法

```java
  public void trimToSize() {
    modCount++;
    int oldCapacity = elementData.length;
    if (size < oldCapacity) {
        elementData = Arrays.copyOf(elementData, size);
    }
}
```



另外有个高频率出现的属性但是我们没有提过它就是modCount ,它是当前这个ArrayList被修改的次数

JDK注释

```java
	/**
     * The number of times this list has been <i>structurally modified</i>.
     * Structural modifications are those that change the size of the
     * list, or otherwise perturb it in such a fashion that iterations in
     * progress may yield incorrect results.
     *
     * <p>This field is used by the iterator and list iterator implementation
     * returned by the {@code iterator} and {@code listIterator} methods.
     * If the value of this field changes unexpectedly, the iterator (or list
     * iterator) will throw a {@code ConcurrentModificationException} in
     * response to the {@code next}, {@code remove}, {@code previous},
     * {@code set} or {@code add} operations.  This provides
     * <i>fail-fast</i> behavior, rather than non-deterministic behavior in
     * the face of concurrent modification during iteration.
     *
     * <p><b>Use of this field by subclasses is optional.</b> If a subclass
     * wishes to provide fail-fast iterators (and list iterators), then it
     * merely has to increment this field in its {@code add(int, E)} and
     * {@code remove(int)} methods (and any other methods that it overrides
     * that result in structural modifications to the list).  A single call to
     * {@code add(int, E)} or {@code remove(int)} must add no more than
     * one to this field, or the iterators (and list iterators) will throw
     * bogus {@code ConcurrentModificationExceptions}.  If an implementation
     * does not wish to provide fail-fast iterators, this field may be
     * ignored.
     */
    protected transient int modCount = 0;
```
