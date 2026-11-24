# 动态代理

动态代理在不修改目标类的前提下，为方法调用增加统一逻辑（日志、事务、权限、重试等），是 AOP 与各类框架基础设施的常用实现手段。

## JDK 动态代理

基于**接口**生成代理类，运行时创建实现同一组接口的 `$Proxy0` 类：

```java
public interface UserService {
    void create(String name);
}

UserService target = new UserServiceImpl();

UserService proxy = (UserService) Proxy.newProxyInstance(
        target.getClass().getClassLoader(),
        new Class<?>[]{UserService.class},
        (Object p, Method method, Object[] args) -> {
            System.out.println("before: " + method.getName());
            try {
                return method.invoke(target, args); // 调用真实对象
            } finally {
                System.out.println("after: " + method.getName());
            }
        });

proxy.create("Tom");
```

要点：

- 只能代理**接口方法**，目标类必须实现接口；
- `InvocationHandler.invoke(Object proxy, Method method, Object[] args)` 中若把 `proxy` 再传给 `method.invoke`，会递归调用代理自身，应传入 `target`；
- 从接口中继承的 `equals` / `hashCode` / `toString` 也会被代理。

## CGLIB 动态代理

基于**继承**生成目标类的子类，通过方法拦截实现增强，无需接口：

```java
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(UserServiceImpl.class);
enhancer.setCallback((MethodInterceptor) (obj, method, args, proxy) -> {
    System.out.println("before: " + method.getName());
    try {
        return proxy.invokeSuper(obj, args); // 调用父类（目标类）方法
    } finally {
        System.out.println("after: " + method.getName());
    }
});
UserServiceImpl proxy = (UserServiceImpl) enhancer.create();
```

要点：

- `final` 类无法被继承，`final` / `private` / `static` 方法无法被重写增强；
- 通过 ASM 直接操作字节码生成子类，首次创建代理的成本通常高于 JDK 代理；
- Spring 若目标实现了接口默认使用 JDK 代理，否则使用 CGLIB；可通过 `proxyTargetClass = true` 强制使用 CGLIB。

## 两种方式对比

| 维度 | JDK 动态代理 | CGLIB |
| --- | --- | --- |
| 实现方式 | 生成实现接口的代理类 | 生成目标类的子类 |
| 是否需要接口 | 必须 | 不需要 |
| `final` 类/方法 | 不涉及（接口方法无 `final`） | 不能代理/增强 |
| 创建成本 | 较低 | 较高（字节码生成） |
| 调用性能 | 现代 JDK 中差距很小 | 现代 JDK 中差距很小 |
| 依赖 | JDK 内置 `java.lang.reflect.Proxy` | 第三方库（或框架内置，如 Spring Core 中的 CGLIB） |

## 典型应用

- Spring AOP：声明式事务、日志、权限校验；
- MyBatis：为 Mapper 接口生成代理对象，将方法调用转为 `SqlSession` 操作；
- RPC 框架：生成本地 Stub，把接口调用封装成网络请求。

## 延伸阅读

- [反射](reflection.md)：`Method.invoke` 与 `setAccessible`
- [SPI 机制](spi.md)：另一种“面向接口编程”的扩展机制
