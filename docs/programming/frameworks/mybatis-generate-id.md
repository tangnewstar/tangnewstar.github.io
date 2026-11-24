# 拓展篇：MyBatis 生成数据库 ID

mybatis生成主键的五种策略

```java
public enum IdType {
		// 数据库主键自增
    AUTO(0),
    // 用户自行处理生成逻辑,由数据库底层机制生成（如序列、触发器），或需自定义生成器（如实现 IdentifierGenerator 接口
    NONE(1),
    // 用户手动赋值, 代码手动设置值
    INPUT(2),
    // ：自定义算法生成数值型主键（如雪花算法）,由 MyBatis-Plus 内置的 IdentifierGenerator 自动生成 Long 类型 的分布式 ID（默认雪花算法）
    ASSIGN_ID(3),
    // UUID 主键, 由 MyBatis-Plus 自动生成 32 位 UUID 字符串（不含短横线）
    ASSIGN_UUID(4);
}
```

**`MybatisParameterHandler`**

通过看MyBatis Plus的文档，发现这个主键生成过程应该是在ParameterHandler中进行处理的。在MyBatis Plus中ParameterHandler是用来设置参数规则的，当StatementHandler调用prepare方法之后，接下来就是调用它来进行设置参数。

```java
protected void populateKeys(TableInfo tableInfo, MetaObject metaObject, Object entity) {
        IdType idType = tableInfo.getIdType();
        String keyProperty = tableInfo.getKeyProperty();
        if (StringUtils.isNotBlank(keyProperty) && null != idType && idType.getKey() >= 3) {
        //如果是 ASSIGN_ID | ASSIGN_UUID
            IdentifierGenerator identifierGenerator = GlobalConfigUtils.getGlobalConfig(this.configuration).getIdentifierGenerator();
            Object idValue = metaObject.getValue(keyProperty);
            // 如果idValue不存在，根据类型进行计算
            if (identifierGenerator.assignId(idValue)) {
                if (idType.getKey() == IdType.ASSIGN_ID.getKey()) {
                    Class<?> keyType = tableInfo.getKeyType();
                    // 如果是Number，计算nextId（基于时间戳）DefaultIdentifierGenerator
                    if (Number.class.isAssignableFrom(keyType)) {
                        Number id = identifierGenerator.nextId(entity);
                        if (keyType == id.getClass()) {
                            metaObject.setValue(keyProperty, id);
                        } else if (Integer.class == keyType) {
                            metaObject.setValue(keyProperty, id.intValue());
                        } else if (Long.class == keyType) {
                            metaObject.setValue(keyProperty, id.longValue());
                        } else if (BigDecimal.class.isAssignableFrom(keyType)) {
                            metaObject.setValue(keyProperty, new BigDecimal(id.longValue()));
                        } else {
                            if (!BigInteger.class.isAssignableFrom(keyType)) {
                                throw new MybatisPlusException("Key type '" + keyType + "' not supported");
                            }

                            metaObject.setValue(keyProperty, new BigInteger(id.toString()));
                        }
                    } else {
		                    // 否则设置主键为nextId的字符串
                        metaObject.setValue(keyProperty, identifierGenerator.nextId(entity).toString());
                    }
                } else if (idType.getKey() == IdType.ASSIGN_UUID.getKey()) {
		                // ASSIGN_UUID 计算
                    metaObject.setValue(keyProperty, identifierGenerator.nextUUID(entity));
                }
            }
        }

    }
```

`DefaultIdentifierGenerator`

```java
public class DefaultIdentifierGenerator implements IdentifierGenerator {
    private final Sequence sequence;

    public DefaultIdentifierGenerator() {
        this.sequence = new Sequence((InetAddress)null);
    }

    public DefaultIdentifierGenerator(InetAddress inetAddress) {
        this.sequence = new Sequence(inetAddress);
    }

    public DefaultIdentifierGenerator(long workerId, long dataCenterId) {
        this.sequence = new Sequence(workerId, dataCenterId);
    }

    public DefaultIdentifierGenerator(Sequence sequence) {
        this.sequence = sequence;
    }
		
    public Long nextId(Object entity) {
		    // 调用sequence.nextId计算
        return this.sequence.nextId();
    }
}
```

`Sequence`

```java
public synchronized long nextId() {
        long timestamp = this.timeGen();
        // 维护了一个SystemClock的单例每毫秒去刷新当前时间，用AtomicLong now存储
        // this.timeGen()拿的就是now的时间
        // 系统当前时间对比上一次生成ID的时间戳
        if (timestamp < this.lastTimestamp) {
            long offset = this.lastTimestamp - timestamp;
            // 允许回拨最多5s
            if (offset > 5L) {
                throw new RuntimeException(String.format("Clock moved backwards.  Refusing to generate id for %d milliseconds", offset));
            }

            try {
		            // 等待两倍偏移时间（补偿可能的时钟误差）
                this.wait(offset << 1);
                timestamp = this.timeGen();
                // 回拨后没有追上上次生成ID的时间，抛出异常
                if (timestamp < this.lastTimestamp) {
                    throw new RuntimeException(String.format("Clock moved backwards.  Refusing to generate id for %d milliseconds", offset));
                }
            } catch (Exception var6) {
                throw new RuntimeException(var6);
            }
        }

        if (this.lastTimestamp == timestamp) {
		        // 同一毫秒内序列号递增，最多4096
            this.sequence = this.sequence + 1L & 4095L;
            if (this.sequence == 0L) {
			          // 序列号耗尽，等待下一毫秒
                timestamp = this.tilNextMillis(this.lastTimestamp);
            }
        } else {
		        // 新的毫秒随机选择1,2，降低连续Id可预测性       
            this.sequence = ThreadLocalRandom.current().nextLong(1L, 3L);
        }

        this.lastTimestamp = timestamp;
        
        // 雪花算法
        // 时间戳：64-1-22 = 41bit
        // datacenterId，workerId：5bit
        // sequence：12bit
        return timestamp - 1288834974657L << 22 | this.datacenterId << 17 | this.workerId << 12 | this.sequence;
    }
```

<aside>
💡

**为什么使用 SystemClock 单例而非直接调用 System.currentTimeMillis()？**

- **性能优化：降低系统调用开销**

**System.currentTimeMillis()** 是一个 **本地方法（Native Method）**，每次调用需从操作系统内核获取时间戳，而内核时间更新频率可能低于应用层需求（如 Linux 默认 1ms 更新一次）。在高并发场景下（如每秒生成数十万 ID），频繁调用此方法会带来 **显著的性能损耗**。
**SystemClock** 单例通过 **后台线程定时刷新时间戳**（如每 1ms），将时间戳存储到 **AtomicLong** 中。此时获取时间戳只需 **读取内存变量**，性能远高于系统调用。
单次调用耗时：**System.currentTimeMillis()** 20~50ns,  **SystemClock.now()** 约 1~5ns ****

- **时间一致性：避免同一毫秒内时间戳跳跃**

**System.currentTimeMillis()**，在同一毫秒内多次调用可能返回 **不同的值**（如第一次调用返回 **t1**，第二次调用返回 **t1+1**），导致时间戳跳跃。而雪花算法要求同一毫秒内生成的 ID 必须使用 **相同的时间戳**，仅通过序列号区分唯一性。
**SystemClock** 单例 **固定每毫秒更新一次时间戳，**所有线程在同一毫秒内调用 **timeGen()** 时，获取的时间戳完全一致，确保 **时间戳部分的稳定性**。

- **容错性：规避时钟回拨风险**

直接依赖 **System.currentTimeMillis()** 可能因 **NTP 时间同步** 或 **人为修改系统时间** 导致时钟回拨（时间戳突然变小）。雪花算法对时钟回拨极其敏感，若时间戳回退，生成的 ID 可能重复。
1. **定期同步时间**：每毫秒从系统获取一次时间，而非每次生成 ID 时获取，减少突发性时钟回拨的影响。
2. **统一异常处理**：时钟回拨检测逻辑集中在 **SystemClock** 中，而非分散在每次 ID 生成时。

- **雪花算法的时间戳要求**

雪花算法的核心是 **时间戳的单调递增性**。

**基准时间对齐**：雪花算法的时间戳通常以某个固定时间点（如**1288834974657L**）为基准，计算相对时间差。

**序列号连续性**： 同一毫秒内的时间戳固定，序列号从随机初始值（如 1 或 2）递增，确保同一毫秒内生成的 ID 连续且唯一。

总结：减少系统调用（高性能），时钟一致性（不会同一毫秒内时间戳不同），降低时钟回拨风险（逻辑统一维护在SystemClock）

</aside>

<aside>
💡

相同毫秒的序号递增，最多4096？

1. this.sequence + 1L
首先，this.sequence + 1L 表示将当前的 sequence 值加 1。这里使用了长整型（1L）来确保运算不会溢出，尤其是在处理大数值时。
2. & 4095L
接下来，& 4095L 是一个位操作符，具体来说是 按位与 操作。4095 的二进制表示是 111111111111（12个1），也就是 2^12 - 1 或者 0xFFF。这个操作的作用是将 this.sequence + 1L 的结果限制在 0 到 4095 之间循环。
• **按位与操作**：当我们将任意一个数与 **4095** 进行按位与操作时，实际上是在保留该数的低 12 位，而将其高 12 位清零。
</aside>
