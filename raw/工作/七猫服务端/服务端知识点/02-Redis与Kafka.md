# Redis 与 Kafka

Redis 和 Kafka 都是常见的后端基础设施，但解决的问题不同：

- Redis 主要解决快速读写、缓存和临时状态存储。
- Kafka 主要解决异步消息、系统解耦和事件流传输。

## 1. Redis 是什么

Redis 是一个以内存为主的高性能键值数据库。因为数据主要保存在内存中，它的读写通常比 MySQL 等磁盘数据库快很多。

典型缓存链路：

```text
用户请求 -> 应用 -> Redis
                    |
                  未命中
                    v
                  MySQL
```

读取热门小说时，可以先查询 Redis；缓存未命中时再查 MySQL，并将结果写回 Redis。

### 常见使用场景

- 缓存热门小说、商品和用户信息
- 保存登录状态、验证码和短期任务状态
- 接口限流
- 阅读量、点赞量等计数器
- 排行榜
- 分布式锁
- 缓存重建协调

### 基础命令

```redis
SET user:1001:name "Tom"
GET user:1001:name
DEL user:1001:name

EXPIRE user:1001:name 3600
TTL user:1001:name
```

写入时直接设置过期时间：

```redis
SET verify_code:13800138000 "9527" EX 300
```

### Redis 数据结构

| 类型 | 特点 | 常见场景 |
|---|---|---|
| String | 单个字符串、数字或二进制值 | 缓存、验证码、计数器 |
| Hash | 一个 Key 下保存多个字段 | 用户信息、商品信息 |
| List | 有顺序、允许重复 | 简单队列、最新记录 |
| Set | 无顺序、元素不重复 | 标签、关注关系、去重 |
| Sorted Set | 元素不重复，按照分数排序 | 排行榜、热度榜 |

示例：

```redis
HSET user:1001 name "Tom" age 18
HGET user:1001 name

LPUSH recent:books "book-1"
SADD book:1001:tags "悬疑" "推理"
ZADD book:ranking 9800 "book-1"
```

### 哪些数据适合缓存

适合缓存的数据通常具有以下特点：

- 读取频率高，更新频率较低
- 计算或数据库查询成本较高
- 短时间内允许一定程度的旧数据
- 丢失后可以从数据库或其他来源重建

不应仅保存在缓存中的数据：支付结果、订单最终状态等不能丢失的核心数据。

### Key 如何命名

建议使用统一、可读且有层次的格式：

```text
业务:对象:标识:字段
```

例如：

```text
book:detail:1001
user:session:2001
ranking:book:daily
lock:product:1001
```

### 过期时间如何确定

需要结合业务容忍度和数据库压力：

- 验证码：数分钟
- 登录状态：数小时或数天
- 热门内容：数分钟至数小时
- 变化很少的配置：更长时间，并在修改后主动删除

可以为过期时间增加少量随机值，避免大量 Key 同时过期导致数据库压力突然升高。

### Redis 与 MySQL 如何保持一致

最常见的是 Cache Aside 模式：

```text
读取：先查 Redis -> 未命中则查 MySQL -> 写入 Redis
更新：先更新 MySQL -> 再删除 Redis 缓存
```

通常选择删除缓存，而不是同时更新缓存。删除后，下次读取会从 MySQL 加载最新数据。

需要注意：Redis 和 MySQL 很难在所有异常情况下始终保持瞬时强一致。重要业务应以 MySQL 为准，并结合重试、消息通知、过期时间和补偿任务缩短不一致时间。

## 2. Kafka 是什么

Kafka 是一个分布式事件流和消息系统。一个系统可以向 Kafka 发送消息，其他系统再异步消费这些消息。

例如用户下单后需要扣库存、发积分、发送通知和更新报表。直接调用会让订单服务依赖所有下游系统：

```text
订单服务 -> 库存服务
        -> 积分服务
        -> 通知服务
        -> 报表服务
```

使用 Kafka 后：

```text
                 -> 库存服务
订单服务 -> Kafka -> 积分服务
                 -> 通知服务
                 -> 报表服务
```

订单服务只负责发布“订单已创建”事件，不必同步等待所有后续操作完成。

### 常见使用场景

- 异步处理
- 微服务之间解耦
- 流量削峰填谷
- 订单、支付和内容更新通知
- 日志和埋点采集
- 数据同步
- 实时数据分析

### 基础概念

| 概念 | 含义 |
|---|---|
| Message / Record | Kafka 中传输的一条消息 |
| Producer | 生产消息的程序 |
| Consumer | 消费消息的程序 |
| Topic | 消息的逻辑分类，如 `order-created` |
| Partition | Topic 的分片，用于提高并发能力 |
| Broker | 一台 Kafka 服务器节点 |
| Consumer Group | 一组协作消费消息的消费者 |
| Offset | 消息在 Partition 中的位置编号 |

一个 Topic 可以包含多个 Partition：

```text
Topic: order-created
├── Partition 0: 消息 0、3、6
├── Partition 1: 消息 1、4、7
└── Partition 2: 消息 2、5、8
```

同一个 Partition 内的消息有顺序，不同 Partition 之间通常不保证全局顺序。

### Consumer Group

同一个消费者组内，一个 Partition 同时只交给一个消费者处理：

```text
order-handler 消费者组
├── Consumer A -> Partition 0
├── Consumer B -> Partition 1
└── Consumer C -> Partition 2
```

不同消费者组可以各自消费同一条消息：

```text
order-created Topic
├── inventory-group：处理库存
├── notification-group：发送通知
└── analytics-group：更新报表
```

## 3. Kafka 消息去重

Kafka 消息可能被重复消费。例如消费者已经完成业务处理，但还没提交消费进度就崩溃，重启后 Kafka 会再次投递该消息。

消息中通常携带唯一事件 ID：

```json
{
  "event_id": "order-paid-10001",
  "order_id": 10001,
  "type": "ORDER_PAID"
}
```

消费者处理前检查该 ID：

```text
从未处理 -> 执行业务，并记录 event_id
已经处理 -> 跳过，不重复执行业务
```

可以在数据库中为 `event_id` 建立唯一索引：

```sql
CREATE UNIQUE INDEX uk_event_id
ON consumed_events(event_id);
```

Kafka 消息去重的本质，是把消费逻辑设计成幂等操作。

## 4. Redis 与 Kafka 的区别

| 对比项 | Redis | Kafka |
|---|---|---|
| 核心用途 | 缓存和高速数据访问 | 消息传递和事件流 |
| 典型操作 | 按 Key 读写数据 | 生产和消费消息 |
| 数据组织 | String、Hash、List 等 | Topic、Partition、Record |
| 关注重点 | 查询速度、过期时间 | 吞吐量、可靠投递、消费进度 |
| 典型例子 | 缓存小说详情和排行榜 | 发布“小说已更新”事件 |

Redis 也有 Pub/Sub 和 Streams，但需要大规模消息持久化、消息回放、高吞吐量和多个消费者组时，Kafka 通常更合适。实际系统中两者经常同时使用。

