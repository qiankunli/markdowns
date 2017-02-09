## mesos

Apache Mesos abstracts CPU, memory, storage, and other compute resources away from machines (physical or virtual), enabling fault-tolerant and elastic distributed systems to easily be built and run effectively.

即简化了分布式系统开发，主要就是两个方面：

1. 资源管理，有几个节点，谁挂了
2. 任务调度，任务个数控制，挂后重启等

mesos的思想也是将这两个方面的实现分开，“军种主建，战区主战”



## loadbalance

https://www.nginx.com/blog/service-discovery-in-a-microservices-architecture/

cat xxx.json | jq . > xx.json


## marathon

marathon的操作对象，分为app,group,task,deployment,event stream,event subscription,queue,server info,Miscellaneous

1. 不同的对象可能从不同的版本开始支持

对app的操作，主要是通过更改json文件实现的。比如，我部署了一个example-2 app。然后想新增一个healthcheck特性，就是会通过发送PUT请求，将json文件带过去，实现的。

如何订阅事件？

系统启动的时候，配置一下，那么当发生某个事件事，marathon会向外界发送一个http请求，请求中说明了event信息。我们只要自己写一个http服务接收这个信息就好了。

有一个go—marathon框架，可以借此搞搞go语言。
