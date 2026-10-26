# 常用中间件使用范式

六大类高并发中间件的落地配置范式与关键参数。方法论层面见各场景分册（caching/async-mq/stability/data-layer），本册讲"怎么配"。

## 1. Redis

### 部署形态
- 单机 → 主从（读写分离）→ 哨兵（Sentinel，自动故障转移）→ Cluster（分片，16384 slot）。
- 高并发选型：纯缓存扩容用 Cluster；主从 + 哨兵适合量级中等但要高可用。
- 关键参数：`maxmemory` + 淘汰策略（缓存场景用 allkeys-lru 或 volatile-lru，禁 noeviction）；`appendonly yes` + `appendfsync everysec` 兼顾持久化与性能。

### 分布式锁范式
- 加锁必须原子：`SET lock_key unique_value NX EX 30`（NX + 过期时间一条命令，禁用 setnx + expire 两步——中途宕机死锁）。
- value 用唯一标识（UUID），释放锁用 Lua 脚本先比对 value 再 DEL，防止误删他人锁。
- 过期时间设为业务耗时上限；业务可能超时用看门狗续期（Redisson watchdog 模式，或自实现定时续期）。
- 主从切换丢锁风险：极端一致性要求用 RedLock 或改用 ZooKeeper/etcd（CP 系统）。

### 热点 Key
- 本地缓存 + Redis 双层；Key 打散（加随机后缀分片）；读写分离 + 从库扩容。
- 大 Key 治理：单 Key 值 < 10KB，集合元素 < 5000；大 Key 删除用 UNLINK（异步）或分批删除，禁线上 DEL 大 Key（阻塞主线程）。

## 2. 消息队列（Kafka / RocketMQ）

### Kafka 关键配置范式
- Topic 分区数 = 目标吞吐 / 单消费者吞吐，预留 2 倍余量；分区是并行度单位，消费组实例数 ≤ 分区数。
- 可靠性：producer `acks=all` + `retries>0` + `enable.idempotence=true`；broker `min.insync.replicas=2` 配 3 副本；consumer 关自动提交、处理完手动 commit。
- 顺序性：只保证分区内有序，需全局有序用单分区（牺牲吞吐）；按业务 Key 哈希到同分区保局部有序。
- 消息积压：先查消费端瓶颈（慢逻辑/下游限速），临时扩容受分区数上限约束 → 提前规划分区数；紧急情况用"新 topic + 大分区 + 快速消费转发"迁积压。

### RocketMQ 补充
- 事务消息：half 消息 → 本地事务 → commit/rollback，解决"本地事务与发消息的原子性"。
- 延时消息：`setDelayTimeLevel`（固定档位），用于订单超时关闭等场景，替代定时扫表。
- 死信队列：重试 16 次后进入 `%DLQ%+consumerGroup`，必须配人工/定时对账处理，不能只建不盯。

### 通用消费范式
- 消费幂等三件套：业务唯一 ID 去重表、数据库唯一约束、状态机校验。
- 重试队列与死信队列分离；监控两个核心指标：消费延迟（Lag）和积压量。

## 3. 限流熔断（Sentinel / Resilience4j 类）

### 限流规则范式
- 网关层：按 IP/接口粗粒度限流（Nginx `limit_req_zone`，令牌桶 `rate=10r/s burst=20 nodelay`）。
- 应用层：核心接口 QPS 限流 + 并发线程数限流双保险；热点参数限流（如按商品 ID 限流，防单点爆款打垮全局）。
- 阈值来源：压测得出的容量上限 × 0.8，而不是拍脑袋；水位见 stability.md。

### 熔断规则范式
- 慢调用熔断：RT 超过阈值（如 200ms）且比例超 50% 触发；异常比例熔断：错误率 > 50% 触发。
- 三参数拍定：统计窗口（如 10s）、最小请求数（如 20，防小样本误熔断）、熔断时长（如 10s，半开试探）。
- 降级返回必须是兜底数据（缓存旧值/默认值），不能抛异常裸奔；降级逻辑本身要轻，不能再依赖外部服务。

## 4. 负载均衡（Nginx）

### 反向代理范式
- 上游池：`upstream` + 健康检查（`max_fails=3 fail_timeout=30s` 自动摘除故障节点）。
- 算法选择：默认轮询；机器配置不均用 weight；有会话粘性需求用 ip_hash（优先外置 Session 到 Redis，避免粘性）。
- 静态资源与动态请求分离：静态直接 root/缓存，动态 proxy_pass 到应用集群。
- 超时链路完整设置：`proxy_connect_timeout` / `proxy_read_timeout` / `proxy_send_timeout`，与上游应用超时联动（外层 ≥ 内层）。

## 5. 分库分表中间件（ShardingSphere 类）

### 接入范式
- 逻辑表 → 真实节点映射：`db${0..3}.order_${0..15}` 这类行表达式声明分片节点。
- 分片算法：分片键取模（均匀）、范围（利于按时间查询）、哈希（散列好）；订单类常用 user_id 取模 + 按时间冷热归档。
- 广播表（字典表小表全库冗余）+ 绑定表（主子表同分片键，避免跨分片 join）。
- 跨分片查询兜底：异构索引（ES/宽表）承担多维查询，分片库只承担主查询路径；禁止运行时全分片路由聚合大结果集。
- 分布式 ID：雪花算法或号段模式，ID 趋势递增利于 B+ 树写入。

## 6. 连接池

### 参数范式（HikariCP 类通用）
- `maximumPoolSize`：不是越大越好。经验公式：连接数 ≈ CPU 核数 × 2 + 磁盘数；DB 端配合 `max_connections` 全局预算（各实例分账，总和 < DB 上限）。
- 必配超时三件套：获取连接超时（如 3s，快速失败别排队等死）、连接超时（如 1s）、空闲回收（`idleTimeout` + `maxLifetime` 略小于 DB 端 wait_timeout，避免拿到已被服务端断开的连接）。
- 泄漏检测：开启连接借出堆栈追踪（`leakDetectionThreshold`），定位未归还连接。
- 池化监控：活跃数/等待线程数/获取耗时三个指标必须进监控，等待线程堆积 = 池太小或 SQL 太慢。

## 中间件选型总原则

- 同类组件只选一个主力，减少运维面；引入新中间件必须对应一个明确痛点。
- 每个中间件都问三个问题：怎么扩容、怎么监控、挂了怎么降级。答不上来就先别上。
