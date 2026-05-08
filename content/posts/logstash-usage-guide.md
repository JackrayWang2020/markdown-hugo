---
title: "Logstash 使用手册：从入门到实践"
date: 2026-05-08
draft: false
tags: ["Logstash", "ELK", "日志处理", "数据管道"]
categories: ["技术", "Logstash管理"]
summary: "Logstash 核心概念、配置语法、常用插件及实战示例的完整指南"
showToc: true
TocOpen: true
weight: 1
series: ["Logstash管理"]
---

## 一、Logstash 是什么？

Logstash 是 Elastic 公司开源的**服务器端数据处理管道**，能够同时从多个数据源采集数据，进行转换处理后输出到指定目的地。

核心特点：
- **输入 → 过滤 → 输出** 三段式管道架构
- 丰富的插件生态（200+ 插件）
- 支持多种数据源和输出目标
- 基于 JVM，跨平台运行

> 当前环境版本：**Logstash 9.3.2**

---

## 二、核心架构

```
Input → Filter → Output
  │        │        │
  │        │        └─ 输出到 ES / 文件 / Kafka 等
  │        └─ 解析、转换、 enrich 数据
  └─ 从文件 / Kafka / Beats / TCP 等采集
```

### 2.1 事件处理流程

1. **Input**：从数据源读取原始数据，生成 Event
2. **Filter**：对 Event 进行解析、转换、增删字段
3. **Output**：将处理后的 Event 发送到目的地
4. **Codec**：编解码器，在 Input/Output 中进行数据格式转换

### 2.2 队列机制

| 队列类型 | 说明 | 适用场景 |
|---------|------|---------|
| `memory` | 内存队列（默认） | 丢失可容忍、高性能 |
| `persisted` | 持久化磁盘队列 | 数据不可丢、需可靠投递 |

---

## 三、安装与目录结构

### 3.1 安装

```bash
# 下载解压即可
tar -xzf logstash-9.3.2.tar.gz
cd logstash-9.3.2
```

### 3.2 目录结构

```
logstash-9.3.2/
├── bin/            # 可执行脚本（logstash, logstash-plugin 等）
├── config/         # 配置文件
│   ├── logstash.yml      # 主配置文件
│   ├── pipelines.yml     # 多管道配置
│   ├── jvm.options       # JVM 参数
│   └── log4j2.properties # 日志配置
├── data/           # 持久化数据目录
├── logs/           # 运行日志
├── pipeline/       # 管道配置文件存放目录
└── vendor/         # 依赖包
```

---

## 四、配置文件详解

### 4.1 logstash.yml — 主配置

关键配置项：

```yaml
# 节点名称
node.name: logstash-node-01

# 数据存储路径
path.data: /var/lib/logstash

# 管道设置
pipeline.id: main
pipeline.workers: 4          # 工作线程数，默认 CPU 核数
pipeline.batch.size: 125     # 每批次处理事件数
pipeline.batch.delay: 50     # 批次等待时间（ms）

# 自动重载配置
config.reload.automatic: true
config.reload.interval: 3s

# 队列类型
queue.type: memory           # memory 或 persisted

# API 设置
api.http.host: 127.0.0.1
api.http.port: 9600

# 日志级别
log.level: info              # fatal/error/warn/info/debug/trace
```

### 4.2 pipelines.yml — 多管道配置

```yaml
- pipeline.id: nginx-access
  path.config: "/etc/logstash/pipeline/nginx-access.conf"
  pipeline.workers: 2

- pipeline.id: app-error
  path.config: "/etc/logstash/pipeline/app-error.conf"
  pipeline.workers: 1
  queue.type: persisted
```

### 4.3 管道配置文件语法

```conf
input {
  <插件名> {
    <参数> => <值>
  }
}

filter {
  <插件名> {
    <参数> => <值>
  }
}

output {
  <插件名> {
    <参数> => <值>
  }
}
```

---

## 五、Input 输入插件

### 5.1 file — 读取文件

```conf
input {
  file {
    path => ["/var/log/nginx/access.log"]
    start_position => "beginning"    # beginning 或 end
    sincedb_path => "/dev/null"      # 重置读取位置（调试用）
    codec => plain { charset => "UTF-8" }
  }
}
```

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `path` | 文件路径，支持通配符 | 必填 |
| `start_position` | 首次读取位置 | `end` |
| `sincedb_path` | 记录读取位置的文件 | 自动 |
| `mode` | `tail`（追尾）或 `read`（读完退出） | `tail` |

### 5.2 beats — 接收 Filebeat/Metricbeat

```conf
input {
  beats {
    port => 5044
    host => "0.0.0.0"
    ssl => false
  }
}
```

### 5.3 kafka — 消费 Kafka

```conf
input {
  kafka {
    bootstrap_servers => "kafka1:9092,kafka2:9092"
    topics => ["app-logs", "error-logs"]
    group_id => "logstash-consumer"
    consumer_threads => 3
    decorate_events => true
  }
}
```

### 5.4 tcp / udp — 网络监听

```conf
input {
  tcp {
    port => 514
    codec => json_lines
  }
}
```

### 5.5 stdin — 标准输入（调试用）

```conf
input {
  stdin { }
}
```

---

## 六、Filter 过滤插件

### 6.1 grok — 正则解析

最常用的日志解析插件：

```conf
filter {
  grok {
    # 解析 Nginx 访问日志
    match => {
      "message" => '%{IPORHOST:client_ip} - %{USERNAME:remote_user} \[%{HTTPDATE:timestamp}\] "%{WORD:method} %{URIPATHPARAM:request} HTTP/%{NUMBER:http_version}" %{NUMBER:status} %{NUMBER:bytes} "%{DATA:referrer}" "%{DATA:user_agent}"'
    }
    # 解析失败时添加标签
    tag_on_failure => ["_grokparsefailure"]
  }
}
```

**常用内置模式**：

| 模式 | 匹配内容 | 示例 |
|------|---------|------|
| `IP` | IP 地址 | `192.168.1.1` |
| `INT` | 整数 | `42` |
| `NUMBER` | 数字（含小数） | `3.14` |
| `WORD` | 单词 | `hello` |
| `DATA` | 非贪婪任意字符 | 短文本 |
| `GREEDYDATA` | 贪婪任意字符 | 长文本 |
| `HTTPDATE` | HTTP 日期 | `08/May/2026:18:00:00 +0800` |
| `URIPATHPARAM` | URI 路径+参数 | `/api/user?id=1` |
| `LOGLEVEL` | 日志级别 | `ERROR` |

### 6.2 mutate — 字段操作

```conf
filter {
  mutate {
    # 删除字段
    remove_field => ["@version", "tags"]
    # 重命名字段
    rename => { "old_name" => "new_name" }
    # 类型转换
    convert => {
      "status" => "integer"
      "bytes"  => "integer"
      "cost"   => "float"
    }
    # 替换值
    replace => { "host" => "%{client_ip}" }
    # 追加
    merge => { "tags" => "new_tag" }
  }
}
```

### 6.3 date — 时间解析

```conf
filter {
  date {
    match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"]
    target => "@timestamp"
    timezone => "Asia/Shanghai"
  }
}
```

### 6.4 json — JSON 解析

```conf
filter {
  json {
    source => "message"
    target => "json_content"    # 解析结果放入指定字段
    remove_field => ["message"]
  }
}
```

### 6.5 dissect — 简单分隔解析

比 grok 更快，适合固定格式：

```conf
filter {
  dissect {
    mapping => {
      "message" => "%{client_ip} - - [%{timestamp}] \"%{method} %{uri} %{proto}\" %{status} %{size}"
    }
  }
}
```

### 6.6 ruby — 自定义逻辑

```conf
filter {
  ruby {
    code => '
      event.set("lower_host", event.get("host").downcase) if event.get("host")
    '
  }
}
```

### 6.7 条件判断

```conf
filter {
  if [log_level] == "ERROR" or [log_level] == "FATAL" {
    mutate { add_field => { "alert" => "true" } }
  } else if [log_level] == "WARN" {
    mutate { add_field => { "alert" => "watch" } }
  } else {
    drop { }   # 丢弃其他级别
  }

  if "_grokparsefailure" in [tags] {
    drop { }
  }
}
```

---

## 七、Output 输出插件

### 7.1 elasticsearch — 输出到 ES

```conf
output {
  elasticsearch {
    hosts => ["http://es-node1:9200", "http://es-node2:9200"]
    index => "app-logs-%{+YYYY.MM.dd}"    # 按日期分索引
    # 认证
    user => "elastic"
    password => "changeme"
    # 使用模板
    template => "/etc/logstash/templates/es-template.json"
    template_name => "app-logs"
    # 动态索引名
    # index => "%{[app_name]}-%{+YYYY.MM.dd}"
  }
}
```

### 7.2 file — 输出到文件

```conf
output {
  file {
    path => "/var/log/logstash/processed-%{+YYYY-MM-dd}.log"
    codec => line { format => "%{message}" }
  }
}
```

### 7.3 kafka — 输出到 Kafka

```conf
output {
  kafka {
    bootstrap_servers => "kafka1:9092"
    topic_id => "processed-logs"
    compression_type => "gzip"
  }
}
```

### 7.4 stdout — 标准输出（调试）

```conf
output {
  stdout {
    codec => rubydebug    # 格式化输出，调试神器
  }
}
```

---

## 八、Codec 编解码器

| Codec | 说明 | 用法 |
|-------|------|------|
| `plain` | 纯文本 | `codec => plain` |
| `json` | JSON 单行 | `codec => json` |
| `json_lines` | JSON 多行 | `codec => json_lines` |
| `multiline` | 多行合并 | 见下方示例 |
| `rubydebug` | 调试输出 | 仅 stdout |

### 多行日志合并示例

```conf
input {
  file {
    path => "/var/log/app/stacktrace.log"
    codec => multiline {
      pattern => "^%{TIMESTAMP_ISO8601}"
      negate => true
      what => "previous"      # 不匹配的行归属到上一条
    }
  }
}
```

---

## 九、启动与运维

### 9.1 基本启动

```bash
# 前台启动（调试）
bin/logstash -f pipeline/test.conf --config.test_and_exit

# 前台运行
bin/logstash -f pipeline/test.conf

# 后台运行
nohup bin/logstash -f pipeline/test.conf > logs/logstash.log 2>&1 &

# 自动重载配置
bin/logstash -f pipeline/test.conf --config.reload.automatic
```

### 9.2 常用启动参数

| 参数 | 说明 |
|------|------|
| `-f <path>` | 指定管道配置文件或目录 |
| `-e <string>` | 命令行直接传入配置 |
| `--config.test_and_exit` | 只检查配置是否合法，不启动 |
| `--config.reload.automatic` | 自动重载配置变更 |
| `--log.level=<level>` | 日志级别 |
| `-w <count>` | pipeline.workers |
| `-b <count>` | pipeline.batch.size |

### 9.3 健康检查

```bash
# API 端口默认 9600
curl http://localhost:9600/
curl http://localhost:9600/_node/stats
curl http://localhost:9600/_node/pipelines
```

### 9.4 插件管理

```bash
# 查看已安装插件
bin/logstash-plugin list

# 安装插件
bin/logstash-plugin install logstash-filter-uuid

# 更新插件
bin/logstash-plugin update logstash-filter-json

# 更新所有插件
bin/logstash-plugin update
```

---

## 十、完整配置示例

### 10.1 Nginx 日志采集

```conf
input {
  file {
    path => "/var/log/nginx/access.log"
    start_position => "beginning"
    tags => ["nginx"]
  }
}

filter {
  if "nginx" in [tags] {
    grok {
      match => {
        "message" => '%{IPORHOST:client_ip} - %{USERNAME:remote_user} \[%{HTTPDATE:timestamp}\] "%{WORD:method} %{URIPATHPARAM:request} HTTP/%{NUMBER:http_version}" %{NUMBER:status} %{NUMBER:bytes} "%{DATA:referrer}" "%{DATA:user_agent}"'
      }
    }
    date {
      match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"]
      timezone => "Asia/Shanghai"
    }
    mutate {
      convert => {
        "status" => "integer"
        "bytes"  => "integer"
      }
      remove_field => ["message", "timestamp"]
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "nginx-access-%{+YYYY.MM.dd}"
  }
  stdout { codec => rubydebug }
}
```

### 10.2 Kafka → Logstash → Elasticsearch

```conf
input {
  kafka {
    bootstrap_servers => "kafka1:9092,kafka2:9092,kafka3:9092"
    topics => ["app-logs"]
    group_id => "logstash-consumer-group"
    consumer_threads => 4
    decorate_events => true
    codec => json
  }
}

filter {
  json {
    source => "message"
    remove_field => ["message"]
  }

  if [level] == "ERROR" {
    mutate { add_field => { "priority" => "high" } }
  }

  date {
    match => ["log_time", "yyyy-MM-dd HH:mm:ss.SSS"]
    target => "@timestamp"
  }
}

output {
  elasticsearch {
    hosts => ["http://es1:9200", "http://es2:9200"]
    index => "app-%{[app_name]}-%{+YYYY.MM.dd}"
  }
}
```

---

## 十一、性能调优

### 11.1 JVM 调优

编辑 `config/jvm.options`：

```
-Xms2g
-Xmx2g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
```

> 建议 `-Xms` 和 `-Xmx` 设为相同值，避免运行时动态扩缩

### 11.2 Pipeline 调优

| 参数 | 调优方向 |
|------|---------|
| `pipeline.workers` | 设为 CPU 核数，CPU 密集型可减少 |
| `pipeline.batch.size` | 增大可提高吞吐，但增加延迟 |
| `pipeline.batch.delay` | 减小可降低延迟，但增加 CPU 开销 |
| `queue.type: persisted` | 数据可靠但性能下降 |

### 11.3 常见性能问题

| 现象 | 原因 | 解决方案 |
|------|------|---------|
| CPU 100% | grok 正则复杂 | 优化正则、改用 dissect |
| OOM | batch.size 过大 | 调小 batch.size，增大 JVM |
| 输出慢 | ES 批量写入瓶颈 | 调大 ES flush_size |
| 积压 | 输入速度 > 处理速度 | 增加 workers 或优化 filter |

---

## 十二、注意事项

1. **grok 性能**：正则越复杂越慢，固定格式优先用 `dissect`
2. **@timestamp 时区**：默认 UTC，显示时需转换
3. **配置语法**：用 `--config.test_and_exit` 先验证再启动
4. **索引命名**：避免过多小索引，推荐 `app-name-YYYY.MM.dd` 格式
5. **持久化队列**：生产环境建议 `queue.type: persisted` 防数据丢失
6. **多管道**：不同数据源用不同 pipeline，隔离资源避免互相影响
