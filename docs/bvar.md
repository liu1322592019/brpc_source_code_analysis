[bvar是什么](#bvar是什么)

## bvar是什么

以下摘抄自brpc官方文档:
- bvar是多线程环境下的计数器类库，支持单维度bvar和多维度mbvar，方便记录和查看用户程序中的各类数值，它利用了thread local存储减少了cache bouncing，bvar的本质是把`写时的竞争转移到了读`：读得合并所有写过的线程中的数据，而不可避免地变慢了。

