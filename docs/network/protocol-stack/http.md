# HTTP

HTTP 1.1

- 支持长连接（默认开启）单个连接处理多个请求
- 支持管道化：同一连接并发请求，按序响应

HTTP 2

- 二进制分帧
    - 多路复用：同一个TCP连接多个请求并行传输，帧独立标识编号，无队头阻塞）
- Server Push：服务器主动推送关联资源 JC/CSS
- 头部压缩：减少重复请求头

HTTP3

- QUIC协议，彻底解决TCP对头阻塞的问题

<aside>
💡

HTTP和HTTPS

</aside>

HTTPS是HTTP+TLS/SSL，在传统的TCP三次握手上，加上TLS四次握手

- HTTP是明文传输，HTTPS是加密传输。加密方式：非对称加密 + 对称加密
    - TLS四次握手（非对称加密）：
        - client → server：TLS1.2 + RSA + client random
        - server → client：TLS1.2 + RSA + server random + 证书
        - client → server：公钥（证书取出）+ pre_master_key → 公钥加密pre_master_key，客户端使用3个随机数生成会话密钥，生成Finish摘要（用会话密钥加密） → server
        - server → client：私钥解密pre_master_key，3个随机数用加密算法生成会话密钥，生成Finish摘要发送客户端
    - 使用会话密钥继续通信（对称加密）