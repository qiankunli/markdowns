## 容器技术的一些实践

registry  harbor
k8s			外界如何无感知的通信，还是需要办法的


1.2 新特性系列文章
http://www.dockerinfo.net/1132.html

1. autoscaler 自动采集cpu使用率，进行扩容缩容
2. configmap pod可以从这里拿配置信息（而不是提前保存在pod内app的配置文件中），相当于有个配置中心（但不能动态更新）。
3. namespace，以及基于namespace的权限和资源限额管理
4. deployment 描述你对pod的理想状态，重新创建pod（比如更新image），回滚
5. ingress将请求配置成外网可以访问的url  == host：port
6. daemonset 有时候一些pod要每个node都运行，比如一个日志采集进程。



源码分析和下载

https://github.com/ironcladlou/kubernetes

这个github repo可以k8s和各个addons的源码。

## k8s



Even though Kubernetes provides a lot of functionality, there are always new scenarios（脚本） that would benefit from new features. Application-specific workflows can be streamlined（流线型的） to accelerate（增加） developer velocity. Ad hoc orchestration（结合） that is acceptable initially often requires robust（强健的） automation（自动化） at scale. This is why Kubernetes was also designed to serve as a platform for building an ecosystem of components and tools to make it easier to deploy, scale, and manage applications.
Labels empower users to organize their resources however they please. Annotations enable users to decorate resources with custom information to facilitate their workflows and provide an easy way for management tools to checkpoint state.
Additionally, the Kubernetes control plane is built upon the same APIs that are available to developers and users. Users can write their own controllers, schedulers, etc., if they choose, with their own APIs that can be targeted by a general-purpose command-line tool.
This design has enabled a number of other systems to build atop Kubernetes.

## 应用程序的安装和使用

k8s其实和etcd很像，估计是go语言的风格，可执行文件就是一个文件，配置文件和数据文件的位置在可执行文件启动时作为参数。

1. etcd和docker 可执行文件就一个，k8s有多个，还分master和minion
2. etcd和k8s有专门的xxxctl，docker client和server是一个程序

还有一点要明确说明的是，xxctl并不是对应的服务端绑死的，只是一个Command line tool.配上不同的参数（**或配置文件**）就可以访问不同的后端。

In order for kubectl to find and access the Kubernetes cluster, it needs a kubeconfig file.

1. etcdctl 使用参数
2. kubectl 参数 + 配置文件

一个应用对外展示使用的几个方式

1. command line tool
2. web ui
3. driver jar
4. restful api


1. etcd/docker command-line-tool，restful api
2. k8s command-line-tool，restful api,web ui。其中k8s的restful api有一种提供方式，kubectl 运行一个代理，请求通过 Kubectl proxy借用kubectl的配置访问api server。

## 三大组件

这个烂大街，kubectl就是对三大组建的增删改查操作。

容器有一个天然优势，那就是某些服务可以以容器的形式存在。k8s现在也有这个苗头

**三大组件抽象了一个全新的容器（已经不能称之为容器了）使用方式。docker swarm从某种程度来说，你还是在跟容器打交道，从容器切换到docker swarm上没什么不自然的。而使用k8s，则要求你必须认同它的那一套理念。**

1. 访问容器，以及访问容器所提供服务的方式。比如，一个pod中提供了web服务，那么有至少两种方式可以访问其web服务

	a. 为pod提供public ip
    b. 通过 api server 去proxy。

《k8s权威指南》书中，作者对k8s有着不一样的入门介绍

1. k8s是application-specific，是一个完备的分布式系统支撑平台。何谓application-specific，看第二条。
2. Service是分布式集群架构的核心，**先说service**。一个service

	a. 拥有一个唯一指定的名字
    b. 拥有一个虚拟ip和端口号（通过ip:port的方式提供服务的定位）
    c. 提供某种远程服务的能力
    d. 被映射到提供这种服务能力的：服务进程，容器（运行服务进程）等
    
也就是说，k8s是一个分布式架构解决方案，开发和支撑平台。容器因为其隔离功能，用来承载服务进程。也就说：容器（具体说是docker）是一个重要手段，但不是k8s理念的主角。**k8s最初的立意，不是去做一个容器管理系统，而是一个分布式系统支撑平台。 这是k8s抽象出service概念的重要动机。**

也就是说，三大组件的介绍顺序是：

1. 因为k8s application-specific的理念，有一个基本抽象Service
2. Service被映射到Pod，由Pod提供服务。Pod的抽象也是k8s的一个重要理念。
3. Pod的管理由RC负责

在集群管理方面，k8s和docker swarm的映射如下：

| master | node/minion | 最小管理单元 |
| ------| ------ | ------ |
| docker swarm | docker | container |
| kube-apiserver,controller-manager,scheduler | kubelet,kube-proxy | pod|


    

## 一些概念
  
### Labels and Selectors

在k8s中，我们可以为每个object定义metadata，在yaml或json中进行配置。比如，可以为pod配置一个`project:xdns`来提醒自己这个pod运行的是xdns项目。

Labels are key/value pairs that are attached to objects, such as pods

	kubectl get pods -l environment=production,tier=frontend
    kubectl get pods -l 'environment,environment notin (frontend)'


`environment=production,tier=frontend`就是label selector

Non-identifying information should be recorded using annotations.

	"objects" : [ {
        "apiVersion" : "v1",
        "kind" : "Service",
        "metadata" : {
          "annotations" : {
            "protocol" : "REST",
            "servicepath" : "cxfcdi",
            "descriptionpath" : "cxfcdi/swagger.json",
            "descriptiontype" : "SwaggerJSON"
          },
          
此处，annotations中的key value有必要存储下来，但是难以像label的一样进行filter

k8s提供了三大组件（在新的版本上组件更多），每个组件可以附加标签，kubectl的操作对象就是这几个组件，通过标签进行增删改查。

## 网络

k8s其实和网络实现之间，只是有一个约定，并不是非得用谁。这个约定就是，一个pod一个ip，pod通过ip可以跨主机访问。

所以，你用路由技术也好，用隧道技术也好，对于k8s都无所谓。


## 系统实现

### apiserer

apiserver是k8s系统中所有对象的**增删查改盯**的http/restful式服务端，其中盯是指watch操作。数据最终存储在分布式一致的etcd存储内，apiserver本身是无状态的，提供了这些数据访问的认证鉴权、缓存、api版本适配转换等一系列的功能。

