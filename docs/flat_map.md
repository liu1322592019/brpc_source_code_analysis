[FlatMap官方释义](#FlatMap官方释义)

[FlatMap相关代码](#FlatMap相关代码)

## FlatMap官方释义
[FlatMap](https://github.com/apache/brpc/blob/master/src/butil/containers/flat_map.h)可能是最快的哈希表，但当value较大时它需要更多的内存，它最适合作为检索过程中需要极快查找的**小字典**。

原理：`把开链桶中第一个节点的内容直接放桶内。由于在实践中，大部分桶没有冲突或冲突较少，所以大部分操作只需要一次内存跳转`：通过哈希值访问对应的桶。桶内两个及以上元素仍存放在链表中，由于桶之间彼此独立，一个桶的冲突不会影响其他桶，性能很稳定。**在很多时候，FlatMap的查找性能和原生数组接近**。

## FlatMap相关代码

数据结构:
```C++
struct Bucket {
    Bucket* next;
    char element_spaces[sizeof(Element)];
};

template <typename _K, typename _T,
          typename _Hash = DefaultHasher<_K>, 
          typename _Equal = DefaultEqualTo<_K>,
          bool _Sparse = false,
          typename _Alloc = PtAllocator>
class FlatMap {
private:
    size_t _size;
    size_t _nbucket;
    Bucket* _buckets; // _buckets桶, array
    uint64_t* _thumbnail;
    u_int _load_factor; // 扩容的指数
    hasher _hashfn; // Compute hash code from key, 默认使用polynomial hash function, 几乎和stl同款
    key_equal _eql; // 比较key是否一致的函数
    SingleThreadedPool<sizeof(Bucket), 1024, 16, allocator_type> _pool; // 内存池, 类似于slab
}
```
operator[]:
```c++
template <typename _K, typename _T, typename _H, typename _E, bool _S, typename _A>
_T& FlatMap<_K, _T, _H, _E, _S, _A>::operator[](const key_type& key) {
    // 直接指针偏移, 快的一批
    const size_t index = flatmap_mod(_hashfn(key), _nbucket);
    Bucket& first_node = _buckets[index];

    if (!first_node.is_valid()) {
        // 节点不可用状态
        ++_size;
        if (_S) {
            bit_array_set(_thumbnail, index);
        }
        // 新增节点
        new (&first_node) Bucket(key);
        return first_node.element().second_ref();
    }

    if (_eql(first_node.element().first_ref(), key)) {
        // 第一个元素key相等, 返回
        return first_node.element().second_ref();
    }

    Bucket *p = first_node.next;
    if (NULL == p) {
        // 链表没有下一个元素 -> key不存在
        if (is_too_crowded(_size)) {
            // 扩容
            if (resize(_nbucket + 1)) {
                return operator[](key);
            }
            // fail to resize is OK
        }
        ++_size;
        // 从内存池申请块内存, new一下返回
        Bucket* newp = new (_pool.get()) Bucket(key);
        first_node.next = newp;
        return newp->element().second_ref();
    }

    // 遍历链表
    while (1) {
        if (_eql(p->element().first_ref(), key)) {
            // 第一个元素key相等, 返回
            return p->element().second_ref();
        }
        // 链表没有下一个元素 -> key不存在
        if (NULL == p->next) {
            if (is_too_crowded(_size)) {
                // 扩容
                if (resize(_nbucket + 1)) {
                    return operator[](key);
                }
                // fail to resize is OK
            }
            ++_size;
            // 从内存池申请块内存, new一下返回
            Bucket* newp = new (_pool.get()) Bucket(key);
            p->next = newp;
            return newp->element().second_ref();
        }
        p = p->next;
    }
}
```
