## rpc层

抽象的入口是invoker和exporer，分别作为服务提供者和服务使用者。

Invoker ==> AbstractInvoker（iface,url成员） ==> AbstractRemoteInvoker（client成员） ==> thriftInvoker

invoker具备执行invocation的能力，invoker可以是本地的，也可以是远程的。

如果说transport层解决了心跳、netty封装、异步调用封装等问题，那么rpc层：

1. 使用代理方式简化了用户使用transport层的问题，包括将用户model编解码，附着在（封装成）transport层的message
2. 隐藏了路由原则的问题。
3. 跟thrift层的边界在哪里？



这几条，客户端和服务端封装的方式和逻辑不同。

    invoker{
        Result invoke(Invocation){
            Method method = invocation.getMethod();
            method.invoke();
        }
    }

如果是本地调用，invoker就直接执行方法。如果是一个远程调用，invoker就负责根据url拿到远程服务信息，启动client（总得知道远端的ip和端口才启的动）。

远程调用，为什么抽象出一个`Result invoke(Invocation)`呢，method执行要为它准备一个实例。这个实例，如果本地执行，就是方法的实现类。如果是远端执行，就是一个代理类。也就是说，method的执行时一定的，但method所附属的model是不一定的。

## thriftInvoker

mainstay使用自定义的传输层替代了thrift的传输层。

    thriftInvoker{
        启动时，根据url信息拿到远端ip和端口，client.open()
        假设有一个iface，得到一个本地实现类，将所有方法的实现变成send和receive
        mothod.invoke(本地实现类)
    }

    异步模式的本地实现类{
        transport.client = new xxx(url,encoder,decoder)
        Future method(args){
            ThriftMessage thriftMessage = new ThriftMessage(args)
            Future<Message> = transport.client.send(thriftMessage);
            // 用一个future去解封另一个future
            final SettableFuture<String> settableFuture = Futures.newSettableFuture();
            Futures.addCallback(responseFuture, new FutureCallback<Message>() {
                onSucess(){
                    settableFuture.set()
                }
            }
            return settableFuture;
        }
    }

此处这个future的包装在netty中很常用

## route层 如何与 通信层隔离

route层不是和通信层隔离的，route层在rpc层之上。体现在abstractClusterInvoker中。在非集群模式的invoker和exporter中，也就谈不上路由问题。

    interface router {
        List<Invoker> route(List<invoker> ,invocaton)
    }

## extention

Protocol protocol = ExtentionLoader.getExtentionLoader(Protocol.class).getExtention("injvm");

假设有个InjvmProtocol类，通过这种方式，就可以得到InjvmProtol的实例。

transport层抽象出来client和server，通过transport接口起到socket的作用，获取client和server。

rpc层抽象出来invoker和exporer，通过protocol起到xxx作用，获取invoker和exporter

## 分层review

mainstay是分层的，thrift也是分层的，所以层与层之间可以相互替代，只是thrift有很多概念不是了解，mainstay提供了很多工具类去简化调用。netty作为transport层暴露的接口虽然不是很直接，但也算开了眼。

## 待做

mainstay中的那个Codec

## 实现一个自定义协议

就是依着thriftInvoker依葫芦画瓢，invoker，codec等

AbstractClusterInvoker,按照最原始的方式进行invoker调用，先把整个流程走通了再说。

整个封装完毕后，只用于continue与segment server之间的通信。