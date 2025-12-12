# AI陪聊系统 - 大模型服务设计

## 1. 模块概述

### 1.1 大模型服务架构

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              大模型服务架构                                           │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                           请求入口                                           │    │
│  │                                                                              │    │
│  │   chat_stream(messages, character_config)                                   │    │
│  │                         │                                                    │    │
│  └─────────────────────────┼────────────────────────────────────────────────────┘    │
│                            │                                                         │
│                            ▼                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                        模型路由层 (Model Router)                             │    │
│  │                                                                              │    │
│  │   ┌─────────────────────────────────────────────────────────────────────┐   │    │
│  │   │                      熔断器管理 (Circuit Breaker)                    │   │    │
│  │   │                                                                      │   │    │
│  │   │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐            │   │    │
│  │   │   │ Primary CB  │    │Secondary CB │    │ Fallback CB │            │   │    │
│  │   │   │ (主模型)    │    │ (备用模型)   │    │ (兜底模型)  │            │   │    │
│  │   │   │             │    │             │    │             │            │   │    │
│  │   │   │ State:      │    │ State:      │    │ State:      │            │   │    │
│  │   │   │ CLOSED/OPEN │    │ CLOSED/OPEN │    │ CLOSED/OPEN │            │   │    │
│  │   │   │ /HALF_OPEN  │    │ /HALF_OPEN  │    │ /HALF_OPEN  │            │   │    │
│  │   │   └─────────────┘    └─────────────┘    └─────────────┘            │   │    │
│  │   └─────────────────────────────────────────────────────────────────────┘   │    │
│  │                                                                              │    │
│  │   降级链: Primary → Secondary → Fallback → 固定话术                         │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                            │                                                         │
│                            ▼                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                        模型调用层 (Model Caller)                             │    │
│  │                                                                              │    │
│  │   ┌──────────────────────────────────────────────────────────────────────┐  │    │
│  │   │                     火山方舟 API 客户端                                │  │    │
│  │   │                                                                       │  │    │
│  │   │  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐   │  │    │
│  │   │  │  doubao-seed-   │    │  doubao-pro-    │    │  doubao-lite-   │   │  │    │
│  │   │  │  character      │    │  32k            │    │  32k            │   │  │    │
│  │   │  │  (主模型)       │    │  (备用)         │    │  (兜底)         │   │  │    │
│  │   │  │                 │    │                 │    │                 │   │  │    │
│  │   │  │  角色扮演专用   │    │  通用对话       │    │  轻量快速       │   │  │    │
│  │   │  │  质量最高       │    │  质量良好       │    │  成本最低       │   │  │    │
│  │   │  └─────────────────┘    └─────────────────┘    └─────────────────┘   │  │    │
│  │   └──────────────────────────────────────────────────────────────────────┘  │    │
│  │                                                                              │    │
│  │   特性:                                                                      │    │
│  │   • 流式输出 (SSE/Server-Sent Events)                                       │    │
│  │   • 超时控制 (首Token超时 + 总超时)                                          │    │
│  │   • 重试机制 (可配置重试次数)                                                │    │
│  │   • 请求追踪 (Request ID)                                                   │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                            │                                                         │
│                            ▼                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                        响应处理层                                            │    │
│  │                                                                              │    │
│  │   • 流式数据解析                                                             │    │
│  │   • Token 统计                                                              │    │
│  │   • 异常处理 & 降级                                                          │    │
│  │   • 监控指标上报                                                             │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 模型配置对比

| 模型 | 用途 | 上下文 | 角色扮演 | 响应速度 | 成本 |
|------|------|--------|---------|---------|------|
| doubao-seed-character | 主模型 | 32K | ⭐⭐⭐⭐⭐ | 中 | 高 |
| doubao-pro-32k | 备用模型 | 32K | ⭐⭐⭐⭐ | 中 | 中 |
| doubao-lite-32k | 兜底模型 | 32K | ⭐⭐⭐ | 快 | 低 |

---

## 2. 熔断器设计

### 2.1 熔断器状态机

```
                                    ┌─────────────────────────────────────┐
                                    │           熔断器状态机               │
                                    └─────────────────────────────────────┘

                                              初始状态
                                                 │
                                                 ▼
                                    ┌─────────────────────────┐
                        ┌──────────│        CLOSED            │──────────┐
                        │          │        (关闭)            │          │
                        │          │                         │          │
                        │          │  • 正常处理请求          │          │
                        │          │  • 记录失败次数          │          │
                        │          └─────────────────────────┘          │
                        │                      │                        │
                        │                      │ 失败次数 >= 阈值        │
                        │                      │ (5次连续失败)          │
                        │                      ▼                        │
                        │          ┌─────────────────────────┐          │
                        │          │         OPEN             │          │
              请求成功   │          │        (打开)            │          │ 请求成功
                        │          │                         │          │
                        │          │  • 直接拒绝请求          │          │
                        │          │  • 快速失败              │          │
                        │          │  • 启动恢复计时器        │          │
                        │          └─────────────────────────┘          │
                        │                      │                        │
                        │                      │ 恢复超时               │
                        │                      │ (60秒后)               │
                        │                      ▼                        │
                        │          ┌─────────────────────────┐          │
                        └──────────│       HALF_OPEN         │──────────┘
                                   │       (半开)            │
                                   │                         │
                                   │  • 允许少量试探请求      │
                                   │  • 成功则关闭熔断器      │
                                   │  • 失败则重新打开        │
                                   └─────────────────────────┘
                                               │
                                               │ 请求失败
                                               │
                                               ▼
                                    返回 OPEN 状态
```

### 2.2 熔断器实现

```python
# services/circuit_breaker.py

import asyncio
import logging
from datetime import datetime, timedelta
from enum import Enum
from typing import Callable, Any, Optional
from dataclasses import dataclass, field
import threading

logger = logging.getLogger(__name__)


class CircuitState(Enum):
    """熔断器状态"""
    CLOSED = "closed"       # 关闭（正常）
    OPEN = "open"           # 打开（熔断）
    HALF_OPEN = "half_open" # 半开（试探）


@dataclass
class CircuitBreakerConfig:
    """熔断器配置"""
    failure_threshold: int = 5          # 失败阈值
    recovery_timeout: float = 60.0      # 恢复超时（秒）
    half_open_requests: int = 3         # 半开状态允许请求数
    success_threshold: int = 2          # 半开转关闭需要的成功数
    
    # 统计窗口
    window_size: float = 60.0           # 统计窗口大小（秒）


@dataclass
class CircuitBreakerState:
    """熔断器状态数据"""
    state: CircuitState = CircuitState.CLOSED
    failure_count: int = 0
    success_count: int = 0
    last_failure_time: Optional[datetime] = None
    last_state_change: datetime = field(default_factory=datetime.now)
    half_open_requests_made: int = 0


class CircuitBreaker:
    """
    熔断器
    
    使用方式:
    ```python
    cb = CircuitBreaker("model_name", config)
    
    async def call_with_breaker():
        if not cb.allow_request():
            raise CircuitBreakerOpenError()
        
        try:
            result = await actual_call()
            cb.record_success()
            return result
        except Exception as e:
            cb.record_failure()
            raise
    ```
    """
    
    def __init__(self, name: str, config: CircuitBreakerConfig = None):
        self.name = name
        self.config = config or CircuitBreakerConfig()
        self._state = CircuitBreakerState()
        self._lock = threading.Lock()
    
    @property
    def state(self) -> CircuitState:
        """获取当前状态"""
        self._check_state_transition()
        return self._state.state
    
    @property
    def is_open(self) -> bool:
        """熔断器是否打开"""
        return self.state == CircuitState.OPEN
    
    def allow_request(self) -> bool:
        """
        是否允许请求通过
        
        Returns:
            True: 允许请求
            False: 拒绝请求（熔断中）
        """
        with self._lock:
            self._check_state_transition()
            
            if self._state.state == CircuitState.CLOSED:
                return True
            
            if self._state.state == CircuitState.OPEN:
                return False
            
            # HALF_OPEN: 限制请求数
            if self._state.half_open_requests_made < self.config.half_open_requests:
                self._state.half_open_requests_made += 1
                return True
            
            return False
    
    def record_success(self):
        """记录成功"""
        with self._lock:
            if self._state.state == CircuitState.HALF_OPEN:
                self._state.success_count += 1
                
                # 达到成功阈值，关闭熔断器
                if self._state.success_count >= self.config.success_threshold:
                    self._transition_to(CircuitState.CLOSED)
                    logger.info(f"Circuit breaker [{self.name}] closed after recovery")
            
            elif self._state.state == CircuitState.CLOSED:
                # 重置失败计数
                self._state.failure_count = 0
    
    def record_failure(self):
        """记录失败"""
        with self._lock:
            self._state.failure_count += 1
            self._state.last_failure_time = datetime.now()
            
            if self._state.state == CircuitState.HALF_OPEN:
                # 半开状态失败，重新打开
                self._transition_to(CircuitState.OPEN)
                logger.warning(f"Circuit breaker [{self.name}] reopened after probe failure")
            
            elif self._state.state == CircuitState.CLOSED:
                # 检查是否达到失败阈值
                if self._state.failure_count >= self.config.failure_threshold:
                    self._transition_to(CircuitState.OPEN)
                    logger.warning(
                        f"Circuit breaker [{self.name}] opened after "
                        f"{self._state.failure_count} failures"
                    )
    
    def _check_state_transition(self):
        """检查是否需要状态转换"""
        if self._state.state == CircuitState.OPEN:
            # 检查是否可以进入半开状态
            time_since_open = datetime.now() - self._state.last_state_change
            if time_since_open.total_seconds() >= self.config.recovery_timeout:
                self._transition_to(CircuitState.HALF_OPEN)
                logger.info(f"Circuit breaker [{self.name}] entering half-open state")
    
    def _transition_to(self, new_state: CircuitState):
        """状态转换"""
        self._state.state = new_state
        self._state.last_state_change = datetime.now()
        self._state.failure_count = 0
        self._state.success_count = 0
        self._state.half_open_requests_made = 0
    
    def get_stats(self) -> dict:
        """获取统计信息"""
        return {
            "name": self.name,
            "state": self._state.state.value,
            "failure_count": self._state.failure_count,
            "success_count": self._state.success_count,
            "last_failure": self._state.last_failure_time.isoformat() if self._state.last_failure_time else None,
            "last_state_change": self._state.last_state_change.isoformat(),
        }


class CircuitBreakerOpenError(Exception):
    """熔断器打开异常"""
    def __init__(self, breaker_name: str):
        self.breaker_name = breaker_name
        super().__init__(f"Circuit breaker [{breaker_name}] is open")


class CircuitBreakerManager:
    """熔断器管理器"""
    
    _breakers: dict = {}
    
    @classmethod
    def get_breaker(cls, name: str, config: CircuitBreakerConfig = None) -> CircuitBreaker:
        """获取或创建熔断器"""
        if name not in cls._breakers:
            cls._breakers[name] = CircuitBreaker(name, config)
        return cls._breakers[name]
    
    @classmethod
    def get_all_stats(cls) -> dict:
        """获取所有熔断器状态"""
        return {name: breaker.get_stats() for name, breaker in cls._breakers.items()}
```

---

## 3. 大模型服务实现

### 3.1 配置定义

```python
# config/llm_config.py

from dataclasses import dataclass, field
from typing import List, Optional
from enum import Enum


class ModelTier(str, Enum):
    """模型层级"""
    PRIMARY = "primary"
    SECONDARY = "secondary"
    FALLBACK = "fallback"


@dataclass
class ModelConfig:
    """单个模型配置"""
    name: str                           # 模型名称
    endpoint_id: str                    # 火山方舟 endpoint ID
    tier: ModelTier                     # 层级
    
    # 超时配置
    first_token_timeout: float = 10.0   # 首Token超时
    total_timeout: float = 60.0         # 总超时
    
    # 重试配置
    max_retries: int = 2
    retry_delay: float = 0.5
    
    # 熔断器配置
    circuit_failure_threshold: int = 5
    circuit_recovery_timeout: float = 60.0
    
    # 模型参数
    max_tokens: int = 4096
    temperature: float = 0.8
    top_p: float = 0.9


@dataclass
class LLMServiceConfig:
    """LLM服务配置"""
    
    # 火山方舟配置
    ark_api_key: str = "your-api-key"
    ark_base_url: str = "https://ark.cn-beijing.volces.com/api/v3"
    
    # 模型配置
    models: List[ModelConfig] = field(default_factory=lambda: [
        ModelConfig(
            name="doubao-seed-character",
            endpoint_id="ep-xxx-seed-character",
            tier=ModelTier.PRIMARY,
            first_token_timeout=10.0,
            total_timeout=60.0,
            max_retries=2,
            circuit_failure_threshold=5,
            circuit_recovery_timeout=60.0,
            temperature=0.85,
        ),
        ModelConfig(
            name="doubao-pro-32k",
            endpoint_id="ep-xxx-pro-32k",
            tier=ModelTier.SECONDARY,
            first_token_timeout=8.0,
            total_timeout=45.0,
            max_retries=2,
            circuit_failure_threshold=5,
            circuit_recovery_timeout=60.0,
            temperature=0.8,
        ),
        ModelConfig(
            name="doubao-lite-32k",
            endpoint_id="ep-xxx-lite-32k",
            tier=ModelTier.FALLBACK,
            first_token_timeout=5.0,
            total_timeout=30.0,
            max_retries=1,
            circuit_failure_threshold=10,
            circuit_recovery_timeout=30.0,
            temperature=0.7,
        ),
    ])
    
    # 兜底话术
    fallback_responses: List[str] = field(default_factory=lambda: [
        "抱歉，我现在有点忙，稍后再聊好吗？",
        "emmm...让我想想，我们待会儿再聊这个话题吧~",
        "哎呀，我刚才走神了，你能再说一遍吗？",
    ])
    
    def get_model_by_tier(self, tier: ModelTier) -> Optional[ModelConfig]:
        """根据层级获取模型配置"""
        for model in self.models:
            if model.tier == tier:
                return model
        return None
    
    def get_models_in_order(self) -> List[ModelConfig]:
        """按优先级返回模型列表"""
        tier_order = [ModelTier.PRIMARY, ModelTier.SECONDARY, ModelTier.FALLBACK]
        return sorted(self.models, key=lambda m: tier_order.index(m.tier))


# 默认配置
llm_config = LLMServiceConfig()
```

### 3.2 火山方舟客户端

```python
# clients/ark_client.py

import logging
import asyncio
import json
from typing import List, Dict, Any, AsyncIterator, Optional
import httpx
from dataclasses import dataclass

logger = logging.getLogger(__name__)


@dataclass
class StreamChunk:
    """流式响应片段"""
    content: str
    finish_reason: Optional[str] = None
    usage: Optional[Dict[str, int]] = None


class ArkClientError(Exception):
    """Ark客户端错误"""
    def __init__(self, message: str, status_code: int = None, response: str = None):
        self.message = message
        self.status_code = status_code
        self.response = response
        super().__init__(message)


class ArkClient:
    """
    火山方舟 API 客户端
    
    支持流式和非流式调用
    """
    
    def __init__(self, api_key: str, base_url: str):
        self.api_key = api_key
        self.base_url = base_url.rstrip('/')
        self._client: Optional[httpx.AsyncClient] = None
    
    async def _get_client(self) -> httpx.AsyncClient:
        """获取HTTP客户端（懒加载）"""
        if self._client is None:
            self._client = httpx.AsyncClient(
                timeout=httpx.Timeout(
                    connect=5.0,
                    read=60.0,
                    write=10.0,
                    pool=5.0,
                ),
                headers={
                    "Authorization": f"Bearer {self.api_key}",
                    "Content-Type": "application/json",
                },
            )
        return self._client
    
    async def chat_stream(
        self,
        endpoint_id: str,
        messages: List[Dict[str, str]],
        max_tokens: int = 4096,
        temperature: float = 0.8,
        top_p: float = 0.9,
        **kwargs
    ) -> AsyncIterator[StreamChunk]:
        """
        流式对话
        
        Args:
            endpoint_id: 模型端点ID
            messages: 消息列表
            max_tokens: 最大Token数
            temperature: 温度
            top_p: Top-P采样
            
        Yields:
            流式响应片段
        """
        client = await self._get_client()
        
        payload = {
            "model": endpoint_id,
            "messages": messages,
            "max_tokens": max_tokens,
            "temperature": temperature,
            "top_p": top_p,
            "stream": True,
            **kwargs
        }
        
        url = f"{self.base_url}/chat/completions"
        
        async with client.stream("POST", url, json=payload) as response:
            if response.status_code != 200:
                error_body = await response.aread()
                raise ArkClientError(
                    f"API error: {response.status_code}",
                    status_code=response.status_code,
                    response=error_body.decode('utf-8')
                )
            
            async for line in response.aiter_lines():
                if not line:
                    continue
                
                if line.startswith("data: "):
                    data = line[6:]
                    
                    if data == "[DONE]":
                        break
                    
                    try:
                        chunk_data = json.loads(data)
                        choices = chunk_data.get("choices", [])
                        
                        if choices:
                            delta = choices[0].get("delta", {})
                            content = delta.get("content", "")
                            finish_reason = choices[0].get("finish_reason")
                            
                            if content or finish_reason:
                                yield StreamChunk(
                                    content=content,
                                    finish_reason=finish_reason,
                                    usage=chunk_data.get("usage"),
                                )
                    except json.JSONDecodeError:
                        logger.warning(f"Failed to parse SSE data: {data}")
                        continue
    
    async def chat(
        self,
        endpoint_id: str,
        messages: List[Dict[str, str]],
        max_tokens: int = 4096,
        temperature: float = 0.8,
        top_p: float = 0.9,
        **kwargs
    ) -> str:
        """
        非流式对话
        
        Args:
            endpoint_id: 模型端点ID
            messages: 消息列表
            
        Returns:
            完整回复文本
        """
        client = await self._get_client()
        
        payload = {
            "model": endpoint_id,
            "messages": messages,
            "max_tokens": max_tokens,
            "temperature": temperature,
            "top_p": top_p,
            "stream": False,
            **kwargs
        }
        
        url = f"{self.base_url}/chat/completions"
        
        response = await client.post(url, json=payload)
        
        if response.status_code != 200:
            raise ArkClientError(
                f"API error: {response.status_code}",
                status_code=response.status_code,
                response=response.text
            )
        
        data = response.json()
        choices = data.get("choices", [])
        
        if not choices:
            raise ArkClientError("Empty response from API")
        
        return choices[0]["message"]["content"]
    
    async def close(self):
        """关闭客户端"""
        if self._client:
            await self._client.aclose()
            self._client = None
```

### 3.3 LLM 服务主类

```python
# services/llm_service.py

import logging
import asyncio
import random
from typing import List, Dict, Any, AsyncIterator, Optional
from datetime import datetime
import uuid

from config.llm_config import LLMServiceConfig, ModelConfig, ModelTier, llm_config
from clients.ark_client import ArkClient, ArkClientError, StreamChunk
from services.circuit_breaker import (
    CircuitBreaker, CircuitBreakerConfig, 
    CircuitBreakerManager, CircuitBreakerOpenError
)

logger = logging.getLogger(__name__)


class LLMServiceError(Exception):
    """LLM服务错误"""
    pass


class AllModelsUnavailableError(LLMServiceError):
    """所有模型都不可用"""
    pass


class LLMService:
    """
    大模型服务
    
    特性:
    - 模型降级链: Primary → Secondary → Fallback → 固定话术
    - 熔断器: 自动熔断故障模型
    - 流式输出: SSE流式响应
    - 超时控制: 首Token超时 + 总超时
    - 重试机制: 可配置重试
    """
    
    def __init__(self, config: LLMServiceConfig = None):
        self.config = config or llm_config
        self.ark_client = ArkClient(
            api_key=self.config.ark_api_key,
            base_url=self.config.ark_base_url,
        )
        self._init_circuit_breakers()
    
    def _init_circuit_breakers(self):
        """初始化熔断器"""
        for model in self.config.models:
            CircuitBreakerManager.get_breaker(
                model.name,
                CircuitBreakerConfig(
                    failure_threshold=model.circuit_failure_threshold,
                    recovery_timeout=model.circuit_recovery_timeout,
                )
            )
    
    def _get_circuit_breaker(self, model_name: str) -> CircuitBreaker:
        """获取模型的熔断器"""
        return CircuitBreakerManager.get_breaker(model_name)
    
    async def chat_stream(
        self,
        messages: List[Dict[str, str]],
        character_config: Optional[Dict] = None,
        request_id: str = None,
    ) -> AsyncIterator[str]:
        """
        流式对话（带自动降级）
        
        Args:
            messages: 消息列表
            character_config: 角色配置（可选）
            request_id: 请求ID（可选，用于追踪）
            
        Yields:
            回复文本片段
        """
        request_id = request_id or str(uuid.uuid4())[:8]
        models = self.config.get_models_in_order()
        last_error = None
        
        for model in models:
            breaker = self._get_circuit_breaker(model.name)
            
            # 检查熔断器
            if not breaker.allow_request():
                logger.info(f"[{request_id}] Model {model.name} circuit is open, skipping")
                continue
            
            try:
                logger.info(f"[{request_id}] Trying model: {model.name}")
                
                async for chunk in self._call_model_stream(
                    model=model,
                    messages=messages,
                    request_id=request_id,
                ):
                    yield chunk
                
                # 成功，记录并返回
                breaker.record_success()
                logger.info(f"[{request_id}] Model {model.name} succeeded")
                return
                
            except asyncio.TimeoutError as e:
                last_error = e
                breaker.record_failure()
                logger.warning(f"[{request_id}] Model {model.name} timeout")
                continue
                
            except ArkClientError as e:
                last_error = e
                breaker.record_failure()
                logger.warning(f"[{request_id}] Model {model.name} error: {e.message}")
                continue
                
            except Exception as e:
                last_error = e
                breaker.record_failure()
                logger.error(f"[{request_id}] Model {model.name} unexpected error: {e}")
                continue
        
        # 所有模型都失败，返回兜底话术
        logger.error(f"[{request_id}] All models failed, using fallback response")
        fallback = random.choice(self.config.fallback_responses)
        yield fallback
    
    async def _call_model_stream(
        self,
        model: ModelConfig,
        messages: List[Dict[str, str]],
        request_id: str,
    ) -> AsyncIterator[str]:
        """
        调用单个模型（流式）
        
        带首Token超时和总超时控制
        """
        first_chunk_received = False
        start_time = datetime.now()
        
        try:
            # 创建流式调用任务
            stream = self.ark_client.chat_stream(
                endpoint_id=model.endpoint_id,
                messages=messages,
                max_tokens=model.max_tokens,
                temperature=model.temperature,
                top_p=model.top_p,
            )
            
            async for chunk in stream:
                # 首Token超时检查
                if not first_chunk_received:
                    elapsed = (datetime.now() - start_time).total_seconds()
                    if elapsed > model.first_token_timeout:
                        raise asyncio.TimeoutError(
                            f"First token timeout ({model.first_token_timeout}s)"
                        )
                    first_chunk_received = True
                    logger.debug(f"[{request_id}] First token received in {elapsed:.2f}s")
                
                # 总超时检查
                elapsed = (datetime.now() - start_time).total_seconds()
                if elapsed > model.total_timeout:
                    raise asyncio.TimeoutError(
                        f"Total timeout ({model.total_timeout}s)"
                    )
                
                if chunk.content:
                    yield chunk.content
                    
        except asyncio.TimeoutError:
            raise
        except Exception as e:
            raise ArkClientError(str(e))
    
    async def chat_simple(
        self,
        messages: List[Dict[str, str]],
        model: str = None,
        temperature: float = 0.7,
        max_tokens: int = 2048,
    ) -> str:
        """
        简单非流式对话（用于内部任务，如摘要生成）
        
        Args:
            messages: 消息列表
            model: 指定模型名称（可选）
            temperature: 温度
            max_tokens: 最大Token数
            
        Returns:
            完整回复文本
        """
        # 默认使用 pro 模型（性价比）
        model_config = None
        if model:
            for m in self.config.models:
                if m.name == model:
                    model_config = m
                    break
        
        if not model_config:
            model_config = self.config.get_model_by_tier(ModelTier.SECONDARY)
        
        try:
            return await asyncio.wait_for(
                self.ark_client.chat(
                    endpoint_id=model_config.endpoint_id,
                    messages=messages,
                    max_tokens=max_tokens,
                    temperature=temperature,
                ),
                timeout=model_config.total_timeout,
            )
        except asyncio.TimeoutError:
            raise LLMServiceError(f"Model {model_config.name} timeout")
        except ArkClientError as e:
            raise LLMServiceError(f"Model error: {e.message}")
    
    def get_health_status(self) -> Dict[str, Any]:
        """获取服务健康状态"""
        return {
            "models": CircuitBreakerManager.get_all_stats(),
            "available_models": [
                model.name for model in self.config.models
                if not self._get_circuit_breaker(model.name).is_open
            ],
        }
    
    async def close(self):
        """关闭服务"""
        await self.ark_client.close()


# 全局实例
llm_service = LLMService()
```

### 3.4 流式响应处理器

```python
# services/stream_handler.py

import logging
import asyncio
from typing import AsyncIterator, Callable, Optional
from dataclasses import dataclass
import time

logger = logging.getLogger(__name__)


@dataclass
class StreamStats:
    """流式响应统计"""
    first_token_time: float = 0.0       # 首Token时间（秒）
    total_time: float = 0.0             # 总时间（秒）
    chunk_count: int = 0                # 片段数量
    total_chars: int = 0                # 总字符数
    tokens_per_second: float = 0.0      # 每秒Token数（估算）


class StreamHandler:
    """
    流式响应处理器
    
    功能:
    - 首Token时间追踪
    - 响应统计
    - 超时保护
    - 回调支持
    """
    
    def __init__(
        self,
        on_chunk: Optional[Callable[[str], None]] = None,
        on_complete: Optional[Callable[[str, StreamStats], None]] = None,
        on_error: Optional[Callable[[Exception], None]] = None,
    ):
        self.on_chunk = on_chunk
        self.on_complete = on_complete
        self.on_error = on_error
    
    async def handle_stream(
        self,
        stream: AsyncIterator[str],
        timeout: float = 60.0,
    ) -> tuple[str, StreamStats]:
        """
        处理流式响应
        
        Args:
            stream: 流式响应迭代器
            timeout: 总超时时间
            
        Returns:
            (完整文本, 统计信息)
        """
        stats = StreamStats()
        full_text = ""
        start_time = time.time()
        first_chunk = True
        
        try:
            async for chunk in stream:
                current_time = time.time()
                
                # 首Token时间
                if first_chunk:
                    stats.first_token_time = current_time - start_time
                    first_chunk = False
                
                # 超时检查
                if current_time - start_time > timeout:
                    raise asyncio.TimeoutError(f"Stream timeout after {timeout}s")
                
                # 处理片段
                full_text += chunk
                stats.chunk_count += 1
                stats.total_chars += len(chunk)
                
                # 回调
                if self.on_chunk:
                    self.on_chunk(chunk)
            
            # 计算统计
            stats.total_time = time.time() - start_time
            if stats.total_time > 0:
                # 估算每秒Token数（假设平均1.5字符/token）
                estimated_tokens = stats.total_chars / 1.5
                stats.tokens_per_second = estimated_tokens / stats.total_time
            
            # 完成回调
            if self.on_complete:
                self.on_complete(full_text, stats)
            
            return full_text, stats
            
        except Exception as e:
            if self.on_error:
                self.on_error(e)
            raise


class StreamBuffer:
    """
    流式缓冲区
    
    用于平滑输出，避免过于频繁的小片段
    """
    
    def __init__(
        self,
        min_chunk_size: int = 1,        # 最小片段大小
        max_buffer_time: float = 0.1,   # 最大缓冲时间
    ):
        self.min_chunk_size = min_chunk_size
        self.max_buffer_time = max_buffer_time
        self._buffer = ""
        self._last_flush_time = time.time()
    
    async def process(
        self,
        stream: AsyncIterator[str]
    ) -> AsyncIterator[str]:
        """处理流并输出平滑的片段"""
        async for chunk in stream:
            self._buffer += chunk
            
            current_time = time.time()
            time_since_flush = current_time - self._last_flush_time
            
            # 满足条件则输出
            if (len(self._buffer) >= self.min_chunk_size or 
                time_since_flush >= self.max_buffer_time):
                if self._buffer:
                    yield self._buffer
                    self._buffer = ""
                    self._last_flush_time = current_time
        
        # 输出剩余内容
        if self._buffer:
            yield self._buffer
```

---

## 4. 监控与指标

### 4.1 监控指标定义

```python
# metrics/llm_metrics.py

from prometheus_client import Counter, Histogram, Gauge, Info

# 请求计数
LLM_REQUESTS = Counter(
    'llm_requests_total',
    'Total LLM requests',
    ['model', 'status']  # status: success/failure/timeout/circuit_open
)

# 模型使用计数（追踪降级情况）
LLM_MODEL_USAGE = Counter(
    'llm_model_usage_total',
    'LLM model usage count',
    ['model', 'tier']  # tier: primary/secondary/fallback
)

# 响应延迟
LLM_LATENCY = Histogram(
    'llm_latency_seconds',
    'LLM response latency',
    ['model', 'type'],  # type: first_token/total
    buckets=[0.5, 1, 2, 3, 5, 8, 10, 15, 20, 30, 45, 60]
)

# Token 使用量
LLM_TOKENS = Histogram(
    'llm_tokens_total',
    'LLM tokens used',
    ['model', 'type'],  # type: input/output
    buckets=[100, 500, 1000, 2000, 4000, 8000, 16000, 32000]
)

# 熔断器状态
LLM_CIRCUIT_STATE = Gauge(
    'llm_circuit_breaker_state',
    'Circuit breaker state (0=closed, 1=half_open, 2=open)',
    ['model']
)

# 降级计数
LLM_DEGRADATION = Counter(
    'llm_degradation_total',
    'LLM degradation events',
    ['from_model', 'to_model', 'reason']
)

# 兜底话术使用
LLM_FALLBACK_RESPONSES = Counter(
    'llm_fallback_responses_total',
    'Fallback response usage count'
)
```

### 4.2 监控集成

```python
# services/llm_service_monitored.py

import time
import logging
from typing import List, Dict, Any, AsyncIterator
from functools import wraps

from services.llm_service import LLMService
from config.llm_config import ModelConfig, ModelTier
from metrics.llm_metrics import (
    LLM_REQUESTS, LLM_MODEL_USAGE, LLM_LATENCY,
    LLM_TOKENS, LLM_CIRCUIT_STATE, LLM_DEGRADATION,
    LLM_FALLBACK_RESPONSES
)

logger = logging.getLogger(__name__)


class MonitoredLLMService(LLMService):
    """
    带监控的LLM服务
    
    在原有服务基础上添加 Prometheus 指标采集
    """
    
    async def chat_stream(
        self,
        messages: List[Dict[str, str]],
        character_config: Dict = None,
        request_id: str = None,
    ) -> AsyncIterator[str]:
        """流式对话（带监控）"""
        start_time = time.time()
        first_token_time = None
        used_model = None
        fallback_used = False
        total_chars = 0
        
        try:
            models = self.config.get_models_in_order()
            attempted_models = []
            
            for model in models:
                breaker = self._get_circuit_breaker(model.name)
                
                # 更新熔断器状态指标
                state_value = {"closed": 0, "half_open": 1, "open": 2}
                LLM_CIRCUIT_STATE.labels(model=model.name).set(
                    state_value.get(breaker.state.value, 0)
                )
                
                if not breaker.allow_request():
                    LLM_REQUESTS.labels(model=model.name, status='circuit_open').inc()
                    continue
                
                attempted_models.append(model.name)
                
                try:
                    async for chunk in self._call_model_stream(
                        model=model,
                        messages=messages,
                        request_id=request_id,
                    ):
                        # 首Token时间
                        if first_token_time is None:
                            first_token_time = time.time() - start_time
                            LLM_LATENCY.labels(
                                model=model.name, type='first_token'
                            ).observe(first_token_time)
                        
                        total_chars += len(chunk)
                        yield chunk
                    
                    # 成功
                    used_model = model
                    breaker.record_success()
                    
                    # 记录指标
                    LLM_REQUESTS.labels(model=model.name, status='success').inc()
                    LLM_MODEL_USAGE.labels(model=model.name, tier=model.tier.value).inc()
                    LLM_LATENCY.labels(model=model.name, type='total').observe(
                        time.time() - start_time
                    )
                    
                    # 估算Token数
                    estimated_output_tokens = total_chars / 1.5
                    LLM_TOKENS.labels(model=model.name, type='output').observe(
                        estimated_output_tokens
                    )
                    
                    return
                    
                except Exception as e:
                    breaker.record_failure()
                    
                    if isinstance(e, TimeoutError):
                        LLM_REQUESTS.labels(model=model.name, status='timeout').inc()
                    else:
                        LLM_REQUESTS.labels(model=model.name, status='failure').inc()
                    
                    # 记录降级
                    if len(attempted_models) > 1:
                        LLM_DEGRADATION.labels(
                            from_model=attempted_models[-2] if len(attempted_models) > 1 else 'none',
                            to_model=model.name,
                            reason=type(e).__name__
                        ).inc()
                    
                    continue
            
            # 所有模型失败，使用兜底
            fallback_used = True
            LLM_FALLBACK_RESPONSES.inc()
            
            import random
            fallback = random.choice(self.config.fallback_responses)
            yield fallback
            
        finally:
            # 记录总延迟（包括失败情况）
            total_time = time.time() - start_time
            logger.info(
                f"LLM request completed: model={used_model.name if used_model else 'fallback'}, "
                f"time={total_time:.2f}s, first_token={first_token_time:.2f}s if first_token_time else 'N/A', "
                f"fallback={fallback_used}"
            )
```

---

## 5. 使用示例

### 5.1 在对话服务中使用

```python
# services/chat_service.py (更新)

from services.llm_service import llm_service
from services.stream_handler import StreamHandler, StreamStats

class ChatService:
    
    async def handle_message(
        self,
        user_id: str,
        character_id: str,
        user_input: str
    ) -> AsyncIterator[str]:
        """处理用户消息"""
        
        # ... 获取上下文 ...
        
        # 调用大模型（流式）
        request_id = str(uuid.uuid4())[:8]
        
        async for chunk in llm_service.chat_stream(
            messages=context_messages,
            character_config=character_config.dict(),
            request_id=request_id,
        ):
            yield chunk
```

### 5.2 健康检查接口

```python
# api/routes/health.py

from fastapi import APIRouter
from services.llm_service import llm_service

router = APIRouter(prefix="/api/v1/health", tags=["health"])


@router.get("/llm")
async def llm_health():
    """LLM服务健康状态"""
    status = llm_service.get_health_status()
    
    # 判断整体健康状态
    available_count = len(status["available_models"])
    total_count = len(status["models"])
    
    if available_count == 0:
        health = "unhealthy"
    elif available_count < total_count:
        health = "degraded"
    else:
        health = "healthy"
    
    return {
        "status": health,
        "available_models": status["available_models"],
        "circuit_breakers": status["models"],
    }
```

---

## 6. 告警配置

```yaml
# 告警规则
alerts:
  # 主模型熔断
  - name: "主模型熔断"
    expr: llm_circuit_breaker_state{model="doubao-seed-character"} == 2
    for: 1m
    severity: warning
    message: "主模型 doubao-seed-character 已熔断，已降级到备用模型"
    
  # 所有模型熔断
  - name: "所有模型熔断"
    expr: sum(llm_circuit_breaker_state == 2) == 3
    for: 30s
    severity: critical
    message: "所有LLM模型已熔断，服务降级到兜底话术"
    
  # 首Token延迟过高
  - name: "首Token延迟过高"
    expr: |
      histogram_quantile(0.99, 
        rate(llm_latency_seconds_bucket{type="first_token"}[5m])
      ) > 8
    for: 5m
    severity: warning
    message: "LLM首Token P99延迟超过8秒"
    
  # 总延迟过高
  - name: "LLM总延迟过高"
    expr: |
      histogram_quantile(0.99,
        rate(llm_latency_seconds_bucket{type="total"}[5m])
      ) > 30
    for: 5m
    severity: warning
    message: "LLM总响应P99延迟超过30秒"
    
  # 降级率过高
  - name: "LLM降级率过高"
    expr: |
      rate(llm_model_usage_total{tier!="primary"}[5m])
      / rate(llm_model_usage_total[5m]) > 0.2
    for: 10m
    severity: warning
    message: "LLM降级率超过20%"
    
  # 兜底话术使用过多
  - name: "兜底话术使用频繁"
    expr: rate(llm_fallback_responses_total[5m]) > 0.1
    for: 5m
    severity: critical
    message: "LLM兜底话术使用率过高，请检查模型服务"
```

---

## 7. 总结

### 大模型服务核心特性

| 特性 | 实现方式 | 效果 |
|------|----------|------|
| **模型降级链** | Primary → Secondary → Fallback → 话术 | 100%可用 |
| **熔断器** | 5次失败触发，60秒恢复 | 快速失败 |
| **流式输出** | SSE + 异步迭代器 | 低首Token延迟 |
| **超时控制** | 首Token + 总超时双重控制 | 防止卡死 |
| **监控指标** | Prometheus + 多维度指标 | 全面可观测 |

### 性能目标

| 指标 | 目标 | 降级时 |
|------|------|--------|
| 首Token延迟 | P99 < 5s | P99 < 8s |
| 总延迟 | P99 < 30s | P99 < 45s |
| 成功率 | > 99.9% | > 99% (含降级) |
| 降级率 | < 5% | < 20% |

### 文件结构

```
src/
├── config/
│   └── llm_config.py           # LLM配置
├── clients/
│   └── ark_client.py           # 火山方舟客户端
├── services/
│   ├── circuit_breaker.py      # 熔断器
│   ├── llm_service.py          # LLM服务主类
│   └── stream_handler.py       # 流式处理
└── metrics/
    └── llm_metrics.py          # 监控指标
```
