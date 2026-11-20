# 分布式事务

**TCC 模式：业务侵入式的 “强一致性” 方案**

TCC（Try-Confirm-Cancel）完全脱离数据库事务，通过 **业务代码手动定义 “预操作” 和 “补偿操作”**，适合需要跨服务（如调用第三方 API）的场景。

**Saga 模式：长事务的 “最终一致性” 方案**

Saga 适合 **长事务** 或 **无法预留资源** 的场景（如物流、订单履约流程），通过 **“正向流程 + 反向补偿”** 实现最终一致，允许中间状态存在。

Seata Example

```java
package com.example.member.service;

import com.example.member.mapper.MemberMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import io.seata.spring.annotation.GlobalTransactional;

@Service
public class MemberService {

    @Autowired
    private MemberMapper memberMapper;  // 操作会员数据库
    @Autowired
    private PayFeignClient payFeignClient;  // 调用支付服务
    @Autowired
    private PointFeignClient pointFeignClient;  // 调用积分服务

    // 标记为全局事务，TM 角色
    @GlobalTransactional(rollbackFor = Exception.class)  // 发生任何异常都会触发全局回滚
    public void purchaseMember(Long userId, BigDecimal amount, int points) {
        // 1. 本地事务：创建会员订单（状态为"已支付"）
        memberMapper.createMemberOrder(userId, amount);
        
        // 2. 远程调用支付服务：扣减用户余额（分支事务1）
        boolean paySuccess = payFeignClient.deductBalance(userId, amount);
        if (!paySuccess) {
            throw new RuntimeException("支付失败，触发全局回滚");
        }
        
        // 3. 远程调用积分服务：赠送积分（分支事务2）
        boolean pointSuccess = pointFeignClient.addPoints(userId, points);
        if (!pointSuccess) {
            throw new RuntimeException("积分赠送失败，触发全局回滚");
        }
    }

    // Feign 客户端：调用支付服务
    @FeignClient(name = "pay-service")
    public interface PayFeignClient {
        @Transactional
        boolean deductBalance(Long userId, BigDecimal amount);
    }

    // Feign 客户端：调用积分服务
    @FeignClient(name = "point-service")
    public interface PointFeignClient {
        @Transactional
        boolean addPoints(Long userId, int points);
    }
}

```