---
title: "RocketMQ 深度教程"
date: 2024-05-14
draft: false
categories: ["中间件"]
tags: ["RocketMQ", "消息队列", "阿里"]
author: "Jackray"
---

## RocketMQ 架构深度解析

### 核心组件

#### Producer（生产者）
- 发送消息的应用
- 支持多种发送方式
- 与NameServer建立连接获取路由信息

#### Consumer（消费者）
- 接收消息的应用
- 支持Push和Pull模式
- 与NameServer建立连接获取路由信息

#### Broker
- 消息服务器，负责存储和转发
- Master（主）和Slave（从）节点
- 每个Broker与NameServer保持心跳

#### NameServer
- 路由注册中心
- 类似注册中心
- 无状态设计，节点间无通信

### 架构拓扑

```
Producer集群
    ↓
NameServer集群（路由中心）
    ↓
Broker集群（消息存储）
├─ Broker-Master-A
├─ Broker-Slave-A
├─ Broker-Master-B
└─ Broker-Slave-B
    ↓
Consumer集群
```

---

## RocketMQ 核心概念

### 1. Topic

#### 定义
- 消息第一级分类
- 生产者发送消息的类别
- 消费者订阅消息的类别

#### Topic与Queue关系
```
Topic: ORDER_TOPIC
├─ Queue 0 (Broker-Master-A)
├─ Queue 1 (Broker-Master-A)
├─ Queue 2 (Broker-Master-B)
└─ Queue 3 (Broker-Master-B)

每个Topic默认4个Queue
```

### 2. Queue（消息队列）

#### 定义
- Topic的分片
- 消息存储的物理队列
- 每个Queue有序

#### Queue作用
```
1. 提高并发能力（多Queue并行）
2. 保证顺序性（单Queue有序）
3. 数据分片存储
```

### 3. Message

#### 消息结构
```java
Message {
    topic: "ORDER_TOPIC",       // Topic
    tags: "TAG_ORDER_CREATE",   // Tag（第二级分类）
    keys: "ORDER_123456",       // 消息Key（查询用）
    body: "订单内容",            // 消息体
    delayTimeLevel: 0,          // 延迟级别
    waitStoreMsgOK: true        // 是否等待存储确认
}
```

### 4. Consumer Group

#### 定义
- 消费者分组
- 同组内消费者负载均衡
- 不同组消费者广播消费

#### 消费模式
```
集群模式（Clustering）：
- 同组消费者分担Queue
- 每条消息只被消费一次

广播模式（Broadcasting）：
- 同组消费者都收到消息
- 每条消息被消费多次
```

---

## RocketMQ 核心原理

### 1. 消息发送流程

#### 发送步骤
```
1. Producer连接NameServer
2. 获取Topic路由信息
3. 选择Queue
4. 连接Broker
5. 发送消息
6. 等待Broker响应
7. 返回发送结果
```

#### 发送方式

##### 同步发送
```java
DefaultMQProducer producer = new DefaultMQProducer("producer_group");
producer.start();

Message msg = new Message("TopicTest", "Hello RocketMQ".getBytes());
SendResult result = producer.send(msg);

System.out.println(result.getSendStatus()); // SEND_OK
```

##### 异步发送
```java
producer.send(msg, new SendCallback() {
    @Override
    public void onSuccess(SendResult result) {
        System.out.println("发送成功");
    }
    
    @Override
    public void onException(Throwable e) {
        System.out.println("发送失败");
    }
});
```

##### 单向发送（One-way）
```java
// 不等待响应，最快但可能丢失
producer.sendOneway(msg);
```

### 2. 消息存储原理

#### CommitLog（提交日志）
```
存储结构：
/root/store/commitlog/
├─ 00000000000000000000.log
├─ 00000000000000000001.log
└─ ...

特点：
- 所有消息顺序写入
- 单文件1GB
- Append-only（追加写）
```

#### ConsumeQueue（消费队列）
```
存储结构：
/root/store/consumequeue/TopicTest/0/
├─ 00000000000000000000
├─ 00000000000000000001
└─ ...

内容：
每条记录24字节：
├─ CommitLog Offset (8字节)
├─ Message Size (4字节)
├─ Tags Code (8字节)
```

#### IndexFile（索引文件）
```
存储结构：
/root/store/index/
├─ 00000000000000000000
└─ ...

用途：
- 按Key查询消息
- 按时间范围查询
```

### 3. 消息消费流程

#### Pull模式
```java
DefaultMQPullConsumer consumer = new DefaultMQPullConsumer("consumer_group");
consumer.start();

while (true) {
    // 拉取消息
    PullResult result = consumer.pull("TopicTest", "*", 0, 32);
    
    if (result.getPullStatus() == PullStatus.FOUND) {
        List<MessageExt> msgs = result.getMsgFoundList();
        // 处理消息
        for (MessageExt msg : msgs) {
            process(msg);
        }
        // 提交Offset
        consumer.updateConsumeOffset("TopicTest", 0, result.getNextBeginOffset());
    }
}
```

#### Push模式（推荐）
```java
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("consumer_group");
consumer.subscribe("TopicTest", "*");

consumer.registerMessageListener(new MessageListenerConcurrently() {
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(
        List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
        // 处理消息
        for (MessageExt msg : msgs) {
            process(msg);
        }
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
});

consumer.start();
```

#### Push模式原理
```
内部仍然是Pull：
Consumer → Pull Request → Broker
Broker → 返回消息（或空）
Consumer → 处理消息
Consumer → 长轮询（等待新消息）
```

---

## RocketMQ 特殊消息

### 1. 顺序消息

#### 全局顺序消息
```java
// 单Queue
producer.send(msg, new MessageQueueSelector() {
    @Override
    public MessageQueue select(List<MessageQueue> mqs, 
                               Message msg, Object arg) {
        return mqs.get(0); // 固定选择Queue 0
    }
}, null);
```

#### 分区顺序消息
```java
// 按Key分区（如订单ID）
producer.send(msg, new MessageQueueSelector() {
    @Override
    public MessageQueue select(List<MessageQueue> mqs, 
                               Message msg, Object arg) {
        Long orderId = (Long) arg;
        long index = orderId % mqs.size();
        return mqs.get((int)index);
    }
}, orderId);
```

### 2. 事务消息

#### 事务流程
```
1. 发送半消息（Half Message）
2. 执行本地事务
3. 提交或回滚消息

半消息：
- 暂存，不可消费
- 等待事务确认
```

#### 事务消息实现
```java
TransactionMQProducer producer = new TransactionMQProducer("transaction_group");
producer.setTransactionListener(new TransactionListener() {
    
    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        // 执行本地事务
        try {
            doLocalTransaction();
            return LocalTransactionState.COMMIT_MESSAGE; // 提交
        } catch (Exception e) {
            return LocalTransactionState.ROLLBACK_MESSAGE; // 回滚
        }
    }
    
    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        // 事务回查
        if (transactionSuccess(msg)) {
            return LocalTransactionState.COMMIT_MESSAGE;
        }
        return LocalTransactionState.ROLLBACK_MESSAGE;
    }
});

producer.start();

// 发送事务消息
Message msg = new Message("TopicTest", "Body".getBytes());
TransactionSendResult result = producer.sendMessageInTransaction(msg, null);
```

### 3. 延迟消息

#### 延迟级别
```
RocketMQ预设延迟级别：
1s, 5s, 10s, 30s, 1m, 2m, 3m, 4m, 5m, 6m, 7m, 8m, 9m, 
10m, 20m, 30m, 1h, 2h

级别对应：
delayTimeLevel=1 → 1秒
delayTimeLevel=2 → 5秒
...
```

#### 发送延迟消息
```java
Message msg = new Message("TopicTest", "Body".getBytes());
msg.setDelayTimeLevel(3); // 10秒后消费

producer.send(msg);
```

---

## RocketMQ 高可用

### 1. Broker主从架构

#### 主从复制
```
Broker-Master:
├─ 写入消息
├─ 同步给Slave
└─ 返回ACK给Producer

Broker-Slave:
├─ 接收Master同步
├─ 写入CommitLog
└─ 不对外服务（或只读）
```

#### 同步复制 vs 异步复制
```bash
# 配置（broker.conf）
brokerRole=SYNC_MASTER   # 同步复制
brokerRole=ASYNC_MASTER  # 异步复制
brokerRole=SLAVE         # Slave节点

区别：
同步复制：
- Master等待Slave确认
- 数据可靠，性能稍低

异步复制：
- Master不等待Slave
- 性能高，可能丢失数据
```

### 2. 故障转移

#### Master故障
```
场景：Broker-Master宕机

处理：
1. Slave继续提供读服务
2. 不接受新写入
3. 手动切换或自动切换（Dledger）
```

#### Dledger模式（RocketMQ 4.5+）
```bash
# 配置
enableDLegerCommitLog=true
dLedgerGroup=group1
dLedgerPeers=n0@broker1:10912;n1@broker2:10912;n2@broker3:10912
dLedgerSelfId=n0

特点：
- 自动Leader选举
- 自动故障转移
- 类似Raft协议
```

### 3. NameServer高可用

#### 多节点部署
```
推荐：至少3个NameServer

启动：
nohup sh mqnamesrv -n "192.168.1.1:9876;192.168.1.2:9876;192.168.1.3:9876"

特点：
- 无状态
- 节点间无通信
- 客户端连接所有节点
```

---

## RocketMQ 性能优化

### 1. Producer优化

#### 批量发送
```java
List<Message> messages = new ArrayList<>();
for (int i = 0; i < 100; i++) {
    messages.add(new Message("TopicTest", ("Msg" + i).getBytes()));
}

SendResult result = producer.send(messages);
```

#### 发送优化配置
```java
producer.setMaxMessageSize(4 * 1024 * 1024);  // 单消息最大4MB
producer.setRetryTimesWhenSendFailed(3);      // 重试次数
producer.setSendMsgTimeout(3000);              // 超时时间
```

### 2. Broker优化

#### JVM配置
```bash
# runbroker.sh
JAVA_OPT="${JAVA_OPT} -server -Xms8g -Xmx8g -Xmn4g"
JAVA_OPT="${JAVA_OPT} -XX:+UseG1GC -XX:G1HeapRegionSize=16m"
JAVA_OPT="${JAVA_OPT} -XX:MaxGCPauseMillis=50"
```

#### 系统参数
```bash
# 内存映射
ulimit -n 100000

# PageCache
vm.dirty_ratio=60
vm.dirty_background_ratio=30
```

#### Broker配置
```bash
# broker.conf
flushCommitLogTimed=false       # 异步刷盘
flushCommitLogIntervel=500      # 刷盘间隔
messageStoreMaxLogSize=1024     # CommitLog文件大小
```

### 3. Consumer优化

#### 消费优化配置
```java
consumer.setConsumeThreadMin(20);       // 最小消费线程
consumer.setConsumeThreadMax(64);       // 最大消费线程
consumer.setPullBatchSize(32);          // 单次拉取数量
consumer.setConsumeMessageBatchMaxSize(16); // 单次消费数量
```

---

## RocketMQ 监控

### 1. 关键指标

#### Producer指标
```
- sendMsgNums: 发送消息数
- sendMsgSize: 发送消息大小
- sendMsgRT: 发送延迟
- sendFailedNums: 发送失败数
```

#### Broker指标
```
- putMsgNums: 写入消息数
- putMsgSize: 写入消息大小
- getMsgNums: 读取消息数
- commitLogSize: CommitLog大小
- consumeQueueSize: ConsumeQueue大小
```

#### Consumer指标
```
- consumeMsgNums: 消费消息数
- consumeMsgRT: 消费延迟
- consumeFailedNums: 消费失败数
- pullMsgNums: 拉取消息数
```

### 2. 监控工具

#### RocketMQ Console
```bash
# 启动Console
java -jar rocketmq-console-ng-2.0.0.jar \
  --server.port=8080 \
  --rocketmq.config.namesrvAddr=127.0.0.1:9876

访问：http://localhost:8080
```

#### Prometheus + Grafana
```yaml
# RocketMQ Exporter
rocketmq_exporter:
  image: apache/rocketmq-exporter:latest
  environment:
    NAMESRV_ADDR: 127.0.0.1:9876
```

---

## RocketMQ vs Kafka

### 对比

| 特性 | RocketMQ | Kafka |
|------|----------|-------|
| **吞吐量** | 十万级 | 百万级 |
| **延迟** | ms级 | ms级 |
| **顺序性** | Queue内有序 | Partition内有序 |
| **事务** | 支持 | 支持 |
| **延迟消息** | 支持 | 不支持（需外部） |
| **消息回溯** | 支持 | 支持 |
| **过滤** | Tag过滤 | 不支持 |
| **适用场景** | 金融、电商 | 大数据、日志 |

---

## RocketMQ 实战案例

### 1. 订单系统

#### 场景
```
订单创建 → 多系统通知：
├─ 库存系统（扣减库存）
├─ 支付系统（创建支付单）
├─ 物流系统（创建物流单）
└─ 用户系统（积分奖励）
```

#### 实现
```java
// 订单服务发送消息
Message msg = new Message("ORDER_TOPIC", 
                          "ORDER_CREATE", 
                          orderId.toString(),
                          orderJson.getBytes());
producer.send(msg);

// 各系统订阅
consumer.subscribe("ORDER_TOPIC", "ORDER_CREATE");
```

### 2. 分布式事务

#### 场景
```
下单 + 扣库存（分布式事务）：
1. 创建订单（半消息）
2. 扣减库存（本地事务）
3. 提交消息
4. 订单服务消费消息
```

#### 实现
```java
// 库存服务（事务消息）
producer.sendMessageInTransaction(msg, null);

// 本地事务：扣减库存
@Override
public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
    try {
        deductInventory();
        return LocalTransactionState.COMMIT_MESSAGE;
    } catch (Exception e) {
        return LocalTransactionState.ROLLBACK_MESSAGE;
    }
}
```

---

## 总结

### RocketMQ核心优势
1. 低延迟（毫秒级）
2. 高可靠（同步复制）
3. 顺序消息
4. 事务消息
5. 延迟消息
6. 消息过滤

### RocketMQ适用场景
- 金融交易系统
- 电商订单系统
- 分布式事务
- 实时通知系统

### 学习建议
1. 官方文档：https://rocketmq.apache.org/docs/
2. 源码阅读：https://github.com/apache/rocketmq
3. 实践项目：电商订单、分布式事务
4. 进阶：集群部署、性能调优

---

## 参考资料

- [RocketMQ官方文档](https://rocketmq.apache.org/docs/)
- [RocketMQ源码](https://github.com/apache/rocketmq)
- 《RocketMQ技术内幕》
- 《RocketMQ实战指南》