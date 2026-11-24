# Java 规范

Java 语言层面的核心规范与契约，从 `hashCode` / `equals` 契约开始。

## hashCode 与 equals 契约

官方规范

- 如果对象相等，则hashCode必须相等。
- 如果hashCode相等，equals不一定相等。

<aside>
💡

如果hashCode不等equals相等会怎么样

</aside>

hashCode计算Bucket的位置，如果hashCode不等

- HashMap get取不出相等元素
- 重复插入相等元素，无法更新元素

[HashMap底层原理](../data-structures/hashmap.md) 

**`hashCode`**

- **哈希码契约**（Hash Code Contract）
    - 多次调用结果相同
    - equals相等则hashCode必须相同
- 作用
    - hashmap and hashtable，需要hashCode计算哈希值确定Bucket位置
    - 当Bucket位置相同时，需要equals判断是否为同一key

## toString / compareTo

### toString

`Object.toString()` 默认返回 `类名@十六进制hashCode`，对排查问题几乎没有帮助。POJO、值对象应重写 `toString`，输出关键字段（注意脱敏），便于日志与调试；也可由 IDE 或 Lombok `@ToString` 生成。

### Comparable 与 Comparator

- **Comparable<T>.compareTo(T o)**：定义类的**自然排序**，`this < o` 返回负数、相等返回 0、大于返回正数。
- **Comparator<T>**：定义**外部排序**，无需修改类本身；JDK 8 可用 `Comparator.comparing(Person::getAge).thenComparing(Person::getName)` 组合比较器，用 `nullsLast()` 处理 null。
- **约定**：建议让 `compareTo == 0` 与 `equals == true` 保持一致，否则 `TreeSet` / `TreeMap` 等有序集合会与 `HashSet` / `HashMap` 的“相等”判断不一致。
- **避免溢出**：不要写 `return a - b`（大数相减可能溢出），应使用 `Integer.compare(a, b)` 或 `Long.compare(a, b)`。

```java
class Person implements Comparable<Person> {
    private final int age;

    @Override
    public int compareTo(Person other) {
        return Integer.compare(this.age, other.age);
    }
}

Comparator<Person> byAgeDesc = Comparator.comparingInt(Person::getAge).reversed();
```

## clone

- `Object.clone()` 是 `protected` 方法，默认执行**浅拷贝**：基本类型复制值，引用类型只复制引用，新旧对象仍共享同一份引用对象。
- 类必须实现标记接口 `Cloneable`，否则调用 `clone()` 会抛 `CloneNotSupportedException`。
- 约定：将 `clone()` 重写为 `public`，返回类型使用协变类型，并在方法内先调用 `super.clone()`。

```java
class Point implements Cloneable {
    int x, y;

    @Override
    public Point clone() throws CloneNotSupportedException {
        return (Point) super.clone(); // 浅拷贝
    }
}
```

- **深拷贝**：除复制对象本身外，还要递归复制其引用对象。可通过拷贝构造器、`ObjectOutputStream` 序列化、JSON / ProtoBuf 转换等方式实现。
- **局限**：`Cloneable` 没有方法，约束依赖约定；`clone()` 不调用构造器，`final` 字段难以处理。《Effective Java》更推荐用**拷贝构造器或拷贝工厂**替代 `Cloneable`。

## Serializable

- `Serializable` 是标记接口，通过 `ObjectOutputStream` / `ObjectInputStream` 将对象写入字节流并从字节流恢复。
- **serialVersionUID**：显式声明 `private static final long serialVersionUID`。未声明时 JVM 会根据类结构计算，类结构稍作修改就可能导致反序列化抛 `InvalidClassException`。
- **transient**：被修饰的字段不参与序列化，反序列化后为默认值；需要自定义逻辑时提供 `readObject(ObjectInputStream)` / `writeObject(ObjectOutputStream)`。
- **反序列化安全**：
  - 反序列化不经过构造器，必须校验数据来源与内容；对不可信输入使用 `ObjectInputFilter`（JDK 9+）限制可反序列化的类。
  - 单例类需提供 `readResolve()` 返回既有实例，避免反序列化破坏单例。
  - 跨服务传输优先选择 JSON / ProtoBuf 等更安全、可演进的格式。

```java
class User implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private transient String password; // 不落盘、不传输
}
```

## 编码规约（对齐《阿里 Java 开发手册》）

| 类别 | 规约要点 |
| --- | --- |
| 命名 | 类名 `UpperCamelCase`；方法/变量 `lowerCamelCase`；常量 `UPPER_SNAKE_CASE`；包名全小写；POJO 布尔属性不要以 `is` 开头；缩写保持统一（如 `userId`） |
| 常量 | 不允许魔法值直接出现在代码中；常量类使用 `private` 构造器，禁止实例化 |
| 判空 | 对象相等用 `Objects.equals(a, b)`；字符串用常量调用 `CONSTANT.equals(str)`；集合用 `CollectionUtils.isEmpty(...)` 或 `list == null \|\| list.isEmpty()`；尽早返回避免深层嵌套 |
| 集合处理 | 遍历中删除元素用 `Iterator.remove()` 或 `removeIf()`，不要用 for-each 直接 `remove`；`new HashMap<>(expectedSize / 0.75f + 1)` 预分配容量；`Arrays.asList` 返回的列表不可 `add` / `remove`；空结果返回 `Collections.emptyList()` 而不是 `null` |
| 分支与日志 | `switch` 必须有 `default`；超过 3 层 `if-else` 改用卫语句；使用日志框架与占位符 `{}`，不用 `System.out.println` 或字符串拼接 |

## 延伸阅读

- [异常体系](exceptions.md)：异常设计与处理原则
