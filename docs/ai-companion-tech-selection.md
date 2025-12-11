# AI陪聊系统 - 技术选型与决策指南

## 1. 短期记忆方案决策矩阵

### 1.1 三种方案对比

| 评估维度 | 方案A: 纯Redis | 方案B: 本地缓存+Redis | 方案C: Redis+持久化 |
|---------|---------------|---------------------|-------------------|
| **架构复杂度** | ⭐ 简单 | ⭐⭐⭐ 复杂 | ⭐⭐ 中等 |
| **开发成本** | 1周 | 2-3周 | 2周 |
| **运维成本** | 低 | 中 | 中 |
| **读延迟** | 1-5ms | <1ms (本地命中) | 1-5ms |
| **写延迟** | 1-5ms | 1-5ms (异步写) | 1-5ms + 异步 |
| **一致性** | 强一致 | 最终一致 | 最终一致 |
| **数据可恢复** | Redis RDB | Redis RDB | 完全可恢复 |
| **支持数据分析** | ❌ | ❌ | ✅ |
| **水平扩展能力** | Redis Cluster | 需要处理缓存同步 | 需要分片 |
| **初期成本** | ¥800/月 | ¥800/月 | ¥1,500/月 |
| **后期成本** | ¥20,000/月 | ¥15,000/月 | ¥35,000/月 |

### 1.2 决策建议

```
                              ┌─────────────────────┐
                              │  你的业务阶段是?    │
                              └──────────┬──────────┘
                                         │
               ┌─────────────────────────┼─────────────────────────┐
               │                         │                         │
               ▼                         ▼                         ▼
        ┌─────────────┐          ┌─────────────┐          ┌─────────────┐
        │  MVP/初创期  │          │  成长期     │          │  成熟期     │
        │  (<10万用户) │          │ (10-50万)   │          │ (>50万)     │
        └──────┬──────┘          └──────┬──────┘          └──────┬──────┘
               │                        │                        │
               ▼                        ▼                        ▼
        ┌─────────────┐          ┌─────────────┐          ┌─────────────┐
        │  ✅ 方案A    │          │  根据需求:   │          │  ✅ 方案C   │
        │  纯Redis    │          │             │          │  完整链路   │
        │             │          │  性能优先:   │          │             │
        │  快速上线   │          │  → 方案B    │          │  数据分析   │
        │  验证产品   │          │             │          │  用户画像   │
        └─────────────┘          │  分析优先:   │          └─────────────┘
                                 │  → 方案C    │
                                 └─────────────┘
```

### 1.3 各方案详细技术规格

#### 方案A: 纯Redis

```yaml
# 火山云 Redis 配置
产品: 云数据库 Redis
版本: 6.0
架构: 主从版 (初期) / 集群版 (后期)
规格:
  初期: 4GB 主从版
  中期: 16GB 主从版
  后期: 64GB 集群版 (8分片)
  
# 数据模型
Key Pattern: chat:{user_id}:{character_id}:messages
Data Type: List
Max Length: 100 条消息
TTL: 86400 秒 (24小时)

# 连接配置
连接池大小: 20 (单Pod)
连接超时: 1000ms
读超时: 500ms
写超时: 500ms
重试次数: 2
```

#### 方案B: 本地缓存+Redis

```yaml
# 本地缓存配置
类型: LRU + TTL
实现: cachetools.TTLCache
容量: 10,000 个会话
TTL: 300 秒 (5分钟)
内存占用: ~500MB/Pod

# 缓存同步策略
同步方式: Redis Pub/Sub
失效通道: cache:invalidate:{key_pattern}
失效延迟: < 100ms

# 一致性保证
写策略: Write-Through (先写Redis,再更新本地)
读策略: Read-Through (本地未命中则读Redis)
失效策略: 其他Pod写入时广播失效消息
```

#### 方案C: Redis+持久化

```yaml
# 消息队列配置
产品: 火山云 Kafka
Topic: conversation_persist
分区数: 16
副本数: 3
消息保留: 7天

# 持久化存储
数据库: 火山云 MySQL
表: conversation_messages
分区: 按 user_id hash 分区
归档: 7天后转移到 TOS 冷存储

# 消费者配置
消费者组: message-persist-worker
并发数: 8
批量大小: 100条
提交策略: 处理成功后提交
```

---

## 2. 大模型选型与降级策略

### 2.1 火山方舟模型对比

| 模型 | 上下文窗口 | 角色扮演能力 | 响应速度 | 成本 | 用途 |
|------|-----------|------------|---------|------|------|
| doubao-seed-character | 32K | ⭐⭐⭐⭐⭐ | 中 | 高 | 主模型 |
| doubao-pro-32k | 32K | ⭐⭐⭐⭐ | 中 | 中 | 备用模型 |
| doubao-pro-128k | 128K | ⭐⭐⭐⭐ | 慢 | 高 | 超长上下文 |
| doubao-lite-32k | 32K | ⭐⭐⭐ | 快 | 低 | 兜底模型 |

### 2.2 模型调用策略

```python
# 模型配置
MODEL_CONFIG = {
    "primary": {
        "name": "doubao-seed-character",
        "timeout": 60,
        "max_retries": 2,
        "circuit_breaker": {
            "failure_threshold": 5,      # 5次失败触发熔断
            "recovery_timeout": 60,      # 60秒后尝试恢复
            "half_open_requests": 3      # 半开状态允许3个请求
        }
    },
    "secondary": {
        "name": "doubao-pro-32k",
        "timeout": 60,
        "max_retries": 2,
        "circuit_breaker": {
            "failure_threshold": 5,
            "recovery_timeout": 60,
            "half_open_requests": 3
        }
    },
    "fallback": {
        "name": "doubao-lite-32k",
        "timeout": 30,
        "max_retries": 1,
        "circuit_breaker": {
            "failure_threshold": 10,     # 兜底模型更宽容
            "recovery_timeout": 30,
            "half_open_requests": 5
        }
    }
}
```

### 2.3 降级决策流程

```
模型调用请求
     │
     ▼
┌────────────────────────────────────────┐
│  检查主模型(doubao-seed)熔断器状态     │
└────────────────┬───────────────────────┘
                 │
       ┌─────────┴─────────┐
       │                   │
    关闭                  打开
       │                   │
       ▼                   ▼
  调用主模型          检查备用模型熔断器
       │                   │
  ┌────┴────┐        ┌─────┴─────┐
  │         │        │           │
成功      失败      关闭        打开
  │         │        │           │
  ▼         ▼        ▼           ▼
返回     记录失败  调用备用     检查兜底模型
结果     次数+1    模型             │
             │         │      ┌─────┴─────┐
             │    ┌────┴────┐ │           │
             │    │         │ 关闭       打开
             │  成功      失败  │           │
             │    │         │  ▼           ▼
             │    ▼         ▼ 调用      返回固定
             │  返回     记录失败 兜底     兜底话术
             │  结果     次数+1  模型
             │              │     │
             │              │  ┌──┴──┐
             │              │ 成功  失败
             │              │  │     │
             │              │  ▼     ▼
             │              │ 返回  返回固定
             │              │ 结果  兜底话术
             │              │
             └──────────────┘
                    │
                    ▼
            检查是否达到熔断阈值
                    │
              ┌─────┴─────┐
              │           │
             是          否
              │           │
              ▼           ▼
         打开熔断器    继续监控
```

---

## 3. 数据库选型

### 3.1 MySQL vs PostgreSQL

| 维度 | MySQL (推荐) | PostgreSQL |
|------|-------------|------------|
| 火山云支持 | ✅ 成熟 | ✅ 支持 |
| 运维成本 | 低 | 中 |
| JSON支持 | 良好 | 优秀 |
| 分库分表生态 | 成熟 (ShardingSphere) | 相对弱 |
| 读写分离 | 原生支持 | 需配置 |
| 团队熟悉度 | 通常更高 | - |

**推荐: MySQL**，因为火山云RDS MySQL更成熟，且团队通常更熟悉。

### 3.2 表设计建议

```sql
-- 分表策略: 按 user_id 取模分16张表
-- user_profiles_00 到 user_profiles_15

-- 索引策略
CREATE TABLE user_profiles (
    -- 主键使用自增ID，便于分页
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    
    -- 业务主键，用于去重和更新
    user_id VARCHAR(64) NOT NULL,
    character_id VARCHAR(64) NOT NULL,
    
    -- 联合唯一索引
    UNIQUE KEY uk_user_char_key (user_id, character_id, profile_type, profile_key),
    
    -- 常用查询索引
    INDEX idx_user_char (user_id, character_id),
    INDEX idx_updated (updated_at)  -- 用于增量同步
);
```

---

## 4. 向量数据库: VikingDB 配置指南

### 4.1 Collection 设计

```python
# VikingDB Collection 配置
collection_config = {
    "collection_name": "ai_companion_memory",
    "description": "AI陪聊系统长期记忆存储",
    
    # 字段定义
    "fields": [
        {
            "field_name": "id",
            "field_type": "int64",
            "is_primary": True,
            "auto_id": True
        },
        {
            "field_name": "vector",
            "field_type": "float_vector",
            "dim": 1536,  # 使用火山方舟 Embedding 维度
        },
        {
            "field_name": "user_id",
            "field_type": "string",
            "max_length": 64
        },
        {
            "field_name": "character_id",
            "field_type": "string",
            "max_length": 64
        },
        {
            "field_name": "memory_type",
            "field_type": "string",
            "max_length": 32  # summary, event, profile
        },
        {
            "field_name": "content",
            "field_type": "string",
            "max_length": 4096
        },
        {
            "field_name": "timestamp",
            "field_type": "int64"
        },
        {
            "field_name": "importance",
            "field_type": "int64"
        }
    ],
    
    # 索引配置
    "index": {
        "index_type": "HNSW",
        "metric_type": "IP",  # 内积相似度
        "params": {
            "M": 16,
            "efConstruction": 256
        }
    }
}
```

### 4.2 检索优化

```python
# 检索参数配置
search_params = {
    "anns_field": "vector",
    "topk": 10,
    "metric_type": "IP",
    "params": {
        "ef": 128  # 检索时的候选集大小
    },
    # 标量过滤
    "filter": {
        "must": [
            {"term": {"user_id": user_id}},
            {"term": {"character_id": character_id}}
        ],
        "should": [
            {"term": {"memory_type": "summary"}},
            {"term": {"memory_type": "event"}}
        ]
    }
}
```

---

## 5. 消息队列: Kafka 配置

### 5.1 Topic 设计

```yaml
Topics:
  # 长期记忆生成任务
  - name: memory_generation_tasks
    partitions: 16
    replication_factor: 3
    retention: 7d
    cleanup_policy: delete
    
  # 用户画像更新
  - name: profile_update_tasks
    partitions: 8
    replication_factor: 3
    retention: 3d
    cleanup_policy: delete
    
  # 会话结束事件
  - name: session_end_events
    partitions: 16
    replication_factor: 3
    retention: 1d
    cleanup_policy: delete
```

### 5.2 消费者配置

```python
# Kafka Consumer 配置
consumer_config = {
    "bootstrap_servers": ["kafka-1:9092", "kafka-2:9092", "kafka-3:9092"],
    "group_id": "memory-worker-group",
    "auto_offset_reset": "earliest",
    "enable_auto_commit": False,  # 手动提交
    "max_poll_records": 100,
    "max_poll_interval_ms": 300000,  # 5分钟
    "session_timeout_ms": 30000,
    "heartbeat_interval_ms": 10000
}

# 消费处理逻辑
async def process_message(msg):
    try:
        # 处理消息
        await generate_memory(msg)
        # 手动提交
        await consumer.commit()
    except Exception as e:
        # 失败消息发送到死信队列
        await send_to_dlq(msg, e)
        await consumer.commit()
```

---

## 6. 缓存策略详解

### 6.1 多级缓存架构

```
                    ┌─────────────────────────────────────────┐
                    │              请求入口                    │
                    └────────────────────┬────────────────────┘
                                         │
                                         ▼
                    ┌─────────────────────────────────────────┐
                    │         L1: 本地内存缓存 (可选)          │
                    │                                         │
                    │  • 容量: 10,000 条目                    │
                    │  • TTL: 5 分钟                          │
                    │  • 命中率目标: 60-80%                   │
                    │  • 延迟: < 1ms                          │
                    └────────────────────┬────────────────────┘
                                         │ Miss
                                         ▼
                    ┌─────────────────────────────────────────┐
                    │         L2: Redis 分布式缓存             │
                    │                                         │
                    │  • 容量: 按需扩展                        │
                    │  • TTL: 24 小时                         │
                    │  • 命中率目标: 95%+                      │
                    │  • 延迟: 1-5ms                          │
                    └────────────────────┬────────────────────┘
                                         │ Miss
                                         ▼
                    ┌─────────────────────────────────────────┐
                    │         L3: 持久化存储 (MySQL/TOS)       │
                    │                                         │
                    │  • 容量: 无限                           │
                    │  • 延迟: 5-50ms                         │
                    └─────────────────────────────────────────┘
```

### 6.2 缓存一致性策略

```python
# 缓存更新策略: Cache-Aside Pattern

class CacheAsideManager:
    """Cache-Aside 缓存管理"""
    
    async def get(self, key: str) -> Optional[Any]:
        """读取数据"""
        # 1. 先查缓存
        cached = await self.redis.get(key)
        if cached:
            return json.loads(cached)
        
        # 2. 缓存未命中，查数据库
        data = await self.db.get(key)
        if data:
            # 3. 回填缓存
            await self.redis.setex(
                key, 
                self.ttl, 
                json.dumps(data)
            )
        return data
    
    async def update(self, key: str, value: Any):
        """更新数据"""
        # 1. 先更新数据库
        await self.db.update(key, value)
        
        # 2. 删除缓存（而不是更新）
        # 下次读取时会自动回填最新数据
        await self.redis.delete(key)
        
    async def delete(self, key: str):
        """删除数据"""
        # 1. 先删除数据库
        await self.db.delete(key)
        
        # 2. 再删除缓存
        await self.redis.delete(key)
```

---

## 7. 性能优化建议

### 7.1 延迟优化清单

| 优化点 | 预期收益 | 实施难度 | 优先级 |
|--------|---------|---------|--------|
| Redis 连接池复用 | 减少 5-10ms | 低 | P0 |
| 批量读取消息 | 减少 RTT | 低 | P0 |
| 模型流式输出 | 感知延迟降低 50% | 中 | P0 |
| 意图识别并行化 | 减少 100-200ms | 中 | P1 |
| 本地缓存热点数据 | 减少 1-3ms | 中 | P1 |
| 预加载角色配置 | 减少 5-10ms | 低 | P1 |
| gRPC替代HTTP | 减少 10-20ms | 高 | P2 |
| 就近部署 | 减少网络延迟 | 中 | P2 |

### 7.2 吞吐量优化

```python
# 异步处理最佳实践

import asyncio
from concurrent.futures import ThreadPoolExecutor

class OptimizedChatService:
    def __init__(self):
        self.executor = ThreadPoolExecutor(max_workers=4)
        
    async def handle_message(self, user_id, char_id, message):
        # 并行执行独立任务
        short_memory_task = self.get_short_memory(user_id, char_id)
        character_task = self.get_character(char_id)
        
        # 等待必要的前置任务
        short_memory, character = await asyncio.gather(
            short_memory_task,
            character_task
        )
        
        # 意图识别
        intent = await self.check_recall_intent(message, short_memory)
        
        # 条件性并行: 长期记忆召回
        long_memory = []
        if intent.need_recall:
            long_memory = await self.recall_long_memory(
                user_id, char_id, intent.query
            )
        
        # 流式调用LLM
        async for chunk in self.call_llm_stream(
            character, short_memory, long_memory, message
        ):
            yield chunk
        
        # 异步保存，不阻塞响应
        asyncio.create_task(
            self.save_message(user_id, char_id, message)
        )
```

---

## 8. 安全配置检查清单

### 8.1 网络安全

```yaml
# VPC 网络隔离
网络架构:
  - 公网入口: 仅 CLB 暴露
  - 应用层: 私有子网
  - 数据层: 私有子网 + 安全组
  
安全组规则:
  应用层:
    入站:
      - CLB -> 应用 Port: 8080
    出站:
      - 应用 -> Redis Port: 6379
      - 应用 -> MySQL Port: 3306
      - 应用 -> Kafka Port: 9092
      - 应用 -> 方舟API Port: 443
      
  数据层:
    入站:
      - 应用 -> Redis/MySQL/Kafka
    出站:
      - 无
```

### 8.2 敏感信息处理

```python
# 日志脱敏配置
import re

SENSITIVE_PATTERNS = [
    (r'"content":\s*"[^"]*"', '"content": "[REDACTED]"'),
    (r'"message":\s*"[^"]*"', '"message": "[REDACTED]"'),
    (r'Authorization:\s*\S+', 'Authorization: [REDACTED]'),
]

def sanitize_log(log_message: str) -> str:
    """脱敏日志消息"""
    for pattern, replacement in SENSITIVE_PATTERNS:
        log_message = re.sub(pattern, replacement, log_message)
    return log_message

# 配置日志处理器
class SanitizedFormatter(logging.Formatter):
    def format(self, record):
        message = super().format(record)
        return sanitize_log(message)
```

---

## 9. 成本优化建议

### 9.1 模型调用成本优化

```python
# Token 使用优化策略

class TokenOptimizer:
    """Token 使用优化"""
    
    def optimize_context(self, messages: List[dict], max_tokens: int) -> List[dict]:
        """智能裁剪上下文"""
        
        total_tokens = sum(self.count_tokens(m) for m in messages)
        
        if total_tokens <= max_tokens:
            return messages
        
        # 策略1: 保留首尾，压缩中间
        optimized = [messages[0]]  # 系统提示
        
        # 保留最近的对话
        recent = messages[-10:]
        
        # 中间部分做摘要
        middle = messages[1:-10]
        if middle:
            summary = self.summarize_messages(middle)
            optimized.append({
                "role": "system",
                "content": f"[历史对话摘要]: {summary}"
            })
        
        optimized.extend(recent)
        return optimized
    
    def should_use_lite_model(self, message: str) -> bool:
        """判断是否可以使用轻量模型"""
        # 简单问候、确认等使用轻量模型
        simple_patterns = [
            r'^(好的|嗯|哦|是的|对|不是|没有|谢谢)$',
            r'^.{1,10}$'  # 非常短的消息
        ]
        for pattern in simple_patterns:
            if re.match(pattern, message.strip()):
                return True
        return False
```

### 9.2 资源规格建议

| 阶段 | 用户量 | VKE | Redis | MySQL | 月成本估算 |
|------|--------|-----|-------|-------|-----------|
| MVP | 1千 | 2节点 4C8G | 2G主从 | 2C4G | ¥5,000 |
| 初期 | 1万 | 3节点 8C16G | 4G主从 | 4C8G | ¥14,000 |
| 中期 | 10万 | 10节点 8C16G | 16G主从 | 8C32G | ¥50,000 |
| 后期 | 100万 | 50节点 16C32G | 64G集群 | 16分片 | ¥830,000 |

---

## 10. 实施路线图

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              实施路线图                                              │
└─────────────────────────────────────────────────────────────────────────────────────┘

Phase 1: MVP (第1-4周)
├── Week 1: 基础架构
│   ├── VKE 集群搭建
│   ├── Redis 部署
│   └── MySQL 部署
├── Week 2: 核心服务
│   ├── 对话服务 (流式)
│   ├── 短期记忆 (方案A)
│   └── 模型调用 (主模型)
├── Week 3: 基础功能
│   ├── 角色管理
│   ├── 会话管理
│   └── API Gateway
└── Week 4: 测试部署
    ├── 单元测试
    ├── 集成测试
    └── 灰度发布

Phase 2: 完善 (第5-8周)
├── Week 5-6: 长期记忆
│   ├── VikingDB 集成
│   ├── 记忆生成 Worker
│   ├── 意图识别
│   └── 记忆召回
├── Week 7: 高可用
│   ├── 模型降级链
│   ├── 缓存降级
│   └── 熔断器
└── Week 8: 监控告警
    ├── APMPlus 集成
    ├── 告警配置
    └── Dashboard

Phase 3: 扩展 (第9-12周)
├── Week 9-10: 性能优化
│   ├── 本地缓存 (方案B升级)
│   ├── 连接池优化
│   └── 批量处理
├── Week 11: 可扩展性
│   ├── HPA 配置
│   ├── Redis Cluster
│   └── MySQL 读写分离
└── Week 12: 压测验证
    ├── 性能压测
    ├── 故障演练
    └── 上线准备
```
