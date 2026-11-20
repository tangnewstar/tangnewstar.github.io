# 核心概念

## **🧠 二、Netty 的设计理念与关键抽象**

Netty 的核心设计理念是 **异步、非阻塞、事件驱动**。它封装了底层的 NIO，提供了更高级、更易用的 API。

### **📌 1. 核心类概览：**

| **类名** | **作用描述** |
| --- | --- |
| `NioEventLoopGroup` | 管理多个 `EventLoop` 的线程池，用于处理 I/O 事件 |
| `NioEventLoop` | 单个线程，处理一个 Channel 的 I/O 事件（包括读、写、连接事件等） |
| `Channel` | 表示网络通道（Socket 的封装） |
| `ChannelHandlerContext` | 每个 Channel 上的事件处理器上下文，用于处理 Channel 的事件 |
| `ChannelHandler` | 用于处理 I/O 事件的接口（如 `ChannelInboundHandler`、`ChannelOutboundHandler`） |
| `ChannelInitializer` | 在 Channel 初始化时被调用，用于添加自定义的 `ChannelHandler` |
| `ChannelPipeline` | 将 `ChannelHandler` 组织成一个链，处理数据和事件 |
| `ByteBuf` | Netty 的缓冲区，代替 Java 的 `ByteBuffer`，支持多种内存分配方式 |
| `ChannelInboundHandlerAdapter` | 提供默认实现的 `ChannelInboundHandler`，用于处理接收事件（如数据读取） |
| `ChannelOutboundHandlerAdapter` | 提供默认实现的 `ChannelOutboundHandler`，用于处理发送事件（如数据写入） |
| `ChannelFuture` | 用于监听 Channel 的操作结果（如连接、绑定等） |
| `ChannelHandler` | 通用通道处理器接口，包含接收和发送方向的处理方法 |

### **📌 2. 关键抽象与对象关系**

- **`Channel`** 是网络通信的“门”，每个网络连接都会对应一个 `Channel`。
    - 它是 Netty 的核心接口，代表一个网络端点。
- **`ChannelPipeline`** 是 `Channel` 的事件处理链，负责将 I/O 事件分发给不同的 `ChannelHandler`。
    - 是 Netty 的事件驱动机制的核心。
- **`ChannelHandler`** 是处理 I/O 事件的组件，如 `ChannelInboundHandler` 处理读事件，`ChannelOutboundHandler` 处理写事件。
    - `ChannelInboundHandlerAdapter` 是一个常用的实现类，用于处理接收事件。
- **`NioEventLoopGroup`** 用于管理多个 `NioEventLoop`，这些 `NioEventLoop` 分别负责处理一个 `Channel` 的 I/O 事件。
    - `NioEventLoop` 是一个线程，为一个 Channel 提供专属的 I/O 事件处理能力。
- **`ByteBuf`** 是 Netty 的核心缓冲区类，比 Java 的 `ByteBuffer` 更强大，支持多个内存分配方式（堆、直接、池等）。
    - 提供了读写操作的完整封装。

---

## **🧠 三、Netty 的关键机制结合代码片段（示例）**

```
<JAVA>

// 创建 EventLoopGroup
EventLoopGroup group = new NioEventLoopGroup();

// 创建 ServerSocketChannel（与 TCP 操作有关）
ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.group(group)
        .channel(NioServerSocketChannel.class)
        .childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            protected void initChannel(SocketChannel ch) {
                ch.pipeline().addLast(new StringDecoder());
                ch.pipeline().addLast(new StringEncoder());
                ch.pipeline().addLast(new MyChannelHandler());
            }
        })
        .option(ChannelOption.SO_BACKLOG, 1024);

// 启动服务器
Channel channel = bootstrap.bind(8080).sync().channel();

```

### **解释：**

- `NioEventLoopGroup`：管理多个 `NioEventLoop`，它们各自负责一个 Channel 的 I/O 事件。
- `ServerBootstrap`：封装了 Channel 的创建逻辑，用于启动和绑定 Channel。
- `ChannelInitializer`：用于初始化 ChannelPipeline，添加如 `StringDecoder`、`StringEncoder`、`MyChannelHandler` 等处理类。
- `ChannelPipeline`：是处理 I/O 事件的链式结构，Netty 的内容和事件处理都基于它。
- `StringDecoder` 和 `StringEncoder`：将 ByteBuf 转换为 String，并支持自动处理拆包和粘包。
- `MyChannelHandler`：自定义的处理器，继承 `ChannelInboundHandlerAdapter`，处理接收事件。