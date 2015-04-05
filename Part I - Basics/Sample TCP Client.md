TCP Client 示例
====

在上文中我们已经了解了客户端架构。现在我们将展示一个客户端实现的示例。
        
我们将使用 [Sumup Client](http://mina.apache.org/mina-project/xref/org/apache/mina/example/sumup/Client.html) 作为一个参考实现。
        
我们将移除掉样板代码并专注于重要结构上。以下是为客户端代码：
	
	public static void main(String[] args) throws Throwable {
	    NioSocketConnector connector = new NioSocketConnector();
	    connector.setConnectTimeoutMillis(CONNECT_TIMEOUT);
	
	    if (USE_CUSTOM_CODEC) {
	    connector.getFilterChain().addLast("codec",
	        new ProtocolCodecFilter(new SumUpProtocolCodecFactory(false)));
	    } else {
	        connector.getFilterChain().addLast("codec",
	            new ProtocolCodecFilter(new ObjectSerializationCodecFactory()));
	    }
	
	    connector.getFilterChain().addLast("logger", new LoggingFilter());
	    connector.setHandler(new ClientSessionHandler(values));
	    IoSession session;
	
	    for (;;) {
	        try {
	            ConnectFuture future = connector.connect(new InetSocketAddress(HOSTNAME, PORT));
	            future.awaitUninterruptibly();
	            session = future.getSession();
	            break;
	        } catch (RuntimeIoException e) {
	            System.err.println("Failed to connect.");
	            e.printStackTrace();
	            Thread.sleep(5000);
	        }
	    }
	
	    // wait until the summation is done
	    session.getCloseFuture().awaitUninterruptibly();
	    connector.dispose();
	}

要构建一个客户端，我们需要做以下事情：

* 创建一个 Connector
* 创建一个 Filter Chain
* 创建一个 IOHandler 并添加到 Connector
* 绑定到服务器

下面解释每个细节

## 创建一个 Connector

	NioSocketConnector connector = new NioSocketConnector();

现在，我们已经创建了一个 NIO Socket 连接器

## 创建一个 Filter Chain

	if (USE_CUSTOM_CODEC) {
	    connector.getFilterChain().addLast("codec",
	        new ProtocolCodecFilter(new SumUpProtocolCodecFactory(false)));
	} else {
	    connector.getFilterChain().addLast("codec",
	        new ProtocolCodecFilter(new ObjectSerializationCodecFactory()));
	}

我们为 Connector 的 Filter Chain 添加了一些过滤器。这里我们添加的是一个 ProtocolCodec 到过滤器链。 

## 创建 IOHandler

	connector.setHandler(new ClientSessionHandler(values));

这里我们创建了一个 [ClientSessionHandler](http://mina.apache.org/mina-project/xref/org/apache/mina/example/sumup/ClientSessionHandler.html) 的实例并将其设置为 Connector 的处理器。

## 绑定到服务器
	
	IoSession session;
	
	for (;;) {
	    try {
	        ConnectFuture future = connector.connect(new InetSocketAddress(HOSTNAME, PORT));
	        future.awaitUninterruptibly();
	        session = future.getSession();
	        break;
	    } catch (RuntimeIoException e) {
	        System.err.println("Failed to connect.");
	        e.printStackTrace();
	        Thread.sleep(5000);
	    }
	}

这是最重要的部分。我们将连接到远程服务器。因为是异步连接任务，我们使用了 [ConnectFuture](http://mina.apache.org/mina-project/xref/org/apache/mina/core/future/ConnectFuture.html) 来了解何时连接完成。一旦连接完成，我们将得到相关联的 [IoSession](http://mina.apache.org/mina-project/xref/org/apache/mina/core/session/IoSession.html)。要向服务器端发送任何消息，我们都要写入 session。所有来自服务器端的响应或者消息都将穿越 Filter chain 并最终由 IoHandler 处理。

*译者注：翻译版本的项目源码见 <https://github.com/waylau/apache-mina-2-user-guide-demos> 中的`com.waylau.mina.demo.sumup`包下*