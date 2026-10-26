# 高并发架构范式（high-concurrency-architecture）

高并发互联网架构设计范式的 AI Skill 知识库。给 AI Agent 装上一套经过验证的架构方法论，用于设计、评审、优化与讲解高并发系统。

## 核心方法论：六字诀

| 诀 | 手段 | 一句话 |
|---|---|---|
| 缓 | 多级缓存 | 浏览器 → CDN → 网关 → 进程内缓存 → Redis，层层挡读流量，可拦截 90%+ 请求 |
| 异 | MQ 异步化 | 非核心链路异步，削峰填谷，用最终一致性换吞吐 |
| 池 | 池化复用 | 线程池、连接池、对象池，避免频繁创建销毁 |
| 拆 | 无状态拆分 | 服务拆分 + 数据拆分，无状态才能水平扩容 |
| 预 | 预计算预热 | 大促前预热缓存、提前生成静态页，实时计算变查表 |
| 水 | 水平扩容 | 无状态服务按需扩容 + 限流兜底，日常水位 50%~60% |

## 目录结构

```
SKILL.md                          # 主文件：六字诀总纲、范式速查、触发示例、能力边界
references/
├── caching.md                    # 多级缓存、Cache Aside、穿透/击穿/雪崩、一致性
├── async-mq.md                   # MQ 选型、削峰填谷、可靠性与幂等、秒杀范式
├── data-layer.md                 # 读写分离、分库分表、分布式 ID、SQL 优化
├── stability.md                  # 限流、熔断、降级、隔离、容量水位、可观测
├── microservices.md              # 拆分时机与原则、分层架构、服务治理
├── middleware.md                 # Redis/MQ/限流熔断/Nginx/分库分表/连接池 配置范式
├── ali-db-guidelines.md          # 后端通用数据库规约（去语言化提炼自阿里开发手册）
└── faq-antipatterns.md           # 反模式、FAQ、组合场景、模糊需求处理范式
```

## 安装

### SkillHub

```bash
skillhub install high-concurrency-architecture --namespace user_fd70e67b
```

或访问 https://skillhub.cn/skills/user_fd70e67b/high-concurrency-architecture

### 手动安装

```bash
git clone https://github.com/little-apppple/high-concurrency-architecture.git
cp -r high-concurrency-architecture ~/.workbuddy/skills/
```

Claude Code 用 `~/.claude/skills/`，Cursor 用 `~/.cursor/skills/`，其他 Agent 按其 skills 目录放置。

## 覆盖场景

秒杀系统 · 缓存体系设计 · 消息队列削峰 · 分库分表 · 限流熔断降级 · 微服务拆分 · 容量规划 · 架构评审 · 数据库规约落地

## 适用边界

不适用于：单机小应用（并发非瓶颈）、大数据离线计算架构、纯前端性能优化、特定云厂商产品选型。

## License

MIT
