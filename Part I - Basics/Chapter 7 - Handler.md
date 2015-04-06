处理器
====

处理 MINA 所触发 I/O 事件。这一接口时在过滤器链最后完成的所有活动的中心。
        
IoHandler 具有以下方法：

* sessionCreated
* sessionOpened
* sessionClosed
* sessionIdle
* exceptionCaught
* messageReceived
* messageSent
        
## sessionCreated

会话建立事件在一个新的连接被创建时触发。对于 TCP 来说这是连接接受的结果，而对于 UDP 这个在接收到一个 UDP 包时产生。这一方法可以被用于初始化会话属性，并为一些特定连接执行一次性活动。
        
这个方法由 I/O 处理线程的环境中调用，因此应该以一个低耗时的方式实现，因为同一个线程要处理多个会话。

## sessionOpened

会话打开事件是在一个连接被打开时调用。它总是在 sessionCreated 事件之后调用。如果配置了一个线程模型，这一方法将在 I/O 处理线程之外的一个线程中调用。

## sessionClosed
   
会话关闭事件在会话被关闭时调用。会话清理活动比如清理支付信息可以在这里执行。

## sessionIdle

会话空闲时间在会话变为闲置状态时触发。这一方法并不为基于 UDP 的传输调用。

## exceptionCaught

这一方法在用户代码或者 MINA 抛异常时调用。如果是一个 IOException 异常的话当前连接将被关闭。

## messageReceived

消息接收事件在一个消息被接收到时触发。这是一个应用最常发生的处理。你需要照顾到所有要碰到的消息类型

## messageSent

消息发送事件在消息响应被发送 (调用 IoSession.write()) 时触发。