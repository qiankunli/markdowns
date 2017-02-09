## 源码分析

这个软件的就是从marathon中拿到app的各个实例的ip和port，然后通过go的模板文件，生成新的nginx.conf，触发nginx来刷新

新的nginx.conf会在nixy.toml 相关的目录下生成，是一个隐藏的临时文件。通过观察这个临时文件的生成内容，再对一对模板文件，可以自己做一些修改。域名的访问等，主要就是观察这个nginx.conf临时文件。


nixy本身提供http server，可以通过一些api查看nixy的一些信息。


nixy会监听event，但是不区分具体的事件，**只要有event，就reload数据**。它在调用reload命令时，通过apps拿到所有信息，生成conf，然后将新的nginx.conf文件与上一次使用的nginx.conf比对，如果不一致，才实际的触发刷新。

由此，nginx动态发现marathon后端实例的变化，有三个方案：

1. 使用marathon实现的lb（可以学习其源代码实现）
2. 生成新的nginx文件，reload nginx
3. 使用nginx etcd等插件，将新的app实例信息写入到etcd中。



## vendor directory

`https://blog.gopheracademy.com/advent-2015/vendor-folder/`

With the release of Go 1.5, there is a new way the go tool can discover go packages. However in Go 1.6 this method will be on for everyone and other tools will have support for it as well. This new package discovery method is the vendor folder.

以前一个项目引代码，代码地址都要配在`$GOPATH`中，现在不用了。

这样解决了一个问题，比如从github上拉了一段代码，依赖的库比较多，自己还得一个个`go get`，现在直接拉下来代码就能用了。


如何使用`https://ipfans.github.io/2016/01/golang-vendor/`

