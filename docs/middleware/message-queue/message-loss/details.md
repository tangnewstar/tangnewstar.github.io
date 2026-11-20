# MQ → 工程场景 → 消息丢失

#### RabbitMQ

- **生产端**：事务同步、confirm机制
    - 事务同步：**开启事务会带来性能消耗，吞吐量下降（同步）**
        
        > 
        > 
        > - 发送数据之前开启事务 `channel.txSelect` ，发送消息
        > - 如果RabbitMQ没有接收到消息生产者会接收到异常，此时回滚事务`channel.txRollback()`，然后重试
        > - 如果接收到消息，那么提交事务`channel.txCommit()`
    - confirm机制：**支持异步回调不阻塞发送**
        
        > 
        > 
        > - 生产者开启confirm模式后，每次写入消息都会分配一个唯一id
        > - 如果RabbitMQ接收并成功处理则返回`ack`
        > - 如果没有能处理这个消息则会回调 `nack` 接口通知失败，进行重试
        > - confirm模式有两种：同步与异步
        >     - 同步confirm：发送一条或一批消息后，调用`channel.waitForConfirms()` 确认confirm状态
        >     - 异步confirm：提供一个回调方法，服务端confirm后通知客户端回调该方法。`channel.addConfirmListener(new ConfirmListener() {})`
- MQ：消息持久化
    - 声明队列为持久 `durable = true`
    - 消息投递模式设置持久化 `deliveryMode = 2`
    
    > 防止临时消息在内存中重启丢失：**本地消息表**（先插消息表标记待发送，定时扫描失败消息重试，收到ack后更新为成功）
    > 
- 消费端：消息确认机制
    - **关闭自动ack：**消费者在声明队列时，可以指定 noAck 参数，当 noAck=false，RabbitMQ 会等待消费者显式发回 ack 信号后，才从内存（和磁盘，如果是持久化消息）中移去消息。

#### Kafka

- 生产端：未收到 broker 确认，程序认为成功
    - `acks=all` （acks是ISR列表中等待应答的实例数量）
    - `retries>0`：发送失败自动重试
- MQ：leader 宕机，follower 切换为 leader 之后，leader节点数据丢失
    - 防止单节点故障：`min.insync.replicas ≥2`
    - 多副本：`replication.factor ≥3`
    - 刷盘策略：不强制刷盘（操作系统页缓存）提升性能
- 消费端：自动提交 offset 时机早于业务处理完成
    - 关闭自动提交 `enable.auto.commit=false`
    - 手动提交offset：**完全处理成功之后**，再调用 commitSync/commitAsync 提交偏移量
    - 异常处理：消费失败不提交 offset；抛出异常，消息下次重拉。

#### RocketMQ

- 生产端：发送未抵达 Broker，业务误以为发送成功
    - 优先使用**同步发送 sendSync**；异步发送必须实现回调处理失败消息
    - 设置重试次数 `retryTimesWhenSendFailed`，发送失败主动重试
    - 使用**事务消息 / 本地消息表**保证本地事务与发消息原子性（防止本地库成功、消息发送失败）
- MQ：消息存入内存还未落盘、主节点宕机，副本未同步
    - 集群部署 **主从架构（DLedger 集群）**，开启消息同步复制
    - 刷盘策略：同步刷盘 `FlushDiskType=SYNC_FLUSH`（高可靠，性能低）
- 消费端：消息到达消费者，Broker 标记消费成功，但业务逻辑执行失败
    - 同步执行业务，处理成功才返回成功
    - 异常返回重试，不吞噬异常；监听死信队列