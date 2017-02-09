# netty

netty 是对java nio原生类库的包装，那么首先我们要了解java nio原生的类体系

    public interface Channel extends Closeable {
        public boolean isOpen();
        public void close() throws IOException;
    }

	// 并没有新增方法，只是说明，实现这个接口的类，要支持Interruptible特性。
    public interface InterruptibleChannel
        extends Channel
        public void close() throws IOException;

    }

 A channel that can be asynchronously closed and interrupted. A channel that implements this interface is asynchronously closeable: If a thread is blocked in an I/O operation on an interruptible channel then another thread may invoke the channel's close method.  This will cause the blocked thread to receive an AsynchronousCloseException.

 这就解释了，好多类携带Interruptible的含义。
 
  public abstract class SelectableChannel
    extends AbstractInterruptibleChannel
    implements Channel
{
    public abstract SelectorProvider provider();
    public abstract int validOps();
    public abstract boolean isRegistered();
    public abstract SelectionKey register(Selector sel, int ops, Object att)
        throws ClosedChannelException;
    public final SelectionKey register(Selector sel, int ops)
        throws ClosedChannelException
    {
        return register(sel, ops, null);
    }
    public abstract SelectableChannel configureBlocking(boolean block)
        throws IOException;
    public abstract boolean isBlocking();
    public abstract Object blockingLock();

}


     public abstract class SelectableChannel
        extends AbstractInterruptibleChannel
        implements Channel
    {
        public abstract SelectorProvider provider();
        public abstract int validOps();
        public abstract boolean isRegistered();
        public abstract SelectionKey register(Selector sel, int ops, Object att)
            throws ClosedChannelException;
        public final SelectionKey register(Selector sel, int ops)
            throws ClosedChannelException
        {
            return register(sel, ops, null);
        }
        public abstract SelectableChannel configureBlocking(boolean block)
            throws IOException;
        public abstract boolean isBlocking();
        public abstract Object blockingLock();

    }

  In order to be used with a selector, an instance of this class must first be egistered via the register method.  This method returns a new {@link SelectionKey object that represents the channel's registration with the selector.


SelectorProvider，Service-provider class for selectors and selectable channels.

AbstractSelectableChannel 研读


据此，就可以分析出来selector和channel之间的交互关系。

selector和selectableChannel是多对多的关系，数据库中表示多对多关系，需要一个中间表。面向对象表示多对多关系则需要一个中间对象，SelectionKey。selector和selectableChannel都持有这个selectionkey集合。

可惜selector的select方法是一个抽象类，靠具体的虚拟机提供实现。


# 包装了什么

java nio类库的三个基本组件bytebuffer ,channel,selector, 它们是spi接口，java规范并不提供详细的实现，java只是将这三个组件赤裸裸的提供给你，线程模型由我们自己决定采用，数据协议由我们自己制定并解析。

netty提供的channel，则一旦初始化完毕

ChannelRead()方法总可以实现数据的读取，channel.write实现的数据的发送。bytebuf在编解码器中才会用到，selector则完全隐藏了，线程模型也固定好了。

就好像，我曾经对netty实现的一个封装invoker，提供一个`response invoke(request)`的抽象，完全不管底层的编解码，线程模型，甚至是不是用netty实现的。


netty的bytebuf提供的接口与nio的bytebuffer是一致的，只是功能的增强。而netty的channel则是对nio的channel的重写。**AbstractChannel聚合了所有channel使用到的能力对象，由AbstractChannel提供初始化和统一封装，如果功能和子类强相关，则定义成抽象方法，由子类具体实现。

AbstractChannel{
	Channel parent;
    Unsafe unsafe;
    // 读写操作全部转到pipeline上
    DefaultChannelPipeline pipeline;
	EventLoop eventloop;
    // 保有这么多future，这是要干啥
    SuccessedFuture,ClosedFuture,voidPromise,unsafeVoidPromise
    localAddress,remoteAddress
}

AbstractChannel的抽象，还没有区分bio和nio。那么进行nio的Channel抽象，则肯定有一个SelectableChannel（java.nio.channels也是类似的道理）

channeld写，是自己出发，调用write，进而调用pipeline的write


channel，怎么说呢，我们看一段代码

proteceted void doRegister(){
	selectionKey = javaChannel().register(eventloop.selector(),0,this);
    
}
    
连接和读写操作，还是放在channel类以及unsafe中实现。在eventloop中的run方法中被触发执行。

	private static void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
        final NioUnsafe unsafe = ch.unsafe();
        try {
            int readyOps = k.readyOps();
            if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                unsafe.read();
                if (!ch.isOpen()) {
                    return;
                }
            }
            if ((readyOps & SelectionKey.OP_WRITE) != 0) {
                ch.unsafe().forceFlush();
            }
            if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
                int ops = k.interestOps();
                ops &= ~SelectionKey.OP_CONNECT;
                k.interestOps(ops);
                unsafe.finishConnect();
            }
        } catch (CancelledKeyException ignored) {
            unsafe.close(unsafe.voidPromise());
        }
    }

服务端启动的时候，创建了两个nioeventloopgroup，它们实际上是两个独立的reactor线程池。何谓reactor线程，就是聚合了一个selector。

**netty我们常用它的nio特性，但netty不只是为了nio，往往一些较高层的类的，bio也可以使用。以nio为前缀的子类，才是针对nio的扩展。**

比如eventloop，其一个实现子类SingleThreadEventLoop就维护了一个Queue<Runnable> taskQueue.可以添加任务，还有shutdownhooks，以及对线程关闭的封装（清理queue以及执行shutdownhooks）。

到其实现的子类，nioeventloop，则聚合了以下成员，以及提供selector对应的register等方法。

    class NioEventLoop extends SingleThreadEventLoop{
        Selector selector;
        private SelectedSelectionKeySet selectedKeys;
    }

以nio代码，划分代码的角度，抽取的角度再分析下，一个线程和selector一起被抽出来。

	public class NIOServer {
        public static void main(String[] args) throws IOException {
            Selector selector = Selector.open();
            ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
            serverSocketChannel.configureBlocking(false);
            serverSocketChannel.socket().bind(new InetSocketAddress(8080));
            serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
            while (true) {
                selector.select(1000);
                Set<SelectionKey> selectedKeys = selector.selectedKeys();
                Iterator<SelectionKey> it = selectedKeys.iterator();
                SelectionKey key = null;
                while (it.hasNext()) {
                    key = it.next();
                    it.remove();
                    handleKey(key);
                }
            }
        }
        public static void handleKey(SelectionKey key) throws IOException {
            if (key.isAcceptable()) {
                // Accept the new connection
                ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
                SocketChannel sc = ssc.accept();
                sc.configureBlocking(false);
                // Add the new connection to the selector
                sc.register(key.selector(), SelectionKey.OP_READ | SelectionKey.OP_WRITE);
                System.out.println("accept...");
            } else if (key.isReadable()) {
                SocketChannel sc = (SocketChannel) key.channel();
                ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                // handle buffer
                int count = sc.read(readBuffer);
                if (count > 0) {
                    String receiveText = new String(readBuffer.array(), 0, count);
                    System.out.println("服务器端接受客户端数据--:" + receiveText);
                }
            }
        }
	}
    
所谓的eventloop，就是这个

    while(true){
        selector.select()
        ...
    }
    
笔者曾经进行过简单的抽取，就是将`while(true){..}，handleKey(){...}`抽取到新的线程中，于是

	class NIOServer{
    	ServerSocketChannel ...
        new Worker().start();
    }
    
当然了，大家都提倡acceptable和read/write分开，我们可以


	class NIOServer{
    	ServerSocketChannel ...
        Selector selectror = ...
        new Boss(selector).start();
    	new Worker(selector).start();
    }
    
    
后来发现，boss和worker共享一个selector虽然简单，但是扩展性太低，因此让boss和worker各自有自己的selector，boss thread accept之后得到的socketchannel通过queue传给worker。简单的想法是

	class NIOServer{
    	Queue<SocketChannel> queue = ...
        new Boss(selector).start();
    	new Worker(selector).start();
    }
    
再进一步，可以将queue内置到worker线程中，workerThread除了run方法，还提供addSocketChannel等方法，bossThread保有workerThread的引用。

然后再将Boss和worker线程池化，是不是功德圆满了呢？还没有，nio类库提供给用户的三个基本操作类bytebuffer,channel,selector。我们这种方式，程序的驱动来自boss和worker线程，读取的数据怎么处理（尤其是复杂的处理），我们如何主动地写入呢？最好的办法，还是要将Channel从worker线程中提取出来，channel提出来之后worker线程提供自己内部的selector与channel交互的手段，比如register。

其实channel提出来之后，跟着channel一起出来的代码，比如读写数据的具体逻辑，这样workerThread中的代码可以更简洁。就读写方面，还是很复杂的，因为本质上，还是handlekey的位置知道什么时候读到了数据，什么时候可以写数据，所以我们要对channel作一定的封装。

ChannelFacade{
	channel	// 实际的channel
  	writeBuffer	// 发送缓冲区 
    handleReadData(Buffer){}	// 如何处理读到的数据，由workerThread触发	
    write()					// 对外提供的写数据接口
    doWrite()			// 实际写数据，由workerThread触发
}

这样，我们就以《how tomcat works》的方式，猜想了netty的主要实现思路，当然，netty的实现远较这个复杂，那就是程序的健壮性，特性的丰富性的事了，主要的思路应该是这样的。