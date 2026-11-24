# 拓展篇：HashMap 源码深度分析

HashMap的数据结构是 **数组 + 链表/红黑树**

```java
transient Node<K,V>[] table; // 数组
static class Node<K,V> {      // 链表节点
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
}
static final class TreeNode<K,V> 
extends LinkedHashMap.Entry<K,V> { 
// 红黑树节点
    TreeNode<K,V> parent;
    TreeNode<K,V> left;
    TreeNode<K,V> right;
}

// 链表 -> 红黑树 阈值
static final int TREEIFY_THRESHOLD = 8;

// 红黑树 -> 链表 阈值
static final int UNTREEIFY_THRESHOLD = 6;
```

key → hash(key) → **index = hash & (n-1)** → table[index].put(Node<K,V>)

### hash(key) 哈希值计算

`(h = key.hashCode()) ^ (h >>> 16)`

hashCode的高16和低16位的异或计算，减少哈希冲突

### **hash & (n - 1) Bucket计算**

**get**

*通过getNode(hash(key), key)实现，返回node.value*

查找的节点条件是hash相同且key相等：`hash(node) = hash(key) & key.equals(node.key)`

- hash(key) 计算hash值，然后Bucket计算出索引位置
- 目标bucket找不到返回 null
- 链首节点是否为查找节点, hash(first) = hash(key) & key.equals(first.key)，返回first
- 如果是红黑树，从红黑树获取TreeNode
- 遍历链表，hash(node) = hash(key) & key.equals(node.key) 返回当前Node

**containsKey**

`*getNode(hash(key), key)  ≠ null` 即包含key*

**put**

- hash(key) 计算hash值，然后Bucket计算出索引位置
- 如果没有哈希碰撞，直接放在table[index]的bucket中（创建一个新Node）
- 如果存在哈希碰撞（table[index]不为空），和链首Node比较
    - 与链首Node相同（hash值和key值相等），替换Node旧的value, 返回旧值
    - 与链首Node不相同，遍历链表（红黑树）
        - 如果过程中发现Node相同，则替换Node旧value，返回旧值
        - 如果发现链表长度超过7，则treefiBin，**链表转化为红黑树**
        - 遍历到链尾没有相同的Node，则在链尾创建新的Node
- size累加，如果大于threshold则进行resize扩容

**resize()**

哈希表扩容，默认大小是16，负载因子是0.75

- 计算旧的哈希表的长度oldCap
- `oldCap > 0`
    - `*oldCap` > `MAXIMUM_CAPACITY(2^30)`,  capacity和threshold*保持不变
    - oldCap * 2 < 最大容量 & oldCap * 2 > `*DEFAULT_INITIAL_CAPACITY`(16)，newCap = oldCap double，同时 newThr = threshold double*
- old Cap = 0
    - threshold > 0, newCap = threshold, newThr = `newCap * loadFactor`
    - threshold = 0, newCap=默认容量16, *newThr = `DEFAULT_LOAD_FACTOR(0.75)* * *DEFAULT_INITIAL_CAPACITY`(16)*
- 初始化新的table容量为newCap,并遍历旧的table且table[index]不为空
    - 如果当前bucket(即table[index])只有链首Node，将改Node放入新的table（hash & (newCap - 1)重新计算相信table的index）
    - 如果当前bucket是红黑树，分割红黑树
    - 当前bucket是链表，遍历链表
        - 按顺序依次连接链表，head → … → tail → null
        - 分别维护两个链表，`hash & oldCap` == 0 旧的链表Node进入低位链表，否则进入高位链表
        - newTab[i] = 低位链表，newTab[i + oldCap]进入高位链表
    
    <aside>
    💡
    
    我的理解：将旧的bucket中的链表复制到新的bucket时，如果index相同，则新扩容出来的bucket全是空的失去了resize意义。通过`hash & oldCap`(1/2)位与预算，可以将旧的链表均匀分散到新的table中，减少哈希碰撞。
    
    </aside>
    

哈希表（数组）+ 链表（红黑树）实现了HashMap，链地址法（当索格key哈希索引位置相同时，在该索引位置创建一个链表并将多个key放入其中）

<aside>
💡

在JDK8中，如果链表的长度超过`TREEIFY_THRESHOLD` （默认值8），则转为红黑树

相反，如果红黑树在resize时长度低于`UNTREEIFY_THRESHOLD` （默认值6）则转为链表

</aside>

**什么是哈希冲突（碰撞）？**

哈希冲突（Hash Collision）是指在哈希表（Hash Table）中，不同的键（Key）经过哈希函数计算后，得到了相同的哈希值，从而被映射到同一个位置的现象。

这是HashMap计算hash值的逻辑：

```java
//Computes key. hashCode() and spreads (XORs) higher bits of hash to lower. 
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

**为什么不直接使用hashCod而是使用高16位和低16位的异或运算？**

- 对象的hashCode相同或分布不均可能导致哈希冲突
- 高16位参与到散列值的计算，减少hashCode低位相同时的哈希冲突
- 分散散列值使得散列值的分布均匀。提高链表和红黑树的长度，提升效率

**为什么不使用 key.hashCode % table.length，这样可以得到一个[0, table.length - 1)的值，为什么不使用取模运算来确定哈希表的索引位置呢？**

位运算比取模运算更高效（取模运算涉及到除）

**为什么哈希表的长度需要为n的2次幂？**

- n = 2 ^ k, **二进制是k个连续的1**，进行hash & (n -1) 与运算时能均匀分散到各个哈希表的索引中。
- n != 2 ^ k, **二进制不是连续的1**，进行hash & (n -1)与运算时，为0的位相当于总是0，这样会导致部分哈希表索引负载不均（被过度使用，一些索引很少用到）。

<aside>
💡

> n = 8, n - 1 = 7 (**0111)** 如果 n  = 9, n -1 = 8 **(1000) ,**这样任意数与运算的时候，低3位都失效了（全部是0）
> 
</aside>

**还有哪些哈希算法呢？**

先说⼀下hash算法⼲嘛的，Hash函数是指把⼀个⼤范围映射到⼀个⼩范围。把⼤范围映射到⼀个⼩范围的⽬的往往是为了 节省空间，使得数据容易保存。

⽐较出名的有`MurmurHash`、`MD4`、`MD5`等等

**为什么hashmap的在链表元素数量超过8时候改为红黑树**

- 知道jdk1.8中hashmap改了什么吗
    1. 由数组+链表的结构改为数组+链表+红⿊树
    2. 优化了⾼位运算的hash算法：h^(h>>>16)
    3. 扩容后，元素要么是在原位置，要么是在原位置再移动2次幂的位置，且链表顺序不变
- 说一下为什么会出现线程的不安全性
    1. HashMap 在`jdk1.7`中 使用 数组加链表的方式，并且在进行链表插入时候使用的是头结点插入的方法
        
        <aside>
        💡
        
        使用头插法，不用每次遍历链表。但是在进行resize的时候会**出现倒序**的情况，这个不难理解因为遍历顺序是head→tail, 和链表插入顺序相反。而且，使用头插法可能会导致链表成环，多线程下插入Node，交错执行会导致head节点被多次修改从而成环。
        
        示例伪代码：
        
        ```java
        head = null
        public void insertNode(Node newNode) {
        	newNode.next = head;
        	head = newNode;
        }
        ```
        
        </aside>
        
    2. 在JDK1.8中，虽然使用了尾插法，但仍不能保证线程安全。
        1. 并发插入同一个bucket会导致竞态条件数据不一致或丢失。
        2. 链表转为红黑树会进行遍历，会导致数据不一致
        3. iterator不支持并发修改，会导致**ConcurrentModificationException**
        4. resize操作的时候发生插入或删除，数据不一致或死锁
    3. 多线程使用ConcurrentHashMap，通过分段锁（Segment）或更细粒度的锁（在 Java 8 中）来保证线程安全。**synchronizedMap和Hashtable是线程安全，但是操作都是同步的，性能较差。**

- 为什么不一开始就使用红黑树，而是链表长度大于8转换为红黑树
    - 短链表（通常长度小于8）的性能优于红黑树。链表的插入和删除简单，不需要复杂的平衡操作。
    - 红黑树初始化成本高，每次插入删除需要平衡操作比链表效率低。当链表长度超过一定阈值（比如8）的时候，红黑树带来的查询效率提高，会优化HashMap的性能。
    - 实际应用中，bucket中的链表一般都是短链。没必要上来直接使用红黑树，当某些链表长度超长时，再进行红黑树的转化优化查询效率是更加平衡的选择。
- 红黑树何时转化为为链表
    - 长度低于`UNTREEIFY_THRESHOLD` （默认值6）则转为链表
    - 为什么是6是因为如果HashMap插入和删除频繁，设置和8距离差值2可以避免链表→红黑树→链表的来回转换。

**一般用什么作为HashMap的key值**

- **key可以是null吗，value可以是null吗**
    
    当然都是可以的，但是对于 key来说只能运行出现一个key值为null，但是可以出现多个value值为null
    
- **一般用什么作为key值**
    
    ⼀般⽤Integer、String这种不可变类当HashMap当key，⽽且String最为常⽤。
    
    (1)因为字符串是不可变的，所以在它创建的时候hashcode就被缓存了，不需要重新计算。 这就使得字符串很适合作为Map中的键，字符串的处理速度要快过其它的键对象。 这就是HashMap中的键往往都使⽤字符串。
    
    (2)因为获取对象的时候要⽤到equals()和hashCode()⽅法，那么键对象正确的重写这两个⽅法是⾮常重要的,这些类已 经很规范的覆写了hashCode()以及equals()⽅法。
    
    ```java
    // String, BigInteger, HashMap hashCode算法
    h * 31 + value[i]
    比如字符串 "abc"，计算哈希码时：
    
    初始 h = 0
    h = 31*0 + 'a' → h = 97
    h = 31*97 + 'b' → h = 3009
    h = 31*3009 + 'c' → h = 93300
    ```
    
- **用可变类当Hashmap1的Key会有什么问题**
    
    hashcode可能会发生变化，导致put进行的值，无法get出来.
    
- **实现一个自定义的class作为Hashmap的key该如何实现**
    - 重写hashcode和equals方法需要注意什么？
        
        两个对象相等，hashcode⼀定相等;两个对象不等，hashcode不⼀定不等
        
        hashcode相等，两个对象不⼀定相等;hashcode不等，两个对象⼀定不等
        
    - 如何设计一个不变的类
        - 类添加final修饰符，保证类不被继承。 如果类可以被继承会破坏类的不可变性机制，只要继承类覆盖⽗类的⽅法并且继承类可以改变成员变量值，那么⼀旦⼦类 以⽗类的形式出现时，不能保证当前类是否可变
        - 保证所有成员变量必须私有，并且加上final修饰 通过这种⽅式保证成员变量不可改变。但只做到这⼀步还不够，因为如果是对象成员变量有可能再外部改变其值。所以第4 点弥补这个不⾜
        - 不提供改变成员变量的⽅法，包括setter 避免通过其他接⼝改变成员变量的值，破坏不可变特性
        - 防御性拷贝，通过构造器初始化所有成员，进⾏深拷⻉(deep copy)
        - 在getter⽅法中，不要直接返回对象本⾝，⽽是克隆对象，并返回对象的拷⻉ 这种做法也是防⽌对象外泄，防⽌通过getter获得内部可变成员对象后对成员变量直接操作，导致成员变量发⽣改变

**总结**

1. 对于HashMap而言，扩容是一个特别消耗内存的操作。所以当程序员在使用HashMap的时候，估算map的大小，初始化的时候给一个大致的数值，避免map进行频繁的扩容
2. 负载因子是可以修改的，也可以大于1，但是建议不要轻易修改，除非情况非常特殊
3. HashMap是线程不安全的，不要在并发的环境中同时操作HashMap，建议使用ConcurrentHashMap
