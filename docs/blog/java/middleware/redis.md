# Redis 基础

## 概述

Redis（Remote Dictionary Server）是一个开源的、基于内存的键值对存储数据库，支持数据持久化。它支持多种数据结构，常用于缓存、消息队列、分布式锁等场景。

- **默认端口：** 6379
- **默认数据库数量：** 16（编号 0~15）
- **单线程模型：** 基于 I/O 多路复用，命令执行是原子性的

---

## 通用命令

```bash
SELECT 1              # 切换数据库
KEYS *                # 查看所有键（生产慎用）
EXISTS key            # 键是否存在
TYPE key              # 查看键的类型
DEL key               # 删除键
EXPIRE key 60         # 设置过期时间（秒）
TTL key               # 查看剩余过期时间（-1 永不过期，-2 已删除）
PERSIST key           # 移除过期时间
DBSIZE                # 当前数据库键数量
FLUSHDB               # 清空当前数据库
FLUSHALL              # 清空所有数据库
```

---

## 五大基本数据结构

### 1. String（字符串）

最基本的类型，值为字符串或数字，最大 512MB。

```bash
SET key value              # 设置键值
GET key                    # 获取值
DEL key                    # 删除
SETNX key value            # 键不存在时才设置（分布式锁核心）
SETEX key 60 value         # 设置值并指定过期时间（秒）
INCR key                   # 值加 1
INCRBY key 10              # 值加指定数
DECR key                   # 值减 1
STRLEN key                 # 获取字符串长度
MSET k1 v1 k2 v2           # 批量设置
MGET k1 k2                 # 批量获取
```

**应用场景：** 缓存、计数器、分布式锁、Session 共享

---

### 2. List（列表）

有序、可重复的字符串列表，底层是双向链表，支持两端推入/弹出。

```bash
LPUSH key a b c           # 从左侧推入元素
RPUSH key a b c           # 从右侧推入元素
LPOP key                  # 从左侧弹出
RPOP key                  # 从右侧弹出
LRANGE key 0 -1           # 获取范围内元素（0 到 -1 表示全部）
LLEN key                  # 获取列表长度
LINDEX key 0              # 获取指定索引元素
LREM key 1 "a"            # 删除 1 个值为 "a" 的元素
BLPOP key 30              # 阻塞式左侧弹出（超时 30 秒）
```

**应用场景：** 消息队列、最新动态列表、文章评论列表

---

### 3. Hash（哈希）

键值对集合，适合存储对象。

```bash
HSET key field value       # 设置字段值
HGET key field             # 获取字段值
HMSET key f1 v1 f2 v2      # 批量设置字段
HMGET key f1 f2            # 批量获取字段
HGETALL key                # 获取所有字段和值
HDEL key field             # 删除字段
HEXISTS key field          # 判断字段是否存在
HKEYS key                  # 所有字段名
HVALS key                  # 所有字段值
HLEN key                   # 字段数量
```

**应用场景：** 存储对象（用户信息、商品信息），比 JSON 字符串更省内存且可部分更新

---

### 4. Set（集合）

无序、不重复的字符串集合，支持集合间运算。

```bash
SADD key a b c            # 添加元素
SREM key a                 # 删除元素
SMEMBERS key              # 获取所有元素
SISMEMBER key a           # 判断元素是否存在
SCARD key                 # 获取集合大小
SRANDMEMBER key 3         # 随机获取 3 个元素
SPOP key                  # 随机弹出 1 个元素

# 集合运算
SUNION key1 key2          # 并集
SINTER key1 key2          # 交集
SDIFF key1 key2           # 差集（key1 有而 key2 没有）
```

**应用场景：** 标签系统、好友共同关注、抽奖（SPOP）、去重

---

### 5. ZSet / Sorted Set（有序集合）

每个元素关联一个分数（score），按分数从小到大排序，元素唯一但分数可重复。

```bash
ZADD key 80 "a" 90 "b"    # 添加元素（带分数）
ZREM key a                # 删除元素
ZSCORE key a              # 获取元素分数
ZINCRBY key 5 a           # 元素分数加 5
ZRANK key a               # 获取元素排名（从小到大）
ZREVRANK key a            # 获取排名（从大到小）
ZRANGE key 0 -1           # 按分数从小到大获取所有元素
ZREVRANGE key 0 -1        # 按分数从大到小获取所有元素
ZRANGEBYSCORE key 60 100  # 获取分数在 60~100 之间的元素
ZCARD key                 # 获取集合大小
ZCOUNT key 60 100         # 统计分数区间内的元素个数
```

**应用场景：** 排行榜、延迟队列（score 存时间戳）、限流

---

## 三种特殊数据类型

### 1. Bitmap（位图）

基于 String 类型，按位操作，适合二值状态统计。

```bash
SETBIT key 7 1            # 将第 7 位设为 1
GETBIT key 7              # 获取第 7 位的值
BITCOUNT key              # 统计值为 1 的位数
BITPOS key 1              # 第一个值为 1 的位置
```

**应用场景：** 签到统计、在线状态、布隆过滤器

---

### 2. HyperLogLog

基于基数统计的概率数据结构，占用固定 12KB 内存，存在约 0.81% 误差。

```bash
PFADD key a b c          # 添加元素
PFCOUNT key              # 统计基数（去重后的数量）
PFMERGE destkey k1 k2    # 合并多个 HyperLogLog
```

**应用场景：** UV 统计、独立访客计数、大数据量去重计数

---

### 3. Geospatial（地理位置）

基于 ZSet 实现，存储经纬度，支持距离计算。

```bash
GEOADD city 116.40 39.90 "beijing" 121.47 31.23 "shanghai"   # 添加位置
GEODIST city beijing shanghai km                              # 计算距离（千米）
GEORADIUS city 116.40 39.90 100 km                            # 搜索附近 100km 的城市
GEOPOS city beijing                                           # 获取经纬度
```

**应用场景：** 附近的人、打车距离计算、LBS 服务

---

## Java 操作 Redis（Jedis 示例）

### Maven 依赖

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>5.1.0</version>
</dependency>
```

### 基本使用

```java
import redis.clients.jedis.Jedis;
import java.util.Set;

public class RedisDemo {

    public static void main(String[] args) {
        // 连接 Redis
        Jedis jedis = new Jedis("localhost", 6379);
        // jedis.auth("password");  // 如有密码

        // String
        jedis.set("name", "zhangsan");
        System.out.println(jedis.get("name"));
        jedis.incr("count");

        // Hash
        jedis.hset("user:1", "name", "lisi");
        jedis.hset("user:1", "age", "25");
        System.out.println(jedis.hgetAll("user:1"));

        // List
        jedis.lpush("queue", "msg1", "msg2");
        System.out.println(jedis.rpop("queue"));

        // Set
        jedis.sadd("tags", "java", "redis", "mysql");
        Set<String> members = jedis.smembers("tags");
        System.out.println(members);
        System.out.println(jedis.sismember("tags", "java"));

        // ZSet
        jedis.zadd("rank", 95.5, "alice");
        jedis.zadd("rank", 88.0, "bob");
        System.out.println(jedis.zrevrangeWithScores("rank", 0, -1));

        jedis.close();
    }
}
```

### Spring Boot + RedisTemplate

```java
@Autowired
private RedisTemplate<String, Object> redisTemplate;

// String
redisTemplate.opsForValue().set("key", "value");
Object val = redisTemplate.opsForValue().get("key");

// Hash
redisTemplate.opsForHash().put("user", "name", "zhangsan");
Object name = redisTemplate.opsForHash().get("user", "name");

// List
redisTemplate.opsForList().leftPush("list", "item");
Object item = redisTemplate.opsForList().rightPop("list");

// Set
redisTemplate.opsForSet().add("set", "a", "b", "c");
Set<Object> members = redisTemplate.opsForSet().members("set");

// ZSet
redisTemplate.opsForZSet().add("zset", "member", 100);
Set<Object> range = redisTemplate.opsForZSet().range("zset", 0, -1);
```

---

## 持久化

| 方式 | 全称 | 原理 | 优点 | 缺点 |
|------|------|------|------|------|
| RDB | Redis DataBase | 定时生成内存快照 | 文件紧凑，恢复快 | 可能丢失最后一次快照后的数据 |
| AOF | Append Only File | 记录每条写命令 | 数据更完整 | 文件体积大，恢复慢 |
| 混合持久化 | RDB + AOF | 4.0+ 支持，AOF 重写时用 RDB 格式开头 | 兼顾恢复速度和数据安全 | - |

```bash
# redis.conf 配置
save 900 1              # 900 秒内至少 1 次修改则触发 RDB
appendonly yes          # 开启 AOF
appendfsync everysec    # 每秒刷盘一次（推荐）
```

---

## 常见问题

### 缓存穿透

**问题：** 查询一个一定不存在的数据，缓存和存储层都查不到，每次都打到数据库。

**解决方案：**
- 布隆过滤器拦截不存在的请求
- 缓存空值（设置较短过期时间）

### 缓存击穿

**问题：** 某个热点 key 过期瞬间，大量请求同时打到数据库。

**解决方案：**
- 热点数据设置永不过期
- 加互斥锁（SETNX），只允许一个线程回源

### 缓存雪崩

**问题：** 大量 key 同时过期，请求全部打到数据库。

**解决方案：**
- 过期时间加随机值，避免同时失效
- 多级缓存（本地缓存 + Redis）
- 限流降级
