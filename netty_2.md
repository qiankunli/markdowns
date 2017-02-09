## netty

http://de.slideshare.net/normanmaurer/netty4  这个ppt写的很精辟

netty is a nio client server framework which enable quick and easy development of network applications such as protocol servers and clients.

features

1. asynchronous,what does this mean?

	a. all i/o operations don not block
	b. get notified once the operation completes
	c. be able to share one thread across many connections
2. simple and unified api
3. high performing
4. listeners and callback support
5. zero-file-copy
6. buffer pooling
7. gathering/scattering,scatter / gather经常用于需要将传输的数据分开处理的场合，例如传输一个由消息头和消息体组成的消息，你可能会将消息体和消息头分散到不同的buffer中，这样你可以方便的处理消息头和消息体。

## byte buf

java nio中有Buffer，除了buffer本身的好处外，就是r/w操作变成了非阻塞,`channel.read(bytebuffer)`。

1. seperate index for reader/writer
2. composite bytebuf
3. direct and heap implementations
4. resizable with max capacity
5. support for reference-counting
6. method-chaining  `bytebuf.writeInt().writeBytes().write..`


## channel handler

your business logic needs a place,and this is where it belongs too

1. inbound vs outbound
2. state vs operation
	 
   a. chanel registered ==> channel active ==> channel xxx
	
   b. channel read ==> channel read complete ==> channel xxx

3. byte or message based

	a. channelread(byteBuf)
	
	b. channelread(Object you define)

4. always called by the assigned eventExecutor
5. type-safe


bio，阻塞在每一个io操作，accept，r/w.send和write都在一个线程中，并且是同步的。

nio，阻塞集中在select，同一个connection r/w操作可以在一个线程中，也可以在多个。但已经不是同步的了。

## channel pipeline

1. holds the channelhandlers for a channel
2. all events will get passed through it 
3. on the fly modification
4. one-to-one relation between channel and channelpipeline

## event loop

1. one to mandy relation between eventloop and channel
2. **process events and hand over work to the channel pipeline**

以我自己的理解，还要加一点内容，process events,listener/callback and hand over work to the channel pipeline.

all the submitted tasks will get executed in the io-thread.

## bootstrap

1. fluent-api
2. graceful shutdown
3. lightweight

到现在，我们再来理解netty demo代码，结构就清晰了。

	serverBootstrap b = new ServerBootStrap();
	// 设置eventloop
	b.group(new NioEventLoopGroup())
	// 设置channel
	.channel(NioServerSocketChannel.class)
	.localAddress(new InetSocketAddress(port))
	// 设置pipeline
	.childHandler(new ChannelInitializer(SocketChannel)(){
		public void initChannel(SocketChannel ch){
			ch.pipeline().addLast(new xxxChannelHandler());
		}
	})
	channelFuture f = b.bind().sync();
	f.channel().closeFuture().sync();
	b.shutdown();
	

此处的channelFuture和closeFuture是future的两个典型例子

1. channelFuture代表了异步操作的结果
2. closeFuture，不是`channel.close()`的返回值，通常我们也不会主动关闭channel，但channel确实可能被connection的另一端关闭，所以这时，closeFuture一般会作为channel的成员，作为channel close的结果，只是这个操作通常不是由我们主动引发的。


bootstrap connnect ==> AbstractChannel connect ==>  channel.eventLoop().execute  ==> pipeline connect ==>
tail(channnel handler) connect ==> ChannelHandlerInvoker.invokeConnect ==> unsafe oonnect ==> nio socket channel doConnect

这个流程分析不一定对，可以参见[Netty5源码分析（四） -- 事件分发模型](http://blog.csdn.net/iter_zc/article/details/39474727)

           
从这个过程能看出什么呢? channel 的io操作不是直接运行的，要绕一下pipeline。

1. io操作,实际还是channel + 与之绑定的unsafe实现的，这一点跟socket很像。
2. 一个channel绑定了一个线程,如果调用channle.io()的是这个线程，则直接执行，否则向这个线程提交任务。ChannelHandlerInvoker，负责这个判断。
3. 要支持state flow。channel.connect ==> channel.handerl().connect ==> channel.pipeline.connect 如果成功 ==> pipeline.channelActive
3. unsafe,之所以取名UnSafe是因为不能从外部线程调用UnSafe接口中的方法，只能在Channel当前相关的I/O线程中调用
4. 前一个能调用下一个,是因为前一个聚合了下一个的操作对象

            


## 和java nio的对比

**重要**

nio和bio的区别是，`socket.read()`如果返回，一定能读到数据。`channel.read(bytebuffer)`不阻塞，但不一定能读到数据，so，nio多了一个selector，知道什么时候read（这样，就跟bio扯平了），channel退化为一个读写数据的操作对象。bio是表白的时候不知道人家的心意，nio是先知道心意，再表的白。


那么java nio和netty的不同呢？那就是在channel怎么干和selector什么时候干两个维度上进行了深度的封装。

||channel 怎么干|selector 什么时候干|启动|
|---|---|---|---|
|java nio|read/write buffer|selector + 线程|一堆代码|
|netty|handler + pipeline；state flow|eventloop|从代码中抽象出参数，聚合成一个bootstap，启动的一堆代码，成了bootstrap的一个方法|

netty直接对channel 封装出了新花样，弄出了pipeline什么的，对selector花样更大，加上线程，直接改成了eventloop。


处理空闲连接是一项常见的任务,Netty 提供了几个 ChannelHandler 实现此目的。
IdleStateHandler, ReadTimeoutHandler, WriteTimeoutHandler.



eventexecutor（group）和eventloop（group）之间的关系有点错综复杂，SingleThreadEventExecutor（thread加一个队列,nioeventloop 的父类singlethreadeventloop的父类），thread run方法的工作就是从队列中取出一个任务并执行。nioeventloop实现了thread的run方法，nioeventloop聚合了selector，run中先selector.select == selectorkey.channel.unsafe.r/w 。完事之后，执行eventexecutor.runAllTasks。

也就是说，nioeventloop上层接口中ScheduledExecutorService定义的submit(runnable)、schedule(runable,time)之类交给eventexecutor实现，跟selector相关的，有eventloop实现。


||主要的作用|次要的作用是实现父接口的方法|
|---|---|---|
|EventExecutorGroup extends ScheduledExecutorService|为了提供eventexecutor，或者说eventexecutor的容器||
|EventExecutor|提供inEventLoop()方法||
|EventLoopGroup extends EventExecutorGroup|为了提供EventLoop|覆盖了EventExecutorGroup的next方法|
|EventLoop|asInvoker||
|SingleThreadEventExecutor| execute all its submitted tasks in a single thread||


EventExecutor  extends EventExecutorGroup extends ScheduledExecutorService extends ExecutorService extends Executor，也就是说，EventExecutor也是一个Executor, EventExecutorGroup 事不过是为了分组罢了。

SingleThreadEventExecutor维护了两个任务队列，taskQueue和scheduledTaskQueue, SingleThreadEventExecutor 确保这两个queue的任务都由一个线程执行。如何确保呢？一个简单的想法是，一个SingleThreadEventExecutor聚合一个thread成员，由这个thread处理所有任务.或者，干脆用java原生的SingleThreadExecutor（**netty为什么没有这么做呢**）。

但，SingleThreadEventExecutor 用传入的Executor执行task，SingleThreadEventExecutor的thread引用成员用来记住是哪个线程，赋予这个Executor.thread的任务就是，不停的takeTask并执行。无论是谁，调用SingleThreadEventExecutor.execute()其实就是在addtask。所有的SingleThreadEventExecutor都共享一个Executor，我们通过设定executor，来设定一个EventExecutorGroup使用的线程数。










eventexecutor


## pipeline

filter能够**以声明的方式**插入到http请求响应的处理过程中。

inbound事件通常由io线程触发，outbound事件通常由用户主动发起。

ChannelPipeline的代码相对比较简单，**内部维护了一个ChannelHandler的容器和迭代器**（pipeline模式都是如此），可以方便的进行ChannelHandler的增删改查。


ChannelPipeline
DefaultChannelPipeline
ChannelHandler
ChannelHandlerContext,**Enables a ChannelHandler to interact with its ChannelPipeline and other handlers. ** A handler can notify the next ChannelHandler in the ChannelPipeline,modify the ChannelPipeline it belongs to dynamically.

几个类之间的关系

channelpipeline保有channelhandler的容器，这在java里实现办法可就多了

1. channelpipeline直接保有一个list（底层实现可以是array或者list）
2. 链表实现，Channelpipeline只保有一个header引用（想支持特性更多的话，就得加tail）。只不过这样有一个问题，handler本身要保有一个next引用。如果既想这么做，又想让handler干净点，那就得加一个channelhandlercontext类，替handler保有next引用。

代码如下

    channelpipeline{
        channelhandlercontext header;
    }
    channelhandlercontext{
        channelhandler handler;
        channelhandlercontext next;
        EventExecutor executor;
        @Override
        public ChannelHandlerContext fireChannelActive() {
            final AbstractChannelHandlerContext next = findContextInbound();
            EventExecutor executor = next.executor();
            if (executor.inEventLoop()) {
                next.invokeChannelActive();
            } else {
                executor.execute(new OneTimeTask() {
                    @Override
                    public void run() {
                        next.invokeChannelActive();
                    }
                });
            }
            return this;
        }
        private void invokeChannelActive() {
            try {
                ((ChannelInboundHandler) handler()).channelActive(this);
            } catch (Throwable t) {
                notifyHandlerException(t);
            }
        }
    }


从这就可以看到，Channelhandlercontext不只是替Channelhandler保有下next指针，将pipeline的fireChannelxxx 转化为自己的channelxxx方法。

上次我们提过，将channel与其对应的reactor线程（带有selector成员与while(true){selector.select();...}的线程）剥离，剥离之后，一个重要的问题是：确保该channel.read/write处理是在其约定的reactor线程下（一段代码总在一个线程下执行，那么这段代码就是线程安全的）。办法就是上述代码展示的，每个channel（或channel对应的channelhandler，ChannelhandlerContext）持有其约定reactor线程的引用，每次执行方式时判断下：如果在本线程，则执行，如果不在约定线程，则向约定线程提交本任务。**channelhandler一门心思处理业务数据，channelhandlercontenxt触发事件函数的调用，并保证其在绑定的reactor线程下执行**


## 线程模型问题

我们提到，将channel从work thread抽取出来后，channel和 work thread的交互方式。

1. read pipeline由work thread驱动，work thread 通过select.select()得到selectkey中拿到channel和niosocketchannel（保存在attachment中），就可以调用netty socketchannel的读方法。
2. write pipeline由netty socketchannel直接驱动，但问题是,socketchannel作为入口对象，`socketchanel.write`可能在多个线程中被调用，多个线程同时执行`channel.write`，同样都是目的缓冲区，你写点，我写点，数据就乱套了。解决方法就是，为每个channel绑定一个work thread（一个work thread可以处理多个channel，一个channel却只能被同一个work thread处理）（正好netty socketchannel持有了work thread引用，执行chanel.write时先判断现在是不是在自己绑定的work thread，是，则直接执行；如果不是，则向work thread提交一个任务，work thread在合适的时机处理（这就需要work thread有接受任务队列的能力）。）


读事件的处理过程，触发unsafe.read, ==>  pipeline.fireChannelReadComplete() ==> head(channelhandlercontext).fireChannelReadComplete

            if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                unsafe.read();
                if (!ch.isOpen()) {
                    // Connection already closed - no need to handle write.
                    return;
                }
            }
            
write分为两条线：worker thread在可写的时候，调用unsafe.forFlush，将写缓冲区数据发出。第二条线，用户ctx.write的时候，一直运行到headContext.write ==> unsafe.write()，将数据加入到写缓冲区中。

unsafe.forceFlush() == AbstractUnsafe.flush0() ==> doWrite(outboundBuffer);

AbstractUnsafe{
	ChannelOutboundBuffer outboundBuffer	// 写缓冲区
    write(msg)		将数据加入到outboundBuffer中
	dowrite()	// 实际的发送数据
}

            if ((readyOps & SelectionKey.OP_WRITE) != 0) {
                // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
                ch.unsafe().forceFlush();
            } 
            
DefaultChannlePipeline有一个HeadContext和TailContext，是默认的pipeline的头和尾，outbound事件会从tail outbound context开始，一直到headcontenxt。  

		@Override
        public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
            unsafe.write(msg, promise);
        }   
        
## 小结



把书读薄

1. 把死记硬背的细节转化为逻辑上可以理解的东西
2. 很多知识都可以公用，必须此处的链式模式，如果以前知道链式模式的实现，那么理解这块就会很简单。