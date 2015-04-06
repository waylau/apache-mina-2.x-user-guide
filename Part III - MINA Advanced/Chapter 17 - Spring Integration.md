Spring 集成
====

本文演示了 MINA 应用和 Spring 的整合。我在我的博客中写了这篇文章，后来也把它放在了这里，这里是这篇信息本来就该归类的地方。你可以在 [Integrating Apache MINA with Spring](http://www.ashishpaliwal.com/blog/2008/11/integrating-apache-mina-with-spring/) 找到原始文本。
        
## 应用架构
        
一个标准的 MINA 应用应该具有以下构造：

* 一个 Handler (处理器)
* 两个 Filter (过滤器) - Logging 过滤器和 ProtocolCodec 过滤器
* NioDatagram Socket
        
### 初始化代码
        
我们先看一下代码。简单起见我们忽略掉了负责粘合的相关代码。

	public void initialize() throws IOException {
	
	    // Create an Acceptor
	    NioDatagramAcceptor acceptor = new NioDatagramAcceptor();
	
	    // Add Handler
	    acceptor.setHandler(new ServerHandler());
	
	    acceptor.getFilterChain().addLast("logging",
	                new LoggingFilter());
	    acceptor.getFilterChain().addLast("codec",
	                new ProtocolCodecFilter(new SNMPCodecFactory()));
	
	    // Create Session Configuration
	    DatagramSessionConfig dcfg = acceptor.getSessionConfig();
	        dcfg.setReuseAddress(true);
	        logger.debug("Starting Server......");
	        // Bind and be ready to listen
	        acceptor.bind(new InetSocketAddress(DEFAULT_PORT));
	        logger.debug("Server listening on "+DEFAULT_PORT);
	}

## 集成过程
        
要集成 Spring，我们需要做以下事情：

* 设置 IO 处理器
* 创建过滤器并依次添加到过滤器链
* 创建套接字并设置套接字参数
        
注意：最新的 MINA 发布版本没有像之前的版本那样具有针对 Spring 的包。该包现在命名为 Integration Beans，以使得实现能够支持所有 DI 框架。
        
看一下 Spring xml 文件。请注意我已经将 xml 中的通用部分移除，仅仅放进了集成需要的部分。本文示例是由 MINA 的应用 [聊天示例](http://svn.apache.org/viewvc/mina/mina/branches/2.0/mina-example/src/main/java/org/apache/mina/example/chat/) 衍生而来。请参阅聊天示例的 xml。
        
*译者注：翻译版本的项目源码见 <https://github.com/waylau/apache-mina-2-user-guide-demos> 中的`com.waylau.mina.demo.chat`包下*

现在让我们把事情放在一起。
        
在 Spring context 文件中设置 IO 处理器：
	
	!-- The IoHandler implementation -->
	<bean id="trapHandler" class="com.ashishpaliwal.udp.mina.server.ServerHandler">

创建过滤器链

	<bean id="snmpCodecFilter" class="org.apache.mina.filter.codec.ProtocolCodecFilter">
	  <constructor-arg>
	    <bean class="com.ashishpaliwal.udp.mina.snmp.SNMPCodecFactory" />
	  </constructor-arg>
	</bean>
	
	<bean id="loggingFilter" class="org.apache.mina.filter.logging.LoggingFilter" />
	
	<!-- The filter chain. -->
	<bean id="filterChainBuilder" class="org.apache.mina.core.filterchain.DefaultIoFilterChainBuilder">
	  <property name="filters">
	    <map>
	      <entry key="loggingFilter" value-ref="loggingFilter"/>
	      <entry key="codecFilter" value-ref="snmpCodecFilter"/>
	    </map>
	  </property>
	</bean>

这里，创建了我们的 IoFilter 实例。参考 ProtocolCodec 工厂，我们使用了构造子注入。Logging 过滤器的创建则直截了当。定义了要使用的过滤器的 bean 之后，现在我们来创建用于实现的过滤器链。我们定义了一个 id 为 "FilterChainBuidler" 的 bean 并将定义好的过滤器添加给它。差不多了，现在要做的只是创建套接字和调用绑定了。
        
来完成最后的部分，创建套接字并完成过滤器链：

	<bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
	    <property name="customEditors">
	      <map>
	        <entry key="java.net.SocketAddress">
	          <bean class="org.apache.mina.integration.beans.InetSocketAddressEditor" />
	        </entry>
	      </map>
	    </property>
	</bean>
	
	<!-- The IoAcceptor which binds to port 161 -->
	<bean id="ioAcceptor" class="org.apache.mina.transport.socket.nio.NioDatagramAcceptor" init-method="bind" destroy-method="unbind">
	  <property name="defaultLocalAddress" value=":161" />
	  <property name="handler" ref="trapHandler" />
	  <property name="filterChainBuilder" ref="filterChainBuilder" />
	</bean>

现在我们创建了我们的 ioAcceptor，设置了 IO 处理器和过滤器链。现在我们就可以写一个方法来使用 Spring 读取这个文件并启动我们的应用了。代码如下：

	public void initializeViaSpring() throws Exception {
	    new ClassPathXmlApplicationContext("trapReceiverContext.xml");
	}