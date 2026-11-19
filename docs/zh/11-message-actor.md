# 第11章：消息传递与Actor模型

## 学习目标

通过本章学习，您将：
- 理解 NautilusTrader 的消息总线架构
- 掌握 Actor 模型的核心概念和实现
- 学会组件间通信机制
- 实现高性能的并发处理
- 构建可扩展的系统架构

## 11.1 消息总线架构

### 11.1.1 消息总线基础

```python
"""
NautilusTrader 消息总线实现
"""

from abc import ABC, abstractmethod
from typing import Any, Dict, List, Optional, Callable, Set
from collections import defaultdict, deque
from dataclasses import dataclass
from enum import Enum
import asyncio
import threading
import time
import uuid
from datetime import datetime

from nautilus_trader.core.message import Message
from nautilus_trader.core.uuid import UUID4
from nautilus_trader.model.identifiers import TraderId


class MessageType(Enum):
    """消息类型枚举"""
    COMMAND = "command"
    EVENT = "event"
    QUERY = "query"
    RESPONSE = "response"
    HEARTBEAT = "heartbeat"


@dataclass
class MessageEnvelope:
    """消息封装"""
    id: UUID4
    type: MessageType
    topic: str
    payload: Any
    sender: str
    recipient: Optional[str] = None
    timestamp: Optional[datetime] = None
    correlation_id: Optional[UUID4] = None
    priority: int = 0  # 优先级，数字越大优先级越高

    def __post_init__(self):
        if self.timestamp is None:
            self.timestamp = datetime.utcnow()


class IMessageHandler(ABC):
    """消息处理器接口"""

    @abstractmethod
    def handle(self, envelope: MessageEnvelope) -> None:
        """处理消息"""
        pass

    @abstractmethod
    def get_subscriptions(self) -> List[str]:
        """获取订阅的主题"""
        pass


class MessageBus:
    """高性能消息总线实现"""

    def __init__(self, max_queue_size: int = 10000):
        """
        初始化消息总线

        Parameters
        ----------
        max_queue_size : int
            最大队列大小
        """
        self._handlers: Dict[str, List[IMessageHandler]] = defaultdict(list)
        self._message_queue = asyncio.Queue(maxsize=max_queue_size)
        self._priority_queue = []
        self._running = False
        self._worker_tasks: List[asyncio.Task] = []
        self._stats = {
            'messages_sent': 0,
            'messages_received': 0,
            'messages_processed': 0,
            'handlers_registered': 0,
            'errors': 0
        }
        self._lock = threading.RLock()

    def subscribe(self, topic: str, handler: IMessageHandler):
        """
        订阅主题

        Parameters
        ----------
        topic : str
            主题名称
        handler : IMessageHandler
            消息处理器
        """
        with self._lock:
            self._handlers[topic].append(handler)
            self._stats['handlers_registered'] += 1

    def unsubscribe(self, topic: str, handler: IMessageHandler):
        """
        取消订阅

        Parameters
        ----------
        topic : str
            主题名称
        handler : IMessageHandler
            消息处理器
        """
        with self._lock:
            if topic in self._handlers and handler in self._handlers[topic]:
                self._handlers[topic].remove(handler)
                self._stats['handlers_registered'] -= 1

    async def publish(self, envelope: MessageEnvelope) -> bool:
        """
        发布消息

        Parameters
        ----------
        envelope : MessageEnvelope
            消息封装

        Returns
        -------
        bool
            是否成功发布
        """
        try:
            if envelope.priority > 0:
                # 高优先级消息
                await self._message_queue.put(envelope)
            else:
                # 普通消息
                await self._message_queue.put(envelope)

            self._stats['messages_sent'] += 1
            return True
        except asyncio.QueueFull:
            self._stats['errors'] += 1
            return False

    async def send(self, topic: str, payload: Any, sender: str,
                   recipient: Optional[str] = None,
                   message_type: MessageType = MessageType.EVENT,
                   priority: int = 0) -> bool:
        """
        发送消息的便捷方法

        Parameters
        ----------
        topic : str
            主题名称
        payload : Any
            消息负载
        sender : str
            发送者
        recipient : str, optional
            接收者
        message_type : MessageType
            消息类型
        priority : int
            优先级

        Returns
        -------
        bool
            是否成功发送
        """
        envelope = MessageEnvelope(
            id=UUID4(),
            type=message_type,
            topic=topic,
            payload=payload,
            sender=sender,
            recipient=recipient,
            priority=priority
        )
        return await self.publish(envelope)

    async def start(self, num_workers: int = 4):
        """启动消息总线"""
        self._running = True
        self._worker_tasks = [
            asyncio.create_task(self._worker(f"worker-{i}"))
            for i in range(num_workers)
        ]

    async def stop(self):
        """停止消息总线"""
        self._running = False

        # 等待所有任务完成
        if self._worker_tasks:
            await asyncio.gather(*self._worker_tasks, return_exceptions=True)
            self._worker_tasks.clear()

    async def _worker(self, name: str):
        """工作线程"""
        while self._running:
            try:
                # 获取消息（带超时）
                envelope = await asyncio.wait_for(
                    self._message_queue.get(),
                    timeout=1.0
                )

                # 处理消息
                await self._process_message(envelope)
                self._message_queue.task_done()
                self._stats['messages_processed'] += 1

            except asyncio.TimeoutError:
                continue
            except Exception as e:
                print(f"Worker {name} error: {e}")
                self._stats['errors'] += 1

    async def _process_message(self, envelope: MessageEnvelope):
        """处理消息"""
        handlers = self._handlers.get(envelope.topic, [])

        # 如果指定了接收者，只调用该接收者
        if envelope.recipient:
            handlers = [h for h in handlers if getattr(h, 'name', None) == envelope.recipient]

        # 并发调用所有处理器
        tasks = []
        for handler in handlers:
            task = asyncio.create_task(self._safe_handle(handler, envelope))
            tasks.append(task)

        if tasks:
            await asyncio.gather(*tasks, return_exceptions=True)

    async def _safe_handle(self, handler: IMessageHandler, envelope: MessageEnvelope):
        """安全调用处理器"""
        try:
            # 如果是同步方法，在线程池中执行
            if not asyncio.iscoroutinefunction(handler.handle):
                loop = asyncio.get_event_loop()
                await loop.run_in_executor(None, handler.handle, envelope)
            else:
                await handler.handle(envelope)
        except Exception as e:
            print(f"Handler error: {e}")
            self._stats['errors'] += 1

    def get_stats(self) -> Dict:
        """获取统计信息"""
        with self._lock:
            return self._stats.copy()

    def get_queue_size(self) -> int:
        """获取队列大小"""
        return self._message_queue.qsize()


# 使用示例
class OrderEventHandler(IMessageHandler):
    """订单事件处理器"""

    def __init__(self, trader_id: TraderId):
        self.trader_id = trader_id
        self.order_count = 0

    def handle(self, envelope: MessageEnvelope) -> None:
        """处理订单事件"""
        if envelope.type == MessageType.EVENT:
            self.order_count += 1
            print(f"[{self.trader_id.value}] Received order: {envelope.payload}")

    def get_subscriptions(self) -> List[str]:
        return ["orders.*"]


class MarketDataHandler(IMessageHandler):
    """市场数据处理器"""

    def __init__(self, name: str):
        self.name = name
        self.quote_count = 0

    def handle(self, envelope: MessageEnvelope) -> None:
        """处理市场数据"""
        if envelope.topic.startswith("market."):
            self.quote_count += 1
            # 可以在这里进行技术指标计算等

    def get_subscriptions(self) -> List[str]:
        return ["market.*"]


# 消息总线使用示例
class MessageBusExample:
    """消息总线使用示例"""

    def __init__(self):
        self.message_bus = MessageBus(max_queue_size=5000)

    async def run_example(self):
        """运行示例"""
        print("启动消息总线示例...")

        # 创建处理器
        order_handler = OrderEventHandler(TraderId("TRADER-001"))
        market_handler = MarketDataHandler("MARKET-001")

        # 注册订阅
        self.message_bus.subscribe("orders.*", order_handler)
        self.message_bus.subscribe("market.BTCUSDT", market_handler)

        # 启动消息总线
        await self.message_bus.start(num_workers=2)

        # 发送测试消息
        await self._send_test_messages()

        # 等待处理完成
        await asyncio.sleep(2)

        # 输出统计
        stats = self.message_bus.get_stats()
        print("\n消息总线统计:")
        print(f"  发送消息数: {stats['messages_sent']}")
        print(f"  处理消息数: {stats['messages_processed']}")
        print(f"  错误数: {stats['errors']}")
        print(f"  队列大小: {self.message_bus.get_queue_size()}")

        # 停止消息总线
        await self.message_bus.stop()

    async def _send_test_messages(self):
        """发送测试消息"""
        # 发送订单事件
        for i in range(100):
            await self.message_bus.send(
                topic="orders.new",
                payload={"order_id": f"ORDER-{i}", "symbol": "BTCUSDT", "quantity": 1.0},
                sender="ORDER-MANAGER",
                message_type=MessageType.EVENT
            )

        # 发送市场数据
        for i in range(200):
            await self.message_bus.send(
                topic="market.BTCUSDT",
                payload={"bid": 50000.0 + i, "ask": 50001.0 + i, "volume": 10.0},
                sender="MARKET-FEED",
                message_type=MessageType.EVENT,
                priority=1  # 市场数据高优先级
            )
```

### 11.1.2 消息路由和过滤

```python
"""
消息路由和过滤系统
"""

import re
from typing import Pattern, Dict, List, Callable
from dataclasses import dataclass


@dataclass
class RoutingRule:
    """路由规则"""
    pattern: str
    handler: IMessageHandler
    priority: int = 0
    condition: Optional[Callable[[MessageEnvelope], bool]] = None


class MessageRouter:
    """消息路由器"""

    def __init__(self):
        self._rules: List[RoutingRule] = []
        self._compiled_patterns: Dict[str, Pattern] = {}

    def add_rule(self, rule: RoutingRule):
        """
        添加路由规则

        Parameters
        ----------
        rule : RoutingRule
            路由规则
        """
        # 编译正则表达式
        if rule.pattern not in self._compiled_patterns:
            # 将通配符转换为正则表达式
            regex_pattern = rule.pattern.replace("*", ".*").replace("?", ".")
            self._compiled_patterns[rule.pattern] = re.compile(f"^{regex_pattern}$")

        self._rules.append(rule)
        # 按优先级排序
        self._rules.sort(key=lambda r: r.priority, reverse=True)

    def remove_rule(self, pattern: str, handler: IMessageHandler):
        """移除路由规则"""
        self._rules = [
            rule for rule in self._rules
            if rule.pattern != pattern or rule.handler != handler
        ]

    def route(self, envelope: MessageEnvelope) -> List[IMessageHandler]:
        """
        路由消息到合适的处理器

        Parameters
        ----------
        envelope : MessageEnvelope
            消息封装

        Returns
        -------
        List[IMessageHandler]
            匹配的处理器列表
        """
        handlers = []

        for rule in self._rules:
            # 检查主题是否匹配
            pattern = self._compiled_patterns[rule.pattern]
            if pattern.match(envelope.topic):
                # 检查额外条件
                if rule.condition is None or rule.condition(envelope):
                    handlers.append(rule.handler)

        return handlers


class MessageFilter:
    """消息过滤器"""

    @staticmethod
    def by_sender(sender: str) -> Callable[[MessageEnvelope], bool]:
        """按发送者过滤"""
        return lambda envelope: envelope.sender == sender

    @staticmethod
    def by_type(message_type: MessageType) -> Callable[[MessageEnvelope], bool]:
        """按消息类型过滤"""
        return lambda envelope: envelope.type == message_type

    @staticmethod
    def by_priority(min_priority: int) -> Callable[[MessageEnvelope], bool]:
        """按最小优先级过滤"""
        return lambda envelope: envelope.priority >= min_priority

    @staticmethod
    def by_payload_field(field: str, value: Any) -> Callable[[MessageEnvelope], bool]:
        """按负载字段过滤"""
        return lambda envelope: (
            isinstance(envelope.payload, dict) and
            envelope.payload.get(field) == value
        )

    @staticmethod
    def by_time_range(start_time: datetime, end_time: datetime) -> Callable[[MessageEnvelope], bool]:
        """按时间范围过滤"""
        return lambda envelope: (
            start_time <= envelope.timestamp <= end_time if envelope.timestamp else False
        )

    @staticmethod
    def composite(*filters: Callable) -> Callable[[MessageEnvelope], bool]:
        """组合过滤器（AND）"""
        return lambda envelope: all(f(envelope) for f in filters)

    @staticmethod
    def any_of(*filters: Callable) -> Callable[[MessageEnvelope], bool]:
        """任意过滤器（OR）"""
        return lambda envelope: any(f(envelope) for f in filters)


# 增强的消息总线
class AdvancedMessageBus(MessageBus):
    """增强的消息总线，支持路由和过滤"""

    def __init__(self, max_queue_size: int = 10000):
        super().__init__(max_queue_size)
        self._router = MessageRouter()
        self._filters: List[Callable] = []

    def add_routing_rule(self, pattern: str, handler: IMessageHandler,
                        priority: int = 0,
                        condition: Optional[Callable] = None):
        """添加路由规则"""
        rule = RoutingRule(
            pattern=pattern,
            handler=handler,
            priority=priority,
            condition=condition
        )
        self._router.add_rule(rule)

    def add_global_filter(self, filter_func: Callable):
        """添加全局过滤器"""
        self._filters.append(filter_func)

    async def _process_message(self, envelope: MessageEnvelope):
        """处理消息（带路由和过滤）"""
        # 应用全局过滤器
        for filter_func in self._filters:
            if not filter_func(envelope):
                return  # 消息被过滤

        # 获取路由的处理器
        handlers = self._router.route(envelope)

        # 如果指定了接收者，额外检查
        if envelope.recipient:
            handlers = [h for h in handlers if getattr(h, 'name', None) == envelope.recipient]

        # 并发调用所有处理器
        tasks = []
        for handler in handlers:
            task = asyncio.create_task(self._safe_handle(handler, envelope))
            tasks.append(task)

        if tasks:
            await asyncio.gather(*tasks, return_exceptions=True)
```

## 11.2 Actor 模型实现

### 11.2.1 Actor 基础框架

```python
"""
Actor 模型实现
"""

import asyncio
from typing import Any, Dict, Optional, Callable, Set
from collections import deque
from dataclasses import dataclass
from enum import Enum
import threading
import time
import weakref

from nautilus_trader.core.uuid import UUID4


class ActorState(Enum):
    """Actor状态"""
    STARTING = "starting"
    RUNNING = "running"
    STOPPING = "stopping"
    STOPPED = "stopped"
    CRASHED = "crashed"


@dataclass
class ActorMessage:
    """Actor消息"""
    sender: str
    message: Any
    correlation_id: Optional[UUID4] = None
    timestamp: float = None

    def __post_init__(self):
        if self.timestamp is None:
            self.timestamp = time.time()


class IActor(ABC):
    """Actor接口"""

    @abstractmethod
    async def receive(self, message: ActorMessage) -> Any:
        """接收并处理消息"""
        pass

    @abstractmethod
    def get_name(self) -> str:
        """获取Actor名称"""
        pass


class ActorRef:
    """Actor引用"""

    def __init__(self, name: str, actor_system: 'ActorSystem'):
        self.name = name
        self._actor_system = actor_system

    async def tell(self, message: Any, sender: Optional[str] = None):
        """异步发送消息（fire-and-forget）"""
        await self._actor_system.send(self.name, message, sender)

    async def ask(self, message: Any, timeout: float = 5.0) -> Any:
        """异步发送消息并等待响应"""
        return await self._actor_system.request(self.name, message, timeout)

    def __repr__(self):
        return f"ActorRef({self.name})"


class Actor(IActor):
    """基础Actor实现"""

    def __init__(self, name: str):
        self.name = name
        self.state = ActorState.STOPPED
        self.mailbox = asyncio.Queue()
        self._actor_system = None
        self._supervisor = None
        self._children: Set[str] = set()
        self._task = None
        self._context = {}

    def get_name(self) -> str:
        return self.name

    async def start(self, actor_system: 'ActorSystem', supervisor: Optional['Actor'] = None):
        """启动Actor"""
        self.state = ActorState.STARTING
        self._actor_system = actor_system
        self._supervisor = supervisor

        # 调用启动钩子
        await self.on_start()

        # 启动消息处理循环
        self._task = asyncio.create_task(self._run())
        self.state = ActorState.RUNNING

    async def stop(self):
        """停止Actor"""
        if self.state != ActorState.RUNNING:
            return

        self.state = ActorState.STOPPING

        # 停止所有子Actor
        for child_name in list(self._children):
            await self._actor_system.stop_actor(child_name)

        # 调用停止钩子
        await self.on_stop()

        # 取消任务
        if self._task:
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass

        self.state = ActorState.STOPPED

    async def _run(self):
        """Actor主循环"""
        while self.state == ActorState.RUNNING:
            try:
                # 获取消息（带超时，以便检查状态）
                message = await asyncio.wait_for(
                    self.mailbox.get(),
                    timeout=1.0
                )

                # 处理消息
                await self.receive(message)

            except asyncio.TimeoutError:
                continue
            except Exception as e:
                # 处理异常
                await self.handle_error(e)

    async def receive(self, message: ActorMessage) -> Any:
        """接收并处理消息（子类实现）"""
        # 子类应该重写此方法
        pass

    async def spawn(self, actor_class, name: str, *args, **kwargs) -> ActorRef:
        """创建子Actor"""
        actor = actor_class(name, *args, **kwargs)
        await self._actor_system.register_actor(actor, parent=self)
        self._children.add(name)
        return ActorRef(name, self._actor_system)

    async def send_to_child(self, child_name: str, message: Any):
        """发送消息给子Actor"""
        await self._actor_system.send(child_name, message, self.name)

    async def handle_error(self, error: Exception):
        """处理错误"""
        self.state = ActorState.CRASHED

        # 通知监督者
        if self._supervisor:
            await self._supervisor.handle_child_error(self.name, error)

    # 生命周期钩子
    async def on_start(self):
        """启动时调用"""
        pass

    async def on_stop(self):
        """停止时调用"""
        pass

    async def on_child_terminated(self, child_name: str):
        """子Actor终止时调用"""
        self._children.discard(child_name)


class SupervisorStrategy:
    """监督策略"""

    async def handle_failure(self, supervisor: Actor, child_name: str,
                           error: Exception) -> bool:
        """处理失败"""
        raise NotImplementedError


class RestartStrategy(SupervisorStrategy):
    """重启策略"""

    def __init__(self, max_retries: int = 3, backoff: float = 1.0):
        self.max_retries = max_retries
        self.backoff = backoff
        self._retry_counts: Dict[str, int] = {}

    async def handle_failure(self, supervisor: Actor, child_name: str,
                           error: Exception) -> bool:
        """处理失败"""
        retry_count = self._retry_counts.get(child_name, 0)

        if retry_count < self.max_retries:
            self._retry_counts[child_name] = retry_count + 1

            # 等待一段时间再重启
            await asyncio.sleep(self.backoff * (2 ** retry_count))

            # 重启子Actor
            await supervisor._actor_system.restart_actor(child_name)
            return True
        else:
            # 超过最大重试次数，停止子Actor
            await supervisor._actor_system.stop_actor(child_name)
            self._retry_counts[child_name] = 0
            return False


class OneForOneStrategy(RestartStrategy):
    """一对一重启策略"""

    async def handle_failure(self, supervisor: Actor, child_name: str,
                           error: Exception) -> bool:
        """只重启失败的子Actor"""
        return await super().handle_failure(supervisor, child_name, error)


class AllForOneStrategy(RestartStrategy):
    """全体重启策略"""

    async def handle_failure(self, supervisor: Actor, child_name: str,
                           error: Exception) -> bool:
        """重启所有子Actor"""
        # 重启所有子Actor
        for child in list(supervisor._children):
            await super().handle_failure(supervisor, child, Exception("Cascading restart"))
        return True
```

### 11.2.2 Actor 系统

```python
"""
Actor系统实现
"""

import asyncio
from typing import Dict, Any, Optional, List
from collections import defaultdict


class ActorSystem:
    """Actor系统"""

    def __init__(self, name: str):
        self.name = name
        self._actors: Dict[str, Actor] = {}
        self._actor_tasks: Dict[str, asyncio.Task] = {}
        self._pending_requests: Dict[UUID4, asyncio.Future] = {}
        self._stats = {
            'actors_created': 0,
            'actors_started': 0,
            'messages_sent': 0,
            'messages_processed': 0,
            'errors': 0
        }
        self._running = False

    async def start(self):
        """启动Actor系统"""
        self._running = True

    async def stop(self):
        """停止Actor系统"""
        self._running = False

        # 停止所有Actor
        for actor_name in list(self._actors.keys()):
            await self.stop_actor(actor_name)

    async def register_actor(self, actor: Actor, parent: Optional[Actor] = None) -> ActorRef:
        """注册Actor"""
        self._actors[actor.name] = actor
        self._stats['actors_created'] += 1
        return ActorRef(actor.name, self)

    async def start_actor(self, actor_name: str, supervisor: Optional[Actor] = None):
        """启动Actor"""
        if actor_name not in self._actors:
            raise ValueError(f"Actor {actor_name} not registered")

        actor = self._actors[actor_name]
        await actor.start(self, supervisor)
        self._stats['actors_started'] += 1

    async def stop_actor(self, actor_name: str):
        """停止Actor"""
        if actor_name in self._actors:
            actor = self._actors[actor_name]
            await actor.stop()
            del self._actors[actor_name]

    async def restart_actor(self, actor_name: str):
        """重启Actor"""
        if actor_name in self._actors:
            actor = self._actors[actor_name]
            await actor.stop()
            await actor.start(self, actor._supervisor)

    async def send(self, recipient: str, message: Any, sender: Optional[str] = None):
        """发送消息"""
        if recipient not in self._actors:
            raise ValueError(f"Actor {recipient} not found")

        actor = self._actors[recipient]
        actor_message = ActorMessage(
            sender=sender or "system",
            message=message
        )

        await actor.mailbox.put(actor_message)
        self._stats['messages_sent'] += 1

    async def request(self, recipient: str, message: Any, timeout: float = 5.0) -> Any:
        """发送请求并等待响应"""
        correlation_id = UUID4()
        future = asyncio.Future()
        self._pending_requests[correlation_id] = future

        # 发送请求
        request_message = ActorMessage(
            sender="system",
            message={
                'type': 'request',
                'payload': message,
                'correlation_id': correlation_id
            },
            correlation_id=correlation_id
        )

        await self.send(recipient, request_message)

        # 等待响应
        try:
            response = await asyncio.wait_for(future, timeout=timeout)
            return response
        except asyncio.TimeoutError:
            del self._pending_requests[correlation_id]
            raise TimeoutError(f"Request to {recipient} timed out")
        finally:
            if correlation_id in self._pending_requests:
                del self._pending_requests[correlation_id]

    async def respond(self, sender: str, correlation_id: UUID4, response: Any):
        """响应请求"""
        if correlation_id in self._pending_requests:
            future = self._pending_requests[correlation_id]
            if not future.done():
                future.set_result(response)

    def get_actor_ref(self, name: str) -> ActorRef:
        """获取Actor引用"""
        if name not in self._actors:
            raise ValueError(f"Actor {name} not found")
        return ActorRef(name, self)

    def get_stats(self) -> Dict:
        """获取统计信息"""
        self._stats.update({
            'active_actors': len(self._actors),
            'pending_requests': len(self._pending_requests)
        })
        return self._stats.copy()


# 具体Actor实现示例

class OrderManagerActor(Actor):
    """订单管理Actor"""

    def __init__(self, name: str):
        super().__init__(name)
        self.orders = {}
        self.next_order_id = 1

    async def receive(self, message: ActorMessage) -> Any:
        """处理订单相关消息"""
        msg = message.message

        if isinstance(msg, dict):
            msg_type = msg.get('type')

            if msg_type == 'submit_order':
                return await self._handle_submit_order(msg)
            elif msg_type == 'cancel_order':
                return await self._handle_cancel_order(msg)
            elif msg_type == 'get_order':
                return await self._handle_get_order(msg)
            elif msg_type == 'request':
                # 处理请求-响应模式
                payload = msg.get('payload')
                correlation_id = msg.get('correlation_id')
                response = await self._process_request(payload)
                if correlation_id:
                    await self._actor_system.respond(
                        self.name,
                        correlation_id,
                        response
                    )

    async def _handle_submit_order(self, msg: dict) -> str:
        """处理提交订单"""
        order_id = f"ORDER-{self.next_order_id:06d}"
        self.next_order_id += 1

        order = {
            'order_id': order_id,
            'symbol': msg.get('symbol'),
            'side': msg.get('side'),
            'quantity': msg.get('quantity'),
            'price': msg.get('price'),
            'status': 'PENDING',
            'created_at': time.time()
        }

        self.orders[order_id] = order

        # 模拟发送到交易所
        await asyncio.sleep(0.1)
        order['status'] = 'SUBMITTED'

        self.log.info(f"Order submitted: {order_id}")
        return order_id

    async def _handle_cancel_order(self, msg: dict) -> bool:
        """处理取消订单"""
        order_id = msg.get('order_id')
        if order_id in self.orders:
            self.orders[order_id]['status'] = 'CANCELLED'
            self.log.info(f"Order cancelled: {order_id}")
            return True
        return False

    async def _handle_get_order(self, msg: dict) -> Optional[dict]:
        """处理查询订单"""
        order_id = msg.get('order_id')
        return self.orders.get(order_id)

    async def _process_request(self, payload: dict) -> Any:
        """处理请求"""
        if payload.get('action') == 'get_all_orders':
            return list(self.orders.values())
        elif payload.get('action') == 'get_position':
            # 计算持仓
            position = 0
            for order in self.orders.values():
                if order['status'] == 'FILLED':
                    if order['side'] == 'BUY':
                        position += order['quantity']
                    else:
                        position -= order['quantity']
            return {'position': position}
        else:
            return {'error': 'Unknown action'}


class MarketDataActor(Actor):
    """市场数据Actor"""

    def __init__(self, name: str):
        super().__init__(name)
        self.subscribers = set()
        self.running = False

    async def on_start(self):
        """启动时开始推送市场数据"""
        await super().on_start()
        self.running = True
        # 启动市场数据推送任务
        asyncio.create_task(self._push_market_data())

    async def on_stop(self):
        """停止时停止推送"""
        await super().on_stop()
        self.running = False

    async def receive(self, message: ActorMessage) -> Any:
        """处理订阅请求"""
        msg = message.message

        if isinstance(msg, dict):
            if msg.get('type') == 'subscribe':
                self.subscribers.add(message.sender)
                self.log.info(f"Added subscriber: {message.sender}")
            elif msg.get('type') == 'unsubscribe':
                self.subscribers.discard(message.sender)
                self.log.info(f"Removed subscriber: {message.sender}")

    async def _push_market_data(self):
        """推送市场数据"""
        while self.running:
            # 生成模拟市场数据
            market_data = {
                'symbol': 'BTCUSDT',
                'bid': 50000.0 + (time.time() % 1000),
                'ask': 50001.0 + (time.time() % 1000),
                'volume': 10.0,
                'timestamp': time.time()
            }

            # 发送给所有订阅者
            for subscriber in list(self.subscribers):
                try:
                    await self._actor_system.send(subscriber, {
                        'type': 'market_data',
                        'data': market_data
                    }, self.name)
                except:
                    # 订阅者可能已经停止
                    self.subscribers.discard(subscriber)

            await asyncio.sleep(0.1)  # 10 Hz

    def get_name(self) -> str:
        return self.name


class StrategyActor(Actor):
    """策略Actor"""

    def __init__(self, name: str, strategy_config: dict):
        super().__init__(name)
        self.config = strategy_config
        self.position = 0
        self.last_price = None

    async def on_start(self):
        """启动时订阅市场数据"""
        await super().on_start()

        # 订阅市场数据
        market_data_ref = self._actor_system.get_actor_ref("market_data")
        await market_data_ref.tell({
            'type': 'subscribe'
        }, self.name)

        # 获取订单管理器引用
        self.order_manager = self._actor_system.get_actor_ref("order_manager")

    async def receive(self, message: ActorMessage) -> Any:
        """处理市场数据和订单更新"""
        msg = message.message

        if isinstance(msg, dict):
            if msg.get('type') == 'market_data':
                await self._handle_market_data(msg.get('data'))
            elif msg.get('type') == 'order_update':
                await self._handle_order_update(msg.get('data'))

    async def _handle_market_data(self, data: dict):
        """处理市场数据"""
        price = (data['bid'] + data['ask']) / 2

        if self.last_price is None:
            self.last_price = price
            return

        # 简单的移动平均策略
        price_change = price - self.last_price

        if price_change > 100 and self.position <= 0:
            # 价格上涨，买入
            await self.order_manager.tell({
                'type': 'submit_order',
                'symbol': 'BTCUSDT',
                'side': 'BUY',
                'quantity': 1.0,
                'price': data['ask']
            }, self.name)
            self.position = 1

        elif price_change < -100 and self.position >= 0:
            # 价格下跌，卖出
            await self.order_manager.tell({
                'type': 'submit_order',
                'symbol': 'BTCUSDT',
                'side': 'SELL',
                'quantity': 1.0,
                'price': data['bid']
            }, self.name)
            self.position = -1

        self.last_price = price

    async def _handle_order_update(self, data: dict):
        """处理订单更新"""
        if data.get('status') == 'FILLED':
            self.log.info(f"Order filled: {data.get('order_id')}")


# Actor系统使用示例
async def actor_system_example():
    """Actor系统使用示例"""
    print("启动Actor系统示例...")

    # 创建Actor系统
    system = ActorSystem("TRADING_SYSTEM")
    await system.start()

    # 创建Actor
    market_data_actor = MarketDataActor("market_data")
    order_manager_actor = OrderManagerActor("order_manager")
    strategy_actor = StrategyActor("strategy", {"symbol": "BTCUSDT"})

    # 注册Actor
    await system.register_actor(market_data_actor)
    await system.register_actor(order_manager_actor)
    await system.register_actor(strategy_actor)

    # 启动Actor
    await system.start_actor("market_data")
    await system.start_actor("order_manager")
    await system.start_actor("strategy")

    # 运行一段时间
    await asyncio.sleep(10)

    # 获取统计信息
    stats = system.get_stats()
    print("\nActor系统统计:")
    print(f"  活跃Actor数: {stats['active_actors']}")
    print(f"  发送消息数: {stats['messages_sent']}")
    print(f"  处理消息数: {stats['messages_processed']}")

    # 停止系统
    await system.stop()
    print("\nActor系统已停止")


if __name__ == "__main__":
    asyncio.run(actor_system_example())
```

## 11.3 组件间通信

### 11.3.1 请求-响应模式

```python
"""
请求-响应通信模式
"""

import asyncio
from typing import Dict, Any, Optional, Callable, Awaitable
from dataclasses import dataclass
import time


@dataclass
class Request:
    """请求"""
    id: str
    method: str
    params: Dict[str, Any]
    timeout: float = 5.0
    timestamp: float = None

    def __post_init__(self):
        if self.timestamp is None:
            self.timestamp = time.time()


@dataclass
class Response:
    """响应"""
    request_id: str
    result: Any = None
    error: Optional[str] = None
    timestamp: float = None

    def __post_init__(self):
        if self.timestamp is None:
            self.timestamp = time.time()


class RPCRegistry:
    """RPC注册表"""

    def __init__(self):
        self._methods: Dict[str, Callable] = {}

    def register(self, name: str, method: Callable):
        """注册RPC方法"""
        self._methods[name] = method

    def unregister(self, name: str):
        """注销RPC方法"""
        if name in self._methods:
            del self._methods[name]

    async def call(self, request: Request) -> Response:
        """调用RPC方法"""
        if request.method not in self._methods:
            return Response(
                request_id=request.id,
                error=f"Method '{request.method}' not found"
            )

        try:
            method = self._methods[request.method]

            # 支持异步和同步方法
            if asyncio.iscoroutinefunction(method):
                result = await asyncio.wait_for(
                    method(**request.params),
                    timeout=request.timeout
                )
            else:
                result = await asyncio.wait_for(
                    asyncio.get_event_loop().run_in_executor(
                        None,
                        lambda: method(**request.params)
                    ),
                    timeout=request.timeout
                )

            return Response(
                request_id=request.id,
                result=result
            )

        except asyncio.TimeoutError:
            return Response(
                request_id=request.id,
                error=f"Method '{request.method}' timed out"
            )
        except Exception as e:
            return Response(
                request_id=request.id,
                error=str(e)
            )


class RPCClient:
    """RPC客户端"""

    def __init__(self, message_bus: MessageBus, service_name: str):
        self.message_bus = message_bus
        self.service_name = service_name
        self._pending_requests: Dict[str, asyncio.Future] = {}

    async def call(self, method: str, params: Dict[str, Any],
                   timeout: float = 5.0) -> Any:
        """调用远程方法"""
        request_id = str(uuid.uuid4())

        # 创建请求
        request = Request(
            id=request_id,
            method=method,
            params=params,
            timeout=timeout
        )

        # 创建Future等待响应
        future = asyncio.Future()
        self._pending_requests[request_id] = future

        # 发送请求
        await self.message_bus.send(
            topic=f"rpc.{self.service_name}",
            payload=request,
            sender="rpc_client"
        )

        # 等待响应
        try:
            response = await asyncio.wait_for(future, timeout=timeout)
            if response.error:
                raise Exception(response.error)
            return response.result
        except asyncio.TimeoutError:
            if request_id in self._pending_requests:
                del self._pending_requests[request_id]
            raise TimeoutError(f"RPC call to {method} timed out")
        finally:
            if request_id in self._pending_requests:
                del self._pending_requests[request_id]

    async def handle_response(self, response: Response):
        """处理响应"""
        if response.request_id in self._pending_requests:
            future = self._pending_requests[response.request_id]
            if not future.done():
                future.set_result(response)


class RPCServer:
    """RPC服务器"""

    def __init__(self, message_bus: MessageBus, service_name: str):
        self.message_bus = message_bus
        self.service_name = service_name
        self.registry = RPCRegistry()

    def register_method(self, name: str, method: Callable):
        """注册方法"""
        self.registry.register(name, method)

    async def start(self):
        """启动RPC服务器"""
        # 订阅RPC请求
        await self.message_bus.subscribe(
            f"rpc.{self.service_name}",
            self
        )

    async def handle(self, envelope: MessageEnvelope):
        """处理RPC请求"""
        if isinstance(envelope.payload, Request):
            request = envelope.payload

            # 调用方法
            response = await self.registry.call(request)

            # 发送响应
            await self.message_bus.send(
                topic=f"rpc_response.{envelope.sender}",
                payload=response,
                sender=self.service_name
            )


# 使用示例

class TradingService:
    """交易服务"""

    def __init__(self):
        self.positions = {}
        self.orders = {}

    async def submit_order(self, symbol: str, side: str,
                          quantity: float, price: float) -> str:
        """提交订单"""
        order_id = f"ORDER-{int(time.time())}"

        order = {
            'order_id': order_id,
            'symbol': symbol,
            'side': side,
            'quantity': quantity,
            'price': price,
            'status': 'PENDING',
            'created_at': time.time()
        }

        self.orders[order_id] = order

        # 模拟订单处理
        await asyncio.sleep(0.1)
        order['status'] = 'FILLED'

        # 更新持仓
        self._update_position(symbol, side, quantity)

        return order_id

    def _update_position(self, symbol: str, side: str, quantity: float):
        """更新持仓"""
        if symbol not in self.positions:
            self.positions[symbol] = 0

        if side == 'BUY':
            self.positions[symbol] += quantity
        else:
            self.positions[symbol] -= quantity

    def get_position(self, symbol: str) -> float:
        """获取持仓"""
        return self.positions.get(symbol, 0)

    def get_order(self, order_id: str) -> Optional[dict]:
        """获取订单"""
        return self.orders.get(order_id)


async def rpc_example():
    """RPC使用示例"""
    print("启动RPC示例...")

    # 创建消息总线
    message_bus = MessageBus()
    await message_bus.start()

    # 创建交易服务
    trading_service = TradingService()

    # 创建RPC服务器
    rpc_server = RPCServer(message_bus, "trading")
    rpc_server.register_method("submit_order", trading_service.submit_order)
    rpc_server.register_method("get_position", trading_service.get_position)
    rpc_server.register_method("get_order", trading_service.get_order)
    await rpc_server.start()

    # 创建RPC客户端
    rpc_client = RPCClient(message_bus, "trading")

    # 测试RPC调用
    print("\n测试RPC调用...")

    # 提交订单
    order_id = await rpc_client.call("submit_order", {
        'symbol': 'BTCUSDT',
        'side': 'BUY',
        'quantity': 1.0,
        'price': 50000.0
    })
    print(f"提交订单成功: {order_id}")

    # 查询订单
    order = await rpc_client.call("get_order", {
        'order_id': order_id
    })
    print(f"订单状态: {order['status']}")

    # 查询持仓
    position = await rpc_client.call("get_position", {
        'symbol': 'BTCUSDT'
    })
    print(f"BTC持仓: {position}")

    # 停止
    await message_bus.stop()
    print("\nRPC示例完成")
```

### 11.3.2 发布-订阅模式

```python
"""
发布-订阅通信模式
"""

from typing import Dict, List, Any, Callable, Set
from collections import defaultdict
import asyncio


class Event:
    """事件基类"""

    def __init__(self, topic: str, data: Any = None):
        self.topic = topic
        self.data = data
        self.timestamp = time.time()
        self.id = str(uuid.uuid4())


class EventSubscriber:
    """事件订阅者"""

    def __init__(self, name: str):
        self.name = name
        self.subscriptions: Set[str] = set()

    def subscribe(self, topic: str):
        """订阅主题"""
        self.subscriptions.add(topic)

    def unsubscribe(self, topic: str):
        """取消订阅"""
        self.subscriptions.discard(topic)

    async def handle_event(self, event: Event):
        """处理事件（子类实现）"""
        raise NotImplementedError


class EventBus:
    """事件总线"""

    def __init__(self):
        self._subscribers: Dict[str, List[EventSubscriber]] = defaultdict(list)
        self._event_queue = asyncio.Queue()
        self._running = False
        self._worker_task = None

    def subscribe(self, topic: str, subscriber: EventSubscriber):
        """订阅主题"""
        self._subscribers[topic].append(subscriber)
        subscriber.subscribe(topic)

    def unsubscribe(self, topic: str, subscriber: EventSubscriber):
        """取消订阅"""
        if topic in self._subscribers:
            self._subscribers[topic].remove(subscriber)
        subscriber.unsubscribe(topic)

    async def publish(self, event: Event):
        """发布事件"""
        await self._event_queue.put(event)

    async def start(self):
        """启动事件总线"""
        self._running = True
        self._worker_task = asyncio.create_task(self._worker())

    async def stop(self):
        """停止事件总线"""
        self._running = False
        if self._worker_task:
            self._worker_task.cancel()
            try:
                await self._worker_task
            except asyncio.CancelledError:
                pass

    async def _worker(self):
        """事件处理工作线程"""
        while self._running:
            try:
                # 获取事件
                event = await asyncio.wait_for(
                    self._event_queue.get(),
                    timeout=1.0
                )

                # 获取订阅者
                subscribers = self._subscribers.get(event.topic, [])

                # 并发通知所有订阅者
                tasks = []
                for subscriber in subscribers:
                    task = asyncio.create_task(
                        self._safe_notify(subscriber, event)
                    )
                    tasks.append(task)

                if tasks:
                    await asyncio.gather(*tasks, return_exceptions=True)

            except asyncio.TimeoutError:
                continue
            except Exception as e:
                print(f"Event bus error: {e}")

    async def _safe_notify(self, subscriber: EventSubscriber, event: Event):
        """安全通知订阅者"""
        try:
            await subscriber.handle_event(event)
        except Exception as e:
            print(f"Subscriber {subscriber.name} error: {e}")


# 具体事件实现

class OrderEvent(Event):
    """订单事件"""

    def __init__(self, order_id: str, status: str, data: dict = None):
        super().__init__("orders", data)
        self.order_id = order_id
        self.status = status


class MarketEvent(Event):
    """市场事件"""

    def __init__(self, symbol: str, price: float, volume: float):
        super().__init__("market", {
            'symbol': symbol,
            'price': price,
            'volume': volume
        })
        self.symbol = symbol
        self.price = price
        self.volume = volume


class PositionEvent(Event):
    """持仓事件"""

    def __init__(self, symbol: str, position: float, pnl: float):
        super().__init__("position", {
            'symbol': symbol,
            'position': position,
            'pnl': pnl
        })
        self.symbol = symbol
        self.position = position
        self.pnl = pnl


# 事件订阅者实现

class RiskManager(EventSubscriber):
    """风险管理器"""

    def __init__(self, max_position: float = 10.0):
        super().__init__("RiskManager")
        self.max_position = max_position
        self.positions = {}

    async def handle_event(self, event: Event):
        """处理事件"""
        if event.topic == "orders":
            await self._handle_order_event(event)
        elif event.topic == "position":
            await self._handle_position_event(event)

    async def _handle_order_event(self, event: OrderEvent):
        """处理订单事件"""
        if event.status == "FILLED":
            # 检查是否超出仓位限制
            symbol = event.data['symbol']
            current_position = self.positions.get(symbol, 0)

            if event.data['side'] == 'BUY':
                new_position = current_position + event.data['quantity']
            else:
                new_position = current_position - event.data['quantity']

            if abs(new_position) > self.max_position:
                print(f"警告: {symbol} 仓位超出限制!")
                # 可以在这里发送平仓订单

    async def _handle_position_event(self, event: PositionEvent):
        """处理持仓事件"""
        self.positions[event.symbol] = event.position


class PortfolioManager(EventSubscriber):
    """投资组合管理器"""

    def __init__(self):
        super().__init__("PortfolioManager")
        self.portfolio_value = 0
        self.positions = {}

    async def handle_event(self, event: Event):
        """处理事件"""
        if event.topic == "market":
            await self._handle_market_event(event)
        elif event.topic == "position":
            await self._handle_position_event(event)

    async def _handle_market_event(self, event: MarketEvent):
        """处理市场事件"""
        # 更新投资组合价值
        if event.symbol in self.positions:
            position = self.positions[event.symbol]
            self.portfolio_value += position * event.price
            print(f"投资组合价值更新: ${self.portfolio_value:,.2f}")

    async def _handle_position_event(self, event: PositionEvent):
        """处理持仓事件"""
        self.positions[event.symbol] = event.position
        print(f"持仓更新: {event.symbol} = {event.position}")


class ExecutionAlgo(EventSubscriber):
    """执行算法"""

    def __init__(self, name: str):
        super().__init__(name)
        self.active_orders = {}

    async def handle_event(self, event: Event):
        """处理事件"""
        if event.topic == "market":
            await self._handle_market_event(event)

    async def _handle_market_event(self, event: MarketEvent):
        """处理市场事件"""
        # 检查是否有需要执行的订单
        # 这里可以实现TWAP、VWAP等算法
        pass


# 发布-订阅模式使用示例

async def pubsub_example():
    """发布-订阅模式示例"""
    print("\n启动发布-订阅示例...")

    # 创建事件总线
    event_bus = EventBus()
    await event_bus.start()

    # 创建订阅者
    risk_manager = RiskManager(max_position=5.0)
    portfolio_manager = PortfolioManager()
    execution_algo = ExecutionAlgo("TWAP")

    # 订阅事件
    event_bus.subscribe("orders", risk_manager)
    event_bus.subscribe("position", risk_manager)
    event_bus.subscribe("position", portfolio_manager)
    event_bus.subscribe("market", portfolio_manager)
    event_bus.subscribe("market", execution_algo)

    # 发布订单事件
    await event_bus.publish(OrderEvent(
        order_id="ORDER-001",
        status="FILLED",
        data={
            'symbol': 'BTCUSDT',
            'side': 'BUY',
            'quantity': 2.0,
            'price': 50000.0
        }
    ))

    # 发布市场数据事件
    for i in range(5):
        await event_bus.publish(MarketEvent(
            symbol='BTCUSDT',
            price=50000.0 + i * 100,
            volume=10.0
        ))
        await asyncio.sleep(0.1)

    # 发布持仓事件
    await event_bus.publish(PositionEvent(
        symbol='BTCUSDT',
        position=2.0,
        pnl=0.0
    ))

    # 等待处理完成
    await asyncio.sleep(1)

    # 停止事件总线
    await event_bus.stop()
    print("发布-订阅示例完成")


if __name__ == "__main__":
    asyncio.run(pubsub_example())
```

## 11.4 并发处理模式

### 11.4.1 工作池模式

```python
"""
工作池并发模式
"""

import asyncio
from typing import List, Callable, Any, Optional
from dataclasses import dataclass
import time


@dataclass
class Task:
    """任务"""
    id: str
    func: Callable
    args: tuple
    kwargs: dict
    priority: int = 0
    timeout: Optional[float] = None
    retry_count: int = 0
    max_retries: int = 0


class Worker:
    """工作线程"""

    def __init__(self, worker_id: str, task_queue: asyncio.Queue):
        self.worker_id = worker_id
        self.task_queue = task_queue
        self.running = False
        self.current_task = None
        self.stats = {
            'tasks_completed': 0,
            'tasks_failed': 0,
            'total_time': 0
        }

    async def start(self):
        """启动工作线程"""
        self.running = True
        asyncio.create_task(self._worker_loop())

    async def stop(self):
        """停止工作线程"""
        self.running = False

    async def _worker_loop(self):
        """工作循环"""
        while self.running:
            try:
                # 获取任务
                task = await asyncio.wait_for(
                    self.task_queue.get(),
                    timeout=1.0
                )

                self.current_task = task
                start_time = time.time()

                # 执行任务
                try:
                    if asyncio.iscoroutinefunction(task.func):
                        if task.timeout:
                            result = await asyncio.wait_for(
                                task.func(*task.args, **task.kwargs),
                                timeout=task.timeout
                            )
                        else:
                            result = await task.func(*task.args, **task.kwargs)
                    else:
                        # 同步函数在线程池中执行
                        loop = asyncio.get_event_loop()
                        if task.timeout:
                            result = await asyncio.wait_for(
                                loop.run_in_executor(
                                    None,
                                    lambda: task.func(*task.args, **task.kwargs)
                                ),
                                timeout=task.timeout
                            )
                        else:
                            result = await loop.run_in_executor(
                                None,
                                lambda: task.func(*task.args, **task.kwargs)
                            )

                    # 任务成功
                    self.stats['tasks_completed'] += 1
                    await self._handle_task_success(task, result)

                except Exception as e:
                    # 任务失败
                    self.stats['tasks_failed'] += 1
                    await self._handle_task_failure(task, e)

                finally:
                    self.current_task = None
                    elapsed = time.time() - start_time
                    self.stats['total_time'] += elapsed
                    self.task_queue.task_done()

            except asyncio.TimeoutError:
                continue
            except Exception as e:
                print(f"Worker {self.worker_id} error: {e}")

    async def _handle_task_success(self, task: Task, result: Any):
        """处理任务成功"""
        # 可以在这里添加成功回调
        pass

    async def _handle_task_failure(self, task: Task, error: Exception):
        """处理任务失败"""
        # 重试逻辑
        if task.retry_count < task.max_retries:
            task.retry_count += 1
            await self.task_queue.put(task)
            print(f"Task {task.id} failed, retrying ({task.retry_count}/{task.max_retries})")
        else:
            print(f"Task {task.id} failed after {task.max_retries} retries: {error}")


class ThreadPool:
    """线程池"""

    def __init__(self, num_workers: int = 4, max_queue_size: int = 1000):
        self.num_workers = num_workers
        self.task_queue = asyncio.Queue(maxsize=max_queue_size)
        self.workers: List[Worker] = []
        self.running = False
        self.stats = {
            'tasks_submitted': 0,
            'tasks_completed': 0,
            'tasks_failed': 0,
            'queue_size': 0
        }

    async def start(self):
        """启动线程池"""
        self.running = True

        # 创建工作线程
        for i in range(self.num_workers):
            worker = Worker(f"worker-{i}", self.task_queue)
            self.workers.append(worker)
            await worker.start()

    async def stop(self):
        """停止线程池"""
        self.running = False

        # 停止所有工作线程
        for worker in self.workers:
            await worker.stop()

    async def submit(self, func: Callable, *args, priority: int = 0,
                     timeout: Optional[float] = None,
                     max_retries: int = 0, **kwargs) -> str:
        """提交任务"""
        task_id = str(uuid.uuid4())
        task = Task(
            id=task_id,
            func=func,
            args=args,
            kwargs=kwargs,
            priority=priority,
            timeout=timeout,
            max_retries=max_retries
        )

        await self.task_queue.put(task)
        self.stats['tasks_submitted'] += 1
        return task_id

    async def submit_batch(self, tasks: List[dict]) -> List[str]:
        """批量提交任务"""
        task_ids = []
        for task_config in tasks:
            task_id = await self.submit(**task_config)
            task_ids.append(task_id)
        return task_ids

    def get_stats(self) -> dict:
        """获取统计信息"""
        self.stats['queue_size'] = self.task_queue.qsize()

        # 聚合工作线程统计
        total_completed = sum(w.stats['tasks_completed'] for w in self.workers)
        total_failed = sum(w.stats['tasks_failed'] for w in self.workers)
        total_time = sum(w.stats['total_time'] for w in self.workers)

        self.stats.update({
            'tasks_completed': total_completed,
            'tasks_failed': total_failed,
            'avg_task_time': total_time / max(total_completed, 1)
        })

        return self.stats.copy()


# 使用示例

async def process_market_data(symbol: str, data: dict) -> dict:
    """处理市场数据"""
    # 模拟耗时操作
    await asyncio.sleep(0.1)

    # 计算技术指标
    price = data['price']
    volume = data['volume']

    result = {
        'symbol': symbol,
        'price': price,
        'volume': volume,
        'vwap': price,  # 简化
        'rsi': 50,      # 简化
        'timestamp': time.time()
    }

    return result


async def execute_order(order: dict) -> dict:
    """执行订单"""
    # 模拟订单执行
    await asyncio.sleep(0.2)

    result = {
        'order_id': order['order_id'],
        'status': 'FILLED',
        'filled_quantity': order['quantity'],
        'avg_price': order['price'],
        'timestamp': time.time()
    }

    return result


def calculate_risk(positions: dict) -> dict:
    """计算风险指标（CPU密集型）"""
    # 模拟CPU密集型计算
    total = 0
    for i in range(1000000):
        total += i * 0.000001

    risk_metrics = {
        'total_exposure': sum(pos['quantity'] * pos['price'] for pos in positions.values()),
        'var': total * 0.01,  # 简化
        'beta': 1.5,
        'sharpe': 2.0
    }

    return risk_metrics


async def thread_pool_example():
    """线程池使用示例"""
    print("\n启动线程池示例...")

    # 创建线程池
    thread_pool = ThreadPool(num_workers=4, max_queue_size=100)
    await thread_pool.start()

    # 提交市场数据处理任务
    market_tasks = []
    for i in range(20):
        task_id = await thread_pool.submit(
            process_market_data,
            f"BTCUSDT-{i}",
            {'price': 50000.0 + i, 'volume': 10.0},
            priority=1
        )
        market_tasks.append(task_id)

    # 提交订单执行任务
    order_tasks = []
    for i in range(10):
        task_id = await thread_pool.submit(
            execute_order,
            {
                'order_id': f'ORDER-{i}',
                'symbol': 'BTCUSDT',
                'quantity': 1.0,
                'price': 50000.0
            },
            priority=2,
            timeout=1.0,
            max_retries=2
        )
        order_tasks.append(task_id)

    # 提交风险计算任务
    positions = {
        'BTCUSDT': {'quantity': 10, 'price': 50000},
        'ETHUSDT': {'quantity': 100, 'price': 3000}
    }
    risk_task_id = await thread_pool.submit(
        calculate_risk,
        positions,
        priority=0
    )

    # 等待所有任务完成
    print("等待任务完成...")
    await thread_pool.task_queue.join()

    # 输出统计信息
    stats = thread_pool.get_stats()
    print("\n线程池统计:")
    print(f"  提交任务数: {stats['tasks_submitted']}")
    print(f"  完成任务数: {stats['tasks_completed']}")
    print(f"  失败任务数: {stats['tasks_failed']}")
    print(f"  平均任务时间: {stats['avg_task_time']:.3f}秒")

    # 停止线程池
    await thread_pool.stop()
    print("\n线程池示例完成")


if __name__ == "__main__":
    asyncio.run(thread_pool_example())
```

## 11.5 错误处理和恢复

### 11.5.1 断路器模式

```python
"""
断路器模式实现
"""

import asyncio
import time
from enum import Enum
from typing import Callable, Any, Optional
from dataclasses import dataclass


class CircuitState(Enum):
    """断路器状态"""
    CLOSED = "closed"      # 正常状态
    OPEN = "open"          # 断路状态
    HALF_OPEN = "half_open"  # 半开状态


@dataclass
class CircuitBreakerConfig:
    """断路器配置"""
    failure_threshold: int = 5      # 失败阈值
    recovery_timeout: float = 60.0  # 恢复超时
    expected_exception: type = Exception  # 预期异常
    success_threshold: int = 3      # 成功阈值（半开状态）


class CircuitBreaker:
    """断路器实现"""

    def __init__(self, config: CircuitBreakerConfig):
        self.config = config
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None
        self.stats = {
            'calls': 0,
            'successes': 0,
            'failures': 0,
            'timeouts': 0
        }

    async def call(self, func: Callable, *args, timeout: Optional[float] = None, **kwargs) -> Any:
        """通过断路器调用函数"""
        self.stats['calls'] += 1

        # 检查断路器状态
        if self.state == CircuitState.OPEN:
            if self._should_attempt_reset():
                self.state = CircuitState.HALF_OPEN
                self.success_count = 0
            else:
                raise Exception("Circuit breaker is OPEN")

        # 执行函数
        start_time = time.time()
        try:
            if timeout:
                result = await asyncio.wait_for(
                    func(*args, **kwargs),
                    timeout=timeout
                )
            else:
                result = await func(*args, **kwargs)

            # 成功
            self._on_success()
            self.stats['successes'] += 1
            return result

        except asyncio.TimeoutError:
            # 超时
            self._on_failure()
            self.stats['timeouts'] += 1
            raise

        except self.config.expected_exception as e:
            # 预期异常
            self._on_failure()
            self.stats['failures'] += 1
            raise

        except Exception as e:
            # 未预期异常，也算失败
            self._on_failure()
            self.stats['failures'] += 1
            raise

        finally:
            # 记录调用时间
            elapsed = time.time() - start_time
            if 'total_time' not in self.stats:
                self.stats['total_time'] = 0
            self.stats['total_time'] += elapsed

    def _should_attempt_reset(self) -> bool:
        """是否应该尝试重置"""
        return (
            self.last_failure_time and
            time.time() - self.last_failure_time >= self.config.recovery_timeout
        )

    def _on_success(self):
        """处理成功"""
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.config.success_threshold:
                self.state = CircuitState.CLOSED
                self.failure_count = 0
        else:
            self.failure_count = 0

    def _on_failure(self):
        """处理失败"""
        self.failure_count += 1
        self.last_failure_time = time.time()

        if self.failure_count >= self.config.failure_threshold:
            self.state = CircuitState.OPEN

    def get_state(self) -> CircuitState:
        """获取当前状态"""
        return self.state

    def get_stats(self) -> dict:
        """获取统计信息"""
        stats = self.stats.copy()
        stats['failure_rate'] = (
            stats['failures'] / max(stats['calls'], 1)
        )
        stats['avg_time'] = (
            stats.get('total_time', 0) / max(stats['successes'], 1)
        )
        stats['state'] = self.state.value
        return stats


# 使用示例

async def unreliable_api_call(endpoint: str) -> dict:
    """模拟不可靠的API调用"""
    # 模拟50%的失败率
    if time.time() % 2 < 1:
        raise Exception("API Error")

    # 模拟网络延迟
    await asyncio.sleep(0.1)

    return {
        'endpoint': endpoint,
        'data': f"Response from {endpoint}",
        'timestamp': time.time()
    }


async def circuit_breaker_example():
    """断路器使用示例"""
    print("\n启动断路器示例...")

    # 创建断路器
    config = CircuitBreakerConfig(
        failure_threshold=3,
        recovery_timeout=2.0,
        success_threshold=2
    )
    breaker = CircuitBreaker(config)

    # 测试调用
    for i in range(20):
        try:
            result = await breaker.call(
                unreliable_api_call,
                f"/api/v1/data/{i}",
                timeout=0.5
            )
            print(f"调用 {i}: 成功 - {result['data']}")
        except Exception as e:
            print(f"调用 {i}: 失败 - {str(e)}")

        await asyncio.sleep(0.5)

    # 输出统计
    stats = breaker.get_stats()
    print("\n断路器统计:")
    print(f"  总调用数: {stats['calls']}")
    print(f"  成功数: {stats['successes']}")
    print(f"  失败数: {stats['failures']}")
    print(f"  超时数: {stats['timeouts']}")
    print(f"  失败率: {stats['failure_rate']:.2%}")
    print(f"  当前状态: {stats['state']}")
    print(f"  平均时间: {stats['avg_time']:.3f}秒")


if __name__ == "__main__":
    asyncio.run(circuit_breaker_example())
```

## 11.6 系统架构最佳实践

### 11.6.1 微服务架构

```python
"""
微服务架构实现
"""

from typing import Dict, Any, List
import asyncio


class MicroService:
    """微服务基类"""

    def __init__(self, name: str, message_bus: MessageBus):
        self.name = name
        self.message_bus = message_bus
        self.running = False
        self.health_check_interval = 30.0
        self.last_health_check = 0

    async def start(self):
        """启动服务"""
        self.running = True
        await self.on_start()

        # 启动健康检查
        asyncio.create_task(self._health_check_loop())

    async def stop(self):
        """停止服务"""
        self.running = False
        await self.on_stop()

    async def on_start(self):
        """启动回调"""
        pass

    async def on_stop(self):
        """停止回调"""
        pass

    async def handle_message(self, envelope: MessageEnvelope):
        """处理消息"""
        pass

    async def _health_check_loop(self):
        """健康检查循环"""
        while self.running:
            try:
                health_status = await self.check_health()
                await self.message_bus.send(
                    topic="service.health",
                    payload={
                        'service': self.name,
                        'status': health_status,
                        'timestamp': time.time()
                    },
                    sender=self.name
                )
                self.last_health_check = time.time()
            except Exception as e:
                print(f"Health check error for {self.name}: {e}")

            await asyncio.sleep(self.health_check_interval)

    async def check_health(self) -> dict:
        """检查服务健康状态"""
        return {
            'status': 'healthy',
            'uptime': time.time() - getattr(self, 'start_time', time.time())
        }


class ServiceRegistry:
    """服务注册中心"""

    def __init__(self):
        self.services: Dict[str, MicroService] = {}
        self.service_health: Dict[str, dict] = {}

    def register(self, service: MicroService):
        """注册服务"""
        self.services[service.name] = service

    def unregister(self, service_name: str):
        """注销服务"""
        if service_name in self.services:
            del self.services[service_name]

    def get_service(self, name: str) -> Optional[MicroService]:
        """获取服务"""
        return self.services.get(name)

    async def start_all(self):
        """启动所有服务"""
        for service in self.services.values():
            await service.start()

    async def stop_all(self):
        """停止所有服务"""
        for service in self.services.values():
            await service.stop()

    def update_health(self, service_name: str, health: dict):
        """更新服务健康状态"""
        self.service_health[service_name] = health

    def get_healthy_services(self) -> List[str]:
        """获取健康的服务列表"""
        healthy = []
        for name, health in self.service_health.items():
            if health.get('status') == 'healthy':
                # 检查最近更新时间
                if time.time() - health.get('timestamp', 0) < 60:
                    healthy.append(name)
        return healthy


# 微服务实现示例

class MarketDataService(MicroService):
    """市场数据服务"""

    def __init__(self, message_bus: MessageBus):
        super().__init__("market_data_service", message_bus)
        self.subscribers = set()

    async def on_start(self):
        """启动服务"""
        await super().on_start()

        # 订阅市场数据请求
        await self.message_bus.subscribe(
            "market_data.request",
            self
        )

        # 启动数据推送
        asyncio.create_task(self._push_market_data())

    async def handle_message(self, envelope: MessageEnvelope):
        """处理消息"""
        if envelope.topic == "market_data.request":
            msg = envelope.payload
            if msg.get('action') == 'subscribe':
                self.subscribers.add(envelope.sender)
            elif msg.get('action') == 'unsubscribe':
                self.subscribers.discard(envelope.sender)

    async def _push_market_data(self):
        """推送市场数据"""
        while self.running:
            # 生成市场数据
            market_data = {
                'BTCUSDT': {
                    'bid': 50000.0 + (time.time() % 1000),
                    'ask': 50001.0 + (time.time() % 1000),
                    'volume': 10.0
                },
                'ETHUSDT': {
                    'bid': 3000.0 + (time.time() % 500),
                    'ask': 3001.0 + (time.time() % 500),
                    'volume': 50.0
                }
            }

            # 发送给订阅者
            for subscriber in list(self.subscribers):
                await self.message_bus.send(
                    topic="market_data.update",
                    payload=market_data,
                    sender=self.name
                )

            await asyncio.sleep(0.1)  # 10 Hz

    async def check_health(self) -> dict:
        """检查健康状态"""
        base_health = await super().check_health()
        base_health.update({
            'subscribers': len(self.subscribers),
            'data_rate': 10.0  # Hz
        })
        return base_health


class OrderService(MicroService):
    """订单服务"""

    def __init__(self, message_bus: MessageBus):
        super().__init__("order_service", message_bus)
        self.orders = {}
        self.order_id_counter = 1

    async def on_start(self):
        """启动服务"""
        await super().on_start()

        # 订阅订单请求
        await self.message_bus.subscribe(
            "order.request",
            self
        )

    async def handle_message(self, envelope: MessageEnvelope):
        """处理消息"""
        if envelope.topic == "order.request":
            await self._handle_order_request(envelope)

    async def _handle_order_request(self, envelope: MessageEnvelope):
        """处理订单请求"""
        request = envelope.payload
        action = request.get('action')

        if action == 'submit':
            order_id = await self._submit_order(request)
            await self.message_bus.send(
                topic="order.response",
                payload={
                    'order_id': order_id,
                    'status': 'SUBMITTED'
                },
                sender=self.name,
                recipient=envelope.sender
            )
        elif action == 'cancel':
            success = await self._cancel_order(request.get('order_id'))
            await self.message_bus.send(
                topic="order.response",
                payload={
                    'order_id': request.get('order_id'),
                    'status': 'CANCELLED' if success else 'NOT_FOUND'
                },
                sender=self.name,
                recipient=envelope.sender
            )

    async def _submit_order(self, request: dict) -> str:
        """提交订单"""
        order_id = f"ORDER-{self.order_id_counter:06d}"
        self.order_id_counter += 1

        self.orders[order_id] = {
            'order_id': order_id,
            'symbol': request.get('symbol'),
            'side': request.get('side'),
            'quantity': request.get('quantity'),
            'price': request.get('price'),
            'status': 'SUBMITTED',
            'created_at': time.time()
        }

        # 模拟订单处理
        asyncio.create_task(self._process_order(order_id))

        return order_id

    async def _process_order(self, order_id: str):
        """处理订单"""
        await asyncio.sleep(0.5)  # 模拟延迟

        if order_id in self.orders:
            self.orders[order_id]['status'] = 'FILLED'
            self.orders[order_id]['filled_at'] = time.time()

    async def _cancel_order(self, order_id: str) -> bool:
        """取消订单"""
        if order_id in self.orders:
            self.orders[order_id]['status'] = 'CANCELLED'
            return True
        return False

    async def check_health(self) -> dict:
        """检查健康状态"""
        base_health = await super().check_health()
        base_health.update({
            'total_orders': len(self.orders),
            'pending_orders': sum(
                1 for o in self.orders.values()
                if o['status'] == 'SUBMITTED'
            )
        })
        return base_health


# 微服务架构示例

async def microservices_example():
    """微服务架构示例"""
    print("\n启动微服务架构示例...")

    # 创建消息总线
    message_bus = MessageBus()
    await message_bus.start()

    # 创建服务注册中心
    registry = ServiceRegistry()

    # 创建微服务
    market_service = MarketDataService(message_bus)
    order_service = OrderService(message_bus)

    # 注册服务
    registry.register(market_service)
    registry.register(order_service)

    # 启动所有服务
    await registry.start_all()

    # 运行一段时间
    await asyncio.sleep(10)

    # 获取健康服务列表
    healthy = registry.get_healthy_services()
    print(f"\n健康的服务: {healthy}")

    # 停止所有服务
    await registry.stop_all()
    await message_bus.stop()
    print("\n微服务架构示例完成")


if __name__ == "__main__":
    asyncio.run(microservices_example())
```

## 11.7 下一步

在本章中，我们学习了：

1. 消息总线的实现和使用
2. Actor模型的核心概念
3. 请求-响应和发布-订阅模式
4. 并发处理和线程池
5. 错误处理和恢复机制
6. 微服务架构设计

在下一章中，我们将学习：

1. 性能优化技巧
2. Cython和Rust集成
3. 内存管理和垃圾回收
4. 性能分析和监控
5. 高并发处理模式

## 11.8 总结

### 关键要点
1. 消息总线是组件间通信的核心
2. Actor模型提供了并发编程的抽象
3. 断路器模式提高了系统可靠性
4. 微服务架构提高了系统可扩展性

### 最佳实践
1. 使用消息总线解耦组件
2. 合理设计Actor层次结构
3. 实现完善的错误处理机制
4. 采用服务发现和健康检查
5. 监控系统性能和指标

## 11.9 参考资料

- [Akka Actor 模型文档](https://doc.akka.io/docs/akka/current/)
- [Erlang/OTP 设计原则](https://erlang.org/doc/design_principles/)
- [微服务架构模式](https://microservices.io/patterns/)
- [断路器模式详解](https://martinfowler.com/bliki/CircuitBreaker.html)
## 本章可运行代码入口（源码版）

- 入口脚本：`docs/tutorials_cn/code/chapter11/run_execution_algorithms.py`
- 关键示例：`docs/tutorials_cn/code/chapter11/execution_strategy.py`、`docs/tutorials_cn/code/chapter11/market_impact.py`

依赖策略说明：默认优先使用 Nautilus API；若场景受限则自动回退到纯 Python 逻辑，脚本保持可运行。

已验证命令：

```bash
uv run --python 3.13 python docs/tutorials_cn/code/chapter11/run_execution_algorithms.py
```
