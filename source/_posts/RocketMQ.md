---
title: RocketMQ系列
date: 2020-08-17 10:16:14
tags: JAVA
---


读完RocketMQ系列将学会已下内容： 



* 掌握RokectMQ集群搭建

* 生产者：解决消息同步/异步，延迟，顺序等问题

* 消费者： 真正理解Offset存储机制,消费端充实，幂等策略

* 核心原理，进阶掌握高可用机制，协调服务，刷盘赋值策略

* 分流/限流，缓存路由，负载均衡，分库分表，抗压点分析

# RocketMQ 初探门径

## RocketMQ整体认知

RocketMQ是一款分布式，队列模型的消息中间件   

4.3.0版本之后开源事务消息的代码 

截止2019年03月31日10:07:59最新版本为4.4.0 

* 最新版支持**分布式事务**  
* 天然支持集群模式，另外还支持负载均衡，水平扩展能力，支持广播模型 
* 亿级别的消息堆积能力  
* 采用零拷贝的原理，顺序写盘，随机读
* 丰富的API(同步消息，异步消息，顺序消息，延迟消息，事务消息) 
* 代码优秀，底层通讯框架采用Netty
* NameServer代替Zookeeper(自己造轮子，相比zk更轻量) 
* 强调集群无单点，可扩展，任意一点高可用，水平可扩展  
* 消息失败重试机制，消息可查询 (RabbitMQ消息失败是不支持重试的，rocketmq如果失败了，broker负责重发)
* 开源社区活跃，成熟度高

## RocketMQ概念模型

* Producer : 消息生产者，负责生产消息,一般由业务系统负责产生消息

* Consumer：消息消费者，负责消费消息，一般是后台系统负责异步消费

* Push Consumer(推送的Consumer):Consumer的一种，需要向Consumer对象注册监听，监听mq向我们推消息 
* Pull Consumer(拉取消息):Consumer的一种，需要主动请求Broker拉取消息 
* Produce Group 生产者集合 ，一般用于发送一类消息。生产者可以有多个，启动时候可以定义多个节点同一个组。在RocketMQ中生产者组合消费者组是必须要定义的，并且一个环境只能起一个相同名称的group  
* Consumer Group：消费者集合。 一般用于接受一类消息进行消费。这个比较好理解，比如三个消费者组成一个组，那么**可以设置成发过来的消息都接受到，或者负载均衡各自接受消息(互相不重复)**  (集群模式或广播模式)
* Broker:MQ消息服务(中转角色，用于消息存储与生产消费转发) 



## RocketMQ源码包编译

这里我们使用4.4.0 ,因为官网4.5.0的包下载不了

https://www.apache.org/dyn/closer.cgi?path=rocketmq/4.4.0/rocketmq-all-4.4.0-source-release.zip

编译很简单 ,[quickstart](http://rocketmq.apache.org/docs/quick-start/)  

```
	unzip rocketmq-all-4.4.0-source-release.zip
  > cd rocketmq-all-4.4.0/
  > mvn -Prelease-all -DskipTests clean install -U
  > cd distribution/target/apache-rocketmq
```



编译中 

![](11.png) 

编译完成
![](14.png) 



![](12.png) 



## 源码包结构介绍

![](13.png) 



rocketmq-broker 主要的业务逻辑，消息收发，主从同步，packagecache

rocketmq-client 客户端接口，比如生产者和消费者

rocketmq-example 示例，比如生产者消费者 

rocketmq-common 公用数据结构

rocketmq-distribution 编译模块，编译输出

rocketmq-filter  进行Broker过滤的不感兴趣的消息传输，减小带宽压力 

rocketmq-logappender,rocketmq-logging 日志相关  

rocketmq-nameserv :  Nameserver服务，用于进行服务协调

rocketmq-openmessaging 对外提供服务 

rocketmq-remoting  远程调用接口，封装Netty底层通信 ，最底层的远程通信  

rocketmq-srvutil   提供一些公用的工具方法，比如解析命令行参数  

rocketmq-store 消息存储

rocketmq-test,rocketmq-example

rocketmq-tools 管理工具，比如有名的mqadmin工具 



编译完成后tar.gz包就在distribution/target目录下，另外rocketmq-distribution 

![](15.png) 

rockeqmq-distribution/conf目录里面是一些配置  

![](16.png) 



rocketmq-client主要提供客户端api

### 环境搭建

JDK1.8+

RocketMQ 4.3.X



nameserver用于协调，broker是服务器   



我们先用单点，ip是192.168.0.111

### 1 编辑/etc/hosts

```shell
192.168.0.111 rocketmq-nameserver1
192.168.0.111 rocketmq-master1
```

s

### 2 上传，解压程序,建立软连接 

```shell
tar -zxvf apache-rocketmq.tar.gz  -C  /usr/local/apache-rocketmq
#然后在/usr/local目录下建立软连接
sudo ln -s apache-rocketmq rocketmq  
```



![](17.png) 

我们进到rocketmq的目录(其实是apache-rocketmq,软连接过了)里看下 



![](21.png) 

bin 一些脚本

conf配置

lib目录

logs 我们新建的，放日志的

store 存储的 





### 3 创建数据存储位置  

这里的/usr/local/rocketmq就是刚才软连接过的目录

```shell
mkdir /usr/local/rocketmq/store  # 实际数据存储位置
mkdir /usr/local/rocketmq/store/commitlog  #生产者的数据存储在这里  
mkdir /usr/local/rocketmq/store/consumequeue # 逻辑存储，类似于数据库的索引文件，放的是offset等
mkdir /usr/local/rocketmq/store/index   # 索引 rocketmq是顺序写盘，随机读，随即读性能不高，所以加了index作了索引 

```

![](18.png) 



### 4 修改rocketmq配置文件 

我们修改2m-2s-async文件夹下的broker-a.properties文件

```shell
vim /usr/local/rocketmq/conf/2m-2s-async/broker-a.properties

```



```properties
#所属集群名字
brokerClusterName=rico-rocketmq-cluster
# broker名字,注意此处不同的配置文件填写的不一样
brokerName=broker-a
# id=0表示主节点，>0表示slave
brokerId=0
# nameserver地址,如果有多节点，就逗号分隔  这个主机名称是我们在etc/hosts中配置的 没配就写ip
namesrvAddr=rocketmq-nameserver1:9876
# 存储路径
storePathRootDir=/usr/local/rocketmq/store
#commitLog存储路径
storePathCommitLog=/usr/local/rocketmq/store/commitlog
#消费队列存储路径
storePathConsumeQueue=/usr/local/rocketmq/store/consumequeue
#消费索引存储路径
storePathIndex=/usr/local/rocketmq/store/index
#限制消息大小
maxMessageSize=65536
deleteWhen=04
fileReservedTime=48
brokerRole=ASYNC_MASTER
flushDiskType=ASYNC_FLUSH
```



### 5 修改日志配置文件

```shell
cd /usr/local/rocketmq/conf && sed -i  's#${user.home}#/usr/local/rocketmq#g' *.xml
```

意思是把rocketmq/conf文件夹下所有的xml文件中的`${usr.home}`替换为`/usr/local/rocketmq` 



替换前

![](19.png) 

替换后

![](20.png)   



![](25.png) 



### 6 修改启动脚本参数

#### 修改runbroker.sh

`vim /usr/local/rocketmq/bin/runbroker.sh`   broker就是服务器，接收消息存储的地方就是broker  



官方默认配了8g内存，rocketmq是比较吃内存的 ，如果机器内存太小，可能会出问题 

```shell
JAVA_OPT="${JAVA_OPT} -server -Xms8g -Xmx8g -Xmn4g"
```

xms 默认  xmx最大 xmn最小 

最小是1g，小于1g，nameserver和broker起不来 

由于我这台笔记本是2010年买的，8年前的电脑了，当时买来是1g内存，后来加了2g内存变成3g

所以我设了1g的最大内存 

![](22.png) 





#### 修改runserver.sh

`vim /usr/local/rocketmq/bin/runserver.sh`  runserver其实就是nameserver 

server要求低一点

我也设了1g内存



### 启动

#### 先启动nameserver

```shell
nohup ./mqnamesrv &
```

![](23.png) 

如果出现namesrvStartup说明启动成功 

#### 再启动broker

```shell
nohup ./mqbroker -c /usr/local/rocketmq/conf/2m-2s-async/broker-a.properties  >/dev/null 2>&1 &
```

-c表示启动指定的配置文件  

关于 `>/dev/null 2>&1` 可以看[这篇文章](<https://www.cnblogs.com/520playboy/p/6275022.html>)    简单理解就是  表示标准输出重定向到空设备文件,也就是不输出任何信息到终端,不显示任何信息  



![](24.png) 

启动后发现有日志输出了

使用命令查看broker的日志

```
tail -f -n 300 broker.log
```



![](26.png) 



![](27.png) 



再查看下namesrv.log如果没报错的话那么说明启动成功 



### 关闭

#### 先关闭broker 

![](34.png) 



#### 再关闭nameserver

![](35.png) 



## RocketMQ控制台使用

https://github.com/apache/rocketmq-externals 

这是扩展

有控制台扩展，php，go等 

下载解压缩后进入到 `rocketmq-externals-master/rocketmq-console`目录 

修改 src/main/resources/application.properties文件 

![](28.png) 

默认通信端口为9876 

然后启动这个SpringBoot项目`mvn spring-boot:run` 或先打包`mvn package -Dmaven.test.skip=true` 



> 在部署RocketMQ插件时，遇到org.apache.rocketmq:rocketmq-tools:jar:4.4.0-SNAPSHOT包无法下载的问题：
> rocketmq-externals源码中rocketmq-console-ng工程下的pom.xml文件中<rocketmq.version>4.4.0-SNAPSHOT</rocketmq.version>声明的版本应改为4.4.0。





![](29.png) 



http://192.168.0.111:8088/#/



![](30.png) 

可以查看topic

![](31.png) 

可以新建topic

![](32.png) 



![](33.png) 



## 云主机部署rocketmq的配置： 

在云服务器上 启动好nameServer和Broker之后, 启动生产者会报这样的错误

```
RemotingTooMuchRequestException: sendDefaultImpl call timeout
```

原因： BrokerIP展示的是云服务器的本地IP，不是公网IP  据说这个问题的原因是因为虚拟机的多网卡造成的. 

![](36.png) 

**解决方法：** 

在broker 中 加入 两行配置

namesrvAddr = 127.0.0.1:9876

brokerIP1=你的公网IP

![](37.png) 

 启动broker的指令要修改下 

```shell
nohup ./mqbroker -n 公网IP:9876 autoCreateTopicEnable=true -c /usr/local/rocketmq/conf/2m-2s-async/broker-a.properties  >/dev/null 2>&1 &
```

![](38.png) 

控制台显示端口是10911,但是 **程序仍然连接9876** 

# RocketMQ入门

## 生产者使用

rocketmq在发消息的时候，**如果mq中事先不存在这个消息中设置的topic，会自动帮你创建**,并创建默认的队列 

```java

public class Producer {
    public static void main(String[] args) throws MQClientException, RemotingException, InterruptedException, MQBrokerException {
        //1 创建生产者对象DefaultMQProducer(一个应用中组名称不能重复)
        //2 设置NamesrvAddr
        //3 启动生产者服务
        //4 创建消息并发送
        DefaultMQProducer producer=new DefaultMQProducer("test_pull_producer_name");//传入生产者组组名
        producer.setNamesrvAddr(Const.NAMESERV_ADDR_SINGLE);
        producer.start();
        //RocketMQ的消息封装成了Message的对象
        for(int i=0;i<10;i++){
            /** 创建消息
             * new Message的参数
             * 1 topic  主题 消费端订阅某一个主题
             * 2 tags  标签   主要做一个过滤
             * 3 keys  唯一id  用户自定义的
             * 4 body  消息体 是字节格式的
             */
            Message message=new Message("test_pull_topic","TagA","key"+i,("hello,rocketmq"+i).getBytes());//1 topic  2 tags 3 keys 4 body
            message.setDelayTimeLevel(1);
            /** 发送消息
             * producer.send的参数
             * msg  消息
             * selector 某个topic的指定队列
             * arg  参数
             * sendCallback  回调
             * timeout  超时时间
             */
            SendResult sendResult=producer.send(message);//有返回值就是sendresult
            /* 指定队列
            SendResult sendResult=producer.send(message, new MessageQueueSelector() {
                @Override
                public MessageQueue select(List<MessageQueue> msgQueueList, Message message, Object arg) {
                    Integer queueNumber = (int) arg;
                    return msgQueueList.get(queueNumber);
                }
            },1);//这个1会回调给  public MessageQueue select(List<MessageQueue> list, Message message, Object arg) 的 arg,也就是指定发送到第1个队列
            */
            //System.out.printf(sendResult.getSendStatus());//可以获取发送状态
            System.out.println("消息发出"+sendResult.toString());//可以获取发送状态
            Thread.sleep(1000);
        }
        producer.shutdown();
    }
}
```

注意pom引入的client版本和rocketmq程序版本要一致

运行后控制台输出

```
消息发出SendResult [sendStatus=SEND_OK, msgId=0200021A487C18B4AAC204C1C3CD0000, offsetMsgId=51446DF900002A9F0000000000000000, messageQueue=MessageQueue [topic=test_pull_topic, brokerName=broker-a, queueId=2], queueOffset=0]
....
```

msgID自动生成的 

messageQueue 表示放到哪个队列里 

rocketmq在一个topic下默认会有4个队列，所以topic对queue是一对多的关系

![](39.png) 



![](40.png) 





## 消费者使用 & 重试机制

编写消费者代码

```java
public static void main(String[] args) throws MQClientException {
    /**
         * 1创建消费者对象DefaultMQPushConsumer
         * 2设置NamesrvAddr及其消费位置ConsumeFromWhere(消息消费的点在哪里?从哪里开始消费,有first,last)
         * 3进行订阅主题subscribe
         * 4注册监听并消费registerMessageListener  (所以并不是mq主动推送给消费者的，上面的mqpushConsumer感觉像是主动推，其实并不是主动发消息的，是消费者注册监听，内部是长轮训的机制 )
         */
        DefaultMQPushConsumer consumer=new DefaultMQPushConsumer("test_consumer");//传入消费者组名  有push和pull，最常用的是push
        consumer.setNamesrvAddr(Const.NAMESERV_ADDR);
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET);//last从最后端开始消费,first从头开始消费
        consumer.subscribe("test_topic","TagA");//tag其实起到一个过滤的作用，当然你也可以写*,这样就不过滤了topic下所有的都消费
        consumer.registerMessageListener(new MessageListenerConcurrently() {//并发的接口
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgList, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
                //对消息处理的逻辑
                MessageExt messageExt=msgList.get(0);//由于我们发的是单条消息，所以获取0就好了
                try{
                    String topic=messageExt.getTopic();
                    String tag=messageExt.getTags();
                    String keys=messageExt.getKeys();
                    String msgBody=new String(messageExt.getBody(), RemotingHelper.DEFAULT_CHARSET);
                    System.out.println("topic:"+topic+"tag:"+tag+"keys"+keys+"msgBody:"+msgBody);
                }catch (Exception e){

                    return ConsumeConcurrentlyStatus.RECONSUME_LATER;//如果失败，稍后重试
                }
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        consumer.start();
        System.out.println("consumer start");
    }
```

![](41.png) 

上面代码中有

```java
catch (Exception e){
	return ConsumeConcurrentlyStatus.RECONSUME_LATER;//如果失败，稍后重试
}
```

意思是如果发生异常，会再次重新push过来，过1s ,2s,5s,10s等间隔

那么如何知道目前是第几次重试呢？ 答：其实 **MessageExt** 中包含了重试次数 



### 重试机制

其实mq在启动的时候进行了很多的配置项，可以去broker.log那里去看

![](42.png) 

总共有15次自动重试(如果一直失败)

### 模拟消息异常

```java
public class Consumer {
    public static void main(String[] args) throws MQClientException {
        /**
         * 1创建消费者对象DefaultMQPushConsumer
         * 2设置NamesrvAddr及其消费位置ConsumeFromWhere(消息消费的点在哪里?从哪里开始消费,有first,last)
         * 3进行订阅主题subscribe
         * 4注册监听并消费registerMessageListener  (所以并不是mq主动推送给消费者的，上面的mqpushConsumer感觉像是主动推，其实并不是主动发消息的，是消费者注册监听，内部是长轮训的机制 )
         */
        DefaultMQPushConsumer consumer=new DefaultMQPushConsumer("test_consumer");//传入消费者组名  有push和pull，最常用的是push
        consumer.setNamesrvAddr(Const.NAMESERV_ADDR_SINGLE);
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET);//last从最后端开始消费,first从头开始消费
        consumer.subscribe("test_pull_topic","TagA");//tag其实起到一个过滤的作用，当然你也可以写*,这样就不过滤了topic下所有的都消费
        consumer.setConsumeMessageBatchMaxSize(1);//一次最多可以拉取多少消息
        consumer.setConsumeThreadMax(20);//最多会有20个线程去拉取消息
        consumer.setMaxReconsumeTimes(3);//最大重试次数3次
        consumer.setMessageModel(MessageModel.CLUSTERING);//默认是CLUSTERING集群模式
        consumer.registerMessageListener(//注册消息监听
          new MessageListenerConcurrently() {//并发的监听,还有一个是MessageListenerOrderly,它是用作顺序消费的
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgList, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
                //对消息处理的逻辑
                MessageExt messageExt=msgList.get(0);//由于我们发的是单条消息，所以获取0就好了
                try{
                    String topic=messageExt.getTopic();
                    String tag=messageExt.getTags();
                    String keys=messageExt.getKeys();
                    /*
                    if (messageExt.getQueueId() == 1) {
                        ....
                    }*/
                    if(keys.equals("key2")) {//我们选取key2来当作失败
                        System.out.println("消息消费失败");
                        int a=1/0;
                    }
                    String msgBody=new String(messageExt.getBody(), RemotingHelper.DEFAULT_CHARSET);
                    System.out.println("topic:"+topic+"tag:"+tag+"keys"+keys+"msgBody:"+msgBody);
                }catch (Exception e){
                    e.printStackTrace();
                    int reconsumeTimes = messageExt.getReconsumeTimes();//当前这条消息被重发了多少次
                    System.err.println("当前消息重发了: "+reconsumeTimes+"次");
                    if(reconsumeTimes == 3){
                        //记录日志,作补偿处理.....
                        return  ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
                    }
                    return ConsumeConcurrentlyStatus.RECONSUME_LATER;//如果失败，稍后重试
                }
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        consumer.start();
        System.out.println("消费者启动了");
    }
}
```

![](43.png) 



## 四种集群环境讲解

待完善

## 主从集群环境构建

待完善

## 数据高可用机制故障演练

待完善





# RocketMQ  生产者核心

配置参数解析

主从同步机制解析

同步/异步消息发送解析

底层Netty通信解析

延迟消息投递解析



# RocketMQ 消费者核心


