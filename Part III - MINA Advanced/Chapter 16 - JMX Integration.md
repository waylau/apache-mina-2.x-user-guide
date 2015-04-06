JMX 支持
====

Java Management Extensions(JMX，Java 管理扩展)，用于管理和监控 Java 应用。本文将提供一个示例，以演示如何对基于 MINA 的应用集成 JMX。
        
本文旨在帮你将 JMX 技术集成到你的基于 MINA 的应用。在本文中，我们将把 MINA-JMX 相关类集成进图片服务器示例程序。

## 添加 JMX 支持
        
MINA 应用启用 JMX，我们需要执行以下步骤：

* 创建或者获取 MBean 服务器
* 实例化所需的 MBean 类 (IoAcceptor、IoFilter)
* 将 MBean 注册到 MBean 服务器
        
接下来的讨论中我们将跟随 \src\main\java\org\apache\mina\example\imagine\step3\server\ImageServer.java
     
### 创建或者获取 MBean 服务器

	// create a JMX MBean Server server instance
	MBeanServer mBeanServer = ManagementFactory.getPlatformMBeanServer();

这段代码获取 MBean 服务器实例。
        
### MBean 实例化
        
我们为 IoService 创建一个 IoService：

	// create a JMX-aware bean that wraps a MINA IoService object.  In this
	// case, a NioSocketAcceptor. 
	IoServiceMBean acceptorMBean = new IoServiceMBean( acceptor );

这里创建了一个 IoService MBean。它接收到一个将由 JMX 揭示的 acceptor 的实例。
       
同理，我们如法将 IoFilterMBean 等其他自定义 MBean 炮制添加。
       
### 将 MBean 注册到 MBean 服务器

	// create a JMX ObjectName.  This has to be in a specific format.  
	ObjectName acceptorName = new ObjectName( acceptor.getClass().getPackage().getName() +
	        ":type=acceptor,name=" + acceptor.getClass().getSimpleName());
	
	// register the bean on the MBeanServer.  Without this line, no JMX will happen for
	// this acceptor.
	mBeanServer.registerMBean( acceptorMBean, acceptorName );

我们创建了一个 ObjectName，它需要被用作逻辑名以访问 MBean 并且将 MBean 注册到 MBean 服务器。现在我们的应用启用了 JMX。我们来看看它的实际应用。
        
### 启动 Imagine Server
        
如果你使用的是 Java 5 或者更老的版本：

	java -Dcom.sun.management.jmxremote -classpath <CLASSPATH> org.apache.mina.example.imagine.step3.server.ImageServer

如果是使用 Java 6:

	java  -classpath <CLASSPATH> }}{{{}org.apache.mina.example.imagine.step3.server.ImageServer

### 启动 JConsole

执行

	/bin/jconsole

我们将会看到由 MBean 揭示的不同的属性和操作。