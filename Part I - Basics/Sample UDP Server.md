UDP Server 示例
====

现在我们看一下 [org.apache.mina.example.udp](http://mina.apache.org/mina-project/xref/org/apache/mina/example/udp/package-summary.html) 包里的代码。简单起见，我们将只专注于 MINA 相关构建方面的东西。

*译者注：翻译版本的项目源码见 <https://github.com/waylau/apache-mina-2-user-guide-demos> 中的`com.waylau.mina.demo.udp`包下*
        
要构建服务器我们需要做以下事情：

1. 创建一个 Datagram Socket 以监听连入的客户端请求 (参考 [MemoryMonitor.java](http://mina.apache.org/mina-project/xref/org/apache/mina/example/udp/MemoryMonitor.html))

2. 创建一个 IoHandler 以处理 MINA 框架生成的事件 (参考 [MemoryMonitorHandler.java](http://mina.apache.org/mina-project/xref/org/apache/mina/example/udp/MemoryMonitorHandler.html))
        
这里是第 1 点提到的一些代码片段：

	NioDatagramAcceptor acceptor = new NioDatagramAcceptor();
	acceptor.setHandler(new MemoryMonitorHandler(this));

这里我们创建了一个 NioDatagramAcceptor 以监听连入的客户端请求，并设置了 IoHandler。"PORT" 是一整型变量。下一步将要为这一 DatagramAcceptor 使用的过滤器链添加一个日志过滤器。LoggingFilter 实在 MINA 中表现不错的一个选择。它在不同阶段产生日志事务，以为我们观察 MINA 的工作情况。

   	DefaultIoFilterChainBuilder chain = acceptor.getFilterChain();
	chain.addLast("logger", new LoggingFilter());

接下来我们来看一些更具体的 UDP 传输的代码。我们设置 acceptor 以复用地址：

	DatagramSessionConfig dcfg = acceptor.getSessionConfig();
	dcfg.setReuseAddress(true);acceptor.bind(new InetSocketAddress(PORT));

  当然，要做的最后一件事就是调用 bind()。
        
## IoHandler 实现
        
对于我们服务器实现有三个主要事件：

* Session 创建
* Message 接收
* Session 关闭
        
我们分别看看它们的具体细节。
        
### Session 创建事件

	@Override
	public void sessionCreated(IoSession session) throws Exception {
	    SocketAddress remoteAddress = session.getRemoteAddress();
	    server.addClient(remoteAddress);
	}

在这个 session 创建事件中，我们调用了 addClient() 方法，它为界面添加了一个选项卡。
        
### Message 收到事件

	@Override
	public void messageReceived(IoSession session, Object message) throws Exception {
	    if (message instanceof IoBuffer) {
	        IoBuffer buffer = (IoBuffer) message;
	        SocketAddress remoteAddress = session.getRemoteAddress();
	        server.recvUpdate(remoteAddress, buffer.getLong());
	    }
	 }

在这个消息接收到事件中，我们对接收到的消息中的数据进行了处理。需要发送返回的应用，可以在这个方法中处理消息并回写响应。
        
### Session 关闭事件

	@Override
	public void sessionClosed(IoSession session) throws Exception {
	    System.out.println("Session closed...");
	    SocketAddress remoteAddress = session.getRemoteAddress();
	    server.removeClient(remoteAddress);
	}

在 Session 关闭事件中，我们将客户端选项卡从界面中移除。