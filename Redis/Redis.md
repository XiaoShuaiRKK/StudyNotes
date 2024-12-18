# Redis

## 单线程 IO多路复用

![image-20241110153052887](./assets/image-20241110153052887.png)

随着redis的版本迭代，为了性能的考虑。redis在**4.0**逐渐引入多线程

**为了迎合硬件的升级，CPU核心数的不断增多，Redis也逐渐引入多线程，提升性能**

而影响redis性能的也并不主要是CPU，而是**网络和内存**



### Redis 3.x单线程的主要问题

正常情况下**使用del来删除key的时候可以很快删除数据**，而当**key是一个非常大的对象**时候，例如包含了成千上万元素的hash集合 ，那么此时**del指令就会造成Redis卡顿**

如果更多的客户端进行大key操作，那么**性能就会极具下降**，宛如加锁操作

### Redis 4.x

**于是在Redis 4.0 中就新增了多线程的模块，主要是为了解决删除数据效率比较低的问题**

| **unlink key**                   |
| -------------------------------- |
| **flushdb async**                |
| **flushall async**               |
| 把删除工作交给了子线程异步来删除 |

### Redis 6/7

虽然有些命令可以用后台线程或子线程执行(比如数据删除，快照生成，AOF重写)。但是，从**网络IO处理到实际的读写命令出来**，都是由单个线程完成的。

**Redis的性能瓶颈有时会出现在网络IO处理上**，也就是说，单个主线程出来**网络请求的速度跟不上底层网络硬件的速度**

**到了Redis6/7 采用多个IO线程来处理网络请求，提高请求的处理和并行度**

**Redis的多IO线程只是用来处理网络请求的，对于读写操作还是使用单线程处理，毕竟主要的瓶颈在IO而不是CPU。也为了保障事务的原子性，Lua脚本，也不需要进行加锁操作**

## Big Key 问题

**场景：**

存入100万数据

使用pipe管道来进行命令装箱，速度快很多

```java
log.info("===========Redis Test================");
redisUtil.pipeline(connection -> {
    for(int i=1;i<100*10000;i++){
        redisUtil.setForever("key:"+i,"value"+i);
    }
    return null;
});
log.info("===========Redis Test Over================");
```

100万的数据插入完成之后 只用了15s

### key * 如何避免

**这时候如果要遍历所有的数据 使用key*的话会出现卡顿的现象**

那不用**key*避免卡顿**，那该用什么

#### **SCAN命令**

用于**迭代数据库中的数据库键**

类似于Mysql 的 limit

该命令和密切相关的命令 [`SSCAN、HSCAN`](https://redis.io/docs/latest/commands/sscan/) 和 [`ZSCAN`](https://redis.io/docs/latest/commands/zscan/) 用于逐步迭代元素集合。

- `SCAN`迭代当前所选 Redis 数据库中的键集。
- [`SSCAN`](https://redis.io/docs/latest/commands/sscan/) 迭代 Sets 类型的元素。
- [`HSCAN`](https://redis.io/docs/latest/commands/hscan/) 迭代 Hash 类型的字段及其关联值。
- [`ZSCAN`](https://redis.io/docs/latest/commands/zscan/) 迭代 Sorted Set（ZSet） 类型的元素及其相关分数。

语法如下

```
SCAN cursor [MATCH pattern] [COUNT cout]
```

**cursor - 游标**

**pattern - 匹配模式**

**count - 指定从数据库集里面返回多少元素，默认为 10**

基于游标的迭代器，需要基于上一次的游标延续之前的迭代过程

以0作为游标开始一次新的迭代，直到命令返回游标0完成一次遍历

**就是比如,scan 0 之后 返回的值如果是0，则代表完成了数据集的遍历**

```
> scan 0
1) "17"
2)  1) "key:12"
    2) "key:8"
    3) "key:4"
    4) "key:14"
    5) "key:16"
    6) "key:17"
    7) "key:15"
    8) "key:10"
    9) "key:3"
   10) "key:7"
   11) "key:1"
> scan 17
1) "0"
2) 1) "key:5"
   2) "key:18"
   3) "key:0"
   4) "key:2"
   5) "key:19"
   6) "key:13"
   7) "key:6"
   8) "key:9"
   9) "key:11"
```

**用户在下次迭代时候需要使用这个新游标来作为SCAN命令的游标参数**，以此来延续之前的迭代过程

#### SCAN的遍历顺序

它不是从第一位数组的第0位一直遍历到末尾的，而是采用了**高位进位加法来遍历**。**考虑到了字典的扩容和缩容时避免槽位的遍历重复和遗漏**

**缺点：**

**不保证每次执行都返回某个给定数量的元素，支持模糊查询，一次返回的数量不可控，只能是大概率符合count参数**



### 危险命令的禁用

**注意：**

在正式环境中要禁用掉redis的几个危险命令

- **keys** *
- **flushdb**
- **flushall**

这几个命令都可能发生严重的数据事故，**导致数据丢失，缓存雪崩，redis宕机**

通过配置设置**禁用**这些命令，**redis.conf在SECURITY这一项中**

```conf
rename-command keys ""
rename-command flushdb ""
rename-command flushall ""
```

将这些危险命令直接设为空，以达到禁用的效果

### Big key

大的内容不是key本身，而是它对应的value

#### 多大算Big

**string类型控制在10KB以内，hash，list，set，zset元素个数不要超过5000**

通过指令查找bigkey

追加参数 来查找

```sh
redis-cli ..... --bigkeys
```

使用MEMORY USAGE 来查找key和它的值在RAM中所占用的字节数

```
MEMORY USAGE key
```

#### 如何删除

**非字符串的bigkey，不要使用del删除，使用scan、hscan....渐进式删除。同时防止Bigkey的过期时间自动删除问题**

##### hash

使用hscan**每次获得少量field-value**，**再使用hdel删除每个field**

##### list

使用ltrim

##### Set

使用sscan + srem

##### SortedSet

使用zscan + zrem

#### Lazy Freeing

调整redis.conf配置文**LAZY FREEING**

##### 阻塞和非阻塞删除命令

```properties
lazyfree-lazy-server-del no -->> yes
replica-lazy-flush no -->> yes
lazyfree-lazy-user-del no -->> yes
```

## 缓存双写一致性问题

### 读写缓存

根据业务的不同选择不同的策略

#### 同步直写策略

指的是直接回写到redis，当数据库写入之后，**不开启异步线程**。**同步写入redis**

**当你的业务数据是热点敏感信息的时候就要使用**。比如：VIP充值

#### 异步缓写策略

指的是当数据库写入之后，**开启异步线程，或者发消息给MQ来异步写入redis**

**当你的业务数据不需要立即返回，不是热点信息的时候可以使用**

### 双检加锁策略

**防止在同一时间过多的请求打到Mysql，当过多的请求达到Mysql，也同时会过多的写入Redis**

```java
User user = redisTemplate.opsForValue().get(key);
if(user == null){
    synchronized(UserService.class){
        user = redisTemplate.opsForValue().get(key);
        if(user == null){
            user = userMapper.select(id);
            if(user == null){
                return null;
            }else{
                redisTemplate.opsForValue().set(key,user);
            }
        }
    }
}
return userl
```

## 大数据统计

### 常见术语

**PV**: （Page View）访问量, 即页面浏览量或点击量，衡量网站用户访问的网页数量；在一定统计周期内用户每打开或刷新一个页面就记录1次，多次打开或刷新同一页面则浏览量累计。
**UV**: （Unique Visitor）独立访客，统计1天内访问某站点的用户数。可以理解成访问某网站的电脑的数量。页面访问人数，同一个账号访问同一个页面两次，UV算1次
**DAU**： (Daily Active User)日活跃用户数量。常用于反映网站、互联网应用或网络游戏的运营情况。DAU通常统计一日（统计日）之内，登录或使用了某个产品的用户数（去除重复登录的用户），这与流量统计工具里的访客（UV）概念相似。
MAU： (monthly active users)月活跃用户人数。是在线游戏的一个用户数量统计名词，数量越大意味着玩这款游戏的人越多。
PCU： (Peak concurrent users )最高同时在线玩家人数。
KPI： 关键绩效指标法，是企业绩效考核的方法之一，其特点是考核指标 围绕关键成果领域进行选取。
ARPU： (Average Revenue Per User)即每用户平均收入，用于衡量电信运营商和互联网公司业务收入的指标。

### 常见的四种统计

- 聚合统计：共同好友、猜你喜欢。**指的是交集，并集，差集**
- 排序统计：评论区根据日期排序统计。**ZSET**
- 二值统计：钉钉中的签到打卡。指只存放true或false 1或0的统计。**bitmap**
- 基数统计：对UV的统计。**指存放指定不重复的key的数据**。**hyperloglog**

### hyperloglog / GEO / bitmap

