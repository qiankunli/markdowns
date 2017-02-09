## docker 目前的实践

和jenkins的整合

网络，无法跨主机访问。

## k8s 目前的实践

网络，采用路由技术，pod和物理机可以直接访问

web项目，采用svc + pod，以规避pod变化的问题。
thrift项目，采用rc + pod

dns域名解析方案

1. http://www.cnblogs.com/openxxs/p/5015734.html#commentform 

	改kube2sky的源码，使其支持hostname的解析

2. dnsmap，svc + pod


和jenkins的整合


## 调度方案的选择

http://www.infoq.com/cn/articles/practice-of-mosos-in-qunar#anch135371
去哪网在mesos上的实践

http://dockone.io/article/1138
swarm,k8s和mesos的对比

综上，看来各家最后都推荐使用mesos，哪怕是从最后找方案方案的角度看，也比较推荐使用mesos

## 网络方案的选择

http://blog.dataman-inc.com/shurenyun-docker-133/

作者做了各方面的对比，最后推荐选择calico。

总的来说，作者不是很推荐使用隧道的方式，因为封包解包对性能损耗比较大。推荐使用路由技术

路由技术的简单实践：http://dockone.io/article/466

pptv 有一套方案两个都没用，是他们自己的，看着也蛮不错的。可以上dockerone上搜索


### calico

[calico docker 应用实例](http://www.cnblogs.com/zhenyuyaodidiao/p/5322782.html)

[洪强宁：宜信的PaaS平台基于Calico的容器网络实践](http://www.infoq.com/cn/articles/ECUG2015-PaaS-Calico)

## docker化的深度

http://dockone.io/article/1620 领科云基于Mesos和Docker的企业级移动应用实践分享

1. web程序的docker化
2. 服务型程序的docker化
2. 大数据服务的docker化


## 对比

k8s，能不能充分利用k8s的特性。

如果整个方案采用路由技术 + pod（thrift，以便充分利用mainstay） + dnsmap，那k8s就退化成了docker swarm的角色了。