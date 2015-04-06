UDP Client 示例
====

前面我们讲了 UDP 服务器端示例，现在我们来看一下与之对应的客户端代码。
        
客户端的实现需要做的事情：

* 创建套接字并连接到服务器端
* 设置 IoHandler
* 收集空闲内存
* 发送数据到服务器端
        
现在我们看一下 org.apache.mina.example.udp.client 包中的 [MemMonClient.java](http://mina.apache.org/mina-project/xref/org/apache/mina/example/udp/client/MemMonClient.html)。前几行代码简单明了：

	connector = new NioDatagramConnector();
	connector.setHandler( this );
	ConnectFuture connFuture = connector.connect( new InetSocketAddress("localhost", MemoryMonitor.PORT ));

我们创建了一个 NioDatagramConnector，设置了处理器然后连接到服务器。我曾经落入的一个陷阱是，你必须在 InetSocketAddress 对象中设置主机，否则它将什么也不干。这个例子是在一台 Windows XP 主机上编写并测试，因此在其他环境中可能会有所不同。解析来我们将等待客户端连接到的主机的确认。一旦得知我们已经建立连接，我们就可以开始向服务器端写数据了：

	connFuture.addListener( new IoFutureListener(){
	    public void operationComplete(IoFuture future) {
	        ConnectFuture connFuture = (ConnectFuture)future;
	        if( connFuture.isConnected() ){
	            session = future.getSession();
	            try {
	                sendData();
	            } catch (InterruptedException e) {
	                e.printStackTrace();
	            }
	        } else {
	            log.error("Not connected...exiting");
	        }
	    }
    });

这里我们为 ConnectFuture 对象添加了一个监听者，当我们接收到客户端已建立连接的回调时，我们就可以写数据了。向服务器端写数据将会由一个叫做 sendData 的方法处理。这个方法如下所示：

	private void sendData() throws InterruptedException {
	    for (int i = 0; i < 30; i++) {
	        long free = Runtime.getRuntime().freeMemory();
	        IoBuffer buffer = IoBuffer.allocate(8);
	        buffer.putLong(free);
	        buffer.flip();
	        session.write(buffer);
	        try {
	            Thread.sleep(1000);
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	            throw new InterruptedException(e.getMessage());
	        }
	    }
	}

这个方法将在 30 秒之内的每秒钟向服务器端发送一次空闲内存的数量。在这里你可以看到我们分配了一个足够大的 IoBuffer 来保存一个 long 类型变量，然后将空闲内存的数量放进缓存。缓冲随即写给服务器端。

UDP Client 实现完成。