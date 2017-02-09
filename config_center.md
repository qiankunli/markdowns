## 配置中心

服务端用配置中心

## 客户端用配置中心

不要写死，一个永恒的话题，这个话题会一直持续下去。而动态性这件事，是移动设备App当下最热门的话题。在PC时代，我们的系统经历了C/S到B/S的转换，终于实现了最大程度的动态化。而在无线时代，移动设备有他独特的属性，B/S模式无法满足无线时代的业务需求，至少当下是这样。那么Native动态化这条路，就还需要我们坚定的走下去，这条路的尽头可能是另一个B/S模式，也可能我们找到了完美的Dynamic Wireless C/S模式。

## 要考虑的几个基本问题

1. 这是一个系统的东西，假设以后会越来越多。那么app第一次初始化时，返回json字符串的方式就不合适了，可能是要返回一个文件，这就需要通过cdn来加速。
2. 增量下发的粒度
	1. 客户端发送请求，服务端返回全量
	2. 客户端汇报自己的数据状态，精确到模块。服务端返回变化后的模块
	3. 客户端汇报自己的数据状态，精确到配置项。服务端返回变化后的配置项，新增，修改，删除
3. 分版本,平台，地区，uid投放
4. 数据模型要能够满足各个业务线的需求
5. 数据加密
6. 是否要实时更新？ 1. app每次启动后拉去； 2. 服务端有更新时发推送，或者和客户端有长连接
7. 请求和响应数据是否进行压缩，比如protobuf


## 一些对比

跟补丁系统的异同，能不能做一点融合工作。是否需要和版本系统打通。

跟补丁系统的不同之处：

1. 因为即便使用了补丁，客户端也不是说不发包了。所以，补丁适配的版本是有上限的，或者说某个版本能用的补丁总是有限的。而配置信息，作为业务逻辑的一部分，只会越弄越多。
2. 补丁是分两阶段下载的，先返回补丁的url，再由客户端决定下载。所以客户端可以做比对工作。
3. 补丁做abtest，或者做条件匹配，就是返回与不返回。而配置信息，则是返回最新值和非最新值。

由此带来的不同： 

1. 服务端返回增量数据还是全量数据。服务端不用替客户端计算哪些补丁要更新，哪些不要，只是根据客户端的数据，匹配所有能下载的补丁就行了。

	配置中心中的配置项没有这个上限，会不停地更新，不停地增加。所以，服务端不敢每次返回所有数据  ==> 返回增量数据 ==> 要做比对   ==> 客户端要传递自身已下载配置的版本信息  ==> 不能太多  ==> 版本信息精确到模块.
	
2. 因为某个版本能用的补丁总是有限的，所以可以做缓存。对于配置中心，每个客户端的情况不一样，所以只能每次都查询。这就要求，数据存储访问延时比较低，用数据库存储就比较麻烦，至少要加一个cache层。

3. 条件的匹配分为基本匹配和可选匹配，补丁的可选匹配是一个链，这个链挂在基本条件上。配置的链挂在 基本条件  + version 上。

因为客户端请求会做签名，如果签名只包含客户端配置中心信息的话，可以<签名，返回值>，做一个无差别缓存。(这个对自定义下发条件就比较苦难)

跟补丁的系统的相同之处

1. 自定义下发的条件，可以共用。

## 客户端要注意的一些事

1. 重新安装app时，不要删掉在本地的config 数据文件
2. 如果删掉了，发一个init请求。

## 问题的解决

key,value 全部为字符串格式

### 数据模型

1. 返回的数据模型
	
		[
			{	
				name:moudle name
				moudle id:xx
				data/map:{
					k1,v1
					k2,v2
				},
				version:xx
			}
		]
		
 	返回moudle id的用意是，moudle id是不会变的，而prop id 在客户端升级的时候，是会变的。

2. 存储模型

### 基本思路

因为不管怎么弄，匹配标准本质上都是到配置项的，而我们因为上文的需求，匹配的粒度只到moudle。**这就导致用户插入的视图和查询的视图是割裂的。**

所有，数据库的存储结构由两种方式：

1. 面向查询。查询时，osType,appVersion等条件匹配的是moudle。插入时，要根据每一个配置项的匹配条件，更新到moudle prop中。
2. 面向插入，那就是存储配置项，及其所有的匹配条件。可以使用ignite来加快查询的速度。

总之，就是插入方便，查询不方便。查询方便，插入不方便。因为查询的次数远大于更新的次数，所以我们选择面向查询。

这样，在后端插入时，就要做很多工作。但用户查询时，根据moudle id 就可以直接查询，加上service cache，效率会很好。

三个基本特征prop,moudle,item，三者之间的关系。

item由moudle组织，item有默认值value，但针对prop可能有特别的值。moudle有默认值version，但针对prop可能有特别的值。


### 具体设计


|app prop|描述模块/配置项的基本下发条件。或者可以理解为，对客户端的基本情况做一个标识|
|---|---|
|id||
|package name||
|osType|ios/andriod|
|deviceType|iphone,ipad.etc|
|appVersion||
|app channel||


| moudle |moudle表|
|---|---|
|id||
|moudle name||
|default version|默认的数据版本|
|detail||


|moudle prop||
|---|---|
|id||
|moudle id||
|prop id||
|data version|数据的版本|


|configitem|配置项|
|---|---|
|id||
|moudle id||
|config name||
|default config value||
|detail||


|config prop|特殊客户端的的config item配置|
|---|---|
|id||
|config item id||
|prop id||
|config value||


|moudle change|记录moudle prop版本之间的变化|
|---|---|
|id||
|moudle prop id|这个可以分成两个字段|
|old version||
|new version||
|change value|增量下发的内容|



|chain item|自定义下发条件，在返回moudle change的change value之前，进行该匹配|
|---|---|
|id||
|moudle change id||
|item name||
|item value||


1. 新增一个config item

	 a. 根据客户端数据拿到moudle id ，prop id，old version
	 b. 插入config item
	 d. 如果config item是全局的，则更改对应moudle prop的default version。否则，检索现有的moudle prop记录，data version加1.否则，新增一个moudle prop记录，data version = default version + 1
	 d. 新增多个moudle change 记录
	 f. 如果自定义下发条件，则遍历所有的moule change 关联的chain item，更新
	
2. 修改

3. 删除

4. 查询
		
 根据客户端数据拿到moudle id ，prop id，old version，找到new version，进而找到匹配的moudle change id，返回change value即可。
	


### app第一次所有的配置如何下载

#### 文件方式

周期性的根据数据库数据，生成一份最新的文件，放到cdn上，可以针对每个版本。

1. 请求配置文件地址（周期性更新）
2. 下载配置文件，作为一个基准
3. 发送check请求，请求增量数据对配置文件进行更新。

#### json

page形式，一页一页下载


## 架构设计

project-backend
project-core
project-portal


## 其它

1. 客户端启动时，先发一个common的（返回有几个模块的配置），然后到某个模块后，再拉模块的。
2. 实时的要发推送
3. 配置项要记住是否存储在本地
4. 配置中心也要准备给服务端使用。


## 其它

学习[不要写死！天猫App的动态化配置中心实践](http://mp.weixin.qq.com/s?__biz=MzA3ODg4MDk0Ng==&mid=402842876&idx=1&sn=e15d596c95bf7d1ed579cfd7e410696a&scene=21%23wechat_redirect)

[天猫App A/B测试实践](http://mp.weixin.qq.com/s?__biz=MzA3ODg4MDk0Ng==&mid=402918999&idx=1&sn=e1dcacc4dd314013a1153ec90ab4bb82&scene=21%23wechat_redirect)