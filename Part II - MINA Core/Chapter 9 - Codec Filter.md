编解码器过滤器
====

本文解释一下为什么以及如何使用一个 ProtocolCodecFilter。

## 为什么使用 ProtocolCodecFilter？

* TCP 担保以正确的顺序交付所有数据包。但是没有担保对于在发送端写操作时影响到接收端的读事件。参考 <http://en.wikipedia.org/wiki/IPv4#Fragmentation_and_reassembly> 和 <http://en.wikipedia.org/wiki/Nagle%27s_algorithm>。在 MINA 术语中：如果没有 ProtocolCodecFilter 的话，一个由发送端对 IoSession.write(Object message) 的调用将导致多个接收端的 messageReceived(IoSession session, Object message) 事件，而多个 IoSession.write(Object message) 的调用只会引起唯一的一个 messageReceived 事件。当你的客户端和服务器端运行在同一台主机上 (或者处于同一个本地网) 而你的应用能够应对这种情况时，你可能不会遭遇到这种情况。
* 大多数网络应用需要一种方法以找出当前消息的结束位置和下一条消息的起始位置。
* 你完全可以在你的 IoHandler 实现所有的这些逻辑，但是添加一个 ProtocolCodecFilter 可以让你的代码更加清晰并容易维护。
* 它将你的协议逻辑从你的业务逻辑中 (IoHandler) 剥离开来。
        
## 如何使用？
        
你的应用接收到的基本上是一串字节，你需要将这些字节转换为消息 (高层对象)。
        
有三种常见的技术用来将字节流分割为消息：

* 使用固定长度的字节
* 使用固定长度的报头来指示出报体的长度
* 使用定界符。例如，许多基于文本的协议在每条消息 (<http://www.faqs.org/rfcs/rfc977.html>) 后面紧跟一个新的空行 (或者 CR LF 对)。
        
本文将使用第一种和第二种方法，因为它们很明显更容易实现。然后我们再看一下使用定界符的做法。

## 例子
        
我们将会开发一个 (毫无用处的) 图形字符发生器协议服务器来说明如何实现你自己协议的编解码器 (ProtocolEncoder、ProtocolDecoder 和 ProtocolCodecFactory)。这个协议是很简单的。这是请求消息的布局：

<table>
<thead>
<tr>
<th>4 bytes</th>
<th>4 bytes</th>
<th>4 bytes</th>
</tr>
</thead>
<tbody>
<tr>
<td>width</td>
<td>height</td>
<td>numchars</td>
</tr>
</tbody>
</table>

* width：被请求图片的宽度 (网络字节顺序的一个整数)
* height：被请求图片的高度 (网络字节顺序的一个整数)
* numchars：要生成的字符数 (网络字节顺序的一个整数)

   
服务器以符合请求尺寸的两张图片进行响应，每张图片上打上请求的字符数。这是一个响应消息的布局：

<table>
<thead>
<tr>
<th>4 bytes</th>
<th>variable length body</th>
<th>4 bytes</th>
<th>variable length body</th>
</tr>
</thead>
<tbody>
<tr>
<td>length1</td>
<td>image1</td>
<td>length2</td>
<td>image2</td>
</tr>
</tbody>
</table>

我们需要的用于请求和响应的编码和解码的类的概述：

* ImageRequest：一个表示对我们的图片服务器请求的简单的 POJO
* ImageRequestEncoder：将 ImageRequest 对象编码为协议专用数据 (由客户端使用)
* ImageRequestDecoder：将协议专用数据解码为 ImageRequest 对象 (由服务器端使用)
* ImageResponse：一个表示来自我们图片服务器端的响应的简单的 POJO
* ImageResponseEncoder：服务器端用以对 ImageResponse 对象编码的类
* ImageResponseDecoder：客户端用以对 ImageResponse 对象解码的类
* ImageCodecFactory：这个类负责创建需要的编码器和解码器
        
ImageRequest 类源代码如下：

	public class ImageRequest {
	
	    private int width;
	    private int height;
	    private int numberOfCharacters;
	
	    public ImageRequest(int width, int height, int numberOfCharacters) {
	        this.width = width;
	        this.height = height;
	        this.numberOfCharacters = numberOfCharacters;
	    }
	
	    public int getWidth() {
	        return width;
	    }
	
	    public int getHeight() {
	        return height;
	    }
	
	    public int getNumberOfCharacters() {
	        return numberOfCharacters;
	    }
	}

编码往往比解码容易，因此我们先从 ImageRequestEncoder 开始：

	public class ImageRequestEncoder implements ProtocolEncoder {
	
	    public void encode(IoSession session, Object message, ProtocolEncoderOutput out) throws Exception {
	        ImageRequest request = (ImageRequest) message;
	        IoBuffer buffer = IoBuffer.allocate(12, false);
	        buffer.putInt(request.getWidth());
	        buffer.putInt(request.getHeight());
	        buffer.putInt(request.getNumberOfCharacters());
	        buffer.flip();
	        out.write(buffer);
	    }
	
	    public void dispose(IoSession session) throws Exception {
	        // nothing to dispose
	    }
	}

备注：

* MINA 将会为所有在 IoSession 的写队列里的消息调用编码方法。既然我们的客户端只会写 ImageRequest 对象，我们可以安全地把消息放进 ImageRequest
* 我们在堆空间分配了一个新的 IoBuffer。最好避免使用直接缓存，因为通常堆缓存表现的更好。参考 <http://issues.apache.org/jira/browse/DIRMINA-289>
* 不需要你去释放缓存，MINA 为你代劳了。参考<http://mina.apache.org/mina-project/apidocs/org/apache/mina/core/buffer/IoBuffer.html>
* 你应该在 dispose() 方法中释放所有在为特定会话编码时所获取的资源。如果没有任何事情要处理你可以让你的编码器继承自 ProtocolEncoderAdapter
        
现在我们来看一下解码器。CumulativeProtocolDecoder 绝对对写你自己的编码器有很大帮助：它将把你的解码器决定对连入数据可以做一些事情之前都缓存起来。在这种情况下消息具有固定大小，因此很容易等待所有的数据到齐以后再进行一些操作：

	public class ImageRequestDecoder extends CumulativeProtocolDecoder {
	
	    protected boolean doDecode(IoSession session, IoBuffer in, ProtocolDecoderOutput out) throws Exception {
	        if (in.remaining() >= 12) {
	            int width = in.getInt();
	            int height = in.getInt();
	            int numberOfCharachters = in.getInt();
	            ImageRequest request = new ImageRequest(width, height, numberOfCharachters);
	            out.write(request);
	            return true;
	        } else {
	            return false;
	        }
	    }
	}


备注：

* 每次在对一个完整的消息编码时，你应该将其写进
ProtocolDecoderOutput；这些消息将穿过过滤器链并最终到达你的 IoHandler.messageReceived 方法
* 你不必负责释放 IoBuffer
* 如果没有足够的可用数据用以解码一条消息，只需返回 false
        
响应也是一个非常简单的 POJO：

	public class ImageResponse {
	
	    private BufferedImage image1;
	
	    private BufferedImage image2;
	
	    public ImageResponse(BufferedImage image1, BufferedImage image2) {
	        this.image1 = image1;
	        this.image2 = image2;
	    }
	
	    public BufferedImage getImage1() {
	        return image1;
	    }
	
	    public BufferedImage getImage2() {
	        return image2;
	    }
	}

响应的编码也很简单：

	public class ImageResponseEncoder extends ProtocolEncoderAdapter {
	
	    public void encode(IoSession session, Object message, ProtocolEncoderOutput out) throws Exception {
	        ImageResponse imageResponse = (ImageResponse) message;
	        byte[] bytes1 = getBytes(imageResponse.getImage1());
	        byte[] bytes2 = getBytes(imageResponse.getImage2());
	        int capacity = bytes1.length + bytes2.length + 8;
	        IoBuffer buffer = IoBuffer.allocate(capacity, false);
	        buffer.setAutoExpand(true);
	        buffer.putInt(bytes1.length);
	        buffer.put(bytes1);
	        buffer.putInt(bytes2.length);
	        buffer.put(bytes2);
	        buffer.flip();
	        out.write(buffer);
	    }
	
	    private byte[] getBytes(BufferedImage image) throws IOException {
	        ByteArrayOutputStream baos = new ByteArrayOutputStream();
	        ImageIO.write(image, "PNG", baos);
	        return baos.toByteArray();
	    }
	}

备注：

当无法提前预测 IoBuffer 的长度时，你可以通过调用 buffer.setAutoExpand(true); 使用一个自动扩展缓存
        
现在我们来看一下响应的解码：
	
	public class ImageResponseDecoder extends CumulativeProtocolDecoder {
	
	    private static final String DECODER_STATE_KEY = ImageResponseDecoder.class.getName() + ".STATE";
	
	    public static final int MAX_IMAGE_SIZE = 5 * 1024 * 1024;
	
	    private static class DecoderState {
	        BufferedImage image1;
	    }
	
	    protected boolean doDecode(IoSession session, IoBuffer in, ProtocolDecoderOutput out) throws Exception {
	        DecoderState decoderState = (DecoderState) session.getAttribute(DECODER_STATE_KEY);
	        if (decoderState == null) {
	            decoderState = new DecoderState();
	            session.setAttribute(DECODER_STATE_KEY, decoderState);
	        }
	        if (decoderState.image1 == null) {
	            // try to read first image
	            if (in.prefixedDataAvailable(4, MAX_IMAGE_SIZE)) {
	                decoderState.image1 = readImage(in);
	            } else {
	                // not enough data available to read first image
	                return false;
	            }
	        }
	        if (decoderState.image1 != null) {
	            // try to read second image
	            if (in.prefixedDataAvailable(4, MAX_IMAGE_SIZE)) {
	                BufferedImage image2 = readImage(in);
	                ImageResponse imageResponse = new ImageResponse(decoderState.image1, image2);
	                out.write(imageResponse);
	                decoderState.image1 = null;
	                return true;
	            } else {
	                // not enough data available to read second image
	                return false;
	            }
	        }
	        return false;
	    }
	
	    private BufferedImage readImage(IoBuffer in) throws IOException {
	        int length = in.getInt();
	        byte[] bytes = new byte[length];
	        in.get(bytes);
	        ByteArrayInputStream bais = new ByteArrayInputStream(bytes);
	        return ImageIO.read(bais);
	    }
	}

备注：

* 我们将解码过程的状态保存为一个会话属性。也可以将这一状态保存在解码器对象自身中，但这么干有一些缺点：         
	* 每个 IoSession 需要自己的解码器实例
    * MINA 能够确保不会有多个线程同时为同一个 IoSession 执行 decode() 方法，但这并不能担保执行这个方法的总述同一个线程。假设第一块数据有线程 1 处理，线程 1 认为还不能解码，当下一块数据到达时，它可能会被另一个线程处理。为了避免可见性问题，你必须对这一解码状态进行适当加锁 (IoSession 的属性保存在一个 ConcurrentHashMap中，因此它们对于其他线程是自动可见的)
    * 在邮件列表中的一个讨论得出了这一结论：关于是把状态保存在 IoSession 还是解码器实例中的选择是一个很有趣的问题。要确保不会有两个以上的线程为同一个 IoSession 运行解码方法，MINA 需要做某种形式的同步 => 这一同步也可以保证你不会遇到上面描述的可见性问题。(感谢 Adam Fisk 指出这一点) 参考 <http://www.nabble.com/Tutorial-on-ProtocolCodecFilter,-state-and-threads-t3965413.html>
* IoBuffer.prefixedDataAvailable() 在你的协议事宜一个长度前缀时很是便利；它支持 1、2 或 4 个字节的前缀
* 在你解码一个响应的时候别忘了复位解码状态 (将该会话属性删除是解决这种问题的另一种方法)
        
如果响应只有单一的一个图片，我们就无需保存解码状态了：

	protected boolean doDecode(IoSession session, IoBuffer in, ProtocolDecoderOutput out) throws Exception {
	    if (in.prefixedDataAvailable(4)) {
	        int length = in.getInt();
	        byte[] bytes = new byte[length];
	        in.get(bytes);
	        ByteArrayInputStream bais = new ByteArrayInputStream(bytes);
	        BufferedImage image = ImageIO.read(bais);
	        out.write(image);
	        return true;
	    } else {
	        return false;
	    }
	}

现在我们把它们都组合在一起：
	
	public class ImageCodecFactory implements ProtocolCodecFactory {
	    private ProtocolEncoder encoder;
	    private ProtocolDecoder decoder;
	
	    public ImageCodecFactory(boolean client) {
	        if (client) {
	            encoder = new ImageRequestEncoder();
	            decoder = new ImageResponseDecoder();
	        } else {
	            encoder = new ImageResponseEncoder();
	            decoder = new ImageRequestDecoder();
	        }
	    }
	
	    public ProtocolEncoder getEncoder(IoSession ioSession) throws Exception {
	        return encoder;
	    }
	
	    public ProtocolDecoder getDecoder(IoSession ioSession) throws Exception {
	        return decoder;
	    }
	}

备注：

* 对于每个新会话，MINA 将会请求 ImageCodecFactory 以获取一个编码器或者一个解码器
* 因为我们的编码器和解码器没有保存会话状态，因此让所有会话共享一个单一实例是安全的
        
这里是服务器端对 ProtocolCodecFactory 的使用：

	public class ImageServer {
	    public static final int PORT = 33789;
	
	    public static void main(String[] args) throws IOException {
	        ImageServerIoHandler handler = new ImageServerIoHandler();
	        NioSocketAcceptor acceptor = new NioSocketAcceptor();
	        acceptor.getFilterChain().addLast("protocol", new ProtocolCodecFilter(new ImageCodecFactory(false)));
	        acceptor.setLocalAddress(new InetSocketAddress(PORT));
	        acceptor.setHandler(handler);
	        acceptor.bind();
	        System.out.println("server is listenig at port " + PORT);
	    }
	}

客户端的使用完全一致：

	public class ImageClient extends IoHandlerAdapter {
	    public static final int CONNECT_TIMEOUT = 3000;
	
	    private String host;
	    private int port;
	    private SocketConnector connector;
	    private IoSession session;
	    private ImageListener imageListener;
	
	    public ImageClient(String host, int port, ImageListener imageListener) {
	        this.host = host;
	        this.port = port;
	        this.imageListener = imageListener;
	        connector = new NioSocketConnector();
	        connector.getFilterChain().addLast("codec", new ProtocolCodecFilter(new ImageCodecFactory(true)));
	        connector.setHandler(this);
	    }
	
	    public void messageReceived(IoSession session, Object message) throws Exception {
	        ImageResponse response = (ImageResponse) message;
	        imageListener.onImages(response.getImage1(), response.getImage2());
	    }
	    ...

完整性考虑，现在附加一个服务器端的 IoHandler 代码：

	public class ImageServerIoHandler extends IoHandlerAdapter {
	
	    private final static String characters = "mina rocks abcdefghijklmnopqrstuvwxyz0123456789";
	
	    public static final String INDEX_KEY = ImageServerIoHandler.class.getName() + ".INDEX";
	
	    private Logger logger = LoggerFactory.getLogger(this.getClass());
	
	    public void sessionOpened(IoSession session) throws Exception {
	        session.setAttribute(INDEX_KEY, 0);
	    }
	
	    public void exceptionCaught(IoSession session, Throwable cause) throws Exception {
	        IoSessionLogger sessionLogger = IoSessionLogger.getLogger(session, logger);
	        sessionLogger.warn(cause.getMessage(), cause);
	    }
	
	    public void messageReceived(IoSession session, Object message) throws Exception {
	        ImageRequest request = (ImageRequest) message;
	        String text1 = generateString(session, request.getNumberOfCharacters());
	        String text2 = generateString(session, request.getNumberOfCharacters());
	        BufferedImage image1 = createImage(request, text1);
	        BufferedImage image2 = createImage(request, text2);
	        ImageResponse response = new ImageResponse(image1, image2);
	        session.write(response);
	    }
	
	    private BufferedImage createImage(ImageRequest request, String text) {
	        BufferedImage image = new BufferedImage(request.getWidth(), request.getHeight(), BufferedImage.TYPE_BYTE_INDEXED);
	        Graphics graphics = image.createGraphics();
	        graphics.setColor(Color.YELLOW);
	        graphics.fillRect(0, 0, image.getWidth(), image.getHeight());
	        Font serif = new Font("serif", Font.PLAIN, 30);
	        graphics.setFont(serif);
	        graphics.setColor(Color.BLUE);
	        graphics.drawString(text, 10, 50);
	        return image;
	    }
	
	    private String generateString(IoSession session, int length) {
	        Integer index = (Integer) session.getAttribute(INDEX_KEY);
	        StringBuffer buffer = new StringBuffer(length);
	
	        while (buffer.length() < length) {
	            buffer.append(characters.charAt(index));
	            index++;
	            if (index >= characters.length()) {
	                index = 0;
	            }
	        }
	        session.setAttribute(INDEX_KEY, index);
	        return buffer.toString();
	    }
	}

![](codec-filter.jpeg)

## 结论

关于编码和解码不仅于此。但是我希望本文足以让你开始了。不久的将来我会尝试着添加关于 DemuxingProtocolCodecFactory 的一些介绍。届时我们也将看一下如何使用定界符取代长度前缀的做法


*译者注：翻译版本的项目源码见 <https://github.com/waylau/apache-mina-2-user-guide-demos> 中的`com.waylau.mina.demo.imagine`包下*
