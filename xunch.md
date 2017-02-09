## xunch 项目理解


spring data redis 并没有提供对shard的只是，初步理解，xunch是做这个事的。

sdr的redisTemplate提供了操作jedis的基本逻辑

{
    1. 获取连接（连接工厂化，池化）
    2. 操作
    3. 返回结果
    序列化和反序列化key和value等操作
}

shardRedisTemplate，相比原来，主要的增强应该就是获取连接上。

其shard的主要实现，跟jedis的shard实现非常相似。

还有一个是object cache的实现，这个跟service cache就很相似了（估摸着一开始object cache在这里实现，后来单独提出去了。）

object cache，我们往常的cache，数据结构都是key value，list等。其实也可以key object，这里就涉及到将object的序列化和反序列化。