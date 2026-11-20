# Web Socket

### **🧠 1. WebSocket 是基于 HTTP 的？**

是的，**WebSocket** 是建立在 **HTTP 协议之上** 的，但它只是一个**升级协议**，用于在 **建立连接之后** 将 HTTP 的 **长连接**（keep-alive）升级为 **真正的双向通信通道**。这种升级过程叫做 **WebSocket 协议握手**。

- **握手过程** 是基于 HTTP 的，例如发送 `GET /chat HTTP/1.1` 请求，加上 `Upgrade: websocket` 和 `Sec-WebSocket-Key` 等头部。
- 握手成功后，就建立了一个 **WebSocket 会话**，后续通信不再依赖 HTTP，而是使用 **自定义的 WebSocket 协议**。

---

### **🧠 2. Netty 在 WebSocket 中如何实现消息的自动拆分与合并？**

在 **Netty** 中，WebSocket 的消息处理是通过 **解码器（Decoder）和编码器（Encoder）** 来实现的，特别是 **WebSocketFrame** 和 **WebSocketClientHandler** 等类。

### **✅ 关键机制：**

- **WebSocketFrame** 是 Netty 提供的用于表示 WebSocket 消息的类，它可以是：
    - `TextFrame`（文本消息）
    - `BinaryFrame`（二进制消息）
    - `ContinuationFrame`（延续帧）
    - `PingFrame`、`PongFrame`（用于心跳）
- **Netty 自动处理拆包/粘包**：Netty 的 `WebSocketFrameDecoder` 会自动处理数据的 **拆分与合并**，因为它知道 WebSocket 是一种 **分帧协议**，每帧都有 **帧头信息**，包括：
    - **FIN bit（消息结束标志）**
    - **RSV bits（保留位）**
    - **Opcode（操作码）**（表示数据类型）
    - **Mask（掩码）** 等

> Netty 的 FrameDecoder 会将接收到的 TCP 流按照 WebSocket 的规范，自动识别帧头并解析出完整的帧，从而处理粘包和拆包，用户体验非常友好。
>