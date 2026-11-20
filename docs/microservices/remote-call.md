# 远程调用

@FeignClient → 启动生成代理对象（JDK代理） → 解析接口方法的注解，获取Url和参数 → 负载均衡（Ribbon默认，Nacos） →  选择服务实例 → 发送HTTP请求（Encoder） → 解析响应（Decoder）
