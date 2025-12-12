# AI陪聊系统 - 长期记忆模块设计

## 1. 模块概述

### 1.1 长期记忆架构

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              长期记忆系统架构                                         │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                           记忆存储层 (三种存储协同)                           │    │
│  │                                                                              │    │
│  │   ┌─────────────────────┐                                                   │    │
│  │   │   火山云 VikingDB    │  向量数据库                                       │    │
│  │   │                     │  • 对话摘要向量 (语义检索)                         │    │
│  │   │   Collection:       │  • 关键事件向量                                   │    │
│  │   │   long_term_memory  │  • 用户偏好向量                                   │    │
│  │   │                     │  • Top-K 相似度检索                               │    │
│  │   └─────────────────────┘                                                   │    │
│  │              │                                                               │    │
│  │              │ vector_id 关联                                               │    │
│  │              ▼                                                               │    │
│  │   ┌─────────────────────┐                                                   │    │
│  │   │   火山云 MySQL       │  结构化数据库                                     │    │
│  │   │                     │  • 用户画像 (user_profiles)                       │    │
│  │   │   Tables:           │  • 关键事件 (user_events)                         │    │
│  │   │   • user_profiles   │  • 摘要索引 (conversation_summaries)              │    │
│  │   │   • user_events     │  • 支持精确查询和过滤                              │    │
│  │   │   • conv_summaries  │                                                   │    │
│  │   └─────────────────────┘                                                   │    │
│  │              │                                                               │    │
│  │              │ reference                                                    │    │
│  │              ▼                                                               │    │
│  │   ┌─────────────────────┐                                                   │    │
│  │   │   火山云 TOS         │  对象存储                                         │    │
│  │   │                     │  • 完整对话历史归档                                │    │
│  │   │   Bucket:           │  • 按用户/角色/日期组织                            │    │
│  │   │   conversation-     │  • 支持生命周期管理                                │    │
│  │   │   archive           │  • 低成本冷存储                                   │    │
│  │   └─────────────────────┘                                                   │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                           记忆生成层 (异步处理)                               │    │
│  │                                                                              │    │
│  │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │    │
│  │   │   Kafka     │───▶│  Memory     │───▶│  LLM        │───▶│  Storage    │  │    │
│  │   │   Consumer  │    │  Worker     │    │  Processing │    │  Writer     │  │    │
│  │   └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘  │    │
│  │                                                                              │    │
│  │   处理任务:                                                                  │    │
│  │   • 对话摘要生成                                                             │    │
│  │   • 用户画像提取                                                             │    │
│  │   • 关键事件识别                                                             │    │
│  │   • 向量化 & 存储                                                           │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                           记忆召回层 (实时查询)                               │    │
│  │                                                                              │    │
│  │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │    │
│  │   │   Intent    │───▶│  Parallel   │───▶│  Result     │───▶│  Context    │  │    │
│  │   │   Detector  │    │  Search     │    │  Merger     │    │  Injector   │  │    │
│  │   └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘  │    │
│  │                                                                              │    │
│  │   召回流程:                                                                  │    │
│  │   • 意图识别 (是否需要长期记忆)                                              │    │
│  │   • 并行检索 (VikingDB + MySQL)                                             │    │
│  │   • 结果融合排序                                                             │    │
│  │   • 注入对话上下文                                                           │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 数据流向

```
对话完成
    │
    ▼
┌─────────────────┐
│  短期记忆       │  Redis (实时)
│  (最近24小时)   │
└────────┬────────┘
         │ 触发条件: 会话结束 / 消息累积 / 定时任务
         ▼
┌─────────────────┐
│  Kafka 队列     │  异步解耦
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────┐
│  Memory Worker  │────▶│  LLM 处理       │
│  (消费者)       │     │  • 摘要生成     │
└────────┬────────┘     │  • 画像提取     │
         │              │  • 事件识别     │
         │              └─────────────────┘
         ▼
┌─────────────────────────────────────────┐
│              持久化存储                  │
│                                         │
│  VikingDB ◄── 向量化 ──► MySQL          │
│  (向量)                  (结构化)        │
│                    │                    │
│                    ▼                    │
│                   TOS                   │
│                (归档)                   │
└─────────────────────────────────────────┘
```

---

## 2. 数据模型设计

### 2.1 MySQL 表结构

```sql
-- ============================================
-- 用户画像表
-- 存储从对话中提取的用户个人信息
-- ============================================
CREATE TABLE user_profiles (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    
    -- 业务主键
    user_id VARCHAR(64) NOT NULL COMMENT '用户ID',
    character_id VARCHAR(64) NOT NULL COMMENT '角色ID',
    
    -- 画像内容
    profile_type ENUM('basic_info', 'preference', 'personality', 'relationship') 
        NOT NULL COMMENT '画像类型',
    profile_key VARCHAR(128) NOT NULL COMMENT '画像键名',
    profile_value TEXT NOT NULL COMMENT '画像值',
    
    -- 元数据
    confidence DECIMAL(3,2) DEFAULT 1.00 COMMENT '置信度 0-1',
    source_msg_id VARCHAR(64) COMMENT '来源消息ID',
    extracted_at TIMESTAMP NOT NULL COMMENT '提取时间',
    
    -- 系统字段
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    is_deleted TINYINT(1) DEFAULT 0,
    
    -- 索引
    UNIQUE KEY uk_user_profile (user_id, character_id, profile_type, profile_key),
    INDEX idx_user_char (user_id, character_id),
    INDEX idx_type_key (profile_type, profile_key),
    INDEX idx_updated (updated_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
COMMENT='用户画像表';

-- 画像类型说明:
-- basic_info: 基本信息 (name, age, birthday, occupation, location)
-- preference: 偏好 (favorite_food, hobbies, music_taste)
-- personality: 性格特点 (introvert, optimistic)
-- relationship: 与AI的关系 (how_we_met, nickname_for_ai)


-- ============================================
-- 关键事件表
-- 存储用户提到的重要事件
-- ============================================
CREATE TABLE user_events (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    
    -- 业务主键
    user_id VARCHAR(64) NOT NULL COMMENT '用户ID',
    character_id VARCHAR(64) NOT NULL COMMENT '角色ID',
    
    -- 事件内容
    event_type VARCHAR(64) NOT NULL COMMENT '事件类型',
    event_title VARCHAR(256) NOT NULL COMMENT '事件标题',
    event_summary TEXT NOT NULL COMMENT '事件摘要',
    event_date DATE COMMENT '事件日期',
    
    -- 情感和重要性
    emotion_tag VARCHAR(32) COMMENT '情感标签',
    importance TINYINT UNSIGNED DEFAULT 5 COMMENT '重要性 1-10',
    
    -- 向量关联
    vector_id VARCHAR(128) COMMENT 'VikingDB向量ID',
    
    -- 元数据
    source_summary_id BIGINT UNSIGNED COMMENT '来源摘要ID',
    
    -- 系统字段
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    is_deleted TINYINT(1) DEFAULT 0,
    
    -- 索引
    INDEX idx_user_char (user_id, character_id),
    INDEX idx_event_type (event_type),
    INDEX idx_event_date (event_date),
    INDEX idx_importance (importance DESC),
    INDEX idx_vector (vector_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
COMMENT='用户关键事件表';

-- 事件类型说明:
-- birthday: 生日
-- anniversary: 纪念日
-- achievement: 成就
-- life_event: 人生大事 (毕业、入职、结婚等)
-- emotional: 情感事件 (开心/难过的事)
-- plan: 计划/愿望
-- memory: 回忆


-- ============================================
-- 对话摘要表
-- 存储对话摘要及其向量索引
-- ============================================
CREATE TABLE conversation_summaries (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    
    -- 业务主键
    user_id VARCHAR(64) NOT NULL COMMENT '用户ID',
    character_id VARCHAR(64) NOT NULL COMMENT '角色ID',
    
    -- 摘要内容
    summary_text TEXT NOT NULL COMMENT '摘要文本',
    topic_tags JSON COMMENT '话题标签',
    emotion_tone VARCHAR(32) COMMENT '情感基调',
    
    -- 对话元信息
    start_time TIMESTAMP NOT NULL COMMENT '对话开始时间',
    end_time TIMESTAMP NOT NULL COMMENT '对话结束时间',
    message_count INT UNSIGNED NOT NULL COMMENT '消息数量',
    total_tokens INT UNSIGNED COMMENT '总Token数',
    
    -- 向量关联
    vector_id VARCHAR(128) NOT NULL COMMENT 'VikingDB向量ID',
    
    -- TOS归档关联
    archive_path VARCHAR(512) COMMENT 'TOS归档路径',
    
    -- 系统字段
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_deleted TINYINT(1) DEFAULT 0,
    
    -- 索引
    INDEX idx_user_char (user_id, character_id),
    INDEX idx_time_range (start_time, end_time),
    INDEX idx_vector (vector_id),
    INDEX idx_created (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
COMMENT='对话摘要表';


-- ============================================
-- 记忆生成任务表 (用于追踪和重试)
-- ============================================
CREATE TABLE memory_generation_tasks (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    
    -- 任务标识
    task_id VARCHAR(64) NOT NULL UNIQUE COMMENT '任务ID',
    user_id VARCHAR(64) NOT NULL,
    character_id VARCHAR(64) NOT NULL,
    
    -- 任务内容
    task_type ENUM('summary', 'profile', 'event', 'full') NOT NULL,
    source_data JSON NOT NULL COMMENT '源数据(消息列表)',
    
    -- 任务状态
    status ENUM('pending', 'processing', 'completed', 'failed') DEFAULT 'pending',
    retry_count TINYINT UNSIGNED DEFAULT 0,
    max_retries TINYINT UNSIGNED DEFAULT 3,
    error_message TEXT,
    
    -- 时间追踪
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    
    -- 索引
    INDEX idx_status (status),
    INDEX idx_user_char (user_id, character_id),
    INDEX idx_created (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
COMMENT='记忆生成任务表';
```

### 2.2 VikingDB Collection 设计

```python
# VikingDB Collection 配置
VIKINGDB_COLLECTION_CONFIG = {
    "collection_name": "ai_companion_long_memory",
    "description": "AI陪聊系统长期记忆向量存储",
    
    # 字段定义
    "fields": [
        {
            "field_name": "id",
            "field_type": "string",
            "is_primary_key": True,
            "description": "唯一标识 (UUID)"
        },
        {
            "field_name": "vector",
            "field_type": "vector",
            "dim": 1536,  # doubao embedding 维度
            "description": "文本向量"
        },
        {
            "field_name": "user_id",
            "field_type": "string",
            "description": "用户ID"
        },
        {
            "field_name": "character_id",
            "field_type": "string",
            "description": "角色ID"
        },
        {
            "field_name": "memory_type",
            "field_type": "string",
            "description": "记忆类型: summary/event/profile"
        },
        {
            "field_name": "content",
            "field_type": "string",
            "description": "记忆内容文本"
        },
        {
            "field_name": "timestamp",
            "field_type": "int64",
            "description": "时间戳(秒)"
        },
        {
            "field_name": "importance",
            "field_type": "int64",
            "description": "重要性评分 1-10"
        },
        {
            "field_name": "metadata",
            "field_type": "string",
            "description": "JSON格式元数据"
        }
    ],
    
    # 索引配置
    "index": {
        "index_type": "HNSW",
        "metric_type": "IP",  # Inner Product (余弦相似度)
        "params": {
            "M": 16,              # 每层连接数
            "efConstruction": 256  # 构建时搜索范围
        }
    }
}
```

### 2.3 TOS 存储结构

```
Bucket: ai-companion-archive
│
├── conversations/
│   └── {user_id}/
│       └── {character_id}/
│           └── {year}/
│               └── {month}/
│                   └── {day}/
│                       └── {session_id}.json
│
└── exports/
    └── {user_id}/
        └── full_export_{timestamp}.json

# 单个对话存档格式
{
    "session_id": "uuid",
    "user_id": "user_123",
    "character_id": "char_456",
    "start_time": "2024-01-15T10:30:00Z",
    "end_time": "2024-01-15T11:45:00Z",
    "messages": [
        {
            "msg_id": "uuid",
            "role": "user",
            "content": "...",
            "timestamp": 1705315800
        },
        ...
    ],
    "summary": {
        "text": "对话摘要...",
        "topics": ["工作", "压力"],
        "emotion": "neutral"
    },
    "extracted_profiles": [...],
    "extracted_events": [...]
}
```

---

## 3. Python 实现代码

### 3.1 数据模型

```python
# models/long_memory.py

from datetime import datetime, date
from typing import Optional, List, Dict, Any
from pydantic import BaseModel, Field
from enum import Enum
import uuid
import json


class ProfileType(str, Enum):
    """画像类型"""
    BASIC_INFO = "basic_info"       # 基本信息
    PREFERENCE = "preference"        # 偏好
    PERSONALITY = "personality"      # 性格
    RELATIONSHIP = "relationship"    # 关系


class EventType(str, Enum):
    """事件类型"""
    BIRTHDAY = "birthday"
    ANNIVERSARY = "anniversary"
    ACHIEVEMENT = "achievement"
    LIFE_EVENT = "life_event"
    EMOTIONAL = "emotional"
    PLAN = "plan"
    MEMORY = "memory"


class EmotionTag(str, Enum):
    """情感标签"""
    POSITIVE = "positive"
    NEGATIVE = "negative"
    NEUTRAL = "neutral"
    EXCITED = "excited"
    SAD = "sad"
    ANXIOUS = "anxious"


class MemoryType(str, Enum):
    """长期记忆类型"""
    SUMMARY = "summary"
    EVENT = "event"
    PROFILE = "profile"


# ============================================
# 用户画像模型
# ============================================
class UserProfile(BaseModel):
    """用户画像"""
    id: Optional[int] = None
    user_id: str
    character_id: str
    profile_type: ProfileType
    profile_key: str
    profile_value: str
    confidence: float = Field(default=1.0, ge=0, le=1)
    source_msg_id: Optional[str] = None
    extracted_at: datetime = Field(default_factory=datetime.now)
    
    class Config:
        use_enum_values = True


class UserProfileExtraction(BaseModel):
    """LLM提取的用户画像结构"""
    profiles: List[Dict[str, Any]]
    # 示例: [{"type": "basic_info", "key": "name", "value": "小明", "confidence": 0.95}]


# ============================================
# 关键事件模型
# ============================================
class UserEvent(BaseModel):
    """用户关键事件"""
    id: Optional[int] = None
    user_id: str
    character_id: str
    event_type: EventType
    event_title: str
    event_summary: str
    event_date: Optional[date] = None
    emotion_tag: Optional[EmotionTag] = None
    importance: int = Field(default=5, ge=1, le=10)
    vector_id: Optional[str] = None
    source_summary_id: Optional[int] = None
    
    class Config:
        use_enum_values = True


class EventExtraction(BaseModel):
    """LLM提取的事件结构"""
    events: List[Dict[str, Any]]
    # 示例: [{"type": "birthday", "title": "用户生日", "summary": "...", "date": "03-15", "importance": 8}]


# ============================================
# 对话摘要模型
# ============================================
class ConversationSummary(BaseModel):
    """对话摘要"""
    id: Optional[int] = None
    user_id: str
    character_id: str
    summary_text: str
    topic_tags: List[str] = []
    emotion_tone: Optional[EmotionTag] = None
    start_time: datetime
    end_time: datetime
    message_count: int
    total_tokens: Optional[int] = None
    vector_id: str
    archive_path: Optional[str] = None
    
    class Config:
        use_enum_values = True


class SummaryExtraction(BaseModel):
    """LLM生成的摘要结构"""
    summary: str
    topics: List[str]
    emotion: str


# ============================================
# 向量记忆模型 (VikingDB)
# ============================================
class VectorMemory(BaseModel):
    """向量记忆"""
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    vector: List[float]
    user_id: str
    character_id: str
    memory_type: MemoryType
    content: str
    timestamp: int = Field(default_factory=lambda: int(datetime.now().timestamp()))
    importance: int = Field(default=5, ge=1, le=10)
    metadata: Dict[str, Any] = {}
    
    def to_vikingdb_format(self) -> Dict:
        """转换为VikingDB插入格式"""
        return {
            "id": self.id,
            "vector": self.vector,
            "user_id": self.user_id,
            "character_id": self.character_id,
            "memory_type": self.memory_type.value,
            "content": self.content,
            "timestamp": self.timestamp,
            "importance": self.importance,
            "metadata": json.dumps(self.metadata)
        }


# ============================================
# 记忆召回结果
# ============================================
class RecalledMemory(BaseModel):
    """召回的记忆"""
    memory_type: MemoryType
    content: str
    timestamp: int
    importance: int
    relevance_score: float  # 相关性得分
    source: str  # "vikingdb" / "mysql"
    
    def to_context_string(self) -> str:
        """转换为上下文注入格式"""
        time_str = datetime.fromtimestamp(self.timestamp).strftime("%Y-%m-%d")
        return f"[{time_str}] {self.content}"


class RecallResult(BaseModel):
    """召回结果"""
    memories: List[RecalledMemory]
    profiles: List[UserProfile]
    total_tokens: int = 0
```

### 3.2 VikingDB 客户端

```python
# clients/vikingdb_client.py

import logging
from typing import List, Dict, Any, Optional
import asyncio
from volcengine.viking_db import VikingDBService

logger = logging.getLogger(__name__)


class VikingDBConfig:
    """VikingDB 配置"""
    HOST: str = "vikingdb-xxx.volces.com"
    REGION: str = "cn-beijing"
    AK: str = "your-access-key"
    SK: str = "your-secret-key"
    
    COLLECTION_NAME: str = "ai_companion_long_memory"
    
    # 检索参数
    DEFAULT_TOP_K: int = 10
    DEFAULT_EF: int = 128
    
    # 超时配置
    TIMEOUT: float = 2.0


class VikingDBClient:
    """VikingDB 客户端"""
    
    _instance: Optional["VikingDBClient"] = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._initialized = False
        return cls._instance
    
    def __init__(self):
        if self._initialized:
            return
        
        self._client = VikingDBService(
            host=VikingDBConfig.HOST,
            region=VikingDBConfig.REGION,
            ak=VikingDBConfig.AK,
            sk=VikingDBConfig.SK,
        )
        self._collection = self._client.get_collection(VikingDBConfig.COLLECTION_NAME)
        self._initialized = True
    
    async def insert(self, data: Dict[str, Any]) -> bool:
        """
        插入向量数据
        
        Args:
            data: 向量数据字典
            
        Returns:
            是否成功
        """
        try:
            # VikingDB SDK 同步调用，包装为异步
            loop = asyncio.get_event_loop()
            await loop.run_in_executor(
                None,
                lambda: self._collection.upsert_data([data])
            )
            logger.debug(f"Inserted vector: {data.get('id')}")
            return True
        except Exception as e:
            logger.error(f"VikingDB insert error: {e}")
            return False
    
    async def batch_insert(self, data_list: List[Dict[str, Any]]) -> bool:
        """批量插入向量数据"""
        if not data_list:
            return True
        
        try:
            loop = asyncio.get_event_loop()
            await loop.run_in_executor(
                None,
                lambda: self._collection.upsert_data(data_list)
            )
            logger.debug(f"Batch inserted {len(data_list)} vectors")
            return True
        except Exception as e:
            logger.error(f"VikingDB batch insert error: {e}")
            return False
    
    async def search(
        self,
        vector: List[float],
        user_id: str,
        character_id: str,
        memory_types: Optional[List[str]] = None,
        top_k: int = None,
        min_score: float = 0.5
    ) -> List[Dict[str, Any]]:
        """
        向量相似度检索
        
        Args:
            vector: 查询向量
            user_id: 用户ID
            character_id: 角色ID
            memory_types: 记忆类型过滤
            top_k: 返回数量
            min_score: 最小相似度阈值
            
        Returns:
            检索结果列表
        """
        top_k = top_k or VikingDBConfig.DEFAULT_TOP_K
        
        # 构建过滤条件
        filter_conditions = {
            "op": "and",
            "conditions": [
                {"field": "user_id", "op": "=", "value": user_id},
                {"field": "character_id", "op": "=", "value": character_id},
            ]
        }
        
        if memory_types:
            filter_conditions["conditions"].append({
                "field": "memory_type",
                "op": "in",
                "value": memory_types
            })
        
        try:
            loop = asyncio.get_event_loop()
            results = await asyncio.wait_for(
                loop.run_in_executor(
                    None,
                    lambda: self._collection.search(
                        vectors=[vector],
                        filter=filter_conditions,
                        limit=top_k,
                        output_fields=["id", "user_id", "character_id", "memory_type", 
                                      "content", "timestamp", "importance", "metadata"],
                        params={"ef": VikingDBConfig.DEFAULT_EF}
                    )
                ),
                timeout=VikingDBConfig.TIMEOUT
            )
            
            # 过滤低相似度结果
            filtered = []
            for result in results[0]:  # results[0] 是第一个查询向量的结果
                if result.score >= min_score:
                    filtered.append({
                        "id": result.id,
                        "score": result.score,
                        **result.fields
                    })
            
            logger.debug(f"VikingDB search returned {len(filtered)} results")
            return filtered
            
        except asyncio.TimeoutError:
            logger.warning("VikingDB search timeout")
            return []
        except Exception as e:
            logger.error(f"VikingDB search error: {e}")
            return []
    
    async def delete(self, ids: List[str]) -> bool:
        """删除向量数据"""
        if not ids:
            return True
        
        try:
            loop = asyncio.get_event_loop()
            await loop.run_in_executor(
                None,
                lambda: self._collection.delete_data(ids)
            )
            return True
        except Exception as e:
            logger.error(f"VikingDB delete error: {e}")
            return False


# 全局实例
vikingdb_client = VikingDBClient()
```

### 3.3 MySQL 数据访问层

```python
# repositories/long_memory_repo.py

import logging
from typing import List, Optional, Dict, Any
from datetime import datetime, date
import aiomysql
from contextlib import asynccontextmanager

from models.long_memory import (
    UserProfile, UserEvent, ConversationSummary,
    ProfileType, EventType
)

logger = logging.getLogger(__name__)


class MySQLConfig:
    """MySQL 配置"""
    HOST: str = "mysql-xxx.volces.com"
    PORT: int = 3306
    USER: str = "root"
    PASSWORD: str = "your-password"
    DATABASE: str = "ai_companion"
    
    # 连接池配置
    MIN_SIZE: int = 5
    MAX_SIZE: int = 20
    
    # 超时配置
    CONNECT_TIMEOUT: float = 5.0
    READ_TIMEOUT: float = 2.0


class MySQLPool:
    """MySQL 连接池管理"""
    
    _pool: Optional[aiomysql.Pool] = None
    
    @classmethod
    async def get_pool(cls) -> aiomysql.Pool:
        if cls._pool is None:
            cls._pool = await aiomysql.create_pool(
                host=MySQLConfig.HOST,
                port=MySQLConfig.PORT,
                user=MySQLConfig.USER,
                password=MySQLConfig.PASSWORD,
                db=MySQLConfig.DATABASE,
                minsize=MySQLConfig.MIN_SIZE,
                maxsize=MySQLConfig.MAX_SIZE,
                connect_timeout=MySQLConfig.CONNECT_TIMEOUT,
                charset='utf8mb4',
                autocommit=True,
            )
        return cls._pool
    
    @classmethod
    @asynccontextmanager
    async def acquire(cls):
        pool = await cls.get_pool()
        async with pool.acquire() as conn:
            async with conn.cursor(aiomysql.DictCursor) as cursor:
                yield cursor


class UserProfileRepository:
    """用户画像数据访问"""
    
    async def upsert(self, profile: UserProfile) -> int:
        """
        插入或更新用户画像
        
        Returns:
            affected rows
        """
        sql = """
            INSERT INTO user_profiles 
            (user_id, character_id, profile_type, profile_key, profile_value, 
             confidence, source_msg_id, extracted_at)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE
                profile_value = VALUES(profile_value),
                confidence = GREATEST(confidence, VALUES(confidence)),
                source_msg_id = VALUES(source_msg_id),
                extracted_at = VALUES(extracted_at),
                updated_at = CURRENT_TIMESTAMP
        """
        
        try:
            async with MySQLPool.acquire() as cursor:
                await cursor.execute(sql, (
                    profile.user_id,
                    profile.character_id,
                    profile.profile_type,
                    profile.profile_key,
                    profile.profile_value,
                    profile.confidence,
                    profile.source_msg_id,
                    profile.extracted_at,
                ))
                return cursor.rowcount
        except Exception as e:
            logger.error(f"Upsert profile error: {e}")
            return 0
    
    async def batch_upsert(self, profiles: List[UserProfile]) -> int:
        """批量插入或更新"""
        if not profiles:
            return 0
        
        total = 0
        for profile in profiles:
            total += await self.upsert(profile)
        return total
    
    async def get_by_user_character(
        self,
        user_id: str,
        character_id: str,
        profile_types: Optional[List[ProfileType]] = None
    ) -> List[UserProfile]:
        """获取用户画像"""
        sql = """
            SELECT * FROM user_profiles
            WHERE user_id = %s AND character_id = %s AND is_deleted = 0
        """
        params = [user_id, character_id]
        
        if profile_types:
            placeholders = ','.join(['%s'] * len(profile_types))
            sql += f" AND profile_type IN ({placeholders})"
            params.extend([t.value for t in profile_types])
        
        sql += " ORDER BY confidence DESC, updated_at DESC"
        
        try:
            async with MySQLPool.acquire() as cursor:
                await cursor.execute(sql, params)
                rows = await cursor.fetchall()
                return [UserProfile(**row) for row in rows]
        except Exception as e:
            logger.error(f"Get profiles error: {e}")
            return []


class UserEventRepository:
    """用户事件数据访问"""
    
    async def insert(self, event: UserEvent) -> int:
        """插入事件"""
        sql = """
            INSERT INTO user_events
            (user_id, character_id, event_type, event_title, event_summary,
             event_date, emotion_tag, importance, vector_id, source_summary_id)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """
        
        try:
            async with MySQLPool.acquire() as cursor:
                await cursor.execute(sql, (
                    event.user_id,
                    event.character_id,
                    event.event_type,
                    event.event_title,
                    event.event_summary,
                    event.event_date,
                    event.emotion_tag,
                    event.importance,
                    event.vector_id,
                    event.source_summary_id,
                ))
                return cursor.lastrowid
        except Exception as e:
            logger.error(f"Insert event error: {e}")
            return 0
    
    async def batch_insert(self, events: List[UserEvent]) -> List[int]:
        """批量插入事件"""
        ids = []
        for event in events:
            event_id = await self.insert(event)
            if event_id:
                ids.append(event_id)
        return ids
    
    async def get_by_user_character(
        self,
        user_id: str,
        character_id: str,
        event_types: Optional[List[EventType]] = None,
        min_importance: int = 1,
        limit: int = 20
    ) -> List[UserEvent]:
        """获取用户事件"""
        sql = """
            SELECT * FROM user_events
            WHERE user_id = %s AND character_id = %s 
            AND is_deleted = 0 AND importance >= %s
        """
        params = [user_id, character_id, min_importance]
        
        if event_types:
            placeholders = ','.join(['%s'] * len(event_types))
            sql += f" AND event_type IN ({placeholders})"
            params.extend([t.value for t in event_types])
        
        sql += " ORDER BY importance DESC, created_at DESC LIMIT %s"
        params.append(limit)
        
        try:
            async with MySQLPool.acquire() as cursor:
                await cursor.execute(sql, params)
                rows = await cursor.fetchall()
                return [UserEvent(**row) for row in rows]
        except Exception as e:
            logger.error(f"Get events error: {e}")
            return []
    
    async def get_upcoming_events(
        self,
        user_id: str,
        character_id: str,
        days_ahead: int = 30
    ) -> List[UserEvent]:
        """获取即将到来的事件（如生日）"""
        sql = """
            SELECT * FROM user_events
            WHERE user_id = %s AND character_id = %s AND is_deleted = 0
            AND event_date IS NOT NULL
            AND (
                (MONTH(event_date) = MONTH(CURDATE()) AND DAY(event_date) >= DAY(CURDATE()))
                OR (MONTH(event_date) = MONTH(DATE_ADD(CURDATE(), INTERVAL %s DAY)))
            )
            ORDER BY 
                CASE 
                    WHEN MONTH(event_date) >= MONTH(CURDATE()) 
                    THEN MONTH(event_date) 
                    ELSE MONTH(event_date) + 12 
                END,
                DAY(event_date)
            LIMIT 10
        """
        
        try:
            async with MySQLPool.acquire() as cursor:
                await cursor.execute(sql, (user_id, character_id, days_ahead))
                rows = await cursor.fetchall()
                return [UserEvent(**row) for row in rows]
        except Exception as e:
            logger.error(f"Get upcoming events error: {e}")
            return []


class ConversationSummaryRepository:
    """对话摘要数据访问"""
    
    async def insert(self, summary: ConversationSummary) -> int:
        """插入摘要"""
        import json
        
        sql = """
            INSERT INTO conversation_summaries
            (user_id, character_id, summary_text, topic_tags, emotion_tone,
             start_time, end_time, message_count, total_tokens, vector_id, archive_path)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """
        
        try:
            async with MySQLPool.acquire() as cursor:
                await cursor.execute(sql, (
                    summary.user_id,
                    summary.character_id,
                    summary.summary_text,
                    json.dumps(summary.topic_tags),
                    summary.emotion_tone,
                    summary.start_time,
                    summary.end_time,
                    summary.message_count,
                    summary.total_tokens,
                    summary.vector_id,
                    summary.archive_path,
                ))
                return cursor.lastrowid
        except Exception as e:
            logger.error(f"Insert summary error: {e}")
            return 0
    
    async def get_recent(
        self,
        user_id: str,
        character_id: str,
        days: int = 30,
        limit: int = 20
    ) -> List[ConversationSummary]:
        """获取最近的对话摘要"""
        sql = """
            SELECT * FROM conversation_summaries
            WHERE user_id = %s AND character_id = %s AND is_deleted = 0
            AND created_at >= DATE_SUB(NOW(), INTERVAL %s DAY)
            ORDER BY created_at DESC
            LIMIT %s
        """
        
        try:
            async with MySQLPool.acquire() as cursor:
                await cursor.execute(sql, (user_id, character_id, days, limit))
                rows = await cursor.fetchall()
                return [ConversationSummary(**row) for row in rows]
        except Exception as e:
            logger.error(f"Get recent summaries error: {e}")
            return []


# 全局实例
profile_repo = UserProfileRepository()
event_repo = UserEventRepository()
summary_repo = ConversationSummaryRepository()
```

### 3.4 TOS 客户端

```python
# clients/tos_client.py

import logging
import json
from typing import Optional, Dict, Any
from datetime import datetime
import asyncio

import tos

logger = logging.getLogger(__name__)


class TOSConfig:
    """TOS 配置"""
    ENDPOINT: str = "tos-cn-beijing.volces.com"
    REGION: str = "cn-beijing"
    AK: str = "your-access-key"
    SK: str = "your-secret-key"
    
    # Bucket 配置
    ARCHIVE_BUCKET: str = "ai-companion-archive"
    
    # 路径模板
    CONVERSATION_PATH_TEMPLATE: str = "conversations/{user_id}/{character_id}/{year}/{month}/{day}/{session_id}.json"


class TOSClient:
    """TOS 客户端"""
    
    _instance: Optional["TOSClient"] = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._initialized = False
        return cls._instance
    
    def __init__(self):
        if self._initialized:
            return
        
        self._client = tos.TosClientV2(
            ak=TOSConfig.AK,
            sk=TOSConfig.SK,
            endpoint=TOSConfig.ENDPOINT,
            region=TOSConfig.REGION,
        )
        self._initialized = True
    
    def _generate_conversation_path(
        self,
        user_id: str,
        character_id: str,
        session_id: str,
        timestamp: datetime = None
    ) -> str:
        """生成对话归档路径"""
        timestamp = timestamp or datetime.now()
        return TOSConfig.CONVERSATION_PATH_TEMPLATE.format(
            user_id=user_id,
            character_id=character_id,
            year=timestamp.strftime("%Y"),
            month=timestamp.strftime("%m"),
            day=timestamp.strftime("%d"),
            session_id=session_id,
        )
    
    async def archive_conversation(
        self,
        user_id: str,
        character_id: str,
        session_id: str,
        conversation_data: Dict[str, Any]
    ) -> Optional[str]:
        """
        归档对话到TOS
        
        Args:
            user_id: 用户ID
            character_id: 角色ID
            session_id: 会话ID
            conversation_data: 对话数据
            
        Returns:
            存储路径，失败返回None
        """
        path = self._generate_conversation_path(user_id, character_id, session_id)
        content = json.dumps(conversation_data, ensure_ascii=False, indent=2)
        
        try:
            loop = asyncio.get_event_loop()
            await loop.run_in_executor(
                None,
                lambda: self._client.put_object(
                    bucket=TOSConfig.ARCHIVE_BUCKET,
                    key=path,
                    content=content.encode('utf-8'),
                    content_type='application/json',
                )
            )
            logger.info(f"Archived conversation to: {path}")
            return path
            
        except Exception as e:
            logger.error(f"TOS archive error: {e}")
            return None
    
    async def get_conversation(
        self,
        path: str
    ) -> Optional[Dict[str, Any]]:
        """获取归档的对话"""
        try:
            loop = asyncio.get_event_loop()
            response = await loop.run_in_executor(
                None,
                lambda: self._client.get_object(
                    bucket=TOSConfig.ARCHIVE_BUCKET,
                    key=path,
                )
            )
            content = response.read().decode('utf-8')
            return json.loads(content)
            
        except Exception as e:
            logger.error(f"TOS get error: {e}")
            return None


# 全局实例
tos_client = TOSClient()
```

### 3.5 Embedding 服务

```python
# services/embedding_service.py

import logging
from typing import List, Optional
import asyncio
import httpx

logger = logging.getLogger(__name__)


class EmbeddingConfig:
    """Embedding 配置"""
    # 火山方舟 Embedding API
    API_URL: str = "https://ark.cn-beijing.volces.com/api/v3/embeddings"
    API_KEY: str = "your-api-key"
    MODEL: str = "doubao-embedding"
    
    # 请求配置
    TIMEOUT: float = 10.0
    MAX_RETRIES: int = 2
    
    # 向量维度
    DIMENSION: int = 1536


class EmbeddingService:
    """Embedding 服务"""
    
    def __init__(self):
        self.client = httpx.AsyncClient(
            timeout=EmbeddingConfig.TIMEOUT,
            headers={
                "Authorization": f"Bearer {EmbeddingConfig.API_KEY}",
                "Content-Type": "application/json",
            }
        )
    
    async def embed_text(self, text: str) -> Optional[List[float]]:
        """
        文本向量化
        
        Args:
            text: 输入文本
            
        Returns:
            向量，失败返回None
        """
        return await self.embed_texts([text])
    
    async def embed_texts(self, texts: List[str]) -> Optional[List[List[float]]]:
        """
        批量文本向量化
        
        Args:
            texts: 文本列表
            
        Returns:
            向量列表，失败返回None
        """
        if not texts:
            return []
        
        for attempt in range(EmbeddingConfig.MAX_RETRIES):
            try:
                response = await self.client.post(
                    EmbeddingConfig.API_URL,
                    json={
                        "model": EmbeddingConfig.MODEL,
                        "input": texts,
                    }
                )
                response.raise_for_status()
                
                result = response.json()
                embeddings = [item["embedding"] for item in result["data"]]
                
                logger.debug(f"Generated {len(embeddings)} embeddings")
                return embeddings
                
            except Exception as e:
                logger.warning(f"Embedding attempt {attempt + 1} failed: {e}")
                if attempt < EmbeddingConfig.MAX_RETRIES - 1:
                    await asyncio.sleep(0.5 * (attempt + 1))
                continue
        
        logger.error(f"All embedding attempts failed for {len(texts)} texts")
        return None
    
    async def close(self):
        await self.client.aclose()


# 全局实例
embedding_service = EmbeddingService()
```

---

## 4. 记忆生成 Worker

### 4.1 Kafka 消费者

```python
# workers/memory_worker.py

import logging
import json
import asyncio
from typing import List, Dict, Any, Optional
from datetime import datetime
import uuid

from aiokafka import AIOKafkaConsumer
from models.message import Message
from models.long_memory import (
    UserProfile, UserEvent, ConversationSummary,
    VectorMemory, MemoryType, ProfileType, EventType,
    SummaryExtraction, UserProfileExtraction, EventExtraction
)
from services.llm_service import llm_service
from services.embedding_service import embedding_service
from clients.vikingdb_client import vikingdb_client
from clients.tos_client import tos_client
from repositories.long_memory_repo import profile_repo, event_repo, summary_repo

logger = logging.getLogger(__name__)


class MemoryWorkerConfig:
    """Worker 配置"""
    KAFKA_BOOTSTRAP_SERVERS: str = "kafka-xxx:9092"
    KAFKA_TOPIC: str = "memory_generation_tasks"
    KAFKA_GROUP_ID: str = "memory-worker-group"
    
    # 处理配置
    BATCH_SIZE: int = 10
    PROCESS_TIMEOUT: float = 60.0


class MemoryGenerationTask:
    """记忆生成任务"""
    def __init__(self, data: Dict[str, Any]):
        self.task_id = data.get("task_id", str(uuid.uuid4()))
        self.user_id = data["user_id"]
        self.character_id = data["character_id"]
        self.session_id = data.get("session_id", str(uuid.uuid4()))
        self.messages = [Message(**m) for m in data["messages"]]
        self.trigger = data.get("trigger", "session_end")  # session_end / threshold / scheduled


class MemoryWorker:
    """记忆生成 Worker"""
    
    def __init__(self):
        self.consumer: Optional[AIOKafkaConsumer] = None
        self.running = False
    
    async def start(self):
        """启动 Worker"""
        self.consumer = AIOKafkaConsumer(
            MemoryWorkerConfig.KAFKA_TOPIC,
            bootstrap_servers=MemoryWorkerConfig.KAFKA_BOOTSTRAP_SERVERS,
            group_id=MemoryWorkerConfig.KAFKA_GROUP_ID,
            auto_offset_reset="earliest",
            enable_auto_commit=False,
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
        )
        
        await self.consumer.start()
        self.running = True
        
        logger.info("Memory Worker started")
        
        try:
            await self._consume_loop()
        finally:
            await self.consumer.stop()
    
    async def stop(self):
        """停止 Worker"""
        self.running = False
    
    async def _consume_loop(self):
        """消费循环"""
        while self.running:
            try:
                # 批量获取消息
                messages = await self.consumer.getmany(
                    timeout_ms=1000,
                    max_records=MemoryWorkerConfig.BATCH_SIZE
                )
                
                for tp, msgs in messages.items():
                    for msg in msgs:
                        try:
                            task = MemoryGenerationTask(msg.value)
                            await self._process_task(task)
                            
                            # 手动提交offset
                            await self.consumer.commit()
                            
                        except Exception as e:
                            logger.error(f"Process task error: {e}")
                            # 发送到死信队列或记录失败
                            await self._handle_failure(msg.value, str(e))
                            await self.consumer.commit()
                            
            except Exception as e:
                logger.error(f"Consume loop error: {e}")
                await asyncio.sleep(1)
    
    async def _process_task(self, task: MemoryGenerationTask):
        """
        处理记忆生成任务
        
        流程:
        1. 生成对话摘要 (LLM)
        2. 提取用户画像 (LLM)
        3. 识别关键事件 (LLM)
        4. 向量化并存储
        5. 归档原始对话
        """
        logger.info(f"Processing task: {task.task_id}")
        
        # 将消息列表转为对话文本
        conversation_text = self._format_conversation(task.messages)
        
        # 1. 并行执行 LLM 分析
        summary_task = self._generate_summary(conversation_text)
        profile_task = self._extract_profiles(conversation_text)
        event_task = self._extract_events(conversation_text)
        
        summary_result, profile_result, event_result = await asyncio.gather(
            summary_task, profile_task, event_task,
            return_exceptions=True
        )
        
        # 2. 处理摘要
        summary_id = None
        if isinstance(summary_result, SummaryExtraction):
            summary_id = await self._save_summary(task, summary_result)
        else:
            logger.warning(f"Summary generation failed: {summary_result}")
        
        # 3. 处理画像
        if isinstance(profile_result, UserProfileExtraction):
            await self._save_profiles(task, profile_result)
        else:
            logger.warning(f"Profile extraction failed: {profile_result}")
        
        # 4. 处理事件
        if isinstance(event_result, EventExtraction):
            await self._save_events(task, event_result, summary_id)
        else:
            logger.warning(f"Event extraction failed: {event_result}")
        
        # 5. 归档原始对话
        await self._archive_conversation(task, summary_result)
        
        logger.info(f"Task completed: {task.task_id}")
    
    def _format_conversation(self, messages: List[Message]) -> str:
        """格式化对话为文本"""
        lines = []
        for msg in messages:
            role = "用户" if msg.role == "user" else "AI"
            lines.append(f"{role}: {msg.content}")
        return "\n".join(lines)
    
    async def _generate_summary(self, conversation: str) -> SummaryExtraction:
        """生成对话摘要"""
        prompt = f"""请分析以下对话并生成摘要。

对话内容：
{conversation}

请输出JSON格式：
{{
    "summary": "100字以内的对话摘要",
    "topics": ["话题1", "话题2"],
    "emotion": "positive/negative/neutral"
}}

只输出JSON，不要其他内容。"""
        
        response = await llm_service.chat_simple(
            messages=[{"role": "user", "content": prompt}],
            model="doubao-pro-32k",  # 使用较便宜的模型
            temperature=0.3
        )
        
        result = json.loads(response)
        return SummaryExtraction(**result)
    
    async def _extract_profiles(self, conversation: str) -> UserProfileExtraction:
        """提取用户画像"""
        prompt = f"""从以下对话中提取用户透露的个人信息。

对话内容：
{conversation}

可提取的信息类型：
- basic_info: 姓名、年龄、生日、职业、居住地
- preference: 喜好、厌恶、习惯
- personality: 性格特点
- relationship: 与AI的关系

请输出JSON格式：
{{
    "profiles": [
        {{"type": "basic_info", "key": "name", "value": "小明", "confidence": 0.95}},
        {{"type": "preference", "key": "favorite_food", "value": "火锅", "confidence": 0.8}}
    ]
}}

注意：
- 只提取用户明确说出的信息
- confidence 表示置信度 (0-1)
- 如果没有发现任何信息，返回空列表

只输出JSON，不要其他内容。"""
        
        response = await llm_service.chat_simple(
            messages=[{"role": "user", "content": prompt}],
            model="doubao-pro-32k",
            temperature=0.3
        )
        
        result = json.loads(response)
        return UserProfileExtraction(**result)
    
    async def _extract_events(self, conversation: str) -> EventExtraction:
        """识别关键事件"""
        prompt = f"""从以下对话中识别用户提到的重要事件。

对话内容：
{conversation}

事件类型：
- birthday: 生日
- anniversary: 纪念日
- achievement: 成就
- life_event: 人生大事（毕业、入职、结婚等）
- emotional: 情感事件（开心/难过的事）
- plan: 计划/愿望
- memory: 回忆

请输出JSON格式：
{{
    "events": [
        {{
            "type": "birthday",
            "title": "用户生日",
            "summary": "用户说自己的生日是3月15日",
            "date": "03-15",
            "emotion": "positive",
            "importance": 8
        }}
    ]
}}

注意：
- importance 范围 1-10
- date 格式为 MM-DD 或 YYYY-MM-DD，如不确定可为null
- 如果没有发现事件，返回空列表

只输出JSON，不要其他内容。"""
        
        response = await llm_service.chat_simple(
            messages=[{"role": "user", "content": prompt}],
            model="doubao-pro-32k",
            temperature=0.3
        )
        
        result = json.loads(response)
        return EventExtraction(**result)
    
    async def _save_summary(
        self,
        task: MemoryGenerationTask,
        summary_result: SummaryExtraction
    ) -> Optional[int]:
        """保存摘要"""
        # 1. 生成向量
        vectors = await embedding_service.embed_texts([summary_result.summary])
        if not vectors:
            logger.error("Failed to generate summary vector")
            return None
        
        vector_id = str(uuid.uuid4())
        
        # 2. 存储到 VikingDB
        vector_memory = VectorMemory(
            id=vector_id,
            vector=vectors[0],
            user_id=task.user_id,
            character_id=task.character_id,
            memory_type=MemoryType.SUMMARY,
            content=summary_result.summary,
            importance=5,
            metadata={
                "topics": summary_result.topics,
                "emotion": summary_result.emotion,
                "session_id": task.session_id,
            }
        )
        
        await vikingdb_client.insert(vector_memory.to_vikingdb_format())
        
        # 3. 存储到 MySQL
        summary = ConversationSummary(
            user_id=task.user_id,
            character_id=task.character_id,
            summary_text=summary_result.summary,
            topic_tags=summary_result.topics,
            emotion_tone=summary_result.emotion,
            start_time=datetime.fromtimestamp(task.messages[0].timestamp),
            end_time=datetime.fromtimestamp(task.messages[-1].timestamp),
            message_count=len(task.messages),
            total_tokens=sum(m.token_count for m in task.messages),
            vector_id=vector_id,
        )
        
        return await summary_repo.insert(summary)
    
    async def _save_profiles(
        self,
        task: MemoryGenerationTask,
        profile_result: UserProfileExtraction
    ):
        """保存用户画像"""
        profiles = []
        for p in profile_result.profiles:
            profile = UserProfile(
                user_id=task.user_id,
                character_id=task.character_id,
                profile_type=ProfileType(p["type"]),
                profile_key=p["key"],
                profile_value=p["value"],
                confidence=p.get("confidence", 1.0),
            )
            profiles.append(profile)
        
        if profiles:
            await profile_repo.batch_upsert(profiles)
            logger.info(f"Saved {len(profiles)} profiles for user {task.user_id}")
    
    async def _save_events(
        self,
        task: MemoryGenerationTask,
        event_result: EventExtraction,
        summary_id: Optional[int]
    ):
        """保存关键事件"""
        events = []
        vectors_to_insert = []
        
        for e in event_result.events:
            vector_id = str(uuid.uuid4())
            
            # 生成事件向量
            vectors = await embedding_service.embed_texts([e["summary"]])
            if vectors:
                vector_memory = VectorMemory(
                    id=vector_id,
                    vector=vectors[0],
                    user_id=task.user_id,
                    character_id=task.character_id,
                    memory_type=MemoryType.EVENT,
                    content=e["summary"],
                    importance=e.get("importance", 5),
                    metadata={
                        "event_type": e["type"],
                        "event_title": e["title"],
                    }
                )
                vectors_to_insert.append(vector_memory.to_vikingdb_format())
            
            # 解析日期
            event_date = None
            if e.get("date"):
                try:
                    if len(e["date"]) == 5:  # MM-DD
                        event_date = datetime.strptime(e["date"], "%m-%d").date()
                    else:  # YYYY-MM-DD
                        event_date = datetime.strptime(e["date"], "%Y-%m-%d").date()
                except:
                    pass
            
            event = UserEvent(
                user_id=task.user_id,
                character_id=task.character_id,
                event_type=EventType(e["type"]),
                event_title=e["title"],
                event_summary=e["summary"],
                event_date=event_date,
                emotion_tag=e.get("emotion"),
                importance=e.get("importance", 5),
                vector_id=vector_id if vectors else None,
                source_summary_id=summary_id,
            )
            events.append(event)
        
        # 批量插入向量
        if vectors_to_insert:
            await vikingdb_client.batch_insert(vectors_to_insert)
        
        # 批量插入事件
        if events:
            await event_repo.batch_insert(events)
            logger.info(f"Saved {len(events)} events for user {task.user_id}")
    
    async def _archive_conversation(
        self,
        task: MemoryGenerationTask,
        summary_result: Optional[SummaryExtraction]
    ):
        """归档对话到 TOS"""
        archive_data = {
            "session_id": task.session_id,
            "user_id": task.user_id,
            "character_id": task.character_id,
            "start_time": datetime.fromtimestamp(task.messages[0].timestamp).isoformat(),
            "end_time": datetime.fromtimestamp(task.messages[-1].timestamp).isoformat(),
            "messages": [m.dict() for m in task.messages],
            "summary": summary_result.dict() if summary_result else None,
        }
        
        await tos_client.archive_conversation(
            user_id=task.user_id,
            character_id=task.character_id,
            session_id=task.session_id,
            conversation_data=archive_data,
        )
    
    async def _handle_failure(self, task_data: Dict, error: str):
        """处理失败任务"""
        # TODO: 发送到死信队列或记录到数据库
        logger.error(f"Task failed: {task_data.get('task_id')}, error: {error}")


# 启动入口
async def main():
    worker = MemoryWorker()
    await worker.start()


if __name__ == "__main__":
    asyncio.run(main())
```

### 4.2 任务发布服务

```python
# services/memory_task_publisher.py

import logging
import json
from typing import List
from datetime import datetime
import uuid

from aiokafka import AIOKafkaProducer
from models.message import Message

logger = logging.getLogger(__name__)


class MemoryTaskPublisher:
    """记忆生成任务发布器"""
    
    def __init__(self):
        self.producer: AIOKafkaProducer = None
        self.topic = "memory_generation_tasks"
    
    async def start(self):
        self.producer = AIOKafkaProducer(
            bootstrap_servers="kafka-xxx:9092",
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
        )
        await self.producer.start()
    
    async def stop(self):
        if self.producer:
            await self.producer.stop()
    
    async def publish_task(
        self,
        user_id: str,
        character_id: str,
        messages: List[Message],
        trigger: str = "session_end"
    ) -> str:
        """
        发布记忆生成任务
        
        Args:
            user_id: 用户ID
            character_id: 角色ID
            messages: 消息列表
            trigger: 触发类型 (session_end/threshold/scheduled)
            
        Returns:
            任务ID
        """
        task_id = str(uuid.uuid4())
        session_id = str(uuid.uuid4())
        
        task_data = {
            "task_id": task_id,
            "user_id": user_id,
            "character_id": character_id,
            "session_id": session_id,
            "messages": [m.dict() for m in messages],
            "trigger": trigger,
            "created_at": datetime.now().isoformat(),
        }
        
        await self.producer.send_and_wait(
            self.topic,
            value=task_data,
            key=f"{user_id}:{character_id}".encode('utf-8'),
        )
        
        logger.info(f"Published memory task: {task_id}")
        return task_id


# 全局实例
memory_task_publisher = MemoryTaskPublisher()
```

---

## 5. 记忆召回服务

### 5.1 意图识别

```python
# services/recall_intent_service.py

import logging
import re
from typing import Optional, List
from dataclasses import dataclass
from enum import Enum

logger = logging.getLogger(__name__)


class RecallType(str, Enum):
    """召回类型"""
    NONE = "none"           # 不需要召回
    PROFILE = "profile"     # 召回用户画像
    EVENT = "event"         # 召回事件
    HISTORY = "history"     # 召回历史对话
    MIXED = "mixed"         # 混合召回


@dataclass
class RecallIntent:
    """召回意图"""
    need_recall: bool
    recall_type: RecallType
    search_query: Optional[str] = None
    confidence: float = 1.0


class RecallIntentDetector:
    """
    召回意图检测器
    
    使用规则 + 关键词匹配快速判断
    复杂场景可降级到 LLM 判断
    """
    
    # 需要召回历史的关键词
    HISTORY_PATTERNS = [
        r"上次|之前|以前|那次|那天",
        r"还记得|你记得|记不记得",
        r"我们聊过|说过|提过|讲过",
        r"你忘了|忘记了",
    ]
    
    # 需要召回画像的关键词
    PROFILE_PATTERNS = [
        r"我的名字|我叫什么|叫我什么",
        r"我的生日|我几岁|我多大",
        r"我喜欢|我讨厌|我爱吃|我不吃",
        r"我是做什么|我的工作|我在哪",
    ]
    
    # 需要召回事件的关键词
    EVENT_PATTERNS = [
        r"那件事|这件事|那个事",
        r"发生了什么|怎么了",
        r"生日|纪念日|重要的日子",
    ]
    
    def __init__(self):
        self.history_regex = re.compile("|".join(self.HISTORY_PATTERNS))
        self.profile_regex = re.compile("|".join(self.PROFILE_PATTERNS))
        self.event_regex = re.compile("|".join(self.EVENT_PATTERNS))
    
    async def detect(
        self,
        user_message: str,
        recent_context: Optional[str] = None
    ) -> RecallIntent:
        """
        检测是否需要召回长期记忆
        
        Args:
            user_message: 用户消息
            recent_context: 最近对话上下文（可选）
            
        Returns:
            召回意图
        """
        # 规则匹配
        has_history = bool(self.history_regex.search(user_message))
        has_profile = bool(self.profile_regex.search(user_message))
        has_event = bool(self.event_regex.search(user_message))
        
        # 判断召回类型
        recall_types = []
        if has_history:
            recall_types.append(RecallType.HISTORY)
        if has_profile:
            recall_types.append(RecallType.PROFILE)
        if has_event:
            recall_types.append(RecallType.EVENT)
        
        if not recall_types:
            return RecallIntent(
                need_recall=False,
                recall_type=RecallType.NONE,
                confidence=0.9
            )
        
        # 提取搜索关键词
        search_query = self._extract_search_query(user_message)
        
        # 确定最终召回类型
        if len(recall_types) > 1:
            recall_type = RecallType.MIXED
        else:
            recall_type = recall_types[0]
        
        return RecallIntent(
            need_recall=True,
            recall_type=recall_type,
            search_query=search_query,
            confidence=0.8
        )
    
    def _extract_search_query(self, message: str) -> str:
        """提取搜索关键词"""
        # 移除常见的无意义词
        stop_words = ["你", "我", "的", "了", "吗", "呢", "啊", "吧", "是", "不"]
        words = list(message)
        query = "".join([w for w in words if w not in stop_words])
        return query[:50]  # 限制长度


# 全局实例
recall_intent_detector = RecallIntentDetector()
```

### 5.2 记忆召回服务

```python
# services/memory_recall_service.py

import logging
import asyncio
from typing import List, Optional
from datetime import datetime

from models.long_memory import (
    RecalledMemory, RecallResult, MemoryType,
    UserProfile, ProfileType
)
from services.recall_intent_service import RecallIntent, RecallType
from services.embedding_service import embedding_service
from clients.vikingdb_client import vikingdb_client
from repositories.long_memory_repo import profile_repo, event_repo, summary_repo

logger = logging.getLogger(__name__)


class MemoryRecallConfig:
    """召回配置"""
    # 超时配置
    TOTAL_TIMEOUT: float = 2.0      # 总超时
    VIKINGDB_TIMEOUT: float = 1.0   # VikingDB 超时
    MYSQL_TIMEOUT: float = 1.0      # MySQL 超时
    
    # 召回数量
    VECTOR_TOP_K: int = 5           # 向量检索数量
    PROFILE_LIMIT: int = 20         # 画像数量
    EVENT_LIMIT: int = 10           # 事件数量
    
    # 结果限制
    MAX_MEMORIES: int = 5           # 最终返回记忆数
    MAX_TOKENS: int = 1000          # 最大Token数


class MemoryRecallService:
    """记忆召回服务"""
    
    def __init__(self, config: MemoryRecallConfig = None):
        self.config = config or MemoryRecallConfig()
    
    async def recall(
        self,
        user_id: str,
        character_id: str,
        intent: RecallIntent
    ) -> RecallResult:
        """
        召回长期记忆
        
        Args:
            user_id: 用户ID
            character_id: 角色ID
            intent: 召回意图
            
        Returns:
            召回结果
        """
        if not intent.need_recall:
            return RecallResult(memories=[], profiles=[])
        
        try:
            return await asyncio.wait_for(
                self._recall_impl(user_id, character_id, intent),
                timeout=self.config.TOTAL_TIMEOUT
            )
        except asyncio.TimeoutError:
            logger.warning("Memory recall timeout, returning empty result")
            return RecallResult(memories=[], profiles=[])
        except Exception as e:
            logger.error(f"Memory recall error: {e}")
            return RecallResult(memories=[], profiles=[])
    
    async def _recall_impl(
        self,
        user_id: str,
        character_id: str,
        intent: RecallIntent
    ) -> RecallResult:
        """召回实现"""
        tasks = []
        
        # 根据意图类型决定召回内容
        if intent.recall_type in [RecallType.HISTORY, RecallType.MIXED]:
            tasks.append(self._search_vector_memories(
                user_id, character_id, intent.search_query, ["summary"]
            ))
        
        if intent.recall_type in [RecallType.EVENT, RecallType.MIXED]:
            tasks.append(self._search_vector_memories(
                user_id, character_id, intent.search_query, ["event"]
            ))
            tasks.append(self._get_mysql_events(user_id, character_id))
        
        if intent.recall_type in [RecallType.PROFILE, RecallType.MIXED]:
            tasks.append(self._get_mysql_profiles(user_id, character_id))
        
        # 并行执行
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        # 合并结果
        memories = []
        profiles = []
        
        for result in results:
            if isinstance(result, Exception):
                logger.warning(f"Recall task failed: {result}")
                continue
            
            if isinstance(result, list):
                if result and isinstance(result[0], RecalledMemory):
                    memories.extend(result)
                elif result and isinstance(result[0], UserProfile):
                    profiles.extend(result)
        
        # 排序和截断
        memories = self._rank_and_truncate(memories)
        
        return RecallResult(
            memories=memories,
            profiles=profiles,
        )
    
    async def _search_vector_memories(
        self,
        user_id: str,
        character_id: str,
        query: str,
        memory_types: List[str]
    ) -> List[RecalledMemory]:
        """向量检索记忆"""
        # 生成查询向量
        vectors = await embedding_service.embed_texts([query])
        if not vectors:
            return []
        
        # VikingDB 检索
        results = await vikingdb_client.search(
            vector=vectors[0],
            user_id=user_id,
            character_id=character_id,
            memory_types=memory_types,
            top_k=self.config.VECTOR_TOP_K,
        )
        
        # 转换结果
        memories = []
        for r in results:
            memories.append(RecalledMemory(
                memory_type=MemoryType(r["memory_type"]),
                content=r["content"],
                timestamp=r["timestamp"],
                importance=r["importance"],
                relevance_score=r["score"],
                source="vikingdb",
            ))
        
        return memories
    
    async def _get_mysql_profiles(
        self,
        user_id: str,
        character_id: str
    ) -> List[UserProfile]:
        """获取用户画像"""
        return await profile_repo.get_by_user_character(
            user_id=user_id,
            character_id=character_id,
        )
    
    async def _get_mysql_events(
        self,
        user_id: str,
        character_id: str
    ) -> List[RecalledMemory]:
        """获取事件（转换为记忆格式）"""
        events = await event_repo.get_by_user_character(
            user_id=user_id,
            character_id=character_id,
            min_importance=5,
            limit=self.config.EVENT_LIMIT,
        )
        
        memories = []
        for e in events:
            memories.append(RecalledMemory(
                memory_type=MemoryType.EVENT,
                content=f"{e.event_title}: {e.event_summary}",
                timestamp=int(e.created_at.timestamp()) if e.created_at else 0,
                importance=e.importance,
                relevance_score=0.5,  # MySQL结果给默认分数
                source="mysql",
            ))
        
        return memories
    
    def _rank_and_truncate(
        self,
        memories: List[RecalledMemory]
    ) -> List[RecalledMemory]:
        """排序和截断"""
        if not memories:
            return []
        
        # 计算综合得分
        now = datetime.now().timestamp()
        for m in memories:
            # 时间衰减 (30天半衰期)
            days_ago = (now - m.timestamp) / 86400
            recency_score = 0.5 ** (days_ago / 30)
            
            # 综合得分 = 相关性 * 0.5 + 时间 * 0.3 + 重要性 * 0.2
            m.relevance_score = (
                m.relevance_score * 0.5 +
                recency_score * 0.3 +
                (m.importance / 10) * 0.2
            )
        
        # 排序
        memories.sort(key=lambda x: x.relevance_score, reverse=True)
        
        # 去重 (相似内容)
        unique_memories = []
        seen_contents = set()
        for m in memories:
            content_key = m.content[:50]
            if content_key not in seen_contents:
                seen_contents.add(content_key)
                unique_memories.append(m)
        
        # 截断
        return unique_memories[:self.config.MAX_MEMORIES]
    
    def format_for_context(
        self,
        recall_result: RecallResult
    ) -> str:
        """
        格式化召回结果为上下文注入格式
        
        Args:
            recall_result: 召回结果
            
        Returns:
            格式化的字符串
        """
        lines = []
        
        # 用户画像
        if recall_result.profiles:
            lines.append("### 关于用户的信息")
            profile_dict = {}
            for p in recall_result.profiles:
                if p.profile_type not in profile_dict:
                    profile_dict[p.profile_type] = []
                profile_dict[p.profile_type].append(f"- {p.profile_key}: {p.profile_value}")
            
            for ptype, items in profile_dict.items():
                lines.extend(items)
            lines.append("")
        
        # 相关记忆
        if recall_result.memories:
            lines.append("### 相关的历史记忆")
            for m in recall_result.memories:
                lines.append(m.to_context_string())
            lines.append("")
        
        return "\n".join(lines)


# 全局实例
memory_recall_service = MemoryRecallService()
```

---

## 6. 降级策略

### 6.1 降级策略矩阵

| 组件 | 故障场景 | 降级策略 | 影响 |
|------|----------|----------|------|
| **VikingDB** | 超时/不可用 | 跳过向量检索，只用MySQL | 语义召回失效 |
| **MySQL** | 超时/不可用 | 跳过结构化查询 | 无画像和事件 |
| **Embedding** | 超时/不可用 | 跳过向量检索 | 语义召回失效 |
| **全部长期记忆** | 全部故障 | 只使用短期记忆 | 无长期记忆 |
| **Kafka** | 不可用 | 本地队列暂存，重试 | 记忆生成延迟 |
| **TOS** | 不可用 | 跳过归档，不影响主流程 | 无归档 |

### 6.2 降级代码

```python
# services/resilient_memory_service.py

import asyncio
import logging
from typing import Optional

from models.long_memory import RecallResult
from services.recall_intent_service import RecallIntent, recall_intent_detector
from services.memory_recall_service import memory_recall_service

logger = logging.getLogger(__name__)


class ResilientMemoryService:
    """
    高可用记忆服务
    
    封装长期记忆的所有操作，提供统一的降级处理
    """
    
    async def check_and_recall(
        self,
        user_id: str,
        character_id: str,
        user_message: str,
        recent_context: Optional[str] = None
    ) -> RecallResult:
        """
        检查是否需要召回，并执行召回
        
        带完整的降级处理：
        1. 意图检测失败 → 不召回
        2. 召回超时 → 返回空结果
        3. 部分组件失败 → 返回可用部分
        """
        try:
            # 1. 意图检测
            intent = await self._safe_detect_intent(user_message, recent_context)
            
            if not intent.need_recall:
                return RecallResult(memories=[], profiles=[])
            
            # 2. 执行召回
            return await self._safe_recall(user_id, character_id, intent)
            
        except Exception as e:
            logger.error(f"Memory recall failed completely: {e}")
            return RecallResult(memories=[], profiles=[])
    
    async def _safe_detect_intent(
        self,
        user_message: str,
        recent_context: Optional[str]
    ) -> RecallIntent:
        """安全的意图检测"""
        try:
            return await asyncio.wait_for(
                recall_intent_detector.detect(user_message, recent_context),
                timeout=0.5  # 意图检测要快
            )
        except Exception as e:
            logger.warning(f"Intent detection failed: {e}, skipping recall")
            return RecallIntent(need_recall=False, recall_type="none")
    
    async def _safe_recall(
        self,
        user_id: str,
        character_id: str,
        intent: RecallIntent
    ) -> RecallResult:
        """安全的记忆召回"""
        try:
            return await memory_recall_service.recall(
                user_id=user_id,
                character_id=character_id,
                intent=intent,
            )
        except Exception as e:
            logger.warning(f"Memory recall failed: {e}")
            return RecallResult(memories=[], profiles=[])


# 全局实例
resilient_memory_service = ResilientMemoryService()
```

---

## 7. 监控指标

```python
# metrics/long_memory_metrics.py

from prometheus_client import Counter, Histogram, Gauge

# 记忆生成指标
MEMORY_GENERATION_TASKS = Counter(
    'memory_generation_tasks_total',
    'Total memory generation tasks',
    ['status']  # success/failure
)

MEMORY_GENERATION_LATENCY = Histogram(
    'memory_generation_latency_seconds',
    'Memory generation latency',
    buckets=[1, 5, 10, 30, 60, 120]
)

# 记忆召回指标
MEMORY_RECALL_REQUESTS = Counter(
    'memory_recall_requests_total',
    'Total memory recall requests',
    ['intent_type', 'status']
)

MEMORY_RECALL_LATENCY = Histogram(
    'memory_recall_latency_seconds',
    'Memory recall latency',
    buckets=[0.1, 0.25, 0.5, 1, 2, 5]
)

MEMORY_RECALL_COUNT = Histogram(
    'memory_recall_count',
    'Number of memories recalled',
    buckets=[0, 1, 2, 3, 5, 10]
)

# VikingDB 指标
VIKINGDB_OPERATIONS = Counter(
    'vikingdb_operations_total',
    'VikingDB operations',
    ['operation', 'status']
)

VIKINGDB_LATENCY = Histogram(
    'vikingdb_latency_seconds',
    'VikingDB operation latency',
    ['operation'],
    buckets=[0.05, 0.1, 0.25, 0.5, 1, 2]
)
```

---

## 8. 总结

### 长期记忆模块核心组件

| 组件 | 技术 | 职责 |
|------|------|------|
| 向量存储 | VikingDB | 语义相似度检索 |
| 结构化存储 | MySQL | 画像、事件、索引 |
| 归档存储 | TOS | 完整对话归档 |
| 异步处理 | Kafka + Worker | 记忆生成 |
| 向量化 | 方舟 Embedding | 文本向量化 |

### 数据流

```
对话完成 → Kafka → Worker → LLM分析 → 向量化 → VikingDB/MySQL/TOS
                                              ↓
用户对话 → 意图检测 → 并行检索 → 结果融合 → 注入上下文
```

### 降级保障

- 任何组件故障都不会阻塞对话
- 最差情况：只使用短期记忆继续对话
