串行传输
====

使用 MINA 2.0 你可以连接到串行端口，就像你使用 MINA 连接到一个 TCP/IP 端口一样。
        
## 获取 MINA 2.0
        
你可以下载最新构建的版本 (2.x)。

## 先决条件

在使用 Java 程序访问串行端口之前你需要一个本地库 (因你的操作系统不同可能是 .DLL 或者 .so)。MINA 使用的是来自 RXTX.org 的：<ftp://ftp.qbang.org/pub/rxtx/rxtx-2.1-7-bins-r2.zip。>
        
只需要把合适的 .dll 或者 .so 放在你的 JDK/JRE 的 jre/lib/i386/ 目录下，或者使用 -Djava.library.path= argument 来制定你所放置的本地库。

mina-transport-serial 的 jar 不包括在完整发布版本里头。你可以从这里[下载](http://repo1.maven.org/maven2/org/apache/mina/mina-transport-serial/)到它。
        
## 连接到串行端口
        
MINA 所提供的串行通信只有一个 IoConnector，根据点对点通信媒体的性质。
        
这里假定你已经读过了 MINA 指南。
        
你需要一个 SerialConnector 以连接到一个串行端口：

	// create your connector
	IoConnector connector = new SerialConnector()
	connector.setHandler( ... here your buisness logic IoHandler ... );

除了 SocketConnector 之外没啥不同的。
        
现在为连接到我们的串行端口创建一个地址：

	SerialAddress portAddress=new SerialAddress( "/dev/ttyS0", 38400, 8, StopBits.BITS_1, Parity.NONE, FlowControl.NONE );

第一个参数是你的端口标识。对于 Windows 系统的电脑，串行端口被叫做 "COM1"、"COM2" 等等...对于 Linux 和其他 Unix 系统："/dev/ttyS0"、"/dev/ttyS1"、"/dev/ttyUSB0"。
        
其他参数取决于你的设备和通信特性。

* 波特率
* 数据位
* 奇偶性
* 流控制机制
        
这个完成之后，将连接器连接到相应地址：

	ConnectFuture future = connector.connect( portAddress );
	future.await();
	IoSession sessin = future.getSession();

就这些！其他照常，你可以插进你的过滤器和编解码器。更多信息请参考 RS232：<http://en.wikipedia.org/wiki/RS232>。