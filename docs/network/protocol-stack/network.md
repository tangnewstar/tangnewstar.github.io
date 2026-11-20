# Network

网络：不同设备之间的数据交换

为了标准化网络通信，ISO提出七层模型

- 物理层：网线光纤，处理光电信号
- 数据链路层：物理层的光电信号封装成“帧”（加入MAC地址）负责局域网内的设备识别（以太网）
- 网络层：实现跨局域网的路由，核心是IP协议（32位IP地址定位设备）
- 传输层：负责端到端的可靠传输（确保数据不丢失，按序到达），核心协议是TCP和UDP
- 会话层：管理会话（建立/断开连接，基本被应用层合并）
- 表示层：数据格式处理（加密，压缩，序列化）
- 应用层：为应用程序提供服务，HTTP

TCP/IP四层模型（实际应用）

- 网络接口层：处理硬件和物理通信，对应OSI的物理层和数据链路层
- 网络层：IP协议 ICMP（ping命令）ARP（IP转MAC）
- 传输层：TCP UDP
- 应用层：HTTP FTP DNS RPC

数据传输时，发送方由上至下逐层加首部，接收方由下至上逐层解首部

应用层 → 加TCP首部（传输层）→ IP首部（网络层）→ MAC首部（链路层）→ 物理层

### 传输层 TCP UDP

负责端到端的数据传输

例如HTTP的80端口，HTTPS的443端口

**UDP User Data Protocol**

发送方将数据打包成数据报文（加端口号）发送给接收方，不确认对方是否收到也不保证顺序

- 特点：无连接，不可靠，速度快
- 适用：实时性优先，允许少量丢失，如视频通话、游戏联机、DNS查询

```java
// 发送方
DatagramSocket socket = new DatagramSocket();
byte[] data = "hello".getBytes();
DatagramPacket packet = new DatagramPacket(data, data.length, InetAddress.getByName("127.0.0.1"), 8888);
socket.send(packet);

// 接收方
DatagramSocket socket = new DatagramSocket(8888);
byte[] buf = new byte[1024];
DatagramPacket packet = new DatagramPacket(buf, buf.length);
socket.receive(packet); // 阻塞等待
System.out.println(new String(packet.getData(), 0, packet.getLength()));
```

**TCP Transmission Control Protocol**

发送前需建立连接，传输中通过“确认、重传、排序” 保证数据不丢、不重、按序到达，最后断开连接

- 特点：面向连接、可靠、字节流
- 核心机制：
    - 3次握手（建立连接）：确保双方收发能力正常（A→B：能收到吗？；B→A：能，你呢？A→B：我也能）
    - 4次挥手（断开连接）：确保双方传输完毕
    - 滑动窗口：控制发送速率，避免接收方处理不过来（流量控制）
    - 拥塞控制：避免网络拥堵（慢开始，拥塞避免）
    - 重传机制：超时未收到确认则重发
- 适用场景：可靠的场景（文件传输，HTTP通信，数据库连接）

```java
        // 服务端
        ServerSocket serverSocket = new ServerSocket(8888);
        Socket socket = serverSocket.accept(); //阻塞等待
        InputStream in = socket.getInputStream();
        byte[] buf = new byte[1024];
        int len = in.read(buf);
        System.out.println(new String(buf, 0, len));
        
        // 客户端
        Socket socket = new Socket("127.0.0.1", 8888);
        OutputStream out = socket.getOutputStream();
        out.write("world".getBytes());
```

Socket 

应用层和传输层之间的接口，封装了TCP/UDP的复杂细节，让应用程序能够简单的发送和接收数据

- 本质：Socket = IP + Port（`192.168.0.1:8000`）用于唯一标识网络中的应用
- TCP Socket：基于TCP的Socket，双向字节流（双方可同时读写，类似管道）
- UDP Socket：基于UDP的Socket，是 数据报文 的发送接收接口（`DatagramSocket`）

Socket是传输层协议的具象化工具，应用层通过Socket调用TCP/UDP实现网络通信

应用层：HTTP and RPC

应用层协议定义了：应用程序之间交换数据的格式和规则

HTTP：基于TCP的应用层协议，最初为传输HTML设计，现在是互联网通用

- 特点：
    - 基于 “请求-响应” 模型
    - 无状态，每次请求独立，服务端不保存客户端状态（通过Session/Cookie维护）
    - **文本协议**：头信息是文本，body可以是文本/二进制
- 版本：
    - 1.1：支持长连接（一个TCP连接复用多次）管道化（一次多个请求）队头阻塞问题
    - 2：使用二级制帧和多路复用（一个TCP连接处理多个请求，帧带标识，避免队头阻塞）
    - 3：基于UDP的QUIC协议（解决TCP对头阻塞）更快的连接建立（合并TCP，TLS）
- Java中的HTTP：
    - 原生：HttpURLConnection（简单功能弱）java11的 HttpClient（支持异步/HTTP2）
    - 框架：SpringMVC 用到 RestTemplate，底层还是OkHttp或者HttpClient
    - 开源：OkHttp，Apache HttpClient

RPC：远程调用本地化