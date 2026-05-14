---
title: "Kafka 深度教程"
date: 2024-05-14
draft: false
categories: ["中间件"]
tags: ["Kafka", "消息队列", "分布式"]
author: "Jackray"
---

## Kafka 架构深度解析

### 核心概念

#### Broker
- Kafka节点，负责消息存储和转发
- 每个Broker有唯一ID
- 集群中至少3个Broker保证高可用

#### Topic
- 消息的逻辑分类
- 类似数据库的表
- 支持多消费者订阅

#### Partition
- Topic的物理分片
- 每个Partition是一个有序队列
- 分布在不同Broker上

```
Topic: orders
├─ Partition 0 (Broker 1)
├─ Partition 1 (Broker 2)
├─ Partition 2 (Broker 3)
```

#### Offset
- 消息在Partition中的位置标识
- 每条消息有唯一Offset
- 消费者通过Offset消费消息

---

## Kafka 核心原理

### 1. 消息存储机制

#### 日志段文件（Log Segment）
```
Partition存储结构：
/path/to/kafka-logs/topic-partition/
├─ 00000000000000000000.log  # 消息数据
├─ 00000000000000000000.index # 索引文件
├─ 00000000000000000000.timeindex # 时间索引
└─ 00000000000000000000.snapshot # 快照
```

#### 消息格式
```java
Message结构：
┌─────────────────────────────────────┐
│ OFFSET (8 bytes)                    │
│ SIZE (4 bytes)                      │
│ CRC32 (4 bytes)                     │
│ MAGIC (1 byte)                      │
│ ATTRIBUTES (1 byte)                 │
│ TIMESTAMP (8 bytes)                 │
│ KEY LENGTH (4 bytes)                │
│ KEY (variable)                      │
│ VALUE LENGTH (4 bytes)              │
│ VALUE (variable)                    │
└─────────────────────────────────────┘
```

### 2. 分区策略

#### 默认分区器
```java
public class DefaultPartitioner implements Partitioner {
    
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        // key为null：轮询分区
        if (keyBytes == null) {
            return roundRobinPartition(topic, cluster);
        }
        
        // key不为null：hash分区
        return Utils.murmur2(keyBytes) % cluster.partitionCountForTopic(topic);
    }
}
```

#### 自定义分区策略
```java
public class OrderPartitioner implements Partitioner {
    
    @Override
    public int partition(String topic, Object key, 
                         byte[] keyBytes, Cluster cluster) {
        if (key instanceof Order) {
            Order order = (Order) key;
            // 按订单类型分区
            return order.getType().hashCode() % 
                   cluster.partitionCountForTopic(topic);
        }
        return 0;
    }
}
```

### 3. 副本机制

#### ISR（In-Sync Replicas）
```
Partition副本分布：
Partition 0:
├─ Leader: Broker 1（处理读写）
├─ ISR: Broker 2, Broker 3（同步副本）
└─ OSR: Broker 4（非同步副本）
```

#### 副本同步流程
```
Producer → Leader Broker → 写入日志
                      ↓
              ISR副本拉取数据（Fetch）
                      ↓
              ISR副本写入日志
                      ↓
              Leader确认（HW更新）
                      ↓
              Producer收到ACK
```

#### HW（High Watermark）
- ISR副本同步完成的位置
- 消费者最多消费到HW位置
- 保证消息一致性

---

## Kafka 性能优化

### 1. 生产者优化

#### 批量发送
```java
ProducerConfig config = new ProducerConfig();
// 批量大小（16KB）
config.put("batch.size", 16384);
// 等待时间（5ms）
config.put("linger.ms", 5);
// 缓冲区大小（32MB）
config.put("buffer.memory", 33554432);
```

#### 压缩算法
```java
// 压缩类型
config.put("compression.type", "snappy"); 
// 或 "gzip", "lz4", "zstd"

性能对比：
- Snappy: 压缩率50%, 速度最快
- LZ4: 压缩率60%, 速度快
- Gzip: 压缩率70%, 速度慢
- Zstd: 压缩率65%, 平衡
```

#### ACK配置
```java
// ACK级别
config.put("acks", "all"); // 最可靠

acks选项：
- 0: 不等待ACK（最快，可能丢失）
- 1: 等待Leader ACK（平衡）
- all/-1: 等待ISR所有副本ACK（最可靠）
```

### 2. 消费者优化

#### 消费组配置
```java
ConsumerConfig config = new ConsumerConfig();
// 消费组ID
config.put("group.id", "order-consumer-group");
// 自动提交间隔
config.put("auto.commit.interval.ms", 5000);
// 手动提交（推荐）
config.put("enable.auto.commit", false);
```

#### 消费性能优化
```java
// 单次拉取数量
config.put("max.poll.records", 500);
// 拉取间隔（避免频繁拉取）
config.put("fetch.max.wait.ms", 500);
// 拉取最小字节数
config.put("fetch.min.bytes", 1);
```

### 3. Broker优化

#### JVM配置
```bash
# Kafka Broker启动参数
export KAFKA_HEAP_OPTS="-Xms6g -Xmx6g"
export KAFKA_JVM_PERFORMANCE_OPTS="-XX:+UseG1GC -XX:MaxGCPauseMillis=20"
```

#### 系统参数
```bash
# 文件描述符
ulimit -n 100000

# TCP优化
net.core.somaxconn = 1024
net.ipv4.tcp_max_syn_backlog = 1024
```

---

## Kafka 高可用

### 1. 集群部署

#### 推荐配置
```
生产环境推荐：
- Broker数量：至少3个（推荐5-7个）
- 副本因子：至少3（推荐ISR=2+）
- ISR最小数量：min.insync.replicas = 2
```

#### 跨机房部署
```
集群拓扑：
北京机房：
├─ Broker 1
├─ Broker 2
└─ Broker 3

上海机房：
├─ Broker 4
├─ Broker 5
└─ Broker 6

Partition分配策略：
- Leader在北京
- ISR副本在上海（跨机房同步）
```

### 2. 故障恢复

#### Leader选举
```java
// Controller检测Broker故障
Controller.watchBrokerFailure();

// 触发Leader选举
if (brokerFailed) {
    // 从ISR中选择新Leader
    newLeader = selectFromISR(partition);
    // 更新元数据
    updateMetadata(newLeader);
    // 通知所有Broker
    notifyBrokers(newLeader);
}
```

#### 消费者故障转移
```java
// 消费者Rebalance
ConsumerCoordinator.handleRebalance();

流程：
1. 消费者故障
2. Coordinator检测心跳超时
3. 触发Rebalance
4. 重新分配Partition
5. 其他消费者接管Partition
```

---

## Kafka 消息可靠性

### 1. 生产者端

#### 幂等性生产者
```java
ProducerConfig config = new ProducerConfig();
// 开启幂等性
config.put("enable.idempotence", true);

原理：
- ProducerID + SequenceNumber唯一标识
- Broker拒绝重复消息
- 保证单Partition幂等
```

#### 事务消息
```java
// 开启事务
KafkaProducer producer = new KafkaProducer(config);

producer.beginTransaction();
try {
    producer.send(new ProducerRecord("topic1", "msg1"));
    producer.send(new ProducerRecord("topic2", "msg2"));
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

### 2. 消费者端

#### 消息不丢失
```java
ConsumerConfig config = new ConsumerConfig();
// 关闭自动提交
config.put("enable.auto.commit", false);

// 手动提交Offset
while (true) {
    ConsumerRecords<String, String> records = 
        consumer.poll(Duration.ofMillis(100));
    
    for (ConsumerRecord record : records) {
        // 处理消息
        processMessage(record);
    }
    
    // 处理完成后手动提交
    consumer.commitSync();
}
```

#### 消息不重复
```java
// 业务层幂等
public void processMessage(ConsumerRecord record) {
    String messageId = record.key();
    
    // 检查是否已处理
    if (!processedCache.contains(messageId)) {
        // 处理消息
        doProcess(record);
        // 标记已处理
        processedCache.add(messageId);
    }
}
```

---

## Kafka 监控

### 1. 关键指标

#### Broker指标
```
- MessagesInPerSec: 消息写入速率
- BytesInPerSec: 字节写入速率
- BytesOutPerSec: 字节读取速率
- UnderReplicatedPartitions: 未同步分区数
- OfflinePartitionsCount: 离线分区数
- ActiveControllerCount: 活跃Controller数
```

#### Topic指标
```
- MessagesInPerSec: Topic消息速率
- BytesInPerSec: Topic字节速率
- ProduceMessageConversionsPerSec: 消息转换速率
- FetchMessageConversionsPerSec: 消息拉取转换
```

#### 消费者指标
```
- records-lag-max: 最大消费延迟
- records-consumed-rate: 消费速率
- bytes-consumed-rate: 字节消费速率
- commit-rate: 提交速率
```

### 2. 监控工具

#### Prometheus + Grafana
```yaml
# Prometheus配置
scrape_configs:
  - job_name: 'kafka'
    static_configs:
      - targets: ['kafka-broker:7071']

# Grafana Dashboard
推荐Dashboard:
- Kafka Broker Dashboard
- Kafka Topic Dashboard
- Kafka Consumer Dashboard
```

#### Kafka自带工具
```bash
# 查看Topic详情
kafka-topics.sh --describe --topic orders \
  --bootstrap-server localhost:9092

# 查看消费组
kafka-consumer-groups.sh --describe \
  --group order-consumer-group \
  --bootstrap-server localhost:9092

# 查看Broker指标
kafka-broker-api-versions.sh \
  --bootstrap-server localhost:9092
```

---

## Kafka 实战案例

### 1. 高吞吐场景

#### 配置
```java
ProducerConfig config = new ProducerConfig();
// 批量大小（1MB）
config.put("batch.size", 1048576);
// 等待时间（10ms）
config.put("linger.ms", 10);
// 缓冲区（64MB）
config.put("buffer.memory", 67108864);
// 压缩
config.put("compression.type", "lz4");
// ACK=1（平衡可靠性和性能）
config.put("acks", "1");
```

#### 性能
```
单Broker吞吐：
- 单Partition: 10万条/秒
- 多Partition: 50万条/秒
- 压缩后: 100万条/秒

集群吞吐（5 Broker）：
- 总吞吐: 500万条/秒
```

### 2. 低延迟场景

#### 配置
```java
ProducerConfig config = new ProducerConfig();
// 批量大小（1KB）
config.put("batch.size", 1024);
// 等待时间（0ms）
config.put("linger.ms", 0);
// ACK=all
config.put("acks", "all");
// 发送超时（30秒）
config.put("request.timeout.ms", 30000);
```

#### 性能
```
延迟：
- Producer到Broker: 2-5ms
- Broker到Consumer: 5-10ms
- 端到端: 10-20ms
```

---

## Kafka vs RabbitMQ vs RocketMQ

### 对比表格

| 特性 | Kafka | RabbitMQ | RocketMQ |
|------|-------|----------|----------|
| **吞吐量** | 百万级 | 万级 | 十万级 |
| **延迟** | ms级 | μs级 | ms级 |
| **可靠性** | 高 | 高 | 高 |
| **顺序性** | Partition内有序 | 队列有序 | 队列有序 |
| **事务** | 支持 | 不支持 | 支持 |
| **消息回溯** | 支持 | 不支持 | 支持 |
| **适用场景** | 大数据、日志 | 传统MQ | 金融、交易 |

---

## 总结

### Kafka核心优势
1. 高吞吐（百万级QPS）
2. 分布式架构（易扩展）
3. 消息持久化（可回溯）
4. 流式计算集成（Kafka Streams）

### Kafka适用场景
- 日志收集系统
- 大数据处理管道
- 实时数据流
- 事件驱动架构

### 学习建议
1. 官方文档：https://kafka.apache.org/documentation/
2. 源码阅读：https://github.com/apache/kafka
3. 实践项目：搭建集群、性能测试
4. 进阶：Kafka Streams、Kafka Connect

---

## 参考资料

- [Kafka官方文档](https://kafka.apache.org/documentation/)
- [Kafka源码](https://github.com/apache/kafka)
- 《深入理解Kafka》
- 《Kafka权威指南》