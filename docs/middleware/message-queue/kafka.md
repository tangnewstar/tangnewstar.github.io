# MQ → Kafka

**Kafka集群的读写都是在Leader节点，其他副本Follower会异步同步Leader的数据。**

**Leader是by per partition，如果Leader节点不可用会触发重新选举。**

Kafka Topic → Partition(P0,P1,P2,P3) → Replicas(Leader, Follower)

- 每个分区中有多个replica，包含一个Leader和多个Follower
- 生产者将消息发送到Leader，消费者从Leader读取消息
- Leader在创建分区时由Controller初始化，或在Failover时重新选举产生
    - Kafka心跳机制检测leader不可用,Controller触发重新选举
    - Kafka Controller自动从当前分区的**ISR集合**中选取一个replica作为新的Leader
    - 如果ISR为空（其他replica落后leader太多）
        - **unclean.leader.election.enable = false，只能从ISR集合选Leader**
        
        > 分区不可用直到有replica追上Leader，生产者异常**NotEnoughReplicasException** or **NotLeaderForPartitionException，**消费者异常
        > 
        - **unclean.leader.election.enable = true, 可以从任意可用replica选Leader**
        
        > 分区仍然可用，但是会导致数据丢失不一致（生产者数据丢失，消费者消费过时数据）
        > 
- 每个分区有多个Follower，能追赶上Leader的replica进入ISR，落后的从ISR移除
    - **滞后时间 replica.lag.time.max.ms**
        - **如果Follower在这个时间内追上了Leader的数据，会加入ISR集合**
        - 如果Follower在这个时间内没有追上，会从ISR集合移除
    - **日志末尾偏移量 Log End Offset, LEO**
        - 每个副本都有一个日志末尾偏移量（LEO），表示该副本上最后一条消息的偏移量
        - 每个分区有一个高水位线（HW），表示所有ISR中的副本都已复制到的最小偏移量
        - Follower副本的LEO必须大于或等于Leader的HW，才能被认为是同步的
    - **心跳和拉取请求**
        - Follower副本会定期向Leader发送心跳，以表明自己仍然活跃
        - Follower副本会定期从Leader拉取消息数据。如果Follower副本能够及时地从Leader拉取并写入数据，那么它会被认为是同步的
    
- Controller更新分区元数据，并将Leader信息广播给所有broker
- 客户端重定向，客户端的生产者和消费者会根据更新后的元数据将请求定向到新的Leader

Kafka Cluster → kafka broker 0(1,2,3) → topic_a(b,c…) → partiontions

### Kafka的运行模式

- 单节点模式：只有一个broker，没有高可用和容错
- 多节点模式：多个broker实例，提供高可用和容错
    - 支持多个分区和副本，实现负载均衡和数据冗余
- 分布式模式：多个Kafka集群分布在多个IDC或Area，通过KafkaMirrorMaker实现跨集群复制和同步
- Kafka Connect：用于将Kafka与其他文件系统（如数据库，文件系统，消息队列）集成，提供Source Connector（数据导入Kafka）和 Sink  Connector（数据导出Kafka）
- Kafka Streams：用于实时流处理
- Kafka On k8s：使用k8s来管理Kafka集群

### 是否必须配置多个分区

- 单个分区：这种情况下，该topic的所有消息都是顺序写入和读取。但是会限制并行处理的能力，所有消息通过一个分区处理
- 多个分区：为了提高吞吐量和并行处理能力，通常建议为一个Topic配置多个分区。这样可以实现负载均衡，允许多个消费者同时消费不同的分区，提高整体速度

### 多broker环境下生产者和消费者

kafka集群下，一个topic的分区可以分布在多个broker上。生产者和消费者通过与这些broker来进行交互发送和消费消息

**生产者的连接和数据发送**

- **初始连接**：生产者通常会配置一个或多个“引导”broker（bootstrap broker）。这些broker是生产者首次连接的目标。生产者通过这些broker获取集群的元数据，包括所有broker的信息、主题及其分区的分布情况等

<aside>
💡

bootstrap broker 配置一个或多个有区别吗

- 一个broker：配置简单，但也可能存在当前broker不可用，无法与集群建立连接
- 多个broker：增加了容错，一个broker不可用仍可以和kafka进群连接并获取元数据
</aside>

- **元数据获取**：生产者从引导broker获取集群的元数据后，会知道每个主题的分区以及这些分区所在的broker。这样，生产者就可以直接与负责特定分区的broker通信

<aside>
💡

元数据获取完如何确定要连接的broker

- 元数据的信息：broker的ID和地址；每个topic的分区列表；每个分区的Leader broker和其他副本broker
- 生产者确定目标broker
    - 根据消息的键（key）或自定义的分区策略，确定消息应该发送到哪个分区
    - 通过元数据找到该分区的Leader broker，并直接将消息发送到该broker
- 消费者确定目标broker
    - 根据消费者的分配策略（如Range Assignment、Round Robin Assignment或Sticky Assignment），确定每个消费者应该消费哪些分区。
    - 通过元数据找到这些分区的Leader broker，并从这些broker拉取消息进行消费。
</aside>

- **发送到哪个分区**
    - **分区策略**：生产者可以选择多种策略来决定将消息发送到哪个分区
        - **轮询（Round Robin）**：消息依次发送到不同的分区。
        - **键哈希（Key Hashing）**：根据消息的键（key）计算哈希值，并将其映射到特定的分区。这样，具有相同键的消息总是会被发送到同一个分区。
        - **自定义策略**：生产者可以实现自定义的分区器（partitioner），根据业务逻辑决定消息发送到哪个分区。

**消费者的连接和数据消费**

- **初始连接**：消费者同样需要配置一个或多个引导broker。消费者通过这些broker获取集群的元数据。
- **元数据获取**：消费者从引导broker获取集群的元数据后，会知道每个主题的分区以及这些分区所在的broker。这样，消费者就可以直接与负责特定分区的broker通信。
- **消费哪个分区的数据**
    - **消费者组**：消费者通常属于一个消费者组（consumer group）。同一个消费者组内的消费者共同消费一个主题的所有分区。每个消费者组中的消费者会分配到该主题的不同分区。
    - **分区分配策略**：Kafka提供了几种分区分配策略
        - **Range Assignment**：按范围分配分区。例如，如果有10个分区和3个消费者，第一个消费者会消费分区0-2，第二个消费者会消费分区3-6，第三个消费者会消费分区7-9。
        - **Round Robin Assignment**：按轮询方式分配分区。每个消费者依次分配一个分区，直到所有分区都被分配完毕。
        - **Sticky Assignment**：粘性分配。尽量保持消费者对分区的分配不变，以减少重新平衡时的开销。

<aside>
💡

**集群的元数据信息**

- **集群ID**：唯一标识一个Kafka集群。
- **控制器ID**：当前担任控制器角色的broker ID。
- **活跃的Broker列表**：包含所有活跃的broker的ID和地址。
    - **Broker ID**：每个broker的唯一标识
    - **主机名/地址**：broker的网络地址。
    - **端口**：broker监听的端口号。
    - **Rack信息**：broker所在的机架信息（如果配置了RackAwareness）。
- **主题信息**
    - **主题名称**：唯一标识一个topic。
    - **分区数量**：该topic被划分为多少个分区。
    - **副本因子**：每个分区有多少个副本。
    - **配置属性**：包括压缩类型、保留策略等。
- **分区信息**
    - **分区ID**：每个分区的唯一标识。
    - **Leader**：当前负责处理该分区读写请求的broker ID。
    - **Replicas**：该分区的所有副本所在的broker ID列表。
    - **ISR (In-Sync Replicas)**：与Leader保持同步的副本列表。
    - **HW (High Watermark)**：消费者可以读取的最大偏移量。
    - **LEO (Log End Offset)**：Leader分区日志文件中的最后一个消息的偏移量。
</aside>

**broker不可用时如何处理**

- **Broker故障检测：**Kafka集群Controller监控broker状态
- **重新选举Leader**：broker宕机，上面所有的leader分区会重新选举
- **数据读取和写入**：选举过程中，受影响的分区暂时不可用。选举完成后恢复。