TCP Server 示例
====

接下来的教程介绍构建基于 MINA 的应用的过程。这个教程介绍的是构建一个时间服务器。本教程需要以下先决条件：

* MINA 2.x Core
* JDK 1.5 或更高
* SLF4J 1.3.0 或更高
	- Log4J 1.2 用户：slf4j-api.jar、slf4j-log4j12.jar 和 Log4J 1.2.x
	- Log4J 1.3 用户：slf4j-api.jar、slf4j-log4j13.jar 和 Log4J 1.3.x
	- java.util.logging 用户：slf4j-api.jar 和 slf4j-jdk14.jar
	- 重要：请确认你用的是和你的日志框架匹配的 slf4j-*.jar

例如，slf4j-log4j12.jar 和 log4j-1.3.x.jar 一起使用的话，将会发生故障

我们已经在 Windows 2000 professional 和 linux 之上对程序进行了测试。如果你在运行本程序时遇到任何问题，不要犹豫请[联系我们](http://mina.apache.org/contact.html)，让我们通知 MINA 的开发人员。另外，本教程尽量保留了独立的开发环境 (IDE、编辑器...等等)。本教程示例适用于你所喜爱的任何开发环境。篇幅所限，本文省略掉了关于程序的编译命令以及执行步骤。如果你在编译或执行 Java 程序时需要帮助，请参考 [Java 教程](http://docs.oracle.com/javase/tutorial/)。

##编写 MINA 时间服务器
        
我们以创建一个叫做 MinaTimeServer.java 的文件开始。初始化代码如下：

	public class MinaTimeServer {
	    public static void main(String[] args) {
	        // code will go here next
	    }
	}

这段程序对所有人来说都很简单明了。我们简单定义了一个用于启动程序的 main 方法。现在，我们开始添加组成我们服务器的代码。首先，我们需要一个用于监听传入连接的对象。因为本程序基于 TCP/IP，我们在程序中添加了 SocketAcceptor。

import org.apache.mina.transport.socket.nio.NioSocketAcceptor;

	public class MinaTimeServer
	{
	    public static void main( String[] args )
	    {
	        IoAcceptor acceptor = new NioSocketAcceptor();
	    }
	}

NioSocketAcceptor 类就绪了，我们继续定义处理类并绑定 NioSocketAcceptor 到一个端口：

	import java.net.InetSocketAddress;
	
	import org.apache.mina.core.service.IoAcceptor;
	import org.apache.mina.transport.socket.nio.NioSocketAcceptor;
	
	public class MinaTimeServer
	{
	    private static final int PORT = 9123;
	    public static void main( String[] args ) throws IOException
	    {
	        IoAcceptor acceptor = new NioSocketAcceptor();
	        acceptor.bind( new InetSocketAddress(PORT) );
	    }
	}

如你所见，有一个关于 acceptor.setLocalAddress( new InetSocketAddress(PORT) ); 的调用。这个方法定义了这一服务器要监听到的主机和端口。最后一个方法是 IoAcceptor.bind() 调用。这个方法将会绑定到指定端口并开始处理远程客户端请求。

接下来我们在配置中添加一个过滤器。这个过滤器将会日志记录所有信息，比如 session 的新建、接收到的消息、发送的消息、session 的关闭。接下来的过滤器是一个 ProtocolCodecFilter。这个过滤器将会把二进制或者协议特定的数据翻译为消息对象，反之亦然。我们使用一个现有的 TextLine 工厂因为它将为你处理基于文本的消息 (你无须去编写 codec 部分)。
	
	import java.io.IOException;
	import java.net.InetSocketAddress;
	import java.nio.charset.Charset;
	
	import org.apache.mina.core.service.IoAcceptor;
	import org.apache.mina.filter.codec.ProtocolCodecFilter;
	import org.apache.mina.filter.codec.textline.TextLineCodecFactory;
	import org.apache.mina.filter.logging.LoggingFilter;
	import org.apache.mina.transport.socket.nio.NioSocketAcceptor;
	
	public class MinaTimeServer
	{
	    public static void main( String[] args )
	    {
	        IoAcceptor acceptor = new NioSocketAcceptor();
	        acceptor.getFilterChain().addLast( "logger", new LoggingFilter() );
	        acceptor.getFilterChain().addLast( "codec", new ProtocolCodecFilter( new TextLineCodecFactory( Charset.forName( "UTF-8" ))));
	        acceptor.bind( new InetSocketAddress(PORT) );
	    }
	}

接下来，我们将定义用于服务客户端连接和当前时间的请求的处理器。处理器类是一个必须实现 IoHandler 接口的类。对于几乎所有的使用 MINA 的程序，这里都会变成程序的主要工作所在，因为它将服务所有来自客户端的请求。本文我们将扩展 IoHandlerAdapter 类。这个类遵循了[适配器设计模式](http://en.wikipedia.org/wiki/Adapter_pattern)，简化了需要为满足在一个类中传递实现了 IoHandler 接口的需求而要编写的代码量。
	
	import java.net.InetSocketAddress;
	import java.nio.charset.Charset;
	
	import org.apache.mina.core.service.IoAcceptor;
	import org.apache.mina.filter.codec.ProtocolCodecFilter;
	import org.apache.mina.filter.codec.textline.TextLineCodecFactory;
	import org.apache.mina.filter.logging.LoggingFilter;
	import org.apache.mina.transport.socket.nio.NioSocketAcceptor;
	
	public class MinaTimeServer
	{
	    public static void main( String[] args ) throws IOException
	    {
	        IoAcceptor acceptor = new NioSocketAcceptor();
	        acceptor.getFilterChain().addLast( "logger", new LoggingFilter() );
	        acceptor.getFilterChain().addLast( "codec", new ProtocolCodecFilter( new TextLineCodecFactory( Charset.forName( "UTF-8" ))));
	        acceptor.setHandler(  new TimeServerHandler() );
	        acceptor.bind( new InetSocketAddress(PORT) );
	    }
	}

现在我们对 NioSocketAcceptor 中的配置进行添加。这将允许我们为用于接收客户端连接的 socket 进行 socket 特有的设置。

	import java.net.InetSocketAddress;
	import java.nio.charset.Charset;
	
	import org.apache.mina.core.session.IdleStatus;
	import org.apache.mina.core.service.IoAcceptor;
	import org.apache.mina.filter.codec.ProtocolCodecFilter;
	import org.apache.mina.filter.codec.textline.TextLineCodecFactory;
	import org.apache.mina.filter.logging.LoggingFilter;
	import org.apache.mina.transport.socket.nio.NioSocketAcceptor;
	
	public class MinaTimeServer
	{
	    public static void main( String[] args ) throws IOException
	    {
	        IoAcceptor acceptor = new NioSocketAcceptor();
	        acceptor.getFilterChain().addLast( "logger", new LoggingFilter() );
	        acceptor.getFilterChain().addLast( "codec", new ProtocolCodecFilter( new TextLineCodecFactory( Charset.forName( "UTF-8" ))));
	        acceptor.setHandler(  new TimeServerHandler() );
	        acceptor.getSessionConfig().setReadBufferSize( 2048 );
	        acceptor.getSessionConfig().setIdleTime( IdleStatus.BOTH_IDLE, 10 );
	        acceptor.bind( new InetSocketAddress(PORT) );
	    }
	}

MinaTimeServer 类中新加了两行。这些方法设置了 IoHandler，为 session 设置了输入缓冲区大小以及 idle 属性。指定缓冲区大小以通知底层操作系统为传入的数据分配多少空间。第二行指定了什么时候检查空闲 session。在对 setIdleTime 的调用中，第一个参数定义了再断定 session 是否闲置时要检查的行为，第二个参数定义了在 session 被视为空闲之前以毫秒为单位的时间长度内必须发生。

处理器代码如下所示：
	
	import java.util.Date;
	
	import org.apache.mina.core.session.IdleStatus;
	import org.apache.mina.core.service.IoHandlerAdapter;
	import org.apache.mina.core.session.IoSession;
	
	public class TimeServerHandler extends IoHandlerAdapter
	{
	    @Override
	    public void exceptionCaught( IoSession session, Throwable cause ) throws Exception
	    {
	        cause.printStackTrace();
	    }
	    @Override
	    public void messageReceived( IoSession session, Object message ) throws Exception
	    {
	        String str = message.toString();
	        if( str.trim().equalsIgnoreCase("quit") ) {
	            session.close();
	            return;
	        }
	        Date date = new Date();
	        session.write( date.toString() );
	        System.out.println("Message written...");
	    }
	    @Override
	    public void sessionIdle( IoSession session, IdleStatus status ) throws Exception
	    {
	        System.out.println( "IDLE " + session.getIdleCount( status ));
	    }
	}

这个类中所用的方法是为 exceptionCaught、messageReceived 和 sessionIdle。exceptionCaught 应该总是在处理器中进行定义，以处理正常的远程连接过程时抛出的异常。如果这一方法没有定义，可能无法正常报告异常。
        
exceptionCaught 方法将会对错误和 session 关闭进行简单  stack trace 打印。对于更多的程序，这将是常规，除非处理器能够从异常情况下进行恢复。
        
messageReceived 方法会从客户端接收数据并将当前时间回写给客户端。如果接收自客户端的消息是单词 "quit"，那么当前 session 将被关闭。这一方法也会向客户端打印输出当前时间。取决于你所使用的协议编解码器，传递到这一方法的对象 (第二个参数) 会有所不同，就和你传给 session.write(Object) 方法的对象一样。如果你不定义一个协议编码器，你很可能会接收到一个 IoBuffer 对象，而且被要求写出一个 IoBuffer 对象。

一旦 session 保持空闲状态到达 acceptor.getSessionConfig().setIdleTime( IdleStatus.BOTH_IDLE, 10 ); 所定义的时间长度，sessionIdle 方法会被调用。
        
剩下的工作就是定义服务器端将要监听的 socket 地址，并进行启动服务的调用。代码如下所示：
	
	import java.io.IOException;
	import java.net.InetSocketAddress;
	import java.nio.charset.Charset;
	
	import org.apache.mina.core.service.IoAcceptor;
	import org.apache.mina.core.session.IdleStatus;
	import org.apache.mina.filter.codec.ProtocolCodecFilter;
	import org.apache.mina.filter.codec.textline.TextLineCodecFactory;
	import org.apache.mina.filter.logging.LoggingFilter;
	import org.apache.mina.transport.socket.nio.NioSocketAcceptor;
	
	public class MinaTimeServer
	{
	    private static final int PORT = 9123;
	    public static void main( String[] args ) throws IOException
	    {
	        IoAcceptor acceptor = new NioSocketAcceptor();
	        acceptor.getFilterChain().addLast( "logger", new LoggingFilter() );
	        acceptor.getFilterChain().addLast( "codec", new ProtocolCodecFilter( new TextLineCodecFactory( Charset.forName( "UTF-8" ))));
	        acceptor.setHandler( new TimeServerHandler() );
	        acceptor.getSessionConfig().setReadBufferSize( 2048 );
	        acceptor.getSessionConfig().setIdleTime( IdleStatus.BOTH_IDLE, 10 );
	        acceptor.bind( new InetSocketAddress(PORT) );
	    }
	}

## 测试时间服务器
        
现在我们开始对程序进行编译。你编译好程序以后你就可以运行它了 ，你可以测试将会发生什么。测试程序最简单的方法就是启动这个程序，然后对程序进行 telnet：

<table>
<thead>
<tr>
<th>客户端输出</th>
<th>服务端输出</th>
</tr>
</thead>
<tbody>
<tr>
<td>user@myhost:~&gt; telnet 127.0.0.1 9123 <br>Trying 127.0.0.1... <br>Connected to 127.0.0.1. <br>Escape character is '^]'. <br>hello <br>Wed Oct 17 23:23:36 EDT 2007 <br>quit <br>Connection closed by foreign host. <br>user@myhost:~&gt;</td>
<td>MINA Time server started. <br>Message written...</td>
</tr>
</tbody>
</table>

![](http://99btgc01.info/uploads/2015/04/001.jpg)

## 接下来是什么？
        
请访问我们的文档页面以查找更多资源。当然你也可以继续去阅读其他教程。

*译者注：翻译版本的项目源码见 <https://github.com/waylau/apache-mina-2-user-guide-demos> 中的`com.waylau.mina.demo.time`包下*