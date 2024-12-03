---
title: WebSocket握手认证实践
date: 2024-12-03 11:06:49
tags:
  - Netty
  - WebSocket
  - Java
catagories: 技术
cover: https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/2024-12-3post_img.png
---

{% note info flat %}

注意！本实践基于 Netty 网络通讯工具。

{% endnote %}

WebSocket 是一种全双工通信协议，它允许客户端和服务器之间的持续连接。在建立 WebSocket 连接之前，必须通过 HTTP 完成一次握手，称为 **WebSocket 握手**。在握手过程中，通常可以实现 **身份认证** 和 **权限验证**，以确保只有合法用户能够建立 WebSocket 连接。  



# 实战背景

现在设想这样一个场景，用户登录使用在线聊天软件时，通常使用 WebSocket 连接建立通道实现”客户端-服务器“双向通信，而通道在登录后会标识其为哪个用户。

![img](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/1733145277024-164ac567-f5e1-4c29-ba71-8a9f2cbe9136.png)

但用户可能刷新前端，websocket 连接就重建了，这时难道用户需要再次登录么，很麻烦对吧。所以需要携带一个登录凭证，在后续的操作中，根据登录凭证（Token）就能重建连接并进行身份验证。我们需要知道的是每个 channel 对应的用户是谁。



# 传统方案

![](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/1733189844847-e87e2d73-7571-4f2a-816d-1dd2df0b94b5.png)

通常情况下，首先想到的可能是会把登录认证和建立 websocket 连接分开来，但这样一来一回需要三次请求，可能会提高前端反馈延迟。而第一步和第二步能否合并在一起呢，在 websocket 的建立阶段进行认证。



# 方案调研

要想实现上述的握手认证，首先要明白，websocket 协议其实是从 http 协议升级而来，首先会发送一个请求升级的 http 协议，那我们可以在这个阶段，通过 http 携带的一些参数进行 token 认证。

前端发送 websocket 请求的时候有哪些机会可以携带参数呢？

![](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/1733190265457-1165619b-8117-4fbe-a5b0-6480e045f8cd.png)

1. 如果是常规 http 请求，可以在请求头中自定义属性，但原生的 websocket 在请求时好像无法添加请求头。
2. 将 token 拼接在 url 后作为参数进行传递。
3. 可以看到上述建立 websocket 连接时，可以传递第二个参数 `protocols`协议的意思。我们不妨尝试下，反正有个传参的地方。



# 方案实现

## protocols 传参

看到建立 websocket 连接传入参数，首先想到的就是在 Netty 握手阶段进行处理，那么我们就去看握手处理器的源码。

通过查看源码和查询资料发现，Netty 握手认证在 `io.netty.handler.codec.http.websocketx.WebSocketServerProtocolHandshakeHandler` 中实现。

![](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/1733191344879-d2192b5e-ae1c-436a-9a8f-4bf6481acf3c.png)

那这个 `protocols` 在服务端哪里设置呢，可以看到它是在构建握手处理器的地方填写的：

![](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/1733191634776-e016dd83-f77c-4b50-9f64-f850166fa46d.png)

那么就有个疑问了，Netty 既没开放 `protocols` 的处理，服务端对比参数时又只能写死。再联想 `protocols`本身就是协议的意思，应该是双方规定好的一些东西，我们拿来传 token，那这条路可能就错了。

_当然也不是没有办法，我们只需要将 _`_WebSocketServerProtocolHandshakeHandler_`_ 的握手过程全部重写一边即可，不过太麻烦了对吧，一看就不符合 Netty 的设计理念。



## url 传参

上述方案行不通，就试一下第二种方案，url 传参。相比于上一种方法 url 传参就简单很多了，只需要在 websocket 建立连接之前进行处理拿到 token 即可。

首先通过 `pipeline` 在ws握手处理器之前添加一个自定的处理器：

![](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/1733194681207-bcb397f1-de97-4b85-9b70-6ab05ca202e4.png)

该处理器只能需要处理接收到的数据即可，具体代码如下：

```java
/**
 * http请求头处理器
 *
 * @author Ershi
 * @date 2024/12/01
 */
public class HttpHeaderHandler extends ChannelInboundHandlerAdapter {

    /**
     * 读取通道中的数据，并处理HTTP请求 <br>
     *
     * @param ctx 通道处理上下文
     * @param msg 接收到的消息对象
     * @throws Exception 如果处理过程中发生异常
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        // 判断接收到的消息是否为HttpRequest实例
        if (msg instanceof FullHttpRequest) {
            // 对HttpRequest实例进行处理
            FullHttpRequest request = (FullHttpRequest) msg;
            // 解析请求URI并构建UrlBuilder对象，用于后续获取查询参数
            UrlBuilder urlBuilder = UrlBuilder.ofHttp(request.uri());

            // 从查询参数中获取token，如果不存在，则默认为空字符串
            String token = Optional.ofNullable(urlBuilder.getQuery()).map(k -> k.get("token")).map(CharSequence::toString).orElse("");
            // 将获取到的token作为属性存储在通道中，以供后续使用
            NettyUtil.setAttr(ctx.channel(), NettyUtil.TOKEN, token);
            // 移除token参数，为了路径后续能够匹配到websocket升级处理器
            request.setUri(urlBuilder.getPath().toString());
        }
        ctx.fireChannelRead(msg);
    }
}
```

需要注意的是，在验证完 token 后需要把参数移除，不然后面的请求路径无法匹配 ws 握手处理器的路径匹配规则 `"/"`。

`pipeline.addLast(new WebSocketServerProtocolHandler("/"));`

_ps：http url 有长度限制，不过一般 token 不会超过这个限制。_



# 总结

通过不断探索，我们深入理解了 websocket 的协议升级原理，以及方案中牵扯到 Netty websocket 协议升级的源码，但后来发现这条路过于麻烦不符合设计理念，又转而选择另一种方法。

通过 url 携带参数的方式获取了 token 进行验证。



