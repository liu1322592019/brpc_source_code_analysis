建议参考官方文档，在我的实际使用过程中，主要体现为
- 对客户端重试的限制
    - 使用backup request
    - 使用MQ削峰填谷
    - 服务降级
    - ...
- 对服务端并发的限制
    - 设置最大并发
    - 设置HPA自动扩容分散单机压力
    - ...







