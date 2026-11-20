# hashCode and equals

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

[HashMap底层原理](hash-map/hashmap-underlying.md) 

**`hashCode`**

- **哈希码契约**（Hash Code Contract）
    - 多次调用结果相同
    - equals相等则hashCode必须相同
- 作用
    - hashmap and hashtable，需要hashCode计算哈希值确定Bucket位置
    - 当Bucket位置相同时，需要equals判断是否为同一key