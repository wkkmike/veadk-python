# AI陪聊系统 - 短期记忆实现方案（方案A：纯Redis）

## 1. 方案概述

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              方案A：纯Redis 架构                                     │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                            对话服务 (Chat Service)                           │    │
│  │                                                                              │    │
│  │  ┌─────────────────┐                                                        │    │
│  │  │  ShortMemory    │                                                        │    │
│  │  │  Manager        │                                                        │    │
│  │  │                 │                                                        │    │
│  │  │  • get_messages │                                                        │    │
│  │  │  • add_message  │                                                        │    │
│  │  │  • clear        │                                                        │    │
│  │  └────────┬────────┘                                                        │    │
│  └───────────┼──────────────────────────────────────────────────────────────────┘    │
│              │                                                                       │
│              │  Redis Protocol (RESP)                                               │
│              │  连接池复用                                                           │
│              ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         火山云 Redis (主从+哨兵)                              │    │
│  │                                                                              │    │
│  │  ┌─────────────────────────────────────────────────────────────────────┐    │    │
│  │  │  数据模型                                                            │    │    │
│  │  │                                                                      │    │    │
│  │  │  Key: chat:{user_id}:{character_id}:messages                        │    │    │
│  │  │  Type: List                                                         │    │    │
│  │  │  Value: [msg_json_1, msg_json_2, ...]  (最新消息在头部)               │    │    │
│  │  │  TTL: 86400s (24小时)                                               │    │    │
│  │  │  Max Length: 100 条                                                 │    │    │
│  │  └─────────────────────────────────────────────────────────────────────┘    │    │
│  │                                                                              │    │
│  │  ┌─────────────────────────────────────────────────────────────────────┐    │    │
│  │  │  高可用配置                                                          │    │    │
│  │  │                                                                      │    │    │
│  │  │  Master ◄──同步──► Slave                                            │    │    │
│  │  │     │                 │                                              │    │    │
│  │  │     └────── Sentinel ─┘  (自动故障切换)                              │    │    │
│  │  └─────────────────────────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## 2. Redis 数据结构设计

### 2.1 消息存储结构

```python
# Key 命名规范
SHORT_MEMORY_KEY_PATTERN = "chat:{user_id}:{character_id}:messages"

# 示例 Key
# chat:user_12345:char_67890:messages

# 消息结构 (JSON序列化存储)
message_schema = {
    "msg_id": "uuid-string",           # 消息唯一ID
    "role": "user|assistant",          # 角色：用户或AI
    "content": "消息内容",              # 消息文本
    "timestamp": 1699999999,           # Unix时间戳（秒）
    "token_count": 150,                # Token数量（用于上下文裁剪）
    "metadata": {                      # 可选元数据
        "emotion": "happy",            # 情感标签
        "intent": "greeting"           # 意图标签
    }
}
```

### 2.2 会话元数据结构

```python
# 会话元数据 Key
SESSION_META_KEY_PATTERN = "chat:{user_id}:{character_id}:meta"

# 元数据结构 (Hash类型)
session_meta_schema = {
    "state": "ACTIVE|INACTIVE|ENDED",  # 会话状态
    "last_active_at": "1699999999",    # 最后活跃时间
    "message_count": "25",             # 消息计数
    "session_start": "1699998888",     # 会话开始时间
    "total_tokens": "5000"             # 累计Token数
}
```

### 2.3 Redis 操作命令

```bash
# 添加新消息（头部插入）
LPUSH chat:user_123:char_456:messages '{"msg_id":"...","role":"user","content":"你好",...}'

# 保留最近100条消息
LTRIM chat:user_123:char_456:messages 0 99

# 设置24小时过期
EXPIRE chat:user_123:char_456:messages 86400

# 获取最近N条消息
LRANGE chat:user_123:char_456:messages 0 49  # 获取最近50条

# 获取所有消息
LRANGE chat:user_123:char_456:messages 0 -1

# 更新会话元数据
HSET chat:user_123:char_456:meta state ACTIVE last_active_at 1699999999

# 批量操作（Pipeline）
MULTI
LPUSH chat:user_123:char_456:messages '{"msg_id":"msg1",...}'
LPUSH chat:user_123:char_456:messages '{"msg_id":"msg2",...}'
LTRIM chat:user_123:char_456:messages 0 99
EXPIRE chat:user_123:char_456:messages 86400
HSET chat:user_123:char_456:meta last_active_at 1699999999 message_count 26
EXEC
```

## 3. Python 实现代码

### 3.1 数据模型定义

```python
# models/message.py

from datetime import datetime
from typing import Optional, Dict, Any
from pydantic import BaseModel, Field
from enum import Enum
import uuid


class MessageRole(str, Enum):
    USER = "user"
    ASSISTANT = "assistant"
    SYSTEM = "system"


class Message(BaseModel):
    """对话消息模型"""
    msg_id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    role: MessageRole
    content: str
    timestamp: int = Field(default_factory=lambda: int(datetime.now().timestamp()))
    token_count: int = 0
    metadata: Optional[Dict[str, Any]] = None
    
    class Config:
        use_enum_values = True
    
    def to_llm_format(self) -> Dict[str, str]:
        """转换为LLM API格式"""
        return {
            "role": self.role,
            "content": self.content
        }
    
    @classmethod
    def from_redis(cls, data: str) -> "Message":
        """从Redis JSON字符串解析"""
        import json
        return cls.parse_obj(json.loads(data))
    
    def to_redis(self) -> str:
        """序列化为Redis存储格式"""
        return self.json()


class SessionState(str, Enum):
    ACTIVE = "ACTIVE"
    INACTIVE = "INACTIVE"
    ENDED = "ENDED"


class SessionMeta(BaseModel):
    """会话元数据"""
    state: SessionState = SessionState.ACTIVE
    last_active_at: int = Field(default_factory=lambda: int(datetime.now().timestamp()))
    message_count: int = 0
    session_start: int = Field(default_factory=lambda: int(datetime.now().timestamp()))
    total_tokens: int = 0
```

### 3.2 Redis 客户端配置

```python
# clients/redis_client.py

import redis.asyncio as redis
from redis.asyncio.sentinel import Sentinel
from typing import Optional
import logging

logger = logging.getLogger(__name__)


class RedisConfig:
    """Redis配置"""
    # 火山云 Redis 连接信息
    HOST: str = "redis-xxx.ivolces.com"
    PORT: int = 6379
    PASSWORD: str = "your-password"
    DB: int = 0
    
    # 连接池配置
    MAX_CONNECTIONS: int = 20
    SOCKET_TIMEOUT: float = 1.0
    SOCKET_CONNECT_TIMEOUT: float = 1.0
    RETRY_ON_TIMEOUT: bool = True
    
    # 哨兵配置（生产环境使用）
    SENTINEL_HOSTS: list = [
        ("sentinel-1.xxx.com", 26379),
        ("sentinel-2.xxx.com", 26379),
        ("sentinel-3.xxx.com", 26379),
    ]
    SENTINEL_MASTER_NAME: str = "mymaster"
    
    # 是否使用哨兵模式
    USE_SENTINEL: bool = True


class RedisClient:
    """Redis客户端管理器"""
    
    _instance: Optional["RedisClient"] = None
    _redis: Optional[redis.Redis] = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
    
    async def get_client(self) -> redis.Redis:
        """获取Redis客户端（单例+连接池）"""
        if self._redis is None:
            self._redis = await self._create_client()
        return self._redis
    
    async def _create_client(self) -> redis.Redis:
        """创建Redis客户端"""
        if RedisConfig.USE_SENTINEL:
            return await self._create_sentinel_client()
        else:
            return await self._create_standalone_client()
    
    async def _create_standalone_client(self) -> redis.Redis:
        """创建单机客户端"""
        pool = redis.ConnectionPool(
            host=RedisConfig.HOST,
            port=RedisConfig.PORT,
            password=RedisConfig.PASSWORD,
            db=RedisConfig.DB,
            max_connections=RedisConfig.MAX_CONNECTIONS,
            socket_timeout=RedisConfig.SOCKET_TIMEOUT,
            socket_connect_timeout=RedisConfig.SOCKET_CONNECT_TIMEOUT,
            retry_on_timeout=RedisConfig.RETRY_ON_TIMEOUT,
            decode_responses=True,
        )
        return redis.Redis(connection_pool=pool)
    
    async def _create_sentinel_client(self) -> redis.Redis:
        """创建哨兵客户端"""
        sentinel = Sentinel(
            RedisConfig.SENTINEL_HOSTS,
            socket_timeout=RedisConfig.SOCKET_TIMEOUT,
            password=RedisConfig.PASSWORD,
        )
        return sentinel.master_for(
            RedisConfig.SENTINEL_MASTER_NAME,
            socket_timeout=RedisConfig.SOCKET_TIMEOUT,
            password=RedisConfig.PASSWORD,
            db=RedisConfig.DB,
            decode_responses=True,
        )
    
    async def close(self):
        """关闭连接"""
        if self._redis:
            await self._redis.close()
            self._redis = None


# 全局客户端实例
redis_client = RedisClient()


async def get_redis() -> redis.Redis:
    """获取Redis客户端的便捷函数"""
    return await redis_client.get_client()
```

### 3.3 短期记忆管理器

```python
# memory/short_memory.py

import json
import logging
from typing import List, Optional, Tuple
from datetime import datetime

from models.message import Message, MessageRole, SessionMeta, SessionState
from clients.redis_client import get_redis

logger = logging.getLogger(__name__)


class ShortMemoryConfig:
    """短期记忆配置"""
    # Key 模式
    MESSAGE_KEY_PATTERN = "chat:{user_id}:{character_id}:messages"
    META_KEY_PATTERN = "chat:{user_id}:{character_id}:meta"
    
    # 存储限制
    MAX_MESSAGES = 100          # 最多保存100条消息
    TTL_SECONDS = 86400         # 24小时过期
    
    # 上下文限制
    MAX_CONTEXT_TOKENS = 20000  # 上下文最大Token数
    MAX_CONTEXT_MESSAGES = 50   # 上下文最大消息数
    
    # 超时配置
    READ_TIMEOUT = 1.0          # 读取超时（秒）
    WRITE_TIMEOUT = 1.0         # 写入超时（秒）


class ShortMemoryManager:
    """短期记忆管理器"""
    
    def __init__(self, config: ShortMemoryConfig = None):
        self.config = config or ShortMemoryConfig()
    
    def _get_message_key(self, user_id: str, character_id: str) -> str:
        """生成消息存储Key"""
        return self.config.MESSAGE_KEY_PATTERN.format(
            user_id=user_id,
            character_id=character_id
        )
    
    def _get_meta_key(self, user_id: str, character_id: str) -> str:
        """生成元数据Key"""
        return self.config.META_KEY_PATTERN.format(
            user_id=user_id,
            character_id=character_id
        )
    
    async def get_messages(
        self,
        user_id: str,
        character_id: str,
        limit: Optional[int] = None,
        max_tokens: Optional[int] = None
    ) -> List[Message]:
        """
        获取短期记忆消息
        
        Args:
            user_id: 用户ID
            character_id: 角色ID
            limit: 最大消息数量
            max_tokens: 最大Token数量
            
        Returns:
            消息列表（按时间正序，最早的在前）
        """
        limit = limit or self.config.MAX_CONTEXT_MESSAGES
        max_tokens = max_tokens or self.config.MAX_CONTEXT_TOKENS
        
        key = self._get_message_key(user_id, character_id)
        
        try:
            redis = await get_redis()
            
            # 获取消息（Redis中最新的在前，需要反转）
            raw_messages = await redis.lrange(key, 0, limit - 1)
            
            if not raw_messages:
                return []
            
            # 解析消息
            messages = []
            total_tokens = 0
            
            for raw in raw_messages:
                try:
                    msg = Message.from_redis(raw)
                    
                    # Token 限制检查
                    if total_tokens + msg.token_count > max_tokens:
                        break
                    
                    total_tokens += msg.token_count
                    messages.append(msg)
                except Exception as e:
                    logger.warning(f"Failed to parse message: {e}")
                    continue
            
            # 反转为时间正序（最早的在前）
            messages.reverse()
            
            logger.debug(
                f"Retrieved {len(messages)} messages for user={user_id}, "
                f"char={character_id}, total_tokens={total_tokens}"
            )
            
            return messages
            
        except Exception as e:
            logger.error(f"Failed to get short memory: {e}")
            # 降级：返回空列表，继续对话但无历史
            return []
    
    async def add_message(
        self,
        user_id: str,
        character_id: str,
        message: Message
    ) -> bool:
        """
        添加新消息到短期记忆
        
        Args:
            user_id: 用户ID
            character_id: 角色ID
            message: 消息对象
            
        Returns:
            是否成功
        """
        message_key = self._get_message_key(user_id, character_id)
        meta_key = self._get_meta_key(user_id, character_id)
        
        try:
            redis = await get_redis()
            
            # 使用 Pipeline 批量操作
            async with redis.pipeline(transaction=True) as pipe:
                # 1. 添加消息到列表头部
                pipe.lpush(message_key, message.to_redis())
                
                # 2. 裁剪列表，保留最近N条
                pipe.ltrim(message_key, 0, self.config.MAX_MESSAGES - 1)
                
                # 3. 设置/刷新过期时间
                pipe.expire(message_key, self.config.TTL_SECONDS)
                
                # 4. 更新会话元数据
                now = int(datetime.now().timestamp())
                pipe.hset(meta_key, mapping={
                    "state": SessionState.ACTIVE.value,
                    "last_active_at": str(now),
                })
                pipe.hincrby(meta_key, "message_count", 1)
                pipe.hincrby(meta_key, "total_tokens", message.token_count)
                pipe.expire(meta_key, self.config.TTL_SECONDS)
                
                # 执行
                await pipe.execute()
            
            logger.debug(f"Added message {message.msg_id} for user={user_id}")
            return True
            
        except Exception as e:
            logger.error(f"Failed to add message: {e}")
            # 写入失败不影响主流程，消息会丢失但对话可继续
            return False
    
    async def add_messages_batch(
        self,
        user_id: str,
        character_id: str,
        messages: List[Message]
    ) -> bool:
        """
        批量添加消息（用于同时保存用户消息和AI回复）
        
        Args:
            user_id: 用户ID
            character_id: 角色ID
            messages: 消息列表
            
        Returns:
            是否成功
        """
        if not messages:
            return True
        
        message_key = self._get_message_key(user_id, character_id)
        meta_key = self._get_meta_key(user_id, character_id)
        
        try:
            redis = await get_redis()
            
            total_tokens = sum(m.token_count for m in messages)
            
            async with redis.pipeline(transaction=True) as pipe:
                # 批量添加消息（注意顺序：最新的最后添加，lpush后会在最前）
                for msg in reversed(messages):
                    pipe.lpush(message_key, msg.to_redis())
                
                pipe.ltrim(message_key, 0, self.config.MAX_MESSAGES - 1)
                pipe.expire(message_key, self.config.TTL_SECONDS)
                
                now = int(datetime.now().timestamp())
                pipe.hset(meta_key, mapping={
                    "state": SessionState.ACTIVE.value,
                    "last_active_at": str(now),
                })
                pipe.hincrby(meta_key, "message_count", len(messages))
                pipe.hincrby(meta_key, "total_tokens", total_tokens)
                pipe.expire(meta_key, self.config.TTL_SECONDS)
                
                await pipe.execute()
            
            logger.debug(f"Added {len(messages)} messages for user={user_id}")
            return True
            
        except Exception as e:
            logger.error(f"Failed to add messages batch: {e}")
            return False
    
    async def get_session_meta(
        self,
        user_id: str,
        character_id: str
    ) -> Optional[SessionMeta]:
        """获取会话元数据"""
        key = self._get_meta_key(user_id, character_id)
        
        try:
            redis = await get_redis()
            data = await redis.hgetall(key)
            
            if not data:
                return None
            
            return SessionMeta(
                state=SessionState(data.get("state", "ACTIVE")),
                last_active_at=int(data.get("last_active_at", 0)),
                message_count=int(data.get("message_count", 0)),
                session_start=int(data.get("session_start", 0)),
                total_tokens=int(data.get("total_tokens", 0)),
            )
            
        except Exception as e:
            logger.error(f"Failed to get session meta: {e}")
            return None
    
    async def update_session_state(
        self,
        user_id: str,
        character_id: str,
        state: SessionState
    ) -> bool:
        """更新会话状态"""
        key = self._get_meta_key(user_id, character_id)
        
        try:
            redis = await get_redis()
            await redis.hset(key, "state", state.value)
            return True
        except Exception as e:
            logger.error(f"Failed to update session state: {e}")
            return False
    
    async def clear_messages(
        self,
        user_id: str,
        character_id: str
    ) -> bool:
        """清空会话消息（用于开始新会话）"""
        message_key = self._get_message_key(user_id, character_id)
        meta_key = self._get_meta_key(user_id, character_id)
        
        try:
            redis = await get_redis()
            await redis.delete(message_key, meta_key)
            return True
        except Exception as e:
            logger.error(f"Failed to clear messages: {e}")
            return False
    
    async def get_message_count(
        self,
        user_id: str,
        character_id: str
    ) -> int:
        """获取消息数量"""
        key = self._get_message_key(user_id, character_id)
        
        try:
            redis = await get_redis()
            return await redis.llen(key)
        except Exception as e:
            logger.error(f"Failed to get message count: {e}")
            return 0


# 全局实例
short_memory_manager = ShortMemoryManager()
```

### 3.4 带降级的短期记忆服务

```python
# services/memory_service.py

import asyncio
import logging
from typing import List, Optional
from functools import wraps

from models.message import Message, MessageRole
from memory.short_memory import short_memory_manager, ShortMemoryManager

logger = logging.getLogger(__name__)


class MemoryServiceConfig:
    """记忆服务配置"""
    # 超时配置
    GET_TIMEOUT = 1.0           # 获取记忆超时
    SAVE_TIMEOUT = 1.0          # 保存记忆超时
    
    # 降级配置
    ENABLE_DEGRADATION = True   # 是否启用降级
    MAX_RETRIES = 2             # 最大重试次数
    RETRY_DELAY = 0.1           # 重试延迟（秒）


def with_timeout_and_fallback(timeout: float, fallback_value):
    """超时和降级装饰器"""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            try:
                return await asyncio.wait_for(
                    func(*args, **kwargs),
                    timeout=timeout
                )
            except asyncio.TimeoutError:
                logger.warning(f"{func.__name__} timeout, using fallback")
                return fallback_value
            except Exception as e:
                logger.error(f"{func.__name__} error: {e}, using fallback")
                return fallback_value
        return wrapper
    return decorator


class MemoryService:
    """
    记忆服务（封装短期记忆和长期记忆）
    
    提供统一的记忆访问接口，包含：
    - 超时控制
    - 自动降级
    - 重试机制
    """
    
    def __init__(
        self,
        short_memory: ShortMemoryManager = None,
        config: MemoryServiceConfig = None
    ):
        self.short_memory = short_memory or short_memory_manager
        self.config = config or MemoryServiceConfig()
    
    async def get_context_messages(
        self,
        user_id: str,
        character_id: str,
        max_messages: int = 50,
        max_tokens: int = 20000
    ) -> List[Message]:
        """
        获取对话上下文消息
        
        带超时和降级：
        - 超时1秒自动返回空列表
        - Redis故障返回空列表
        - 对话可继续，但无历史上下文
        
        Args:
            user_id: 用户ID
            character_id: 角色ID
            max_messages: 最大消息数
            max_tokens: 最大Token数
            
        Returns:
            消息列表（时间正序）
        """
        try:
            messages = await asyncio.wait_for(
                self.short_memory.get_messages(
                    user_id=user_id,
                    character_id=character_id,
                    limit=max_messages,
                    max_tokens=max_tokens
                ),
                timeout=self.config.GET_TIMEOUT
            )
            return messages
            
        except asyncio.TimeoutError:
            logger.warning(
                f"Get context timeout for user={user_id}, "
                f"degrading to empty context"
            )
            return []
            
        except Exception as e:
            logger.error(
                f"Get context failed for user={user_id}: {e}, "
                f"degrading to empty context"
            )
            return []
    
    async def save_turn(
        self,
        user_id: str,
        character_id: str,
        user_message: Message,
        assistant_message: Message
    ) -> bool:
        """
        保存一轮对话（用户消息 + AI回复）
        
        异步执行，不阻塞响应。失败时仅记录日志。
        
        Args:
            user_id: 用户ID
            character_id: 角色ID
            user_message: 用户消息
            assistant_message: AI回复
            
        Returns:
            是否成功
        """
        try:
            return await asyncio.wait_for(
                self.short_memory.add_messages_batch(
                    user_id=user_id,
                    character_id=character_id,
                    messages=[user_message, assistant_message]
                ),
                timeout=self.config.SAVE_TIMEOUT
            )
            
        except asyncio.TimeoutError:
            logger.warning(f"Save turn timeout for user={user_id}")
            return False
            
        except Exception as e:
            logger.error(f"Save turn failed for user={user_id}: {e}")
            return False
    
    async def save_turn_async(
        self,
        user_id: str,
        character_id: str,
        user_message: Message,
        assistant_message: Message
    ):
        """
        异步保存对话（Fire and Forget）
        
        用于在流式响应完成后保存消息，不等待结果
        """
        asyncio.create_task(
            self.save_turn(
                user_id=user_id,
                character_id=character_id,
                user_message=user_message,
                assistant_message=assistant_message
            )
        )
    
    def format_messages_for_llm(
        self,
        messages: List[Message],
        system_prompt: str = None
    ) -> List[dict]:
        """
        格式化消息为LLM API格式
        
        Args:
            messages: 消息列表
            system_prompt: 系统提示词
            
        Returns:
            LLM API格式的消息列表
        """
        result = []
        
        # 添加系统提示
        if system_prompt:
            result.append({
                "role": "system",
                "content": system_prompt
            })
        
        # 添加历史消息
        for msg in messages:
            result.append(msg.to_llm_format())
        
        return result


# 全局实例
memory_service = MemoryService()
```

### 3.5 Token 计算工具

```python
# utils/token_counter.py

import tiktoken
from functools import lru_cache
from typing import List


class TokenCounter:
    """Token 计算器"""
    
    def __init__(self, model: str = "gpt-4"):
        """
        初始化Token计算器
        
        注意：doubao模型的tokenizer与GPT-4类似，
        这里使用tiktoken作为近似计算
        """
        self.model = model
        self._encoder = None
    
    @property
    def encoder(self):
        if self._encoder is None:
            try:
                self._encoder = tiktoken.encoding_for_model(self.model)
            except KeyError:
                self._encoder = tiktoken.get_encoding("cl100k_base")
        return self._encoder
    
    def count(self, text: str) -> int:
        """计算文本的Token数量"""
        if not text:
            return 0
        return len(self.encoder.encode(text))
    
    def count_messages(self, messages: List[dict]) -> int:
        """
        计算消息列表的总Token数
        
        参考OpenAI的计算方式，每条消息额外计入格式开销
        """
        total = 0
        for msg in messages:
            # 每条消息的格式开销
            total += 4  # <im_start>{role}\n{content}<im_end>\n
            
            if "role" in msg:
                total += self.count(msg["role"])
            if "content" in msg:
                total += self.count(msg["content"])
            if "name" in msg:
                total += self.count(msg["name"])
                total += 1  # 额外的格式开销
        
        total += 2  # 回复的开始标记
        return total
    
    @lru_cache(maxsize=10000)
    def count_cached(self, text: str) -> int:
        """带缓存的Token计算（适用于重复计算场景）"""
        return self.count(text)


# 全局实例
token_counter = TokenCounter()


def estimate_tokens(text: str) -> int:
    """
    快速估算Token数（不使用tokenizer）
    
    适用于不需要精确计算的场景
    平均每个中文字符约1.5个token
    平均每个英文单词约1个token
    """
    if not text:
        return 0
    
    # 简单估算：中文字符数 * 1.5 + 英文单词数
    chinese_chars = sum(1 for c in text if '\u4e00' <= c <= '\u9fff')
    other_chars = len(text) - chinese_chars
    
    return int(chinese_chars * 1.5 + other_chars * 0.3)
```

## 4. 对话服务集成

### 4.1 完整的对话处理流程

```python
# services/chat_service.py

import asyncio
import logging
from typing import AsyncIterator, List, Optional

from models.message import Message, MessageRole
from services.memory_service import memory_service
from services.llm_service import llm_service
from services.character_service import character_service
from utils.token_counter import token_counter

logger = logging.getLogger(__name__)


class ChatService:
    """对话服务"""
    
    def __init__(self):
        self.memory = memory_service
        self.llm = llm_service
        self.character = character_service
    
    async def handle_message(
        self,
        user_id: str,
        character_id: str,
        user_input: str
    ) -> AsyncIterator[str]:
        """
        处理用户消息，返回流式响应
        
        Args:
            user_id: 用户ID
            character_id: 角色ID
            user_input: 用户输入
            
        Yields:
            AI回复的文本片段
        """
        # 1. 创建用户消息对象
        user_message = Message(
            role=MessageRole.USER,
            content=user_input,
            token_count=token_counter.count(user_input)
        )
        
        # 2. 并行获取角色配置和短期记忆
        character_task = self.character.get_character(character_id)
        memory_task = self.memory.get_context_messages(user_id, character_id)
        
        character_config, history_messages = await asyncio.gather(
            character_task,
            memory_task
        )
        
        # 3. 组装上下文
        context_messages = self.memory.format_messages_for_llm(
            messages=history_messages,
            system_prompt=character_config.system_prompt
        )
        
        # 4. 添加当前用户消息
        context_messages.append(user_message.to_llm_format())
        
        # 5. 调用LLM（流式）
        response_text = ""
        async for chunk in self.llm.chat_stream(
            messages=context_messages,
            character_config=character_config
        ):
            response_text += chunk
            yield chunk
        
        # 6. 创建AI回复消息对象
        assistant_message = Message(
            role=MessageRole.ASSISTANT,
            content=response_text,
            token_count=token_counter.count(response_text)
        )
        
        # 7. 异步保存消息（不阻塞）
        self.memory.save_turn_async(
            user_id=user_id,
            character_id=character_id,
            user_message=user_message,
            assistant_message=assistant_message
        )
        
        logger.info(
            f"Chat completed: user={user_id}, char={character_id}, "
            f"input_tokens={user_message.token_count}, "
            f"output_tokens={assistant_message.token_count}"
        )


# 全局实例
chat_service = ChatService()
```

### 4.2 FastAPI 路由

```python
# api/routes/chat.py

import json
import logging
from typing import Optional

from fastapi import APIRouter, HTTPException, Request, Depends
from fastapi.responses import StreamingResponse
from pydantic import BaseModel, Field
from sse_starlette.sse import EventSourceResponse

from services.chat_service import chat_service
from api.middleware.auth import get_current_user

logger = logging.getLogger(__name__)
router = APIRouter(prefix="/api/v1/chat", tags=["chat"])


class ChatRequest(BaseModel):
    """对话请求"""
    character_id: str = Field(..., description="角色ID")
    message: str = Field(..., max_length=2000, description="用户消息")


class ChatChunk(BaseModel):
    """流式响应片段"""
    content: str
    done: bool = False


@router.post("/stream")
async def chat_stream(
    request: ChatRequest,
    user_id: str = Depends(get_current_user)
):
    """
    流式对话接口 (Server-Sent Events)
    
    响应格式:
    ```
    event: message
    data: {"content": "你好", "done": false}
    
    event: message
    data: {"content": "！", "done": false}
    
    event: message
    data: {"content": "", "done": true}
    ```
    """
    async def event_generator():
        try:
            async for chunk in chat_service.handle_message(
                user_id=user_id,
                character_id=request.character_id,
                user_input=request.message
            ):
                yield {
                    "event": "message",
                    "data": json.dumps(
                        {"content": chunk, "done": False},
                        ensure_ascii=False
                    )
                }
            
            # 发送完成标记
            yield {
                "event": "message",
                "data": json.dumps(
                    {"content": "", "done": True},
                    ensure_ascii=False
                )
            }
            
        except Exception as e:
            logger.error(f"Chat stream error: {e}")
            yield {
                "event": "error",
                "data": json.dumps(
                    {"error": "服务暂时不可用，请稍后重试"},
                    ensure_ascii=False
                )
            }
    
    return EventSourceResponse(event_generator())


@router.delete("/history")
async def clear_history(
    character_id: str,
    user_id: str = Depends(get_current_user)
):
    """清空对话历史"""
    from memory.short_memory import short_memory_manager
    
    success = await short_memory_manager.clear_messages(user_id, character_id)
    
    if not success:
        raise HTTPException(status_code=500, detail="清空历史失败")
    
    return {"message": "历史已清空"}
```

## 5. 火山云 Redis 配置

### 5.1 推荐规格

| 阶段 | 用户规模 | Redis规格 | 预估内存使用 | 月费用 |
|------|---------|----------|-------------|--------|
| 初期 | 1万 | 4GB 主从版 | ~2GB | ¥800 |
| 中期 | 10万 | 16GB 主从版 | ~10GB | ¥3,000 |
| 后期 | 100万 | 64GB 集群版 | ~50GB | ¥20,000 |

### 5.2 内存估算

```python
# 内存估算
def estimate_redis_memory(user_count: int) -> dict:
    """
    估算Redis内存使用
    
    假设：
    - 50%用户在24小时内有活跃会话
    - 平均每个会话50条消息
    - 每条消息平均500字节
    """
    active_users = user_count * 0.5
    messages_per_user = 50
    bytes_per_message = 500
    
    # 消息数据
    message_data = active_users * messages_per_user * bytes_per_message
    
    # Redis额外开销（约30%）
    overhead = message_data * 0.3
    
    # 元数据（每用户约200字节）
    metadata = active_users * 200
    
    total_bytes = message_data + overhead + metadata
    total_gb = total_bytes / (1024 ** 3)
    
    return {
        "active_users": int(active_users),
        "total_messages": int(active_users * messages_per_user),
        "estimated_gb": round(total_gb, 2),
        "recommended_gb": int(total_gb * 1.5)  # 留50%余量
    }

# 示例
print(estimate_redis_memory(10000))
# {'active_users': 5000, 'total_messages': 250000, 'estimated_gb': 1.4, 'recommended_gb': 2}

print(estimate_redis_memory(1000000))
# {'active_users': 500000, 'total_messages': 25000000, 'estimated_gb': 14.0, 'recommended_gb': 21}
```

### 5.3 火山云 Redis 创建配置

```yaml
# 火山云 Redis 配置建议
产品: 云数据库 Redis
版本: 6.0

# 初期配置
初期配置:
  架构: 主从版
  规格: 4GB
  可用区: 单可用区
  网络: VPC内网
  参数配置:
    maxmemory-policy: volatile-lru  # 优先淘汰有TTL的Key
    timeout: 300                     # 空闲连接超时
    tcp-keepalive: 60               # TCP保活

# 后期配置（扩展后）
后期配置:
  架构: 集群版
  规格: 64GB (8分片)
  可用区: 多可用区
  参数配置:
    cluster-require-full-coverage: no  # 部分节点故障时继续服务
```

## 6. 监控与告警

### 6.1 关键监控指标

```python
# metrics/short_memory_metrics.py

from prometheus_client import Counter, Histogram, Gauge

# 操作计数器
SHORT_MEMORY_OPS = Counter(
    'short_memory_operations_total',
    'Total short memory operations',
    ['operation', 'status']  # operation: get/add, status: success/failure/timeout
)

# 延迟直方图
SHORT_MEMORY_LATENCY = Histogram(
    'short_memory_latency_seconds',
    'Short memory operation latency',
    ['operation'],
    buckets=[0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0]
)

# 消息数量
SHORT_MEMORY_MESSAGE_COUNT = Gauge(
    'short_memory_message_count',
    'Number of messages in short memory',
    ['user_id', 'character_id']
)

# 降级计数
SHORT_MEMORY_DEGRADATION = Counter(
    'short_memory_degradation_total',
    'Total short memory degradation events',
    ['reason']  # timeout/error
)
```

### 6.2 监控埋点示例

```python
# 在 ShortMemoryManager 中添加监控

import time
from metrics.short_memory_metrics import (
    SHORT_MEMORY_OPS,
    SHORT_MEMORY_LATENCY,
    SHORT_MEMORY_DEGRADATION
)

async def get_messages_with_metrics(self, user_id: str, character_id: str, ...):
    start_time = time.time()
    
    try:
        result = await self._get_messages_impl(user_id, character_id, ...)
        
        # 记录成功
        SHORT_MEMORY_OPS.labels(operation='get', status='success').inc()
        SHORT_MEMORY_LATENCY.labels(operation='get').observe(
            time.time() - start_time
        )
        
        return result
        
    except asyncio.TimeoutError:
        SHORT_MEMORY_OPS.labels(operation='get', status='timeout').inc()
        SHORT_MEMORY_DEGRADATION.labels(reason='timeout').inc()
        return []
        
    except Exception as e:
        SHORT_MEMORY_OPS.labels(operation='get', status='failure').inc()
        SHORT_MEMORY_DEGRADATION.labels(reason='error').inc()
        return []
```

### 6.3 告警规则

```yaml
# 告警配置
alerts:
  - name: "短期记忆读取超时率过高"
    expr: |
      rate(short_memory_operations_total{operation="get",status="timeout"}[5m])
      / rate(short_memory_operations_total{operation="get"}[5m]) > 0.05
    for: 3m
    severity: warning
    message: "短期记忆读取超时率超过5%"
    
  - name: "短期记忆完全不可用"
    expr: |
      rate(short_memory_operations_total{operation="get",status="success"}[5m]) == 0
      and rate(short_memory_operations_total{operation="get"}[5m]) > 0
    for: 1m
    severity: critical
    message: "短期记忆服务完全不可用"
    
  - name: "短期记忆延迟过高"
    expr: |
      histogram_quantile(0.99, rate(short_memory_latency_seconds_bucket{operation="get"}[5m])) > 0.5
    for: 5m
    severity: warning
    message: "短期记忆P99延迟超过500ms"
```

## 7. 测试用例

```python
# tests/test_short_memory.py

import pytest
import asyncio
from unittest.mock import AsyncMock, patch

from models.message import Message, MessageRole
from memory.short_memory import ShortMemoryManager, ShortMemoryConfig


@pytest.fixture
def memory_manager():
    return ShortMemoryManager()


@pytest.fixture
def sample_messages():
    return [
        Message(role=MessageRole.USER, content="你好", token_count=2),
        Message(role=MessageRole.ASSISTANT, content="你好！很高兴见到你", token_count=10),
        Message(role=MessageRole.USER, content="今天天气怎么样", token_count=5),
    ]


class TestShortMemoryManager:
    """短期记忆管理器测试"""
    
    @pytest.mark.asyncio
    async def test_add_and_get_messages(self, memory_manager, sample_messages):
        """测试添加和获取消息"""
        user_id = "test_user_1"
        char_id = "test_char_1"
        
        # 添加消息
        for msg in sample_messages:
            result = await memory_manager.add_message(user_id, char_id, msg)
            assert result is True
        
        # 获取消息
        messages = await memory_manager.get_messages(user_id, char_id)
        
        assert len(messages) == 3
        assert messages[0].content == "你好"  # 最早的在前
        assert messages[-1].content == "今天天气怎么样"  # 最新的在后
    
    @pytest.mark.asyncio
    async def test_message_limit(self, memory_manager):
        """测试消息数量限制"""
        user_id = "test_user_2"
        char_id = "test_char_2"
        
        # 添加超过限制的消息
        for i in range(150):
            msg = Message(role=MessageRole.USER, content=f"消息{i}", token_count=3)
            await memory_manager.add_message(user_id, char_id, msg)
        
        # 验证只保留最近100条
        messages = await memory_manager.get_messages(user_id, char_id, limit=200)
        assert len(messages) == 100
        assert messages[-1].content == "消息149"  # 最新的
    
    @pytest.mark.asyncio
    async def test_token_limit(self, memory_manager):
        """测试Token数量限制"""
        user_id = "test_user_3"
        char_id = "test_char_3"
        
        # 添加消息
        for i in range(10):
            msg = Message(
                role=MessageRole.USER,
                content=f"消息{i}" * 100,
                token_count=1000  # 每条1000 tokens
            )
            await memory_manager.add_message(user_id, char_id, msg)
        
        # 获取时限制5000 tokens
        messages = await memory_manager.get_messages(
            user_id, char_id, max_tokens=5000
        )
        
        # 应该只返回5条（5000 tokens / 1000 tokens per msg）
        assert len(messages) == 5
    
    @pytest.mark.asyncio
    async def test_degradation_on_timeout(self, memory_manager):
        """测试超时降级"""
        user_id = "test_user_4"
        char_id = "test_char_4"
        
        # Mock Redis超时
        with patch.object(
            memory_manager, '_get_from_redis',
            side_effect=asyncio.TimeoutError()
        ):
            messages = await memory_manager.get_messages(user_id, char_id)
            
            # 降级返回空列表
            assert messages == []
    
    @pytest.mark.asyncio
    async def test_batch_add_messages(self, memory_manager, sample_messages):
        """测试批量添加消息"""
        user_id = "test_user_5"
        char_id = "test_char_5"
        
        result = await memory_manager.add_messages_batch(
            user_id, char_id, sample_messages
        )
        
        assert result is True
        
        messages = await memory_manager.get_messages(user_id, char_id)
        assert len(messages) == 3
    
    @pytest.mark.asyncio
    async def test_clear_messages(self, memory_manager, sample_messages):
        """测试清空消息"""
        user_id = "test_user_6"
        char_id = "test_char_6"
        
        # 先添加消息
        await memory_manager.add_messages_batch(user_id, char_id, sample_messages)
        
        # 清空
        result = await memory_manager.clear_messages(user_id, char_id)
        assert result is True
        
        # 验证已清空
        messages = await memory_manager.get_messages(user_id, char_id)
        assert len(messages) == 0
```

## 8. 目录结构

```
ai-companion/
├── src/
│   ├── api/
│   │   ├── __init__.py
│   │   ├── routes/
│   │   │   ├── __init__.py
│   │   │   └── chat.py              # 对话API路由
│   │   └── middleware/
│   │       ├── __init__.py
│   │       └── auth.py              # 认证中间件
│   │
│   ├── models/
│   │   ├── __init__.py
│   │   └── message.py               # 消息数据模型
│   │
│   ├── memory/
│   │   ├── __init__.py
│   │   └── short_memory.py          # 短期记忆实现
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   ├── chat_service.py          # 对话服务
│   │   ├── memory_service.py        # 记忆服务（封装层）
│   │   ├── llm_service.py           # 大模型服务
│   │   └── character_service.py     # 角色服务
│   │
│   ├── clients/
│   │   ├── __init__.py
│   │   └── redis_client.py          # Redis客户端
│   │
│   ├── utils/
│   │   ├── __init__.py
│   │   └── token_counter.py         # Token计算
│   │
│   └── metrics/
│       ├── __init__.py
│       └── short_memory_metrics.py  # 监控指标
│
├── tests/
│   ├── __init__.py
│   └── test_short_memory.py         # 单元测试
│
├── config/
│   ├── __init__.py
│   └── settings.py                  # 配置管理
│
└── requirements.txt
```

## 9. 快速开始

```bash
# 1. 安装依赖
pip install redis[hiredis] pydantic fastapi sse-starlette tiktoken

# 2. 配置Redis连接（修改 clients/redis_client.py 中的配置）

# 3. 运行测试
pytest tests/test_short_memory.py -v

# 4. 启动服务
uvicorn main:app --host 0.0.0.0 --port 8080
```

---

## 总结

方案A（纯Redis）的核心优势：

1. **简单可靠**：架构清晰，代码量少，易于维护
2. **高性能**：Redis单次操作延迟 1-5ms
3. **自动过期**：TTL机制自动清理过期数据
4. **高可用**：火山云Redis主从+哨兵，自动故障切换
5. **优雅降级**：Redis故障时自动降级，不影响对话

后续扩展路径：
- 用户量增长 → 升级Redis规格或切换集群版
- 需要更低延迟 → 升级到方案B（加本地缓存）
- 需要数据分析 → 升级到方案C（加持久化）
