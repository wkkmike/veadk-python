# AI陪聊系统架构设计文档

## 1. 概述

### 1.1 项目背景
本文档描述一个基于火山引擎云平台的高可用AI陪聊系统架构设计。系统支持用户与AI角色进行长期、多轮对话，具备短期记忆和长期记忆能力。

### 1.2 核心需求
| 维度 | 需求 |
|------|------|
| 用户规模 | 初期1万，后期100万 |
| 峰值同时在线 | 50%（初期5千，后期50万） |
| 响应时间 | P99 < 10秒 |
| 短期记忆 | 最近一天左右的对话 |
| 长期记忆 | 用户全生命周期（画像、事件、偏好、摘要） |
| 大模型 | 火山方舟 doubao-seed-character |
| 输出方式 | 流式输出（SSE） |
| 可用性 | 高可用优先，可降级但必须可用 |

### 1.3 流量预估
```
初期峰值：
- 同时在线：5,000 用户
- 假设每用户每分钟1条消息
- 峰值QPS：~83 req/s

后期峰值：
- 同时在线：500,000 用户
- 峰值QPS：~8,333 req/s
- 考虑突发流量：设计容量 ~15,000 req/s
```

---

## 2. 整体架构

### 2.1 架构图

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    用户终端 (Web)                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              接入层 (火山云 CLB + WAF)                                │
│                         ┌─────────────────────────────────────┐                      │
│                         │   负载均衡 (多可用区, 主备热切换)      │                      │
│                         └─────────────────────────────────────┘                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              网关层 (API Gateway)                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │  认证鉴权     │  │  限流熔断     │  │  路由分发     │  │  协议转换     │             │
│  │  (JWT)       │  │  (令牌桶)     │  │              │  │  (WS/HTTP)   │             │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘             │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    ▼                     ▼                     ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              应用层 (VKE Kubernetes)                                 │
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────────┐ │
│  │                          对话服务 (Chat Service)                                │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │ │
│  │  │  会话管理     │  │  消息处理     │  │  流式响应     │  │  上下文组装   │        │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘        │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────────┐ │
│  │                          记忆服务 (Memory Service)                              │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │ │
│  │  │  短期记忆     │  │  长期记忆     │  │  记忆召回     │  │  记忆摘要     │        │ │
│  │  │  管理        │  │  管理        │  │  (意图触发)   │  │  (异步)      │        │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘        │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────────┐ │
│  │                          角色服务 (Character Service)                           │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                          │ │
│  │  │  角色管理     │  │  人设配置     │  │  Prompt模板   │                          │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘                          │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────────┐ │
│  │                          异步任务服务 (Async Worker)                            │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                          │ │
│  │  │  对话摘要     │  │  画像更新     │  │  记忆持久化   │                          │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘                          │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    ▼                     ▼                     ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    数据层                                            │
│                                                                                      │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐         │
│  │   火山云 Redis       │  │   火山云 VikingDB    │  │   火山云 RDS MySQL   │         │
│  │   (短期记忆缓存)      │  │   (向量检索)         │  │   (用户/角色/元数据)  │         │
│  │   主从 + 哨兵        │  │                     │  │   主从 + 读写分离    │         │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘         │
│                                                                                      │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐         │
│  │   火山云 TOS         │  │   火山云 Kafka       │  │   火山云 Redis       │         │
│  │   (长期记忆存储)      │  │   (异步消息队列)      │  │   (分布式锁/限流)    │         │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘         │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              AI模型层 (火山方舟)                                      │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                              模型路由 & 熔断器                                │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │    │
│  │  │ doubao-seed-    │  │ doubao-pro      │  │ doubao-lite     │              │    │
│  │  │ character       │  │ (备用模型)       │  │ (兜底模型)       │              │    │
│  │  │ (主模型)         │  │                 │  │                 │              │    │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘              │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 分层说明

| 层级 | 组件 | 职责 |
|------|------|------|
| 接入层 | CLB + WAF | 负载均衡、DDoS防护、SSL终结 |
| 网关层 | API Gateway | 认证、限流、路由、协议转换 |
| 应用层 | VKE (K8s) | 业务逻辑、无状态服务 |
| 数据层 | Redis/MySQL/TOS/VikingDB | 数据存储与检索 |
| AI层 | 火山方舟 | 大模型推理 |

---

## 3. 短期记忆方案（多选一）

### 3.1 方案对比

| 方案 | 架构 | 优点 | 缺点 | 推荐场景 |
|------|------|------|------|----------|
| **方案A** | 纯Redis | 简单可靠，运维成本低 | 跨请求延迟略高 | 初期推荐 |
| **方案B** | 本地缓存+Redis | 极低延迟，减少Redis压力 | 一致性复杂，内存占用高 | 追求极致性能 |
| **方案C** | Redis+异步持久化 | 数据可恢复，支持更长时间 | 架构复杂 | 需要数据分析 |

---

### 3.2 方案A：纯Redis方案（推荐初期使用）

```
┌─────────────────────────────────────────────────────────────────┐
│                        对话服务                                  │
│  ┌─────────────┐                                                │
│  │  请求处理    │                                                │
│  └──────┬──────┘                                                │
│         │                                                        │
│         ▼                                                        │
│  ┌─────────────┐      ┌─────────────────────────────────────┐   │
│  │ Redis客户端  │─────▶│  火山云 Redis (主从+哨兵)            │   │
│  │ (连接池)     │      │                                     │   │
│  └─────────────┘      │  Key: chat:{user_id}:{character_id} │   │
│                       │  Value: List<Message>                │   │
│                       │  TTL: 86400s (24小时)                │   │
│                       └─────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

**数据结构设计：**
```python
# Redis Key 设计
SHORT_MEMORY_KEY = "chat:{user_id}:{character_id}:messages"
SESSION_META_KEY = "chat:{user_id}:{character_id}:meta"

# 消息结构 (JSON序列化存储)
message = {
    "msg_id": "uuid",
    "role": "user|assistant",
    "content": "消息内容",
    "timestamp": 1699999999,
    "token_count": 150
}

# 操作示例
LPUSH SHORT_MEMORY_KEY message_json  # 新消息入队
LTRIM SHORT_MEMORY_KEY 0 99          # 保留最近100条
EXPIRE SHORT_MEMORY_KEY 86400        # 24小时过期
```

**优点：**
- 架构简单，运维成本低
- 火山云Redis支持主从+哨兵，自动故障切换
- TTL自动清理过期数据

**降级策略：**
```python
async def get_short_memory(user_id: str, character_id: str) -> List[Message]:
    try:
        messages = await redis.lrange(key, 0, -1)
        return [Message.parse(m) for m in messages]
    except RedisError:
        # 降级：返回空列表，本次对话无历史上下文
        logger.warning(f"Redis unavailable, degrading to no history")
        return []
```

---

### 3.3 方案B：本地缓存 + Redis（追求极致性能）

```
┌─────────────────────────────────────────────────────────────────┐
│                        对话服务 Pod                              │
│  ┌─────────────┐                                                │
│  │  请求处理    │                                                │
│  └──────┬──────┘                                                │
│         │                                                        │
│         ▼                                                        │
│  ┌─────────────┐     Cache Miss      ┌──────────────────────┐   │
│  │ 本地LRU缓存  │────────────────────▶│  火山云 Redis        │   │
│  │ (Cachetools)│◀────────────────────│  (主从+哨兵)         │   │
│  │ TTL: 5min   │     Load Data       │  TTL: 24h           │   │
│  │ Max: 10000  │                     └──────────────────────┘   │
│  └──────┬──────┘                                                │
│         │ Write Through                                         │
│         ▼                                                        │
│  ┌─────────────┐                                                │
│  │ 异步写Redis  │                                                │
│  └─────────────┘                                                │
└─────────────────────────────────────────────────────────────────┘

                              │
                              │ Pub/Sub 同步
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     其他对话服务 Pod                              │
│                    (通过Redis Pub/Sub 失效本地缓存)               │
└─────────────────────────────────────────────────────────────────┘
```

**实现代码：**
```python
from cachetools import TTLCache
import asyncio

class TwoLevelCache:
    def __init__(self):
        # 本地缓存：最多10000个会话，5分钟TTL
        self.local_cache = TTLCache(maxsize=10000, ttl=300)
        self.redis = redis_client
        
    async def get_messages(self, user_id: str, char_id: str) -> List[Message]:
        key = f"chat:{user_id}:{char_id}:messages"
        
        # L1: 本地缓存
        if key in self.local_cache:
            return self.local_cache[key]
        
        # L2: Redis
        try:
            messages = await self.redis.lrange(key, 0, -1)
            parsed = [Message.parse(m) for m in messages]
            self.local_cache[key] = parsed
            return parsed
        except RedisError:
            return []
    
    async def add_message(self, user_id: str, char_id: str, msg: Message):
        key = f"chat:{user_id}:{char_id}:messages"
        
        # 更新本地缓存
        if key in self.local_cache:
            self.local_cache[key].insert(0, msg)
        
        # 异步写Redis + 通知其他Pod
        asyncio.create_task(self._async_write(key, msg))
        
    async def _async_write(self, key: str, msg: Message):
        try:
            await self.redis.lpush(key, msg.json())
            await self.redis.ltrim(key, 0, 99)
            await self.redis.expire(key, 86400)
            # 发布失效通知
            await self.redis.publish(f"invalidate:{key}", "update")
        except RedisError:
            logger.error("Failed to write to Redis")
```

**优点：**
- 本地缓存命中时延迟 < 1ms
- 减少Redis网络开销

**缺点：**
- 多Pod间需要缓存同步
- 内存占用较高
- 架构复杂度增加

---

### 3.4 方案C：Redis + 异步持久化（支持数据分析）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              对话服务                                        │
└─────────────────────────────────────────────────────────────────────────────┘
         │                              │
         │ 实时读写                      │ 异步写入
         ▼                              ▼
┌─────────────────┐           ┌─────────────────┐
│  火山云 Redis    │           │  火山云 Kafka    │
│  (热数据 24h)    │           │  (消息队列)      │
└─────────────────┘           └────────┬────────┘
                                       │
                                       ▼
                              ┌─────────────────┐
                              │  异步Worker      │
                              │  (批量持久化)    │
                              └────────┬────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
           ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
           │ 火山云 TOS    │   │ 火山云 RDS    │   │ 数据分析平台  │
           │ (冷数据归档)  │   │ (结构化存储)  │   │ (可选)       │
           └──────────────┘   └──────────────┘   └──────────────┘
```

**适用场景：**
- 需要对历史对话进行数据分析
- 短期记忆周期较长（超过24小时）
- 需要支持用户查看历史对话记录

---

### 3.5 短期记忆方案推荐

| 阶段 | 推荐方案 | 理由 |
|------|----------|------|
| **初期（1万用户）** | 方案A：纯Redis | 简单可靠，快速上线 |
| **中期（10-50万用户）** | 方案B：本地缓存+Redis | 降低Redis压力，提升性能 |
| **后期（100万用户）** | 方案C：Redis+持久化 | 支持数据分析，完整数据链路 |

**我的建议：从方案A开始，根据实际监控数据决定是否升级。**

---

## 4. 长期记忆方案

### 4.1 长期记忆架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              长期记忆系统                                     │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                          记忆存储层                                   │    │
│  │                                                                       │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐       │    │
│  │  │ 火山云 VikingDB  │  │ 火山云 RDS      │  │ 火山云 TOS      │       │    │
│  │  │                 │  │                 │  │                 │       │    │
│  │  │ • 对话摘要向量   │  │ • 用户画像      │  │ • 完整对话归档   │       │    │
│  │  │ • 情感记忆向量   │  │ • 关键事件      │  │ • 大文本存储    │       │    │
│  │  │ • 关键信息向量   │  │ • 用户偏好      │  │                 │       │    │
│  │  │                 │  │ • 角色关系      │  │                 │       │    │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘       │    │
│  │           │                   │                    │                 │    │
│  │           └───────────────────┼────────────────────┘                 │    │
│  │                               │                                       │    │
│  │                               ▼                                       │    │
│  │                    ┌─────────────────────┐                           │    │
│  │                    │    记忆检索引擎      │                           │    │
│  │                    │                     │                           │    │
│  │                    │ • 向量相似度检索     │                           │    │
│  │                    │ • 结构化条件过滤     │                           │    │
│  │                    │ • 混合排序          │                           │    │
│  │                    └─────────────────────┘                           │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                          记忆生成层 (异步)                            │    │
│  │                                                                       │    │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐             │    │
│  │  │  对话摘要生成   │  │  画像提取      │  │  关键事件识别   │             │    │
│  │  │  (LLM)        │  │  (LLM)        │  │  (LLM)        │             │    │
│  │  └───────────────┘  └───────────────┘  └───────────────┘             │    │
│  │           │                 │                  │                     │    │
│  │           └─────────────────┼──────────────────┘                     │    │
│  │                             │                                         │    │
│  │                             ▼                                         │    │
│  │                    ┌───────────────┐                                 │    │
│  │                    │  Embedding    │                                 │    │
│  │                    │  (向量化)     │                                 │    │
│  │                    └───────────────┘                                 │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 长期记忆数据模型

**MySQL 表结构：**
```sql
-- 用户画像表
CREATE TABLE user_profiles (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id VARCHAR(64) NOT NULL,
    character_id VARCHAR(64) NOT NULL,
    profile_type ENUM('basic_info', 'preference', 'relationship') NOT NULL,
    profile_key VARCHAR(128) NOT NULL,      -- 如: name, birthday, favorite_food
    profile_value TEXT NOT NULL,
    confidence FLOAT DEFAULT 1.0,            -- 置信度
    source_msg_id VARCHAR(64),               -- 来源消息ID
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE KEY uk_user_char_key (user_id, character_id, profile_type, profile_key),
    INDEX idx_user_char (user_id, character_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 关键事件表
CREATE TABLE user_events (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id VARCHAR(64) NOT NULL,
    character_id VARCHAR(64) NOT NULL,
    event_type VARCHAR(64) NOT NULL,         -- birthday, anniversary, achievement
    event_summary TEXT NOT NULL,
    event_date DATE,
    emotion_tag VARCHAR(32),                 -- happy, sad, excited
    importance TINYINT DEFAULT 5,            -- 1-10
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_user_char (user_id, character_id),
    INDEX idx_event_date (event_date)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 对话摘要索引表（与VikingDB向量关联）
CREATE TABLE conversation_summaries (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id VARCHAR(64) NOT NULL,
    character_id VARCHAR(64) NOT NULL,
    summary_text TEXT NOT NULL,
    vector_id VARCHAR(128) NOT NULL,         -- VikingDB中的向量ID
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP NOT NULL,
    message_count INT NOT NULL,
    topic_tags JSON,                         -- ["旅行", "美食", "工作"]
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_user_char (user_id, character_id),
    INDEX idx_time_range (start_time, end_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**VikingDB 向量数据模型：**
```python
# VikingDB Collection 设计
collection_config = {
    "collection_name": "long_term_memory",
    "fields": [
        {"field_name": "vector", "field_type": "vector", "dim": 1536},
        {"field_name": "user_id", "field_type": "string"},
        {"field_name": "character_id", "field_type": "string"},
        {"field_name": "memory_type", "field_type": "string"},  # summary/event/profile
        {"field_name": "content", "field_type": "string"},
        {"field_name": "timestamp", "field_type": "int64"},
        {"field_name": "importance", "field_type": "int64"},
    ],
    "indexes": [
        {
            "index_type": "HNSW",
            "metric_type": "IP",  # Inner Product
            "params": {"M": 16, "efConstruction": 200}
        }
    ]
}
```

### 4.3 长期记忆召回流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          长期记忆召回流程                                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │  1. 意图识别 (快速LLM判断)      │
                    │                               │
                    │  输入: 用户最新消息            │
                    │  输出: 是否需要召回长期记忆     │
                    │        召回类型: profile/     │
                    │        event/history          │
                    └───────────────┬───────────────┘
                                    │
                          需要召回   │   不需要召回
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
    ┌───────────────────────────────┐   ┌───────────────────────────┐
    │  2. 并行检索                   │   │  直接使用短期记忆对话      │
    │                               │   └───────────────────────────┘
    │  ┌─────────┐  ┌─────────┐    │
    │  │VikingDB │  │  MySQL  │    │
    │  │向量检索  │  │结构化查询│    │
    │  └────┬────┘  └────┬────┘    │
    │       │            │         │
    │       └─────┬──────┘         │
    │             ▼                │
    │  ┌─────────────────────┐     │
    │  │  3. 结果融合 & 排序   │     │
    │  │                     │     │
    │  │  • 相关性评分        │     │
    │  │  • 时间衰减         │     │
    │  │  • 重要性加权        │     │
    │  └──────────┬──────────┘     │
    └─────────────┼────────────────┘
                  │
                  ▼
    ┌───────────────────────────────┐
    │  4. 记忆注入上下文             │
    │                               │
    │  System Prompt += 长期记忆     │
    └───────────────────────────────┘
```

**意图识别 Prompt：**
```python
RECALL_INTENT_PROMPT = """
判断用户消息是否需要召回长期记忆。

用户消息: {user_message}
最近对话摘要: {recent_context}

请判断:
1. 是否需要召回长期记忆 (yes/no)
2. 如需召回，类型是什么:
   - profile: 用户个人信息（名字、生日、喜好等）
   - event: 历史事件（之前聊过的重要事情）
   - history: 历史对话（之前讨论过的话题）

输出JSON格式:
{"need_recall": true/false, "recall_type": "profile/event/history", "search_query": "检索关键词"}
"""
```

### 4.4 长期记忆生成（异步处理）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        长期记忆生成流程 (异步)                                │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                          触发条件                                    │    │
│  │                                                                       │    │
│  │  • 会话结束（LLM判断或超时）                                           │    │
│  │  • 累积消息数达到阈值（如50条）                                        │    │
│  │  • 定时任务（每日凌晨处理前一天数据）                                   │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                         │
│                                    ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    Kafka Consumer (异步Worker)                       │    │
│  │                                                                       │    │
│  │  ┌─────────────────────────────────────────────────────────────┐     │    │
│  │  │  Step 1: 对话摘要生成                                        │     │    │
│  │  │                                                              │     │    │
│  │  │  Prompt: 请总结以下对话的主要内容和情感基调...                  │     │    │
│  │  │  输出: 结构化摘要 + 话题标签                                   │     │    │
│  │  └─────────────────────────────────────────────────────────────┘     │    │
│  │                                                                       │    │
│  │  ┌─────────────────────────────────────────────────────────────┐     │    │
│  │  │  Step 2: 用户画像提取                                        │     │    │
│  │  │                                                              │     │    │
│  │  │  Prompt: 从对话中提取用户透露的个人信息...                     │     │    │
│  │  │  输出: {key: value, confidence} 结构化画像                   │     │    │
│  │  └─────────────────────────────────────────────────────────────┘     │    │
│  │                                                                       │    │
│  │  ┌─────────────────────────────────────────────────────────────┐     │    │
│  │  │  Step 3: 关键事件识别                                        │     │    │
│  │  │                                                              │     │    │
│  │  │  Prompt: 识别对话中提到的重要事件（生日、纪念日等）...          │     │    │
│  │  │  输出: 事件列表 + 重要性评分                                   │     │    │
│  │  └─────────────────────────────────────────────────────────────┘     │    │
│  │                                                                       │    │
│  │  ┌─────────────────────────────────────────────────────────────┐     │    │
│  │  │  Step 4: 向量化 & 存储                                       │     │    │
│  │  │                                                              │     │    │
│  │  │  • Embedding API 生成向量                                     │     │    │
│  │  │  • 写入 VikingDB                                              │     │    │
│  │  │  • 更新 MySQL 索引                                            │     │    │
│  │  │  • 原始对话归档到 TOS                                         │     │    │
│  │  └─────────────────────────────────────────────────────────────┘     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. 核心服务设计

### 5.1 对话服务 (Chat Service)

```python
# 对话服务核心流程
class ChatService:
    async def handle_message(
        self,
        user_id: str,
        character_id: str,
        message: str
    ) -> AsyncIterator[str]:
        """处理用户消息，返回流式响应"""
        
        # 1. 获取角色配置
        character = await self.character_service.get_character(character_id)
        
        # 2. 获取短期记忆
        short_memory = await self.memory_service.get_short_memory(
            user_id, character_id
        )
        
        # 3. 判断是否需要长期记忆召回
        recall_decision = await self.memory_service.check_recall_intent(
            message, short_memory
        )
        
        # 4. 并行获取长期记忆（如果需要）
        long_memory = []
        if recall_decision.need_recall:
            long_memory = await self.memory_service.recall_long_memory(
                user_id, character_id, recall_decision
            )
        
        # 5. 组装上下文
        context = self.build_context(
            character=character,
            short_memory=short_memory,
            long_memory=long_memory,
            user_message=message
        )
        
        # 6. 调用大模型（流式）
        response_text = ""
        async for chunk in self.llm_service.chat_stream(context):
            response_text += chunk
            yield chunk
        
        # 7. 异步保存消息
        asyncio.create_task(
            self.memory_service.save_messages(
                user_id, character_id, message, response_text
            )
        )
```

### 5.2 大模型服务（含熔断降级）

```python
from circuitbreaker import circuit
from enum import Enum

class ModelTier(Enum):
    PRIMARY = "doubao-seed-character"      # 主模型
    SECONDARY = "doubao-pro-32k"           # 备用模型
    FALLBACK = "doubao-lite-32k"           # 兜底模型

class LLMService:
    def __init__(self):
        self.ark_client = ArkClient()
        self.model_health = {tier: True for tier in ModelTier}
        
    @circuit(failure_threshold=5, recovery_timeout=60)
    async def _call_model(
        self,
        model: ModelTier,
        messages: List[dict]
    ) -> AsyncIterator[str]:
        """带熔断器的模型调用"""
        async for chunk in self.ark_client.chat_stream(
            model=model.value,
            messages=messages
        ):
            yield chunk
    
    async def chat_stream(
        self,
        messages: List[dict]
    ) -> AsyncIterator[str]:
        """支持自动降级的流式对话"""
        
        # 按优先级尝试模型
        for model in ModelTier:
            if not self.model_health[model]:
                continue
                
            try:
                async for chunk in self._call_model(model, messages):
                    yield chunk
                return  # 成功则返回
                
            except CircuitBreakerError:
                logger.warning(f"Model {model.value} circuit open, trying next")
                self.model_health[model] = False
                continue
                
            except Exception as e:
                logger.error(f"Model {model.value} error: {e}")
                continue
        
        # 所有模型都失败，返回兜底话术
        yield "抱歉，我现在有点忙，稍后再聊好吗？"
```

### 5.3 流式输出实现（SSE）

```python
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from sse_starlette.sse import EventSourceResponse

app = FastAPI()

@app.post("/api/v1/chat/stream")
async def chat_stream(request: ChatRequest):
    """流式对话接口 (Server-Sent Events)"""
    
    async def event_generator():
        try:
            async for chunk in chat_service.handle_message(
                user_id=request.user_id,
                character_id=request.character_id,
                message=request.message
            ):
                yield {
                    "event": "message",
                    "data": json.dumps({
                        "content": chunk,
                        "done": False
                    })
                }
            
            # 发送完成标记
            yield {
                "event": "message",
                "data": json.dumps({
                    "content": "",
                    "done": True
                })
            }
            
        except Exception as e:
            yield {
                "event": "error",
                "data": json.dumps({"error": str(e)})
            }
    
    return EventSourceResponse(event_generator())
```

---

## 6. 高可用设计

### 6.1 多可用区部署

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            火山云 Region                                     │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                          可用区 A (主)                                │    │
│  │                                                                       │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                   │    │
│  │  │ VKE Node 1  │  │ VKE Node 2  │  │ VKE Node 3  │                   │    │
│  │  │             │  │             │  │             │                   │    │
│  │  │ Chat Pod x3 │  │ Chat Pod x3 │  │ Memory Pod  │                   │    │
│  │  │ Memory Pod  │  │ Char Pod    │  │ Worker Pod  │                   │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                   │    │
│  │                                                                       │    │
│  │  ┌─────────────┐  ┌─────────────┐                                    │    │
│  │  │ Redis Master│  │ MySQL Master│                                    │    │
│  │  └─────────────┘  └─────────────┘                                    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                         │
│                                    │ 同步复制                                │
│                                    ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                          可用区 B (备)                                │    │
│  │                                                                       │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                   │    │
│  │  │ VKE Node 4  │  │ VKE Node 5  │  │ VKE Node 6  │                   │    │
│  │  │             │  │             │  │             │                   │    │
│  │  │ Chat Pod x2 │  │ Chat Pod x2 │  │ Memory Pod  │                   │    │
│  │  │ Memory Pod  │  │ Char Pod    │  │ Worker Pod  │                   │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                   │    │
│  │                                                                       │    │
│  │  ┌─────────────┐  ┌─────────────┐                                    │    │
│  │  │ Redis Slave │  │ MySQL Slave │                                    │    │
│  │  └─────────────┘  └─────────────┘                                    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 降级策略矩阵

| 组件 | 故障场景 | 降级策略 | 影响 |
|------|----------|----------|------|
| **Redis** | 主节点故障 | 自动切换从节点 | 无感知 |
| **Redis** | 全部不可用 | 跳过短期记忆，继续对话 | 丢失历史上下文 |
| **VikingDB** | 检索超时/失败 | 跳过长期记忆召回 | 无法引用历史 |
| **MySQL** | 主节点故障 | 切换读副本 | 短暂不可写 |
| **MySQL** | 全部不可用 | 使用缓存中的角色配置 | 无法更新配置 |
| **Kafka** | 不可用 | 本地队列缓冲，重试 | 记忆生成延迟 |
| **主模型** | 超时/限流 | 切换备用模型 | 回复质量可能下降 |
| **所有模型** | 全部不可用 | 返回预设话术 | 无法正常对话 |

### 6.3 降级代码实现

```python
class ResilientMemoryService:
    """高可用记忆服务"""
    
    async def get_short_memory(
        self,
        user_id: str,
        character_id: str
    ) -> List[Message]:
        """获取短期记忆，支持降级"""
        try:
            return await asyncio.wait_for(
                self._get_from_redis(user_id, character_id),
                timeout=1.0  # 1秒超时
            )
        except (asyncio.TimeoutError, RedisError) as e:
            logger.warning(f"Short memory fetch failed: {e}, degrading")
            # 降级：返回空列表
            return []
    
    async def recall_long_memory(
        self,
        user_id: str,
        character_id: str,
        query: str
    ) -> List[MemoryItem]:
        """召回长期记忆，支持降级"""
        results = []
        
        # 并行查询，任一失败不影响其他
        tasks = [
            self._search_vikingdb(user_id, character_id, query),
            self._query_mysql_profile(user_id, character_id),
            self._query_mysql_events(user_id, character_id),
        ]
        
        for task in asyncio.as_completed(tasks, timeout=2.0):
            try:
                result = await task
                results.extend(result)
            except Exception as e:
                logger.warning(f"Long memory query failed: {e}")
                continue
        
        return results

class ResilientCharacterService:
    """高可用角色服务"""
    
    def __init__(self):
        # 本地缓存角色配置
        self.character_cache = TTLCache(maxsize=1000, ttl=3600)
        
    async def get_character(self, character_id: str) -> Character:
        """获取角色配置，支持多级降级"""
        
        # L1: 本地缓存
        if character_id in self.character_cache:
            return self.character_cache[character_id]
        
        # L2: Redis缓存
        try:
            cached = await self.redis.get(f"character:{character_id}")
            if cached:
                char = Character.parse(cached)
                self.character_cache[character_id] = char
                return char
        except RedisError:
            pass
        
        # L3: MySQL
        try:
            char = await self.mysql.fetch_character(character_id)
            if char:
                self.character_cache[character_id] = char
                # 异步回填Redis
                asyncio.create_task(self._cache_to_redis(character_id, char))
                return char
        except Exception:
            pass
        
        # L4: 默认角色（兜底）
        logger.error(f"All sources failed for character {character_id}, using default")
        return Character.default()
```

### 6.4 限流与过载保护

```python
from fastapi import Request, HTTPException
from fastapi.middleware.base import BaseHTTPMiddleware
import redis

class RateLimitMiddleware(BaseHTTPMiddleware):
    """基于Redis的滑动窗口限流"""
    
    def __init__(self, app, redis_client):
        super().__init__(app)
        self.redis = redis_client
        
        # 限流配置
        self.limits = {
            "user": {"requests": 60, "window": 60},    # 每用户60次/分钟
            "global": {"requests": 10000, "window": 1}  # 全局10000次/秒
        }
    
    async def dispatch(self, request: Request, call_next):
        user_id = request.headers.get("X-User-ID", "anonymous")
        
        # 检查用户级限流
        if not await self._check_rate_limit(f"rate:user:{user_id}", self.limits["user"]):
            raise HTTPException(status_code=429, detail="请求太频繁，请稍后再试")
        
        # 检查全局限流
        if not await self._check_rate_limit("rate:global", self.limits["global"]):
            raise HTTPException(status_code=503, detail="系统繁忙，请稍后再试")
        
        return await call_next(request)
    
    async def _check_rate_limit(self, key: str, limit: dict) -> bool:
        """滑动窗口限流检查"""
        try:
            pipe = self.redis.pipeline()
            now = time.time()
            window_start = now - limit["window"]
            
            pipe.zremrangebyscore(key, 0, window_start)
            pipe.zadd(key, {str(now): now})
            pipe.zcard(key)
            pipe.expire(key, limit["window"])
            
            results = await pipe.execute()
            current_count = results[2]
            
            return current_count <= limit["requests"]
            
        except RedisError:
            # Redis故障时，允许通过（宁可过载也不拒绝服务）
            logger.warning("Rate limit check failed, allowing request")
            return True
```

---

## 7. 可扩展性设计

### 7.1 水平扩展策略

```yaml
# Kubernetes HPA 配置
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: chat-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: chat-service
  minReplicas: 3
  maxReplicas: 100
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
```

### 7.2 数据库分片策略（后期）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          数据分片策略 (100万用户)                             │
│                                                                              │
│  分片键: user_id                                                             │
│  分片数: 16                                                                  │
│  分片算法: user_id_hash % 16                                                 │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                          MySQL 分片                                  │    │
│  │                                                                       │    │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐       ┌─────────┐            │    │
│  │  │ Shard 0 │  │ Shard 1 │  │ Shard 2 │  ...  │ Shard 15│            │    │
│  │  │ 0-6.25% │  │6.25-12.5│  │12.5-18.7│       │93.75-100│            │    │
│  │  └─────────┘  └─────────┘  └─────────┘       └─────────┘            │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                          Redis Cluster                               │    │
│  │                                                                       │    │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐       ┌─────────┐            │    │
│  │  │ Node 0  │  │ Node 1  │  │ Node 2  │  ...  │ Node 7  │            │    │
│  │  │ Slot    │  │ Slot    │  │ Slot    │       │ Slot    │            │    │
│  │  │ 0-2047  │  │2048-4095│  │4096-6143│       │14336-   │            │    │
│  │  └─────────┘  └─────────┘  └─────────┘       │   16383 │            │    │
│  │                                               └─────────┘            │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.3 扩展路线图

| 阶段 | 用户规模 | 架构调整 |
|------|----------|----------|
| **Phase 1** | 1万 | 单可用区，单实例Redis/MySQL |
| **Phase 2** | 10万 | 多可用区，Redis主从，MySQL读写分离 |
| **Phase 3** | 50万 | Redis Cluster，MySQL分库 |
| **Phase 4** | 100万 | 全面分片，CDN边缘加速 |

---

## 8. 火山云组件选型

### 8.1 组件清单

| 组件 | 火山云服务 | 规格（初期） | 规格（后期） | 用途 |
|------|-----------|-------------|-------------|------|
| 容器平台 | VKE | 3节点 8C16G | 50节点 16C32G | 应用部署 |
| 负载均衡 | CLB | 标准型 | 性能型 | 流量分发 |
| 缓存 | 云数据库Redis | 4G主从 | 64G集群 | 短期记忆 |
| 数据库 | 云数据库MySQL | 4C8G主从 | 16C64G x 16分片 | 结构化数据 |
| 向量库 | VikingDB | 基础版 | 专业版 | 长期记忆检索 |
| 对象存储 | TOS | 标准存储 | 标准存储 | 对话归档 |
| 消息队列 | Kafka | 3节点 | 9节点 | 异步任务 |
| 大模型 | 火山方舟 | 按量付费 | 预留实例 | AI推理 |

### 8.2 成本估算（月）

| 组件 | 初期（1万用户） | 后期（100万用户） |
|------|----------------|-------------------|
| VKE | ¥3,000 | ¥150,000 |
| CLB | ¥500 | ¥5,000 |
| Redis | ¥800 | ¥20,000 |
| MySQL | ¥1,500 | ¥80,000 |
| VikingDB | ¥2,000 | ¥50,000 |
| TOS | ¥200 | ¥10,000 |
| Kafka | ¥1,000 | ¥15,000 |
| 方舟（模型） | ¥5,000 | ¥500,000 |
| **合计** | **~¥14,000** | **~¥830,000** |

*注：大模型调用费用占主要成本，建议关注方舟的预留实例优惠*

---

## 9. 监控与可观测性

### 9.1 监控体系

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              可观测性体系                                     │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                          Metrics (指标)                              │    │
│  │                                                                       │    │
│  │  业务指标:                                                            │    │
│  │  • 请求QPS / 成功率 / P99延迟                                         │    │
│  │  • 模型调用成功率 / 降级率                                             │    │
│  │  • 记忆召回命中率                                                      │    │
│  │                                                                       │    │
│  │  系统指标:                                                            │    │
│  │  • CPU / Memory / Network                                            │    │
│  │  • Redis 连接数 / 命中率                                              │    │
│  │  • MySQL QPS / 慢查询                                                 │    │
│  │                                                                       │    │
│  │  工具: 火山云 APMPlus / Prometheus                                    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                          Logging (日志)                              │    │
│  │                                                                       │    │
│  │  • 结构化日志 (JSON格式)                                              │    │
│  │  • 请求链路追踪ID                                                     │    │
│  │  • 敏感信息脱敏                                                       │    │
│  │                                                                       │    │
│  │  工具: 火山云 TLS                                                     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                          Tracing (链路追踪)                          │    │
│  │                                                                       │    │
│  │  • 全链路追踪: 网关 → 服务 → 缓存 → DB → 模型                          │    │
│  │  • 自动埋点 + 手动埋点                                                │    │
│  │                                                                       │    │
│  │  工具: 火山云 APMPlus / OpenTelemetry                                 │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 9.2 关键告警配置

```yaml
# 告警规则
alerts:
  - name: "P99延迟超阈值"
    condition: "chat_latency_p99 > 10s"
    duration: "5m"
    severity: critical
    
  - name: "模型调用成功率低"
    condition: "llm_success_rate < 95%"
    duration: "3m"
    severity: critical
    
  - name: "Redis连接池耗尽"
    condition: "redis_connection_available < 10%"
    duration: "1m"
    severity: warning
    
  - name: "错误率突增"
    condition: "error_rate > 5%"
    duration: "2m"
    severity: critical
```

---

## 10. 部署架构

### 10.1 CI/CD 流水线

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CI/CD 流水线                                    │
│                                                                              │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐   │
│  │  Code   │───▶│  Build  │───▶│  Test   │───▶│  Image  │───▶│ Deploy  │   │
│  │  Push   │    │         │    │         │    │  Push   │    │         │   │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘   │
│                                                                              │
│  工具: 火山云 CI (代码仓库) + 火山云 CR (镜像仓库) + VKE (部署)               │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 10.2 灰度发布策略

```yaml
# VKE 灰度发布配置
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: chat-service
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: {duration: 5m}
        - setWeight: 30
        - pause: {duration: 5m}
        - setWeight: 50
        - pause: {duration: 10m}
        - setWeight: 100
      analysis:
        templates:
          - templateName: success-rate
        args:
          - name: service-name
            value: chat-service
```

---

## 11. 安全设计

### 11.1 认证与鉴权

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              认证流程                                        │
│                                                                              │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐                   │
│  │  用户   │───▶│  登录   │───▶│  JWT    │───▶│  API    │                   │
│  │  登录   │    │  服务   │    │  签发   │    │  访问   │                   │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘                   │
│                                                                              │
│  JWT Claims:                                                                 │
│  • user_id: 用户唯一标识                                                     │
│  • exp: 过期时间 (24h)                                                       │
│  • permissions: 权限列表                                                     │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 11.2 数据安全

| 场景 | 措施 |
|------|------|
| 传输加密 | 全链路 HTTPS/TLS 1.3 |
| 存储加密 | 火山云托管密钥加密 |
| 敏感数据 | 日志脱敏，不记录对话内容 |
| 访问控制 | RBAC + 最小权限原则 |

---

## 12. 附录

### 12.1 API 接口设计

```yaml
openapi: 3.0.0
info:
  title: AI陪聊系统 API
  version: 1.0.0

paths:
  /api/v1/chat/stream:
    post:
      summary: 流式对话
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                character_id:
                  type: string
                  description: 角色ID
                message:
                  type: string
                  description: 用户消息
      responses:
        200:
          description: SSE流式响应
          content:
            text/event-stream:
              schema:
                type: string
                
  /api/v1/characters:
    get:
      summary: 获取角色列表
    post:
      summary: 创建自定义角色
      
  /api/v1/characters/{id}:
    get:
      summary: 获取角色详情
    put:
      summary: 更新角色配置
    delete:
      summary: 删除角色
```

### 12.2 目录结构

```
ai-companion/
├── src/
│   ├── api/                    # API层
│   │   ├── routes/
│   │   │   ├── chat.py
│   │   │   └── character.py
│   │   └── middleware/
│   │       ├── auth.py
│   │       └── rate_limit.py
│   │
│   ├── services/               # 服务层
│   │   ├── chat_service.py
│   │   ├── memory_service.py
│   │   ├── character_service.py
│   │   └── llm_service.py
│   │
│   ├── memory/                 # 记忆模块
│   │   ├── short_memory.py
│   │   ├── long_memory.py
│   │   └── recall_engine.py
│   │
│   ├── models/                 # 数据模型
│   │   ├── message.py
│   │   ├── character.py
│   │   └── user_profile.py
│   │
│   ├── workers/                # 异步任务
│   │   ├── summary_worker.py
│   │   └── profile_worker.py
│   │
│   └── utils/                  # 工具类
│       ├── redis_client.py
│       ├── mysql_client.py
│       └── vikingdb_client.py
│
├── config/
│   ├── settings.py
│   └── logging.py
│
├── deploy/
│   ├── kubernetes/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── hpa.yaml
│   └── docker/
│       └── Dockerfile
│
├── tests/
├── requirements.txt
└── README.md
```

---

## 13. 总结

本架构设计遵循以下核心原则：

1. **高可用优先**：多级降级策略，确保任何组件故障都不会导致服务完全不可用
2. **渐进式扩展**：从简单架构起步，预留扩展能力，按需演进
3. **成本可控**：在保证可用性的前提下，选择性价比最优的方案
4. **云原生**：充分利用火山云托管服务，减少运维负担

### 下一步行动建议

1. **短期记忆方案选择**：建议从「方案A：纯Redis」开始
2. **MVP快速验证**：先实现核心对话功能，再迭代记忆系统
3. **监控先行**：在开发初期就建立完善的可观测性体系
4. **压测验证**：上线前进行全链路压测，验证P99延迟目标
