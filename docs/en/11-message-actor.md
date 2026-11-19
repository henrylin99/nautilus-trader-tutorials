# Chapter 11: Message Passing and Actor Model

## Learning Objectives

After completing this chapter, you will:
- Understand NautilusTrader's message bus architecture
- Master the core concepts and implementation of the Actor model
- Learn inter-component communication mechanisms
- Implement high-performance concurrent processing
- Build scalable system architectures

## 11.1 Message Bus Architecture

### 11.1.1 Message Bus Fundamentals

```python
"""
NautilusTrader Message Bus Implementation
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
    """Message Type Enumeration"""
    COMMAND = "command"
    EVENT = "event"
    QUERY = "query"
    RESPONSE = "response"
    HEARTBEAT = "heartbeat"


@dataclass
class MessageEnvelope:
    """Message Envelope"""
    id: UUID4
    type: MessageType
    topic: str
    payload: Any
    sender: str
    recipient: Optional[str] = None
    timestamp: Optional[datetime] = None
    correlation_id: Optional[UUID4] = None
    priority: int = 0  # Priority, higher number means higher priority

    def __post_init__(self):
        if self.timestamp is None:
            self.timestamp = datetime.utcnow()


class IMessageHandler(ABC):
    """Message Handler Interface"""

    @abstractmethod
    def handle(self, envelope: MessageEnvelope) -> None:
        """Handle message"""
        pass

    @abstractmethod
    def get_subscriptions(self) -> List[str]:
        """Get subscribed topics"""
        pass


class MessageBus:
    """High-performance message bus implementation"""

    def __init__(self, max_queue_size: int = 10000):
        """
        Initialize message bus

        Parameters
        ----------
        max_queue_size : int
            Maximum queue size
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
        Subscribe to topic

        Parameters
        ----------
        topic : str
            Topic name
        handler : IMessageHandler
            Message handler
        """
        with self._lock:
            self._handlers[topic].append(handler)
            self._stats['handlers_registered'] += 1

    def unsubscribe(self, topic: str, handler: IMessageHandler):
        """
        Unsubscribe from topic

        Parameters
        ----------
        topic : str
            Topic name
        handler : IMessageHandler
            Message handler
        """
        with self._lock:
            if topic in self._handlers and handler in self._handlers[topic]:
                self._handlers[topic].remove(handler)
                self._stats['handlers_registered'] -= 1

    async def publish(self, envelope: MessageEnvelope) -> bool:
        """
        Publish message

        Parameters
        ----------
        envelope : MessageEnvelope
            Message envelope

        Returns
        -------
        bool
            Whether successfully published
        """
        try:
            if envelope.priority > 0:
                # High priority message
                await self._message_queue.put(envelope)
            else:
                # Regular message
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
        Convenient method to send message

        Parameters
        ----------
        topic : str
            Topic name
        payload : Any
            Message payload
        sender : str
            Sender
        recipient : str, optional
            Receiver
        message_type : MessageType
            Message type
        priority : int
            Priority

        Returns
        -------
        bool
            Whether successfully sent
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
        """Start message bus"""
        self._running = True
        self._worker_tasks = [
            asyncio.create_task(self._worker(f"worker-{i}"))
            for i in range(num_workers)
        ]

    async def stop(self):
        """Stop message bus"""
        self._running = False

        # Wait for all tasks to complete
        if self._worker_tasks:
            await asyncio.gather(*self._worker_tasks, return_exceptions=True)
            self._worker_tasks.clear()

    async def _worker(self, name: str):
        """Worker thread"""
        while self._running:
            try:
                # Get message (with timeout)
                envelope = await asyncio.wait_for(
                    self._message_queue.get(),
                    timeout=1.0
                )

                # Process message
                await self._process_message(envelope)
                self._message_queue.task_done()
                self._stats['messages_processed'] += 1

            except asyncio.TimeoutError:
                continue
            except Exception as e:
                print(f"Worker {name} error: {e}")
                self._stats['errors'] += 1

    async def _process_message(self, envelope: MessageEnvelope):
        """Process message"""
        handlers = self._handlers.get(envelope.topic, [])

        # If receiver specified, only call that receiver
        if envelope.recipient:
            handlers = [h for h in handlers if getattr(h, 'name', None) == envelope.recipient]

        # Concurrently call all handlers
        tasks = []
        for handler in handlers:
            task = asyncio.create_task(self._safe_handle(handler, envelope))
            tasks.append(task)

        if tasks:
            await asyncio.gather(*tasks, return_exceptions=True)

    async def _safe_handle(self, handler: IMessageHandler, envelope: MessageEnvelope):
        """Safely call handler"""
        try:
            # If it's a sync method, execute in thread pool
            if not asyncio.iscoroutinefunction(handler.handle):
                loop = asyncio.get_event_loop()
                await loop.run_in_executor(None, handler.handle, envelope)
            else:
                await handler.handle(envelope)
        except Exception as e:
            print(f"Handler error: {e}")
            self._stats['errors'] += 1

    def get_stats(self) -> Dict:
        """Get statistics"""
        with self._lock:
            return self._stats.copy()

    def get_queue_size(self) -> int:
        """Get queue size"""
        return self._message_queue.qsize()


# Usage example
class OrderEventHandler(IMessageHandler):
    """Order Event Handler"""

    def __init__(self, trader_id: TraderId):
        self.trader_id = trader_id
        self.order_count = 0

    def handle(self, envelope: MessageEnvelope) -> None:
        """Handle order event"""
        if envelope.type == MessageType.EVENT:
            self.order_count += 1
            print(f"[{self.trader_id.value}] Received order: {envelope.payload}")

    def get_subscriptions(self) -> List[str]:
        return ["orders.*"]


class MarketDataHandler(IMessageHandler):
    """Market Data Handler"""

    def __init__(self, name: str):
        self.name = name
        self.quote_count = 0

    def handle(self, envelope: MessageEnvelope) -> None:
        """Handle market data"""
        if envelope.topic.startswith("market."):
            self.quote_count += 1
            # Technical indicator calculations can be done here

    def get_subscriptions(self) -> List[str]:
        return ["market.*"]


# Message bus usage example
class MessageBusExample:
    """Message Bus Usage Example"""

    def __init__(self):
        self.message_bus = MessageBus(max_queue_size=5000)

    async def run_example(self):
        """Run example"""
        print("Starting message bus example...")

        # Create handlers
        order_handler = OrderEventHandler(TraderId("TRADER-001"))
        market_handler = MarketDataHandler("MARKET-001")

        # Register subscriptions
        self.message_bus.subscribe("orders.*", order_handler)
        self.message_bus.subscribe("market.BTCUSDT", market_handler)

        # Start message bus
        await self.message_bus.start(num_workers=2)

        # Send test messages
        await self._send_test_messages()

        # Wait for processing to complete
        await asyncio.sleep(2)

        # Output statistics
        stats = self.message_bus.get_stats()
        print("\nMessage bus statistics:")
        print(f"  Messages sent: {stats['messages_sent']}")
        print(f"  Messages processed: {stats['messages_processed']}")
        print(f"  Errors: {stats['errors']}")
        print(f"  Queue size: {self.message_bus.get_queue_size()}")

        # Stop message bus
        await self.message_bus.stop()

    async def _send_test_messages(self):
        """Send test messages"""
        # Send order events
        for i in range(100):
            await self.message_bus.send(
                topic="orders.new",
                payload={"order_id": f"ORDER-{i}", "symbol": "BTCUSDT", "quantity": 1.0},
                sender="ORDER-MANAGER",
                message_type=MessageType.EVENT
            )

        # Send market data
        for i in range(200):
            await self.message_bus.send(
                topic="market.BTCUSDT",
                payload={"bid": 50000.0 + i, "ask": 50001.0 + i, "volume": 10.0},
                sender="MARKET-FEED",
                message_type=MessageType.EVENT,
                priority=1  # Market data high priority
            )
```

### 11.1.2 Message Routing and Filtering

```python
"""
Message routing and filtering system
"""

import re
from typing import Pattern, Dict, List, Callable
from dataclasses import dataclass


@dataclass
class RoutingRule:
    """Routing rule"""
    pattern: str
    handler: IMessageHandler
    priority: int = 0
    condition: Optional[Callable[[MessageEnvelope], bool]] = None


class MessageRouter:
    """Message router"""

    def __init__(self):
        self._rules: List[RoutingRule] = []
        self._compiled_patterns: Dict[str, Pattern] = {}

    def add_rule(self, rule: RoutingRule):
        """
        Add routing rule

        Parameters
        ----------
        rule : RoutingRule
            Routing rule
        """
        # Compile regex pattern
        if rule.pattern not in self._compiled_patterns:
            # Convert wildcards to regex
            regex_pattern = rule.pattern.replace("*", ".*").replace("?", ".")
            self._compiled_patterns[rule.pattern] = re.compile(f"^{regex_pattern}$")

        self._rules.append(rule)
        # Sort by priority
        self._rules.sort(key=lambda r: r.priority, reverse=True)

    def remove_rule(self, pattern: str, handler: IMessageHandler):
        """Remove routing rule"""
        self._rules = [
            rule for rule in self._rules
            if rule.pattern != pattern or rule.handler != handler
        ]

    def route(self, envelope: MessageEnvelope) -> List[IMessageHandler]:
        """
        Route message to appropriate handlers

        Parameters
        ----------
        envelope : MessageEnvelope
            Message envelope

        Returns
        -------
        List[IMessageHandler]
            List of matching handlers
        """
        handlers = []

        for rule in self._rules:
            # Check if topic matches
            pattern = self._compiled_patterns[rule.pattern]
            if pattern.match(envelope.topic):
                # Check additional conditions
                if rule.condition is None or rule.condition(envelope):
                    handlers.append(rule.handler)

        return handlers


class MessageFilter:
    """Message filter"""

    @staticmethod
    def by_sender(sender: str) -> Callable[[MessageEnvelope], bool]:
        """Filter by sender"""
        return lambda envelope: envelope.sender == sender

    @staticmethod
    def by_type(message_type: MessageType) -> Callable[[MessageEnvelope], bool]:
        """Filter by message type"""
        return lambda envelope: envelope.type == message_type

    @staticmethod
    def by_priority(min_priority: int) -> Callable[[MessageEnvelope], bool]:
        """Filter by minimum priority"""
        return lambda envelope: envelope.priority >= min_priority

    @staticmethod
    def by_payload_field(field: str, value: Any) -> Callable[[MessageEnvelope], bool]:
        """Filter by payload field"""
        return lambda envelope: (
            isinstance(envelope.payload, dict) and
            envelope.payload.get(field) == value
        )

    @staticmethod
    def by_time_range(start_time: datetime, end_time: datetime) -> Callable[[MessageEnvelope], bool]:
        """Filter by time range"""
        return lambda envelope: (
            start_time <= envelope.timestamp <= end_time if envelope.timestamp else False
        )

    @staticmethod
    def composite(*filters: Callable) -> Callable[[MessageEnvelope], bool]:
        """Composite filter (AND)"""
        return lambda envelope: all(f(envelope) for f in filters)

    @staticmethod
    def any_of(*filters: Callable) -> Callable[[MessageEnvelope], bool]:
        """Any filter (OR)"""
        return lambda envelope: any(f(envelope) for f in filters)


# Enhanced message bus
class AdvancedMessageBus(MessageBus):
    """Enhanced message bus with routing and filtering"""

    def __init__(self, max_queue_size: int = 10000):
        super().__init__(max_queue_size)
        self._router = MessageRouter()
        self._filters: List[Callable] = []

    def add_routing_rule(self, pattern: str, handler: IMessageHandler,
                        priority: int = 0,
                        condition: Optional[Callable] = None):
        """Add routing rule"""
        rule = RoutingRule(
            pattern=pattern,
            handler=handler,
            priority=priority,
            condition=condition
        )
        self._router.add_rule(rule)

    def add_global_filter(self, filter_func: Callable):
        """Add global filter"""
        self._filters.append(filter_func)

    async def _process_message(self, envelope: MessageEnvelope):
        """Process message (with routing and filtering)"""
        # Apply global filters
        for filter_func in self._filters:
            if not filter_func(envelope):
                return  # Message filtered

        # Get routed handlers
        handlers = self._router.route(envelope)

        # If receiver specified, additional check
        if envelope.recipient:
            handlers = [h for h in handlers if getattr(h, 'name', None) == envelope.recipient]

        # Concurrently call all handlers
        tasks = []
        for handler in handlers:
            task = asyncio.create_task(self._safe_handle(handler, envelope))
            tasks.append(task)

        if tasks:
            await asyncio.gather(*tasks, return_exceptions=True)
```

## 11.2 Actor Model Implementation

### 11.2.1 Actor Basic Framework

```python
"""
Actor Model Implementation
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
    """Actor state"""
    STARTING = "starting"
    RUNNING = "running"
    STOPPING = "stopping"
    STOPPED = "stopped"
    CRASHED = "crashed"


@dataclass
class ActorMessage:
    """Actor message"""
    sender: str
    message: Any
    correlation_id: Optional[UUID4] = None
    timestamp: float = None

    def __post_init__(self):
        if self.timestamp is None:
            self.timestamp = time.time()


class IActor(ABC):
    """Actor interface"""

    @abstractmethod
    async def receive(self, message: ActorMessage) -> Any:
        """Receive and process message"""
        pass

    @abstractmethod
    def get_name(self) -> str:
        """Get actor name"""
        pass


class ActorRef:
    """Actor reference"""

    def __init__(self, name: str, actor_system: 'ActorSystem'):
        self.name = name
        self._actor_system = actor_system

    async def tell(self, message: Any, sender: Optional[str] = None):
        """Asynchronously send message (fire-and-forget)"""
        await self._actor_system.send(self.name, message, sender)

    async def ask(self, message: Any, timeout: float = 5.0) -> Any:
        """Asynchronously send message and wait for response"""
        return await self._actor_system.request(self.name, message, timeout)

    def __repr__(self):
        return f"ActorRef({self.name})"


class Actor(IActor):
    """Basic actor implementation"""

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
        """Start actor"""
        self.state = ActorState.STARTING
        self._actor_system = actor_system
        self._supervisor = supervisor

        # Call startup hook
        await self.on_start()

        # Start message processing loop
        self._task = asyncio.create_task(self._run())
        self.state = ActorState.RUNNING

    async def stop(self):
        """Stop actor"""
        if self.state != ActorState.RUNNING:
            return

        self.state = ActorState.STOPPING

        # Stop all child actors
        for child_name in list(self._children):
            await self._actor_system.stop_actor(child_name)

        # Call stop hook
        await self.on_stop()

        # Cancel task
        if self._task:
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass

        self.state = ActorState.STOPPED

    async def _run(self):
        """Actor main loop"""
        while self.state == ActorState.RUNNING:
            try:
                # Get message (with timeout to check state)
                message = await asyncio.wait_for(
                    self.mailbox.get(),
                    timeout=1.0
                )

                # Process message
                await self.receive(message)

            except asyncio.TimeoutError:
                continue
            except Exception as e:
                # Handle exception
                await self.handle_error(e)

    async def receive(self, message: ActorMessage) -> Any:
        """Receive and process message (to be implemented by subclasses)"""
        # Subclasses should override this method
        pass

    async def spawn(self, actor_class, name: str, *args, **kwargs) -> ActorRef:
        """Create child actor"""
        actor = actor_class(name, *args, **kwargs)
        await self._actor_system.register_actor(actor, parent=self)
        self._children.add(name)
        return ActorRef(name, self._actor_system)

    async def send_to_child(self, child_name: str, message: Any):
        """Send message to child actor"""
        await self._actor_system.send(child_name, message, self.name)

    async def handle_error(self, error: Exception):
        """Handle error"""
        self.state = ActorState.CRASHED

        # Notify supervisor
        if self._supervisor:
            await self._supervisor.handle_child_error(self.name, error)

    # Lifecycle hooks
    async def on_start(self):
        """Called on start"""
        pass

    async def on_stop(self):
        """Called on stop"""
        pass

    async def on_child_terminated(self, child_name: str):
        """Called when child actor terminates"""
        self._children.discard(child_name)


class SupervisorStrategy:
    """Supervisor strategy"""

    async def handle_failure(self, supervisor: Actor, child_name: str,
                           error: Exception) -> bool:
        """Handle failure"""
        raise NotImplementedError


class RestartStrategy(SupervisorStrategy):
    """Restart strategy"""

    def __init__(self, max_retries: int = 3, backoff: float = 1.0):
        self.max_retries = max_retries
        self.backoff = backoff
        self._retry_counts: Dict[str, int] = {}

    async def handle_failure(self, supervisor: Actor, child_name: str,
                           error: Exception) -> bool:
        """Handle failure"""
        retry_count = self._retry_counts.get(child_name, 0)

        if retry_count < self.max_retries:
            self._retry_counts[child_name] = retry_count + 1

            # Wait some time before restart
            await asyncio.sleep(self.backoff * (2 ** retry_count))

            # Restart child actor
            await supervisor._actor_system.restart_actor(child_name)
            return True
        else:
            # Exceeded max retries, stop child actor
            await supervisor._actor_system.stop_actor(child_name)
            self._retry_counts[child_name] = 0
            return False


class OneForOneStrategy(RestartStrategy):
    """One-for-one restart strategy"""

    async def handle_failure(self, supervisor: Actor, child_name: str,
                           error: Exception) -> bool:
        """Only restart the failed child actor"""
        return await super().handle_failure(supervisor, child_name, error)


class AllForOneStrategy(RestartStrategy):
    """All-for-one restart strategy"""

    async def handle_failure(self, supervisor: Actor, child_name: str,
                           error: Exception) -> bool:
        """Restart all child actors"""
        # Restart all child actors
        for child in list(supervisor._children):
            await super().handle_failure(supervisor, child, Exception("Cascading restart"))
        return True
```

### 11.2.2 Actor System

```python
"""
Actor System Implementation
"""

import asyncio
from typing import Dict, Any, Optional, List
from collections import defaultdict


class ActorSystem:
    """Actor system"""

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
        """Start actor system"""
        self._running = True

    async def stop(self):
        """Stop actor system"""
        self._running = False

        # Stop all actors
        for actor_name in list(self._actors.keys()):
            await self.stop_actor(actor_name)

    async def register_actor(self, actor: Actor, parent: Optional[Actor] = None) -> ActorRef:
        """Register actor"""
        self._actors[actor.name] = actor
        self._stats['actors_created'] += 1
        return ActorRef(actor.name, self)

    async def start_actor(self, actor_name: str, supervisor: Optional[Actor] = None):
        """Start actor"""
        if actor_name not in self._actors:
            raise ValueError(f"Actor {actor_name} not registered")

        actor = self._actors[actor_name]
        await actor.start(self, supervisor)
        self._stats['actors_started'] += 1

    async def stop_actor(self, actor_name: str):
        """Stop actor"""
        if actor_name in self._actors:
            actor = self._actors[actor_name]
            await actor.stop()
            del self._actors[actor_name]

    async def restart_actor(self, actor_name: str):
        """Restart actor"""
        if actor_name in self._actors:
            actor = self._actors[actor_name]
            await actor.stop()
            await actor.start(self, actor._supervisor)

    async def send(self, recipient: str, message: Any, sender: Optional[str] = None):
        """Send message"""
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
        """Send request and wait for response"""
        correlation_id = UUID4()
        future = asyncio.Future()
        self._pending_requests[correlation_id] = future

        # Send request
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

        # Wait for response
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
        """Respond to request"""
        if correlation_id in self._pending_requests:
            future = self._pending_requests[correlation_id]
            if not future.done():
                future.set_result(response)

    def get_actor_ref(self, name: str) -> ActorRef:
        """Get actor reference"""
        if name not in self._actors:
            raise ValueError(f"Actor {name} not found")
        return ActorRef(name, self)

    def get_stats(self) -> Dict:
        """Get statistics"""
        self._stats.update({
            'active_actors': len(self._actors),
            'pending_requests': len(self._pending_requests)
        })
        return self._stats.copy()


# Specific actor implementation examples

class OrderManagerActor(Actor):
    """Order Manager Actor"""

    def __init__(self, name: str):
        super().__init__(name)
        self.orders = {}
        self.next_order_id = 1

    async def receive(self, message: ActorMessage) -> Any:
        """Handle order-related messages"""
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
                # Handle request-response pattern
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
        """Handle submit order"""
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

        # Simulate sending to exchange
        await asyncio.sleep(0.1)
        order['status'] = 'SUBMITTED'

        self.log.info(f"Order submitted: {order_id}")
        return order_id

    async def _handle_cancel_order(self, msg: dict) -> bool:
        """Handle cancel order"""
        order_id = msg.get('order_id')
        if order_id in self.orders:
            self.orders[order_id]['status'] = 'CANCELLED'
            self.log.info(f"Order cancelled: {order_id}")
            return True
        return False

    async def _handle_get_order(self, msg: dict) -> Optional[dict]:
        """Handle get order"""
        order_id = msg.get('order_id')
        return self.orders.get(order_id)

    async def _process_request(self, payload: dict) -> Any:
        """Process request"""
        if payload.get('action') == 'get_all_orders':
            return list(self.orders.values())
        elif payload.get('action') == 'get_position':
            # Calculate position
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
    """Market Data Actor"""

    def __init__(self, name: str):
        super().__init__(name)
        self.subscribers = set()
        self.running = False

    async def on_start(self):
        """Start pushing market data when started"""
        await super().on_start()
        self.running = True
        # Start market data push task
        asyncio.create_task(self._push_market_data())

    async def on_stop(self):
        """Stop pushing when stopped"""
        await super().on_stop()
        self.running = False

    async def receive(self, message: ActorMessage) -> Any:
        """Handle subscription requests"""
        msg = message.message

        if isinstance(msg, dict):
            if msg.get('type') == 'subscribe':
                self.subscribers.add(message.sender)
                self.log.info(f"Added subscriber: {message.sender}")
            elif msg.get('type') == 'unsubscribe':
                self.subscribers.discard(message.sender)
                self.log.info(f"Removed subscriber: {message.sender}")

    async def _push_market_data(self):
        """Push market data"""
        while self.running:
            # Generate simulated market data
            market_data = {
                'symbol': 'BTCUSDT',
                'bid': 50000.0 + (time.time() % 1000),
                'ask': 50001.0 + (time.time() % 1000),
                'volume': 10.0,
                'timestamp': time.time()
            }

            # Send to all subscribers
            for subscriber in list(self.subscribers):
                try:
                    await self._actor_system.send(subscriber, {
                        'type': 'market_data',
                        'data': market_data
                    }, self.name)
                except:
                    # Subscriber may have stopped
                    self.subscribers.discard(subscriber)

            await asyncio.sleep(0.1)  # 10 Hz

    def get_name(self) -> str:
        return self.name


class StrategyActor(Actor):
    """Strategy Actor"""

    def __init__(self, name: str, strategy_config: dict):
        super().__init__(name)
        self.config = strategy_config
        self.position = 0
        self.last_price = None

    async def on_start(self):
        """Subscribe to market data when started"""
        await super().on_start()

        # Subscribe to market data
        market_data_ref = self._actor_system.get_actor_ref("market_data")
        await market_data_ref.tell({
            'type': 'subscribe'
        }, self.name)

        # Get order manager reference
        self.order_manager = self._actor_system.get_actor_ref("order_manager")

    async def receive(self, message: ActorMessage) -> Any:
        """Handle market data and order updates"""
        msg = message.message

        if isinstance(msg, dict):
            if msg.get('type') == 'market_data':
                await self._handle_market_data(msg.get('data'))
            elif msg.get('type') == 'order_update':
                await self._handle_order_update(msg.get('data'))

    async def _handle_market_data(self, data: dict):
        """Handle market data"""
        price = (data['bid'] + data['ask']) / 2

        if self.last_price is None:
            self.last_price = price
            return

        # Simple moving average strategy
        price_change = price - self.last_price

        if price_change > 100 and self.position <= 0:
            # Price up, buy
            await self.order_manager.tell({
                'type': 'submit_order',
                'symbol': 'BTCUSDT',
                'side': 'BUY',
                'quantity': 1.0,
                'price': data['ask']
            }, self.name)
            self.position = 1

        elif price_change < -100 and self.position >= 0:
            # Price down, sell
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
        """Handle order update"""
        if data.get('status') == 'FILLED':
            self.log.info(f"Order filled: {data.get('order_id')}")


# Actor system usage example
async def actor_system_example():
    """Actor system usage example"""
    print("Starting actor system example...")

    # Create actor system
    system = ActorSystem("TRADING_SYSTEM")
    await system.start()

    # Create actors
    market_data_actor = MarketDataActor("market_data")
    order_manager_actor = OrderManagerActor("order_manager")
    strategy_actor = StrategyActor("strategy", {"symbol": "BTCUSDT"})

    # Register actors
    await system.register_actor(market_data_actor)
    await system.register_actor(order_manager_actor)
    await system.register_actor(strategy_actor)

    # Start actors
    await system.start_actor("market_data")
    await system.start_actor("order_manager")
    await system.start_actor("strategy")

    # Run for some time
    await asyncio.sleep(10)

    # Get statistics
    stats = system.get_stats()
    print("\nActor system statistics:")
    print(f"  Active actors: {stats['active_actors']}")
    print(f"  Messages sent: {stats['messages_sent']}")
    print(f"  Messages processed: {stats['messages_processed']}")

    # Stop system
    await system.stop()
    print("\nActor system stopped")


if __name__ == "__main__":
    asyncio.run(actor_system_example())
```

## 11.3 Inter-component Communication

### 11.3.1 Request-Response Pattern

```python
"""
Request-Response Communication Pattern
"""

import asyncio
from typing import Dict, Any, Optional, Callable, Awaitable
from dataclasses import dataclass
import time


@dataclass
class Request:
    """Request"""
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
    """Response"""
    request_id: str
    result: Any = None
    error: Optional[str] = None
    timestamp: float = None

    def __post_init__(self):
        if self.timestamp is None:
            self.timestamp = time.time()


class RPCRegistry:
    """RPC registry"""

    def __init__(self):
        self._methods: Dict[str, Callable] = {}

    def register(self, name: str, method: Callable):
        """Register RPC method"""
        self._methods[name] = method

    def unregister(self, name: str):
        """Unregister RPC method"""
        if name in self._methods:
            del self._methods[name]

    async def call(self, request: Request) -> Response:
        """Call RPC method"""
        if request.method not in self._methods:
            return Response(
                request_id=request.id,
                error=f"Method '{request.method}' not found"
            )

        try:
            method = self._methods[request.method]

            # Support async and sync methods
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
    """RPC client"""

    def __init__(self, message_bus: MessageBus, service_name: str):
        self.message_bus = message_bus
        self.service_name = service_name
        self._pending_requests: Dict[str, asyncio.Future] = {}

    async def call(self, method: str, params: Dict[str, Any],
                   timeout: float = 5.0) -> Any:
        """Call remote method"""
        request_id = str(uuid.uuid4())

        # Create request
        request = Request(
            id=request_id,
            method=method,
            params=params,
            timeout=timeout
        )

        # Create future to wait for response
        future = asyncio.Future()
        self._pending_requests[request_id] = future

        # Send request
        await self.message_bus.send(
            topic=f"rpc.{self.service_name}",
            payload=request,
            sender="rpc_client"
        )

        # Wait for response
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
        """Handle response"""
        if response.request_id in self._pending_requests:
            future = self._pending_requests[response.request_id]
            if not future.done():
                future.set_result(response)


class RPCServer:
    """RPC server"""

    def __init__(self, message_bus: MessageBus, service_name: str):
        self.message_bus = message_bus
        self.service_name = service_name
        self.registry = RPCRegistry()

    def register_method(self, name: str, method: Callable):
        """Register method"""
        self.registry.register(name, method)

    async def start(self):
        """Start RPC server"""
        # Subscribe to RPC requests
        await self.message_bus.subscribe(
            f"rpc.{self.service_name}",
            self
        )

    async def handle(self, envelope: MessageEnvelope):
        """Handle RPC request"""
        if isinstance(envelope.payload, Request):
            request = envelope.payload

            # Call method
            response = await self.registry.call(request)

            # Send response
            await self.message_bus.send(
                topic=f"rpc_response.{envelope.sender}",
                payload=response,
                sender=self.service_name
            )


# Usage example

class TradingService:
    """Trading service"""

    def __init__(self):
        self.positions = {}
        self.orders = {}

    async def submit_order(self, symbol: str, side: str,
                          quantity: float, price: float) -> str:
        """Submit order"""
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

        # Simulate order processing
        await asyncio.sleep(0.1)
        order['status'] = 'FILLED'

        # Update position
        self._update_position(symbol, side, quantity)

        return order_id

    def _update_position(self, symbol: str, side: str, quantity: float):
        """Update position"""
        if symbol not in self.positions:
            self.positions[symbol] = 0

        if side == 'BUY':
            self.positions[symbol] += quantity
        else:
            self.positions[symbol] -= quantity

    def get_position(self, symbol: str) -> float:
        """Get position"""
        return self.positions.get(symbol, 0)

    def get_order(self, order_id: str) -> Optional[dict]:
        """Get order"""
        return self.orders.get(order_id)


async def rpc_example():
    """RPC usage example"""
    print("Starting RPC example...")

    # Create message bus
    message_bus = MessageBus()
    await message_bus.start()

    # Create trading service
    trading_service = TradingService()

    # Create RPC server
    rpc_server = RPCServer(message_bus, "trading")
    rpc_server.register_method("submit_order", trading_service.submit_order)
    rpc_server.register_method("get_position", trading_service.get_position)
    rpc_server.register_method("get_order", trading_service.get_order)
    await rpc_server.start()

    # Create RPC client
    rpc_client = RPCClient(message_bus, "trading")

    # Test RPC calls
    print("\nTesting RPC calls...")

    # Submit order
    order_id = await rpc_client.call("submit_order", {
        'symbol': 'BTCUSDT',
        'side': 'BUY',
        'quantity': 1.0,
        'price': 50000.0
    })
    print(f"Order submitted successfully: {order_id}")

    # Query order
    order = await rpc_client.call("get_order", {
        'order_id': order_id
    })
    print(f"Order status: {order['status']}")

    # Query position
    position = await rpc_client.call("get_position", {
        'symbol': 'BTCUSDT'
    })
    print(f"BTC position: {position}")

    # Stop
    await message_bus.stop()
    print("\nRPC example completed")
```

## 11.4 Concurrent Processing Patterns

### 11.4.1 Worker Pool Pattern

```python
"""
Worker Pool Concurrent Pattern
"""

import asyncio
from typing import List, Callable, Any, Optional
from dataclasses import dataclass
import time


@dataclass
class Task:
    """Task"""
    id: str
    func: Callable
    args: tuple
    kwargs: dict
    priority: int = 0
    timeout: Optional[float] = None
    retry_count: int = 0
    max_retries: int = 0


class Worker:
    """Worker thread"""

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
        """Start worker thread"""
        self.running = False
        asyncio.create_task(self._worker_loop())

    async def stop(self):
        """Stop worker thread"""
        self.running = False

    async def _worker_loop(self):
        """Worker loop"""
        while self.running:
            try:
                # Get task
                task = await asyncio.wait_for(
                    self.task_queue.get(),
                    timeout=1.0
                )

                self.current_task = task
                start_time = time.time()

                # Execute task
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
                        # Sync function executes in thread pool
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

                    # Task successful
                    self.stats['tasks_completed'] += 1
                    await self._handle_task_success(task, result)

                except Exception as e:
                    # Task failed
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
        """Handle successful task"""
        # Can add success callback here
        pass

    async def _handle_task_failure(self, task: Task, error: Exception):
        """Handle failed task"""
        # Retry logic
        if task.retry_count < task.max_retries:
            task.retry_count += 1
            await self.task_queue.put(task)
            print(f"Task {task.id} failed, retrying ({task.retry_count}/{task.max_retries})")
        else:
            print(f"Task {task.id} failed after {task.max_retries} retries: {error}")


class ThreadPool:
    """Thread pool"""

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
        """Start thread pool"""
        self.running = True

        # Create worker threads
        for i in range(self.num_workers):
            worker = Worker(f"worker-{i}", self.task_queue)
            self.workers.append(worker)
            await worker.start()

    async def stop(self):
        """Stop thread pool"""
        self.running = False

        # Stop all worker threads
        for worker in self.workers:
            await worker.stop()

    async def submit(self, func: Callable, *args, priority: int = 0,
                     timeout: Optional[float] = None,
                     max_retries: int = 0, **kwargs) -> str:
        """Submit task"""
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
        """Submit batch of tasks"""
        task_ids = []
        for task_config in tasks:
            task_id = await self.submit(**task_config)
            task_ids.append(task_id)
        return task_ids

    def get_stats(self) -> dict:
        """Get statistics"""
        self.stats['queue_size'] = self.task_queue.qsize()

        # Aggregate worker thread statistics
        total_completed = sum(w.stats['tasks_completed'] for w in self.workers)
        total_failed = sum(w.stats['tasks_failed'] for w in self.workers)
        total_time = sum(w.stats['total_time'] for w in self.workers)

        self.stats.update({
            'tasks_completed': total_completed,
            'tasks_failed': total_failed,
            'avg_task_time': total_time / max(total_completed, 1)
        })

        return self.stats.copy()


# Usage examples

async def process_market_data(symbol: str, data: dict) -> dict:
    """Process market data"""
    # Simulate time-consuming operation
    await asyncio.sleep(0.1)

    # Calculate technical indicators
    price = data['price']
    volume = data['volume']

    result = {
        'symbol': symbol,
        'price': price,
        'volume': volume,
        'vwap': price,  # Simplified
        'rsi': 50,      # Simplified
        'timestamp': time.time()
    }

    return result


async def execute_order(order: dict) -> dict:
    """Execute order"""
    # Simulate order execution
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
    """Calculate risk indicators (CPU-intensive)"""
    # Simulate CPU-intensive calculation
    total = 0
    for i in range(1000000):
        total += i * 0.000001

    risk_metrics = {
        'total_exposure': sum(pos['quantity'] * pos['price'] for pos in positions.values()),
        'var': total * 0.01,  # Simplified
        'beta': 1.5,
        'sharpe': 2.0
    }

    return risk_metrics


async def thread_pool_example():
    """Thread pool usage example"""
    print("\nStarting thread pool example...")

    # Create thread pool
    thread_pool = ThreadPool(num_workers=4, max_queue_size=100)
    await thread_pool.start()

    # Submit market data processing tasks
    market_tasks = []
    for i in range(20):
        task_id = await thread_pool.submit(
            process_market_data,
            f"BTCUSDT-{i}",
            {'price': 50000.0 + i, 'volume': 10.0},
            priority=1
        )
        market_tasks.append(task_id)

    # Submit order execution tasks
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

    # Submit risk calculation tasks
    positions = {
        'BTCUSDT': {'quantity': 10, 'price': 50000},
        'ETHUSDT': {'quantity': 100, 'price': 3000}
    }
    risk_task_id = await thread_pool.submit(
        calculate_risk,
        positions,
        priority=0
    )

    # Wait for all tasks to complete
    print("Waiting for tasks to complete...")
    await thread_pool.task_queue.join()

    # Output statistics
    stats = thread_pool.get_stats()
    print("\nThread pool statistics:")
    print(f"  Submitted tasks: {stats['tasks_submitted']}")
    print(f"  Completed tasks: {stats['tasks_completed']}")
    print(f"  Failed tasks: {stats['tasks_failed']}")
    print(f"  Average task time: {stats['avg_task_time']:.3f}s")

    # Stop thread pool
    await thread_pool.stop()
    print("\nThread pool example completed")


if __name__ == "__main__":
    asyncio.run(thread_pool_example())
```

## 11.5 Error Handling and Recovery

### 11.5.1 Circuit Breaker Pattern

```python
"""
Circuit Breaker Pattern Implementation
"""

import asyncio
import time
from enum import Enum
from typing import Callable, Any, Optional


class CircuitState(Enum):
    """Circuit breaker state"""
    CLOSED = "closed"      # Normal state
    OPEN = "open"          # Circuit is open
    HALF_OPEN = "half_open"  # Half-open state


@dataclass
class CircuitBreakerConfig:
    """Circuit breaker configuration"""
    failure_threshold: int = 5      # Failure threshold
    recovery_timeout: float = 60.0  # Recovery timeout
    expected_exception: type = Exception  # Expected exception
    success_threshold: int = 3      # Success threshold (half-open state)


class CircuitBreaker:
    """Circuit breaker implementation"""

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
        """Call function through circuit breaker"""
        self.stats['calls'] += 1

        # Check circuit breaker state
        if self.state == CircuitState.OPEN:
            if self._should_attempt_reset():
                self.state = CircuitState.HALF_OPEN
                self.success_count = 0
            else:
                raise Exception("Circuit breaker is OPEN")

        # Execute function
        start_time = time.time()
        try:
            if timeout:
                result = await asyncio.wait_for(
                    func(*args, **kwargs),
                    timeout=timeout
                )
            else:
                result = await func(*args, **kwargs)

            # Success
            self._on_success()
            self.stats['successes'] += 1
            return result

        except asyncio.TimeoutError:
            # Timeout
            self._on_failure()
            self.stats['timeouts'] += 1
            raise

        except self.config.expected_exception as e:
            # Expected exception
            self._on_failure()
            self.stats['failures'] += 1
            raise

        except Exception as e:
            # Unexpected exception, also counts as failure
            self._on_failure()
            self.stats['failures'] += 1
            raise

        finally:
            # Record call time
            elapsed = time.time() - start_time
            if 'total_time' not in self.stats:
                self.stats['total_time'] = 0
            self.stats['total_time'] += elapsed

    def _should_attempt_reset(self) -> bool:
        """Whether to attempt reset"""
        return (
            self.last_failure_time and
            time.time() - self.last_failure_time >= self.config.recovery_timeout
        )

    def _on_success(self):
        """Handle success"""
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.config.success_threshold:
                self.state = CircuitState.CLOSED
                self.failure_count = 0
        else:
            self.failure_count = 0

    def _on_failure(self):
        """Handle failure"""
        self.failure_count += 1
        self.last_failure_time = time.time()

        if self.failure_count >= self.config.failure_threshold:
            self.state = CircuitState.OPEN

    def get_state(self) -> CircuitState:
        """Get current state"""
        return self.state

    def get_stats(self) -> dict:
        """Get statistics"""
        stats = self.stats.copy()
        stats['failure_rate'] = (
            stats['failures'] / max(stats['calls'], 1)
        )
        stats['avg_time'] = (
            stats.get('total_time', 0) / max(stats['successes'], 1)
        )
        stats['state'] = self.state.value
        return stats


# Usage example

async def unreliable_api_call(endpoint: str) -> dict:
    """Simulate unreliable API call"""
    # Simulate 50% failure rate
    if time.time() % 2 < 1:
        raise Exception("API Error")

    # Simulate network latency
    await asyncio.sleep(0.1)

    return {
        'endpoint': endpoint,
        'data': f"Response from {endpoint}",
        'timestamp': time.time()
    }


async def circuit_breaker_example():
    """Circuit breaker usage example"""
    print("\nStarting circuit breaker example...")

    # Create circuit breaker
    config = CircuitBreakerConfig(
        failure_threshold=3,
        recovery_timeout=2.0,
        success_threshold=2
    )
    breaker = CircuitBreaker(config)

    # Test calls
    for i in range(20):
        try:
            result = await breaker.call(
                unreliable_api_call,
                f"/api/v1/data/{i}",
                timeout=0.5
            )
            print(f"Call {i}: Success - {result['data']}")
        except Exception as e:
            print(f"Call {i}: Failed - {str(e)}")

        await asyncio.sleep(0.5)

    # Output statistics
    stats = breaker.get_stats()
    print("\nCircuit breaker statistics:")
    print(f"  Total calls: {stats['calls']}")
    print(f"  Successes: {stats['successes']}")
    print(f"  Failures: {stats['failures']}")
    print(f"  Timeouts: {stats['timeouts']}")
    print(f"  Failure rate: {stats['failure_rate']:.2%}")
    print(f"  Current state: {stats['state']}")
    print(f"  Average time: {stats['avg_time']:.3f}s")


if __name__ == "__main__":
    asyncio.run(circuit_breaker_example())
```

## 11.6 System Architecture Best Practices

### 11.6.1 Microservices Architecture

```python
"""
Microservices Architecture Implementation
"""

from typing import Dict, Any, List
import asyncio


class MicroService:
    """Microservice base class"""

    def __init__(self, name: str, message_bus: MessageBus):
        self.name = name
        self.message_bus = message_bus
        self.running = False
        self.health_check_interval = 30.0
        self.last_health_check = 0

    async def start(self):
        """Start service"""
        self.running = True
        await self.on_start()

        # Start health check
        asyncio.create_task(self._health_check_loop())

    async def stop(self):
        """Stop service"""
        self.running = False
        await self.on_stop()

    async def on_start(self):
        """Start callback"""
        pass

    async def on_stop(self):
        """Stop callback"""
        pass

    async def handle_message(self, envelope: MessageEnvelope):
        """Handle message"""
        pass

    async def _health_check_loop(self):
        """Health check loop"""
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
        """Check service health status"""
        return {
            'status': 'healthy',
            'uptime': time.time() - getattr(self, 'start_time', time.time())
        }


class ServiceRegistry:
    """Service registry"""

    def __init__(self):
        self.services: Dict[str, MicroService] = {}
        self.service_health: Dict[str, dict] = {}

    def register(self, service: MicroService):
        """Register service"""
        self.services[service.name] = service

    def unregister(self, service_name: str):
        """Unregister service"""
        if service_name in self.services:
            del self.services[service_name]

    def get_service(self, name: str) -> Optional[MicroService]:
        """Get service"""
        return self.services.get(name)

    async def start_all(self):
        """Start all services"""
        for service in self.services.values():
            await service.start()

    async def stop_all(self):
        """Stop all services"""
        for service in self.services.values():
            await service.stop()

    def update_health(self, service_name: str, health: dict):
        """Update service health status"""
        self.service_health[service_name] = health

    def get_healthy_services(self) -> List[str]:
        """Get list of healthy services"""
        healthy = []
        for name, health in self.service_health.items():
            if health.get('status') == 'healthy':
                # Check recent update time
                if time.time() - health.get('timestamp', 0) < 60:
                    healthy.append(name)
        return healthy


# Microservice implementation examples

class MarketDataService(MicroService):
    """Market data service"""

    def __init__(self, message_bus: MessageBus):
        super().__init__("market_data_service", message_bus)
        self.subscribers = set()

    async def on_start(self):
        """Start service"""
        await super().on_start()

        # Subscribe to market data requests
        await self.message_bus.subscribe(
            "market_data.request",
            self
        )

        # Start data push
        asyncio.create_task(self._push_market_data())

    async def handle_message(self, envelope: MessageEnvelope):
        """Handle message"""
        if envelope.topic == "market_data.request":
            msg = envelope.payload
            if msg.get('action') == 'subscribe':
                self.subscribers.add(envelope.sender)
            elif msg.get('action') == 'unsubscribe':
                self.subscribers.discard(envelope.sender)

    async def _push_market_data(self):
        """Push market data"""
        while self.running:
            # Generate market data
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

            # Send to subscribers
            for subscriber in list(self.subscribers):
                await self.message_bus.send(
                    topic="market_data.update",
                    payload=market_data,
                    sender=self.name
                )

            await asyncio.sleep(0.1)  # 10 Hz

    async def check_health(self) -> dict:
        """Check health status"""
        base_health = await super().check_health()
        base_health.update({
            'subscribers': len(self.subscribers),
            'data_rate': 10.0  # Hz
        })
        return base_health


class OrderService(MicroService):
    """Order service"""

    def __init__(self, message_bus: MessageBus):
        super().__init__("order_service", message_bus)
        self.orders = {}
        self.order_id_counter = 1

    async def on_start(self):
        """Start service"""
        await super().on_start()

        # Subscribe to order requests
        await self.message_bus.subscribe(
            "order.request",
            self
        )

    async def handle_message(self, envelope: MessageEnvelope):
        """Handle message"""
        if envelope.topic == "order.request":
            await self._handle_order_request(envelope)

    async def _handle_order_request(self, envelope: MessageEnvelope):
        """Handle order request"""
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
        """Submit order"""
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

        # Simulate order processing
        asyncio.create_task(self._process_order(order_id))

        return order_id

    async def _process_order(self, order_id: str):
        """Process order"""
        await asyncio.sleep(0.5)  # Simulate delay

        if order_id in self.orders:
            self.orders[order_id]['status'] = 'FILLED'
            self.orders[order_id]['filled_at'] = time.time()

    async def _cancel_order(self, order_id: str) -> bool:
        """Cancel order"""
        if order_id in self.orders:
            self.orders[order_id]['status'] = 'CANCELLED'
            return True
        return False

    async def check_health(self) -> dict:
        """Check health status"""
        base_health = await super().check_health()
        base_health.update({
            'total_orders': len(self.orders),
            'pending_orders': sum(
                1 for o in self.orders.values()
                if o['status'] == 'SUBMITTED'
            )
        })
        return base_health


# Microservices architecture example

async def microservices_example():
    """Microservices architecture example"""
    print("\nStarting microservices architecture example...")

    # Create message bus
    message_bus = MessageBus()
    await message_bus.start()

    # Create service registry
    registry = ServiceRegistry()

    # Create microservices
    market_service = MarketDataService(message_bus)
    order_service = OrderService(message_bus)

    # Register services
    registry.register(market_service)
    registry.register(order_service)

    # Start all services
    await registry.start_all()

    # Run for some time
    await asyncio.sleep(10)

    # Get healthy services list
    healthy = registry.get_healthy_services()
    print(f"\nHealthy services: {healthy}")

    # Stop all services
    await registry.stop_all()
    await message_bus.stop()
    print("\nMicroservices architecture example completed")


if __name__ == "__main__":
    asyncio.run(microservices_example())
```

## 11.7 Next Steps

In this chapter, we learned:

1. Implementation and usage of message bus
2. Core concepts of the Actor model
3. Request-response and publish-subscribe patterns
4. Concurrent processing and thread pools
5. Error handling and recovery mechanisms
6. Microservices architecture design

In the next chapter, we will learn:

1. Performance optimization techniques
2. Cython and Rust integration
3. Memory management and garbage collection
4. Performance analysis and monitoring
5. High-concurrency processing patterns

## 11.8 Summary

### Key Points
1. Message bus is the core of inter-component communication
2. Actor model provides abstractions for concurrent programming
3. Circuit breaker pattern improves system reliability
4. Microservices architecture enhances system scalability

### Best Practices
1. Use message bus to decouple components
2. Design Actor hierarchies appropriately
3. Implement comprehensive error handling mechanisms
4. Adopt service discovery and health checks
5. Monitor system performance and metrics

## 11.9 References

- [Akka Actor Model Documentation](https://doc.akka.io/docs/akka/current/)
- [Erlang/OTP Design Principles](https://erlang.org/doc/design_principles/)
- [Microservices Architecture Patterns](https://microservices.io/patterns/)
- [Circuit Breaker Pattern Details](https://martinfowler.com/bliki/CircuitBreaker.html)

## Runnable Code Entry (Source Build)

- Entry script: docs/tutorials_cn/code/chapter11/run_execution_algorithms.py
- Key examples: docs/tutorials_cn/code/chapter11/execution_strategy.py, docs/tutorials_cn/code/chapter11/market_impact.py

Dependency strategy: Prefer Nautilus APIs when available; automatically fallback to pure-Python logic when optional capabilities are not present.

Verified command:

```bash
uv run --python 3.13 python docs/tutorials_cn/code/chapter11/run_execution_algorithms.py
```
