[Brpc版的std::string](#Brpc版的std::string) 

## Brpc版的std::string

brpc使用[butil::IOBuf](https://github.com/apache/brpc/blob/master/src/butil/iobuf.h)作为一些协议中的附件或http body的数据结构，它是一种`非连续零拷贝缓冲`，在其他项目中得到了验证并有出色的性能。IOBuf的接口和std::string类似，但不相同。

`IOBuf能做的`
- 默认构造不分配内存。
- 可以拷贝，修改拷贝不影响原IOBuf。拷贝的是IOBuf的管理结构而不是数据。
- 可以append另一个IOBuf，不拷贝数据。
- 可以append字符串，拷贝数据。
- 可以从fd读取，可以写入fd。
- 可以解析或序列化为protobuf messages.
- IOBufBuilder可以把IOBuf当std::ostream用。

`IOBuf不能做的`：
- 程序内的通用存储结构。IOBuf应保持较短的生命周期，以避免一个IOBuf锁定了多个block (8K each)。


## 相关代码

// 修改拷贝不影响原IOBuf。拷贝的是IOBuf的管理结构而不是数据

```C++
void IOBuf::append(const IOBuf& other) {
    const size_t nref = other._ref_num();
    for (size_t i = 0; i < nref; ++i) {
        // 遍历数据块, 拷贝数据结构
        _push_back_ref(other._ref_at(i));
    }
}

inline void IOBuf::_push_back_ref(const BlockRef& r) {
    if (_small()) {
        return _push_or_move_back_ref_to_smallview<false>(r);
    } else {
        return _push_or_move_back_ref_to_bigview<false>(r);
    }
}

template <bool MOVE>
void IOBuf::_push_or_move_back_ref_to_bigview(const BlockRef& r) {
    BlockRef& back = _bv.ref_at(_bv.nref - 1);
    // 两个块是连在一起的(指向了同一段内存中相连的部分), 直接合并
    if (back.block == r.block && back.offset + back.length == r.offset) {
        // Merge ref
        back.length += r.length;
        _bv.nbytes += r.length;
        if (MOVE) {
            r.block->dec_ref(); // 原子操作减引用计数, 本场景不执行
        }
        return;
    }

    if (_bv.nref != _bv.capacity()) {
        // 有空闲的控制块, 拷贝r到末尾并增加引用计数
        _bv.ref_at(_bv.nref++) = r;
        _bv.nbytes += r.length;
        if (!MOVE) {
            r.block->inc_ref(); // 增加引用计数
        }
        return;
    }
    
    // 扩容 resize, don't modify bv until new_refs is fully assigned
    const uint32_t new_cap = _bv.capacity() * 2;
    BlockRef* new_refs = iobuf::acquire_blockref_array(new_cap);
    for (uint32_t i = 0; i < _bv.nref; ++i) {
        new_refs[i] = _bv.ref_at(i);
    }
    new_refs[_bv.nref++] = r; // 拷贝r到末尾

    // Change other variables
    _bv.start = 0;
    iobuf::release_blockref_array(_bv.refs, _bv.capacity());
    _bv.refs = new_refs;
    _bv.cap_mask = new_cap - 1;
    _bv.nbytes += r.length;
    if (!MOVE) {
        r.block->inc_ref(); // 增加引用计数
    }
}
```
