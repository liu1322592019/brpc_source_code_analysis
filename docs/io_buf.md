[Brpc版的std::string](#Brpc版的std::string) 

## Brpc版的std::string

 - IOBuf is a `non-continuous buffer` that can be cut and combined w/o copying payload.
 - It can be `read from or flushed into file descriptors` as well.
 - IOBuf is **thread-compatible**. Namely using different IOBuf in different threads simultaneously is safe, and reading a static IOBuf from different threads is safe as well.
 - IOBuf is **NOT thread-safe**. Modifying a same IOBuf from different threads simultaneously is unsafe and likely to crash.

我还没发现其中的奥妙，先不写了
