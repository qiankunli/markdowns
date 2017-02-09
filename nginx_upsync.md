## 使用

主要文档参考


http://blog.csdn.net/yueguanghaidao/article/details/52801043，文中使用consul，使用etcd只需改个字段即可。这个文档有个缺点，我访问的后端实例是tomcat页面，但tomcat页面的css请求等失败，html请求可以成功。


注意

1. 补丁位置一定要在/vgrant下，因为打补丁命令还不懂，一点不一样都不行
2. configure nginx时，要指定pcre的路径
3. 有的时候nginx进程没杀死，还用着原来的配置文件，导致配置文件改了也没用。



可以通过`http://192.168.3.57/upstream_status` 查看一些信息，比如是否正确读到了etcd 节点list。

使用etcd方式，节点的增删改参见`https://github.com/weibocom/nginx-upsync-module#etcd_interface`

我们只要往里面增加`ip:port`（作为key）即可，还可以配置一些参数（可选），比如weight（这个跟nginx的配置有关） 


./configure --add-module=/vagrant/nginx_upstream_check_module  --add-module=/vagrant/nginx-upsync-module --with-pcre=/usr/local/pcre-8.35


curl -X PUT http://192.168.3.57:4001/v2/keys/upstreams/test/192.168.3.57:31524

安装完成后，直接`curl http://ip/`即可以访问后端实例（配置在etcd中），具体还要对应check.conf配置。


整个项目相当于是将nginx的部分配置转移到etcd中，尽可能无损的刷新nginx。