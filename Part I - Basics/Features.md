特性
====

MINA 是一个简单但功能齐全的网络应用框架，它提供：

* 为不同的传输类型统一 API：
	* 基于 Java NIO 的 TCP/IP 和 UDP/IP
	* 基于 RXTX 的 串行通信 (RS232)
	* In-VM 通道通信
	* 你可以实现你自己的！
* 过滤器接口作为扩展点；类似于 Servlet 过滤器
* 低层和高层 API：
    * 低层：使用 ByteBuffers
    * 高层：使用用户定义的消息对象和编解码器
* 高可定制化的线程模型：
    * 单个线程
    * 一个线程池
    * 多个线程池 (比如 SEDA)
* 使用 Java 5 SSLEngine 的开箱即用的 SSL · TLS · StartTLS 支持
* 超负载保护和流量调节
* 使用模拟对象单元测试
* JMX 可管理性
* 使用 StreamIoHandler 的基于流的 I/O 支持
* 知名容器诸如 PicoContainer 和 Spring 的集成
* 从 Netty 的平滑迁移，Netty 是 Apache MINA 的一个祖先 