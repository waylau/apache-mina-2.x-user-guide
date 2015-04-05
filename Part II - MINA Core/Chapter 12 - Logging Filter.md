日志过滤器
====

## 背景

Apache MINA 体系允许基于 MINA 的应用的开发者使用他们自己的日志系统

### SLF4J

MINA 使用了 Simple Logging Facade for Java(SLF4J)。你可以在这里找到 SLF4J 的信息。这个日志工具允许任意数量的日志系统的实现。你可以使用 log4j、java.util.logging 或者其他日志系统。这个工具的好处在于如果你在以后的开发处理中将 java.util.logging 改为 log4j，你完全不需要修改你的源代码。

#### 选择合适的 JAR 包
		
SLF4J 使用静态绑定。这意味着每个支持的日志框架都有自己的一个 JAR 文件。你可以通过选择调用了你静态选择的日志框架的 JAR 文件使用你喜爱的日志框架。以下是使用特定日志框架所需 JAR 包的列表：

<table>
<thead>
<tr>
<th>日志框架</th>
<th>所需 JAR</th>
</tr>
</thead>
<tbody>
<tr>
<td>Log4J 1.2.x</td>
<td><strong>sl</strong>f4j-api.jar<strong>, </strong>slf4j-log4j12.jar**</td>
</tr>
<tr>
<td>Log4J 1.3.x</td>
<td><strong>slf4j-api.jar</strong>, <strong>slf4j-log4j13.jar</strong></td>
</tr>
<tr>
<td>java.util.logging</td>
<td><tt>slf4j-api.jar<strong>, </strong>slf4j-jdk14.jar**</tt></td>
</tr>
<tr>
<td>Commons Logging</td>
<td><strong>slf4j-api.jar</strong>, <strong>slf4j-jcl.jar</strong></td>
</tr>
</tbody>
</table>

有几件事要记住：

* slf4j-api.jar 是任何实现 JAR 包里都必须的
* 重要 你不可以把多个实现 JAR 包放在 class path (比如 slf4j-log4j12.jar 和 slf4j-jdk14.jar 一起)；这样可能会导致你的应用出现意想不到的行为。
* slf4j-api.jar 和 slf4j-.jar 的版本应该一致。
       
配置正确之后，你就可以继续去配置你具体选择的日志框架了 (比如，修改 log4j.properties)。

#### 覆盖 Jakarta Commons Logging

SLF4J 也提供了一种办法在不修改源代码的情况下将使用 Jakarta Commons Logging 的应用转换为使用 SLF4J。只需要将 commons-loggong jar 文件从 class path 中移除，然后添加 jcl104-over-slf4j.jar 到 class path。

### log4j 示例

本示例我们将使用 log4j 日志系统。创建一个项目，然后将下面代码放进一个叫做 log4j.properties 的文件：

	# 设置根日志级别为 DEBUG ，并且它的 appender 只是 A1.
	log4j.rootLogger=DEBUG, A1
	
	# A1 设置为一个 ConsoleAppender.
	log4j.appender.A1=org.apache.log4j.ConsoleAppender
	
	# A1 使用 PatternLayout.
	log4j.appender.A1.layout=org.apache.log4j.PatternLayout
	log4j.appender.A1.layout.ConversionPattern=%-4r [%t] %-5p %c{1} %x - %m%n

这个文件将被放在我们项目的 src 目录。如果你使用的是 IDE，在你测试你的代码的时候你需要将此配置文件放在 JVM 的 classpath 下。
        
*尽管这里演示了如何建立一个使用日志的 IoAcceptor，记住 SLF4J API 可能会在你的程序中的任何地方被使用，以生成适用于你的需求的恰当的日志信息。*
        
接下来我们建立一个简单的示例服务器以生成一些日志。这里我们采用的是 EchoServer 示例项目并添加日志到类中：
	
	public static void main(String[] args) throws Exception {
	    IoAcceptor acceptor = new SocketAcceptor();
	    DefaultIoFilterChainBuilder chain = acceptor.getFilterChain();
	
		LoggingFilter loggingFilter = new LoggingFilter();
	    chain.addLast("logging", loggingFilter);
	
	    acceptor.setLocalAddress(new InetSocketAddress(PORT));
	    acceptor.setHandler(new EchoProtocolHandler());
	    acceptor.bind();
	
	    System.out.println("Listening on port " + PORT);
	}

正如你所看到的，我们移除了 addLogger 方法并在 EchoServer 示例中添加了两行。通过 LoggingFilter 的引用，你可以为你的处理器中 IoAcceptor 关联到的每个事件类型设置日志级别。为了定义触发日志的 IoHandler 事件并指定所执行的日志级别，LoggingFilter 定义了一个叫做 setLogLevel(IoEventType, LogLevel) 方法。以下是为这一方法的选项：

<table>
<thead>
<tr>
<th>IoEventType</th>
<th>描述</th>
</tr>
</thead>
<tbody>
<tr>
<td>SESSION_CREATED</td>
<td>新 session 创建时调用</td>
</tr>
<tr>
<td>SESSION_OPENED</td>
<td>新 session 打开时调用</td>
</tr>
<tr>
<td>SESSION_CLOSED</td>
<td>session 关闭时调用</td>
</tr>
<tr>
<td>MESSAGE_RECEIVED</td>
<td>数据接收到时调用</td>
</tr>
<tr>
<td>MESSAGE_SENT</td>
<td>消息发送时调用</td>
</tr>
<tr>
<td>SESSION_IDLE</td>
<td>session 闲置事件到时调用</td>
</tr>
<tr>
<td>EXCEPTION_CAUGHT</td>
<td>异常抛出时调用</td>
</tr>
</tbody>
</table>

以下是为日志级别的描述：

<table>
<thead>
<tr>
<th>LogLevel</th>
<th>描述</th>
</tr>
</thead>
<tbody>
<tr>
<td>NONE</td>
<td>这个将导致忽略配置的存在而没有任何日志事件被创建</td>
</tr>
<tr>
<td>TRACE</td>
<td>创建 TRACE 事件</td>
</tr>
<tr>
<td>DEBUG</td>
<td>生成 debug 消息</td>
</tr>
<tr>
<td>INFO</td>
<td>生成消息信息</td>
</tr>
<tr>
<td>WARN</td>
<td>生成警告信息</td>
</tr>
<tr>
<td>ERROR</td>
<td>生成错误信息</td>
</tr>
</tbody>
</table>

使用这些信息，你足以建立一个基本的日志系统了，并可以在此示例基础上进行扩展以为你自己的系统产生日志信息。