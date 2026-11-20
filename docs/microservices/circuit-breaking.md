# 熔断限流与降级

Gateway限流：网关层限流

- 算法：令牌桶（RequestRateLimiter），可以扩展其他算法
- 分布式：支持分布式限流（RedisRateLimiter）
- 维度：路径、参数、ip、用户

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10  # 令牌桶每秒填充速率（允许的平均QPS）
                redis-rate-limiter.burstCapacity: 20  # 令牌桶最大容量（允许的突发QPS）
                key-resolver: "#{@ipKeyResolver}"  # 基于IP的限流（需自定义KeyResolver）
```

Hystrix限流：主要用于服务间容错，限流是通过隔离功能实现的（停止维护，推荐resilience4j）

- 原理：基于资源隔离
    - 线程池（默认）：为每个服务分配独立线程池控制线程池大小从而控制并发请求数；
    - 信号量：通过信号量计数器控制并发请求数
- 适用：A调用B的局部限流
- 使用方式：通过注解@EnableCircuitBreaker和在方法上注解@HystrixCommand实现

```java
@HystrixCommand(
    threadPoolKey = "userServicePool",  // 线程池标识
    threadPoolProperties = {
        @HystrixProperty(name = "coreSize", value = "10"),  // 核心线程数（最大并发数）
        @HystrixProperty(name = "maxQueueSize", value = "20")  // 队列最大长度
    },
    fallbackMethod = "queryUserFallback"  // 降级方法
)
public User queryUser(Long id) {
    return userServiceClient.queryById(id);  // 调用下游服务
}
```

Resilience4J：更轻量现代化，默认使用信号量

```java
// Resilience4j 基于函数式编程 + 注解
@CircuitBreaker(name = "userService", fallbackMethod = "getUserFallback")
public User getUser(Long id) {
    return userServiceClient.getById(id);
}

// 降级方法
public User getUserFallback(Long id, Exception e) {
    return new User(id, "默认用户（降级）");
}

// 配置熔断规则（可动态调整）
@Bean
public CircuitBreakerConfig userServiceCircuitBreakerConfig() {
    return CircuitBreakerConfig.custom()
        .failureRateThreshold(50) // 失败率阈值50%
        .slidingWindowSize(10)    // 滑动窗口大小10
        .build();
}
```

Sentinel限流：分布式系统的流量治理，功能全面：限流、熔断、降级、系统保护

- 特点：
    - 支持多种限流模式（QPS，线程数，并发数），多种限流策略（直接拒绝，WarmUp，匀速排队）
    - DashBoard动态配置限流规则
    - 细粒度控制，基于资源（接口，方法）配置限流规则，支持按调用来源，路径，参数等维度限流

```java
@RestController
public class OrderController {
    @GetMapping("/api/orders/{id}")
    @SentinelResource(value = "queryOrder", blockHandler = "queryOrderBlocked")
    public Order queryOrder(@PathVariable Long id) {
        return orderService.getById(id);
    }

    // 限流降级方法
    public Order queryOrderBlocked(Long id, BlockException e) {
        return new Order(id, "限流保护中");
    }
}

@PostConstruct
public void initFlowRules() {
    List<FlowRule> rules = new ArrayList<>();
    FlowRule rule = new FlowRule();
    rule.setResource("queryOrder");  // 关联资源名
    rule.setGrade(RuleConstant.FLOW_GRADE_QPS);  // 基于QPS限流
    rule.setCount(10);  // QPS阈值10
    rules.add(rule);
    FlowRuleManager.loadRules(rules);
}
```
