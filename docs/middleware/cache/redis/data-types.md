# Redis → 数据类型

### 数据类型

- String

one key for one value. 

常用于：缓存，计数器（incr）分布式锁（SETNX）Session共享

- Set

支持操作：交集、并集、差集

no duplicate elements

底层为intset和hashtable（key存储元素，value为null）当存储类型不为整数使用哈希表

常用于：标签系统（用户兴趣标签），去重（活跃用户），共同好友（交集）

- List

有序集合，支持刷想插入和删除。

底层为QuickList（结合压缩列表和双向链表），时间复杂度O(1)的头尾操作

常用于：消息队列（LPUSH + RPOP) 栈（LPUSH + LPOP）历史记录（最近搜索）

- Hash

键值对的无序集合。适用于存储对象的多个属性

底层为压缩列表或哈希表，支持O(1)时间复杂度的读写。

- ZSet

插入、删除、查找的复杂度都是O(logn)，支持分数范围查询（ZRANGEBYSCORE）

底层为Hashtable（存储元素）skiplist（分值排序）

常用于：排行榜（游戏排名）延迟队列（时间戳排序）

- HyperLogLog

基数估算（独立数量统计）命令：`PFADD / PFCOUNT / PFMERGE`

用途：UV、独立 IP 统计，恒定 12KB

- BitMap

利用位操作高效存储和统计大量boolean（用户签到，活跃状态）

Redis提供了对字符串的Bit读写功能（GETBIT/SETBIT），可以将一个字符串视为二进制（每个字符8个bit）

示例：用一个10亿位的BitMap存储用户每天签到状态，仅需119MB

- Stream

Stream是专为消息队列设计的数据类型。支持发布订阅、消费者组、消息持久化等。

每个消息都有唯一id，支持消费者组内负载均衡和消息确认（ACK）

常用于：实时消息系统、异步任务队列