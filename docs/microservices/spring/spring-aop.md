# Spring AOP

Spring AOP 是通过代理模式实现对目标对象的功能增强，而代理创建主要依赖两种技术：JDK动态代理和CGLIB动态代理，具体那种方式取决于目标类是否实现接口。

### JDK动态代理（基于接口）

- 适用：目标类至少实现了一个接口
- 原理：
    - 通过Java原生的 `java.lang.reflect.Proxy` 动态生成代理类对象，代理类实现了目标类的所有接口
    - 代理对象的方法调用会被转发到 `InvocationHandler`接口的 invoke方法，在该方法中实现增强逻辑（日志，事务），再通过反射调用目标对象的原始方法

```java
public static void main(String[] args) {
        UserService target = new UserServiceImpl();
        UserService proxy = (UserService) Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            new LogInvocationHandler(target)
        );
        proxy.save(); // 调用代理对象方法
    }
```

### CGLIB代理（基于继承）

- 适用：目标类没有实现接口（Spring默认选择）
- 原理：
    - CGLIB是一个字节码生成库，通过继承目标类创建代理类，并重写目标方法
    - 增强逻辑通过MethodInterceptor接口的intercept()实现，调用methodProxy.invokeSuper(proxy, args) 执行目标类的原始方法
- 限制：无法代理final方法和final类

```java
public static void main(String[] args) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(OrderService.class); // 设置父类
        enhancer.setCallback(new TransactionInterceptor());
        OrderService proxy = (OrderService) enhancer.create(); // 创建代理对象
        proxy.create(); // 调用代理对象方法
    }
```