---
title: Netty学习实录
abbrlink: 65d26b16
date: 2024-10-22 17:14:26
tags:
    - Netty
    - NIO
    - IO多路复用
    - 网络通信
categories: 网络编程
---

## 前言
关于IO模型的一些概念，在Tomcat文章中有所介绍：[Tomcat学习实录](https://nimbusk.cc/post/2f453177)
这篇围绕netty以及关于hotspot和linux底层epoll原理做学习记录
<!-- more -->


## Java中的NIO多路复用-ServerSocketChannel
服务端使用最多的模型，在Java层面使用的是SocketChannel。
一段代码：

```java
public class NioSelectorServer {

    static List<SocketChannel> channelList = new ArrayList<>();

    public static void main(String[] args) throws IOException, InterruptedException {

        // 创建NIO ServerSocketChannel,与BIO的serverSocket类似
        ServerSocketChannel serverSocket = ServerSocketChannel.open();
        serverSocket.socket().bind(new InetSocketAddress(9000));
        // 设置ServerSocketChannel为非阻塞
        serverSocket.configureBlocking(false);
        System.out.println("服务启动成功");

        while (true) {
            // 非阻塞模式accept方法不会阻塞，否则会阻塞
            // NIO的非阻塞是由操作系统内部实现的，底层调用了linux内核的accept函数
            SocketChannel socketChannel = serverSocket.accept();
            if (socketChannel != null) { // 如果有客户端进行连接
                System.out.println("连接成功");
                // 设置SocketChannel为非阻塞
                socketChannel.configureBlocking(false);
                // 保存客户端连接在List中
                channelList.add(socketChannel);
            }
            // 遍历连接进行数据读取
            Iterator<SocketChannel> iterator = channelList.iterator();
            while (iterator.hasNext()) {
                SocketChannel sc = iterator.next();
                ByteBuffer byteBuffer = ByteBuffer.allocate(128);
                // 非阻塞模式read方法不会阻塞，否则会阻塞
                int len = sc.read(byteBuffer);
                // 如果有数据，把数据打印出来
                if (len > 0) {
                    System.out.println("接收到消息：" + new String(byteBuffer.array()));
                } else if (len == -1) { // 如果客户端断开，把socket从集合中去掉
                    iterator.remove();
                    System.out.println("客户端断开连接");
                }
            }
        }

    }
}
```

这段代码是典型的Java场景使用NIO完成通信的服务端代码，对应的NIO有三大核心组件：Channel(通道)， Buffer(缓冲区)，Selector(多路复用器)
- channel 类似于流，每个 channel 对应一个 buffer缓冲区，buffer 底层就是个数组
- channel 会注册到 selector 上，由 selector 根据 channel 读写事件的发生将其交由某个空闲的线程处理
- NIO 的 Buffer 和 channel 都是既可以读也可以写

对应通讯示意图如下图所示：
![JDK-NIO通信模型](65d26b16/JDK-NIO通信模型.jpg)
NIO底层在JDK1.4版本是用linux的内核函数select()或poll()来实现，selector每次都会轮询所有的sockchannel看下哪个channel有读写事件，有的话就处理，没有就继续遍历。
**JDK1.5开始引入了epoll基于事件响应机制来优化NIO。**

### 底层流程细节
NioSelectorServer代码里如下几个方法非常重要：
```java
Selector.open() //创建多路复用器
socketChannel.register() //将channel注册到多路复用器上
selector.select() //阻塞等待需要处理的事件发生
```
从Hotspot与Linux内核函数级别来理解下，大体对应主要工作流程如下：
![底层流程](65d26b16/底层流程.jpg)


**总结：**NIO整个调用流程就是Java调用了操作系统的内核函数来创建Socket，获取到Socket的文件描述符，再创建一个Selector对象，对应操作系统的Epoll描述符。
将获取到的Socket连接的文件描述符的事件绑定到Selector对应的Epoll文件描述符上，进行事件的异步通知，这样就实现了使用一条线程，并且不需要太多的无效的遍历，将事件处理交给了操作系统内核(操作系统中断程序实现)，大大提高了效率。


## Linux下的epoll函数
I/O多路复用底层主要用的Linux 内核函数（select，poll，epoll）来实现，windows不支持epoll实现，windows底层是基于winsock2的select函数实现的(不开源)
![linux-pool-epoll比较](65d26b16/linux-pool-epoll.png)


## Netty使用
### 一个聊天室Demo
#### 服务端
```java
public static void main(String[] args) throws Exception {

        //创建两个线程组bossGroup和workerGroup, 含有的子线程NioEventLoop的个数默认为cpu核数的两倍
        // bossGroup只是处理连接请求 ,真正的和客户端业务处理，会交给workerGroup完成
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            //创建服务器端的启动对象
            ServerBootstrap bootstrap = new ServerBootstrap();
            //使用链式编程来配置参数
            bootstrap.group(bossGroup, workerGroup) //设置两个线程组
                    .channel(NioServerSocketChannel.class) //使用NioServerSocketChannel作为服务器的通道实现
                    // 初始化服务器连接队列大小，服务端处理客户端连接请求是顺序处理的,所以同一时间只能处理一个客户端连接。
                    // 多个客户端同时来的时候,服务端将不能处理的客户端连接请求放在队列中等待处理
                    .option(ChannelOption.SO_BACKLOG, 1024)
                    .childHandler(new ChannelInitializer<SocketChannel>() {//创建通道初始化对象，设置初始化参数

                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ChannelPipeline pipeline = ch.pipeline();
                            pipeline.addLast("decoder", new StringDecoder());
                            pipeline.addLast("encoder", new StringEncoder());
                            //对workerGroup的SocketChannel设置处理器，注意这个加入的与编解码器的顺序
                            pipeline.addLast(new ChatServerHandler());
                        }
                    });
            System.out.println("chat server start。。");
            //绑定一个端口并且同步, 生成了一个ChannelFuture异步对象，通过isDone()等方法可以判断异步事件的执行情况
            //启动服务器(并绑定端口)，bind是异步操作，sync方法是等待异步操作执行完毕
            ChannelFuture cf = bootstrap.bind(9000).sync();
            cf.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
```
```java
public class ChatServerHandler extends SimpleChannelInboundHandler<String> {
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        System.out.println("caught exception: " + cause.getMessage());
    }

    private static ChannelGroup channelGroup = new DefaultChannelGroup(GlobalEventExecutor.INSTANCE);
    private static SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        Channel channel = ctx.channel();
        channelGroup.writeAndFlush("[client] " + channel.remoteAddress() + " is connected! " + sdf.format(new Date()) + "\r\n");
        channelGroup.add(channel);
        System.out.println("server log: [client]" + channel.remoteAddress() + " is connected! \r\n");
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        Channel channel = ctx.channel();
        channelGroup.writeAndFlush("[client] " + channel.remoteAddress() + " is disconnected!\r\n");
        System.out.println("server log: [client]" + channel.remoteAddress() + " is disconnected! \r\n");
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) throws Exception {
        Channel channel = ctx.channel();
        channelGroup.forEach(ch -> {
            if (ch != channel) {
                ch.writeAndFlush("[client] " + channel.remoteAddress() + " send msg: " + msg + "\r\n");
            } else {
                ch.writeAndFlush("[myself] send msg: " + msg + "\r\n");
            }
        });
    }

}
```

#### 客户端
```java
public class ChatClient {
    public static void main(String[] args) throws Exception {
        //客户端需要一个事件循环组
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            //创建客户端启动对象
            //注意客户端使用的不是 ServerBootstrap 而是 Bootstrap
            Bootstrap bootstrap = new Bootstrap();
            //设置相关参数
            bootstrap.group(group) //设置线程组
                    .channel(NioSocketChannel.class) // 使用 NioSocketChannel 作为客户端的通道实现
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel channel) throws Exception {
                            ChannelPipeline pipeline = channel.pipeline();
                            pipeline.addLast(new StringEncoder());
                            pipeline.addLast(new StringDecoder());
                            //加入处理器
                            pipeline.addLast(new ChatClientHandler());
                        }
                    });
            System.out.println("client client start");
            //启动客户端去连接服务器端
            ChannelFuture channelFuture = bootstrap.connect("127.0.0.1", 9000).sync();
            Channel channel = channelFuture.channel();
            System.out.println("client connect success! " + channel.localAddress());

            Scanner scanner = new Scanner(System.in);
            while (scanner.hasNextLine()) {
                String line = scanner.nextLine();
                channel.writeAndFlush(line);
            }
            //对关闭通道进行监听
//            channel.closeFuture().sync();
        } finally {
            group.shutdownGracefully();
        }
    }
}
```

```java
public class ChatClientHandler extends SimpleChannelInboundHandler<String> {

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        System.out.println("caught exception...." + cause.getMessage());
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) throws Exception {
        System.out.println(msg.trim());
    }
}
```

### Netty线程模型
这里穿插一下关于响应式编程中（Doug Lea）提到的关于主从Reactor模型的含义：
**其实完全可以看作一种多级缓存的概念，增加了一道缓冲，以免造成单一线程池压力【对比上文中的nio程序】**
![multi_reactors](65d26b16/mutil_reactors.png)

看上面的例子中的服务端实现，new了两个netty的```NioEventLoopGroup```，就是这种思想。
**当然，延申出来可以做成“一主多从”，即多个从Reactor模型**

概况起来，netty的工作架构示意图：
![netty-reactor工作架构图](65d26b16/netty-reactor工作架构图.png)

### Netty核心功能
#### 编解码机制
当你通过Netty发送或者接受一个消息的时候，就将会发生一次数据转换：**入站消息会被解码：从字节转换为另一种格式（比如java对象）；如果是出站消息，它会被编码成字节。**
Netty提供了一系列实用的编码解码器，他们都实现了**ChannelInboundHadnler**或者**ChannelOutboundHandler**接口。在这些类中，channelRead方法已经被重写了。以入站为例，对于每个从入站Channel读取的消息，这个方法会被调用。随后，它将调用由已知解码器所提供的decode()方法进行解码，并将已经解码的字节转发给ChannelPipeline中的下一个ChannelInboundHandler。
Netty提供了很多编解码器，比如编解码字符串的StringEncoder和StringDecoder，编解码对象的ObjectEncoder和ObjectDecoder等。
**注：**不同方向的编解码器是有规律注册（继承了Outbound/Inboundhandler）好的，出站编码，入站解码。
下方图为Channle工作示意图：
![netty-channel](65d26b16/netty-channel.png)
##### ChannelHandler
ChannelHandler充当了处理入站和出站数据的应用程序逻辑容器。
例如，实现ChannelInboundHandler接口（或ChannelInboundHandlerAdapter），你就可以接收入站事件和数据，这些数据随后会被你的应用程序的业务逻辑处理。
当你要给连接的客户端发送响应时，也可以从ChannelInboundHandler冲刷数据。你的业务逻辑通常写在一个或者多个ChannelInboundHandler中。ChannelOutboundHandler原理一样，只不过它是用来处理出站数据的。
##### ChannelPipeline
ChannelPipeline提供了ChannelHandler链的容器。
以客户端应用程序为例，如果事件的运动方向是从服务端到客户端的，那么我们称**这些事件为出站的**，即客户端发送给服务端的数据会通过pipeline中的一系列ChannelOutboundHandler(ChannelOutboundHandler调用是从tail到head方向逐个调用每个handler的逻辑)，并被这些Handler处理；
**反之则称为入站的**，入站只调用pipeline里的ChannelInboundHandler逻辑(ChannelInboundHandler调用是从head到tail方向逐个调用每个handler的逻辑)。

#### 粘包拆包方案
TCP是一个流协议，就是没有界限的一长串二进制数据。
TCP作为传输层协议并不不了解上层业务数据的具体含义，它会根据TCP缓冲区的实际情况进行数据包的划分，所以在业务上认为是一个完整的包，可能会被TCP拆分成多个包进行发送，也有可能把多个小的包封装成一个大的数据包发送，这就是所谓的**TCP粘包和拆包**问题。**面向流的通信是无消息保护边界的。**
如下图所示，client发了两个数据包D1和D2，但是server端可能会收到如下几种情况的数据：
![TCP粘包-拆包示意](65d26b16/TCP粘包-拆包示意.png)
常用的解决方案：
1) **消息定长度**，传输的数据大小固定长度，例如每段的长度固定为100字节，如果不够空位补空格
2) 在**数据包尾部添加特殊分隔符**，比如下划线，中划线等，这种方法简单易行，但选择分隔符的时候一定要注意每条数据的内部一定不能出现分隔符。
3) **发送长度**：发送每条数据的时候，将数据的长度一并发送，比如可以选择每条数据的前4位是数据的长度，应用层处理时可以根据长度来判断每条数据的开始和结束。
在netty中提供了下面三种解码器来处理：
- LineBasedFrameDecoder （回车换行分包）
- DelimiterBasedFrameDecoder （特殊分隔符分包）
- FixedLengthFrameDecoder （固定长度报文来分包）
以DelimiterBasedFrameDecoder为例，在IDEA中查看类继承图谱：
![DelimiterBasedFrameDecoder继承图谱](65d26b16/DelimiterBasedFrameDecoder继承图谱.png)
**继承了来自一个抽象类：ByteToMessageDecoder**，归纳有两组抽象类：
- ByteToMessageDecoder/MessageToByteEncoder
- MessageToMessageDecoder/MessageToMessageEncoder
这两组抽象类，抽象了关于解码器的众多细节，由这两组抽象类衍生的，netty实现了非常多的解码器组件，如下图所示（4.1.35版本，后续版本的netty将code包单独独立出pom依赖分支，不再柔和在一个包里面了，注意一下）：
![netty-codec-package](65d26b16/netty-codec-package.png)

#### 心跳检测机制
IdleStateHandler
#### 断线重连机制

### Netty核心源码
