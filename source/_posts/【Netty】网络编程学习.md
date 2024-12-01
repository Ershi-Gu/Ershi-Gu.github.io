---
title: 【Netty】网络编程学习
date: 2024-11-23 10:26:42
tags:
  - Netty
  - NIO
  - Java
catagories: 学习笔记
highlight_shrink: true
---

{% note info flat %}

该文章仅用作个人学习笔记！

参考文章：

1. [柏码知识库 | NIO 笔记（二）Netty框架专题](https://www.itbaima.cn/document/ndz9t0uunrmfmv4n?segment=2#doc2-EventLoop和任务调度)
2. [Java IO 模型详解 | JavaGuide](https://javaguide.cn/java/io/io-model.html#bio-blocking-i-o)
3. [Netty | 工作流程图分析 & 核心组件说明 & 代码案例实践-阿里云开发者社区](https://developer.aliyun.com/article/1412358)

{% endnote %}



# 选择器与 IO 多路复用

常见的 IO 多路复用模型，采用选择器的方案实现。当服务器与客户端存在某一状态（连接请求、读、写）时，才会进行处理，相比于传统的 IO（接收完客户端请求后，会阻塞等待读操作）

![image-20241123104745517](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241123104745517.png)

当有多个连接到服务器时，有下面的方式进行处理：

- **select**：当这些连接出现具体的某个状态时，只是知道已经就绪了，但是不知道详具体是哪一个连接已经就绪，每次调用都进行线性遍历所有连接，时间复杂度为 `O(n)`，并且存在最大连接数限制。

- **poll**：同上，但是由于底层采用链表，所以没有最大连接数限制。

- **epoll**：采用事件通知方式，当某个连接就绪，能够直接进行精准通知（这是因为在内核实现中epoll是根据每个fd上面的callback函数实现的，只要就绪会会直接回调callback函数，实现精准通知，但是只有Linux支持这种方式），时间复杂度 `O(1)`，Java在Linux环境下正是采用的这种模式进行实现的。

再 Java 中网络通信具体实现多路复用的代码如下：

```java
public static void main(String[] args) {
    try (ServerSocketChannel serverChannel = ServerSocketChannel.open();
         Selector selector = Selector.open()){   //开启一个新的Selector，这玩意也是要关闭释放资源的
        serverChannel.bind(new InetSocketAddress(8080));
        //要使用选择器进行操作，必须使用非阻塞的方式，这样才不会像阻塞IO那样卡在accept()，而是直接通过，让选择器去进行下一步操作
        serverChannel.configureBlocking(false);
        //将选择器注册到ServerSocketChannel中，后面是选择需要监听的时间，只有发生对应事件时才会进行选择，多个事件用 | 连接，注意，并不是所有的Channel都支持以下全部四个事件，可能只支持部分
        //因为是ServerSocketChannel这里我们就监听accept就可以了，等待客户端连接
        //SelectionKey.OP_CONNECT --- 连接就绪事件，表示客户端与服务器的连接已经建立成功
        //SelectionKey.OP_ACCEPT --- 接收连接事件，表示服务器监听到了客户连接，服务器可以接收这个连接了
        //SelectionKey.OP_READ --- 读 就绪事件，表示通道中已经有了可读的数据，可以执行读操作了
        //SelectionKey.OP_WRITE --- 写 就绪事件，表示已经可以向通道写数据了（这玩意比较特殊，一般情况下因为都是可以写入的，所以可能会无限循环）
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);
        while (true) {   //无限循环等待新的用户网络操作
            //每次选择都可能会选出多个已经就绪的网络操作，没有操作时会暂时阻塞
            int count = selector.select();
            System.out.println("监听到 "+count+" 个事件");
            Set<SelectionKey> selectionKeys = selector.selectedKeys();
            Iterator<SelectionKey> iterator = selectionKeys.iterator();
            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                //根据不同的事件类型，执行不同的操作即可
                if(key.isAcceptable()) {  //如果当前ServerSocketChannel已经做好准备处理Accept
                    SocketChannel channel = serverChannel.accept();
                    System.out.println("客户端已连接，IP地址为："+channel.getRemoteAddress());
                    //现在连接就建立好了，接着我们需要将连接也注册选择器，比如我们需要当这个连接有内容可读时就进行处理
                    channel.configureBlocking(false);
                    channel.register(selector, SelectionKey.OP_READ);
                    //这样就在连接建立时完成了注册
                } else if(key.isReadable()) {    //如果当前连接有可读的数据并且可以写，那么就开始处理
                    SocketChannel channel = (SocketChannel) key.channel();
                    ByteBuffer buffer = ByteBuffer.allocate(128);
                    channel.read(buffer);
                    buffer.flip();
                    System.out.println("接收到客户端数据："+new String(buffer.array(), 0, buffer.remaining()));

                    //直接向通道中写入数据就行
                    channel.write(ByteBuffer.wrap("已收到！".getBytes()));
                    //别关，说不定用户还要继续通信呢
                }
                //处理完成后，一定记得移出迭代器，不然下次还有
                iterator.remove();
            }
        }
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```



# NIO 框架存在的问题

## 客户端关闭导致服务端空轮询

![image-20241123105736800](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241123105736800.png)

当客户端主动断开与服务端的连接时，服务端会进入一个莫名其妙的 READ 事件，直到 Java 抛出异常。这意味着，在客户端断开连接时，select 会直接允许其通过，从而触发后面的操作。

**原因：**

- 在 TCP 协议中，当客户端关闭连接时，会发送一个 **FIN 包**，通知服务端不再发送数据，但仍可以接收服务端发送的数据。
- 服务端的 `SocketChannel` 会检测到这个 **FIN 包**，从而触发 `SelectionKey.OP_READ` 事件。此时，调用 `SocketChannel.read(ByteBuffer)` 会返回 `-1`，表示客户端已关闭连接。

> `SocketChannel.read(ByteBuffer)` 返回读取到的字节数

这下明白了，原来是因为客户端返回了一个 -1，我们只需要进行额外的判断即可：

```java
if(channel.read(buffer) < 0) {
    System.out.println("客户端已经断开连接了："+channel.getRemoteAddress());
    channel.close();   //直接关闭此通道
    continue;   //继续进行选择
}
```



## 框架本身问题

除上述客户端主动断开发送 FIN 包导致的问题外，Java NIO 框架本身还有问题导致空轮询。

即使没有任何通道准备好 I/O 操作，`Selector.select()` 方法仍然会返回，并陷入无意义的循环，从而导致 CPU 占用率异常升高

JDK 官方认为这是操作系统的 BUG：

- 在 Linux 2.6.32 及更早的版本中，`epoll` 可能会触发 "wake-up FD" 的问题。
- 特定条件下，`epoll` 的内部数据结构会被错误更新，导致 `select()` 方法误以为有事件发生。



## TCP 半包和粘包问题

TCP 时面向流的，数据之间没有界限，且在发送数据前会将数据存放在缓冲区，具体什么时候发送由其自己控制。



### 半包

如果 TCP 一次传输的数据大小超过发送缓冲区的大小，那么一个完整的报文就被拆分了，可能会导致接收端收到不完整的数据。



### 粘包

如果 TCP 一次传输的数据大小小于发送缓冲区，那么可能回合别的报文合并起来一起发送，造成粘包



# Netty 框架

**Netty** 是一个高性能、异步的事件驱动网络框架，基于 Java NIO（New IO）实现，用于构建高并发、低延迟的网络应用。它封装了复杂的底层 I/O 操作（如 Selector、多线程模型、协议解析），并提供了简单易用的 API，适合开发各种通信协议和应用。



## ByteBuf 介绍

相比于原生的 NIO，Netty 使用 ByteBuf 作为缓冲区进行数据装载。

相比于原生 ByteBuffer，其不同之处在于：

- 写操作后无需进行 `flip()` 翻转。
- 具有更快的响应速度
- 动态扩容

![image-20241123161248584](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241123161248584.png)

在内部其同时维护 **读指针** 和 **写指针**，这样就不需要 `flip()` 翻转了。



### 基本使用

```java
public static void main(String[] args) {
    //创建一个初始容量为10的ByteBuf缓冲区，这里的Unpooled是用于快速生成ByteBuf的工具类
    //至于为啥叫Unpooled是池化的意思，ByteBuf有池化和非池化两种，区别在于对内存的复用，我们之后再讨论
    ByteBuf buf = Unpooled.buffer(10);
    System.out.println("初始状态："+Arrays.toString(buf.array()));
    buf.writeInt(-888888888);   //写入一个Int数据
    System.out.println("写入Int后："+Arrays.toString(buf.array()));
    buf.readShort();   //无需翻转，直接读取一个short数据出来
    System.out.println("读取Short后："+Arrays.toString(buf.array()));
    buf.discardReadBytes();   //丢弃操作，会将当前的可读部分内容丢到最前面，并且读写指针向前移动丢弃的距离
    System.out.println("丢弃之后："+Arrays.toString(buf.array()));
    buf.clear();    //清空操作，清空之后读写指针都归零
    System.out.println("清空之后："+Arrays.toString(buf.array()));
}
```

在进行一些涉及删除的操作时，其内部其实并没有修改具体的字节数据，而只是修改了操作指针的合法范围：

![image-20241123161714051](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241123161714051.png)



### 自动扩容

当我们写入一个超过容量的数据时，会进行自动扩容，第一次扩容会从 64 开始，之后每次扩容都会 x2，如果不希望其自动扩容，可以设置最大容量。

```java
public static void main(String[] args) {
    //在生成时指定maxCapacity也为10
    ByteBuf buf = Unpooled.buffer(10, 10);
    System.out.println(buf.capacity());
    buf.writeCharSequence("卢本伟牛逼！", StandardCharsets.UTF_8);
    System.out.println(buf.capacity());
}
```



### 缓冲区的三种实现

实现模式：**堆缓冲区模式**、**直接缓冲区模式**、**复合缓冲区模式。**

前两者和 ByteBufer 一样，前者基于数组，后者基于直接内存，我们直接看第三个，**复合模式**，复合模式可以任意地拼凑组合其他缓冲区。

```java
//创建一个复合缓冲区
CompositeByteBuf buf = Unpooled.compositeBuffer();
buf.addComponent(Unpooled.copiedBuffer("abc".getBytes()));
buf.addComponent(Unpooled.copiedBuffer("def".getBytes()));

for (int i = 0; i < buf.capacity(); i++) {
    System.out.println((char) buf.getByte(i));
}
```

**其本质是缓冲区的组合试图，并没有拷贝组合原先的缓冲区，而是将原先的两个缓冲区映射到一起操作。**

![image-20241123162925247](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241123162925247.png)



### 池化和非池化

`Unpooled` 在内部通过 `ByteBufAllocator` 创建缓冲区，而 `ByteBufAllocator` 具备两个实现，`UnpooledByteBufAllocator` 和 `PooledByteBufAllocator`，一个是非池化缓冲区生成器，还有一个是池化缓冲区生成器。

![image-20241123163306661](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241123163306661.png)

实际上池化缓冲区利用了池化思想，将缓冲区通过设置内存池来进行内存块复用，这样就不用频繁地进行内存的申请，尤其是在使用堆外内存的时候，避免多次重复通过底层 `malloc()` 函数系统调用申请内存造成的性能损失。

比如下面的代码，两次创建的缓冲区是同一块内存。

```java
public static void main(String[] args) {
    ByteBufAllocator allocator = PooledByteBufAllocator.DEFAULT;
    ByteBuf buf = allocator.directBuffer(10);   //申请一个容量为10的直接缓冲区
    buf.writeChar('T');    //随便操作操作
    System.out.println(buf.readChar());
    buf.release();    //释放此缓冲区

    ByteBuf buf2 = allocator.directBuffer(10);   //重新再申请一个同样大小的直接缓冲区
    System.out.println(buf2 == buf);
}
```

![image-20241123163537731](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241123163537731.png)



## 零拷贝

零拷贝是一种 IO 优化操作，简单说就是避免在用户态和内核态之间拷贝数据的技术，从而减少 CPU 占用和内存带宽的消耗。

我们的应用程序实际上是运行在用户空间的，内核空间运行着系统层面的东西，比如我们在 Java 中创建一个新的线程，实际上最终还是要交给操作系统来为我们进行分配的。

IO 操作也是如此，需要操作系统帮我们从磁盘上读取文件数据或是向网络中发送数据，如下流程：

![image-20241123213121173](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241123213121173.png)

这就无可避免的要在内核空间和用户空间进行数据的拷贝，消耗资源。

而实现零拷贝有以下方案：

1. **使用虚拟内存**

   将内核空间和用户空间的虚拟地址都指向同一个物理地址，就像于公用一块区域，也谈不上拷贝了：

   ![image-20241123213303898](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241123213303898.png)

2. **ByteBuf 直接缓冲区**

   ByteBuf 提供了直接缓冲区和堆缓冲区两种类型。

   直接缓冲区可以使用本地内存，正常情况下 JVM 需要数据从堆外拷贝到堆内才能访问，而直接缓冲区可以避免这个操作，减少数据拷贝。

3. **内存映射文件**

   将内核空间的缓存映射到用户空间，这样就不需要从内核空间读取数据到用户空间了，数据处理完毕后，直接在内核空间将数据发送给缓冲区。

![image-20241123214257381](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241123214257381.png)

4. **FileRegion 接口**

   `FileRegion` 是 Netty 提供的用于文件传输的接口，通过调用操作系统的 `sendfile` 函数实现文件零拷贝传输。`sendfile` 函数可以**将文件数据直接从文件系统发送到网络接口，而无需经过用户态内存拷贝**。

   ![image-20241123215145549](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241123215145549.png)

   5. **CompositeByteBuf**

      Netty 中的零拷贝其实还有另一层含义，即避免了 buffer 之间的拷贝。`CompositeByteBuf` 可以将两个 Buffer 逻辑上组成在一起，从而避免数据拷贝。

      ```java
      CompositeByteBuf buf = Unpooled.compositeBuffer();
      buf.addComponent(Unpooled.copiedBuffer("abc".getBytes()));
      buf.addComponent(Unpooled.copiedBuffer("def".getBytes()));
      ```

      

*作者有点懒，笔记持续更新......*
