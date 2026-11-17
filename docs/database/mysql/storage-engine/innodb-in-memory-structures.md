---
title: InnoDB In-Memory Structures
---

# InnoDB In-Memory Structures

### Buffer Pool

The buffer pool is an area in main memory where `InnoDB` caches table and index data as it is accessed. The buffer pool permits frequently used data to be accessed directly from memory, which speeds up processing. On dedicated servers, up to 80% of physical memory is often assigned to the buffer pool.

!!! note "提示"
    Buffer Pool 是主内存的一块区域，用来缓存InnoDB访问的表和索引数据。

    允许频繁访问的数据直接通过内存访问，加速数据处理。在专用于MYSQL的服务器上，通常将物理内存的80%分配给Buffer Pool

For efficiency of high-volume read operations, the buffer pool is divided into pages that can potentially hold multiple rows. For efficiency of cache management, the buffer pool is implemented as a linked list of pages; data that is rarely used is aged out of the cache using a variation of the least recently used (LRU) algorithm.

!!! note "提示"
    为了大容量读操作的效率，buffer pool 被分为可以持有多行的页。

    为了缓存管理的效率，buffer pool被实现为 数据页的链表，很少使用到的数据会使用LRU算法的辩题来过期。
