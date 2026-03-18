# 问题分析案例集

## 案例一：服务OOM导致频繁重启

### 问题现象
生产环境某微服务每2-3小时自动重启，重启期间服务不可用约30秒。

### 分析过程

**1. 查看Pod事件**
```bash
kubectl describe pod <pod-name>
```
发现事件：`OOMKilled` - 容器因内存超出限制被kill。

**2. 查看监控**
- 内存使用率持续上升，直到达到limit（2GB）后触发OOM
- GC频率逐渐增高，Full GC无法释放内存

**3. 分析Heap Dump**
```bash
jmap -dump:format=b,file=heap.hprof <pid>
```
使用MAT分析发现：
- 90%内存被 `ConcurrentHashMap$Node` 占用
- 堆栈指向一个本地缓存组件

**4. 代码审查**
```java
// 问题代码
@Cacheable(value = "userCache", sync = true)
public User getUser(Long id) {
    return userMapper.selectById(id);
}
```
缓存没有设置过期时间，且缓存key为所有用户ID，用户量增长后缓存无限膨胀。

### 根因
本地缓存（Caffeine）未配置最大容量和过期策略，导致内存持续增长最终OOM。

### 解决方案
```java
// 修复后的配置
@Bean
public CacheManager cacheManager() {
    CaffeineCacheManager cacheManager = new CaffeineCacheManager();
    cacheManager.setCaffeine(Caffeine.newBuilder()
        .maximumSize(10000)           // 最大条目数
        .expireAfterWrite(10, TimeUnit.MINUTES)  // 写入后过期时间
        .recordStats());
    return cacheManager;
}
```

### 预防措施
1. 所有本地缓存必须配置容量上限和过期策略
2. 监控缓存命中率和内存使用
3. 考虑使用分布式缓存替代大容量本地缓存

---

## 案例二：数据库连接池耗尽

### 问题现象
高峰期接口响应变慢，大量请求超时，错误日志显示获取连接超时。

### 分析过程

**1. 查看错误日志**
```
Caused by: java.sql.SQLException: Connection pool is full
    at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:180)
```

**2. 检查连接池配置**
```yaml
spring.datasource.hikari.maximum-pool-size: 10
spring.datasource.hikari.connection-timeout: 30000
```

**3. 分析线程Dump**
发现大量线程阻塞在：
```
"http-nio-8080-exec-123" #234 prio=5 os_prio=0 tid=0x00007f123456789 nid=0x1234 waiting for monitor entry
   java.lang.Thread.State: BLOCKED
    at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:180)
```

**4. 代码审查**
发现某查询接口：
```java
@Transactional(readOnly = true)
public List<Order> getOrders(Long userId) {
    List<Order> orders = orderMapper.selectByUserId(userId);
    // 循环内每次查询都新开事务
    orders.forEach(order -> {
        order.setItems(orderItemMapper.selectByOrderId(order.getId())); // 这里有问题
    });
    return orders;
}
```

### 根因
- 连接池配置过小（仅10个）
- N+1查询问题导致连接长时间占用
- 事务范围过大，连接迟迟不释放

### 解决方案
1. **优化连接池配置**：
```yaml
spring.datasource.hikari.maximum-pool-size: 50
spring.datasource.hikari.minimum-idle: 10
spring.datasource.hikari.idle-timeout: 600000
spring.datasource.hikari.max-lifetime: 1800000
spring.datasource.hikari.connection-timeout: 30000
```

2. **修复N+1查询**：
```java
@Transactional(readOnly = true)
public List<Order> getOrders(Long userId) {
    // 使用JOIN一次性查询
    return orderMapper.selectByUserIdWithItems(userId);
}
```

3. **添加连接池监控**：
```java
@Bean
public MeterBinder hikariMetrics(HikariDataSource dataSource) {
    return registry -> {
        registry.gauge("hikari.connections.active", 
            dataSource, HikariDataSource::getActiveConnections);
        registry.gauge("hikari.connections.idle", 
            dataSource, HikariDataSource::getIdleConnections);
        registry.gauge("hikari.connections.pending", 
            dataSource, HikariDataSource::getPendingThreads);
    };
}
```

### 预防措施
1. 根据并发量合理配置连接池大小
2. 避免N+1查询，使用JOIN或批量查询
3. 缩小事务范围，及时释放连接
4. 监控连接池使用率和等待队列

---

## 案例三：Redis缓存穿透引发DB雪崩

### 问题现象
某热点数据接口突然变慢，数据库CPU飙升至100%，大量查询超时。

### 分析过程

**1. 查看监控**
- Redis命中率骤降
- 数据库QPS瞬间增长10倍
- 大量相同key的查询

**2. 查看日志**
```
2024-01-15 10:23:45 [ERROR] Query failed: SELECT * FROM product WHERE id = 999999
2024-01-15 10:23:45 [ERROR] Query failed: SELECT * FROM product WHERE id = 999999
...（重复上千次）
```

**3. 分析代码**
```java
public Product getProduct(Long id) {
    String key = "product:" + id;
    Product product = redisTemplate.opsForValue().get(key);
    if (product == null) {
        // 缓存未命中，查询数据库
        product = productMapper.selectById(id);
        if (product != null) {
            redisTemplate.opsForValue().set(key, product, 1, TimeUnit.HOURS);
        }
    }
    return product;
}
```

**4. 问题定位**
- 某个不存在的商品ID（999999）被大量请求
- 缓存未命中，直接打到数据库
- 数据库查询空结果，也未写入缓存
- 大量并发请求导致数据库压力激增

### 根因
缓存穿透：查询不存在的数据，绕过缓存直接访问数据库，高并发下导致数据库雪崩。

### 解决方案
1. **缓存空值**：
```java
public Product getProduct(Long id) {
    String key = "product:" + id;
    Product product = redisTemplate.opsForValue().get(key);
    if (product == null) {
        product = productMapper.selectById(id);
        if (product != null) {
            redisTemplate.opsForValue().set(key, product, 1, TimeUnit.HOURS);
        } else {
            // 缓存空值，短时间过期
            redisTemplate.opsForValue().set(key, "NULL", 5, TimeUnit.MINUTES);
        }
    }
    // 处理空值标记
    return "NULL".equals(product) ? null : product;
}
```

2. **布隆过滤器**：
```java
@Component
public class ProductBloomFilter {
    @PostConstruct
    public void init() {
        // 加载所有有效ID到布隆过滤器
        List<Long> ids = productMapper.selectAllIds();
        ids.forEach(id -> bloomFilter.put("product:" + id));
    }
    
    public boolean mightContain(Long id) {
        return bloomFilter.mightContain("product:" + id);
    }
}

public Product getProduct(Long id) {
    // 先查布隆过滤器
    if (!bloomFilter.mightContain(id)) {
        return null; // 一定不存在
    }
    // 继续查缓存...
}
```

3. **接口限流**：
```java
@RateLimiter(key = "product", permitsPerSecond = 1000)
public Product getProduct(Long id) {
    // ...
}
```

### 预防措施
1. 对不存在的数据也进行缓存（空值缓存）
2. 使用布隆过滤器前置拦截无效请求
3. 接口限流保护，防止单接口拖垮系统
4. 监控缓存命中率和数据库QPS

---

## 案例四：异步线程池阻塞导致任务堆积

### 问题现象
异步任务执行延迟越来越高，消息队列消费速度变慢，任务堆积严重。

### 分析过程

**1. 查看线程池状态**
```java
ThreadPoolExecutor executor = (ThreadPoolExecutor) asyncExecutor;
System.out.println("Active: " + executor.getActiveCount());
System.out.println("Queue size: " + executor.getQueue().size());
System.out.println("Completed: " + executor.getCompletedTaskCount());
```
输出：
```
Active: 10
Queue size: 5000
Completed: 100
```

**2. 线程Dump分析**
发现所有线程都阻塞在：
```
"async-executor-1" #45 prio=5 os_prio=0 tid=0x00007f123456789 nid=0x2d3 waiting on condition
   java.lang.Thread.State: WAITING (parking)
    at sun.misc.Unsafe.park(Native Method)
    at java.util.concurrent.FutureTask.get(FutureTask.java:192)
    at com.example.service.OrderService.processOrder(OrderService.java:85)
```

**3. 代码审查**
```java
@Async("asyncExecutor")
public void processOrder(Long orderId) {
    Order order = orderMapper.selectById(orderId);
    // 同步调用外部HTTP接口
    User user = userServiceClient.getUser(order.getUserId()); // 阻塞等待
    // 同步发送消息
    messageService.send(order); // 阻塞等待
    // 更新订单状态
    orderMapper.updateStatus(orderId, "PROCESSED");
}
```

### 根因
- 线程池配置过小（核心线程数=10）
- 任务中存在大量同步阻塞调用（HTTP、MQ）
- 线程被长时间占用，新任务进入队列堆积

### 解决方案
1. **增大线程池并配置拒绝策略**：
```java
@Bean("asyncExecutor")
public Executor asyncExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(20);
    executor.setMaxPoolSize(100);
    executor.setQueueCapacity(200);
    executor.setThreadNamePrefix("async-");
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    executor.initialize();
    return executor;
}
```

2. **异步化改造**：
```java
@Async("asyncExecutor")
public CompletableFuture<Void> processOrder(Long orderId) {
    return CompletableFuture.supplyAsync(() -> orderMapper.selectById(orderId))
        .thenComposeAsync(order -> 
            userServiceClient.getUserAsync(order.getUserId()) // 异步HTTP调用
                .thenComposeAsync(user -> 
                    messageService.sendAsync(order) // 异步MQ发送
                )
        )
        .thenAcceptAsync(result -> 
            orderMapper.updateStatus(orderId, "PROCESSED")
        );
}
```

3. **添加监控告警**：
```java
@Scheduled(fixedRate = 60000)
public void monitorThreadPool() {
    ThreadPoolExecutor executor = (ThreadPoolExecutor) asyncExecutor;
    double activeRate = (double) executor.getActiveCount() / executor.getMaximumPoolSize();
    if (activeRate > 0.8) {
        alertService.sendAlert("线程池使用率过高: " + (activeRate * 100) + "%");
    }
    if (executor.getQueue().size() > 1000) {
        alertService.sendAlert("线程池任务队列堆积: " + executor.getQueue().size());
    }
}
```

### 预防措施
1. 根据任务特性合理配置线程池参数
2. 避免在线程池中执行长时间阻塞操作
3. 使用异步非阻塞调用替代同步调用
4. 监控线程池活跃线程数和队列堆积情况

---

## 案例五：分布式锁失效导致数据不一致

### 问题现象
库存扣减出现超卖，实际库存为0但系统显示还有库存可售。

### 分析过程

**1. 查看业务日志**
```
2024-01-20 14:30:10 [INFO] 用户A开始扣减库存，商品ID=1001，数量=1
2024-01-20 14:30:10 [INFO] 用户B开始扣减库存，商品ID=1001，数量=1
2024-01-20 14:30:11 [INFO] 用户A扣减成功，剩余库存=0
2024-01-20 14:30:11 [INFO] 用户B扣减成功，剩余库存=-1
```

**2. 查看锁的实现**
```java
public boolean deductStock(Long productId, Integer quantity) {
    String lockKey = "stock:" + productId;
    // 获取锁
    Boolean locked = redisTemplate.opsForValue().setIfAbsent(lockKey, "1", 5, TimeUnit.SECONDS);
    if (Boolean.TRUE.equals(locked)) {
        try {
            // 查询库存
            Integer stock = stockMapper.selectStock(productId);
            if (stock >= quantity) {
                // 扣减库存
                stockMapper.deduct(productId, quantity);
                return true;
            }
        } finally {
            // 释放锁
            redisTemplate.delete(lockKey);
        }
    }
    return false;
}
```

**3. 问题定位**
- 锁的过期时间只有5秒
- 业务逻辑执行时间可能超过5秒（如包含其他IO操作）
- 锁自动过期后，其他线程可以获取锁
- 导致多个线程同时执行扣减逻辑

### 根因
分布式锁过期时间设置过短，业务执行时间超过锁有效期，导致锁提前释放，失去互斥保护。

### 解决方案
1. **使用Redisson实现可重入锁**：
```java
@Autowired
private RedissonClient redissonClient;

public boolean deductStock(Long productId, Integer quantity) {
    String lockKey = "stock:" + productId;
    RLock lock = redissonClient.getLock(lockKey);
    
    try {
        // 尝试获取锁，最多等待3秒，锁自动续期
        boolean locked = lock.tryLock(3, 30, TimeUnit.SECONDS);
        if (locked) {
            try {
                Integer stock = stockMapper.selectStock(productId);
                if (stock >= quantity) {
                    stockMapper.deduct(productId, quantity);
                    return true;
                }
            } finally {
                lock.unlock();
            }
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
    return false;
}
```

2. **使用Lua脚本保证原子性**：
```java
private static final String DEDUCT_SCRIPT = 
    "if (redis.call('get', KEYS[1]) - ARGV[1] >= 0) then " +
    "    redis.call('decrby', KEYS[1], ARGV[1]); " +
    "    return 1; " +
    "else " +
    "    return 0; " +
    "end";

public boolean deductStock(Long productId, Integer quantity) {
    String stockKey = "stock:" + productId;
    Long result = redisTemplate.execute(
        new DefaultRedisScript<>(DEDUCT_SCRIPT, Long.class),
        Collections.singletonList(stockKey),
        quantity.toString()
    );
    return result != null && result == 1;
}
```

3. **数据库乐观锁兜底**：
```java
@Update("UPDATE product_stock SET stock = stock - #{quantity}, version = version + 1 " +
        "WHERE product_id = #{productId} AND stock >= #{quantity} AND version = #{version}")
int deductWithVersion(@Param("productId") Long productId, 
                      @Param("quantity") Integer quantity,
                      @Param("version") Integer version);
```

### 预防措施
1. 使用成熟的分布式锁实现（Redisson）
2. 设置合理的锁过期时间，并实现看门狗自动续期
3. 关键操作使用Lua脚本保证原子性
4. 数据库层添加乐观锁作为最终兜底
5. 监控锁的获取成功率和等待时间

---

## 案例六：MC补丁包部署主键冲突

### 问题现象
MC安装补丁包时，安装日志报错：
```
traceId:a11c515b3e95702d, deploy error dym file :isv_bd_test.en_US.dym
xkd.bos.exception.KDException: ERROR: duplicate key value violates unique constraint "idx_meta_formdsgn_l_number"
  Detail: Key (fnumber, flocaleid)=(isv_bd_test, en_US) already exists.
RequestContext: tenantId=sit_isvtest, accountId=2194180803615792128
SQL: /*ORM*/ INSERT INTO T_META_FORMDESIGN_L(FLOCALEID,FVERSION,FID,FNAME,FDATA,FPKID,FNUMBER) VALUES (?,?,?,?,?,?,?)
```

### 分析过程

**1. 分析错误信息**
- 错误类型：`duplicate key value violates unique constraint`
- 冲突表：`T_META_FORMDESIGN_L`
- 冲突字段：`fnumber, flocaleid`
- 冲突值：`(isv_bd_test, en_US)`
- 操作类型：`INSERT`

**2. 理解业务场景**
- 补丁包中包含表单 `isv_bd_test` 的元数据
- 目标环境已存在相同编码的表单
- 但两者的主键ID（FID）不同

**3. 追溯变更历史**
- 补丁包首次部署后，开发环境删除了该表单
- 开发环境重新创建了相同编码（isv_bd_test）的表单
- 重新创建后表单分配了新的主键ID
- 新补丁包中包含新ID的表单元数据
- 目标环境仍保留旧ID的表单数据

**4. 冲突原因**
数据库唯一索引 `idx_meta_formdsgn_l_number` 约束了 `(fnumber, flocaleid)` 的组合必须唯一：
```sql
-- 目标环境已有数据
SELECT FID, FNUMBER, FLOCALEID FROM T_META_FORMDESIGN_L 
WHERE FNUMBER = 'isv_bd_test' AND FLOCALEID = 'en_US';
-- 结果：FID = 100001 (旧ID)

-- 补丁包尝试插入
INSERT INTO T_META_FORMDESIGN_L(FLOCALEID,FVERSION,FID,FNAME,FDATA,FPKID,FNUMBER) 
VALUES ('en_US', 1, 100002, 'Test Form', '...', 200002, 'isv_bd_test');
-- 失败：违反唯一约束
```

### 根因分类

| 类型 | 场景描述 | 涉及表 |
|-----|---------|--------|
| **类型A：历史残留** | 之前部署未完全回滚，残留元数据记录 | t_meta_formdesign |
| **类型B：版本不一致** | 补丁包版本号与数据中心版本不匹配，未触发更新逻辑 | t_meta_formdesign |
| **类型C：并发冲突** | 多实例同时部署同一元数据 | t_meta_formdesign |
| **类型D：跨包重复** | 同一表单出现在多个补丁包中 | t_meta_formdesign |
| **类型E：应用重建** ⭐NEW | 开发环境删除应用后重建，编码相同但主键ID变化，部署到已有环境 | **t_meta_bizapp** |

**类型E详细说明（新增）**：

**触发条件**：
1. 应用已部署到目标环境（如生产环境）
2. 开发人员在开发环境删除了该应用
3. 开发人员在开发环境重新创建了**同名应用**（编码相同）
4. 新创建的应用生成了**新的主键ID**
5. 将新应用打包部署到已有该应用的环境

**冲突机制**：
```
目标环境已有：fnumber='p0g4_basedata_ext', fid=原ID
新补丁包包含：fnumber='p0g4_basedata_ext', fid=新ID
部署时尝试INSERT：fnumber冲突 → 报错
```

**与表单级冲突的区别**：
- 表单级冲突：涉及 `t_meta_formdesign` 表，冲突约束为 `(fnumber, flocaleid)`
- 应用级冲突：涉及 `t_meta_bizapp` 表，冲突约束为 `fnumber`

苍穹平台中，表单元数据表使用 `fnumber`（编码）+ `flocaleid`（语言）作为业务唯一键，但补丁包部署时按主键ID进行数据同步。删除后重建相同编码的表单会生成新的主键ID，导致新旧数据在目标环境冲突。

### 解决方案

**方案一：修改表单编码（推荐）**
```sql
-- 在开发环境修改表单编码
UPDATE T_META_FORMDESIGN SET FNUMBER = 'isv_bd_test_v2' WHERE FNUMBER = 'isv_bd_test';
UPDATE T_META_FORMDESIGN_L SET FNUMBER = 'isv_bd_test_v2' WHERE FNUMBER = 'isv_bd_test';
-- 重新导出补丁包部署
```

**方案二：补丁包中增加清理SQL**
```sql
-- 在补丁包SQL脚本中添加前置清理
-- 注意：仅适用于确认目标环境数据可删除的场景

-- 删除旧的多语言数据
DELETE FROM T_META_FORMDESIGN_L 
WHERE FNUMBER = 'isv_bd_test' AND FLOCALEID = 'en_US';

-- 删除旧的表单设计数据
DELETE FROM T_META_FORMDESIGN 
WHERE FNUMBER = 'isv_bd_test';

-- 补丁包中的INSERT将正常执行
```

**方案三：使用MC的数据清理功能**
```bash
# 登录MC管理控制台
# 进入：应用管理 > 元数据清理
# 选择租户和表单编码
# 执行元数据清理后再部署补丁包
```

**类型E专用方案（应用重建冲突）**：
```sql
-- 1. 查询确认应用冲突
SELECT fid, fnumber, fname FROM t_meta_bizapp 
WHERE fnumber = 'p0g4_basedata_ext';

-- 2. 清理应用及相关元数据（谨慎操作）
-- 先清理依赖的表单元数据
DELETE FROM t_meta_formdesign_l 
WHERE fnumber IN (SELECT fnumber FROM t_meta_formdesign WHERE fappid = '应用ID');
DELETE FROM t_meta_formdesign WHERE fappid = '应用ID';
-- 再清理应用定义
DELETE FROM t_meta_bizapp WHERE fnumber = 'p0g4_basedata_ext';

-- 3. 重新部署补丁包
```

**根治方案（推荐）**：
- 开发环境删除应用后，**修改应用编码**再重新创建
- 或在目标环境也删除原应用后重新部署（数据会丢失，谨慎操作）

### 预防措施

1. **规范元数据变更流程**
   - 表单一旦部署到客户环境，禁止通过"删除+重建"方式修改
   - 如需调整，应直接修改现有表单而非重建
   - 重大变更需评估对现有数据的影响

2. **补丁包发布前检查**
   ```bash
   # 检查清单
   - [ ] 确认表单编码是否与已发布版本冲突
   - [ ] 检查主键ID是否发生变化
   - [ ] 验证补丁包在干净环境可正常部署
   - [ ] 验证补丁包在已有数据环境可正常升级
   ```

3. **版本管理规范**
   - 表单编码变更应遵循版本命名规范（如：isv_bd_test_v2）
   - 在补丁包说明文档中标注破坏性变更
   - 保留元数据变更历史记录

4. **部署前备份**
   ```sql
   -- 部署前备份关键表
   CREATE TABLE T_META_FORMDESIGN_BAK_20240120 AS SELECT * FROM T_META_FORMDESIGN;
   CREATE TABLE T_META_FORMDESIGN_L_BAK_20240120 AS SELECT * FROM T_META_FORMDESIGN_L;
   ```

5. **监控部署日志**
   - 关注 `duplicate key`、`unique constraint` 等关键字
   - 部署失败时第一时间分析traceId对应的完整日志
   - 建立部署问题知识库，积累常见错误解决方案

6. **开发规范（针对类型E）**
   ```markdown
   ✗ 禁止操作：
   - 删除已部署的应用后重建相同编码的应用
   - 通过删除+新建方式"重置"应用
   
   ✓ 正确操作：
   - 应用一旦部署，只能通过修改方式调整
   - 如需重大调整，使用版本号区分（如：p0g4_basedata_ext_v2）
   - 保持开发/测试/生产环境元数据ID一致
   ```

### 参考资源
- 更多信息请参考金蝶云社区文章：[MC补丁包部署问题汇总](https://vip.kingdee.com/knowledge/608952192765973504?get_from=knowledge-specialDetail-id&productLineId=40&isKnowledge=2&lang=zh-CN#6)

---

## 案例七：MC补丁包SQL执行顺序依赖错误

### 问题现象
MC安装补丁包时，安装日志报错：
```
traceId:d85237b790975cc4, 脚本文件：/tmp/3d4a55c86f7e2f60dce7bdfd3f5ebb04/datamodel/1.5/main/tcus_shsh/dbschema/tcus_26022501_shouhuo_table.sql中，第1条SQL执行出错，脚本文件内还有0条sql未执行
2026-03-16 16:34:25.427 ERRORINFO - 2436466213459591168 : traceId:d85237b790975cc4, sql执行错误，错误信息：deploy script error--traceId:d85237b790975cc4 --isAll:false--DBRouteKey:secd -- error:ERROR: relation "tk_tcus_change" does not exist
  Where: SQL statement " alter table tk_tcus_change alter column fk_tcus_largetextfield TYPE VARCHAR(2000) "
PL/pgSQL function p_altercolumn(character varying,character varying,character varying,character varying,character varying,character varying) line 1 at EXECUTE
```

### 分析过程

**1. 分析错误信息**
- 错误类型：`ERROR: relation "tk_tcus_change" does not exist`
- 错误SQL：`alter table tk_tcus_change alter column fk_tcus_largetextfield TYPE VARCHAR(2000)`
- 涉及函数：`p_altercolumn`（封装后的字段修改函数）
- 脚本路径：`tcus_shsh/dbschema/tcus_26022501_shouhuo_table.sql`
- 执行顺序：第1条SQL就失败，剩余0条未执行

**2. 理解错误本质**
PostgreSQL 报错 `relation does not exist` 表示表不存在：
```sql
-- 尝试执行的SQL
ALTER TABLE tk_tcus_change ALTER COLUMN fk_tcus_largetextfield TYPE VARCHAR(2000);
-- 错误：表 tk_tcus_change 不存在，无法修改字段
```

**3. 分析补丁包结构**
典型的苍穹补丁包 SQL 目录结构：
```
patch-package/
├── datamodel/
│   └── 1.5/
│       └── main/
│           └── tcus_shsh/
│               ├── dbschema/
│               │   ├── tcus_26022501_shouhuo_table.sql      -- 建表脚本
│               │   └── tcus_26022501_shouhuo_table_alter.sql -- 改字段脚本
│               └── initdata/
│                   └── tcus_26022501_init_data.sql          -- 初始化数据
```

**4. 问题定位**
- 补丁包中同时包含建表脚本和修改字段脚本
- 修改字段的 SQL 被配置为先执行
- 建表 SQL 被配置为后执行（或未包含在当前补丁包中）
- 执行顺序配置错误导致依赖关系被破坏

### 根因
MC 补丁包部署时，SQL 脚本的执行顺序由配置文件控制。当修改字段的 SQL 脚本被配置在创建表之前执行时，由于目标表尚未创建，导致 ALTER TABLE 语句执行失败。

### 解决方案

**方案一：调整 SQL 执行顺序配置（推荐）**

编辑补丁包中的 SQL 执行顺序配置文件（通常是 `dbscript.xml` 或 `patch.xml`）：
```xml
<!-- 错误配置示例 -->
<dbscripts>
    <script file="tcus_shsh/dbschema/tcus_26022501_shouhuo_table_alter.sql" seq="10"/>
    <script file="tcus_shsh/dbschema/tcus_26022501_shouhuo_table.sql" seq="20"/>
</dbscripts>

<!-- 正确配置 -->
<dbscripts>
    <!-- 先执行建表脚本 -->
    <script file="tcus_shsh/dbschema/tcus_26022501_shouhuo_table.sql" seq="10"/>
    <!-- 再执行修改字段脚本 -->
    <script file="tcus_shsh/dbschema/tcus_26022501_shouhuo_table_alter.sql" seq="20"/>
</dbscripts>
```

**方案二：合并 SQL 脚本**

将建表和修改字段合并到同一个 SQL 文件中，确保执行顺序：
```sql
-- tcus_26022501_shouhuo_table.sql
-- 第1步：创建表
CREATE TABLE IF NOT EXISTS tk_tcus_change (
    pkid BIGINT PRIMARY KEY,
    fnumber VARCHAR(50),
    fk_tcus_largetextfield VARCHAR(2000),  -- 直接定义为目标类型
    fcreatetime TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 第2步：如果表已存在，修改字段（使用条件判断避免重复执行报错）
DO $$
BEGIN
    IF EXISTS (
        SELECT 1 FROM information_schema.tables 
        WHERE table_name = 'tk_tcus_change'
    ) THEN
        ALTER TABLE tk_tcus_change 
        ALTER COLUMN fk_tcus_largetextfield TYPE VARCHAR(2000);
    END IF;
END $$;
```

**方案三：使用条件判断增强 SQL 健壮性**
```sql
-- 在修改字段前增加表存在性检查
DO $$
BEGIN
    -- 检查表是否存在
    IF EXISTS (
        SELECT 1 FROM information_schema.tables 
        WHERE table_schema = 'public' 
        AND table_name = 'tk_tcus_change'
    ) THEN
        -- 检查字段是否存在且类型不同
        IF EXISTS (
            SELECT 1 FROM information_schema.columns 
            WHERE table_name = 'tk_tcus_change' 
            AND column_name = 'fk_tcus_largetextfield'
            AND data_type != 'character varying'
            OR character_maximum_length != 2000
        ) THEN
            ALTER TABLE tk_tcus_change 
            ALTER COLUMN fk_tcus_largetextfield TYPE VARCHAR(2000);
        END IF;
    ELSE
        RAISE NOTICE 'Table tk_tcus_change does not exist, skipping alter column';
    END IF;
END $$;
```

**方案四：分版本发布**
```
版本 1.0：仅包含建表脚本
├── tcus_26022501_shouhuo_table.sql（创建 tk_tcus_change 表）

版本 1.1：包含字段修改
├── tcus_26022501_shouhuo_table_alter.sql（修改字段类型）
```
先部署版本 1.0 创建表，再部署版本 1.1 修改字段。

### 预防措施

1. **SQL 脚本命名规范**
   ```
   建表脚本：{module}_{version}_table.sql
   改表脚本：{module}_{version}_table_alter.sql
   索引脚本：{module}_{version}_index.sql
   数据脚本：{module}_{version}_data.sql
   ```

2. **执行顺序约定**
   ```xml
   <dbscripts>
       <!-- seq 10-99：建表 -->
       <script file="dbschema/*_table.sql" seq="10-99"/>
       <!-- seq 100-199：修改表结构 -->
       <script file="dbschema/*_alter.sql" seq="100-199"/>
       <!-- seq 200-299：创建索引 -->
       <script file="dbschema/*_index.sql" seq="200-299"/>
       <!-- seq 300+：初始化数据 -->
       <script file="initdata/*_data.sql" seq="300+"/>
   </dbscripts>
   ```

3. **发布前自检清单**
   ```bash
   # SQL 依赖检查脚本
   #!/bin/bash
   
   # 检查 ALTER TABLE 是否有对应的 CREATE TABLE
   grep -r "ALTER TABLE" dbschema/ | while read line; do
       table=$(echo $line | grep -oP 'ALTER TABLE \K\w+')
       if ! grep -r "CREATE TABLE.*$table" dbschema/ > /dev/null; then
           echo "警告：表 $table 的 ALTER 语句缺少 CREATE TABLE"
       fi
   done
   ```

4. **使用 MC 的 SQL 验证功能**
   - 在 MC 中导出补丁包时，使用 "SQL 语法检查" 功能
   - 在测试环境进行预部署验证
   - 检查 SQL 执行顺序是否合理

5. **代码审查要点**
   - [ ] 新增 ALTER 语句时，确认目标表已存在
   - [ ] 检查 SQL 文件在配置文件中的执行顺序
   - [ ] 确认跨版本升级场景的兼容性
   - [ ] 验证 SQL 在干净环境和升级环境都能正常执行

6. **SQL 防御性编程**
   ```sql
   -- 使用 IF EXISTS / IF NOT EXISTS 避免重复执行报错
   CREATE TABLE IF NOT EXISTS tk_tcus_change (...);
   DROP INDEX IF EXISTS idx_tcus_change_number;
   ALTER TABLE tk_tcus_change DROP COLUMN IF EXISTS f_old_column;
   ```

---

## 案例八：MC补丁包表单主键ID冲突（扩展应用）

### 问题现象
MC安装补丁包时，安装日志报错：
```
traceId:7fc714c381c41872, deploy error dym file:dldk_moya_sa_exce_inh_ext.dym
kd.bos.exception.KDException: ERROR: duplicate key value violates unique constraint "t_meta_formdesign_fnumber_key"
  详细：Key (fnumber)=(dldk_moya_sa_exce_inh_ext) already exists.
RequestContext: tenantId=dkgj.test, accountId=2408871960164532224
TX: TXContext.554459:tag=DesignFormMeta.save, level=1, propagation=REQUIRES_NEW
...
Caused by: java.sql.SQLException: ERROR: duplicate key value violates unique constraint "t_meta_formdesign_fnumber_key"
  详细：Key (fnumber)=(dldk_moya_sa_exce_inh_ext) already exists.
```

### 分析过程

**1. 分析错误信息**
- 错误类型：`duplicate key value violates unique constraint`
- 冲突表：`T_META_FORMDESIGN`（主表，非多语言表）
- 冲突约束：`t_meta_formdesign_fnumber_key`
- 冲突字段：`fnumber`
- 冲突值：`dldk_moya_sa_exce_inh_ext`
- 操作类型：INSERT（通过 `DesignFormMeta.save` 和 `BatchInsertTask` 确认）

**2. 理解业务场景**
- 表单编码：`dldk_moya_sa_exce_inh_ext`
- 这是一个扩展应用中的表单（从命名 `_ext` 后缀推断）
- 补丁包尝试插入新的表单元数据
- 目标环境已存在相同编码的表单

**3. 追溯变更历史**
与案例六类似，但场景更复杂：
- 开发环境创建了扩展表单 `dldk_moya_sa_exce_inh_ext`
- 首次部署到生产环境成功
- 开发环境删除了该扩展应用（或重新创建扩展）
- 重新创建相同编码的扩展表单，分配了新的主键ID
- 新补丁包中包含新ID的表单元数据
- 目标环境仍保留旧ID的表单数据

**4. 关键区别：扩展应用 vs 标准应用**
```
标准应用表单：
- 存储在 T_META_FORMDESIGN
- 约束：idx_meta_formdsgn_l_number (fnumber, flocaleid) - 案例六

扩展应用表单：
- 同样存储在 T_META_FORMDESIGN
- 约束：t_meta_formdesign_fnumber_key (fnumber) - 本案例
- 扩展表单通常有 _ext 后缀命名约定
```

**5. 堆栈分析**
```
DesignFormMeta.save          → 保存表单设计元数据
BatchInsertTask.execute      → 批量插入
DB.executeBatch              → 执行批量SQL
BaseDB.rethrow               → 捕获并重抛异常
SecureExceptionUtil.wrap     → 包装为KDException
```
确认是在保存表单设计元数据时触发的主键冲突。

### 根因
苍穹平台中，扩展应用的表单元数据同样使用 `fnumber` 作为唯一键。当扩展应用被删除后重新创建，即使表单编码相同，系统会分配新的主键ID。补丁包部署时，由于目标环境已存在相同编码但不同ID的表单记录，导致唯一约束冲突。

### 解决方案

**方案一：重新对齐开发环境与生产环境（推荐）**

按照标准流程恢复元数据一致性：

**步骤1：备份开发环境**
```bash
# 导出当前开发环境元数据（以防万一）
# 使用MC的元数据导出功能
# 或直接使用Git仓库备份
```

**步骤2：删除开发环境的扩展应用**
```
MC开发平台 → 应用管理 → 找到扩展应用 → 删除应用
注意：确保Git仓库中对应的元数据文件也已删除
```

**步骤3：从生产环境导入元数据**
```
1. 登录天梯平台
2. 找到上次发布的补丁提单
3. 下载补丁包（包含生产环境的元数据）
4. 在开发环境导入该补丁包
```

**步骤4：重新扩展表单**
```
1. 在导入的扩展应用下
2. 找到需要扩展的表单 dldk_moya_sa_exce_inh_ext
3. 重新执行扩展操作（此时会继承生产环境的ID）
4. 进行必要的扩展开发
```

**步骤5：重新发布**
```
1. 导出元数据到Git仓库
2. 构建新的补丁包
3. 推送到天梯平台
4. 部署到生产环境（此时ID一致，不会冲突）
```

**方案二：直接修改数据库（应急方案，不推荐）**
```sql
-- 仅在无法重新发布补丁时的应急处理
-- 注意：操作前必须备份数据！

-- 1. 查询生产环境现有记录
SELECT FID, FNUMBER, FNAME, FCREATETIME 
FROM T_META_FORMDESIGN 
WHERE FNUMBER = 'dldk_moya_sa_exce_inh_ext';
-- 记录FID值，假设为 100001

-- 2. 查询补丁包中的新ID
-- 从补丁包元数据文件中查看新FID，假设为 100002

-- 3. 更新补丁包中的元数据ID（将100002改为100001）
-- 或者删除生产环境旧记录（风险较高，谨慎操作）

-- 4. 如果确定要删除旧记录
DELETE FROM T_META_FORMDESIGN_L 
WHERE FID IN (SELECT FID FROM T_META_FORMDESIGN WHERE FNUMBER = 'dldk_moya_sa_exce_inh_ext');

DELETE FROM T_META_FORMDESIGN 
WHERE FNUMBER = 'dldk_moya_sa_exce_inh_ext';

-- 5. 重新部署补丁包
```

**方案三：使用MC的元数据覆盖功能**
```
MC管理控制台 → 应用管理 → 元数据管理
1. 搜索表单编码 dldk_moya_sa_exce_inh_ext
2. 选择"强制覆盖"模式
3. 上传补丁包重新部署
注意：此操作会丢失生产环境的表单数据，谨慎使用！
```

### 预防措施

1. **扩展应用开发规范**
   ```
   ✗ 禁止操作：
   - 删除已部署的扩展应用后重建
   - 通过删除+新建方式"重置"扩展
   
   ✓ 正确操作：
   - 扩展应用一旦部署，只能通过修改方式调整
   - 如需重大调整，使用版本号区分（_ext_v2）
   - 保持开发/测试/生产环境元数据ID一致
   ```

2. **元数据版本管理**
   ```bash
   # Git提交规范
   feat(ext): 扩展表单 dldk_moya_sa_exce_inh_ext 增加字段
   
   # 包含信息
   - 表单编码
   - 主键ID（FID）
   - 变更内容
   - 影响范围
   ```

3. **环境同步检查清单**
   ```markdown
   发布前确认：
   - [ ] 开发环境元数据ID与生产环境一致
   - [ ] 扩展应用未做过删除重建操作
   - [ ] Git仓库元数据与开发环境一致
   - [ ] 补丁包在测试环境验证通过
   ```

4. **MC部署前验证**
   ```sql
   -- 预检查脚本
   SELECT FNUMBER, FID, FNAME, FCREATETIME
   FROM T_META_FORMDESIGN
   WHERE FNUMBER IN (
       -- 补丁包中包含的表单编码
       'dldk_moya_sa_exce_inh_ext',
       'other_form_ext'
   );
   -- 对比结果与补丁包中的FID是否一致
   ```

5. **建立元数据基线**
   ```
   每个版本发布后：
   1. 从天梯下载发布的补丁包
   2. 保存为基线版本
   3. 新开发周期开始前，从基线恢复开发环境
   4. 确保开发环境始终与生产环境ID一致
   ```

6. **团队协作规范**
   ```
   多人协作时注意：
   - 禁止各自独立创建相同编码的扩展
   - 元数据变更必须通过Git同步
   - 定期同步开发环境（建议每周）
   - 重大变更前通知团队成员
   ```

---

## 案例九：苍穹开发环境文件下载乱码

### 问题现象
本地调试苍穹应用时，使用 `this.getView().download(url)` 方法下载文件，文件名出现乱码。

### 分析过程

**1. 问题定位**
- 现象：文件下载功能正常，但文件名显示乱码
- 代码：`this.getView().download(url)`
- 环境：本地IDEA开发环境
- 推测：字符编码问题

**2. 排查方向**
文件名乱码通常由以下原因导致：
- HTTP响应头编码设置不正确
- 浏览器解析编码与服务器不一致
- JVM启动参数未指定字符集
- IDE字符集配置问题

**3. 确认根因**
经排查，问题出在IDEA的Gradle配置：
- Gradle运行时的JVM未指定UTF-8编码
- 导致HTTP响应中的文件名编码错误
- 浏览器接收到错误的编码格式，显示乱码

### 根因
IDEA使用Gradle运行项目时，默认未指定`-Dfile.encoding=UTF-8`参数，导致JVM使用系统默认编码（Windows下通常是GBK），与HTTP协议要求的UTF-8编码不一致，造成文件名乱码。

### 解决方案

**步骤1：修改Gradle配置**
在项目根目录的 `gradle.properties` 文件中添加：
```properties
# 设置文件编码为UTF-8
systemProp.file.encoding=UTF-8
# 或者
org.gradle.jvmargs=-Dfile.encoding=UTF-8
```

**步骤2：刷新Gradle项目**
```
IDEA → Gradle面板 → 点击刷新按钮（Reload All Gradle Projects）
或者
./gradlew --stop  # 停止Gradle Daemon
./gradlew clean   # 清理构建缓存
```

**步骤3：重启工程**
```
IDEA → File → Invalidate Caches / Restart → Invalidate and Restart
```

**步骤4：验证修复**
```java
// 测试代码
String fileName = "测试文件_中文名称.xlsx";
String encodedName = URLEncoder.encode(fileName, StandardCharsets.UTF_8);
String url = "/download?fileName=" + encodedName;
this.getView().download(url);
```

### 预防措施

1. **项目初始化配置**
   新建苍穹项目时，在`gradle.properties`中预设编码：
   ```properties
   # 字符编码设置
   systemProp.file.encoding=UTF-8
   systemProp.sun.jnu.encoding=UTF-8
   
   # JVM参数
   org.gradle.jvmargs=-Dfile.encoding=UTF-8 -Xmx2048m
   
   # 控制台输出编码
   systemProp.stdout.encoding=UTF-8
   systemProp.stderr.encoding=UTF-8
   ```

2. **IDEA统一设置**
   ```
   File → Settings → Editor → File Encodings:
   - Global Encoding: UTF-8
   - Project Encoding: UTF-8
   - Default encoding for properties files: UTF-8
   - Transparent native-to-ascii conversion: ✓勾选
   
   File → Settings → Build, Execution, Deployment → Build Tools → Gradle:
   - Gradle JVM: 选择与项目一致的JDK，并确保JDK默认编码为UTF-8
   ```

3. **代码层防御**
   ```java
   // 下载方法中显式指定编码
   public void downloadFile(String fileName, byte[] content) {
       try {
           // 强制使用UTF-8编码文件名
           String encodedFileName = URLEncoder.encode(fileName, "UTF-8")
                   .replaceAll("\\+", "%20");
           
           // 设置响应头
           this.getView().addClientCall(
               ClientCall.of("setHeader", "Content-Disposition", 
                   "attachment; filename*=UTF-8''" + encodedFileName)
           );
           
           this.getView().download(new ByteArrayInputStream(content));
       } catch (UnsupportedEncodingException e) {
           logger.error("文件名编码失败", e);
           throw new KDException("文件下载失败");
       }
   }
   ```

4. **团队规范**
   ```markdown
   项目配置检查清单：
   - [ ] gradle.properties 包含 file.encoding=UTF-8
   - [ ] IDEA文件编码设置为UTF-8
   - [ ] 所有源代码文件使用UTF-8编码
   - [ ] 数据库连接字符串指定charset=utf8
   - [ ] 前端页面声明 charset=UTF-8
   ```

### 相关参考

- 金蝶云社区原文：[苍穹开发环境文件下载乱码问题排查](https://vip.kingdee.com/link/s/Z1wr8)

### 类似问题排查

如果上述方案未解决，可进一步检查：

1. **浏览器编码设置**
   ```
   Chrome: 设置 → 外观 → 自定义字体 → 编码 → Unicode (UTF-8)
   ```

2. **服务器环境变量**
   ```bash
   # Linux服务器
   export LANG=en_US.UTF-8
   export LC_ALL=en_US.UTF-8
   
   # 验证
   locale
   ```

3. **应用服务器配置**
   ```bash
   # Tomcat启动参数
   CATALINA_OPTS="-Dfile.encoding=UTF-8 -Dsun.jnu.encoding=UTF-8"
   ```

4. **数据库编码**
   ```sql
   -- 检查数据库编码
   SHOW VARIABLES LIKE 'character_set%';
   -- 确保 character_set_database 和 character_set_connection 为 utf8mb4
   ```

---

## 案例十：MC补丁包SQL语句缺少分号导致未执行

### 问题现象
MC显示补丁升级成功，但SQL文件中的UPDATE语句未执行，数据未更新。

### 错误信息
执行日志显示"sqlcount:0"，无报错信息。

### 分析过程
1. 检查日志确认SQL文件被读取但未执行
2. 排查SQL文件内容，发现语句缺少结束符
3. MC解析SQL时以分号为语句分隔符

### 根因
SQL语句末尾缺少分号【;】，导致MC不将其识别为完整SQL语句，因此跳过了执行。

### 解决方案
在SQL语句后增加分号，重新打包并安装补丁包。

```sql
-- 错误示例（缺少分号）
UPDATE t_meta_formdesign SET fstatus = 'A' WHERE fnumber = 'test_form'

-- 正确示例
UPDATE t_meta_formdesign SET fstatus = 'A' WHERE fnumber = 'test_form';
```

### 预防措施
1. 开发过程严格遵守苍穹SQL规范，确保每条SQL语句以分号结尾
2. 提交前使用SQL格式化工具检查语法
3. 补丁包发布前在测试环境验证SQL执行效果
4. 代码审查时重点关注SQL文件的语法完整性

**参考链接**：[金蝶云社区 - SQL分号问题](https://vip.kingdee.com/link/s/Z4hxs)

---

## 案例十一：MC补丁包同名元数据文件版本覆盖

### 问题现象
MC安装补丁后，环境表单仍为旧版本元数据。

### 错误信息
日志显示升级执行，但实际未更新到最新版本。

### 分析过程
1. 检查补丁包内容，发现包含两个Zip文件
2. 两个Zip中都包含同名元数据文件但版本不同
3. MC多线程并行执行，旧版本后执行覆盖了新版本

### 根因
并行升级时序冲突，补丁包中包含多个版本的同名元数据文件，旧版本元数据后执行导致覆盖新版本。

### 解决方案
删除补丁包中旧版本的元数据文件，仅保留最新版本，重新打包部署。

```bash
# 检查补丁包结构
unzip -l patch.zip | grep "form_meta"
# 发现重复文件：
# old_version/form_meta/test_form.dym
# new_version/form_meta/test_form.dym

# 删除旧版本，保留新版本
# 重新打包
```

### 预防措施
1. 构建补丁包时确保同名元数据文件唯一且为最新版本
2. 使用MC的"补丁包分析"功能检查重复文件
3. 建立补丁包构建检查清单：
   ```markdown
   - [ ] 检查是否包含重复元数据文件
   - [ ] 确认所有元数据文件为最新版本
   - [ ] 验证补丁包在干净环境正常部署
   ```
4. 版本控制：删除旧版本元数据后再提交新版本

**参考链接**：[金蝶云社区 - 元数据版本覆盖](https://vip.kingdee.com/link/s/Z2w60)

---

## 案例十二：MC补丁包版本号长度超限

### 问题现象
MC安装二开补丁包后，升级日志无详细执行记录。

### 错误信息
日志表（T_LOG_DEPLOY）版本号字段长度限制为10，超长导致记录失败；三段版本号自动补".0"致总长超限。

### 分析过程
1. 服务正常运行，但补丁包部署后无执行日志
2. 排查发现补丁包版本号超过10字符无法写入日志表
3. 使用9字符三段格式（如7.01.002）时，系统自动追加".0"仍导致超长

### 根因
补丁包版本号长度超过数据库字段限制（10字符），且三段格式会被系统自动扩展为四段。

### 解决方案
修改补丁包版本号为四段格式，总长度不超过10位。

```
错误版本号：
- 7.01.002（三段，系统会转为7.01.002.0 = 10位，刚好超限）
- 7.01.002.001（四段但11位，超限）

正确版本号：
- 7.0.1.001（四段，9位，符合规范）
- 7.0.1.01（四段，8位，符合规范）
```

### 预防措施
1. 严格遵守"四段、总长≤10位"的版本号规范
2. 版本号格式：主版本.次版本.修订号.构建号
3. 构建补丁包前检查版本号长度：
   ```bash
   # 检查脚本
   version="7.0.1.001"
   if [ ${#version} -gt 10 ]; then
       echo "版本号长度超限：$version (${#version}位)"
       exit 1
   fi
   ```
4. 避免使用三段版本号，防止系统自动补位

**参考链接**：[金蝶云社区 - 版本号长度问题](https://vip.kingdee.com/link/s/Zpumk)

---

## 案例十三：复制数据中心后元数据缓存冲突

### 问题现象
复制数据中心B登录报错，原数据中心A正常。

### 分析过程
1. 两个数据中心环境配置相同
2. 原数据中心A登录正常，复制后的数据中心B登录报错
3. 怀疑是元数据缓存引起的问题

### 根因
复制数据中心操作导致旧元数据缓存冲突，复制后的数据中心仍保留原数据中心的缓存数据。

### 解决方案
访问元数据重建地址执行所有元数据重建：

```
1. 登录MC管理控制台
2. 进入：系统管理 → 元数据重建
3. 选择目标数据中心
4. 执行"重建所有元数据"
5. 等待重建完成，重新登录验证
```

或者直接访问重建URL：
```
http://{server}/ierp/metadata/rebuild
```

### 预防措施
1. 复制数据中心后，建议主动重建元数据以防缓存异常
2. 建立数据中心复制标准流程：
   ```markdown
   复制数据中心后必做：
   - [ ] 重建元数据缓存
   - [ ] 清理Redis缓存
   - [ ] 重启应用服务
   - [ ] 验证登录功能
   ```
3. 定期检查元数据一致性
4. 复制操作前备份原数据中心配置

**参考链接**：[金蝶云社区 - 数据中心复制缓存问题](https://vip.kingdee.com/link/s/ZUvYD)

---

## 案例十四：MC升级校验分库标识多语言数据重复

### 问题现象
MC升级校验报错【MC-010-001】Duplicate key mob。

### 错误信息
合法性校验不通过：分库标识列表mob有两行重复数据。

### 分析过程
1. 检查分库标识配置
2. 发现数据库多语言表中同一语言存在两行相同数据
3. 分库标识多语言表数据重复导致唯一约束冲突

### 根因
分库标识多语言表（如T_BAS_MOB_L）中存在重复数据，违反唯一约束。

### 解决方案
手工删除重复数据后重试：

```sql
-- 1. 查询重复数据
SELECT FMOBID, FLOCALEID, COUNT(*) as cnt
FROM T_BAS_MOB_L
GROUP BY FMOBID, FLOCALEID
HAVING COUNT(*) > 1;

-- 2. 删除重复数据（保留一条）
DELETE FROM T_BAS_MOB_L
WHERE FPkid IN (
    SELECT FPkid FROM (
        SELECT FPkid, ROW_NUMBER() OVER (PARTITION BY FMOBID, FLOCALEID ORDER BY FPkid) as rn
        FROM T_BAS_MOB_L
    ) t WHERE rn > 1
);

-- 3. 或者使用CTE（PostgreSQL）
WITH duplicates AS (
    SELECT FPkid,
           ROW_NUMBER() OVER (PARTITION BY FMOBID, FLOCALEID ORDER BY FPkid) as rn
    FROM T_BAS_MOB_L
)
DELETE FROM T_BAS_MOB_L
WHERE FPkid IN (SELECT FPkid FROM duplicates WHERE rn > 1);
```

### 预防措施
1. 数据库层面添加唯一约束防止重复数据
   ```sql
   ALTER TABLE T_BAS_MOB_L ADD CONSTRAINT uk_mob_l 
   UNIQUE (FMOBID, FLOCALEID);
   ```
2. 应用层插入前检查是否存在
3. 定期进行数据质量检查
4. 建立数据清理脚本，定期清理重复数据

**参考链接**：[金蝶云社区 - 分库标识重复](https://vip.kingdee.com/link/s/ZmcwS)

---

## 案例十五：MC上传补丁包Nginx密码过期

### 问题现象
MC上传二开补丁包失败。

### 错误信息
"java.io.IOException: Pipe closed"

### 分析过程
1. 检查机器管理，发现Nginx服务器"测试连接"显示失败
2. 登录虚拟机发现用户密码已过期
3. 密码过期导致登录需强制修改，连接失效

### 根因
虚拟机用户密码过期，登录需强制修改密码，导致MC与服务器之间的连接失效。

### 解决方案
1. 登录虚拟机修改密码
2. 在MC机器管理中更新密码
3. 保存配置并测试连接

```bash
# 1. SSH登录服务器修改密码
ssh user@nginx-server
passwd

# 2. 在MC中更新配置
# MC管理控制台 → 机器管理 → 选择Nginx服务器 → 更新密码 → 测试连接
```

### 预防措施
1. 定期监控服务器账户状态，避免密码过期影响运维连接
2. 设置密码过期提醒（提前30天通知）
3. 使用密钥认证替代密码认证，避免密码过期问题
4. 建立服务器账号管理台账：
   ```markdown
   | 服务器 | 账号 | 密码过期时间 | 状态 |
   |--------|------|-------------|------|
   | nginx-01 | admin | 2024-06-01 | 正常 |
   ```
5. 定期执行"测试连接"检查服务器状态

**参考链接**：[金蝶云社区 - Nginx密码过期](https://vip.kingdee.com/link/s/ZcQlc)

---

## 案例十六：轻量环境许可证缺失导致验证码发送失败

### 问题现象
轻量环境新用户首次登录改密时，提示"验证码发送失败"。

### 分析过程
1. 查询数据库发现 `t_lic_license` 表为空
2. 推测因未申请临时许可导致功能受限
3. 验证码功能依赖许可证验证

### 根因
环境缺少有效许可证（许可无效或未导入），导致系统功能受限，验证码服务无法正常发送。

### 解决方案
1. 导入有效许可数据至 `t_lic_license` 表
2. 重启苍穹服务

```sql
-- 导入许可证数据
INSERT INTO t_lic_license (FPkid, FLicenseCode, FValidDate, FStatus)
VALUES (NEXTVAL('seq_lic_license'), 'LICENSE_CODE', '2025-12-31', 'A');

-- 或者直接使用许可证导入工具
```

### 预防措施
1. 新环境创建后应及时申请并导入临时或正式许可证
2. 建立环境创建检查清单：
   ```markdown
   轻量环境初始化必做：
   - [ ] 导入许可证
   - [ ] 配置数据源
   - [ ] 初始化基础数据
   - [ ] 测试登录功能
   - [ ] 测试验证码发送
   ```
3. 设置许可证过期提醒
4. 定期检查许可证有效性

**参考链接**：[金蝶云社区 - 许可证缺失](https://vip.kingdee.com/link/s/ZlEyb)

---

## 案例十七：业务表名超长导致创建失败

### 问题现象
进入带组织基础资料列表报错。

### 错误信息
"创建数据表***USERE失败"

### 分析过程
1. 检查表名长度
2. 发现业务表名超过24位
3. 苍穹平台拼接后缀后超过30位数据库限制

### 根因
业务表名超过24位，系统拼接组织标识后缀后超过30位数据库表名长度限制。

### 解决方案
1. 修改表名至24位以内
2. 重构相关业务对象

```sql
-- 检查表名长度
SELECT LENGTH('T_BD_MATERIAL_ORG_USERE') FROM dual; -- 23位，符合
SELECT LENGTH('T_BD_MATERIAL_ORGANIZATION_USERE') FROM dual; -- 33位，超限

-- 修改表名
ALTER TABLE T_BD_MATERIAL_ORGANIZATION_USERE 
RENAME TO T_BD_MAT_ORG_USERE;
```

### 预防措施
1. 命名规范：业务表名不超过24位
2. 预留后缀空间：系统会自动添加组织标识（如_USERE、_ORG等）
3. 建立命名检查机制：
   ```java
   // 表名长度检查
   if (tableName.length() > 24) {
       throw new IllegalArgumentException("表名超过24位限制：" + tableName);
   }
   ```
4. 使用缩写规范：
   - MATERIAL → MAT
   - ORGANIZATION → ORG
   - USER → USR
   - DEPARTMENT → DEPT

**参考链接**：[金蝶云社区 - 表名超长](https://vip.kingdee.com/link/s/ZfKRE)

---

## 案例十八：开发环境热部署导致本地调试不生效

### 问题现象
IDEA修改插件代码不生效，需新增插件才生效；重启工程、删除jar包均无效。

### 分析过程
1. 排查系统表发现 `t_meta_ext_jar` 中存在二开插件数据
2. 系开发环境误用"热部署"安装补丁所致
3. 热部署jar包优先级高于本地调试jar包

### 根因
开发环境启用"热部署"后，系统优先加载热部署jar包，导致本地调试jar包不被加载，代码修改不生效。

### 解决方案
1. 手工删除表 `t_meta_ext_jar` 中相关数据
2. 重启苍穹服务

```sql
-- 查询热部署插件
SELECT * FROM t_meta_ext_jar 
WHERE FPluginCode LIKE '%your_plugin%';

-- 删除热部署记录
DELETE FROM t_meta_ext_jar 
WHERE FPluginCode = 'your_plugin_code';

-- 或者清空所有热部署记录（谨慎操作）
-- DELETE FROM t_meta_ext_jar;
```

### 预防措施
1. **"热部署"仅用于生产环境**，开发环境避免使用
2. 开发环境不可混合使用热部署与常规部署
3. 建立环境使用规范：
   ```markdown
   | 环境 | 部署方式 | 说明 |
   |------|---------|------|
   | 开发环境 | IDEA本地调试 | 禁用热部署 |
   | 测试环境 | MC常规部署 | 禁用热部署 |
   | 生产环境 | MC热部署 | 减少停机时间 |
   ```
4. 定期检查 `t_meta_ext_jar` 表，确保开发环境无热部署记录
5. 开发环境配置检查脚本：
   ```sql
   -- 检查开发环境是否误用热部署
   SELECT COUNT(*) as hot_deploy_count 
   FROM t_meta_ext_jar;
   -- 如果>0，提示开发人员清理
   ```

**参考链接**：[金蝶云社区 - 热部署冲突](https://vip.kingdee.com/link/s/Zh5n1)

---

## 案例十九：MC上传补丁包认证失败

### 问题现象
在MC上传补丁包时提示"上传补丁包失败，失败原因：Auth fail"。

### 错误信息
"Auth fail"（认证失败）

### 分析过程
1. 初判为目录权限不足，执行 `chmod -R 777` 授权后无效
2. 进一步排查Nginx机器配置
3. 发现访问账号密码已变更导致认证失败

### 根因
Nginx服务器配置的访问用户密码已修改，MC中保存的旧密码无法通过认证。

### 解决方案
1. 登录服务器确认当前密码
2. 在MC机器管理中同步更新密码配置
3. 测试连接验证

```bash
# 1. 确认服务器密码
ssh nginx-user@nginx-server

# 2. 在MC中更新
# MC管理控制台 → 机器管理 → 选择Nginx服务器 → 编辑 → 更新密码 → 保存

# 3. 点击"测试连接"验证
```

### 预防措施
1. 定期核查运维账号密码有效性
2. 确保MC与底层服务器凭证一致
3. 建立密码变更同步机制：
   ```markdown
   服务器密码变更流程：
   1. 修改服务器密码
   2. 同步更新MC机器管理配置
   3. 同步更新CI/CD配置
   4. 同步更新监控工具配置
   5. 验证所有连接正常
   ```
4. 使用统一密码管理工具（如Vault）
5. 设置密码变更通知，确保相关系统同步更新

**参考链接**：[金蝶云社区 - 认证失败](https://vip.kingdee.com/link/s/l0HJ1)

---

## 案例二十：MC补丁包部署报错：数据中心未配置数据源（分库标识缺失分析）

### 场景描述
在MC（Management Center）管理控制台部署数据模型补丁包时，执行到 SQL 脚本阶段报错，提示"数据中心未配置数据源"，导致部署中断。

### 典型错误特征
- **错误信息**：`java.lang.RuntimeException: java.lang.Exception: 数据中心未配置数据源: tenantId=xxx, accountId=xxx, routeKey=tsc`
- **涉及堆栈**：
  - `kd.bos.exception.ExceptionUtil.asRuntimeException`
  - `kd.bos.datasource.DataSourceManager.getSharedRouteKey`
  - `kd.bos.service.upgrade.deploy.DeployDataModelBase.deploySingleSql`
- **触发时机**：执行 `.sql` 脚本进行表结构调整或数据初始化时

### 根本原因
**分库标识缺失**。在苍穹的分库分表架构下，SQL 脚本的执行需要明确其对应的数据库实例。如果 SQL 脚本文件名或配置中未包含正确的分库标识（RouteKey），系统将无法定位到目标数据库，从而抛出"未配置数据源"的异常。

### 解决方案
1. **核对分库标识**：确认报错日志中的 `routeKey`（如截图中的 `tsc`）是否与应用预期的分库标识一致
2. **规范 SQL 文件命名**：
   - 确保 SQL 脚本文件名遵循规范：`应用编码_版本号_序号_分库标识_文件名.sql`
   - 例如：`p0g4_1.5_001_tsc_create_table.sql`
3. **调整补丁包结构**：
   - 在开发环境重新打包，确保导出脚本时选择了正确的"分库标识"
   - 检查 `deploy.xml` 中的脚本配置信息

### 参考资源
- 详细配置方法请参考金蝶云社区文章：[苍穹脚本执行报错：数据中心未配置数据源](https://vip.kingdee.com/link/s/ZUd5x)

### 预防措施
- **开发规范**：在开发阶段严格遵守 SQL 脚本命名规范，必须包含分库标识
- **环境检查**：部署前确认目标环境的数据中心已正确配置对应分库标识的数据源
- **自动化校验**：在补丁包构建阶段增加脚本命名合规性检查

---

## 案例二十一：MC补丁包部署报错：BalTbModifyTimeUpgrade not find（JAR包更新未重启分析）

### 场景描述
在MC（Management Center）管理控制台部署补丁包时，报错提示 `mservice: kd.bos.bal.mservice.BalTbModifyTimeUpgrade not find`，导致升级任务失败。

### 典型错误特征
- **错误信息**：`mservice: kd.bos.bal.mservice.BalTbModifyTimeUpgrade not find`
- **发生时机**：补丁包包含底层 JAR 包更新（如 `kd.bos.bal.mservice` 相关包）时

### 根本原因
**微服务未重启导致包加载异常**。
补丁升级过程中，如果涉及应用仓库（Repository）中底层 JAR 包的替换，由于 JVM 已加载旧版类信息或由于服务实例未感知到文件物理变更，导致新补丁中的类加载逻辑（如 `BalTbModifyTimeUpgrade` 类）无法被正确索引或拉取，从而报类找不到错误。

### 解决方案
1. **重启微服务**：在 MC 中重启报错涉及的微服务实例（通常是 `mservice` 或相关业务微服务）
2. **重新部署补丁**：服务重启完成后，再次触发补丁安装任务

### 参考资源
- 详细说明请参考金蝶云社区文章：[安装补丁报错：BalTbModifyTimeUpgrade not find](https://vip.kingdee.com/link/s/ZUdeh)

### 预防措施
- **前置操作规范**：在执行涉及底层 JAR 包替换的补丁升级前，务必预先重启相关微服务，清理 JVM 缓存并强制重新加载库文件
- **依赖检查**：升级前确认补丁包的依赖项是否已完整下载并解压到应用仓库指定目录
- **自动运维**：在高级部署流程中，增加"自动检测变更并滚动重启"的逻辑

---
