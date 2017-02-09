## 理解mainstay用到的几个背景知识 

1. 通过netty，知道基于java nio的网络通信。作为server端往往是一个类似eventloop线程处理io事件和定时事件，由此可以知道数据通信的基本原理，我们就可以在更高的层面看待业务问题。

2. 通过对zk的了解，我们可以汇报服务端信息，并在客户端发送请求信息前选择服务端。

3. spring的代理模式

4. 还要学习thrift的异步实现原理。thrift本身支持异步操作，我们只需在此基础上包装一下，达成类似redis的pipeline效果。

redis中没有异步操作，pipeline的实现简单说就是：单个请求只发command不接响应，在获取pipeline结果的时候，集中处理接收缓冲区中的数据。

mainstay据猜测，应该是，发送的thrift请求被转换成异步请求，异步返回的结果



    sendThrift                          result list get
        |||                                   |||
    mainstay proxy                 add mainstay result list
        |||                                   |||
    callback=sendThriftAysc                callback
    
mainstay proxy ：choose server by zk,建立callbakc和result list的关系      

## mainstay 组件功能分析

mainstay最开始可能跟thrift关系的比较紧，后来将一些关键部位拆成api，并提供了thrift的实现。

1. mainstay-admin，暂时为空
2. mainstay-cluster，暂时为空
3. mainstay-common，提供一些并发包，已经高层抽象的通用的bean
4. mainstay-config，直接自己定义一些类似于spring中`<bean>`的标签，并提供解析手段
5. mainstay-registry，负责server端数据注册
6. mainstay-rpc

    - mainstay-rpc-api
    - mainstay-rpc-thrift

7. mainstay-transport，Transport Layer。用来定义传输层
    - mainstay-transport-api，类似于thrift中的TTransport，抽闲出数据传输的endpoint（包括client,server），定义通信接口
    - mainstay-transport-netty，使用netty实现上述接口


## 谈一谈基本原理

https://en.wikipedia.org/wiki/Client%E2%80%93server_model

https://en.wikipedia.org/wiki/Transport_layer


## 传输层

    public interface EndPoint {
        boolean open() throws Exception;
        void close();
        void close(int sec);
        boolean isClosed();
        boolean isAvailable();
        URL getUrl();
    }
    //从定义的方法中可以看到，endpoint可以开关，用url来定位
    public interface Client extends EndPoint {
        ListenableFuture send(Object object);
    }
    public interface Server extends EndPoint {}
    public abstract class AbstractClient implements Client {
        URL url;
        InetSocketAddress localAddress;
        InetSocketAddress bindAddress;
        public AbstractClient(URL url) {  this.url = url;}
        public boolean open() throws ExecutionException, InterruptedException { return doOpen();  }
        public void close() { doClose();  }
        public URL getUrl() { return url; }
        protected abstract boolean doOpen() throws ExecutionException, InterruptedException;
        protected abstract void doClose();
    }
    public abstract class AbstractServer implements Server {
        URL url;
        InetSocketAddress localAddress;
        InetSocketAddress bindAddress;
        public AbstractServer(URL url) {  this.url = url; }
        public boolean open() { return doOpen();  }
        public void close() { doClose();  }
        public URL getUrl() { return url; }
        protected abstract boolean doOpen();
        protected abstract void doClose();
    }
    public interface MessageHandler {
        Object handle(Object message);
    }
    public interface Transporter {
        /**
         * Bind a server.
         */
        Server bind(URL url, MessageHandler handler);
        /**
         * Connect to a server.
         */
        Client connect(URL url, MessageHandler handler);
    }
    
**这几个类定义，虽然没有涉及到实质的实现，但比几个技术技巧的意义更加重大。定了Client和Server等Endpoint，它们由Transporter生成，，由MessageHandler处理消息。**

netty的实现有几个可以学习的地方

1. nettyserver 有一个channel manager类，里面可以控制netty server的最大连接数，当然，也可以做连接的判重喽。那个channel manager类是sharalble类型的，因此最后在系统的结束的时候，要调用其close方法，清除掉资源。
2. netty client 有heartbeathandler


## mainstay rpc

报的结构比较简单，包的最外层定义了几个基本的角色，然后每个角色由一个包来实现。把基本的上层实现搞完之后，由thrift完成通讯即可。

1. exepoter `public AbstractExporter(T iFaceImpl, URL url)`将一个iface实现类发布成url服务。

transport和rpc的边界是message handler。将ifaceimpl包装成messagehandler，然后将messagehanlder和url传给server，启动server。

此处，它并没有使用thrift transport层自带的传输，而是自己写了一套。

2. metic，统计各个服务的时间等，使用influxdb存储。这个小组件在很多地方都能用得到。

服务端就是将服务暴露出去，客户端则还有router和lb等逻辑。

## 客户端代码与服务端代码回顾


client : client ==> protocol ==> transport
server : Transport ==> protocol ==> processor ==> Fuction

    TTransport transport = new TSocket("localhost", 1234);
    TProtocol protocol = new TBinaryProtocol(transport);
    RemotePingService.Client client = new RemotePingService.Client(protocol);
    transport.open();
    client.ping(2012);
    transport.close();

client其实就是iface在客户端的实现

    TServerSocket serverTransport = new TServerSocket(1234);
    Test.Processor processor = new Processor(new TestImpl());
    Factory protocolFactory = new TBinaryProtocol.Factory(true, true);
    Args args = new Args(serverTransport);
    args.processor(processor);
    args.protocolFactory(protocolFactory);
    TServer server = new TThreadPoolServer(args);
    server.serve();
    
    
数据流向跟引用的关系很有意思。

客户端是，Client对象（client对象就是客户端的ifaceimpl）依赖protocol。是transport应用传给protocol，因为是protocol的sendMessage驱动transport的senddata(byte[]).


到了服务端，是byte[]先接到，然后变成message，然后执行processor（包装服务端ifaceimpl）

## 如何衡定一个模块的输入与输出

越来越多的，我们开始写一个个模块，每个模块有输入和输出，有开始和关闭。这个模块或许是一个线性代码（小到一个函数），或许是一个拥有多线程运行逻辑的功能模块。我们越来越应该忘掉自己的一个误区，那就是，输入可以是一个变量，也可以是一个执行逻辑。

比如netty，一旦一个netty server开始运行，它就是一个完整的运行逻辑，有自己的线程模型。按照我以前的抽象，我做到了，将处理代码从handler里抽取出来，自己定义一个messageDispatcher，messageDispatcher根据相关字段分发到不同的processor。其实我可以再进一步，将dispatcher和processor完全抽取出来，封装成一个messagehandler，传入nettyserver。

这样，nettyserver就完全可以作为一个数据传输层而存在。messagehandler抽取出一个数据协议层。从代码的运行情况看，数据协议层和数据传输层不分家，但代码层面上是分开的。

## 小结



client : client ==> protocol ==> transport
server : Transport ==> protocol ==> processor ==> Fuction

ThriftExporter 对比 最简单的thrift server代码，你会发现本质是一样的。区别在

    TTransport transport = xx;
    TProcessor processor = new Processor(new IfaceImpl());
    TServer server = xx;

ThriftExporter就是提供了ifaceImpl代理实现，在ifaceImpl真正干活儿前，干一些注册zk等工作。

加一些filter，你没看错，rpc也可以在`response handle(request)`前后加filter，加点日志，加点性能统计的点。客户端和服务端都会用到。

invoker，抽象了一次方法调用，对于客户端是将方法执行包成收发数据，对于服务端主要是包一下ifaceimpl。

mainstay 据ted说，对客户端做的比较多，服务端基本还是沿用thrift的形式（主要是proxy一下，集成zk，日志等）。
