过滤器
====

IoFilter 扮演着很重要角色，它是 MINA 的核心结构之一。它过滤 IoService 和 IoHandler 之间的所有 I/O 事件和请求。如果你有网络应用编程的经验，你完全可以把它当成 Servlet 过滤器的表兄弟。许多开箱即用的过滤器通过使用类似以下的开箱即用过滤器简化横切注入用来提升网络应用的开发速度：

* LoggingFilter 记录所有事件和请求
* ProtocolCodecFilter 将一个连入的 ByteBuffer 转化为消息 POJO，反之亦然
* CompressionFilter 压缩所有数据
* SSLFilter 添加 SSL - TLS - StartTLS 支持
* 更多！
       
本文我们将了解如何为一个真实案例实现一个 IoFilter。通常实现一个 IoFilter 是很简单的，但你也需要知道 MINA 的内部细节。本文将对这些相关内部细节进行解释。
        
## 现有的过滤器
        
我们已经有很多写好的过滤器了。下表列出了所有现有的过滤器，并在他们的用途方面进行简要说明。

<table>
<thead>
<tr>
<th>过滤器</th>
<th>类</th>
<th>描述</th>
</tr>
</thead>
<tbody>
<tr>
<td>Blacklist</td>
<td><a href="http://mina.apache.org/mina-project/xref/org/apache/mina/filter/firewall/BlacklistFilter.html">BlacklistFilter</a></td>
<td>阻止列入黑名单的远程地址的连接</td>
</tr>
<tr>
<td>Buffered Write</td>
<td><a href="http://mina.apache.org/mina-project/xref/org/apache/mina/filter/buffer/BufferedWriteFilter.html">BufferedWriteFilter</a></td>
<td>缓存输出请求，就像 BufferedOutputStream 所做的那样</td>
</tr>
<tr>
<td>Compression</td>
<td><a href="http://mina.apache.org/mina-project/xref/org/apache/mina/filter/compression/CompressionFilter.html">CompressionFilter</a></td>
<td>&nbsp;</td>
</tr>
<tr>
<td>ConnectionThrottle</td>
<td><a href="http://mina.apache.org/mina-project/xref/org/apache/mina/filter/firewall/ConnectionThrottleFilter.html">ConnectionThrottleFilter</a></td>
<td>&nbsp;</td>
</tr>
<tr>
<td>ErrorGenerating</td>
<td><a href="http://mina.apache.org/mina-project/xref/org/apache/mina/filter/errorgenerating/ErrorGeneratingFilter.html">ErrorGeneratingFilter</a></td>
<td>&nbsp;</td>
</tr>
<tr>
<td>Executor</td>
<td><a href="http://mina.apache.org/mina-project/xref/org/apache/mina/filter/executor/ExecutorFilter.html">ExecutorFilter</a></td>
<td>&nbsp;</td>
</tr>
<tr>
<td>FileRegionWrite</td>
<td><a href="http://mina.apache.org/mina-project/xref/org/apache/mina/filter/stream/FileRegionWriteFilter.html">FileRegionWriteFilter</a></td>
<td>&nbsp;</td>
</tr>
<tr>
<td>KeepAlive</td>
<td><a href="http://mina.apache.org/mina-project/xref/org/apache/mina/filter/keepalive/KeepAliveFilter.html">KeepAliveFilter</a></td>
<td>&nbsp;</td>
</tr>
<tr>
<td>Logging</td>
<td><a href="http://mina.apache.org/mina-project/xref/org/apache/mina/filter/logging/LoggingFilter.html">LoggingFilter</a></td>
<td>日志事件消息，比如 MessageReceived、MessageSent、SessionOpened 等等</td>
</tr>
<tr>
<td>MDC Injection</td>
<td><a href="http://mina.apache.org/mina-project/xref/org/apache/mina/filter/logging/MdcInjectionFilter.html">MdcInjectionFilter</a></td>
<td>将关键 IoSession 属性注入 MDC</td>
</tr>
<tr>
<td>Noop</td>
<td><a href="http://mina.apache.org/mina-project/xref/org/apache/mina/filter/util/NoopFilter.html">NoopFilter</a></td>
<td>一个不作任何事情的过滤器。用于测试。</td>
</tr>
<tr>
<td>Profiler</td>
<td><a href="http://mina.apache.org/mina-project/xref/org/apache/mina/filter/statistic/ProfilerTimerFilter.html">ProfilerTimerFilter</a></td>
<td>分析事件消息，比如 MessageReceived、MessageSent、SessionOpened 等等</td>
</tr>
<tr>
<td>ProtocolCodec</td>
<td><a href="http://mina.apache.org/mina-project/xref/org/apache/mina/filter/codec/ProtocolCodecFilter.html">ProtocolCodecFilter</a></td>
<td>负责对消息进行编码和解码的过滤器</td>
</tr>
<tr>
<td>Proxy</td>
<td><a href="http://mina.apache.org/mina-project/xref/org/apache/mina/proxy/filter/ProxyFilter.html">ProxyFilter</a></td>
<td>&nbsp;</td>
</tr>
<tr>
<td>Reference counting</td>
<td><a href="http://mina.apache.org/mina-project/xref/org/apache/mina/filter/util/ReferenceCountingFilter.html">ReferenceCountingFilter</a></td>
<td>跟踪本过滤器的使用次数</td>
</tr>
<tr>
<td>RequestResponse</td>
<td><a href="http://mina.apache.org/mina-project/xref/org/apache/mina/filter/reqres/RequestResponseFilter.html">RequestResponseFilter</a></td>
<td>&nbsp;</td>
</tr>
<tr>
<td>SessionAttributeInitializing</td>
<td><a href="http://mina.apache.org/mina-project/xref/org/apache/mina/filter/util/SessionAttributeInitializingFilter.html">SessionAttributeInitializingFilter</a></td>
<td>&nbsp;</td>
</tr>
<tr>
<td>StreamWrite</td>
<td><a href="http://mina.apache.org/mina-project/xref/org/apache/mina/filter/stream/StreamWriteFilter.html">StreamWriteFilter</a></td>
<td>&nbsp;</td>
</tr>
<tr>
<td>SslFilter</td>
<td><a href="http://mina.apache.org/mina-project/xref/org/apache/mina/filter/ssl/SslFilter.html">SslFilter</a></td>
<td>&nbsp;</td>
</tr>
<tr>
<td>WriteRequest</td>
<td><a href="http://mina.apache.org/mina-project/xref/org/apache/mina/filter/util/WriteRequestFilter.html">WriteRequestFilter</a></td>
<td>&nbsp;</td>
</tr>
</tbody>
</table>


## 选择性重写事件

你可以对 IoAdapter 重写以取代直接实现 IoFilter 的做法。除非重写，否则所有接收到的事件将被直接转发给下一个过滤器。

	public class MyFilter extends IoFilterAdapter {
	    @Override
	    public void sessionOpened(NextFilter nextFilter, IoSession session) throws Exception {
	        // Some logic here...
	        nextFilter.sessionOpened(session);
	        // Some other logic here...
	    }
	}

## 转换写请求
        
如果你要通过 IoSession.write() 对一个连入的写请求进行转换，事情可能会变得相当棘手。例如，假设你的过滤器在一个 HighLevelMessage 对象调用 IoSession.write() 时将 HighLevelMessage 转换为 LowLevelMessage。你可以在你的过滤器的 filterWrite() 方法里添加适当的转换代码并认为一切就这样了。但是你也需要注意 messageSent 事件，因为一个 IoHandler 或者任何之后的过滤器会期望 messageSent() 方法以 HighLevelMessage 作为参数调用，因为让调用者在写 HighLevelMessage 的时候被通知到 HighLevelMessage 已发送是不合理的。因此，如果你的过滤器负责转换时你最好同时实现 filterWrite() 和 messageSent()。
        
另外还要注意的是，你仍旧需要实现类似的机制，即使输入对象和输出对象的类型是一样的，因为 IoSession.write() 的调用者期望它的 messageSent() 处理器方法有一个具体对象。
        
假定你在实现一个将字符串转换为字符数组的过滤器。你的过滤器的 filterWrite() 将会类似于：

	public void filterWrite(NextFilter nextFilter, IoSession session, WriteRequest request) {
	    nextFilter.filterWrite(
	        session, new DefaultWriteRequest(
	                ((String) request.getMessage()).toCharArray(), request.getFuture(), request.getDestination()));
	}

现在我们需要在 messageSent() 中做相反的事情：

	public void messageSent(NextFilter nextFilter, IoSession session, Object message) {
	    nextFilter.messageSent(session, new String((char[]) message));
	}

字符串到字节缓存的转换怎么样？这样我们会更加高效，我们不在需要重建原始消息 (字符串)。但是，这比前面的例子复杂：

	public void filterWrite(NextFilter nextFilter, IoSession session, WriteRequest request) {
	    String m = (String) request.getMessage();
	    ByteBuffer newBuffer = new MyByteBuffer(m, ByteBuffer.wrap(m.getBytes());
	
	    nextFilter.filterWrite(
	            session, new WriteRequest(newBuffer, request.getFuture(), request.getDestination()));
	}
	
	public void messageSent(NextFilter nextFilter, IoSession session, Object message) {
	    if (message instanceof MyByteBuffer) {
	        nextFilter.messageSent(session, ((MyByteBuffer) message).originalValue);
	    } else {
	        nextFilter.messageSent(session, message);
	    }
	}
	
	private static class MyByteBuffer extends ByteBufferProxy {
	    private final Object originalValue;
	    private MyByteBuffer(Object originalValue, ByteBuffer encodedValue) {
	        super(encodedValue);
	        this.originalValue = originalValue;
	    }
	}

如果你使用的是 MINA 2.0，这将跟 1.0 和 1.1 有所不同。同时也参考一下 [CompressionFilter](http://mina.apache.org/mina-project/xref/org/apache/mina/filter/compression/CompressionFilter.html) 和 [RequestResponseFilter](http://mina.apache.org/mina-project/xref/org/apache/mina/filter/reqres/RequestResponseFilter.html)。
        
## 过滤 sessionCreated 事件时须谨慎
        
sessionCreated 是一个特殊事件，它必须在 I/O 处理程序中执行 (参考 线程模型的配置)。决不允许将 sessionCreated 事件转发给其他线程。

	public void sessionCreated(NextFilter nextFilter, IoSession session) throws Exception {
	    // ...
	    nextFilter.sessionCreated(session);
	}
	
	// 不要这么干
	public void sessionCreated(final NextFilter nextFilter, final IoSession session) throws Exception {
	    Executor executor = ...;
	    executor.execute(new Runnable() {
	        nextFilter.sessionCreated(session);
	        });
	    }

## 小心空缓存！
       
在一些案例中 MINA 使用了一个空的缓冲区作为一个内部信号。空缓存有时会成为一个问题，因为它可能会造成各种各样的异常，比如 IndexOutOfBoundsException。这里我们介绍如何避免类似于这种难以预料的情况。
        
ProtocolCodecFilter 使用了一个空缓存 (比如 buf.hasRemaining() = 0) 来标记消息的结束部分。如果你的过滤器放在 ProtocolCodecFilter 之前，如果你的过滤器实现在缓存为空时能抛出一个异常的话，请确认你的过滤器将空缓存转发给了下一个过滤器：

	public void messageSent(NextFilter nextFilter, IoSession session, Object message) {
	    if (message instanceof ByteBuffer && !((ByteBuffer) message).hasRemaining()) {
	        nextFilter.messageSent(nextFilter, session, message);
	        return;
	    }
	    ...
	}
	
	public void filterWrite(NextFilter nextFilter, IoSession session, WriteRequest request) {
	    Object message = request.getMessage();
	    if (message instanceof ByteBuffer && !((ByteBuffer) message).hasRemaining()) {
	        nextFilter.filterWrite(nextFilter, session, request);
	        return;
	    }
	    ...
	}

   
这样的话，我们是否总是要为每个过滤器插入 if 块？幸运的是，不需要。这是处理空缓存的黄金法则：

* 如果你的过滤器及时在缓存为空时也能正常工作，你就完全不需要添加 if 块了
* 如果你的过滤器放在 ProtocolCodecFilter 之后，你也不需要添加 if 块
* 除此之外的话你就需要 if 块了
        
如果你需要加 if 块，记着你不总是需要遵循上面例子所讲的。你可以在任何地方检查缓存是否为空，只要你的过滤器不会抛出异常。
