---
title: Netty入门与实战
date: 2020-07-15 17:59:35
tags: Netty
---




 https://netty.io/wiki/user-guide-for-4.x.html 

# IM系统简介

我们使用 Netty 统一的 IO 读写 API 以及强大的 pipeline 来编写业务处理逻辑，在后续的章节中，我会通过 IM 这个例子，带你逐步了解 Netty 的以下核心知识点。

- 服务端如何启动
- 客户端如何启动
- 数据载体 ByteBuf
- 长连自定义协议如何设计
- 粘包拆包原理与实践
- 如何实现自定义编解码
- pipeline 与 channelHandler
- 定时发心跳怎么做
- 如何进行连接空闲检测

### 客户端使用 Netty 的程序逻辑结构



![](1.png) 



上面这幅图展示了客户端的程序逻辑结构

1. 首先，客户端会解析控制台指令，比如发送消息或者建立群聊等指令
2. 然后，客户端会基于控制台的输入创建一个指令对象，用户告诉服务端具体要干什么事情
3. TCP 通信需要的数据格式为二进制，因此，接下来通过自定义二进制协议将指令对象封装成二进制，这一步称为协议的编码
4. 对于收到服务端的数据，首先需要截取出一段完整的二进制数据包（拆包粘包相关的内容后续小节会讲解）
5. 将此二进制数据包解析成指令对象，比如收到消息
6. 将指令对象送到对应的逻辑处理器来处理



### 服务端使用 Netty 的程序逻辑结构

![](2.png) 



 服务端的程序逻辑结构与客户端非常类似，这里不太赘述。 

# Netty 是什么？

在开始了解 Netty 是什么之前，我们先来回顾一下，如果我们需要实现一个客户端与服务端通信的程序，使用传统的 IO 编程，应该如何来实现？

## IO编程

我们简化下场景：客户端每隔两秒发送一个带有时间戳的 "hello world" 给服务端，服务端收到之后打印。

为了方便演示，下面例子中，服务端和客户端各一个类，把这两个类拷贝到你的 IDE 中，先后运行 `IOServer.java` 和`IOClient.java`可看到效果。

下面是传统的 IO 编程中服务端实现

> IOServer.java

```java
public class IOService {
    public static void main(String[] args) throws Exception {
        /**
          Server 端首先创建了一个serverSocket来监听 8000 端口，然后创建一个线程，线程里面不断调用阻塞方法 serversocket.accept();获取新的连接，见(1)，
         当获取到新的连接之后，给每条连接创建一个新的线程，这个线程负责从该连接中读取数据，见(2)，
         然后读取数据是以字节流的方式，见(3)。
         */
        ServerSocket serverSocket = new ServerSocket(8000);
        // (1) 接收新连接线程
        new Thread(() -> {
            while (true) {
                try {
                    // (1) 阻塞方法获取新的连接
                    Socket socket = serverSocket.accept();
                    // (2) 每一个新的连接都创建一个线程，负责读取数据
                    new Thread(() -> {
                        try {
                            int len;
                            byte[] data = new byte[1024];
                            InputStream inputStream = socket.getInputStream();
                            // (3) 按字节流方式读取数据
                            while ((len = inputStream.read(data)) != -1) {
                                System.out.println("收到客户端发来的数据:"+new String(data, 0, len));
                            }
                        } catch (IOException e) { }
                    }).start();
                } catch (IOException e) { }
            }
        }).start();
    }
}
```

Server 端首先创建了一个`serverSocket`来监听 8000 端口，然后创建一个线程，线程里面不断调用阻塞方法 `serversocket.accept();`获取新的连接，见(1)，当获取到新的连接之后，给每条连接创建一个新的线程，这个线程负责从该连接中读取数据，见(2)，然后读取数据是以字节流的方式，见(3)。

下面是传统的IO编程中客户端实现

> IOClient.java

```java
public class IOClient {
    public static void main(String[] args) {
        new Thread(() -> {
            try {
                Socket socket = new Socket("127.0.0.1", 8000);
                while (true) {
                    try {
                        System.out.println("向服务端发送数据:"+new String(new Date()+"hello world"));
                        socket.getOutputStream().write((new Date() + ": hello world").getBytes());
                        Thread.sleep(2000);
                    } catch (Exception e) { }
                }
            } catch (IOException e) { }
        }).start();
    }
}
```

客户端的代码相对简单，连接上服务端 8000 端口之后，每隔 2 秒，我们向服务端写一个带有时间戳的 "hello world"。

![](3.png) 

![](4.png)



IO 编程模型在客户端较少的情况下运行良好，但是对于客户端比较多的业务来说，单机服务端可能需要支撑成千上万的连接，IO 模型可能就不太合适了，我们来分析一下原因。

上面的 demo，从服务端代码中我们可以看到，在传统的 IO 模型中，每个连接创建成功之后都需要一个线程来维护，每个线程包含一个 while 死循环，那么 1w 个连接对应 1w 个线程，继而 1w 个 while 死循环，这就带来如下几个问题：

1. 线程资源受限：线程是操作系统中非常宝贵的资源，同一时刻有大量的线程处于阻塞状态是非常严重的资源浪费，操作系统耗不起
2. 线程切换效率低下：单机 CPU 核数固定，线程爆炸之后操作系统频繁进行线程切换，应用性能急剧下降。
3. 除了以上两个问题，IO 编程中，我们看到数据读写是以字节流为单位。

为了解决这三个问题，JDK 在 1.4 之后提出了 NIO。

## NIO 编程

关于 NIO 相关的文章网上也有很多，这里不打算详细深入分析，下面简单描述一下 NIO 是如何解决以上三个问题的。

### 线程资源受限

NIO 编程模型中，新来一个连接不再创建一个新的线程，而是可以把这条连接直接绑定到某个固定的线程，然后这条连接所有的读写都由这个线程来负责，那么他是怎么做到的？我们用一幅图来对比一下 IO 与 NIO

![](5.png) 



如上图所示，IO 模型中，一个连接来了，会创建一个线程，对应一个 while 死循环，死循环的目的就是不断监测这条连接上是否有数据可以读，大多数情况下，1w 个连接里面同一时刻只有少量的连接有数据可读，因此，很多个 while 死循环都白白浪费掉了，因为读不出啥数据。

而在 NIO 模型中，**他把这么多 while 死循环变成一个死循环，这个死循环由一个线程控制**，那么他又是如何做到一个线程，一个 while 死循环就能监测1w个连接是否有数据可读的呢？ 这就是 NIO 模型中 selector 的作用，一条连接来了之后，现在不创建一个 while 死循环去监听是否有数据可读了，而是直接把这条连接注册到 selector 上，然后，通过检查这个 selector，就可以批量监测出有数据可读的连接，进而读取数据，下面我再举个非常简单的生活中的例子说明 IO 与 NIO 的区别。

在一家幼儿园里，小朋友有上厕所的需求，小朋友都太小以至于你要问他要不要上厕所，他才会告诉你。幼儿园一共有 100 个小朋友，有两种方案可以解决小朋友上厕所的问题：

1. 每个小朋友配一个老师。每个老师隔段时间询问小朋友是否要上厕所，如果要上，就领他去厕所，100 个小朋友就需要 100 个老师来询问，并且每个小朋友上厕所的时候都需要一个老师领着他去上，这就是IO模型，一个连接对应一个线程。
2. 所有的小朋友都配同一个老师。这个老师隔段时间询问所有的小朋友是否有人要上厕所，然后每一时刻把所有要上厕所的小朋友批量领到厕所，这就是 NIO 模型，所有小朋友都注册到同一个老师，对应的就是所有的连接都注册到一个线程，然后批量轮询。

这就是 NIO 模型解决线程资源受限的方案，实际开发过程中，我们会开多个线程，每个线程都管理着一批连接，相对于 IO 模型中一个线程管理一条连接，消耗的线程资源大幅减少

### 线程切换效率低下

由于 NIO 模型中线程数量大大降低，线程切换效率因此也大幅度提高

### IO读写面向流

IO 读写是面向流的，一次性只能从流中读取一个或者多个字节，并且读完之后流无法再读取，你需要自己缓存数据。 而 NIO 的读写是面向 Buffer 的，你可以随意读取里面任何一个字节数据，不需要你自己缓存数据，这一切只需要移动读写指针即可。

简单讲完了 JDK NIO 的解决方案之后，我们接下来使用 NIO 的方案替换掉 IO 的方案，我们先来看看，如果用 JDK 原生的 NIO 来实现服务端，该怎么做

```java
public class NIOServer {
    public static void main(String[] args) throws IOException {
        Selector serverSelector = Selector.open();
        Selector clientSelector = Selector.open();

        new Thread(() -> {
            try {
                // 对应IO编程中服务端启动
                ServerSocketChannel listenerChannel = ServerSocketChannel.open();
                listenerChannel.socket().bind(new InetSocketAddress(8000));
                listenerChannel.configureBlocking(false);
                listenerChannel.register(serverSelector, SelectionKey.OP_ACCEPT);

                while (true) {
                    // 监测是否有新的连接，这里的1指的是阻塞的时间为 1ms
                    if (serverSelector.select(1) > 0) {
                        Set<SelectionKey> set = serverSelector.selectedKeys();
                        Iterator<SelectionKey> keyIterator = set.iterator();

                        while (keyIterator.hasNext()) {
                            SelectionKey key = keyIterator.next();

                            if (key.isAcceptable()) {
                                try {
                                    // (1) 每来一个新连接，不需要创建一个线程，而是直接注册到clientSelector
                                    SocketChannel clientChannel = ((ServerSocketChannel) key.channel()).accept();
                                    clientChannel.configureBlocking(false);
                                    clientChannel.register(clientSelector, SelectionKey.OP_READ);
                                } finally {
                                    keyIterator.remove();
                                }
                            }

                        }
                    }
                }
            } catch (IOException ignored) {
            }

        }).start();


        new Thread(() -> {
            try {
                while (true) {
                    // (2) 批量轮询是否有哪些连接有数据可读，这里的1指的是阻塞的时间为 1ms
                    if (clientSelector.select(1) > 0) {
                        Set<SelectionKey> set = clientSelector.selectedKeys();
                        Iterator<SelectionKey> keyIterator = set.iterator();

                        while (keyIterator.hasNext()) {
                            SelectionKey key = keyIterator.next();

                            if (key.isReadable()) {
                                try {
                                    SocketChannel clientChannel = (SocketChannel) key.channel();
                                    ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
                                    // (3) 面向 Buffer
                                    clientChannel.read(byteBuffer);
                                    byteBuffer.flip();
                                    System.out.println(Charset.defaultCharset().newDecoder().decode(byteBuffer)
                                            .toString());
                                } finally {
                                    keyIterator.remove();
                                    key.interestOps(SelectionKey.OP_READ);
                                }
                            }

                        }
                    }
                }
            } catch (IOException ignored) {
            }
        }).start();
    }
}
```



相信大部分没有接触过 NIO 的同学应该会直接跳过代码来到这一行：原来使用 JDK 原生 NIO 的 API 实现一个简单的服务端通信程序是如此复杂!

复杂得我都没耐心解释这一坨代码的执行逻辑(开个玩笑)，我们还是先对照 NIO 来解释一下几个核心思路

1. NIO 模型中通常会有两个线程，每个线程绑定一个轮询器 selector ，在我们这个例子中`serverSelector`负责轮询是否有新的连接，`clientSelector`负责轮询连接是否有数据可读
2. 服务端监测到新的连接之后，不再创建一个新的线程，而是直接将新连接绑定到`clientSelector`上，这样就不用 IO 模型中 1w 个 while 循环在死等，参见(1)
3. `clientSelector`被一个 while 死循环包裹着，如果在某一时刻有多条连接有数据可读，那么通过 `clientSelector.select(1)`方法可以轮询出来，进而批量处理，参见(2)
4. 数据的读写面向 Buffer，参见(3)

其他的细节部分，我不愿意多讲，因为实在是太复杂，你也不用对代码的细节深究到底。总之，强烈不建议直接基于JDK原生NIO来进行网络开发，下面是我总结的原因

1. JDK 的 NIO 编程需要了解很多的概念，编程复杂，对 NIO 入门非常不友好，编程模型不友好，ByteBuffer 的 Api 简直反人类
2. 对 NIO 编程来说，一个比较合适的线程模型能充分发挥它的优势，而 JDK 没有给你实现，你需要自己实现，就连简单的自定义协议拆包都要你自己实现
3. JDK 的 NIO 底层由 epoll 实现，该实现饱受诟病的空轮询 bug 会导致 cpu 飙升 100%
4. 项目庞大之后，自行实现的 NIO 很容易出现各类 bug，维护成本较高，上面这一坨代码我都不能保证没有 bug

正因为如此，我客户端代码都懒得写给你看了==!，你可以直接使用`IOClient.java`与`NIOServer.java`通信

JDK 的 NIO 犹如带刺的玫瑰，虽然美好，让人向往，但是使用不当会让你抓耳挠腮，痛不欲生，正因为如此，Netty 横空出世！

## Netty编程

那么 Netty 到底是何方神圣？ 用一句简单的话来说就是：Netty 封装了 JDK 的 NIO，让你用得更爽，你不用再写一大堆复杂的代码了。 用官方正式的话来说就是：Netty 是一个异步事件驱动的网络应用框架，用于快速开发可维护的高性能服务器和客户端。

下面是我总结的使用 Netty 不使用 JDK 原生 NIO 的原因

1. 使用 JDK 自带的NIO需要了解太多的概念，编程复杂，一不小心 bug 横飞
2. Netty 底层 IO 模型随意切换，而这一切只需要做微小的改动，改改参数，Netty可以直接从 NIO 模型变身为 IO 模型
3. Netty 自带的拆包解包，异常检测等机制让你从NIO的繁重细节中脱离出来，让你只需要关心业务逻辑
4. Netty 解决了 JDK 的很多包括空轮询在内的 Bug
5. Netty 底层对线程，selector 做了很多细小的优化，精心设计的 reactor 线程模型做到非常高效的并发处理
6. 自带各种协议栈让你处理任何一种通用协议都几乎不用亲自动手
7. Netty 社区活跃，遇到问题随时邮件列表或者 issue
8. Netty 已经历各大 RPC 框架，消息中间件，分布式通信中间件线上的广泛验证，健壮性无比强大

看不懂没有关系，这些我们在后续的课程中我们都可以学到，接下来我们用 Netty 的版本来重新实现一下本文开篇的功能吧

首先，引入 Maven 依赖，本文后续 Netty 都是基于 4.1.6.Final 版本



 然后，下面是服务端实现部分 

```java
		/**
         * 我们创建了两个NioEventLoopGroup，这两个对象可以看做是传统IO编程模型的两大线程组
         * bossGroup表示监听端口，accept 新连接的线程组
         * workerGroup表示处理每一条连接的数据读写的线程组
         */
        NioEventLoopGroup boss = new NioEventLoopGroup();
        NioEventLoopGroup worker = new NioEventLoopGroup();

        //创建一个引导类 ServerBootstrap，这个类将引导我们进行服务端的启动工作
        ServerBootstrap serverBootstrap = new ServerBootstrap();
        serverBootstrap
                //通过.group(bossGroup, workerGroup)给引导类配置两大线程组，这个引导类的线程模型也就定型了。
                .group(boss, worker)
                //然后，我们指定我们服务端的 IO 模型为NIO，我们通过.channel(NioServerSocketChannel.class)来指定 IO 模型，当然，这里也有其他的选择，如果你想指定 IO 模型为 BIO，那么这里配置上OioServerSocketChannel.class类型即可，当然通常我们也不会这么做，因为Netty的优势就在于NIO。
                .channel(NioServerSocketChannel.class)
                //我们调用childHandler()方法，给这个引导类创建一个ChannelInitializer，这里主要就是定义后续每条连接的数据读写
                //ChannelInitializer这个类中，我们注意到有一个泛型参数NioSocketChannel，这个类呢，就是 Netty 对 NIO 类型的连接的抽象，而我们前面NioServerSocketChannel也是对 NIO 类型的连接的抽象，NioServerSocketChannel和NioSocketChannel的概念可以和 BIO 编程模型中的ServerSocket以及Socket两个概念对应上
                .childHandler(new ChannelInitializer<NioSocketChannel>() {
                    protected void initChannel(NioSocketChannel ch) {
                        ch.pipeline().addLast(new StringDecoder());
                        ch.pipeline().addLast(new SimpleChannelInboundHandler<String>() {
                            @Override
                            protected void channelRead0(ChannelHandlerContext ctx, String msg) {
                                System.out.println(msg);
                            }
                        });
                    }
                });
        serverBootstrap.bind(8000);
        /**
         * 我们的最小化参数配置到这里就完成了
         * 总结一下，要启动一个Netty服务端，必须要指定三类属性，分别是线程模型、IO 模型、连接读写处理逻辑.
         * 有了这三者，之后在调用bind(8000)，我们就可以在本地绑定一个 8000 端口启动起来
         */
```

这么一小段代码就实现了我们前面 NIO 编程中的所有的功能，包括服务端启动，接受新连接，打印客户端传来的数据，怎么样，是不是比 JDK 原生的 NIO 编程优雅许多？

初学 Netty 的时候，由于大部分人对 NIO 编程缺乏经验，因此，将 Netty 里面的概念与 IO 模型结合起来可能更好理解

1. `boss` 对应 `IOServer.java` 中的接受新连接线程，主要负责创建新连接
2. `worker` 对应 `IOServer.java` 中的负责读取数据的线程，主要用于读取数据以及业务逻辑处理

然后剩下的逻辑我在后面的系列文章中会详细分析，你可以先把这段代码拷贝到你的 IDE 里面，然后运行 main 函数

然后下面是客户端 NIO 的实现部分



```java
//NettyClient.java
public class NettyClient {
    public static void main(String[] args) throws InterruptedException {
        Bootstrap bootstrap = new Bootstrap();
        NioEventLoopGroup group = new NioEventLoopGroup();

        bootstrap.group(group)
                .channel(NioSocketChannel.class)
                .handler(new ChannelInitializer<Channel>() {
                    @Override
                    protected void initChannel(Channel ch) {
                        ch.pipeline().addLast(new StringEncoder());
                    }
                });

        Channel channel = bootstrap.connect("127.0.0.1", 8000).channel();

        while (true) {
            channel.writeAndFlush(new Date() + ": hello world!");
            Thread.sleep(2000);
        }
    }
}
```

在客户端程序中，`group`对应了我们`IOClient.java`中 main 函数起的线程，剩下的逻辑我在后面的文章中会详细分析，现在你要做的事情就是把这段代码拷贝到你的 IDE 里面，然后运行 main 函数，最后回到 `NettyServer.java` 的控制台，你会看到效果。

使用 Netty 之后是不是觉得整个世界都美好了，一方面 Netty 对 NIO 封装得如此完美，写出来的代码非常优雅，另外一方面，使用 Netty 之后，网络通信这块的性能问题几乎不用操心，尽情地让 Netty 榨干你的 CPU 吧。



# 服务端启动流程

这一小节，我们来学习一下如何使用 Netty 来启动一个服务端应用程序，以下是服务端启动的一个非常精简的 Demo:

> NettyServer.java

```java
public class NettyServer {
    public static void main(String[] args) {
        NioEventLoopGroup bossGroup = new NioEventLoopGroup();
        NioEventLoopGroup workerGroup = new NioEventLoopGroup();

        ServerBootstrap serverBootstrap = new ServerBootstrap();
        serverBootstrap
                .group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<NioSocketChannel>() {
                    protected void initChannel(NioSocketChannel ch) {
                    }
                });

        serverBootstrap.bind(8000);
    }
}
```



- 首先看到，我们创建了两个`NioEventLoopGroup`，这两个对象可以看做是传统IO编程模型的两大线程组，`bossGroup`表示监听端口，accept 新连接的线程组，`workerGroup`表示处理每一条连接的数据读写的线程组，不理解的同学可以看一下上一小节[《Netty是什么》](https://juejin.im/book/5b4bc28bf265da0f60130116/section/5b4bc28b5188251b1f224ee5)。用生活中的例子来讲就是，一个工厂要运作，必然要有一个老板负责从外面接活，然后有很多员工，负责具体干活，老板就是`bossGroup`，员工们就是`workerGroup`，`bossGroup`接收完连接，扔给`workerGroup`去处理。
- 接下来 我们创建了一个引导类 `ServerBootstrap`，这个类将引导我们进行服务端的启动工作，直接new出来开搞。
- 我们通过`.group(bossGroup, workerGroup)`给引导类配置两大线程组，这个引导类的线程模型也就定型了。
- 然后，我们指定我们服务端的 IO 模型为`NIO`，我们通过`.channel(NioServerSocketChannel.class)`来指定 IO 模型，当然，这里也有其他的选择，如果你想指定 IO 模型为 BIO，那么这里配置上`OioServerSocketChannel.class`类型即可，当然通常我们也不会这么做，因为Netty的优势就在于NIO。
- 接着，我们调用`childHandler()`方法，给这个引导类创建一个`ChannelInitializer`，这里主要就是定义后续每条连接的数据读写，业务处理逻辑，不理解没关系，在后面我们会详细分析。`ChannelInitializer`这个类中，我们注意到有一个泛型参数`NioSocketChannel`，这个类呢，就是 Netty 对 NIO 类型的连接的抽象，而我们前面`NioServerSocketChannel`也是对 NIO 类型的连接的抽象，`NioServerSocketChannel`和`NioSocketChannel`的概念可以和 BIO 编程模型中的`ServerSocket`以及`Socket`两个概念对应上

我们的最小化参数配置到这里就完成了，我们总结一下就是，要启动一个Netty服务端，必须要指定三类属性，分别是线程模型、IO 模型、连接读写处理逻辑，有了这三者，之后在调用`bind(8000)`，我们就可以在本地绑定一个 8000 端口启动起来，以上这段代码读者可以直接拷贝到你的 IDE 中运行。



## 自动绑定递增端口

在上面代码中我们绑定了 8000 端口，接下来我们实现一个稍微复杂一点的逻辑，我们指定一个起始端口号，比如 1000，然后呢，我们从1000号端口往上找一个端口，直到这个端口能够绑定成功，比如 1000 端口不可用，我们就尝试绑定 1001，然后 1002，依次类推。

`serverBootstrap.bind(8000);`这个方法呢，它是一个异步的方法，调用之后是立即返回的，他的返回值是一个`ChannelFuture`，我们可以给这个`ChannelFuture`添加一个监听器`GenericFutureListener`，然后我们在`GenericFutureListener`的`operationComplete`方法里面，我们可以监听端口是否绑定成功，接下来是监测端口是否绑定成功的代码片段



```java
serverBootstrap.bind(8000).addListener(new GenericFutureListener<Future<? super Void>>() {
    public void operationComplete(Future<? super Void> future) {
        if (future.isSuccess()) {
            System.out.println("端口绑定成功!");
        } else {
            System.err.println("端口绑定失败!");
        }
    }
});
```

 我们接下来从 1000 端口号，开始往上找端口号，直到端口绑定成功，我们要做的就是在 `if (future.isSuccess())`的else逻辑里面重新绑定一个递增的端口号，接下来，我们把这段绑定逻辑抽取出一个`bind`方法 

```java
private static void bind(final ServerBootstrap serverBootstrap, final int port) {
    serverBootstrap.bind(port).addListener(new GenericFutureListener<Future<? super Void>>() {
        public void operationComplete(Future<? super Void> future) {
            if (future.isSuccess()) {
                System.out.println("端口[" + port + "]绑定成功!");
            } else {
                System.err.println("端口[" + port + "]绑定失败!");
                bind(serverBootstrap, port + 1);
            }
        }
    });
}
```

然后呢，以上代码中最关键的就是在端口绑定失败之后，重新调用自身方法，并且把端口号加一，然后，在我们的主流程里面，我们就可以直接调用

```java
bind(serverBootstrap, 1000)
```

读者可以自定修改代码，运行之后可以看到效果，最终会发现，端口成功绑定了在1024，从 1000 开始到 1023，端口均绑定失败了，这是因为在我的 MAC 系统下，1023 以下的端口号都是被系统保留了，需要 ROOT 权限才能绑定。

以上就是自动绑定递增端口的逻辑，接下来，我们来一起学习一下，服务端启动，我们的引导类`ServerBootstrap`除了指定线程模型，IO 模型，连接读写处理逻辑之外，他还可以干哪些事情？



## 服务端启动其他方法

### handler() 方法

```java
serverBootstrap.handler(new ChannelInitializer<NioServerSocketChannel>() {
    protected void initChannel(NioServerSocketChannel ch) {
        System.out.println("服务端启动中");
    }
})
```

`handler()`方法呢，可以和我们前面分析的`childHandler()`方法对应起来，**childHandler()用于指定处理新连接数据的读写处理逻辑**，**handler()用于指定在服务端启动过程中的一些逻辑**，通常情况下呢，我们用不着这个方法。

### attr() 方法

```java
serverBootstrap.attr(AttributeKey.newInstance("serverName"), "nettyServer")
```

`attr()`方法可以给服务端的 channel，也就是`NioServerSocketChannel`指定一些自定义属性，然后我们可以通过`channel.attr()`取出这个属性，比如，上面的代码我们指定我们服务端channel的一个`serverName`属性，属性值为`nettyServer`，其实说白了就是给`NioServerSocketChannel`维护一个map而已，通常情况下，我们也用不上这个方法。

那么，当然，除了可以给服务端 channel `NioServerSocketChannel`指定一些自定义属性之外，我们还可以给每一条连接指定自定义属性

### childAttr() 方法

```
serverBootstrap.childAttr(AttributeKey.newInstance("clientKey"), "clientValue")
```

上面的`childAttr`可以给每一条连接指定自定义属性，然后后续我们可以通过`channel.attr()`取出该属性。

### childOption() 方法

```
serverBootstrap
        .childOption(ChannelOption.SO_KEEPALIVE, true)
        .childOption(ChannelOption.TCP_NODELAY, true)
```

`childOption()`可以给每条连接设置一些TCP底层相关的属性，比如上面，我们设置了两种TCP属性，其中

- `ChannelOption.SO_KEEPALIVE`表示是否开启TCP底层心跳机制，true为开启
- `ChannelOption.TCP_NODELAY`表示是否开启Nagle算法，true表示关闭，false表示开启，通俗地说，如果要求高实时性，有数据发送时就马上发送，就关闭，如果需要减少发送次数减少网络交互，就开启。

其他的参数这里就不一一讲解，有兴趣的同学可以去这个类里面自行研究。

### option() 方法

除了给每个连接设置这一系列属性之外，我们还可以给服务端channel设置一些属性，最常见的就是so_backlog，如下设置

```
serverBootstrap.option(ChannelOption.SO_BACKLOG, 1024)
```

表示系统用于临时存放已完成三次握手的请求的队列的最大长度，如果连接建立频繁，服务器处理创建新连接较慢，可以适当调大这个参数

## 总结

- 本文中，我们首先学习了 Netty 服务端启动的流程，一句话来说就是：创建一个引导类，然后给他指定线程模型，IO模型，连接读写处理逻辑，绑定端口之后，服务端就启动起来了。
- 然后，我们学习到 bind 方法是异步的，我们可以通过这个异步机制来实现端口递增绑定。
- 最后呢，我们讨论了 Netty 服务端启动额外的参数，主要包括给服务端 Channel 或者客户端 Channel 设置属性值，设置底层 TCP 参数。

如果，你觉得这个过程比较简单，想深入学习，了解服务端启动的底层原理，可参考[这里](https://coding.imooc.com/class/chapter/230.html#Anchor)。



# 客户端启动流程

上一小节，我们已经学习了 Netty 服务端启动的流程，这一小节，我们来学习一下 Netty 客户端的启动流程。

## 客户端启动 Demo

对于客户端的启动来说，和服务端的启动类似，依然需要线程模型、IO 模型，以及 IO 业务处理逻辑三大参数，下面，我们来看一下客户端启动的标准流程

> NettyClient.java

```java
public class NettyClient {
    public static void main(String[] args) throws InterruptedException {
        //客户端启动的引导类是 Bootstrap，负责启动客户端以及连接服务端，而上一小节我们在描述服务端的启动的时候，这个辅导类是 ServerBootstrap
        Bootstrap bootstrap = new Bootstrap();//服务端是ServerBootstrap,客户端时Bootstrap
        NioEventLoopGroup group = new NioEventLoopGroup();

        bootstrap
                // 1.指定线程模型
                .group(group)
                // 2.指定 IO 类型为 NIO
                .channel(NioSocketChannel.class)
                //给引导类指定一个 handler，这里主要就是定义连接的业务处理逻辑
                .handler(new ChannelInitializer<Channel>() {
                    @Override
                    protected void initChannel(Channel ch) {
                        ch.pipeline().addLast(new StringEncoder());
                    }
                });

        // // 4.建立连接
        //配置完线程模型、IO 模型、业务处理逻辑之后，调用 connect 方法进行连接，可以看到 connect 方法有两个参数，第一个参数可以填写 IP 或者域名，第二个参数填写的是端口号，由于 connect 方法返回的是一个 Future，也就是说这个方是异步的，我们通过 addListener 方法可以监听到连接是否成功，进而打印出连接信息
        Channel channel = bootstrap.connect("127.0.0.1", 8000).addListener(future -> {
            //connect 方法返回的是一个 Future，也就是说这个方是异步的，我们通过 addListener 方法可以监听到连接是否成功，进而打印出连接信息
            if (future.isSuccess()) {
                System.out.println("连接成功!");
            } else {
                System.err.println("连接失败!");
            }
        }).channel();
        while (true) {
            channel.writeAndFlush(new Date() + ": hello world!");
            Thread.sleep(2000);
        }
    }
}
```

从上面代码可以看到，客户端启动的引导类是 `Bootstrap`，负责启动客户端以及连接服务端，而上一小节我们在描述服务端的启动的时候，这个辅导类是 `ServerBootstrap`，引导类创建完成之后，下面我们描述一下客户端启动的流程

1. 首先，与服务端的启动一样，我们需要给它指定线程模型，驱动着连接的数据读写，这个线程的概念可以和第一小节[Netty是什么](https://juejin.im/book/5b4bc28bf265da0f60130116/section/5b4bc28b5188251b1f224ee5)中的 `IOClient.java` 创建的线程联系起来
2. 然后，我们指定 IO 模型为 `NioSocketChannel`，表示 IO 模型为 NIO，当然，你可以可以设置 IO 模型为 `OioSocketChannel`，但是通常不会这么做，因为 Netty 的优势在于 NIO
3. 接着，给引导类指定一个 `handler`，这里主要就是定义连接的业务处理逻辑，不理解没关系，在后面我们会详细分析
4. 配置完线程模型、IO 模型、业务处理逻辑之后，调用 `connect` 方法进行连接，可以看到 `connect` 方法有两个参数，第一个参数可以填写 IP 或者域名，第二个参数填写的是端口号，由于 `connect` 方法返回的是一个 `Future`，也就是说这个方是异步的，我们通过 `addListener` 方法可以监听到连接是否成功，进而打印出连接信息

到了这里，一个客户端的启动的 Demo 就完成了，其实只要和 客户端 Socket 编程模型对应起来，这里的三个概念就会显得非常简单，遗忘掉的同学可以回顾一下 [Netty是什么](https://juejin.im/book/5b4bc28bf265da0f60130116/section/5b4bc28b5188251b1f224ee5)中的 `IOClient.java` 再回来看这里的启动流程哦

## 失败重连

在网络情况差的情况下，客户端第一次连接可能会连接失败，这个时候我们可能会尝试重新连接，重新连接的逻辑写在连接失败的逻辑块里

```java
bootstrap.connect("juejin.im", 80).addListener(future -> {
    if (future.isSuccess()) {
        System.out.println("连接成功!");
    } else {
        System.err.println("连接失败!");
        // 重新连接
    }
});
```

重新连接的时候，依然是调用一样的逻辑，因此，我们把建立连接的逻辑先抽取出来，然后在重连失败的时候，递归调用自身

```java
private static void connect(Bootstrap bootstrap, String host, int port) {
    bootstrap.connect(host, port).addListener(future -> {
        if (future.isSuccess()) {
            System.out.println("连接成功!");
        } else {
            System.err.println("连接失败，开始重连");
            connect(bootstrap, host, port);
        }
    });
}
```

上面这一段便是带有自动重连功能的逻辑，可以看到在连接建立失败的时候，会调用自身进行重连。

但是，通常情况下，连接建立失败不会立即重新连接，而是会通过一个**指数退避**的方式，比如每隔 1 秒、2 秒、4 秒、8 秒，以 2 的幂次来建立连接，然后到达一定次数之后就放弃连接，接下来我们就来实现一下这段逻辑，我们默认重试 5 次

```java
connect(bootstrap, "juejin.im", 80, MAX_RETRY);

private static void connect(Bootstrap bootstrap, String host, int port, int retry) {
    bootstrap.connect(host, port).addListener(future -> {
        if (future.isSuccess()) {
            System.out.println("连接成功!");
        } else if (retry == 0) {
            System.err.println("重试次数已用完，放弃连接！");
        } else {
            // 第几次重连
            int order = (MAX_RETRY - retry) + 1;
            // 本次重连的间隔
            int delay = 1 << order;
            System.err.println(new Date() + ": 连接失败，第" + order + "次重连……");
            bootstrap.config().group().schedule(() -> connect(bootstrap, host, port, retry - 1), delay, TimeUnit
                    .SECONDS);
        }
    });
}
```



从上面的代码可以看到，通过判断连接是否成功以及剩余重试次数，分别执行不同的逻辑

1. 如果连接成功则打印连接成功的消息
2. 如果连接失败但是重试次数已经用完，放弃连接
3. 如果连接失败但是重试次数仍然没有用完，则计算下一次重连间隔 `delay`，然后定期重连

在上面的代码中，我们看到，我们定时任务是调用 `bootstrap.config().group().schedule()`, 其中 `bootstrap.config()` 这个方法返回的是 `BootstrapConfig`，他是对 `Bootstrap` 配置参数的抽象，然后 `bootstrap.config().group()` 返回的就是我们在一开始的时候配置的线程模型 `workerGroup`，调 `workerGroup` 的 `schedule` 方法即可实现定时任务逻辑。

在 `schedule` 方法块里面，前面四个参数我们原封不动地传递，最后一个重试次数参数减掉一，就是下一次建立连接时候的上下文信息。读者可以自行修改代码，更改到一个连接不上的服务端 Host 或者 Port，查看控制台日志就可以看到5次重连日志。

以上就是实现指数退避的客户端重连逻辑，接下来，我们来一起学习一下，客户端启动，我们的引导类`Bootstrap`除了指定线程模型，IO 模型，连接读写处理逻辑之外，他还可以干哪些事情？

## 客户端启动其他方法

### attr() 方法

```java
bootstrap.attr(AttributeKey.newInstance("clientName"), "nettyClient")
```

`attr()` 方法可以给客户端 Channel，也就是`NioSocketChannel`绑定自定义属性，然后我们可以通过`channel.attr()`取出这个属性，比如，上面的代码我们指定我们客户端 Channel 的一个`clientName`属性，属性值为`nettyClient`，其实说白了就是给`NioSocketChannel`维护一个 map 而已，后续在这个 `NioSocketChannel` 通过参数传来传去的时候，就可以通过他来取出设置的属性，非常方便。

### option() 方法

```java
Bootstrap
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
        .option(ChannelOption.SO_KEEPALIVE, true)
        .option(ChannelOption.TCP_NODELAY, true)
```

`option()` 方法可以给连接设置一些 TCP 底层相关的属性，比如上面，我们设置了三种 TCP 属性，其中

- `ChannelOption.CONNECT_TIMEOUT_MILLIS` 表示连接的超时时间，超过这个时间还是建立不上的话则代表连接失败
- `ChannelOption.SO_KEEPALIVE` 表示是否开启 TCP 底层心跳机制，true 为开启
- `ChannelOption.TCP_NODELAY` 表示是否开始 Nagle 算法，true 表示关闭，false 表示开启，通俗地说，如果要求高实时性，有数据发送时就马上发送，就设置为 true 关闭，如果需要减少发送次数减少网络交互，就设置为 false 开启

其他的参数这里就不一一讲解，有兴趣的同学可以去这个类里面自行研究。

## 总结

- 本文中，我们首先学习了 Netty 客户端启动的流程，一句话来说就是：创建一个引导类，然后给他指定线程模型，IO 模型，连接读写处理逻辑，连接上特定主机和端口，客户端就启动起来了。
- 然后，我们学习到 `connect` 方法是异步的，我们可以通过这个异步回调机制来实现指数退避重连逻辑。
- 最后呢，我们讨论了 Netty 客户端启动额外的参数，主要包括给客户端 Channel 绑定自定义属性值，设置底层 TCP 参数。



# 实战：客户端与服务端双向通信

在前面两个小节，我们已经学习了服务端启动与客户端启动的流程，熟悉了这两个过程之后，就可以建立服务端与客户端之间的通信了，本小节，我们用一个非常简单的 Demo 来了解一下服务端和客户端是如何来通信的。

> 本小节，我们要实现的功能是：客户端连接成功之后，向服务端写一段数据 ，服务端收到数据之后打印，并向客户端回一段数据，文章里面展示的是核心代码，完整代码请参考 [GitHub](https://github.com/lightningMan/flash-netty/tree/客户端与服务端双向通信) 



## 客户端发数据到服务端

在[客户端启动流程](https://juejin.im/book/5b4bc28bf265da0f60130116/section/5b4dafd4f265da0f98314cc7)这一小节，我们提到， 客户端相关的数据读写逻辑是通过 `Bootstrap` 的 `handler()` 方法指定 

```java
.handler(new ChannelInitializer<SocketChannel>() {
    @Override
    public void initChannel(SocketChannel ch) {
        // 指定连接数据读写逻辑
    }
});
```

现在，我们在 `initChannel()` 方法里面给客户端添加一个逻辑处理器，这个处理器的作用就是负责向服务端写数据

```java
.handler(new ChannelInitializer<SocketChannel>() {
    @Override
    public void initChannel(SocketChannel ch) {
        ch.pipeline().addLast(new FirstClientHandler());
    }
});
```

1. `ch.pipeline()` 返回的是和这条连接相关的逻辑处理链，采用了责任链模式，这里不理解没关系，后面会讲到
2. 然后再调用 `addLast()` 方法 添加一个逻辑处理器，这个逻辑处理器为的就是在客户端建立连接成功之后，向服务端写数据，下面是这个逻辑处理器相关的代码

```java
public class FirstClientHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        System.out.println(new Date() + ": 客户端写出数据");

        // 1. 获取数据
        ByteBuf buffer = getByteBuf(ctx);

        // 2. 写数据
        ctx.channel().writeAndFlush(buffer);
    }

    private ByteBuf getByteBuf(ChannelHandlerContext ctx) {
        // 1. 获取二进制抽象 ByteBuf
        ByteBuf buffer = ctx.alloc().buffer();
        
        // 2. 准备数据，指定字符串的字符集为 utf-8
        byte[] bytes = "你好，闪电侠!".getBytes(Charset.forName("utf-8"));

        // 3. 填充数据到 ByteBuf
        buffer.writeBytes(bytes);

        return buffer;
    }
}
```

1. 这个逻辑处理器继承自 **ChannelInboundHandlerAdapter**，然后**覆盖了 channelActive()方法 ，这个方法会在客户端连接建立成功之后被调用** 
2. 客户端连接建立成功之后，调用到 `channelActive()` 方法，在这个方法里面，我们编写向服务端写数据的逻辑
3. 写数据的逻辑分为两步：首先我们需要获取一个 netty 对二进制数据的抽象 `ByteBuf`，上面代码中, `ctx.alloc()` 获取到一个 `ByteBuf` 的内存管理器，这个 内存管理器的作用就是分配一个 `ByteBuf`，然后我们把字符串的二进制数据填充到 `ByteBuf`，这样我们就获取到了 Netty 需要的一个数据格式，最后我们调用 `ctx.channel().writeAndFlush()` 把数据写到服务端

以上就是客户端启动之后，向服务端写数据的逻辑，我们可以看到，和传统的 socket 编程不同的是，Netty 里面数据是以 ByteBuf 为单位的， 所有需要写出的数据都必须塞到一个 ByteBuf，数据的写出是如此，数据的读取亦是如此，接下来我们就来看一下服务端是如何读取到这段数据的。



## 服务端读取客户端数据

在[服务端端启动流程](https://juejin.im/book/5b4bc28bf265da0f60130116/section/5b4daf9ee51d4518f543f130)这一小节，我们提到， 服务端相关的数据处理逻辑是通过 `ServerBootstrap` 的 `childHandler()` 方法指定

```java
.childHandler(new ChannelInitializer<NioSocketChannel>() {
   protected void initChannel(NioSocketChannel ch) {
       // 指定连接数据读写逻辑
   }
});
```

现在，我们在 `initChannel()` 方法里面给服务端添加一个逻辑处理器，这个处理器的作用就是负责读取客户端来的数据

```java
.childHandler(new ChannelInitializer<NioSocketChannel>() {
    protected void initChannel(NioSocketChannel ch) {
        ch.pipeline().addLast(new FirstServerHandler());
    }
});
```

这个方法里面的逻辑和客户端侧类似，获取服务端侧关于这条连接的逻辑处理链 `pipeline`，然后添加一个逻辑处理器，负责读取客户端发来的数据

```java
public class FirstServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ByteBuf byteBuf = (ByteBuf) msg;

        System.out.println(new Date() + ": 服务端读到数据 -> " + byteBuf.toString(Charset.forName("utf-8")));
    }
}
```



服务端侧的逻辑处理器同样继承自 `ChannelInboundHandlerAdapter`，与客户端不同的是，这里覆盖的方法是 `channelRead()`，这个方法在接收到客户端发来的数据之后被回调。

这里的 `msg` 参数指的就是 Netty 里面数据读写的载体，为什么这里不直接是 `ByteBuf`，而需要我们强转一下，我们后面会分析到。这里我们强转之后，然后调用 `byteBuf.toString()` 就能够拿到我们客户端发过来的字符串数据。

我们先运行服务端，再运行客户端，下面分别是服务端控制台和客户端控制台的输出



## 服务端回数据给客户端

服务端向客户端写数据逻辑与客户端侧的写数据逻辑一样，先创建一个 `ByteBuf`，然后填充二进制数据，最后调用 `writeAndFlush()` 方法写出去，下面是服务端回数据的代码

```java
public class FirstServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        // ... 收数据逻辑省略

        // 回复数据到客户端
        System.out.println(new Date() + ": 服务端写出数据");
        ByteBuf out = getByteBuf(ctx);
        ctx.channel().writeAndFlush(out);
    }

    private ByteBuf getByteBuf(ChannelHandlerContext ctx) {
        byte[] bytes = "你好，欢迎关注我的微信公众号，《闪电侠的博客》!".getBytes(Charset.forName("utf-8"));

        ByteBuf buffer = ctx.alloc().buffer();

        buffer.writeBytes(bytes);

        return buffer;
    }
}
```



现在，轮到客户端了。客户端的读取数据的逻辑和服务端读取数据的逻辑一样，同样是覆盖 `ChannelRead()` 方法

```java
public class FirstClientHandler extends ChannelInboundHandlerAdapter {

    // 写数据相关的逻辑省略

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ByteBuf byteBuf = (ByteBuf) msg;

        System.out.println(new Date() + ": 客户端读到数据 -> " + byteBuf.toString(Charset.forName("utf-8")));
    }
}
```



将这段逻辑添加到客户端之后逻辑处理器 `FirstClientHandler` 之后，客户端就能收到服务端发来的数据，完整的代码参考 [GitHub](https://github.com/lightningMan/flash-netty/tree/客户端与服务端双向通信)

客户端与服务端的读写数据的逻辑完成之后，我们先运行服务端，再运行客户端，控制台输出如下

![](1.png) 



![](2.png) 

到这里，我们本小节要实现的客户端与服务端双向通信的功能实现完毕，最后，我们对本小节做一个总结。

## 总结

- 本文中，我们了解到客户端和服务端的逻辑处理是均是在启动的时候，通过给逻辑处理链 `pipeline` 添加逻辑处理器，来编写数据的读写逻辑，`pipeline` 的逻辑我们在后面会分析。 
- 接下来，我们学到，在**客户端连接成功之后会回调到逻辑处理器的 channelActive() 方法**，而**不管是服务端还是客户端，收到数据之后都会调用到 channelRead 方法**。
- 写数据调用`writeAndFlush`方法，客户端与服务端交互的二进制数据载体为 `ByteBuf`，`ByteBuf` 通过连接的内存管理器创建，字节数据填充到 `ByteBuf` 之后才能写到对端，接下来一小节，我们就来重点分析 `ByteBuf`。



# 数据传输载体 ByteBuf 介绍

 在前面一小节，我们已经了解到 Netty 里面数据读写是以 ByteBuf 为单位进行交互的，这一小节，我们就来详细剖析一下 ByteBuf 

## ByteBuf结构

首先，我们先来了解一下 ByteBuf 的结构

![](3.png) 



以上就是一个 ByteBuf 的结构图，从上面这幅图可以看到

1. **ByteBuf 是一个字节容器**，容器里面的的数据分为三个部分，**第一个部分是已经丢弃的字节**，这部分数据是无效的；**第二部分是可读字节，这部分数据是 ByteBuf 的主体数据， 从 ByteBuf 里面读取的数据都来自这一部分**;**最后一部分的数据是可写字节**，所有写到 ByteBuf 的数据都会写到这一段。最后一部分虚线表示的是该 ByteBuf 最多还能扩容多少容量
2. 以上三段内容是被两个指针给划分出来的，从左到右，依次是**读指针**（readerIndex）、**写指针**（writerIndex），然后还有一个变量 capacity，表示 ByteBuf 底层内存的总容量
3. 从 ByteBuf 中每读取一个字节，readerIndex 自增1，ByteBuf 里面总共有 writerIndex-readerIndex 个字节可读, 由此可以推论出当 readerIndex 与 writerIndex 相等的时候，ByteBuf 不可读
4. 写数据是从 writerIndex 指向的部分开始写，每写一个字节，writerIndex 自增1，直到增到 capacity，这个时候，表示 ByteBuf 已经不可写了
5. ByteBuf 里面其实还有一个参数 maxCapacity，当向 ByteBuf 写数据的时候，如果容量不足，那么这个时候可以进行扩容，直到 capacity 扩容到 maxCapacity，超过 maxCapacity 就会报错

Netty 使用 ByteBuf 这个数据结构可以有效地区分可读数据和可写数据，读写之间相互没有冲突，当然，ByteBuf 只是对二进制数据的抽象，具体底层的实现我们在下面的小节会讲到，在这一小节，我们 只需要知道 Netty 关于数据读写只认 ByteBuf，下面，我们就来学习一下 ByteBuf 常用的 API



## 容量 API

> capacity()

表示 ByteBuf 底层占用了多少字节的内存（包括丢弃的字节、可读字节、可写字节），不同的底层实现机制有不同的计算方式，后面我们讲 ByteBuf 的分类的时候会讲到

> maxCapacity()

表示 ByteBuf 底层最大能够占用多少字节的内存，当向 ByteBuf 中写数据的时候，如果发现容量不足，则进行扩容，直到扩容到 maxCapacity，超过这个数，就抛异常

> readableBytes() 与 isReadable()

readableBytes() 表示 ByteBuf 当前可读的字节数，它的值等于 writerIndex-readerIndex，如果两者相等，则不可读，isReadable() 方法返回 false

> writableBytes()、 isWritable() 与 maxWritableBytes()

writableBytes() 表示 ByteBuf 当前可写的字节数，它的值等于 capacity-writerIndex，如果两者相等，则表示不可写，isWritable() 返回 false，但是这个时候，并不代表不能往 ByteBuf 中写数据了， 如果发现往 ByteBuf 中写数据写不进去的话，Netty 会自动扩容 ByteBuf，直到扩容到底层的内存大小为 maxCapacity，而 maxWritableBytes() 就表示可写的最大字节数，它的值等于 maxCapacity-writerIndex

## 读写指针相关的 API

> readerIndex() 与 readerIndex(int)

前者表示返回当前的读指针 readerIndex, 后者表示设置读指针

> writeIndex() 与 writeIndex(int)

前者表示返回当前的写指针 writerIndex, 后者表示设置写指针

> markReaderIndex() 与 resetReaderIndex()

前者表示把当前的读指针保存起来，后者表示把当前的读指针恢复到之前保存的值，下面两段代码是等价的

```
// 代码片段1
int readerIndex = buffer.readerIndex();
// .. 其他操作
buffer.readerIndex(readerIndex);


// 代码片段二
buffer.markReaderIndex();
// .. 其他操作
buffer.resetReaderIndex();
```

希望大家多多使用代码片段二这种方式，不需要自己定义变量，无论 buffer 当作参数传递到哪里，调用 resetReaderIndex() 都可以恢复到之前的状态，在解析自定义协议的数据包的时候非常常见，推荐大家使用这一对 API

> markWriterIndex() 与 resetWriterIndex()

这一对 API 的作用与上述一对 API 类似，这里不再 赘述

## 读写 API

本质上，关于 ByteBuf 的读写都可以看作从指针开始的地方开始读写数据

> writeBytes(byte[] src) 与 buffer.readBytes(byte[] dst)

writeBytes() 表示把字节数组 src 里面的数据全部写到 ByteBuf，而 readBytes() 指的是把 ByteBuf 里面的数据全部读取到 dst，这里 dst 字节数组的大小通常等于 readableBytes()，而 src 字节数组大小的长度通常小于等于 writableBytes()

> writeByte(byte b) 与 buffer.readByte()

writeByte() 表示往 ByteBuf 中写一个字节，而 buffer.readByte() 表示从 ByteBuf 中读取一个字节，类似的 API 还有 writeBoolean()、writeChar()、writeShort()、writeInt()、writeLong()、writeFloat()、writeDouble() 与 readBoolean()、readChar()、readShort()、readInt()、readLong()、readFloat()、readDouble() 这里就不一一赘述了，相信读者应该很容易理解这些 API

与读写 API 类似的 API 还有 getBytes、getByte() 与 setBytes()、setByte() 系列，**唯一的区别就是 get/set 不会改变读写指针，而 read/write 会改变读写指针**，这点在解析数据的时候千万要注意

> release() 与 retain()

**由于 Netty 使用了堆外内存，而堆外内存是不被 jvm 直接管理的，也就是说申请到的内存无法被垃圾回收器直接回收，所以需要我们手动回收。有点类似于c语言里面，申请到的内存必须手工释放，否则会造成内存泄漏**。

**Netty 的 ByteBuf 是通过引用计数的方式管理的**，如果一个 ByteBuf 没有地方被引用到，需要回收底层内存。默认情况下，当创建完一个 ByteBuf，它的引用为1，然后每次调用 retain() 方法， 它的引用就加一， release() 方法原理是将引用计数减一，减完之后如果发现引用计数为0，则直接回收 ByteBuf 底层的内存。

> slice()、duplicate()、copy()

这三个方法通常情况会放到一起比较，这三者的返回值都是一个新的 ByteBuf 对象

1. slice() 方法从原始 ByteBuf 中截取一段，这段数据是从 readerIndex 到 writeIndex，同时，返回的新的 ByteBuf 的最大容量 maxCapacity 为原始 ByteBuf 的 readableBytes()
2. duplicate() 方法把整个 ByteBuf 都截取出来，包括所有的数据，指针信息
3. slice() 方法与 duplicate() 方法的相同点是：底层内存以及引用计数与原始的 ByteBuf 共享，也就是说经过 slice() 或者 duplicate() 返回的 ByteBuf 调用 write 系列方法都会影响到 原始的 ByteBuf，但是它们都维持着与原始 ByteBuf 相同的内存引用计数和不同的读写指针
4. slice() 方法与 duplicate() 不同点就是：slice() 只截取从 readerIndex 到 writerIndex 之间的数据，它返回的 ByteBuf 的最大容量被限制到 原始 ByteBuf 的 readableBytes(), 而 duplicate() 是把整个 ByteBuf 都与原始的 ByteBuf 共享
5. slice() 方法与 duplicate() 方法不会拷贝数据，它们只是通过改变读写指针来改变读写的行为，而最后一个方法 copy() 会直接从原始的 ByteBuf 中拷贝所有的信息，包括读写指针以及底层对应的数据，因此，往 copy() 返回的 ByteBuf 中写数据不会影响到原始的 ByteBuf
6. slice() 和 duplicate() 不会改变 ByteBuf 的引用计数，所以原始的 ByteBuf 调用 release() 之后发现引用计数为零，就开始释放内存，调用这两个方法返回的 ByteBuf 也会被释放，这个时候如果再对它们进行读写，就会报错。因此，我们可以通过调用一次 retain() 方法 来增加引用，表示它们对应的底层的内存多了一次引用，引用计数为2，在释放内存的时候，需要调用两次 release() 方法，将引用计数降到零，才会释放内存
7. 这三个方法均维护着自己的读写指针，与原始的 ByteBuf 的读写指针无关，相互之间不受影响

> retainedSlice() 与 retainedDuplicate()

相信读者应该已经猜到这两个 API 的作用了，它们的作用是在截取内存片段的同时，增加内存的引用计数，分别与下面两段代码等价

```java
// retainedSlice 等价于
slice().retain();

// retainedDuplicate() 等价于
duplicate().retain()
```

使用到 slice 和 duplicate 方法的时候，千万要理清 内存共享，引用计数共享，读写指针不共享 这几个概念，下面举两个常见的易犯错的例子

1. 多次释放

```java
Buffer buffer = xxx;
doWith(buffer);
// 一次释放
buffer.release();


public void doWith(Bytebuf buffer) {
// ...    
    
// 没有增加引用计数
Buffer slice = buffer.slice();

foo(slice);

}


public void foo(ByteBuf buffer) {
    // read from buffer
    
    // 重复释放
    buffer.release();
}
```

这里的 doWith 有的时候是用户自定义的方法，有的时候是 Netty 的回调方法，比如 `channelRead()` 等等

1. 不释放造成内存泄漏

```java
Buffer buffer = xxx;
doWith(buffer);
// 引用计数为2，调用 release 方法之后，引用计数为1，无法释放内存 
buffer.release();


public void doWith(Bytebuf buffer) {
// ...    
    
// 增加引用计数
Buffer slice = buffer.retainedSlice();

foo(slice);

// 没有调用 release

}


public void foo(ByteBuf buffer) {
    // read from buffer
}
```

想要避免以上两种情况发生，大家只需要记得一点，在一个函数体里面，只要增加了引用计数（包括 ByteBuf 的创建和手动调用 retain() 方法），就必须调用 release() 方法

## 实战

了解了以上 API 之后，最后我们使用上述 API 来 写一个简单的 demo

> ByteBufTest.java

```java
public class ByteBufTest {
    public static void main(String[] args) {
        ByteBuf buffer = ByteBufAllocator.DEFAULT.buffer(9, 100);

        print("allocate ByteBuf(9, 100)", buffer);

        // write 方法改变写指针，写完之后写指针未到 capacity 的时候，buffer 仍然可写
        buffer.writeBytes(new byte[]{1, 2, 3, 4});
        print("writeBytes(1,2,3,4)", buffer);

        // write 方法改变写指针，写完之后写指针未到 capacity 的时候，buffer 仍然可写, 写完 int 类型之后，写指针增加4
        buffer.writeInt(12);
        print("writeInt(12)", buffer);

        // write 方法改变写指针, 写完之后写指针等于 capacity 的时候，buffer 不可写
        buffer.writeBytes(new byte[]{5});
        print("writeBytes(5)", buffer);

        // write 方法改变写指针，写的时候发现 buffer 不可写则开始扩容，扩容之后 capacity 随即改变
        buffer.writeBytes(new byte[]{6});
        print("writeBytes(6)", buffer);

        // get 方法不改变读写指针
        System.out.println("getByte(3) return: " + buffer.getByte(3));
        System.out.println("getShort(3) return: " + buffer.getShort(3));
        System.out.println("getInt(3) return: " + buffer.getInt(3));
        print("getByte()", buffer);


        // set 方法不改变读写指针
        buffer.setByte(buffer.readableBytes() + 1, 0);
        print("setByte()", buffer);

        // read 方法改变读指针
        byte[] dst = new byte[buffer.readableBytes()];
        buffer.readBytes(dst);
        print("readBytes(" + dst.length + ")", buffer);

    }

    private static void print(String action, ByteBuf buffer) {
        System.out.println("after ===========" + action + "============");
        System.out.println("capacity(): " + buffer.capacity());
        System.out.println("maxCapacity(): " + buffer.maxCapacity());
        System.out.println("readerIndex(): " + buffer.readerIndex());
        System.out.println("readableBytes(): " + buffer.readableBytes());
        System.out.println("isReadable(): " + buffer.isReadable());
        System.out.println("writerIndex(): " + buffer.writerIndex());
        System.out.println("writableBytes(): " + buffer.writableBytes());
        System.out.println("isWritable(): " + buffer.isWritable());
        System.out.println("maxWritableBytes(): " + buffer.maxWritableBytes());
        System.out.println();
    }
}
```



最后，控制台输出

```
after ===========allocate ByteBuf(9, 100)============
capacity(): 9
maxCapacity(): 100
readerIndex(): 0
readableBytes(): 0
isReadable(): false
writerIndex(): 0
writableBytes(): 9
isWritable(): true
maxWritableBytes(): 100

after ===========writeBytes(1,2,3,4)============
capacity(): 9
maxCapacity(): 100
readerIndex(): 0
readableBytes(): 4
isReadable(): true
writerIndex(): 4
writableBytes(): 5
isWritable(): true
maxWritableBytes(): 96

after ===========writeInt(12)============
capacity(): 9
maxCapacity(): 100
readerIndex(): 0
readableBytes(): 8
isReadable(): true
writerIndex(): 8
writableBytes(): 1
isWritable(): true
maxWritableBytes(): 92

after ===========writeBytes(5)============
capacity(): 9
maxCapacity(): 100
readerIndex(): 0
readableBytes(): 9
isReadable(): true
writerIndex(): 9
writableBytes(): 0
isWritable(): false
maxWritableBytes(): 91

after ===========writeBytes(6)============
capacity(): 64
maxCapacity(): 100
readerIndex(): 0
readableBytes(): 10
isReadable(): true
writerIndex(): 10
writableBytes(): 54
isWritable(): true
maxWritableBytes(): 90

getByte(3) return: 4
getShort(3) return: 1024
getInt(3) return: 67108864
after ===========getByte()============
capacity(): 64
maxCapacity(): 100
readerIndex(): 0
readableBytes(): 10
isReadable(): true
writerIndex(): 10
writableBytes(): 54
isWritable(): true
maxWritableBytes(): 90

after ===========setByte()============
capacity(): 64
maxCapacity(): 100
readerIndex(): 0
readableBytes(): 10
isReadable(): true
writerIndex(): 10
writableBytes(): 54
isWritable(): true
maxWritableBytes(): 90

after ===========readBytes(10)============
capacity(): 64
maxCapacity(): 100
readerIndex(): 10
readableBytes(): 0
isReadable(): false
writerIndex(): 10
writableBytes(): 54
isWritable(): true
maxWritableBytes(): 90
```

> 完整代码已放置 [github](https://github.com/lightningMan/flash-netty/tree/数据传输载体ByteBuf)

相信大家在了解了 ByteBuf 的结构之后，不难理解控制台的输出

## 总结

1. 本小节，我们分析了 Netty 对二进制数据的抽象 ByteBuf 的结构，本质上它的原理就是，它引用了一段内存，这段内存可以是堆内也可以是堆外的，然后用引用计数来控制这段内存是否需要被释放，使用读写指针来控制对 ByteBuf 的读写，可以理解为是外观模式的一种使用
2. 基于读写指针和容量、最大可扩容容量，衍生出一系列的读写方法，要注意 read/write 与 get/set 的区别
3. 多个 ByteBuf 可以引用同一段内存，通过引用计数来控制内存的释放，遵循谁 retain() 谁 release() 的原则
4. 最后，我们通过一个具体的例子说明 ByteBuf 的实际使用



# 客户端与服务端通信协议编解码



## 什么是服务端与客户端的通信协议

无论是使用 Netty 还是原始的 Socket 编程，基于 TCP 通信的数据包格式均为二进制，协议指的就是客户端与服务端事先商量好的，每一个二进制数据包中每一段字节分别代表什么含义的规则。如下图的一个简单的登录指令



![](4.png) 

在这个数据包中，第一个字节为 1 表示这是一个登录指令，接下来是用户名和密码，这两个值以 `\0` 分割，客户端发送这段二进制数据包到服务端，服务端就能根据这个协议来取出用户名密码，进行登录逻辑。实际的通信协议设计中，我们会考虑更多细节，要比这个稍微复杂一些。

那么，协议设计好之后，客户端与服务端的通信过程又是怎样的呢？

![](5.png) 

如上图所示，客户端与服务端通信

1. 首先，客户端把一个 Java 对象按照通信协议转换成二进制数据包。
2. 然后通过网络，把这段二进制数据包发送到服务端，数据的传输过程由 TCP/IP 协议负责数据的传输，与我们的应用层无关。
3. 服务端接受到数据之后，按照协议取出二进制数据包中的相应字段，包装成 Java 对象，交给应用逻辑处理。
4. 服务端处理完之后，如果需要吐出响应给客户端，那么按照相同的流程进行。

在本小册子的[第一节](https://juejin.im/book/5b4bc28bf265da0f60130116/section/5b6a1a9cf265da0f87595521)中，我们已经列出了实现一个支持单聊和群聊的 IM 的指令集合，我们设计协议的目的就是为了客户端与服务端能够识别出这些具体的指令。 接下来，我们就来看一下如何设计这个通信协议。

## 通信协议的设计

![](6.png) 

1. 首先，第一个字段是魔数，通常情况下为固定的几个字节（我们这边规定为4个字节）。 为什么需要这个字段，而且还是一个固定的数？假设我们在服务器上开了一个端口，比如 80 端口，如果没有这个魔数，任何数据包传递到服务器，服务器都会根据自定义协议来进行处理，包括不符合自定义协议规范的数据包。例如，我们直接通过 `http://服务器ip` 来访问服务器（默认为 80 端口）， 服务端收到的是一个标准的 HTTP 协议数据包，但是它仍然会按照事先约定好的协议来处理 HTTP 协议，显然，这是会解析出错的。而有了这个魔数之后，服务端首先取出前面四个字节进行比对，能够在第一时间识别出这个数据包并非是遵循自定义协议的，也就是无效数据包，为了安全考虑可以直接关闭连接以节省资源。在 Java 的字节码的二进制文件中，开头的 4 个字节为`0xcafebabe` 用来标识这是个字节码文件，亦是异曲同工之妙。
2. 接下来一个字节为版本号，通常情况下是预留字段，用于协议升级的时候用到，有点类似 TCP 协议中的一个字段标识是 IPV4 协议还是 IPV6 协议，大多数情况下，这个字段是用不到的，不过为了协议能够支持升级，我们还是先留着。
3. 第三部分，序列化算法表示如何把 Java 对象转换二进制数据以及二进制数据如何转换回 Java 对象，比如 Java 自带的序列化，json，hessian 等序列化方式。
4. 第四部分的字段表示指令，关于指令相关的介绍，我们在前面已经讨论过，服务端或者客户端每收到一种指令都会有相应的处理逻辑，这里，我们用一个字节来表示，最高支持256种指令，对于我们这个 IM 系统来说已经完全足够了。
5. 接下来的字段为数据部分的长度，占四个字节。
6. 最后一个部分为数据内容，每一种指令对应的数据是不一样的，比如登录的时候需要用户名密码，收消息的时候需要用户标识和具体消息内容等等。

通常情况下，这样一套标准的协议能够适配大多数情况下的服务端与客户端的通信场景，接下来我们就来看一下我们如何使用 Netty 来实现这套协议。

## 通信协议的实现

我们把 Java 对象根据协议封装成二进制数据包的过程成为编码，而把从二进制数据包中解析出 Java 对象的过程成为解码，在学习如何使用 Netty 进行通信协议的编解码之前，我们先来定义一下客户端与服务端通信的 Java 对象。

### Java 对象

我们如下定义通信过程中的 Java 对象

```java
@Data
public abstract class Packet {
    /**
     * 协议版本
     */
    private Byte version = 1;

    /**
    * 指令
    * /
    public abstract Byte getCommand();
}
```

1. 以上是通信过程中 Java 对象的抽象类，可以看到，我们定义了一个版本号（默认值为 1 ）以及一个获取指令的抽象方法，所有的指令数据包都必须实现这个方法，这样我们就可以知道某种指令的含义。
2. `@Data` 注解由 [lombok](https://www.projectlombok.org/) 提供，它会自动帮我们生产 getter/setter 方法，减少大量重复代码，推荐使用。



接下来，我们拿客户端登录请求为例，定义登录请求数据包

```java
public interface Command {

    Byte LOGIN_REQUEST = 1;
}

@Data
public class LoginRequestPacket extends Packet {
    private Integer userId;

    private String username;

    private String password;

    @Override
    public Byte getCommand() {
        
        return LOGIN_REQUEST;
    }
}
```

登录请求数据包继承自 `Packet`，然后定义了三个字段，分别是用户 ID，用户名，密码，这里最为重要的就是覆盖了父类的 `getCommand()` 方法，值为常量 `LOGIN_REQUEST`。

Java 对象定义完成之后，接下来我们就需要定义一种规则，如何把一个 Java 对象转换成二进制数据，这个规则叫做 Java 对象的序列化。

### 序列化

我们如下定义序列化接口

```java
public interface Serializer {

    /**
     * 序列化算法
     */
    byte getSerializerAlgorithm();
    
    /**
     * java 对象转换成二进制
     */
    byte[] serialize(Object object);

    /**
     * 二进制转换成 java 对象
     */
    <T> T deserialize(Class<T> clazz, byte[] bytes);
}
```

序列化接口有三个方法，`getSerializerAlgorithm()` 获取具体的序列化算法标识，`serialize()` 将 Java 对象转换成字节数组，`deserialize()` 将字节数组转换成某种类型的 Java 对象，在本小册中，我们使用最简单的 json 序列化方式，使用阿里巴巴的 [fastjson](https://github.com/alibaba/fastjson) 作为序列化框架。

```java
public interface SerializerAlgorithm {
    /**
     * json 序列化标识
     */
    byte JSON = 1;
}


public class JSONSerializer implements Serializer {
   
    @Override
    public byte getSerializerAlgorithm() {
        
        return SerializerAlgorithm.JSON;
    } 

    @Override
    public byte[] serialize(Object object) {
        
        return JSON.toJSONBytes(object);
    }

    @Override
    public <T> T deserialize(Class<T> clazz, byte[] bytes) {
        
        return JSON.parseObject(bytes, clazz);
    }
}
```

然后，我们定义一下序列化算法的类型以及默认序列化算法

```java
public interface Serializer {
    /**
     * json 序列化
     */
    byte JSON_SERIALIZER = 1;

    Serializer DEFAULT = new JSONSerializer();

    // ...
}
```

这样，我们就实现了序列化相关的逻辑，如果想要实现其他序列化算法的话，只需要继承一下 `Serializer`，然后定义一下序列化算法的标识，再覆盖一下两个方法即可。

序列化定义了 Java 对象与二进制数据的互转过程，接下来，我们就来学习一下，如何把这部分的数据编码到通信协议的二进制数据包中去。

### 编码：封装成二进制的过程

> PacketCodeC.java

```java
private static final int MAGIC_NUMBER = 0x12345678;

public ByteBuf encode(Packet packet) {
    // 1. 创建 ByteBuf 对象
    ByteBuf byteBuf = ByteBufAllocator.DEFAULT.ioBuffer();
    // 2. 序列化 Java 对象
    byte[] bytes = Serializer.DEFAULT.serialize(packet);

    // 3. 实际编码过程
    byteBuf.writeInt(MAGIC_NUMBER);
    byteBuf.writeByte(packet.getVersion());
    byteBuf.writeByte(Serializer.DEFAULT.getSerializerAlgorithm());
    byteBuf.writeByte(packet.getCommand());
    byteBuf.writeInt(bytes.length);
    byteBuf.writeBytes(bytes);

    return byteBuf;
}
```

编码过程分为三个过程

1. 首先，我们需要创建一个 ByteBuf，这里我们调用 Netty 的 ByteBuf 分配器来创建，`ioBuffer()` 方法会返回适配 io 读写相关的内存，它会尽可能创建一个直接内存，直接内存可以理解为不受 jvm 堆管理的内存空间，写到 IO 缓冲区的效果更高。
2. 接下来，我们将 Java 对象序列化成二进制数据包。
3. 最后，我们对照本小节开头协议的设计以及上一小节 ByteBuf 的 API，逐个往 ByteBuf 写入字段，即实现了编码过程，到此，编码过程结束。

一端实现了编码之后，Netty 会将此 ByteBuf 写到另外一端，另外一端拿到的也是一个 ByteBuf 对象，基于这个 ByteBuf 对象，就可以反解出在对端创建的 Java 对象，这个过程我们称作为解码，下面，我们就来分析一下这个过程。

### 解码：解析 Java 对象的过程

> PacketCodeC.java

```java
public Packet decode(ByteBuf byteBuf) {
    // 跳过 magic number
    byteBuf.skipBytes(4);

    // 跳过版本号
    byteBuf.skipBytes(1);

    // 序列化算法标识
    byte serializeAlgorithm = byteBuf.readByte();

    // 指令
    byte command = byteBuf.readByte();

    // 数据包长度
    int length = byteBuf.readInt();

    byte[] bytes = new byte[length];
    byteBuf.readBytes(bytes);

    Class<? extends Packet> requestType = getRequestType(command);
    Serializer serializer = getSerializer(serializeAlgorithm);

    if (requestType != null && serializer != null) {
        return serializer.deserialize(requestType, bytes);
    }

    return null;
}
```

解码的流程如下：

1. 我们假定 `decode` 方法传递进来的 `ByteBuf` 已经是合法的（在后面小节我们再来实现校验），即首四个字节是我们前面定义的魔数 `0x12345678`，这里我们调用 `skipBytes` 跳过这四个字节。
2. 这里，我们暂时不关注协议版本，通常我们在没有遇到协议升级的时候，这个字段暂时不处理，因为，你会发现，绝大多数情况下，这个字段几乎用不着，但我们仍然需要暂时留着。
3. 接下来，我们调用 `ByteBuf` 的 API 分别拿到序列化算法标识、指令、数据包的长度。
4. 最后，我们根据拿到的数据包的长度取出数据，通过指令拿到该数据包对应的 Java 对象的类型，根据序列化算法标识拿到序列化对象，将字节数组转换为 Java 对象，至此，解码过程结束。

可以看到，解码过程与编码过程正好是一个相反的过程。

## 总结

本小节，我们学到了如下几个知识点

1. 通信协议是为了服务端与客户端交互，双方协商出来的满足一定规则的二进制数据格式。
2. 介绍了一种通用的通信协议的设计，包括魔数、版本号、序列化算法标识、指令、数据长度、数据几个字段，该协议能够满足绝大多数的通信场景。
3. Java 对象以及序列化，目的就是实现 Java 对象与二进制数据的互转。
4. 最后，我们依照我们设计的协议以及 ByteBuf 的 API 实现了通信协议，这个过程称为编解码过程。

> 本小节完整代码参考 [github](https://github.com/lightningMan/flash-netty/tree/客户端与服务端通信协议编解码)， 其中 `PacketCodeCTest.java` 对编解码过程的测试用例，大家可以下载下来跑一跑。



# Netty实现登录和首发消息

## 登录流程

![](7.png) 

从上图中我们可以看到，客户端连接上服务端之后

1. 客户端会构建一个登录请求对象，然后通过编码把请求对象编码为 ByteBuf，写到服务端。
2. 服务端接受到 ByteBuf 之后，首先通过解码把 ByteBuf 解码为登录请求响应，然后进行校验。
3. 服务端校验通过之后，构造一个登录响应对象，依然经过编码，然后再写回到客户端。
4. 客户端接收到服务端的之后，解码 ByteBuf，拿到登录响应响应，判断是否登陆成功



## 逻辑处理器

接下来，我们分别实现一下上述四个过程，开始之前，我们先来回顾一下客户端与服务端的启动流程，客户端启动的时候，我们会在引导类 `Bootstrap` 中配置客户端的处理逻辑，本小节中，我们给客户端配置的逻辑处理器叫做 `ClientHandler`

```
public class ClientHandler extends ChannelInboundHandlerAdapter {
}
```

然后，客户端启动的时候，我们给 `Bootstrap` 配置上这个逻辑处理器

```java
	bootstrap.handler(new ChannelInitializer<SocketChannel>() {
            @Override
            public void initChannel(SocketChannel ch) {
                ch.pipeline().addLast(new ClientHandler());
            }
        });
```

这样，在客户端侧，Netty 中 IO 事件相关的回调就能够回调到我们的 `ClientHandler`。

同样，我们给服务端引导类 `ServerBootstrap` 也配置一个逻辑处理器 `ServerHandler`

```java
public class ServerHandler extends ChannelInboundHandlerAdapter {
}


serverBootstrap.childHandler(new ChannelInitializer<NioSocketChannel>() {
            protected void initChannel(NioSocketChannel ch) {
                ch.pipeline().addLast(new ServerHandler());
            }
        }
```

这样，在服务端侧，Netty 中 IO 事件相关的回调就能够回调到我们的 `ServerHandler`。

接下来，我们就围绕这两个 Handler 来编写我们的处理逻辑。

## 客户端发送登录请求

### 客户端处理登录请求

我们实现在客户端连接上服务端之后，立即登录。在连接上服务端之后，Netty 会回调到 `ClientHandler` 的 `channelActive()` 方法，我们在这个方法体里面编写相应的逻辑

> ClientHandler.java

```java
public void channelActive(ChannelHandlerContext ctx) {
    System.out.println(new Date() + ": 客户端开始登录");

    // 创建登录对象
    LoginRequestPacket loginRequestPacket = new LoginRequestPacket();
    loginRequestPacket.setUserId(UUID.randomUUID().toString());
    loginRequestPacket.setUsername("flash");
    loginRequestPacket.setPassword("pwd");

    // 编码
    ByteBuf buffer = PacketCodeC.INSTANCE.encode(ctx.alloc(), loginRequestPacket);

    // 写数据
    ctx.channel().writeAndFlush(buffer);
}
```

这里，我们按照前面所描述的三个步骤来分别实现，在编码的环节，我们把 `PacketCodeC` 变成单例模式，然后把 `ByteBuf` 分配器抽取出一个参数，这里第一个实参 `ctx.alloc()` 获取的就是与当前连接相关的 `ByteBuf` 分配器，建议这样来使用。

写数据的时候，我们通过 `ctx.channel()` 获取到当前连接（Netty 对连接的抽象为 Channel，后面小节会分析），然后调用 `writeAndFlush()` 就能把二进制数据写到服务端。这样，客户端发送登录请求的逻辑就完成了，接下来，我们来看一下，服务端接受到这个数据之后是如何来处理的。

### 服务端处理登录请求

> ServerHandler.java

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ByteBuf requestByteBuf = (ByteBuf) msg;

    // 解码
    Packet packet = PacketCodeC.INSTANCE.decode(requestByteBuf);

    // 判断是否是登录请求数据包
    if (packet instanceof LoginRequestPacket) {
        LoginRequestPacket loginRequestPacket = (LoginRequestPacket) packet;

        // 登录校验
        if (valid(loginRequestPacket)) {
            // 校验成功
        } else {
            // 校验失败
        }
    }
}

private boolean valid(LoginRequestPacket loginRequestPacket) {
    return true;
}
```

我们向服务端引导类 `ServerBootstrap` 中添加了逻辑处理器 `ServerHandler` 之后，Netty 在收到数据之后，会回调 `channelRead()` 方法，这里的第二个参数 `msg`，在我们这个场景中，可以直接强转为 `ByteBuf`，为什么 Netty 不直接把这个参数类型定义为 `ByteBuf` ？我们在后续的小节会分析到。

拿到 `ByteBuf` 之后，首先要做的事情就是解码，解码出 java 数据包对象，然后判断如果是登录请求数据包 `LoginRequestPacket`，就进行登录逻辑的处理，这里，我们假设所有的登录都是成功的，`valid()` 方法返回 true。 服务端校验通过之后，接下来就需要向客户端发送登录响应，我们继续编写服务端的逻辑。

## 服务端发送登录响应

### 服务端处理登录响应

> ServerHandler.java

```java
LoginResponsePacket loginResponsePacket = new LoginResponsePacket();
loginResponsePacket.setVersion(packet.getVersion());
if (valid(loginRequestPacket)) {
    loginResponsePacket.setSuccess(true);
} else {
    loginResponsePacket.setReason("账号密码校验失败");
    loginResponsePacket.setSuccess(false);
}
// 编码
ByteBuf responseByteBuf = PacketCodeC.INSTANCE.encode(ctx.alloc(), loginResponsePacket);
ctx.channel().writeAndFlush(responseByteBuf);
```

这段逻辑仍然是在服务端逻辑处理器 `ServerHandler` 的 `channelRead()` 方法里，我们构造一个登录响应包 `LoginResponsePacket`，然后在校验成功和失败的时候分别设置标志位，接下来，调用编码器把 Java 对象编码成 `ByteBuf`，调用 `writeAndFlush()` 写到客户端，至此，服务端的登录逻辑编写完成，接下来，我们还有最后一步，客户端处理登录响应。

### 客户端处理登录响应

> ClientHandler.java

客户端接收服务端数据的处理逻辑也是在 `ClientHandler` 的 `channelRead()` 方法

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ByteBuf byteBuf = (ByteBuf) msg;

    Packet packet = PacketCodeC.INSTANCE.decode(byteBuf);

    if (packet instanceof LoginResponsePacket) {
        LoginResponsePacket loginResponsePacket = (LoginResponsePacket) packet;

        if (loginResponsePacket.isSuccess()) {
            System.out.println(new Date() + ": 客户端登录成功");
        } else {
            System.out.println(new Date() + ": 客户端登录失败，原因：" + loginResponsePacket.getReason());
        }
    }
}
```

客户端拿到数据之后，调用 `PacketCodeC` 进行解码操作，如果类型是登录响应数据包，我们这里逻辑比较简单，在控制台打印出一条消息。

至此，客户端整个登录流程到这里就结束了，这里为了给大家演示，我们的客户端和服务端的处理逻辑较为简单，但是相信大家应该已经掌握了使用 Netty 来做服务端与客户端交互的基本思路，基于这个思路，再运用到实际项目中，并不是难事。

最后，我们再来看一下效果，下面分别是客户端与服务端的控制台输出，完整的代码参考 [GitHub](https://github.com/lightningMan/flash-netty/tree/实现客户端登录), 分别启动 `NettyServer.java` 与 `NettyClient.java` 即可看到效果。

> 服务端

![](8.png)

 客户端 

![](9.png)



## 总结

本小节，我们们梳理了一下客户端登录的基本流程，然后结合上一小节的编解码逻辑，我们使用 Netty 实现了完整的客户端登录流程。

## 思考

客户端登录成功或者失败之后，如果把成功或者失败的标识绑定在客户端的连接上？



# 实战：实现客户端与服务端收发消息

> 这一小节，我们来实现客户端与服务端收发消息，我们要实现的具体功能是：在控制台输入一条消息之后按回车，校验完客户端的登录状态之后，把消息发送到服务端，服务端收到消息之后打印并且向客户端发送一条消息，客户端收到之后打印。

## 收发消息对象

首先，我们来定义一下客户端与服务端的收发消息对象，我们把客户端发送至服务端的消息对象定义为 `MessageRequestPacket`。

```
@Data
public class MessageRequestPacket extends Packet {

    private String message;

    @Override
    public Byte getCommand() {
        return MESSAGE_REQUEST;
    }
}
```

指令为 `MESSAGE_REQUEST ＝ 3`

我们把服务端发送至客户端的消息对象定义为 `MessageResponsePacket`

```
@Data
public class MessageResponsePacket extends Packet {

    private String message;

    @Override
    public Byte getCommand() {

        return MESSAGE_RESPONSE;
    }
}
```

指令为 `MESSAGE_RESPONSE = 4`

至此，我们的指令已经有如下四种

```
public interface Command {

    Byte LOGIN_REQUEST = 1;

    Byte LOGIN_RESPONSE = 2;

    Byte MESSAGE_REQUEST = 3;

    Byte MESSAGE_RESPONSE = 4;
}
```

## 判断客户端是否登录成功

在[前面一小节](https://juejin.im/book/5b4bc28bf265da0f60130116/section/5b4db04be51d45191556ee9c)，我们在文末出了一道思考题：如何判断客户端是否已经登录？

在[客户端启动流程](https://juejin.im/book/5b4bc28bf265da0f60130116/section/5b4dafd4f265da0f98314cc7)这一章节，我们有提到可以给客户端连接，也就是 Channel 绑定属性，通过 `channel.attr(xxx).set(xx)` 的方式，那么我们是否可以在登录成功之后，给 Channel 绑定一个登录成功的标志位，然后判断是否登录成功的时候取出这个标志位就可以了呢？答案是肯定的

我们先来定义一下是否登录成功的标志位

```
public interface Attributes {
    AttributeKey<Boolean> LOGIN = AttributeKey.newInstance("login");
}
```

然后，我们在客户端登录成功之后，给客户端绑定登录成功的标志位

> ClientHandler.java

```
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    // ...
        if (loginResponsePacket.isSuccess()) {
            LoginUtil.markAsLogin(ctx.channel());
            System.out.println(new Date() + ": 客户端登录成功");
        } else {
            System.out.println(new Date() + ": 客户端登录失败，原因：" + loginResponsePacket.getReason());
        }
    // ...
}
```

这里，我们省去了非关键代码部分

```
public class LoginUtil {
    public static void markAsLogin(Channel channel) {
        channel.attr(Attributes.LOGIN).set(true);
    }

    public static boolean hasLogin(Channel channel) {
        Attribute<Boolean> loginAttr = channel.attr(Attributes.LOGIN);

        return loginAttr.get() != null;
    }
}
```

如上所示，我们抽取出 `LoginUtil` 用于设置登录标志位以及判断是否有标志位，如果有标志位，不管标志位的值是什么，都表示已经成功登录过，接下来，我们来实现控制台输入消息并发送至服务端。

## 控制台输入消息并发送

在[客户端启动](https://juejin.im/book/5b4bc28bf265da0f60130116/section/5b4daf9ee51d4518f543f130)这小节中，我们已经学到了客户端的启动流程，现在，我们在客户端连接上服务端之后启动控制台线程，从控制台获取消息，然后发送至服务端

> NettyClient.java

```
private static void connect(Bootstrap bootstrap, String host, int port, int retry) {
    bootstrap.connect(host, port).addListener(future -> {
        if (future.isSuccess()) {
            Channel channel = ((ChannelFuture) future).channel();
            // 连接成功之后，启动控制台线程
            startConsoleThread(channel);
        } 
        // ...
    });
}

private static void startConsoleThread(Channel channel) {
    new Thread(() -> {
        while (!Thread.interrupted()) {
            if (LoginUtil.hasLogin(channel)) {
                System.out.println("输入消息发送至服务端: ");
                Scanner sc = new Scanner(System.in);
                String line = sc.nextLine();
                
                MessageRequestPacket packet = new MessageRequestPacket();
                packet.setMessage(line);
                ByteBuf byteBuf = PacketCodeC.INSTANCE.encode(channel.alloc(), packet);
                channel.writeAndFlush(byteBuf);
            }
        }
    }).start();
}
```

这里，我们省略了非关键代码，连接成功之后，我们调用 `startConsoleThread()` 开始启动控制台线程，然后在控制台线程中，判断只要当前 channel 是登录状态，就允许控制台输入消息。

从控制台获取消息之后，将消息封装成消息对象，然后将消息编码成 `ByteBuf`，最后通过 `writeAndFlush()` 将消息写到服务端，这个过程相信大家在学习了上小节的内容之后，应该不会太陌生。接下来，我们来看一下服务端收到消息之后是如何来处理的。

## 服务端收发消息处理

> ServerHandler.java

```
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ByteBuf requestByteBuf = (ByteBuf) msg;

    Packet packet = PacketCodeC.INSTANCE.decode(requestByteBuf);

    if (packet instanceof LoginRequestPacket) {
        // 处理登录..
    } else if (packet instanceof MessageRequestPacket) {
        // 处理消息
        MessageRequestPacket messageRequestPacket = ((MessageRequestPacket) packet);
        System.out.println(new Date() + ": 收到客户端消息: " + messageRequestPacket.getMessage());

        MessageResponsePacket messageResponsePacket = new MessageResponsePacket();
        messageResponsePacket.setMessage("服务端回复【" + messageRequestPacket.getMessage() + "】");
        ByteBuf responseByteBuf = PacketCodeC.INSTANCE.encode(ctx.alloc(), messageResponsePacket);
        ctx.channel().writeAndFlush(responseByteBuf);
    }
}
```

服务端在收到消息之后，仍然是回调到 `channelRead()` 方法，解码之后用一个 `else` 分支进入消息处理的流程。

首先，服务端将收到的消息打印到控制台，然后封装一个消息响应对象 `MessageResponsePacket`，接下来还是老样子，先编码成 `ByteBuf`，然后调用 `writeAndFlush()` 将数据写到客户端，最后，我们再来看一下客户端收到消息的逻辑。

## 客户端收消息处理

> ClientHandler.java

```
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ByteBuf byteBuf = (ByteBuf) msg;

    Packet packet = PacketCodeC.INSTANCE.decode(byteBuf);

    if (packet instanceof LoginResponsePacket) {
        // 登录逻辑...
    } else if (packet instanceof MessageResponsePacket) {
        MessageResponsePacket messageResponsePacket = (MessageResponsePacket) packet;
        System.out.println(new Date() + ": 收到服务端的消息: " + messageResponsePacket.getMessage());
    }
}
```

客户端在收到消息之后，回调到 `channelRead()` 方法，仍然用一个 `else` 逻辑进入到消息处理的逻辑，这里我们仅仅是简单地打印出消息，最后，我们再来看一下服务端和客户端的运行效果

> 完整的代码参考 [github](https://github.com/lightningMan/flash-netty/tree/实现客户端与服务端收发消息), 分别启动 `NettyServer.java` 与 `NettyClient.java` 即可看到效果。

## 控制台输出

> 客户端

![](10.png) 

 服务端 

![](11.png) 

## 总结

在本小节中

1. 我们定义了收发消息的 Java 对象进行消息的收发。
2. 然后我们学到了 channel 的 `attr()` 的实际用法：可以通过给 channel 绑定属性来设置某些状态，获取某些状态，不需要额外的 map 来维持。
3. 接着，我们学习了如何在控制台获取消息并且发送至服务端。
4. 最后，我们实现了服务端回消息，客户端响应的逻辑，可以看到，这里的部分实际上和前面一小节的登录流程有点类似。

## 思考

随着我们实现的指令越来越多，如何避免 `channelRead()` 中对指令处理的 `if else` 泛滥？ 

# pipeline 与 channelHandler

> 这一小节，我们将会学习 Netty 里面一大核心组件： Pipeline 与 ChannelHandler

上一小节的最后，我们提出一个问题：如何避免 `else` 泛滥？我们注意到，不管是服务端还是客户端，处理流程大致分为以下几个步骤

![](12.png) 



我们把这三类逻辑都写在一个类里面，客户端写在 `ClientHandler`，服务端写在 `ServerHandler`，如果要做功能的扩展（比如，我们要校验 magic number，或者其他特殊逻辑），只能在一个类里面去修改， 这个类就会变得越来越臃肿。

另外，我们注意到，每次发指令数据包都要手动调用编码器编码成 `ByteBuf`，对于这类场景的编码优化，我们能想到的办法自然是模块化处理，不同的逻辑放置到单独的类来处理，最后将这些逻辑串联起来，形成一个完整的逻辑处理链。

Netty 中的 pipeline 和 channelHandler 正是用来解决这个问题的：它通过责任链设计模式来组织代码逻辑，并且能够支持逻辑的动态添加和删除 ，Netty 能够支持各类协议的扩展，比如 HTTP，Websocket，Redis，靠的就是 pipeline 和 channelHandler，下面，我们就来一起学习一下这部分内容。



## pipeline 与 channelHandler 的构成

















































 关于 RPC 里使用 Netty 的最佳范例，推荐蚂蚁金服开源框架 [SOFABolt](https://github.com/alipay/sofa-bolt)，可以说是对 Netty 编程最佳实践的提炼 

毋庸置疑，通常情况下，官网一直是学习一门技术最好的资料

[https://netty.io/wiki/user-guide-for-4.x.html](https://netty.io/wiki/user-guide-for-4.x.html) 这里是官方给出的 4.x 版本的 Netty 的一个学习指引，大家可以温习一遍。 [https://netty.io/wiki/new-and-noteworthy-in-4.1.html](https://netty.io/wiki/new-and-noteworthy-in-4.1.html) 4.1x版本的新的特性，大家也可以感受一下 Netty 的版本变化。

另外，大家也可以了解一下目前有哪些开源项目使用了 Netty：[Netty.io/wiki/relate…](https://netty.io/wiki/related-projects.html)

关于 Netty 详细的 demo，也可以在官网找到，比如，你想使用 Netty 来实现一个 Http 服务器，或者实现一个 Websocket 服务器，或者实现 redis 协议等等，都可以在 [官方提供的 demo](https://github.com/netty/netty/tree/4.1/example/src/main/java/io/netty/example) 中找到。



# 脑图



![](brain.png) 
