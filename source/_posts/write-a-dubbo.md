---
title: 自己实现一个Dubbo
date: 2018-12-20 21:53:40
tags: RPC
---


# Dubbo 

Dubbo是一款高性能轻量级的开源Java RPC框架

什么是RPC？

远程过程调用。

![](1.png) 

 HTTP也是一种RPC协议 TCP也是  

![](2.png) 

### 传统的HTTP调用

![](3.png) 

需要关心地址，地址变了消费端也要跟着变 ，需要字符串转换等缺点

### RPC式的调用

![](4.png) 

consumer配置

![](5.png) 

provider配置

![](6.png) 



 特别简单，作为消费者不需要关心地址

dubbo可扩展，它支持多种协议

![](7.png) 





# 自己实现

![](8.png)

 

本项目就不采用模块的形式了（可以采用模块的形式） ，采用包代替模块的方式 



![](9.png) 

我们注册中心一般用zookeeper或者eureka 或者redis  ,注册中心也是可以扩展的 

我们自己实现一个注册中心

注册中心要实现的是保存服务配置，需要保存:  服务名, url, 服务的实现类

形如

```json
{
 服务名://因为可能有多个提供者
   {URL:服务实现类},
   {URL:服务实现类}
}
```

我们用hashmap来存 



一个Java对象要在网络中进行传输，需要序列化 

# 部分源码

## 注册中心

采用一个hashmap来充当注册中心. 注册中心是独立于服务提供方和服务消费方的，但是这里为了省事，放在同一个工程中了   


```java
package org.rico.learnDubbo.register;
import org.rico.learnDubbo.framework.URL;
import java.util.HashMap;
import java.util.Map;
/**
 * 自己实现的一个注册中心,采用map
 * Created by Rico on 2018/12/20.
 */
public class RegisterServer {
    private static Map<String,Map<URL,Class>>   REGISTER =new HashMap<String,Map<URL,Class>>();

    //注册服务
    public static  void register(URL url,String interfaceName,Class implClass){//需要传入URL，服务名，接口名(interface),实现类
        Map<URL,Class> map=new HashMap<URL,Class>();
        map.put(url,implClass);
        System.out.println("服务注册成功!注册的服务接口名为:"+interfaceName+",所在接口类为:"+implClass.getName());
        REGISTER.put(interfaceName,map);
    }

    public static Class get(URL url,String interfaceName){
        return REGISTER.get(interfaceName).get(url);
    }
    //获取可用服务
    public static  URL get(String interfaceName){
      return null;
    }
}
```

## 传输协议
http


```
package org.rico.learnDubbo.protocol.http;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
/**
 * Created by Rico on 2018/12/20.
 */
public class DispatcherServlet extends HttpServlet {
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        //super.service(req, resp);
        new HttpServerHandler().handler(req,resp);
    }
}
```

```
package org.rico.learnDubbo.protocol.http;

import org.apache.commons.io.IOUtils;
import org.rico.learnDubbo.framework.TransferModel;
import org.rico.learnDubbo.framework.URL;
import org.rico.learnDubbo.register.RegisterServer;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.InputStream;
import java.io.ObjectInputStream;
import java.io.OutputStream;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

/**
 * Created by Rico on 2018/12/20.
 */
public class HttpServerHandler{
    //处理请求和响应
    public void handler(HttpServletRequest req, HttpServletResponse resp){
        //对象通过网络传输
        InputStream inputStream= null;
        try {
            inputStream = req.getInputStream();
            ObjectInputStream objectInputStream=new ObjectInputStream(inputStream);
            TransferModel  transferModel =(TransferModel)objectInputStream.readObject();//二进制数据反序列化成一个对象
            //根据transferModel中的信息去注册中心找接口的实现类
            String interfaceName=transferModel.getInterfaceName();
            URL url=new URL("localhost",8080);
            Class implementClass=RegisterServer.get(url,interfaceName);//根据url和接口名找一个实现类
            //然后new出这个实现类,然后执行一下需要调用的这个方法
            Method method=implementClass.getMethod(transferModel.getMethodName(),transferModel.getParamTypes());//这里Class的getMethod方法的第一个参数:方法的名字,第二个参数,方法的参数类型
            //执行,并保存返回值
            String result=(String)method.invoke(implementClass.newInstance(),transferModel.getParams());//第一个参数:new出来的这个实现类,第二个参数:参数 我们这里因为知道sayHello方法返回的是String，所以强转成String了,实际dubbo框架肯定是Object
            //然后把结果返回,通过outputstream返回给resp
            OutputStream outputStream=resp.getOutputStream();
            IOUtils.write(result,outputStream);//把字符串给output了 这样就完成了一个结果的返回
            //server端就写完了
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            System.out.println("找不到这个类");
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            System.out.println("没有这个方法");
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
    }
}

```

```
package org.rico.learnDubbo.protocol.http;

import org.apache.catalina.*;
import org.apache.catalina.connector.Connector;
import org.apache.catalina.core.StandardContext;
import org.apache.catalina.core.StandardEngine;
import org.apache.catalina.core.StandardHost;
import org.apache.catalina.startup.Tomcat;

/**
 * Created by Rico on 2018/12/20.
 */
public class HttpServer {
    public void start(String hostname,Integer port){
        Tomcat tomcat=new Tomcat();

        Server server=tomcat.getServer();
        Service service=server.findService("Tomcat");//默认 叫tomcat

        Connector connector=new Connector();
        connector.setPort(port);

        Engine engine=new StandardEngine();
        engine.setDefaultHost(hostname);

        Host host=new StandardHost();
        host.setName(hostname);

        String contextPath="";
        Context context=new StandardContext();
        context.setPath(contextPath);
        context.addLifecycleListener(new Tomcat.FixContextListener());//加一个tomcat生命周期的监听器

        host.addChild(context);
        engine.addChild(host);

        service.setContainer(engine);
        service.addConnector(connector);
        //tomcat是servlet的容器 所以要加入servlet,servlet用来处理请求的
        tomcat.addServlet(contextPath,"dispatcher",new DispatcherServlet());//指定servlet,第二个参数是servlet的名字，第三个参数是指定其实现类
        context.addServletMappingDecoded("/*","dispatcher");//mapping
        try {
            tomcat.start();//只是初始化
            tomcat.getServer().await();//接收请求
        } catch (LifecycleException e) {
            e.printStackTrace();
        }
    }
}

```


```
package org.rico.learnDubbo.protocol.http;
import org.apache.commons.io.IOUtils;
import org.rico.learnDubbo.framework.TransferModel;
import java.io.IOException;
import java.io.InputStream;
import java.io.ObjectOutputStream;
import java.io.OutputStream;
import java.net.HttpURLConnection;
import java.net.MalformedURLException;
import java.net.ProtocolException;
import java.net.URL;
/**
 * Created by Rico on 2018/12/21.
 */
public class HttpClient {
    public String post(String hostname, Integer port, TransferModel transferModel) {
        try {
            //发送请求
            URL url = new URL("http", hostname, port, "/");
            HttpURLConnection httpURLConnection = (HttpURLConnection) url.openConnection();
            httpURLConnection.setRequestMethod("POST");
            httpURLConnection.setDoOutput(true);
            OutputStream outputStream = httpURLConnection.getOutputStream();
            ObjectOutputStream objectOutputStream = new ObjectOutputStream(outputStream);
            objectOutputStream.writeObject(transferModel);
            objectOutputStream.flush();
            objectOutputStream.close();
            //发送完请求,然后拿到结果
            InputStream inputStream=httpURLConnection.getInputStream();
            return IOUtils.toString(inputStream);
        } catch (MalformedURLException e) {
            e.printStackTrace();
            return "";
        } catch (ProtocolException e) {
            e.printStackTrace();
            return "";
        } catch (IOException e) {
            e.printStackTrace();
            return "";
        }
    }
}
```


未完，待续....

目录结构
![](10.png)





下面2张图帮助理解源码

![](11.png) 

![](12.png)  



[源码下载](mydubbo.zip)   

