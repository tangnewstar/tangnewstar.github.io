# Spring Bean 扫描

### **1. `@ComponentScan` (Spring)：**

- 用于指定哪些包下的组件会被自动注册为Bean
- 主要属性：
    - `basePackages`：指定扫描包路径
    - `excludeFilters`：过滤不需要的组件
- 例：`@ComponentScan("com.example")`

---

### **2. `@SpringBootApplication` (Spring Boot)：**

- 组合注解，包含：
    - `@Configuration`：标记为配置类
    - `@EnableAutoConfiguration`：启用自动配置
    - `@ComponentScan`：自动扫描组件
- 简化配置，默认扫描主类所在包及子包

---

💡 **总结：**

- `@ComponentScan`：Spring基础注解，用于组件扫描
- `@SpringBootApplication`：Spring Boot组合注解，内含`@ComponentScan`
- 使用`@SpringBootApplication`时无需手动配置`@ComponentScan`