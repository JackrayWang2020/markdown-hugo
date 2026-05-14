---
title: "Redis 深度教程"
date: 2024-05-14
draft: false
categories: ["中间件"]
tags: ["Redis", "缓存", "分布式"]
author: "Jackray"
---

## Redis 架构深度解析

### 核心数据结构

#### String（字符串）
```c
// Redis源码定义
struct sdshdr {
    int len;      // 字符串长度
    int free;     // 未使用空间
    char buf[];   // 字符数组
};
```

**应用场景**：
- 缓存用户信息
- 计数器（incr）
- 分布式锁（setnx）

#### Hash（哈希）
```c
// Hash表结构
typedef struct dict {
    dictType *type;
    dictht ht[2];  // 两个Hash表（用于rehash）
    int rehashidx; // rehash进度
} dict;
```

**应用场景**：
- 用户信息（多字段）
- 商品详情
- 对象存储

#### List（列表）
```c
// QuickList结构（Redis 3.2+）
typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;
    unsigned long len;
} quicklist;
```

**应用场景**：
- 消息队列（LPUSH + BRPOP）
- 最新列表
- 关注列表

#### Set（集合）
```c
// Set底层实现
// 小集合：intset（整数集合）
// 大集合：dict（哈希表）
```

**应用场景**：
- 标签系统
- 共同好友
- 唯一性检查

#### ZSet（有序集合）
```c
// ZSet结构
typedef struct zset {
    dict *dict;        // 元素到分数映射
    zskiplist *zsl;    // 跳表（排序）
} zset;

// 跳表结构
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```

**应用场景**：
- 排行榜
- 范围查询
- 带权重的集合

---

## Redis 核心原理

### 1. 单线程架构

#### 为什么单线程快？
```
1. 纯内存操作（微秒级）
2. 非阻塞IO（epoll）
3. 避免线程切换开销
4. 无锁竞争

瓶颈：
- CPU：单核利用率
- 网络：带宽限制
- 内存：数据量限制
```

#### IO多路复用
```c
// Redis事件循环
while (!stop) {
    // epoll等待事件
    numEvents = epoll_wait(epfd, events, maxEvents, timeout);
    
    for (int i = 0; i < numEvents; i++) {
        // 处理事件
        if (events[i].mask & EPOLLIN) {
            readHandler(events[i].data);
        }
        if (events[i].mask & EPOLLOUT) {
            writeHandler(events[i].data);
        }
    }
}
```

### 2. 持久化机制

#### RDB（快照）
```bash
# 配置
save 900 1      # 900秒内至少1个key变化
save 300 10     # 300秒内至少10个key变化
save 60 10000   # 60秒内至少10000个key变化

# 手动触发
redis-cli BGSAVE  # 后台保存
redis-cli SAVE    # 前台保存（阻塞）
```

**原理**：
```
1. Redis fork子进程
2. 子进程写入临时RDB文件
3. 完成后替换旧RDB
4. 主进程继续服务
```

**优点**：
- 文件紧凑
- 恢复快
- 适合备份

**缺点**：
- 可能丢失数据（两次save间隔）
- fork开销（大数据集）

#### AOF（日志）
```bash
# 配置
appendonly yes               # 开启AOF
appendfsync everysec         # 每秒同步
appendfsync always           # 每次写入同步
appendfsync no               # 由操作系统决定

# AOF重写
auto-aof-rewrite-percentage 100  # 100%增长触发
auto-aof-rewrite-min-size 64mb    # 最小64MB
```

**原理**：
```
1. 每个写命令追加到AOF文件
2. AOF文件过大时触发重写
3. 重写：读取当前数据，生成最小命令集
4. 替换旧AOF文件
```

**优点**：
- 数据安全（最多丢失1秒）
- 可读性好

**缺点**：
- 文件大
- 恢复慢

#### 混合持久化（Redis 4.0+）
```bash
# 配置
aof-use-rdb-preamble yes

原理：
- 重写时：RDB格式 + AOF增量
- 恢复：先加载RDB，再执行AOF
```

---

## Redis 集群

### 1. 主从复制

#### 配置
```bash
# 主节点
redis-server --port 6379

# 从节点
redis-server --port 6380 --slaveof 127.0.0.1 6379

# 或在配置文件中
slaveof 127.0.0.1 6379
```

#### 复制流程
```
1. 从节点连接主节点
2. 发送SYNC命令
3. 主节点执行BGSAVE，生成RDB
4. 主节点发送RDB给从节点
5. 从节点加载RDB
6. 主节点发送增量命令
7. 持续同步
```

#### 复制优化
```bash
# 无盘复制（避免磁盘IO）
repl-diskless-sync yes
repl-diskless-sync-delay 5

# 部分同步（减少全量同步）
repl-backlog-size 1mb
```

### 2. 哨兵模式（Sentinel）

#### 配置
```bash
# sentinel.conf
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
```

#### 故障转移流程
```
1. Sentinel检测主节点下线（主观下线）
2. 多个Sentinel确认（客观下线）
3. 选举领头Sentinel
4. 领头Sentinel选举新主节点
5. 更新其他从节点配置
6. 通知客户端
```

#### Sentinel监控
```bash
# 查看主节点信息
redis-cli -p 26379 sentinel master mymaster

# 查看从节点信息
redis-cli -p 26379 sentinel slaves mymaster

# 查看Sentinel信息
redis-cli -p 26379 sentinel sentinels mymaster
```

### 3. Redis Cluster

#### 集群架构
```
集群节点：
├─ Node1 (Slot 0-5460)
├─ Node2 (Slot 5461-10922)
├─ Node3 (Slot 10923-16383)
└─ 每节点主从复制

Slot分配：
16384个槽位 → 平均分配到各节点
Key → CRC16(key) % 16384 → Slot → Node
```

#### 配置
```bash
# cluster.conf
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 15000
cluster-require-full-coverage yes

# 创建集群
redis-cli --cluster create \
  127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 \
  127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
  --cluster-replicas 1
```

#### 集群特性
```
优点：
- 自动分片
- 自动故障转移
- 可扩展

限制：
- 多key操作限制（需同一Slot）
- 事务限制
- 跨节点查询复杂
```

---

## Redis 性能优化

### 1. 内存优化

#### 数据结构优化
```redis
# 小对象使用ziplist
hash-max-ziplist-entries 512
hash-max-ziplist-value 64

list-max-ziplist-size -2
list-compress-depth 0

set-max-intset-entries 512

zset-max-ziplist-entries 128
zset-max-ziplist-value 64
```

#### 内存回收
```redis
# 过期策略
maxmemory-policy volatile-lru  # 推荐
# 或：allkeys-lru, volatile-lfu, allkeys-lfu

# 内存限制
maxmemory 4gb
```

#### 内存碎片
```bash
# 查看内存碎片率
redis-cli INFO memory

# 内存碎片率 > 1.5 需清理
redis-cli MEMORY PURGE
```

### 2. 性能瓶颈

#### CPU瓶颈
```bash
# 单线程限制
# 解决：
- 多实例部署
- Cluster分片
- 使用Redis 6多线程IO
```

#### 网络瓶颈
```bash
# 配置
# 批量操作（Pipeline）
# 本地部署（减少网络延迟）
```

#### 内存瓶颈
```bash
# 内存限制
maxmemory 16gb

# 数据分片
- Cluster
- 多实例
```

---

## Redis 应用场景

### 1. 缓存

#### 缓存策略
```java
// 缓存穿透
// 解决：缓存空值
if (value == null) {
    cache.set(key, "NULL", 5min);
}

// 缓存击穿
// 解决：互斥锁
String lockKey = "lock:" + key;
if (cache.setnx(lockKey, "1", 10sec)) {
    value = db.query(key);
    cache.set(key, value);
    cache.del(lockKey);
}

// 缓存雪崩
// 解决：过期时间随机
cache.set(key, value, randomExpire(300, 600));
```

#### 缓存一致性
```java
// 双删策略
public void updateData(String key, Object value) {
    cache.del(key);
    db.update(key, value);
    Thread.sleep(500ms);
    cache.del(key);
}

// 订阅Binlog
// Canal → MQ → Redis更新
```

### 2. 分布式锁

#### Redisson实现
```java
RedissonClient redisson = Redisson.create(config);
RLock lock = redisson.getLock("order:lock:" + orderId);

try {
    // 尝试加锁，最多等待100秒，锁10秒后自动释放
    if (lock.tryLock(100, 10, TimeUnit.SECONDS)) {
        // 执行业务
        doBusiness();
    }
} finally {
    lock.unlock();
}
```

#### 锁原理
```
加锁：
1. Lua脚本执行
2. exists key → 不存在 → setnx + expire
3. exists key → 存在 → 判断threadId
4. 同threadId → 重入计数+1
5. 不同threadId → 返回失败

解锁：
1. Lua脚本执行
2. 判断threadId
3. 同threadId → 重入计数-1
4. 计数=0 → del key + 发布解锁消息
```

### 3. 消息队列

#### List实现
```redis
# 生产者
LPUSH queue:message "message_data"

# 消费者（阻塞）
BRPOP queue:message 30  # 30秒超时
```

#### Stream实现（Redis 5.0+）
```redis
# 生产者
XADD stream:message * field1 value1 field2 value2

# 消费者
XREAD COUNT 10 BLOCK 2000 STREAMS stream:message $

# 消费组
XGROUP CREATE stream:message group1 $
XREADGROUP GROUP group1 consumer1 COUNT 10 BLOCK 2000 
  STREAMS stream:message >
```

### 4. 排行榜

#### ZSet实现
```redis
# 添加分数
ZADD leaderboard:game1 1000 user1
ZADD leaderboard:game1 950 user2

# 查询排名
ZREVRANK leaderboard:game1 user1  # 返回排名

# Top10
ZREVRANGE leaderboard:game1 0 9 WITHSCORES

# 分数范围
ZRANGEBYSCORE leaderboard:game1 900 1000 WITHSCORES
```

---

## Redis 监控

### 1. 关键指标

#### 性能指标
```
- instantaneous_ops_per_sec: 实时QPS
- hit_rate: 缓存命中率
- memory_fragmentation_ratio: 内存碎片率
- used_memory: 已用内存
- connected_clients: 连接数
- blocked_clients: 阻塞客户端数
```

#### 持久化指标
```
- rdb_last_bgsave_status: RDB状态
- rdb_last_bgsave_time_sec: RDB耗时
- aof_last_bgrewrite_status: AOF重写状态
- aof_current_size: AOF大小
```

#### 复制指标
```
- master_link_status: 主从连接状态
- master_sync_in_progress: 同步状态
- slave_repl_offset: 复制偏移量
```

### 2. 监控工具

#### Redis-cli INFO
```bash
# 查看所有信息
redis-cli INFO

# 查看内存信息
redis-cli INFO memory

# 查看统计信息
redis-cli INFO stats

# 查看持久化信息
redis-cli INFO persistence
```

#### Redis-cli MONITOR
```bash
# 实时监控命令
redis-cli MONITOR

# 输出
1598745678.123456 [0 127.0.0.1:54321] "SET" "key1" "value1"
```

#### Prometheus + Grafana
```yaml
# Redis Exporter
redis_exporter:
  image: oliver006/redis_exporter:latest
  environment:
    REDIS_ADDR: redis://redis:6379
```

---

## Redis 源码解析

### 1. 数据结构源码

#### String（SDS）
```c
// sds.c
struct sdshdr {
    uint32_t len;    // 已用长度
    uint32_t alloc;  // 总分配空间
    char flags;      // 类型标志
    char buf[];      // 字符数组
};

优点：
- O(1)获取长度
- 避免缓冲区溢出
- 减少内存分配
- 二进制安全
```

#### Hash（字典）
```c
// dict.c
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;
    struct dictEntry *next;  // 链地址法
} dictEntry;

Rehash：
- 渐进式rehash
- 维护两个表ht[0]、ht[1]
- 每次操作迁移部分元素
- 避免一次性迁移阻塞
```

#### ZSet（跳表）
```c
// server.h
typedef struct zskiplistNode {
    sds ele;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;
    } level[];
} zskiplistNode;

跳表优势：
- O(logN)查询
- 实现简单（相比平衡树）
- 范围查询高效
```

---

## Redis vs Memcached

### 对比

| 特性 | Redis | Memcached |
|------|-------|-----------|
| **数据结构** | 多种 | 仅String |
| **持久化** | 支持 | 不支持 |
| **集群** | 原生 | 需客户端 |
| **线程** | 单线程 | 多线程 |
| **内存效率** | 高 | 更高 |
| **适用场景** | 缓存+存储 | 纯缓存 |

---

## 总结

### Redis核心优势
1. 丰富的数据结构
2. 高性能（单线程+内存）
3. 持久化支持
4. 集群支持

### Redis适用场景
- 缓存系统
- 分布式锁
- 消息队列
- 排行榜
- 会话存储

### 学习建议
1. 官方文档：https://redis.io/documentation
2. 源码阅读：https://github.com/redis/redis
3. 实践项目：缓存、锁、队列
4. 进阶：集群部署、性能调优

---

## 参考资料

- [Redis官方文档](https://redis.io/documentation)
- [Redis源码](https://github.com/redis/redis)
- 《Redis设计与实现》
- 《Redis深度历险》