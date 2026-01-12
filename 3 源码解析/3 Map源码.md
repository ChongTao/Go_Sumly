# 1 map 内存模型

在源码中，表示 map 的结构体是`hmap`

```go
type hmap struct {
    count     int    // 元素个数
    flags     uint8
    B         uint8  // bucket 数 = 2^B
    noverflow uint16 // 溢出桶的近似数量
    hash0     uint32 // 哈希随机种子

    buckets    unsafe.Pointer // 当前 bucket 数组
    oldbuckets unsafe.Pointer // 扩容时的旧 bucket
    nevacuate  uintptr        // 已迁移 bucket 的进度
    clearSeq   uint64
    extra *mapextra // 可选字段（溢出桶优化）
}
```

buckets是一个指针，指向bmap

```go
type bmap struct {
    // OldMapBucketCountBits = 3 // log2 of number of elements in a bucket.
	// OldMapBucketCount     = 1 << OldMapBucketCountBits
	tophash [abi.OldMapBucketCount]uint8 
}
```

bmap就是map中的桶，该桶容量是8个key，映射关系如下：

![](https://golang.design/go-questions/map/assets/0.png)

当map的key和value值都不是指针，并且size小于128字节时，动态将`bmap`标记为不含指针的结构（目的：避免GC时扫描整个hmap）。

# 2 map原理

Golang中的map实现原理主要基于散列表（hashtable）的思想。散列表是一个数据结构，它使用哈希函数将键（key）映射到数组的一个索引位置，从而实现键值对的快速存储和检索。

1. **哈希函数**：用于将键映射到数组的索引位置。在Golang中，哈希函数的选择和具体实现是语言内部处理的，对用户来说是透明的。
2. **桶（bucket）**：Golang中的map并不是直接存储键值对，而是使用桶作为基本的存储单元。每个桶可以存储一个或多个键值对。当向map中插入一个新的键值对时，根据键的哈希值计算出对应的桶，并将键值对存储在该桶中。
3. **冲突解决**：由于哈希函数可能会将不同的键映射到同一个索引位置（即哈希冲突），Golang使用链表或其他数据结构（如开放寻址法）来解决冲突。当两个或多个键哈希到同一个桶时，它们会按照某种方式（如链表）链接在一起，以便在查找时能够遍历并找到正确的键值对。
4. **动态扩展**：随着map中键值对数量的增加，可能需要更多的桶来存储它们。Golang的map实现了动态扩展的机制，当桶的数量不足以容纳新的键值对时，会重新分配一个更大的数组，并将现有的键值对重新哈希到新的桶中。



