---
title: Mybatis原理/自己实现一个Mybatis
date: 2019-02-26 23:07:48
tags: Spring
---


# Mybatis 使用案例

有这么一张表 



![](9.png)

先简单用一下Mybatis(和Spring整合)



http://www.mybatis.org/spring/

数据源有很多，比如c3p0,druid,Spring也有内置的数据源 ,叫DriverManagerDataSource 



![](2.png)

上图的DataSource会自动注入到下面的SQLSessionFactoryBean中(它在构造函数中依赖了)

![](3.png) 



mybatis官网的截图

![](4.png) 

意思是你定义好了一个mapper接口，然后通过一个xml配置将一个MapperFactoryBean加入到Spring中

这种方式比较麻烦，你每定义一个Mapper就要声明一下 

其实有更加简便的方式，让MapperFactory自己去找mappers 

http://www.mybatis.org/spring/mappers.html#scan     

![](5.png) 

![](6.png) 



![](7.png)

![](8.png) 



![](10.png)

 



然后开始用Anno加载

上面的confi的类还需要加Spring的bean的扫描 

![](11.png)

 

等同于

![](12.png) 

![](13.png) 

AnnotationConfigApplicationContext的构造方法需传入Config的Java类 

结果:

![](14.png)

Mybatis的使用很简单，mybatis与Spring的整合也很简单   

本内容主要讲解Mybatis源码和如何实现而不是Mybatis的使用，故使用部分比较粗略



# Mybatis运行原理

Mybatis如何把一个CityDao这么一个接口变成了一个对象 ，使我们能够调用其query方法（CityService调用了CityDao的query）  ？

答案就是动态代理

![](15.png)

 

动态代理有JDK的动态代理，CGLib的动态代理，

![](16.png) 

![17](17.png) 



http://www.mybatis.org/mybatis-3/zh/getting-started.html  

SqlSession是Mybatis中的一个类   



ibatis是mybatis的前身，mybatis是google基于ibatis开发的

mybatis提供了2中执行sql语句的方式:

第一种是通过session api,需要写sql语句所在的的命名空间,加上sql语句的id ,然后传入参数 

![](20.png) 

第二种是session.getMapper,最后也会调第一种方法 

![](18.png) 



### Acquiring a SqlSession from SqlSessionFactory

Now that you have a SqlSessionFactory, as the name suggests, you can acquire an instance of SqlSession. The SqlSession contains absolutely every method needed to execute SQL commands against the database. You can execute mapped SQL statements directly against the SqlSession instance. For example:

```java
SqlSession session = sqlSessionFactory.openSession();
try {
  Blog blog = session.selectOne(
    "org.mybatis.example.BlogMapper.selectBlog", 101);
} finally {
  session.close();
}
```

While this approach works, and is familiar to users of previous versions of MyBatis, there is now a cleaner approach. Using an interface (e.g. BlogMapper.class) that properly describes the parameter and return value for a given statement, you can now execute cleaner and more type safe code, without error prone string literals and casting.

For example:

```java
SqlSession session = sqlSessionFactory.openSession();
try {
  BlogMapper mapper = session.getMapper(BlogMapper.class);
  Blog blog = mapper.selectBlog(101);
} finally {
  session.close();
}
```

Now let's explore what exactly is being executed here.



![](19.png) 



![](21.png) 

IDEA Ctrl+Alt+B可以查看所以实现接口的类,根据图17我们可以看到目前用的是SqlSessionTemplate,

![](22.png) 

selectOne先去查selectList，如果确实只有1条，那么返回否则抛出异常 .**因为你不能保证某个sql结果只有1条**，如果不是主键或者唯一，它也可能是多条。 

这里有个面试题:    **selectOne是如何实现的?**  



selectList调用了 `configuration.getMappedStatement(String statement)` 

![](23.png)

其中 String类型的statement是sql的命名空间+id

![](24.png) 

而getMappedStatement实际调用的是`mappedStatements.get(id);` 这里的id就是上一个方法传入的statement ,也就是图片中的  包名(命名空间)+方法名(id) 

![](25.png) 

![](26.png) 



**这里的mappedSatement是一个key为String，value为MappedStatement的Map** ，而我们传入的所谓的id是命名空间+id的字符串,举例就是上面的`com.dao.CityDao.query` 

其实这个map就是`Map<ID,SQL语句>`的Map ,上面 的 `mappedStatements.get(id)` 就是为了拿到sql语句  

面试题：为什么方法名要和mapper.xml中的sql的id一样？ 

因为mybatis底层在初始化的时候是拿一个Map将id与sql语句对应起来(其实就是上面的**MappedStatement**), 需要拿到这个id作为key去调用这个方法的  ,如果不一致，会找不到 

![](27.png) 



![](28.png) 





好了，获得了sql语句之后继续往下走 

![29](29.png) 



![](30.png) 

那么DefaultSqlSession用的是哪个Executor实现类呢？ 

![](31.png) 

为什么用这个实现类呢？

因为Mybatis和Hibernate一样，自带一级缓存 ，但是**这个一级缓存在Spring中是失效的**。  

为什么？

![](32.png) 

因为当和Spring整合之后，有个叫SqlSessionFactoryBean的东西，用来替代SqlSessionFactoryBuilder来创建SqlSession。

这个SQLSessionFactoryBean是交给Spring去管理了，Spring当请求结束后会把session关掉，所以一级缓存没用了   



大家在用Spring的时候有去主动关过Session吗？没有吧，都是Spring在帮你关的  ，每个方法一旦执行完了就帮你关了，Spring的AOP帮你关的 



如果是自己来管理session，只要不关闭session，缓存就在

![](33.png) 

接下来讲解图29中的executor.query方法

![34](34.png) 



![](35.png) 



![](36.png)

 

![](37.png) 



![](38.png) 



org.mybatis.mybatis-spring.jar包中有一个 MapperScannerRegistrar.java的scanner.doscan会扫描 mapper  



![](39.png) 



mybatis在启动的时候就会进行mapper扫描 

如下图所示，有2个断点，一个打在了service.query上，一个打在了scanner.doScan上，经过实际调试发现，先到了doScan这个断点上了  

![](40.png) 



![](42.png) 

循环扫描，因为ComponentScan可以指定多个包 

![](41.png)

 



![](43.png) 



![](44.png) 



注意上图中packageSearchPath是`com/dao/**/*.class` 然后通过反射技术拿到这个类然后加载到内存中来

`Class.forName(xxx.class) ` 

但是Spring怎么知道你这个目录(com/dao)下有多少class呢？ 

Spring就是通过class的绝对路径然后+/com+/dao找到文件夹，然后扫描文件夹的方式来找的 

找到之后new出来，就拿到这些mapper了 



过程总结： 

1 初始化 ，扫描mapper(如何扫描?绝对路径+包名扫描文件夹下所有class)，初始化数据库连接

2  把statement放到MapperedStatement这个Map中 

3 sqlsession调用selectList方法,

4 sqlsession调用selectList方法底层调用executor的接口，这个接口有很多实现类，有CachedExecotor和DefaultExecutor，MyBatis默认用的CachedExecutor

5 CachedExecutor调用的JDBC底层的prepareStatement语句 

# 自己实现一个Mybatis

![](46.png) 



![](47.png) 



![](48.png) 



升级版，可以向mybatis一样扫描mapper，然后建立一个有包名+方法名作为id，sql作为value的一个Mapperedstatement的Map，然后代理的时候可以根据包名和方法名获取具体执行的sql语句

![](49.png) 


# 部分源码




DefaultSqlSession.java
```
package org.rico.learnMybatis.test;

/**
 * Created by Rico on 2018/12/18.
 */
public class DefaultSqlSession {
    //最原始的
    public void selectList(){
        System.out.println("select list");
    }
    //可以看到sql了
    public void selectList(String id){
        String sql=Test.mapperedStatement.get(id);
        System.out.println("根据statementid获取到了具体的sql:"+sql);
    }
}

```


UserDaoInvocationHandler.java 
```
package org.rico.learnMybatis.test;
import org.rico.learnMybatis.dao.UserDao;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
/**
 * Created by Rico on 2018/12/18.
 */
public class UserDaoInvocationHandler implements InvocationHandler{
    UserDao userDao;
    DefaultSqlSession sqlSession;
    UserDaoInvocationHandler(DefaultSqlSession sqlSession,UserDao userDao){
        this.sqlSession=sqlSession;
        this.userDao=userDao;
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("开始代理了");
        String id=method.getDeclaringClass().toString().replace("interface","").trim();
        id+="."+method.getName();
        System.out.println(id);
        sqlSession.selectList(id);
        Object o= method.invoke(this.userDao,args);
        System.out.println("代理结束了");
        return o;
    }
}

```

UserDao.java

```
package org.rico.learnMybatis.dao;
import org.rico.learnMybatis.annotation.Select;
/**
 * Created by Rico on 2018/12/18.
 */
public interface  UserDao {
    @Select("select * from user where id = 1;")
    public void query();
}

```


Test.java
```
package org.rico.learnMybatis.test;
import org.rico.learnMybatis.annotation.Select;
import org.rico.learnMybatis.dao.UserDao;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

/**
 * Created by Rico on 2018/12/18.
 */
public class Test {
    public static void main(String[] args) {
        //模拟Mybatis初始化的时候扫描Mapper的操作
        init();
        UserDao userDao =null;//Spring把这个UserDao实现了然后赋给接口UserDao,然后我们就可以真正的拿来用了
        DefaultSqlSession defaultSqlSession=new DefaultSqlSession();
        UserDaoInvocationHandler userDaoInvocationHandler=new UserDaoInvocationHandler(defaultSqlSession, new UserDao() { @Override public void query() { } });
        //第一个参数:ClassLoader,通过哪个ClassLoader来把class加载进来
        //第二个参数:代理哪些接口?
        //第三个参数InvocationHandler:代理接口的哪些方法
        userDao=(UserDao)Proxy.newProxyInstance(UserDao.class.getClassLoader(),new Class[]{UserDao.class},userDaoInvocationHandler);
        userDao.query();
    }

    public static Map<String,String> mapperedStatement=new HashMap<String,String>();

    public static void init(){
        try {
            Class clazz=Class.forName("org.rico.learnMybatis.dao.UserDao");
            Method[] methods = clazz.getDeclaredMethods();
            for(Method m:methods){
                if(m.isAnnotationPresent(Select.class)) {//是否加了@Select的注解
                    Select select=m.getAnnotation(Select.class);//拿到@Select注解中的值
                    String value=select.value();
                    String key=m.getDeclaringClass().getName()+"."+m.getName();
                    mapperedStatement.put(key,value);
                }
            }
            System.out.println(mapperedStatement);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}

```

Select.java  这个是定义注解

```
package org.rico.learnMybatis.annotation;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
/**
 * Created by Rico on 2018/12/18.
 */
@Target(ElementType.METHOD)//方法级别的
@Retention(RetentionPolicy.RUNTIME)//运行时级别
public @interface Select {
    public String value();
}

```


目录结构  
![](50.png)


源码待上传github
