# Redis → 分布式锁实现

分布式锁需要满足：互斥性、安全性、可用性、容错性。

#### 基本实现

**获取锁**：`SET key value NX PX expire_time`

> 
> 
> 
> NX：仅当 key 不存在时才设置 
> 
> PX：设置过期时间（毫秒），防止死锁
> 

**释放锁**（需保证原子性，使用 Lua 脚本）：

```jsx
if redis.call('get', KEYS[1]) == ARGV[1] then
    return redis.call('del', KEYS[1])
else
    return 0
end
```

#### 进阶方案

- **Redisson 框架**：提供自动续期、公平锁、联锁等高级功能

#### 实际使用

**设置合理过期时间**：确保业务能在过期前完成

**重试机制**：获取锁失败时重试

**异步通知**：使用发布订阅机制监听锁释放事件