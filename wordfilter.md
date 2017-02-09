# wordfilter 项目理解

从model开始分析，这些model会从数据库获取，本质上还是为了存储。

spamApp{
    boolean isUserOverflowForbid
    boolean isIpOverflowForbid
    int doSteps
    Integer appId
    String appName
    Integer userOverflowLimit
    Integer userOverflowCount
    Integer userRepeatPeriod
    Integer userRepeatCount
    Integer ipOverflowLimit
    Integer ipOverflowCount
    Integer ipRepeatPeriod
    Integer ipRepeatCount
    Boolean isUserOverflowForbidIp
    Integer forbidUserTime
    Integer forbidIpTime
    Integer wordDeadScore
    String step
    Date createdTime
    Date updatedTime
    Integer userCaptchaOverflowPeriod
    Integer userCaptchaOverflowCount
    Integer ipCaptchaOverflowPeriod
    Integer ipCaptchaOverflowCount
    Integer maxSequenceChinese
    Integer maxSequenceCharAndDigit
}

SpamIp{
    String ip
    Date startTime
    Date releaseTime
    Integer appId
}

SpamUser{
    Integer userid
    Date startTime
    Date releaseTime
    Integer appId
}

SpamWord{
    String word
    Byte length
    Byte weight
    String category
    Integer infocome
    String adder
    Date addTime
    Date lastTime
    String piclink
    String remark
}

从这样的代码中，就可以知道，我们应该从什么样的维度来考虑频率控制问题。

然后分析WordFilterService，发现其一个核心是Filter。

这个项目本身负责的比较多

1. 用户是否在黑名单中（redis）
2. 用户是否次数过多（guava cache）
3. 次数过多被禁了之后，是否在被禁时间段内

wordFilter(int application, long userId, String ip, byte[] txtData) 

filter.Check(txtData, Integer.valueOf(String.valueOf(userId)), ip, null, application)

check(text, userId, ip, agent, application, beginTime)

isSpam(userId, time, spamApp, userOverflowCache, userflowCount, userflowLimit);

OpCountTime{
    int count;
    int lastOpTime;
}
 

## bloomfilter

使用bloomfilter的问题，对于不同的业务，过期时间不同，这个在bloomfilter里需要我们自己包装（貌似也基本做不到）。bloomfilter只能做到，这个bloomfilter负责的数据要么全有，全么全干掉

## 方案一

本地guava cache + 远程redis。读的时候，本地guava cache

## 方案二 

rocksdb 或 ardb + xx 对外提供thrift服务，客户端通过mainstay访问

1. rocksdb或ardb的过期
2. mainstay的分发，因为服务器是本地存储的，必须保证一个id一定咋某个机器上，如果这个机器挂了。那么“宁愿不给用户发，不要多发”

这两样如何保证？
