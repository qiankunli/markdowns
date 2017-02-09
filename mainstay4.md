invoker 开始执行时，线程模型是怎么回事？如何获取channel，如何操作线程？

client端，发送就是取出一个channel并发送，

nettyClient的基本逻辑

	class nettyclient{
		open(){
			init fixchannel pool
		}
		send(){
			channel = fixchannelpool.acquire();
			Future future = channel.send();
			fixchannelpool(channel)
		}
	} 
	
	
重新理解下fixchannelpool

一个fixchannelpool的初始化需要三个基本要素

1. bootstrap，创建新的channel
2. channelpoolhandler，channel生命周期监听，用于外界介入channelpool的工作
3. channelHealthchecker，负责自定义健康检查


任何一个池要解决以下几个基本问题

1. 创建新的池对象
2. 销毁多余的池对象
3. 存储一定数量的池对象
4. 如何确定池对象是否健康
5. （可选）注入一个观察者，对池对象的增删改做出反应。



注入一个观察者的常用代码逻辑

	operator{
		// 构造方法注入，或者set/add方法注入
		public operator(operatorHandler){
		
		}
		operate(){
			notiryOperatorHandler();
		}
	}
	
## HeartBeated channelpoolhandler

channelpoolhandler用于外界介入channelpool的工作，此处的channelpoolhandler传入的比较有意思



1. 完成了心跳包的功能（此功能的代码被注释掉了）
2. channel创建时，它感知到，为Channel配全pipeline


## 为什么会reject

forwarder数据的处理流向

forwarder ==>  replayingdecoder ===弄一堆小的message==>  channelhandler ==> 线程池执行 messageHandler

块越多，短时间没发完，则越容易reject，解决办法

1. 线程池配多点，比如等待队列配的长一点
2. 客户端channel和线程池中的线程建立映射关系，防止一个channel占用太多，影响了其它客户端的线程
3. 因为forwarder的messagehandler主要是发，所以不用线程池。（推荐这个）

		messageHandler(){
			future = invoke(message);
			// 在此处设置一个超时时间即可
			future.get();
		}
		
4. invoker同步发

否则，默认一个messageHandler的最大耗时是future.get的默认最大时间（messageid和future映射关系，callback的取消时间），forwarder最大处理能力取决于线程池的能力。

中心意思就一个，压力大免不了，但是客户端有平等享受服务的权利，不能早来的客户端把服务能力都占用了。


