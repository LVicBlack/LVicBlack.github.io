---
title: 微服务 / 分布式下：MySQL 事务与 Kafka 一致性怎么做
date: 2023-10-11 16:06:00
categories: 
- 分布式
- 一致性
tags:
- 一致性

---



微服务 / 分布式下：MySQL 事务与 Kafka 一致性怎么做？
-----------------------------------------------------------------

在微服务 / 分布式系统里，**“MySQL 本地事务 + 发送 Kafka 消息” 无法做到真正原子性**。经典事故就是：

> 事务里先写库，再发 Kafka；结果 **Kafka 发出去了，但 MySQL 事务回滚**，下游服务收到消息后做了不可逆操作（发货 / 加余额 / 发券），系统直接不一致。

工程上通常不建议上 2PC/XA 这种强一致方案（复杂、性能差、链路脆弱）。主流做法是：  
**用可恢复的最终一致（Eventual Consistency）替代强一致。**

本文基于 **Kafka + Go + MySQL + Redis** 给出一套支付 / 订单通用、可落地的方案：  
**Transactional Outbox（本地消息表）+ 至少一次投递 + 消费端幂等**，并补充 Kafka 工程化配置、顺序性、重试 / DLQ、以及 CDC（Debezium）升级方案。

### 1. 为什么 “事务里直接发 Kafka” 会出事？

反例（不要这么做）：

1.  `BEGIN;`
2.  更新订单、支付记录等
3.  **发送 Kafka 消息（这一步不受 MySQL 事务控制）**
4.  `ROLLBACK;`

此时消息已经被下游消费，但数据库事务回滚，出现：

*   下游：认为订单已支付 -> 发货 / 发券 / 加余额
*   上游：订单仍未支付（因为回滚）
*   **业务不可逆，无法简单回滚**

> 即便 Kafka 支持幂等生产者、事务（EOS），也只能保证 **Kafka 内部** 的一致性（producer → broker → consumer offset），**不能把 MySQL 事务也纳入同一个原子提交**。所以跨系统一致性仍需要 Outbox / CDC / Saga 这类模式。

### 2. 核心原则：先保证 “落库 + 可追溯”，再异步投递

在分布式系统里更可靠的思路是：

*   **“数据写成功” 与 “消息可重放” 必须绑定在同一个本地事务里**
*   消息投递失败要可重试、可告警、可补偿
*   Kafka 投递采用 **至少一次**（At-least-once），所以消费者必须 **幂等**

### 3. 最推荐方案：Transactional Outbox（本地消息表）

#### 3.1 方案描述（强烈推荐，支付 / 订单通用）

**同一个 MySQL 事务里同时做两件事：**

1.  写业务数据（订单、支付状态）
2.  写 outbox 表（待发送事件）

事务提交后，由后台投递器把 outbox 里的事件发送到 Kafka：

*   发成功：标记 outbox 为 `SENT`
*   发失败：重试（指数退避），最终进入死信 / 告警

#### 3.2 为什么 Outbox 能解决 “Kafka 发了但事务回滚”？

因为消息**不是在事务里直接发 Kafka**，而是写一条 outbox 记录：

*   事务回滚 => outbox 记录不存在 => 投递器不会投递 => 下游不会收到消息
*   事务提交 => outbox 记录落库 => 即使服务宕机也可以恢复投递（消息可追溯）

### 4. Outbox 表设计建议（MySQL 示例）

> 表结构按需调整，核心是：可重试、可去重、可观测

```
CREATE TABLE message_outbox (
  id              CHAR(36) PRIMARY KEY,
  biz_key         VARCHAR(64) NOT NULL,          -- 业务主键，如 order_id
  event_type      VARCHAR(64) NOT NULL,          -- 事件类型，如 PaymentSucceeded
  topic           VARCHAR(128) NOT NULL,         -- Kafka topic（可选：也可由 event_type 映射）
  msg_key         VARCHAR(128) NOT NULL,         -- Kafka message key（建议 order_id）
  payload         JSON NOT NULL,                 -- 事件内容
  headers         JSON NULL,                     -- Kafka headers（可选）
  status          VARCHAR(16) NOT NULL,          -- NEW / SENDING / SENT / FAILED
  retry_count     INT NOT NULL DEFAULT 0,
  next_retry_at   DATETIME NULL,
  last_error      TEXT NULL,
  created_at      DATETIME NOT NULL,
  updated_at      DATETIME NOT NULL,
  UNIQUE KEY uk_biz_event (biz_key, event_type), -- 防止同一事件重复写入（可选）
  KEY idx_status_retry (status, next_retry_at),
  KEY idx_biz (biz_key)
);
```

**关键字段说明：**

*   `msg_key`：Kafka 分区 key（强烈建议用 `order_id`），用于**同订单的消息顺序性**
*   `headers`：建议携带 `message_id`/`trace_id`/`idempotency_key`/`schema_version` 等
*   `uk_biz_event`：可选唯一键，避免重复写 outbox（视业务而定）

### 5. Kafka 侧的工程化建议（非常重要）

#### 5.1 Topic / 分区 / 顺序性

*   **同一个订单内的事件要保持顺序**：生产消息时 `key = order_id`，Kafka 保证同 key 的消息落在同分区（从而分区内有序）。
*   分区数量：按吞吐与扩展预估设置（后续扩容会带来重分区与 key 映射变化，需要谨慎评估）。

#### 5.2 Producer（Go）建议配置

目标：减少丢失、减少乱序、提升可观测性。

*   `acks=all`
*   合理 `retries` + `retry.backoff`
*   开启 **幂等生产者**（Kafka 客户端 / 库支持时）：可降低 “网络抖动导致的重复写入”
*   消息体：携带 `message_id`（UUID / 雪花），用于消费端幂等

> 注意：幂等生产者 ≠ 跨 MySQL 一致性；它只能减少 Kafka 内部重复。

#### 5.3 Consumer（Go）建议配置与提交时机

*   **手动提交 offset**（不要 auto-commit）
*   提交顺序：**先完成业务处理（含幂等写入）再提交 offset**
    *   否则可能 “先提交 offset 后宕机”，导致消息丢处理
*   处理要支持重放：重复消费不产生副作用（见第 7 章）

### 6. 投递器（Outbox Dispatcher）在 Go 中怎么做？

#### 6.1 投递流程

*   定时（或常驻 worker）扫描 `NEW/FAILED` 且 `next_retry_at <= now()` 的记录
*   按批次取出，标记 `SENDING`（防止多实例重复投递）
*   发送到 Kafka
*   成功 -> `SENT`
*   失败 -> `FAILED`，计算下一次 `next_retry_at`，`retry_count++`，记录 `last_error`

#### 6.2 多实例并发抢占（MySQL 8 推荐）

MySQL 8 可用：

*   `SELECT ... FOR UPDATE SKIP LOCKED`：多实例并发扫描不会互相阻塞，天然 “抢任务”。

示意 SQL（根据你的字段调整）：

```
BEGIN;

SELECT id, topic, msg_key, payload, headers
FROM message_outbox
WHERE status IN ('NEW','FAILED')
  AND (next_retry_at IS NULL OR next_retry_at <= NOW())
ORDER BY created_at
LIMIT 200
FOR UPDATE SKIP LOCKED;

-- 将选中的记录置为 SENDING（同一事务内）
UPDATE message_outbox
SET status='SENDING', updated_at=NOW()
WHERE id IN (...);

COMMIT;
```

#### 6.3 发送到 Kafka 成功后更新状态

```
UPDATE message_outbox
SET status='SENT', updated_at=NOW()
WHERE id=? AND status='SENDING';
```

失败则：

*   `status='FAILED'`
*   `retry_count=retry_count+1`
*   `next_retry_at=NOW()+退避时间`
*   `last_error=...`

### 7. 消费端必须幂等（Kafka 至少一次投递的代价）

Kafka + 重试 + rebalance + 网络抖动 => **重复消费非常正常**。  
所以消费者必须做到：**重复消息不会造成重复副作用**。

#### 7.1 幂等实现方式 A：MySQL 去重表（最可靠）

```
CREATE TABLE consume_log (
  message_id   CHAR(36) PRIMARY KEY,
  consumed_at  DATETIME NOT NULL,
  biz_key      VARCHAR(64) NULL,
  event_type   VARCHAR(64) NULL
);
```

消费流程（建议包在同一个本地事务里）：

1.  `INSERT INTO consume_log(message_id, consumed_at, biz_key, event_type) VALUES (...)`
2.  插入成功 => 执行业务 => COMMIT => 再提交 offset
3.  主键冲突 => 说明已处理过 => 直接提交 offset

**优点：** 强一致、可审计、重放友好  
**缺点：** 需要额外表写入（但一般可接受）

#### 7.2 幂等实现方式 B：Redis 幂等（更快，但要处理过期与一致性）

用 Redis 记录 `SETNX idem:{message_id} 1 EX 7d`：

*   存在 => 已处理，直接 ack
*   不存在 => 处理后写入

**注意：**

*   Redis 幂等适合 “短期去重 / 削峰”，但不如 MySQL 去重 “可追溯”
*   若 Redis 丢数据或过期，可能发生重复副作用，所以**不可逆操作仍建议 MySQL 唯一约束兜底**

#### 7.3 幂等实现方式 C：业务状态机 + 条件更新

例如订单履约：

*   只有 `PAID` 才能进入 `SHIPPED`
*   更新用条件限制：`UPDATE ... WHERE status='PAID'`
*   影响行数为 0 说明重复 / 非法推进 -> 直接忽略

### 8. Kafka 重试 / DLQ（死信）怎么设计？

Kafka 自身不提供 “像 RabbitMQ 那样原生延迟队列” 功能（可通过多 Topic + 定时转发实现），常见方案：

#### 8.1 多级重试 Topic（推荐）

*   `payment.events`（主 topic）
*   `payment.events.retry.1m`
*   `payment.events.retry.10m`
*   `payment.events.dlq`

消费者失败后把消息投到对应 retry topic，并在 payload/headers 里记录 `retry_count`、`next_retry` 等。  
各 retry topic 的消费者可以用 “延迟处理器”（例如：消费后 sleep 到时间点，或用定时器轮询）——更推荐用**定时任务 / 延迟服务**做转发，避免消费线程阻塞。

#### 8.2 DLQ 处理要求

*   DLQ 必须告警（飞书 / 钉钉 / 邮件）
*   提供人工重放工具（按 message_id 重投）
*   保留必要上下文：原 topic、partition、offset、异常堆栈、trace_id

### 9. Kafka 场景升级：CDC（Debezium）自动投递 Outbox

如果团队重度使用 Kafka，可以让业务只写 MySQL：

*   订单表 + outbox 表
*   Debezium 订阅 binlog，把 outbox 的 INSERT/UPDATE 变成 Kafka 事件

优点：

*   投递链路更稳（减少自研投递器代码）
*   可观测性强（binlog → Kafka）
*   与多语言服务更解耦

缺点：

*   运维复杂度提高（部署 / 维护 CDC 组件）
*   Schema 演进、topic 管理更需要规范化

### 10. 支付业务建议：状态机 + Outbox +（可选）Saga / 补偿

支付通常分两层：

1.  **资金结果层**（渠道回调 / 查单确认）
2.  **业务履约层**（发货、发券、记账、积分）

资金成功后一般不做 “回滚资金”，而是走补偿链路（退款 / 冲正 / 撤销履约）。

#### 10.1 什么时候需要 Saga？

当一个业务流程跨多个服务且步骤多、失败概率高，需要补偿：

*   扣库存失败 -> 释放优惠券
*   发券失败 -> 回滚订单履约状态
*   记账失败 -> 进入人工修复队列

Saga 核心是：

*   每一步都是本地事务 + 事件
*   失败触发补偿事件（反向操作）
*   最终一致

### 11. 一句话总结（可直接落地）

*   不要在 MySQL 事务里直接发 Kafka
*   用 **Transactional Outbox** 把 “业务写库” 和“消息可重放”绑定在同一事务
*   Kafka 投递采用 **至少一次**，消费者必须 **幂等**
*   顺序性：消息 `key=order_id`，同订单事件按分区内顺序处理
*   失败处理：重试 topic + DLQ + 告警 + 可重放
*   Kafka 可用 **CDC（Debezium）** 把 outbox 自动流化