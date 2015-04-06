APR 传输
====

## 介绍

[APR (Apache Portable Runtime)](http://apr.apache.org/) 提供了更好的扩展性、性能以及更好的与本地服务器技术的集成。MINA 照常 APR 传输。现在我们将了解如何使用 MINA 进行 APR 传输。我们将为此使用时间服务器的例子。

## 先决条件
        
APR 传输取决于以下组件
        
APR 库 - 从 http://www.apache.org/dist/tomcat/tomcat-connectors/native/ 为你的平台下载并安装适当的库
        
JNI 包装 (tomcat-apr-5.5.23.jar) 这个 jar 附带于在发布版中
        
把本地库放在环境变量中
        
## 使用 APR 传输
        
访问 [时间服务器](http://mina.apache.org/mina-project/xref/org/apache/mina/example/gettingstarted/timeserver/) 例子以获取完整源代码。


*译者注：翻译版本的项目源码见 <https://github.com/waylau/apache-mina-2-user-guide-demos> 中的`com.waylau.mina.demo.time`包下*
        
现在看一下基于 NIO 的时间服务器应用：

	IoAcceptor acceptor = new NioSocketAcceptor();
	
	acceptor.getFilterChain().addLast( "logger", new LoggingFilter() );
	acceptor.getFilterChain().addLast( "codec", new ProtocolCodecFilter( new TextLineCodecFactory( Charset.forName( "UTF-8" ))));
	
	acceptor.setHandler(  new TimeServerHandler() );
	
	acceptor.getSessionConfig().setReadBufferSize( 2048 );
	acceptor.getSessionConfig().setIdleTime( IdleStatus.BOTH_IDLE, 10 );
	
	acceptor.bind( new InetSocketAddress(PORT) );

然后看一下如何使用 APR 传输：
	
	IoAcceptor acceptor = new AprSocketAcceptor();
	
	acceptor.getFilterChain().addLast( "logger", new LoggingFilter() );
	acceptor.getFilterChain().addLast( "codec", new ProtocolCodecFilter( new TextLineCodecFactory( Charset.forName( "UTF-8" ))));
	
	acceptor.setHandler(  new TimeServerHandler() );
	
	acceptor.getSessionConfig().setReadBufferSize( 2048 );
	acceptor.getSessionConfig().setIdleTime( IdleStatus.BOTH_IDLE, 10 );
	
	acceptor.bind( new InetSocketAddress(PORT) );

我们只是将 NioSocketAcceptor 换成了 AprSocketAcceptor，仅仅如此，我们的时间服务器就是使用 APR 传输了。

其他完成过程保持不变。