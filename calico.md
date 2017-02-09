## calico,docker和cnm版本的关系

cnm的三个基本抽象sandbox，endpoint，network（负责连通endpoint）

较低的docker版本也可以，因为本质上`docker run --net=none`，容器在创建时只有一个network namespace（即cnm三个基本抽象中的cnm）。我们自己手动创建veth，配置好（即搞好三大抽象中的endpoint）。再使用网桥或其它手段将endpoint连通（即三大抽象中的network部分）。只是这样的话，整个过程比较复杂。当我们干掉容器时，跟容器相关的一些网络配置还存在（即声明周期不一致）



[calico docker 应用实例](http://www.cnblogs.com/zhenyuyaodidiao/p/5322782.html)

[在Docker中使用Calico网络](http://www.itdadao.com/articles/c15a108473p0.html) 


跟随这两篇操作，你就可以发现docker libnetwork存在的意义，即通过`docker network`来规范化网络操作，并使容器相关的网络配置的声明周期与容器的周期一致。

从中我们也可以看到，所谓的network，是指人们将endpoint连通起来的各种手段，可以简单的单机，也可以跨界点通信

## docker


docker在1.10版本之后，可以通过`--config-file`指定所有参数


`sudo usermod -aG docker <your_username>`加入docker分组，以后就不用非得root权限了。

## 安装使用

calico文档 http://docs.projectcalico.org/v1.5/getting-started/docker/tutorials/basic
[在Docker中使用Calico网络](http://www.itdadao.com/articles/c15a108473p0.html) 

有的指令每个节点都执行，有的指令只执行一个。

`calicoctl node --libnetwork` 执行时最好指定本机的ip，因为如果本机网卡比较多的话，calicoctl会选错网卡。

可以指定一个容器的ip，前提是创建网络时，要进行一定的设置。

