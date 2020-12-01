---
title: WeakReference
date: 2020-11-30 20:36:38
tags: Java
---


# 简介

在看[ThreadLocal源码](https://hogwartsrico.github.io/2020/11/19/The-use-and-implementation-of-ThreadLocal/)的时候，其中嵌套类ThreadLocalMap中的Entry继承了WeakReferenc，为了能搞清楚ThreadLocal，只能先了解下了WeakReferenc(是的，很多时候为了搞清楚一个东西，不得不往上追好几层，先搞清楚其所依赖的东西。) 
WeakReference如字面意思，弱引用.

 **当一个对象仅仅被weak reference指向, 而没有任何其他strong reference指向的时候, 如果GC运行, 那么这个对象就会被回收**, 不论当前的内存空间是否足够，这个对象都会被回收。

# 认识WeakReference类

 WeakReference继承Reference，其中只有两个构造函数： 

```java
public class WeakReference<T> extends Reference<T> {
    public WeakReference(T referent) {
        super(referent);
    }

    public WeakReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }
}
```

`WeakReference(T referent)`：referent就是被弱引用的对象（注意区分弱引用对象和被弱引用的对应，弱引用对象是指WeakReference的实例或者其子类的实例），比如有一个Apple实例apple，可以如下使用，并且通过get()方法来获取apple引用。也可以再创建一个继承WeakReference的类来对Apple进行弱引用，下面就会使用这种方式。

```apple
WeakReference<Apple> appleWeakReference = new WeakReference<>(apple);
Apple apple2 = appleWeakReference.get();
```

`WeakReference(T referent, ReferenceQueue<? super T> q)`：与上面的构造方法比较，多了个ReferenceQueue，在对象被回收后，会把弱引用对象，也就是WeakReference对象或者其子类的对象，放入队列ReferenceQueue中，注意不是**被**弱引用的对象，**被**弱引用的对象已经被回收了。

# 实战

 下面是使用继承WeakReference的方式来使用软引用，并且不使用ReferenceQueue。 

````java
public class Fruit {
    private String name;
    public Fruit(String name) { this.name = name; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    /**
     * 覆盖finalize，在回收的时候会执行。
     */
    @Override
    protected void finalize() throws Throwable {
        super.finalize();
        System.out.println("Fruit" + name + " finalize。");
    }
    @Override
    public String toString() {
        return "Fruit{" + "name='" + name + '\'' + '}' + ", hashCode:" + this.hashCode();
    }
}
````

```java
public static void main(String[] args) {
        WeakReference<Fruit> weakReference = new WeakReference<>(new Fruit("红富士"));//注意到时候回收的是Fruit，而不是Apple
        //通过WeakReference的get()方法获取Apple
        System.out.println("调用weakference的toString:"+ weakReference.get());
        System.out.println("调用weakference的getName:"+ weakReference.get().getName());
        System.gc();
        try {
            //休眠一下，在运行的时候加上虚拟机参数-XX:+PrintGCDetails，输出gc信息，确定gc发生了。
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        //如果为空，代表被回收了
        if (weakReference.get() == null) {
            System.out.println("弱引用已经被回收");
        }
    }
```

![](1.png) 

可以看到,weakReference这个对象还在，但是它引用的Fruit已经没有了 。 

还可以使用继承WeakReference的方式来使用软引用 

沙拉引用了水果

```java
/**
 * Salad class  继承WeakReference，将Apple作为弱引用。
 * 注意到时候回收的是Fruit，而不是Salad
 */
public class Salad extends WeakReference<Fruit> {
    public Salad(Fruit fruit) {
        super(fruit);
    }
}
```



```java
 public static void main(String[] args) {
        Salad salad = new Salad(new Fruit("红富士"));//注意到时候回收的是Fruit，而不是Salad
        //通过WeakReference的get()方法获取Apple
        System.out.println("调用salad的toString:"+ salad.get());
        System.out.println("调用salad的getName:"+ salad.get().getName());
        System.gc();
        try {
            //休眠一下，在运行的时候加上虚拟机参数-XX:+PrintGCDetails，输出gc信息，确定gc发生了。
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        //如果为空，代表被回收了
        if (salad.get() == null) {
            System.out.println("水果被回收了");
        }
    }
```

![](2.png) 

可以看到，沙拉还在，水果已经没有了。 



# ReferenceQueue的使用

```java
 public static void main(String[] args) {
        ReferenceQueue<Fruit> fruitReferenceQueue = new ReferenceQueue<>();
        WeakReference<Fruit> fruitWeakReference = new WeakReference<Fruit>(new Fruit("草莓"), fruitReferenceQueue);
        WeakReference<Fruit> fruitWeakReference2 = new WeakReference<Fruit>(new Fruit("红苹果"), fruitReferenceQueue);
        System.out.println("=====gc调用前=====");
        Reference<? extends Fruit> reference = null;
        while ((reference = fruitReferenceQueue.poll()) != null ) {
            //不会输出，因为没有回收被弱引用的对象，并不会加入队列中
            System.out.println(reference);
        }
        System.out.println(fruitWeakReference);
        System.out.println(fruitWeakReference2);
        System.out.println(fruitWeakReference.get());
        System.out.println(fruitWeakReference2.get());

        System.out.println("=====调用gc=====");
        System.gc();
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("=====gc调用后=====");

        //下面两个输出为null,表示对象被回收了
        System.out  .println(fruitWeakReference.get());
        System.out.println(fruitWeakReference2.get());

        //输出结果，并且就是上面的appleWeakReference、appleWeakReference2，再次证明对象被回收了
        Reference<? extends Fruit> reference2 = null;
        while ((reference2 = fruitReferenceQueue.poll()) != null ) {
            //如果使用继承的方式就可以包含其他信息了
            System.out.println("fruitReferenceQueue中：" + reference2);
        }
    }
```

![](3.png) 

