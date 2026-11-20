# MQ → 工程场景 → 顺序消费

- **发送端：相同业务 KEY 路由到同一个队列 / 分区**
- **服务端：同一队列内消息 FIFO 存储**
- **消费端：单个队列只能单线程消费，禁止并发**

RabbitMQ

RabbitMQ **Exchange 不保证有序，只有单个队列内消息有序**

#### 实现顺序消费

1. 生产者：同一业务流水，发送到**同一个队列**（固定 routingKey 绑定到同一个 queue）
2. 消费端：**消费者 prefetch=1，一个队列只启动 1 个消费者实例**
3. 禁止开启并发多线程消费该队列

Kafka（最常用有序方案）

Kafka 有序粒度：**分区 (partition) 内有序，分区之间无序**

**生产端**：相同业务 ID（订单 id）做哈希，路由发送到**同一个 partition**

**Broker**：partition 内部日志严格 FIFO

**消费端：**一个 consumer group 内，**一个 partition 只能分配给一个 consumer 线程**（Kafka 原生机制）

RocketMQ

两种模式：分区有序（普通）、全局有序（极少用）

1. **生产端**：使用 `MessageQueueSelector`，相同业务 key 投递到同一个 MessageQueue
2. **Broker**：单个 queue 内部消息有序
3. **消费端**
使用 **顺序监听器 MessageListenerOrderly**
    - 同一队列的消息串行消费；
    - 消息消费失败：默认无限重试，**不会跳过消息**，保证后续消息等待，维持顺序。