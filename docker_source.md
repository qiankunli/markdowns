## docker 源码分析

## 环境准备

golang的工程目录，

	gopath/src
		   /bin
		   /pkg
		   
不管是官方下载的库，还是自己的项目，都建议放在src目录下，这样你引用其它项目来，就很省事了。

## 功能分析

从一个传统程序员的角度来看待docker在设计时的一些问题

请求的执行流程

cmd ==> docker client  ==> docker daemon ==> docker engine

1. docker client 将cmd转换成http
2. docker daemon 将http转成一个job
3. docker engine 运行job

docker daemon和engine 分开的一个重大缘由，就是将io与业务处理分开。io线程接收到请求，包装成job后，job交给job池处理。

这是服务端处理的几个基本范式之一。

服务端处理范式

1. tomcat，io线程会处理业务
2. netty，io线程本身分成两类，并分开处理，worker io线程也会处理业务
2. redis，
3. docker

## docker client


一个进程的执行，本身就有输入输出。为什么，以为键盘和屏幕都用不着你显式的设置。

flag ==> docker cmd 函数   ==> 发送http请求

docker新的代码已经分成了很多的package，一些跟业务不太重要的功能，都被封装到package中去处理，比如flag，所以主干代码中的逻辑还是比较清晰的。

## docker daemon

1. 标准输入是文件描述符0。它是命令的输入，缺省是键盘，也可以是文件或其他命令的输出。
2. 标准输出是文件描述符1。它是命令的输出，缺省是屏幕，也可以是文件。
3. 标准错误是文件描述符2。这是命令错误的输出，缺省是屏幕，同样也可以是文件。

每一个系统与用户进行交流的界面称为终端，每一个从此终端开始运行的进程都会依附于这个终端，这个终端就称为这些进程的控制终端，当控制终端被关闭时，相应的进程都会自动关闭。但是守护进程却能够突破这种限制，它从被执行开始运转，直到整个系统关闭时才退出。如果想让某个进程不因为用户或终端或其他地变化而受到影响，那么就必须把这个进程变成一个守护进程。


程序的执行本质上是一个指令序列，后来人们抽象出函数，程序的执行变成了函数序列，后来人们抽象出对象（因为总有数据属于固定的一部分函数）。不过，到对象这，程序的执行不是对象的序列了，还是对象的函数序列。这就是为什么会有“线程安全的对象”和“线程不安全的对象”

我们想一想

	public class Demo{
		public int abc;
	}

跟

	#include<std.h>
	extern int abc
	
有什么本质区别。但换来的不同就是，我们考虑程序设计时，对象的存在，既要要求又是工具，允许我们从更高的层面去考虑问题。

||指令序列|函数序列|
|---|---|---|
||指令序列|团长给每一个士兵下达作战指令|
||函数序列|团长给每一个营长下达作战指令|
||对象|每一个营除了有三个连之外，还有自己的侦察排、可以呼叫火力支援。团长只需下达作战意图就可以。|


从docker源码，有时候又有一种linux内核的感觉

1. 完成这个事儿，需要哪些数据结构，这些结构在启动的时候如何初始化
2. 在数据完整的情况下，做一个事儿的流程是什么。
3. 那么从分析代码的角度看，1. 各个部分是如何串起来的 2. 第一点知晓了之后，就可以愉快的分析各模块自己的事儿了。

我们为什么不适应分析docker、redis等这类代码，因为我们常写的web服务：

1. 不用处理io与业务的关系
2. 数据一般在db里，不用做数据的初始化工作。或者说，我们根本不用想如何组织数据（除了设计数据库表结构之外）


现在每个go文件都有一个init函数，其实跟java spring都可以有一个initializingBean afterProperties一样。

### docker engine为什么要这么做？

从另一个角度看，docker engine本质上是一个http server。正常情况下http请求的分发，使用`github.com/gorilla/mux`包对url和方法绑定就可以。各个业务模块在自己的包里实现自己的业务逻辑。（看来读docker源码，首先也得了解的go web开发的一般套路）

	project
		moudle1
		moudle2
		main.go
		
	main.go{
		mux.HandleFunc("url1",moudle1.func1)
		mux.HandleFunc("url2",moudle2.func1)
	}


docker engine有以下不同

1. docker engine对这个方法的参数和返回值固定下来`type Handler func(*Job) Status`,自己维护了一个map[string]Handler.
2. Handler不是一个方法光秃秃的执行，而是被包装成job的形式，就**好像java中的executor将Runnable包装成FutureTask**，用意是一样的。
3. 为什么要 `run Handler ==> run job.Run()`.因为Job.Run包装了Handler执行的一些环境，比如engine现在的状态允不允许，Handler执行了多少时间。Handler的运行也要传入job，这样Handler可以从job中获取一些环境信息。

归纳起来，docker 这样做的意图就明了了

1. http server,根据请求，要执行一个方法，这个方法要能感知到engine的状态，同时进行一些统计。
2. 但对于其他moudle来说，人家只想单纯的处理自己的业务逻辑，不想管别的事儿。因为除了业务之外的这些需求也比较统一，适合做一层封装。同时，moudle中的业务方法还需要获取环境的一些信息（类似于`github.com/gorilla/mux`的方法中有io.reader和io.writer）
3. 于是engine就搞出来一个job，

		Job{
			Handler
			env...
			func run(){
				before()
				Handler
				after()
			}
		}
		
4. 那为什么engine不维护一个map[string]Job呢，因为Job维护了一些上下文运行信息啊，你初始化的时候写死了，还有什么卵用。


### daemon对象

engine的map[string]Handler,大部分实现docker功能的Handler，都是daemon的方法，这就意味着daemon必须有结构组织各种资源。所以细化下执行路径：

cmd ==> docker engine ==> daemon func  ==> driver func
