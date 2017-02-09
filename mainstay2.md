

## transport层重新理解

1. mainstay的transport层，从使用上，就可以看到它做的工作。

        nettyServer.open();
        nettyClient.open();
        Message request = new Message();
        request.setBody(Unpooled.buffer().writeInt(1));
        Future future = nettyClient.send(request);
        Message response = input.get();
        relistIds.add(response.getId());
    
这就是张光辉所说的，抽象的功力吧。

层层封装起来，底层都看不出来使用netty实现的，还夹杂了心跳包，异步处理等。

1. 对netty的隐藏

数据的发送和接收本身是channel操作。所谓bootstrap和Channelchannel都是为了channel可以正常工作。

1. future的实现

        client.send(){
            Future future = writeAndFlush(msg);
            map.put(msg.getId,future)
            return future;
        }

2. 心跳包

夹杂在fixChannelPool，fixChannelPool本身有healthCheck  

## 重新理解下分层

网络七层模型是分层。在数据层面上，transport也有自己传输的数据Message，有一些完成通信所必要的字段。
也就是说，在数据层面上，分层是数据的不断嵌套。这就解决了我一直的困惑，transport层和协议层如何隔离。（其实，这个东西课本上也有，可以代码的角度实际的呈现，尤其是java代码，还是第一次）
从测试代码上也可以看到，没有上层，transport层也可以收发数据，只不过此时收发的
数据没有载体，没有意义。

从这个意义上来说，没有必要将message换成我的FileMessage。因为message本身就是
底层通信的抽象。

transport层暴露messageHandler（server端），send（message）(client端)给上层接口。

在新版的mainstay中，client和server新增了`List<ChannelHandlerFactory> channelHandlerFactories`, 即上层在使用client.send(Message)时，可以`send(ChildMessage)`,childMessage暂存上层参数，知道传输时，在netty中将其转化为bytebuf。也就说，编解码的过程延迟到实际通信时进行（同时利用了netty的高效），编解码的实现暴露给上层。


## 使用future的精髓？nettyChannel的是异步操作，连带着io操作无法立即该出结果，那就

AQS牛逼之处在于，提取了一种抽象？什么呢？`acquire/release + 信号量（原子变量） + 队列`,我们要做的，就是为信号的变化赋予意义。

AbstractFuture{
	Sync<V> sync = new Sync<V>();	持有共享数据，线程安全的set和get
    ExecutionList executionList = new ExecutionList(); set时，执行等待的task
}

AbstractFuture有三个子类settableFuture、ChainedFuture、CombinedFuture

1. settableFuture将Abstract中的setxx方法的protected放开为public
2. CombinedFuture维护了一个future集合
3. chainedFuture，待实际用到了再理解

明白了AbstractFuture的原来你，也就是理解了其使用场景：treadlocal可以单线程跨方法get和set，Future可以安全的跨线程get和set。


transport 提供类似于socket之类的接口，既可以获取server端，也可以获取client端。