IoBuffer
====

MINA 应用所用的一个字节缓存。

它用于替代 [ByteBuffer](http://java.sun.com/j2se/1.5.0/docs/api/java/nio/ByteBuffer.html)。MINA 不直接使用 NIO 的 ByteBuffer 有两个原因：

* ByteBuffer 没有提供有用的 getter 和 putter 方法，比如 fill、get/putString 以及 get/putAsciiInt()
* 由于 ByteBuffer 的固定容量的特性，很难写入可变长度的数据
        
*这点将会在 MINA 3 中有所变动。MINA 在 nio ByteBuffer 之上做了自己的封装的主要原因是要获取可以扩展的缓存。这是一个非常糟糕的决定。缓存只是缓存：一个用以保存在其被使用之前的临时数据的临时空间。有很多现有解决方案，比如定义一个依赖于 NIO ByteBuffer 列表的包装器，只需要把现有缓存拷贝到一个更大的缓存中区，因为我们只是需要扩展缓存的容量。*

*也可能使用 InputStream 取代贯穿一系列过滤器字节缓存更合适，因为它并不包含任何存储的数据的性质：可以是一个字节数组、字符串、消息...*

*最后，并非不重要的，当前实现违背了一个目标：零拷贝策略 (比如，一旦我们从套接字中读取了数据，我们想在稍后的过程中避免再次拷贝)。既然我们使用可扩展字节缓存，如果我们需要管理大的消息我们就必须得拷贝这些数据。假定 MINA ByteBuffer 只是 NIO ByteBuffer 之上的一个包装者，这在使用直接缓存的时候就是一个真正的问题了。*

## IoBuffer 操作

### 分配一个新缓存

IoBuffer 是个抽象类，因此不能够直接被实例化。要分配 IoBuffer，我们需要使用两个 allocate() 方法中的其中一个。

	// Allocates a new buffer with a specific size, defining its type (direct or heap)
	public static IoBuffer allocate(int capacity, boolean direct)
	
	// Allocates a new buffer with a specific size
	public static IoBuffer allocate(int capacity)

allocate() 方法具有一个或两个参数。第一种方式具有两个参数：

* capacity - 缓存的容量
* direct - 缓存的类型。true 的话获得直接缓存，false 的话获取堆缓存
       
默认的缓存分配由 [SimpleBufferAllocator](http://mina.apache.org/mina-project/xref/org/apache/mina/core/buffer/SimpleBufferAllocator.html) 进行处理。
        
作为一个选择，也可以使用以下形式：

	// Allocates heap buffer by default.
	IoBuffer.setUseDirectBuffer(false);
	// A new heap buffer is returned.
	IoBuffer buf = IoBuffer.allocate(1024);

当使用第二种形式的时候，别忘了先设置以下默认的缓存类型，否则你将被默认获得堆缓存。
        
## 创建自动扩展缓存
        
使用 Java NIO API 创建自动扩展缓存不是一件容易的事情，因为其缓存大小是固定的。有一个缓冲区，可以根据需要自动扩展是网络应用程序的一大亮点。为解决这个，IoBuffer 引入了 autoExpand 属性。它可以自动扩大容量和限值。
        
我们看一下如何创建一个自动扩展的缓存：

	IoBuffer buffer = IoBuffer.allocate(8);
	buffer.setAutoExpand(true);
	buffer.putString("12345678", encoder);
	// Add more to this buffer
	buffer.put((byte)10);

潜在的 ByteBuffer 会被 IoBuffer 在幕后进行分配，如果上面例子中的编码信息大于 8 个字节的话。其容量将会加倍，而且其上限将会增加到字符串最后写入的位置。这行为很像 StringBuffer 类的工作方式。
        
这种机制很可能要在 MINA 3.0 中被移除，因为它其实并非最好的处理增加缓存大小的方法。它可能会被其它方案代替，像隐藏了一个列表的 InputStream，或者一个装有一系列固定容量的 ByteBuffer 的数组。

## 创建自动收缩的缓存
        
还有一些机制释放缓存中多余分配的字节，以保护内存。IoBuffer 提供了 autoShrink 属性以满足这一需要。如果打开 autoShrink，当 compact() 被调用时 IoBuffer 将缓存分为两半，只有 1/4 或更少的当前容量正在被使用。要手工减少缓存的话使用 shrink() 方法。
        
可以用例子对此进行验证：

	IoBuffer buffer = IoBuffer.allocate(16);
	buffer.setAutoShrink(true);
	buffer.put((byte)1);
	System.out.println("Initial Buffer capacity = "+buffer.capacity());
	buffer.shrink();
	System.out.println("Initial Buffer capacity after shrink = "+buffer.capacity());
	buffer.capacity(32);
	System.out.println("Buffer capacity after incrementing capacity to 32 = "+buffer.capacity());
	buffer.shrink();
	System.out.println("Buffer capacity after shrink= "+buffer.capacity());

我们初始化分配的容量是为 16，并且将 autoShrink 属性设为 true。
        
我们看一下它的输出：

	Initial Buffer capacity = 16
	Initial Buffer capacity after shrink = 16
	Buffer capacity after incrementing capacity to 32 = 32
	Buffer capacity after shrink= 16

现在停下来分析输出：

* 初始化缓存容量是为 16，因为我们使用的这一容量创建了缓存。内部实现使得这成为这个缓存容量的最小值
* 调用 shrink() 之后，容量保持 16，因为实际容量永不会比最小容量更小
* 当扩充容量为 32 之后，容量变为 32
* 调用 shrink()，容量被缩减为 16，从而消除了额外的存储

*再次声明，这种机制是默认的，不需要显式告诉缓存它能够缩减。*

## 缓存分配
        
IoBufferAllocater 负责分配并管理缓存。要获取堆缓存分配的精确控制，你需要实现 IoBufferAllocater 接口。
        
MINA 具有以下 IoBufferAllocater 实现：

* SimpleBufferAllocator (默认) - 每次创建一个新的缓存
* CachedBufferAllocator - 对扩展中可能会被复用的缓存进行高速缓存
        
*对于最新的 JVM，使用高速缓存的 IoBuffer 不太可能提高性能。*
        
你可以实现你自己的 IoBufferAllocator 并通过调用对  IoBuffer 的 setAllocator() 来做同样的事情。