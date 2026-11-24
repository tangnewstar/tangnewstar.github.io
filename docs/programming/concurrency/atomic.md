# 原子类与 CAS

### **atomic**

好的，下面我们来一步步分析Java并发包（J.U.C）中的atomic类。

### **概述**

`java.util.concurrent.atomic`包提供了一组原子变量类，这些类支持对单个变量的无锁线程安全操作。原子变量类包括：

- `AtomicBoolean`
- `AtomicInteger`
- `AtomicLong`
- `AtomicReference`
- `AtomicIntegerArray`
- `AtomicLongArray`
- `AtomicReferenceArray`

这些类通过使用底层的硬件支持（如CAS操作）来实现原子操作，从而避免了使用锁的开销。

### **AtomicInteger**

```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            // 获取内存偏移量，后续的CAS操作可以直接访问该字段
            // 因为Unsafe获取地址偏移量需要Field，所以这里反射获取
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;

    public AtomicInteger(int initialValue) {
        value = initialValue;
    }

    public AtomicInteger() {
    }

    // 常用方法
    `get()`: 获取当前值
    `set(int newValue)`: 设置新值
    `getAndIncrement()`: 获取当前值并递增
    `incrementAndGet()`: 递增并获取新值
    `compareAndSet(int expect, int update)`: 如果当前值等于预期值，则更新为新值
}

```

### **AtomicBoolean**

`AtomicBoolean`类提供了对布尔值的原子操作。它的实现类似于`AtomicInteger`，也是基于CAS操作来实现的。

```java
public class AtomicBoolean implements java.io.Serializable {
    private static final long serialVersionUID = 4654671469794556979L;
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset(AtomicBoolean.class.getDeclaredField("value"));
        } catch (Exception ex) {
            throw new Error(ex);
        }
    }

    // AtomicBoolean 也是用int存储值，将true|false映射成1和0
    private volatile int value;

    public AtomicBoolean(boolean initialValue) {
        value = initialValue ? 1 : 0;
    }

    public AtomicBoolean() {
    }

    public final boolean get() {
        return value != 0;
    }

    public final boolean compareAndSet(boolean expect, boolean update) {
        int e = expect ? 1 : 0;
        int u = update ? 1 : 0;
        return unsafe.compareAndSwapInt(this, valueOffset, e, u);
    }

    public final void set(boolean newValue) {
        value = newValue ? 1 : 0;
    }

    public final boolean getAndSet(boolean newValue) {
        int newVal = newValue ? 1 : 0;
        return unsafe.getAndSetInt(this, valueOffset, newVal) != 0;
    }
}

```

### **AtomicReference**

`AtomicReference`类允许对引用类型进行原子操作。它可以用于实现对对象引用的无锁线程安全操作。

```java
public class AtomicReference<V> implements java.io.Serializable {
    private static final long serialVersionUID = -1848883965231344442L;
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            // 同样获取value的内存偏移量
            // 基于offset去进行CAS操作 compareAndSwap
            valueOffset = unsafe.objectFieldOffset(AtomicReference.class.getDeclaredField("value"));
        } catch (Exception ex) {
            throw new Error(ex);
        }
    }

    private volatile V value;

    public AtomicReference(V initialValue) {
        value = initialValue;
    }

    public AtomicReference() {
    }

    // 获取当前值。
    public final V get() {
        return value;
    }

    // 设置新值。
    public final void set(V newValue) {
        value = newValue;
    }

    // 如果当前值等于预期值，则更新为新值。
    public final boolean compareAndSet(V expect, V update) {
        return unsafe.compareAndSwapObject(this, valueOffset, expect, update);
    }

    // 获取当前值并设置为新值。
    public final V getAndSet(V newValue) {
        return unsafe.getAndSetObject(this, valueOffset, newValue);
    }
}

```

### **AtomicIntegerArray**

`AtomicIntegerArray`类提供了对整数数组元素的原子操作。

和AtomicInteger，AtomicBoolean，AtomicLong反射获取内存偏移量类似的是，`AtomicIntegerArray` 需要操作数组中的元素，首先需要定位到元素的内存偏移量。

元素的内存地址偏移量 = 数组基地址偏移量 + 位移量（具体到那个索引位置）

基地址偏移量通过unsafe.arrayBaseOffset(int[].class)算得，是int[]的对象头大小

位移偏移量则是i << shift 位运算计算的，细节参考源码。

```java
public class AtomicIntegerArray implements java.io.Serializable {
    private static final long serialVersionUID = 2862133569453604235L;
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    // 数组元素的基地址偏移量（即第一个元素的内存偏移量），实际上返回的是数组对象头的大小
    private static final int base = unsafe.arrayBaseOffset(int[].class);
    private static final int shift;

    static {
        // 首先，用arrayIndexScale计算数组中元素的字节数，对于int就是4
        // 然后，校验计算得到的scale是不是2的幂次方(a power of two)，用(scale & (scale - 1)位计算
        // 最后，计算位移值为了高效索引。scale * index也能获取位移量但是效率不如位运算。
        // 31 - Integer.numberOfLeadingZeros(scale), 类似于以2为底的对数操作。log2(sclae)
        // 为什么类似于对数操作呢？Integer.numberOfLeadingZeros返回的是二进制的高位到第一个非0位之间的数量。
        // 31减去这个数相当于计算了scale占到二进制的第几位（从低位数0,1,2），这里算的是2。那么后面计算位移量就可以用 i*2^shift去计算，即 i << shift
        int scale = unsafe.arrayIndexScale(int[].class);
        if ((scale & (scale - 1)) != 0)
            throw new Error("data type scale not a power of two");
        shift = 31 - Integer.numberOfLeadingZeros(scale);
    }

    private final int[] array;

    public AtomicIntegerArray(int length) {
        array = new int[length];
    }

    public AtomicIntegerArray(int[] array) {
        this.array = array.clone();
    }

    private long checkedByteOffset(int i) {
        if (i < 0 || i >= array.length)
            throw new IndexOutOfBoundsException("index " + i);
        return byteOffset(i);
    }

    private static long byteOffset(int i) {
        // 元素内存偏移量 = 基地址偏移量 + (元素大小 * 元素索引)
        // int元素大小是4，i*2^shift = i * 4，不直接用i * 4可能是考虑到位运算性能更高
        return ((long) i << shift) + base;
    }

    // 获取数组中索引`i`处的值。
    public final int get(int i) {
        return getRaw(checkedByteOffset(i));
    }

    private int getRaw(long offset) {
        return unsafe.getIntVolatile(array, offset);
    }

    // 设置数组中索引`i`处的值为`newValue`。
    public final void set(int i, int newValue) {
        unsafe.putIntVolatile(array, checkedByteOffset(i), newValue);
    }

    // 获取数组中索引`i`处的当前值，并设置为`newValue`。
    public final int getAndSet(int i, int newValue) {
        return unsafe.getAndSetInt(array, checkedByteOffset(i), newValue);
    }

    // 如果数组中索引`i`处的值等于预期值`expect`，则更新为`update`。
    public final boolean compareAndSet(int i, int expect, int update) {
        return compareAndSetRaw(checkedByteOffset(i), expect, update);
    }

    private boolean compareAndSetRaw(long offset, int expect, int update) {
        return unsafe.compareAndSwapInt(array, offset, expect, update);
    }
}

```

### **AtomicLongArray**

`AtomicLongArray`类提供了对长整数数组元素的原子操作，其实现与`AtomicIntegerArray`类似，只是操作的数据类型是`long`。

### **AtomicReferenceArray**

`AtomicReferenceArray`类提供了对引用类型数组元素的原子操作。

```java
public class AtomicReferenceArray<E> implements java.io.Serializable {
    private static final long serialVersionUID = -6209656149925076980L;
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final int base = unsafe.arrayBaseOffset(Object[].class);
    private static final int shift;

    static {
        // 这部分和IntegerArray类似
        int scale = unsafe.arrayIndexScale(Object[].class);
        if ((scale & (scale - 1)) != 0)
            throw new Error("data type scale not a power of two");
        shift = 31 - Integer.numberOfLeadingZeros(scale);
    }

    private final Object[] array;

    public AtomicReferenceArray(int length) {
        array = new Object[length];
    }

    public AtomicReferenceArray(E[] array) {
        this.array = array.clone();
    }

    private long checkedByteOffset(int i) {
        if (i < 0 || i >= array.length)
            throw new IndexOutOfBoundsException("index " + i);
        return byteOffset(i);
    }

    // 计算内存字节偏移量
    private static long byteOffset(int i) {
        return ((long) i << shift) + base;
    }

    // 获取数组中索引`i`处的值。泛型
    public final E get(int i) {
        return getRaw(checkedByteOffset(i));
    }

    @SuppressWarnings("unchecked")
    private E getRaw(long offset) {
        return (E) unsafe.getObjectVolatile(array, offset);
    }

    // 设置数组中索引`i`处的值为`newValue`。
    public final void set(int i, E newValue) {
        unsafe.putObjectVolatile(array, checkedByteOffset(i), newValue);
    }

    // 获取数组中索引`i`处的当前值，并设置为`newValue`。
    public final E getAndSet(int i, E newValue) {
        return (E) unsafe.getAndSetObject(array, checkedByteOffset(i), newValue);
    }

    // 如果数组中索引`i`处的值等于预期值`expect`，则更新为`update`。
    public final boolean compareAndSet(int i, E expect, E update) {
        return compareAndSetRaw(checkedByteOffset(i), expect, update);
    }

    private boolean compareAndSetRaw(long offset, E expect, E update) {
        return unsafe.compareAndSwapObject(array, offset, expect, update);
    }
}

```

### **CAS操作**

`AtomicInteger`类的核心是使用CAS（Compare-And-Swap）操作来实现原子更新。CAS操作是一个硬件级别的原子操作，它比较内存中的值是否等于预期值，如果是，则更新为新值。

```java
public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
}

public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}

```

`AtomicInteger`类通过使用CAS操作实现了对单个整数变量的无锁线程安全操作。类似的，`AtomicBoolean`、`AtomicLong`、`AtomicReference`等类也使用了相同的机制来实现原子操作。

`AtomicIntegerArray`、`AtomicLongArray`和`AtomicReferenceArray`类通过使用`Unsafe`类的CAS操作实现了对数组元素的无锁线程安全操作。这些类在Java并发包（J.U.C）中提供了高效的原子操作，避免了使用锁的开销，从而提高了并发性能。

接下来，我们可以继续分析`Lock`及其实现。
