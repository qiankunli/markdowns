## 源码分析

go语言现在看来，基本是上一水的面向对象设计，每个.go文件写一个结构体和操作该结构体的方法。

首先要从application.go看起，我们知道，操作marathon，要写一个比较复杂的json文件，对一个app的增删改，都是从提交这个json文件着手的（put,delete等），那么application.go即定义该json文件的各个配置对应的struct。这是后面许多操作的基础。

cluster 描述 marathon 集群信息，比如active node、unactive node等


订阅事件是如何实现的？


AddEventsListener，为marathonClient.listener[channel]EventsChannelContext添加数据。



1. Event Stream(后面加的新特性)

    a. Only available in Marathon >= 0.9.0. 

    b. Does not require any special configuration or prerequisites.
    
2. Event Subscriptions

Requires to start a built-in web server accessible by Marathon to connect and push events to. go-marathon 会自己创建一个http server，提供callbackurl的处理函数，在函数中将拿到的事件返回。


