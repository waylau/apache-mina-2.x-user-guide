开始的步骤
====

我们将通过运行一个 MINA 包提供的很简单的例子给你演示使用 MINA 是多么简单。
        
想要在你的应用中使用 MINA 的第一件事是要设置环境。我们将描述你需要安装什么，以及如何运行一个 MINA 程序。没啥大不了的，先来体验一下 MINA 吧 ...
        
## 下载

首先，你需要从[下载区](http://mina.apache.org/mina-project/downloads.html)下载到最新 MINA 发布版。建议选择最新版，除非你有非常好的理由不这么做 ...

一般来说，如果你要使用 Maven 来构建你的项目，你甚至不需要下载任何东西，你只需依赖进一个包含了 MINA 库的 repository：也就是说你只需告诉 Maven pom 你需要使用 MINA 包。
        
## 里边是什么

下载完成后，将 tar.gz 或 zip 文件的内容释放到本地磁盘。下载的压缩文件具有以下内容。
        
在 UNIX 系统，输入：

	$ tar xzpf apache-mina-2.0.7-tar.gz

你将会在 apache-mina-2.0.7 目录下得到以下内容：

	 |
	 +- dist
	 +- docs
	 +- lib
	 +- src
	 +- LICENSE.txt
	 +- LICENSE.jzlib.txt
	 +- LICENSE.ognl.txt
	 +- LICENSE.slf4j.txt
	 +- LICENSE.springframework.txt
	 +- NOTICE.txt

## 内容详情

* dist - 包含了 MINA 库代码的 jar 包
* docs - 包含了 API 文档和代码参照
* lib - 包含了使用 MINA 所需要的所有 jar 包
        
除此之外，基目录下还有两个许可和公告文件。

## 运行你的第一个 MINA 程序
        
下载完发布版之后，让我们运行一下发布版附带的第一个 MINA 例子吧。
        
将以下包放进 classpath

* mina-core-2.0.7.jar
* mina-example-2.0.7.jar
* slf4j-api-1.6.6.jar
* slf4j-log4j12-1.6.6.jar
* log4j-1.2.17.jar
        
*日志提示:Log4J 1.2 用户：slf4j-api.jar、slf4j-log4j12.jar 以及 Log4J 1.2.x * Log4J 1.3 用户：slf4j-api.jar、slf4j-log4j13.jar，以及 Log4J 1.3.x * java.util.logging 用户：slf4j-api.jar 和 slf4j-jdk14.jar。重要提示：请确认你使用了匹配于你的日志框架的正确的 slf4j-*.jar。例如，slf4j-log4j12.jar 和 log4j-1.3.x.jar 不能够同时使用，否则会发生故障。如果你不需要一个日志框架，你可以使用没有日志的 slf4j-nop.jar 或者仅有最基本日志的 slf4j-simple.jar。
        
在命令行中输入以下命令：

	$ java org.apache.mina.example.gettingstarted.timeserver.MinaTimeServer

这将启动服务器。现在 telnet 将会看到程序已启动。
        
输入以下命令来 telnet：

	telnet 127.0.0.1 9123

现在我们已经运行了第一个 MINA 程序。请试着运行 MINA 所附带的其他一些例子程序。