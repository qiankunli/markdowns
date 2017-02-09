## service cache 项目理解

service-cache是为了解决cache使用时的一系列问题，固化一些模式的东西，有些碎。有的功能会很常用，有的功能会不常用。

## 先从细处（ServiceObjectCache）着手 

在实际业务中，我们对数据的查询往往是  client ==> redis cache ==> db。

1. 先查缓存，缓存里没有则查询数据库
2. 从数据库拿到后，一方面要返回，一方面加入到缓存中。

实现起来一般是（这段代码，对应service-cache可以参见`RedisServiceObjectCache.get()`方法）

XXXService{
    Object queryXXX(id){
        Object obj = cache.get(id);
        if(null == obj){
            obj = dbService.query();
            if(null != obj){
                cache.set(id,obj);
            }
        }
        return obj;
    }
}

service-cache提供一个ServiceObjectCache这样的抽象，对外所有的查询`ServiceObjectCache.get()`就可以，它帮你封装"有就返回，没有就从数据库里拿，并加入缓存"的操作。

在这里，**这个cache可以是redis，也可以是guava cache，也可以是guava cache和redis做的两级缓存**。

这样做还有一个好处，在上述的service类中，queryXXX虽然线程安全，但多线程可以同时进入方法中，就会存在：一旦数据库中没有的话，所有的线程直奔数据库去了，造成数据库的压力比较大（这种情况在项目刚启动，缓存还没有加载的时候尤其明显）。

多一个`ServiceObjectCache.get()`后，可以**将线程竞争的粒度增大**，我们可以在get方法上加锁（实际实现上，当然可以将锁的粒度减小），如果几个线程访问的都是一个id，那么先只放一个线程去访问，这个线程访问结束，剩下的线程再走“有则返回，无则查db并加载”的逻辑（即所谓的请求合并）。

我们通过查看serviceObjectCache的配置就可以了解其作用。

    <bean id="trackObjectCache" class="com.ximalaya.service.cache.RedisServiceObjectCache">
		<property name="serviceCacheConfig" ref="trackObjectCacheConfig"></property>
	</bean>

    <bean id="trackObjectCacheConfig"
		class="com.ximalaya.service.cache.config.RedisServiceObjectCacheConfig">
		<property name="autoLoad" value="true"></property>
		<property name="version" value="1"></property>
		<property name="localExpireTime" value="86400"></property>
		// redis和cache都支持超时时间的设置
		<property name="expireTime" value="86400"></property>
		<!-- <property name="localSize" value="100000"></property> -->
	    // 缓存哪个类
		<property name="cacheClass" value="com.ximalaya.service.track.model.Track"></property>
		<property name="keyFieldName" value="id"></property>
		// 如果缓存里没有，就使用serviceObjectLoader获取实际数据
		<property name="serviceObjectLoader" ref="trackLoader"></property>
		// 缓存可以做shard
		<property name="shardNodeList">
			<list>
				<ref bean="shard1"></ref>
				<ref bean="shard2"></ref>
				<ref bean="shard3"></ref>
			</list>
		</property>
	</bean>
	

## 再从宏观（ServiceCache）了解下

service-cache是服务层的缓存。

先回顾下数据库访问的情况。web项目数据库的访问最后基本稳定成了“controller，service，dao”三级模式。service封装了数据访问的复杂性（比如一个service方法中有多个dao操作），controller一般不直接操作dao来使用数据。数据操作的动作也比较稳定：CRUD。

有了redis之后，算是可以“跨主机访问内存”了。我们往往将redis操作与db操作结合起来，其基本动作是：“先取redis，没有查db并加入到redis”，如果把直接“redisTemplate.set(xx,xx)”比作redis的“dao”的话，那么我们也需要一个类似“service”的包装，来封装对redis和db的复合操作。

这就涉及到栋栋对ServiceCache的解释

ServiceCache是服务层的缓存，它提供了下列的功能。

- 缓存自动加载，对于调用者透明
- 合并请求，缓存失效，并发量会合并成一个请求去db，防止雪崩
- 多层次缓存，本地缓存-》外部缓存
- 结构化缓存
- 对象级别
- 缓存关联，列表缓存关联对象缓存，返回对象列表
- 支持多种加载方式，同步，异步


主要有以下类

- ServiceCache --> serviceObjectCache,ServiceCollectionCache
- ServiceStuffer --> ServiceObjectStuffer,ServiceListStuffer
- ServiceLoader --> ServiceListLoader,ServiceListLoader

作用介绍

- ServiceCache，业务端缓存操作类
- ServiceObjectLoader: 缓存对象装载器，缓存不存在，装载对象
- ServiceObjectStuffer: 装载对象缓存，对loader进行封装，合并请求，保证缓存的原子性，做相关装载通知等。

MergedServiceObjectStuffer
  
    MergedServiceObjectStuffer{
        public V stuffObject(long shardingKey, long key) {
		    return this.mergedRequestLoader.load(key, Long.valueOf(shardingKey));
	    }
    }
    MergedRequestLoaderImpl{
        MergeRequestLoaderCallback callback
        AtomicReferenceArray loadingHolderArray
        // 主逻辑
        load(key){
            1. 找个一个key对应的loadingholder，没有就创建，并保有key与loadingHolder的映射
            2. 根据key对应的loadingHolder查询这个key的load状态，主要是两种：
                
                a. loadingHolder是刚创建的,那就设置loadingHolder状态，load数据。
                b. 正在load，那就让当前现场阻塞FutureLoadingHolder.waitForValue
                
                如果key已经在cache中了，那根本就走不到MergedServiceObjectStuffer.load
        }
    }
    
loadingHoder是个Loading状态保存，比如某个key当前是否处于正在加载状态。loadingHolderArray负责保存key与loadingHoder的映射，只不过这个映射时通过key ==> index ==> LoadingHolder来实现的。MergeRequestLoaderCallback在主逻辑的各个节点调用不同的方法（比如load前，load后，实际的去数据库拿数据库，也是用callback实现的。）

load主机中，synchronized尽可能细粒度。FutureLoadingHolder就是对SettableFuture方法包装。a线程发起load key的操作，异步操作结果保存在SettableFuture。下一个线程再load key时，要先等SettableFuture load
的结果。

## 分布式缓存

RedisServiceObjectCache本身也是通过redisTemplate从redis中获取的数据的，xunch项目实现了一个ShardRedisTemplate，配置ServiceObjectCache的shard，其实就是配置ShardRedisTemplate的shard。

xm-shard 包中有一些做shard的基本逻辑，比如hash算法，一致性哈希等，可以借鉴。
