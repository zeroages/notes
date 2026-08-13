# Spring Boot 常见使用场景

---

## 限流

在高并发场景下，为了防止系统被突发流量击穿，需要对请求进行速率限制。Spring Boot 环境下常见的限流方案如下：

### 方案一：Guava RateLimiter（单机限流）

基于令牌桶算法，适合单实例应用的限流场景。

```java
@Component
public class RateLimitService {

    // 每秒允许 100 个请求
    private final RateLimiter rateLimiter = RateLimiter.create(100);

    public boolean tryAcquire() {
        return rateLimiter.tryAcquire();
    }
}

@RestController
public class OrderController {

    @Autowired
    private RateLimitService rateLimitService;

    @PostMapping("/order")
    public ResponseEntity<String> createOrder() {
        if (!rateLimitService.tryAcquire()) {
            return ResponseEntity.status(429).body("请求过于频繁，请稍后再试");
        }
        // 业务逻辑
        return ResponseEntity.ok("下单成功");
    }
}
```

### 方案二：Redis + Lua 脚本（分布式限流）

适用于多实例部署的分布式限流场景，利用 Redis 的原子性保证计数准确。

```java
@Component
public class RedisRateLimiter {

    @Autowired
    private StringRedisTemplate redisTemplate;

    private static final String LUA_SCRIPT =
        "local key = KEYS[1] " +
        "local limit = tonumber(ARGV[1]) " +
        "local current = tonumber(redis.call('get', key) or '0') " +
        "if current + 1 > limit then " +
        "    return 0 " +
        "else " +
        "    redis.call('incrby', key, 1) " +
        "    redis.call('expire', key, 1) " +
        "    return 1 " +
        "end";

    public boolean tryAcquire(String key, int limit) {
        DefaultRedisScript<Long> script = new DefaultRedisScript<>(LUA_SCRIPT, Long.class);
        Long result = redisTemplate.execute(script, Collections.singletonList(key), String.valueOf(limit));
        return result != null && result == 1;
    }
}
```

### 方案三：Sentinel（阿里开源限流降级框架）

支持限流、熔断降级、系统负载保护等多种功能，适合微服务架构。

```java
@Configuration
public class SentinelConfig {

    @PostConstruct
    public void initFlowRules() {
        FlowRule rule = new FlowRule();
        rule.setResource("/api/order");
        rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        rule.setCount(20); // QPS 上限为 20
        FlowRuleManager.loadRules(Collections.singletonList(rule));
    }
}
```

### 限流方案对比

| 方案 | 适用场景 | 优点 | 缺点 |
| :--- | :--- | :--- | :--- |
| Guava RateLimiter | 单机限流 | 轻量、无外部依赖 | 无法跨实例 |
| Redis + Lua | 分布式限流 | 精确、跨实例 | 依赖 Redis 网络开销 |
| Sentinel | 微服务全方位保护 | 限流+熔断+降级 | 引入额外组件 |

---

## 一致性

在分布式系统中，数据一致性是核心挑战。Spring Boot 场景下常见的一致性问题及解决方案：

### 场景一：缓存与数据库一致性

**问题**：先更新数据库再删除缓存，如果删除失败会导致缓存与数据库不一致。

**延迟双删策略**：

```java
@Service
@Transactional
public class UserService {

    @Autowired
    private UserMapper userMapper;

    @Autowired
    private StringRedisTemplate redisTemplate;

    public void updateUser(User user) {
        // 1. 先删除缓存
        redisTemplate.delete("user:" + user.getId());

        // 2. 更新数据库
        userMapper.updateById(user);

        // 3. 延迟再删除一次缓存（异步执行）
        asyncDeleteCache("user:" + user.getId());
    }

    @Async
    public void asyncDeleteCache(String key) {
        try {
            Thread.sleep(500); // 延迟时间大于一次读操作的耗时
            redisTemplate.delete(key);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

### 场景二：分布式事务一致性

**Seata AT 模式整合 Spring Boot**：

```yaml
# application.yml
seata:
  enabled: true
  application-id: order-service
  tx-service-group: my_tx_group
  service:
    vgroup-mapping:
      my_tx-service-group: default
  client:
    rm:
      async-commit-buffer-limit: 10000
```

```java
@Service
public class OrderService {

    // Seata 全局事务注解，替代 @Transactional
    @GlobalTransactional(rollbackFor = Exception.class)
    public void createOrder(Order order) {
        // 1. 本地订单入库
        orderMapper.insert(order);

        // 2. 扣减库存（远程调用库存服务）
        stockService.deduct(order.getProductId(), order.getCount());

        // 3. 扣减余额（远程调用账户服务）
        accountService.debit(order.getUserId(), order.getAmount());
    }
}
```

### 场景三：消息最终一致性

利用 RabbitMQ 实现数据的最终一致性：

```java
@Service
public class OrderCreateService {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Transactional
    public void createOrder(Order order) {
        // 1. 保存订单（本地事务）
        orderMapper.insert(order);

        // 2. 发送消息到 MQ（配合本地消息表保证可靠性）
        rabbitTemplate.convertAndSend("order.exchange", "order.create", order);
    }
}
```

---

## 幂等

在分布式网络环境下，由于网络超时、重试机制等原因，同一请求可能被多次提交。幂等性保证多次请求的结果与一次请求相同。

### 场景一：表单重复提交（Token 机制）

**原理**：服务端生成唯一 Token，请求提交后校验并删除 Token，重复提交时 Token 已失效。

```java
@RestController
public class TokenController {

    @Autowired
    private StringRedisTemplate redisTemplate;

    @GetMapping("/token")
    public String generateToken() {
        String token = UUID.randomUUID().toString();
        redisTemplate.opsForValue().set("idempotent:token:" + token, "1", 5, TimeUnit.MINUTES);
        return token;
    }

    @PostMapping("/submit")
    public ResponseEntity<String> submit(@RequestParam("token") String token) {
        // 使用 Lua 脚本保证校验和删除的原子性
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                        "    return redis.call('del', KEYS[1]) " +
                        "else " +
                        "    return 0 " +
                        "end";
        DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>(script, Long.class);
        Long result = redisTemplate.execute(redisScript,
            Collections.singletonList("idempotent:token:" + token), "1");

        if (result == null || result == 0) {
            return ResponseEntity.badRequest().body("请勿重复提交");
        }

        // 执行业务逻辑
        return ResponseEntity.ok("提交成功");
    }
}
```

### 场景二：接口幂等（唯一索引 + 防重表）

**数据库唯一约束**是最可靠的幂等保障手段：

```java
@Service
public class PayService {

    @Transactional
    public void handlePayNotify(PayNotify notify) {
        // 利用订单号的唯一约束保证幂等
        try {
            payNotifyMapper.insertIgnore(notify); // INSERT IGNORE
        } catch (DuplicateKeyException e) {
            // 已处理过的通知，直接返回成功
            log.info("重复通知，orderNo={}", notify.getOrderNo());
            return;
        }

        // 执行业务：更新订单状态、发放权益等
        orderMapper.updateStatus(notify.getOrderNo(), "PAID");
    }
}
```

### 场景三：注解 + AOP 实现通用幂等拦截

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Idempotent {

    /** 幂等键前缀 */
    String prefix() default "";

    /** 过期时间（秒） */
    int expire() default 60;

    /** SpEL 表达式提取 key */
    String key() default "";
}

@Aspect
@Component
public class IdempotentAspect {

    @Autowired
    private StringRedisTemplate redisTemplate;

    @Autowired
    private SpelExpressionParser parser;

    @Autowired
    private DefaultParameterNameDiscoverer discoverer;

    @Around("@annotation(idempotent)")
    public Object around(ProceedingJoinPoint joinPoint, Idempotent idempotent) throws Throwable {
        String key = idempotent.prefix() + resolveKey(idempotent.key(), joinPoint);

        Boolean success = redisTemplate.opsForValue()
            .setIfAbsent(key, "1", idempotent.expire(), TimeUnit.SECONDS);

        if (success == null || !success) {
            throw new BusinessException("请勿重复操作");
        }

        try {
            return joinPoint.proceed();
        } catch (Exception e) {
            redisTemplate.delete(key); // 业务失败，释放幂等锁
            throw e;
        }
    }

    private String resolveKey(String spel, ProceedingJoinPoint joinPoint) {
        if (spel.isEmpty()) return "";
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        EvaluationContext context = new MethodBasedEvaluationContext(
            null, method, joinPoint.getArgs(), discoverer);
        return parser.parseExpression(spel).getValue(context, String.class);
    }
}
```

**使用方式**：

```java
@Idempotent(prefix = "order:create", key = "#order.userId", expire = 30)
public String createOrder(Order order) {
    // 业务逻辑
}
```

---

## 延迟双删

延迟双删是解决 **Redis 缓存与 MySQL 数据库数据不一致** 问题的经典策略。

### 问题背景

在「先更新数据库，再删除缓存」的策略下，可能出现以下时序问题：

```
时间线：
1. 线程A 删除缓存
2. 线程B 读取缓存未命中 → 读取旧数据
3. 线程B 将旧数据写入缓存
4. 线程A 更新数据库完成

结果：缓存中是旧数据，数据库是新数据 → 数据不一致！
```

### 策略原理

延迟双删通过「删一次 → 更新数据库 → 延迟再删一次」来覆盖不一致窗口：

```
时间线：
1. 删除缓存
2. 更新数据库
3. 延迟等待（大于一次读+写的耗时）
4. 再次删除缓存（清除步骤2期间可能被写入的旧数据）
```

### Spring Boot 实现

```java
@Service
public class ProductService {

    @Autowired
    private ProductMapper productMapper;

    @Autowired
    private StringRedisTemplate redisTemplate;

    @Autowired
    private ThreadPoolTaskScheduler taskScheduler;

    private static final String CACHE_KEY = "product:";

    /**
     * 延迟双删更新商品信息
     */
    @Transactional(rollbackFor = Exception.class)
    public void updateProduct(Product product) {
        String cacheKey = CACHE_KEY + product.getId();

        // 第一次删除缓存
        redisTemplate.delete(cacheKey);

        // 更新数据库（事务提交后才生效）
        productMapper.updateById(product);
    }

    /**
     * 事务提交后异步延迟双删
     */
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onProductUpdated(ProductUpdateEvent event) {
        String cacheKey = CACHE_KEY + event.getProductId();

        // 延迟删除（异步线程池执行）
        taskScheduler.schedule(() -> {
            redisTemplate.delete(cacheKey);
            log.info("延迟双删执行，key={}", cacheKey);
        }, Instant.now().plusSeconds(1)); // 延迟 1 秒
    }
}
```

### 更可靠的方案：Canal 异步监听 Binlog

延迟双删存在延迟时间难以精确设定的问题。生产环境中更推荐通过 Canal 监听 MySQL Binlog 来异步删除缓存：

```
MySQL Binlog → Canal → Kafka/RabbitMQ → 缓存删除消费者 → 删除 Redis 缓存
```

```java
@Component
public class CacheDeleteListener {

    @Autowired
    private StringRedisTemplate redisTemplate;

    @RabbitListener(queues = "canal.product")
    public void onBinlogChange(String message) {
        CanalMessage msg = JSON.parseObject(message, CanalMessage.class);
        if ("product".equals(msg.getTable())) {
            for (Map<String, String> row : msg.getData()) {
                String id = row.get("id");
                redisTemplate.delete("product:" + id);
                log.info("Binlog 触发缓存删除，productId={}", id);
            }
        }
    }
}
```

### 方案对比

| 方案 | 一致性保证 | 复杂度 | 实时性 | 适用场景 |
| :--- | :--- | :--- | :--- | :--- |
| 先删缓存再更新数据库 | 弱 | 低 | - | 一致性要求不高 |
| 延迟双删 | 较强 | 中 | 延迟秒级 | 主从延迟较小的场景 |
| Canal 监听 Binlog | 强 | 高 | 秒级 | 高一致性要求的场景 |
| 设置缓存过期时间 | 最终一致 | 低 | 依赖 TTL | 允许短暂不一致 |
