# jedis 源码分析

分为三块

简单，pool，shard

## 简单实现

几个基本的model，jedis，client(Connection)，command。这几个model都有对应的Binaryxx，这表示中间会有一些编码的处理操作。比如Jedis，`set(String key,String value)`，对应BinaryJedis就是`set(byte[] key,byte[] value)`。**对于每一个model，功能的拆分，主要通过父子继承关系来分解**


首先redis协议支持的操作，称为Command，反映在代码中，抽象出了一系列的Command接口，负责不同的操作。

- BasicCommands，比如ping，save，bgsave等
- BinaryJedisCommands，负责各种数据结构的set和get
- MultiKeyBinaryCommands，应该是多个key-value的同时设置
- 其它的Command类一般用不着

对于jedis，就有各种了

1. shardedJedis
2. 还有支持主从的


client负责数据连接，command负责业务数据，jedis负责整合两者。command没有XXXCommand实现类，其最终由Client实现

    //jedis
    public String set(String key, String value) {
        this.checkIsInMultiOrPipeline();
        this.client.set(key, value);
        return this.client.getStatusCodeReply();
    }
    // Client
    public void set(String key, String value) {
        this.set(SafeEncoder.encode(key), SafeEncoder.encode(value));
    }
    // BinaryClient
    public void set(byte[] key, byte[] value, byte[] nxxx, byte[] expx, long time) {
        this.sendCommand(Command.SET, new byte[][]{key, value, nxxx, expx, Protocol.toByteArray(time)});
    }
    Connection sendCommand(Command cmd, byte[]... args) {
        try {
            this.connect();
            Protocol.sendCommand(this.outputStream, cmd, args);
            ++this.pipelinedCommands;
            return this;
        }catch(Exception e){...}
    }
    // Protocol，提供了一些额外的write方法，将command变成符合redis协议的二进制数据，并发送
    private static void sendCommand(RedisOutputStream os, byte[] command, byte[]... args){        
        try {
            os.write((byte)42);
            os.writeIntCrLf(args.length + 1);
            os.write((byte)36);
            os.writeIntCrLf(command.length);
            os.write(command);
            os.writeCrLf();
            byte[][] e = args;
            int len$ = args.length;
            for(int i$ = 0; i$ < len$; ++i$) {
                byte[] arg = e[i$];
                os.write((byte)36);
                os.writeIntCrLf(arg.length);
                os.write(arg);
                os.writeCrLf();
            }

        } catch (IOException var7) {
            throw new JedisConnectionException(var7);
        }
    }
    
整个代码看下来，真是太流畅了，类似的client-server工具程序可以借鉴下。

## pool实现

基于common pool2实现，

## shard （pool）实现

ShardedJedis，BinaryShardedJedis,Sharded 父子关系

    // ShardedJedis
    public String set(String key, String value) {
        Jedis j = (Jedis)this.getShard(key);
        return j.set(key, value);
    }
    // Sharded，先在TreeMap中找到对应key所对应的ShardInfo，然后通过ShardInfo在LinkedHashMap中找到对应的Jedis实例。
    public R getShard(byte[] key) {
        return this.resources.get(this.getShardInfo(key));
    }
    
    // hashinfo会描述一个主机信息
    Sharded<R, S extends ShardInfo<R>> {
        private TreeMap<Long, S> nodes;                hash(虚拟shardinfo)与shardinfo的映射
        private final Hashing algo;    // 哈希算法
        private final Map<ShardInfo<R>, R> resources;    // shardInfo与Jedis的映射
    }
        
至于key到虚拟节点的映射，treemap.tailMap(hash(key))，会返回所有大于hash(key)的key-value,选择第一个即可。

公司那种是分表的（访问的都是一个数据库，所以连接池可以共用），则是虚拟节点与tableSuffix的映射存在数据库中，然后根据key计算tableSuffix即可。

## Pipeline

Pipeline ==》MultiKeyPipelineBase ==》 PipelineBase

有好多pipeline接口，都是干啥的（看它们的任务分工，是不是很熟悉）

- BinaryRedisPipeline，提供操作字节数组的接口（基本的数据操作）
- RedisPipeline，基本的数据操作，原始类型（为转成字节数组），这些节本操作的返回值变了。`Response<String> set(String key, String value)`
- BasicRedisPipeline, bgsave,save的操作
- MultiKeyBinaryRedisPipeline、MultiKeyCommandsPipeline，multikey的原始类型和二进制类型操作
- ClusterPipeline，cluster相关的操作


Queable 这个接口，是搞毛的

    Queable {
        private Queue<Response<?>> pipelinedResponses =  new LinkedList<Response<?>>();
    }

将model包装成response，然后提供从list上加入和取出response的接口

Response

    Response<T> {
    	T response = null;
    	JedisDataException exception = null;
    	boolean building = false;
    	boolean built = false;
    	boolean set = false;
    	Builder<T> builder;
    	Object data;
    	Response<?> dependency = null; 
    }
 
response 在队列的那段时间，好像没有实际的数据。


    Jedis jedis = new Jedis(host, port);
    jedis.auth(password);
    Pipeline p = jedis.pipelined();
    p.get("1");
    p.get("2");
    p.get("3");
    List<Object> responses = p.syncAndReturnAll();
    System.out.println(responses.get(0));
    System.out.println(responses.get(1));
    System.out.println(responses.get(2));
    
get操作知识发出了get请求，然后创建一个response对象加入到list队列中。

pipeline的基本原理是：在pipeline中的请求，会直接发出去，加一个response进入list（相当于约定好返回结果存这里）。网络通信嘛，返回的结果本质上是inputstream，get的时候，或许拿到了inputstream，但是不解析。等到syncAndReturnAll的时候集中解析inputstream。因为redis server端是单线程处理的，所以也不用担心get("2")的结果跑在get("1")的前面。pipeline在收发数据的逻辑上与jedis的不同，基本就是这样了。

Jedis{
    pipeline 成员
    client 成员
}

pipeline(){
    client 成员。（触发client收发逻辑）
}

## spring-data-redis实现

很明显，从RedisTemplate作为入口开始分析

1. RedisTemplate extends RedisAccessor implements RedisOperations.

    redis的操作主要分为两种
    
    - 数据结构操作，list，set等，都有自己的独有的操作，所以将其作为一个类封起来
    - 数据控制操作，各种数据结构通用的，比如过期，删除啊
    - 关键方法是这个：`<T> T execute(RedisCallback<T> callback, boolean b)`，各种操作的最终实现

2. ValueOperations,ListOperation等，各种数据结构操作的方法。
3. AbstractOperations，各种数据结构Operation的公共类
4. RedisCallback，Callback interface for Redis 'low level' code
5. RedisConnection extends RedisCommands，A connection to a Redis server. Acts as an common abstraction across various Redis client libraries 

RedisTemplate将具体的数据结构操作委托给各种数据结构Operation（以下以ValueOperations举例），ValueOperations最终又调用了RedisTemplate的execute方法。

也就是说，RedisTemplate提供了高层调用结构和基本的逻辑实现（execute逻辑就是：建连接，拿数据，返回）。各数据结构特殊的部分由它们自己实现，双方的结合点就是callback。举个例子

    ValueDeserializingRedisCallback{
        protected abstract byte[] inRedis(byte[] key, RedisConnection connection);
    }

RedisTemplate负责提供connection，将key反序列化好 ，具体根据connection如何拿到一个合适的value，各个数据结构有自己的选择。



RedisConnection，由RedisConnectionFactory获取，JedisConnection作为RedisConnection的一种实现

JedisConnection{
      private final Jedis jedis;
      private final Pool<Jedis> pool;
}

redisTemplate --》 redisConnectionTemplate --> redisconnection.而一个redisconnection只对应一个host。

所以，想要弄成shard，而shard只能根据key来做。对于每个RedisOperation，改写其函数，操作key之前，先拿到对应的RedisTemplate。关键就看你对外提供什么样的抽象程度，假设类是RedisTemplateProxy

1. RedisTemplateProxy.template(key).opsForValue
2. RedisTemplateProxy.opsForValue()


## redis的其它应用

http://www.blogjava.net/masfay/archive/2012/07/03/382080.html

1. pipeline
2. 跨jvm的id生成器 
3. 跨jvm的锁实现
