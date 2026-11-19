# Chapter 6: Time Management and Event Handling

## Learning Objectives

By completing this chapter, you will:
- Understand NautilusTrader's time management mechanism
- Master the use of timers
- Learn event-driven programming patterns
- Understand message bus and event routing
- Implement complex scheduling logic

## 6.1 Time Management Architecture

### 6.1.1 Clock Types

NautilusTrader provides multiple clock implementations to suit different needs:

```python
"""
Clock types and use cases
"""

from nautilus_trader.core.datetime import dt_to_unix_nanos
from nautilus_trader.test_kit.stubs.events import TestEventStubs
from datetime import datetime, timezone


# 1. LiveClock - Real-time clock (for live trading)
from nautilus_trader.core.datetime import nanos_to_unix_nanos

class LiveClockExample:
    """Real-time clock example"""

    def __init__(self):
        from nautilus_trader.core.datetime import get_atomic_clock
        self.clock = get_atomic_clock()

    def get_current_time(self):
        """Get current time (nanoseconds)"""
        return self.clock.timestamp_ns()

    def wait_until(self, target_time_ns: int):
        """Wait until specified time"""
        import time
        current_time = self.clock.timestamp_ns()
        if current_time < target_time_ns:
            sleep_seconds = (target_time_ns - current_time) / 1_000_000_000
            time.sleep(sleep_seconds)

# 2. TestClock - Test clock (for backtesting)
class TestClockExample:
    """Test clock example"""

    def __init__(self):
        from nautilus_trader.test_kit.stubs.events import TestClockStub
        # Create test clock
        self.clock = TestClockStub()

        # Set initial time
        self.clock.set_time(dt_to_unix_nanos(datetime(2024, 1, 1)))

    def advance_time(self, seconds: int):
        """Advance time"""
        self.clock.advance_time(seconds * 1_000_000_000)

    def schedule_event(self, delay_ns: int, callback):
        """Schedule event"""
        self.clock.set_time_ns_callback(
            target_time=self.clock.timestamp_ns() + delay_ns,
            callback=callback
        )


# 3. MonotonicClock - Monotonic clock
class MonotonicClockExample:
    """Monotonic clock example"""

    def __init__(self):
        from nautilus_trader.core.datetime import MonotonicClock
        self.clock = MonotonicClock()

    def measure_duration(self, func):
        """Measure function execution time"""
        start = self.clock.timestamp_ns()
        func()
        end = self.clock.timestamp_ns()
        return end - start
```

### 6.1.2 Time Formats and Conversions

```python
"""
Time handling utilities
"""

from datetime import datetime, timedelta, timezone
from decimal import Decimal
import pytz


class TimeUtils:
    """Time handling utility class"""

    @staticmethod
    def to_utc_naive(dt: datetime) -> datetime:
        """Convert to UTC time (timezone naive)"""
        if dt.tzinfo is None:
            return dt
        return dt.astimezone(timezone.utc).replace(tzinfo=None)

    @staticmethod
    def from_utc_naive(dt: datetime, tz_str: str = "UTC") -> datetime:
        """Convert from UTC time to specified timezone"""
        tz = pytz.timezone(tz_str)
        return tz.localize(dt)

    @staticmethod
    def to_nanos(dt: datetime) -> int:
        """Convert to nanosecond timestamp"""
        return int(dt.timestamp() * 1_000_000_000)

    @staticmethod
    def from_nanos(nanos: int) -> datetime:
        """Convert from nanosecond timestamp"""
        return datetime.fromtimestamp(nanos / 1_000_000_000, tz=timezone.utc)

    @staticmethod
    def format_duration(nanos: int) -> str:
        """Format time duration"""
        seconds = nanos / 1_000_000_000

        if seconds < 1:
            return f"{nanos / 1_000_000:.2f}ms"
        elif seconds < 60:
            return f"{seconds:.2f}s"
        elif seconds < 3600:
            minutes = seconds / 60
            return f"{minutes:.2f}min"
        else:
            hours = seconds / 3600
            return f"{hours:.2f}h"

    @staticmethod
    def get_trading_session_time(dt: datetime, timezone_str: str = "UTC") -> tuple:
        """
        Get trading session time

        Returns:
            tuple: (session_start, session_end, is_trading_time)
        """
        tz = pytz.timezone(timezone_str)
        local_dt = dt.astimezone(tz)

        # Standard trading hours: 9:30 - 16:00
        session_start = local_dt.replace(
            hour=9, minute=30, second=0, microsecond=0
        )
        session_end = local_dt.replace(
            hour=16, minute=0, second=0, microsecond=0
        )

        is_trading = session_start <= local_dt <= session_end

        return session_start, session_end, is_trading


# Usage example
def main():
    """Time handling example"""

    utils = TimeUtils()

    # Current time
    now = datetime.now()
    print(f"Local time: {now}")

    # Convert to UTC
    utc_time = utils.to_utc_naive(now)
    print(f"UTC time: {utc_time}")

    # Convert to nanoseconds
    nanos = utils.to_nanos(utc_time)
    print(f"Nanosecond timestamp: {nanos}")

    # Convert back
    dt_back = utils.from_nanos(nanos)
    print(f"Converted back: {dt_back}")

    # Trading time check
    session_start, session_end, is_trading = utils.get_trading_session_time(now)
    print(f"Trading session: {session_start} - {session_end}")
    print(f"Is currently trading time: {is_trading}")


if __name__ == "__main__":
    main()
```

## 6.2 Timer System

### 6.2.1 Basic Timers

```python
"""
Timer usage examples
"""

import asyncio
from datetime import datetime, timedelta
from decimal import Decimal

from nautilus_trader.core.datetime import dt_to_unix_nanos
from nautilus_trader.test_kit.stubs.events import TestClockStub
from nautilus_trader.trading.strategy import Strategy
from nautilus_trader.model.data import Bar
from nautilus_trader.model.events import OrderFilled


class TimerStrategy(Strategy):
    """Strategy example using timers"""

    def __init__(self):
        """Initialize strategy"""
        super().__init__()

        # Timer ID management
        self.timers = {
            'daily_reset': None,
            'hourly_update': None,
            'minute_check': None,
            'exit_check': None,
        }

        # State tracking
        self.daily_high = Decimal('0')
        self.daily_low = Decimal('999999')
        self.hourly_volume = Decimal('0')
        self.last_bar_time = None
        self.positions_held_time = {}

    def on_start(self):
        """Set up timers when strategy starts"""
        self.log.info("Timer strategy starting")

        # Daily reset timer (UTC 00:00)
        self._schedule_daily_reset()

        # Hourly update
        self._schedule_hourly_update()

        # Minute check
        self._schedule_minute_check()

        # Exit condition check every 30 seconds
        self._schedule_exit_check()

    def _schedule_daily_reset(self):
        """Schedule daily reset timer"""
        # Calculate next UTC 00:00
        now = self.clock.timestamp_ns()
        now_dt = datetime.fromtimestamp(now / 1_000_000_000, tz=timezone.utc)

        next_midnight = (now_dt + timedelta(days=1)).replace(
            hour=0, minute=0, second=0, microsecond=0
        )

        delay_ns = dt_to_unix_nanos(next_midnight) - now

        self.timers['daily_reset'] = self.clock.set_timer(
            name="daily_reset",
            interval_ns=delay_ns,
            callback=self._on_daily_reset,
        )

        self.log.info(f"Daily reset timer set, next execution: {next_midnight}")

    def _on_daily_reset(self):
        """Daily reset callback"""
        self.log.info("Executing daily reset")

        # Reset statistics
        self.daily_high = Decimal('0')
        self.daily_low = Decimal('999999')
        self.hourly_volume = Decimal('0')

        # Clear position time records
        self.positions_held_time.clear()

        # Reschedule next reset
        self._schedule_daily_reset()

    def _schedule_hourly_update(self):
        """Schedule hourly update"""
        # Calculate delay to next hour
        now = self.clock.timestamp_ns()
        now_dt = datetime.fromtimestamp(now / 1_000_000_000, tz=timezone.utc)

        next_hour = (now_dt + timedelta(hours=1)).replace(
            minute=0, second=0, microsecond=0
        )

        delay_ns = dt_to_unix_nanos(next_hour) - now

        self.timers['hourly_update'] = self.clock.set_timer(
            name="hourly_update",
            interval_ns=delay_ns,
            callback=self._on_hourly_update,
        )

    def _on_hourly_update(self):
        """Hourly update callback"""
        self.log.info(f"Hourly update - Current hour volume: {self.hourly_volume}")

        # Reset hourly volume
        self.hourly_volume = Decimal('0')

        # Reschedule
        self._schedule_hourly_update()

        # Other hourly tasks can be executed here
        self._analyze_hourly_performance()

    def _schedule_minute_check(self):
        """Schedule minute check"""
        self.timers['minute_check'] = self.clock.set_timer(
            name="minute_check",
            interval_ns=60 * 1_000_000_000,  # 1 minute
            callback=self._on_minute_check,
        )

    def _on_minute_check(self):
        """Minute check callback"""
        current_time = self.clock.timestamp_ns()

        # Check for long-unupdated data
        if self.last_bar_time:
            time_diff = current_time - self.last_bar_time
            if time_diff > 2 * 60 * 1_000_000_000:  # Over 2 minutes
                self.log.warning("Data update anomaly - Possible connection issue")

    def _schedule_exit_check(self):
        """Schedule exit check"""
        self.timers['exit_check'] = self.clock.set_timer(
            name="exit_check",
            interval_ns=30 * 1_000_000_000,  # 30 seconds
            callback=self._on_exit_check,
        )

    def _on_exit_check(self):
        """Exit condition check"""
        # Check position holding time
        current_time = self.clock.timestamp_ns()

        for position_id, entry_time in list(self.positions_held_time.items()):
            hold_duration = current_time - entry_time
            hold_hours = hold_duration / (1_000_000_000 * 3600)

            # Force close positions held over 24 hours
            if hold_hours > 24:
                self.log.warning(
                    f"Position held too long - Force close: {position_id}, "
                    f"Hold time: {hold_hours:.1f} hours"
                )
                self._force_close_position(position_id)

    def on_bar(self, bar: Bar):
        """Process bar data"""
        # Update last bar time
        self.last_bar_time = self.clock.timestamp_ns()

        # Update daily high/low
        if bar.close > self.daily_high:
            self.daily_high = bar.close

        if bar.close < self.daily_low:
            self.daily_low = bar.close

        # Accumulate volume
        self.hourly_volume += bar.volume

        # Check trading signals
        self._check_trading_signal(bar)

    def on_order_filled(self, event: OrderFilled):
        """Handle order filled"""
        # Record position time
        self.positions_held_time[event.position_id] = self.clock.timestamp_ns()

    def _check_trading_signal(self, bar: Bar):
        """Check trading signals"""
        # Strategy based on high/low breakouts
        if bar.close > self.daily_high * Decimal('1.001'):  # Break high by 0.1%
            self._enter_long(bar)
        elif bar.close < self.daily_low * Decimal('0.999'):  # Break low by 0.1%
            self._enter_short(bar)

    def _analyze_hourly_performance(self):
        """Analyze hourly performance"""
        # Hourly performance analysis can be implemented here
        pass

    def _force_close_position(self, position_id):
        """Force close position"""
        # Implement force close logic
        pass

    def on_stop(self):
        """Clean up timers when strategy stops"""
        self.log.info("Stopping timer strategy")

        # Cancel all timers
        for timer_name, timer_id in self.timers.items():
            if timer_id:
                self.clock.cancel_timer(timer_id)
                self.log.info(f"Cancelled timer: {timer_name}")
```

### 6.2.2 Advanced Timer Patterns

```python
"""
Advanced timer patterns
"""

from typing import Callable, Dict, List
from enum import Enum
import heapq


class TimerPriority(Enum):
    """Timer priority"""
    CRITICAL = 1    # Critical tasks (e.g., risk checks)
    HIGH = 2        # High priority (e.g., trading signals)
    NORMAL = 3      # Normal priority (e.g., statistics updates)
    LOW = 4         # Low priority (e.g., cleanup tasks)


class AdvancedTimerManager:
    """Advanced timer manager"""

    def __init__(self, clock):
        """Initialize timer manager"""
        self.clock = clock
        self.timers = {}  # timer_id -> timer_info
        self.timer_heap = []  # Priority queue

    def schedule(
        self,
        name: str,
        callback: Callable,
        delay_ns: int,
        interval_ns: int = None,
        priority: TimerPriority = TimerPriority.NORMAL,
        metadata: Dict = None,
    ) -> str:
        """
        Schedule timer

        Parameters
        ----------
        name : str
            Timer name
        callback : Callable
            Callback function
        delay_ns : int
            Delay time (nanoseconds)
        interval_ns : int
            Repeat interval (nanoseconds), None for one-time
        priority : TimerPriority
            Priority
        metadata : Dict
            Metadata

        Returns
        -------
        str
            Timer ID
        """
        current_time = self.clock.timestamp_ns()

        timer_info = {
            'id': f"{name}_{current_time}",
            'name': name,
            'callback': callback,
            'delay_ns': delay_ns,
            'interval_ns': interval_ns,
            'priority': priority,
            'metadata': metadata or {},
            'next_run': current_time + delay_ns,
            'run_count': 0,
            'active': True,
        }

        self.timers[timer_info['id']] = timer_info

        # Add to priority queue
        heapq.heappush(
            self.timer_heap,
            (timer_info['next_run'], priority.value, timer_info['id'])
        )

        return timer_info['id']

    def cancel(self, timer_id: str):
        """Cancel timer"""
        if timer_id in self.timers:
            self.timers[timer_id]['active'] = False
            del self.timers[timer_id]

    def process_pending(self):
        """Process pending timers"""
        current_time = self.clock.timestamp_ns()

        while self.timer_heap:
            next_time, priority, timer_id = self.timer_heap[0]

            if next_time > current_time:
                break

            # Pop timer
            heapq.heappop(self.timer_heap)

            if timer_id not in self.timers:
                continue

            timer = self.timers[timer_id]
            if not timer['active']:
                continue

            # Execute callback
            try:
                timer['callback']()
                timer['run_count'] += 1
            except Exception as e:
                self.log.error(f"Timer execution error: {timer_id}, Error: {e}")

            # If recurring timer, schedule next run
            if timer['interval_ns']:
                timer['next_run'] = current_time + timer['interval_ns']
                heapq.heappush(
                    self.timer_heap,
                    (timer['next_run'], timer['priority'].value, timer_id)
                )
            else:
                # One-time timer, delete
                del self.timers[timer_id]

    def get_stats(self) -> Dict:
        """Get statistics"""
        total_timers = len(self.timers)

        # Statistics by priority
        priority_stats = {}
        for priority in TimerPriority:
            count = sum(
                1 for t in self.timers.values()
                if t['priority'] == priority and t['active']
            )
            priority_stats[priority.name] = count

        # Next run time
        next_run_time = None
        if self.timer_heap:
            next_run_time = self.timer_heap[0][0]

        return {
            'total_active': total_timers,
            'by_priority': priority_stats,
            'next_run_ns': next_run_time,
        }


class ScheduledTaskStrategy(Strategy):
    """Strategy using advanced timers"""

    def __init__(self):
        """Initialize strategy"""
        super().__init__()

        # Task scheduler
        self.scheduler = AdvancedTimerManager(self.clock)

        # Statistics
        self.stats = {
            'signals_detected': 0,
            'orders_submitted': 0,
            'risk_checks': 0,
            'data_updates': 0,
        }

    def on_start(self):
        """Strategy start"""
        self.log.info("Scheduled task strategy starting")

        # Schedule various tasks

        # 1. Data analysis task (every 5 minutes)
        self.scheduler.schedule(
            name="data_analysis",
            callback=self._analyze_market_data,
            delay_ns=5 * 60 * 1_000_000_000,  # Start after 5 minutes
            interval_ns=5 * 60 * 1_000_000_000,  # Every 5 minutes
            priority=TimerPriority.NORMAL,
        )

        # 2. Risk check task (every minute)
        self.scheduler.schedule(
            name="risk_check",
            callback=self._check_risk,
            delay_ns=60 * 1_000_000_000,  # Start after 1 minute
            interval_ns=60 * 1_000_000_000,  # Every minute
            priority=TimerPriority.CRITICAL,
        )

        # 3. Signal scan task (every 10 seconds)
        self.scheduler.schedule(
            name="signal_scan",
            callback=self._scan_signals,
            delay_ns=10 * 1_000_000_000,  # Start after 10 seconds
            interval_ns=10 * 1_000_000_000,  # Every 10 seconds
            priority=TimerPriority.HIGH,
        )

        # 4. Performance statistics task (every hour)
        self.scheduler.schedule(
            name="performance_stats",
            callback=self._update_performance_stats,
            delay_ns=3600 * 1_000_000_000,  # Start after 1 hour
            interval_ns=3600 * 1_000_000_000,  # Every hour
            priority=TimerPriority.LOW,
        )

        # 5. Cleanup task (daily at midnight)
        self.scheduler.schedule(
            name="daily_cleanup",
            callback=self._daily_cleanup,
            delay_ns=self._get_time_to_next_midnight(),
            interval_ns=24 * 3600 * 1_000_000_000,  # Daily
            priority=TimerPriority.LOW,
        )

        # Start timer processing loop
        self.scheduler.schedule(
            name="timer_processor",
            callback=self._process_timers,
            delay_ns=100_000_000,  # Start after 100ms
            interval_ns=100_000_000,  # Every 100ms
            priority=TimerPriority.CRITICAL,
        )

    def _process_timers(self):
        """Process pending timers"""
        self.scheduler.process_pending()

    def _analyze_market_data(self):
        """Analyze market data"""
        self.stats['data_updates'] += 1
        self.log.debug(f"Executing market data analysis - Update count: {self.stats['data_updates']}")

        # Get latest data
        # ... Analysis logic ...

    def _check_risk(self):
        """Check risk"""
        self.stats['risk_checks'] += 1
        self.log.debug(f"Executing risk check - Check count: {self.stats['risk_checks']}")

        # Check position risk
        # ... Risk logic ...

    def _scan_signals(self):
        """Scan trading signals"""
        self.stats['signals_detected'] += 1
        self.log.debug(f"Scanning trading signals - Scan count: {self.stats['signals_detected']}")

        # Scan market signals
        # ... Signal logic ...

    def _update_performance_stats(self):
        """Update performance statistics"""
        self.log.info(
            f"Performance stats - "
            f"Signals: {self.stats['signals_detected']}, "
            f"Orders: {self.stats['orders_submitted']}, "
            f"Risk checks: {self.stats['risk_checks']}"
        )

    def _daily_cleanup(self):
        """Daily cleanup"""
        self.log.info("Executing daily cleanup")

        # Clean expired data
        # Reset statistics
        # ... Cleanup logic ...

    def _get_time_to_next_midnight(self) -> int:
        """Get time to next midnight"""
        now = datetime.now()
        next_midnight = (now + timedelta(days=1)).replace(
            hour=0, minute=0, second=0, microsecond=0
        )
        return int((next_midnight - now).total_seconds() * 1_000_000_000)

    def on_bar(self, bar: Bar):
        """Process bar data"""
        # Timers will handle automatically, only process real-time data here
        pass

    def on_stop(self):
        """Strategy stop"""
        self.log.info("Scheduled task strategy stopping")

        # Show final statistics
        stats = self.scheduler.get_stats()
        self.log.info(f"Timer statistics: {stats}")
        self.log.info(f"Task statistics: {self.stats}")
```

## 6.3 Event-Driven Architecture

### 6.3.1 Event Types and Inheritance

```python
"""
Event system architecture
"""

from abc import ABC
from typing import Any, Dict, Optional
from datetime import datetime

from nautilus_trader.core.message import Event
from nautilus_trader.model.identifiers import TraderId, StrategyId


class CustomEvent(Event, ABC):
    """Custom event base class"""

    def __init__(
        self,
        trader_id: TraderId,
        strategy_id: Optional[StrategyId] = None,
        event_id: Optional[str] = None,
        ts_init: Optional[int] = None,
        ts_event: Optional[int] = None,
        **kwargs: Any,
    ):
        """
        Initialize custom event

        Parameters
        ----------
        trader_id : TraderId
            Trader ID
        strategy_id : StrategyId, optional
            Strategy ID
        event_id : str, optional
            Event ID
        ts_init : int, optional
            Initialization timestamp
        ts_event : int, optional
            Event timestamp
        **kwargs : Any
            Other parameters
        """
        super().__init__(
            trader_id=trader_id,
            strategy_id=strategy_id,
            event_id=event_id,
            ts_init=ts_init or self.clock.timestamp_ns(),
            ts_event=ts_event or self.clock.timestamp_ns(),
            **kwargs
        )

        self.metadata = kwargs


class SignalEvent(CustomEvent):
    """Trading signal event"""

    def __init__(
        self,
        trader_id: TraderId,
        strategy_id: StrategyId,
        instrument_id: str,
        signal_type: str,  # BUY, SELL, HOLD
        confidence: float,  # 0-1
        price: Optional[float] = None,
        quantity: Optional[float] = None,
        metadata: Optional[Dict] = None,
        **kwargs: Any,
    ):
        """
        Initialize signal event

        Parameters
        ----------
        instrument_id : str
            Instrument ID
        signal_type : str
            Signal type
        confidence : float
            Confidence level
        price : float, optional
            Suggested price
        quantity : float, optional
            Suggested quantity
        metadata : Dict, optional
            Additional metadata
        """
        super().__init__(
            trader_id=trader_id,
            strategy_id=strategy_id,
            instrument_id=instrument_id,
            signal_type=signal_type,
            confidence=confidence,
            price=price,
            quantity=quantity,
            **(metadata or {})
        )

    def to_dict(self) -> Dict:
        """Convert to dictionary"""
        return {
            'instrument_id': self.instrument_id,
            'signal_type': self.signal_type,
            'confidence': self.confidence,
            'price': self.price,
            'quantity': self.quantity,
            'timestamp': self.ts_event,
        }


class RiskEvent(CustomEvent):
    """Risk event"""

    def __init__(
        self,
        trader_id: TraderId,
        risk_type: str,  # POSITION_LIMIT, DRAWDOWN, CORRELATION
        risk_level: str,  # LOW, MEDIUM, HIGH, CRITICAL
        message: str,
        affected_positions: Optional[list] = None,
        **kwargs: Any,
    ):
        """
        Initialize risk event

        Parameters
        ----------
        risk_type : str
            Risk type
        risk_level : str
            Risk level
        message : str
            Risk description
        affected_positions : list, optional
            Affected positions
        """
        super().__init__(
            trader_id=trader_id,
            risk_type=risk_type,
            risk_level=risk_level,
            message=message,
            affected_positions=affected_positions or [],
            **kwargs
        )

    def requires_immediate_action(self) -> bool:
        """Whether immediate action is required"""
        return self.risk_level in ['HIGH', 'CRITICAL']


class PerformanceEvent(CustomEvent):
    """Performance statistics event"""

    def __init__(
        self,
        trader_id: TraderId,
        strategy_id: StrategyId,
        metric_type: str,  # PNL, DRAWDOWN, SHARPE, WIN_RATE
        metric_value: float,
        period: str,  # DAILY, WEEKLY, MONTHLY
        benchmark_value: Optional[float] = None,
        **kwargs: Any,
    ):
        """
        Initialize performance event

        Parameters
        ----------
        metric_type : str
            Metric type
        metric_value : float
            Metric value
        period : str
            Statistics period
        benchmark_value : float, optional
            Benchmark value
        """
        super().__init__(
            trader_id=trader_id,
            strategy_id=strategy_id,
            metric_type=metric_type,
            metric_value=metric_value,
            period=period,
            benchmark_value=benchmark_value,
            **kwargs
        )

    def is_outperforming(self) -> bool:
        """Whether outperforming benchmark"""
        if self.benchmark_value is None:
            return None
        return self.metric_value > self.benchmark_value


class SystemEvent(CustomEvent):
    """System event"""

    def __init__(
        self,
        trader_id: TraderId,
        system_type: str,  # CONNECTIVITY, DATA, EXECUTION
        severity: str,  # INFO, WARNING, ERROR, CRITICAL
        message: str,
        component: Optional[str] = None,
        **kwargs: Any,
    ):
        """
        Initialize system event

        Parameters
        ----------
        system_type : str
            System type
        severity : str
            Severity level
        message : str
            Event message
        component : str, optional
            Component name
        """
        super().__init__(
            trader_id=trader_id,
            system_type=system_type,
            severity=severity,
            message=message,
            component=component,
            **kwargs
        )

    def requires_alert(self) -> bool:
        """Whether alert is required"""
        return self.severity in ['ERROR', 'CRITICAL']
```

### 6.3.2 Event Handlers and Dispatchers

```python
"""
Event handling system
"""

from typing import Callable, Dict, List, Type
from collections import defaultdict
from concurrent.futures import ThreadPoolExecutor
import threading
import queue
import time


class EventBus:
    """Event bus"""

    def __init__(self, max_workers: int = 4):
        """
        Initialize event bus

        Parameters
        ----------
        max_workers : int
            Maximum number of worker threads
        """
        self.handlers = defaultdict(list)
        self.executor = ThreadPoolExecutor(max_workers=max_workers)
        self.running = False
        self.event_queue = queue.Queue()
        self.processing_thread = None

    def subscribe(
        self,
        event_type: Type[Event],
        handler: Callable[[Event], None],
        priority: int = 0
    ):
        """
        Subscribe to events

        Parameters
        ----------
        event_type : Type[Event]
            Event type
        handler : Callable
            Handler function
        priority : int
            Priority (higher number = higher priority)
        """
        self.handlers[event_type].append((handler, priority))
        # Sort by priority
        self.handlers[event_type].sort(key=lambda x: x[1], reverse=True)

    def unsubscribe(self, event_type: Type[Event], handler: Callable):
        """Unsubscribe"""
        handlers = self.handlers.get(event_type, [])
        self.handlers[event_type] = [
            (h, p) for h, p in handlers if h != handler
        ]

    def publish(self, event: Event):
        """
        Publish event (asynchronous)

        Parameters
        ----------
        event : Event
            Event to publish
        """
        self.event_queue.put(event)

    def publish_sync(self, event: Event):
        """
        Publish event (synchronous)

        Parameters
        ----------
        event : Event
            Event to publish
        """
        self._process_event(event)

    def start(self):
        """Start event processing"""
        if not self.running:
            self.running = True
            self.processing_thread = threading.Thread(
                target=self._process_loop,
                daemon=True
            )
            self.processing_thread.start()

    def stop(self):
        """Stop event processing"""
        self.running = False
        if self.processing_thread:
            self.processing_thread.join()

    def _process_loop(self):
        """Event processing loop"""
        while self.running:
            try:
                # Get event from queue
                event = self.event_queue.get(timeout=1.0)
                self._process_event(event)
                self.event_queue.task_done()
            except queue.Empty:
                continue
            except Exception as e:
                print(f"Event processing error: {e}")

    def _process_event(self, event: Event):
        """Process single event"""
        event_type = type(event)
        handlers = self.handlers.get(event_type, [])

        # Process all subscribers in parallel
        futures = []
        for handler, priority in handlers:
            future = self.executor.submit(self._safe_handle, handler, event)
            futures.append(future)

        # Wait for all processing to complete
        for future in futures:
            try:
                future.result(timeout=5.0)
            except Exception as e:
                print(f"Handler execution error: {e}")

    def _safe_handle(self, handler: Callable, event: Event):
        """Safely call handler"""
        try:
            handler(event)
        except Exception as e:
            print(f"Event handling exception: {e}")

    def get_stats(self) -> Dict:
        """Get statistics"""
        return {
            'queue_size': self.event_queue.qsize(),
            'subscriptions': {
                str(event_type): len(handlers)
                for event_type, handlers in self.handlers.items()
            },
            'running': self.running,
        }


class EventDrivenStrategy(Strategy):
    """Event-driven strategy base class"""

    def __init__(self):
        """Initialize strategy"""
        super().__init__()

        # Event bus
        self.event_bus = EventBus()

        # Register event handlers
        self._register_event_handlers()

        # Statistics
        self.event_stats = defaultdict(int)

    def _register_event_handlers(self):
        """Register event handlers"""
        # Register custom event handlers
        self.event_bus.subscribe(SignalEvent, self._handle_signal_event)
        self.event_bus.subscribe(RiskEvent, self._handle_risk_event)
        self.event_bus.subscribe(PerformanceEvent, self._handle_performance_event)
        self.event_bus.subscribe(SystemEvent, self._handle_system_event)

    def on_start(self):
        """Strategy start"""
        self.log.info("Event-driven strategy starting")
        self.event_bus.start()

        # Publish startup event
        self._publish_system_event("STARTUP", "INFO", "Strategy started")

    def on_stop(self):
        """Strategy stop"""
        self.log.info("Event-driven strategy stopping")
        self.event_bus.stop()

        # Publish stop event
        self._publish_system_event("SHUTDOWN", "INFO", "Strategy stopped")

        # Show event statistics
        self.log.info(f"Event statistics: {dict(self.event_stats)}")

    def _handle_signal_event(self, event: SignalEvent):
        """Handle signal event"""
        self.event_stats['signals'] += 1

        self.log.info(
            f"Received signal event: {event.signal_type} "
            f"(confidence: {event.confidence:.2f})"
        )

        # Execute trades based on signals
        if event.signal_type == 'BUY':
            self._execute_buy_signal(event)
        elif event.signal_type == 'SELL':
            self._execute_sell_signal(event)

    def _handle_risk_event(self, event: RiskEvent):
        """Handle risk event"""
        self.event_stats['risk_events'] += 1

        self.log.warning(
            f"Risk event: {event.risk_type} - {event.message}"
        )

        # Take action based on risk level
        if event.requires_immediate_action():
            self._handle_critical_risk(event)
        else:
            self._handle_normal_risk(event)

    def _handle_performance_event(self, event: PerformanceEvent):
        """Handle performance event"""
        self.event_stats['performance_events'] += 1

        self.log.info(
            f"Performance update: {event.metric_type} = {event.metric_value} "
            f"({event.period})"
        )

        # Check performance
        if event.is_outperforming() is False:
            self.log.warning("Strategy performance poor, consider adjusting parameters")

    def _handle_system_event(self, event: SystemEvent):
        """Handle system event"""
        self.event_stats['system_events'] += 1

        if event.requires_alert():
            self.log.error(f"System alert: {event.message}")
        else:
            self.log.info(f"System event: {event.message}")

    def _execute_buy_signal(self, event: SignalEvent):
        """Execute buy signal"""
        # Implement buy logic
        pass

    def _execute_sell_signal(self, event: SignalEvent):
        """Execute sell signal"""
        # Implement sell logic
        pass

    def _handle_critical_risk(self, event: RiskEvent):
        """Handle critical risk"""
        # Take immediate action, such as stopping trading or closing positions
        pass

    def _handle_normal_risk(self, event: RiskEvent):
        """Handle normal risk"""
        # Log and monitor risk development
        pass

    def _publish_signal_event(self, signal_type: str, confidence: float):
        """Publish signal event"""
        event = SignalEvent(
            trader_id=self.trader_id,
            strategy_id=self.id,
            instrument_id="BTCUSDT.BINANCE",
            signal_type=signal_type,
            confidence=confidence,
        )
        self.event_bus.publish(event)

    def _publish_risk_event(self, risk_type: str, risk_level: str, message: str):
        """Publish risk event"""
        event = RiskEvent(
            trader_id=self.trader_id,
            risk_type=risk_type,
            risk_level=risk_level,
            message=message,
        )
        self.event_bus.publish(event)

    def _publish_performance_event(self, metric_type: str, value: float, period: str):
        """Publish performance event"""
        event = PerformanceEvent(
            trader_id=self.trader_id,
            strategy_id=self.id,
            metric_type=metric_type,
            metric_value=value,
            period=period,
        )
        self.event_bus.publish(event)

    def _publish_system_event(self, system_type: str, severity: str, message: str):
        """Publish system event"""
        event = SystemEvent(
            trader_id=self.trader_id,
            system_type=system_type,
            severity=severity,
            message=message,
        )
        self.event_bus.publish(event)


# Usage example
def main():
    """Main function example"""

    # Create event bus
    event_bus = EventBus()

    # Create strategy
    strategy = EventDrivenStrategy()

    # Simulate events
    signal = SignalEvent(
        trader_id=TraderId("TRADER-001"),
        strategy_id=StrategyId("STRATEGY-001"),
        instrument_id="BTCUSDT.BINANCE",
        signal_type="BUY",
        confidence=0.85,
    )

    # Publish event
    event_bus.publish(signal)

    # Run for a while then stop
    time.sleep(1)
    event_bus.stop()


if __name__ == "__main__":
    main()
```

## 6.4 Practical Application Examples

### 6.4.1 Integrated Timer and Event System

```python
"""
Integrated timer and event system example
"""

from nautilus_trader.trading.strategy import Strategy
from nautilus_trader.model.data import Bar
from nautilus_trader.model.events import OrderFilled
from nautilus_trader.model.objects import Money
from decimal import Decimal
from datetime import datetime, timedelta
import asyncio


class ComprehensiveStrategy(Strategy):
    """Strategy using integrated timers and events"""

    def __init__(self):
        """Initialize strategy"""
        super().__init__()

        # Timer management
        self.timers = {
            'market_analysis': None,
            'risk_monitor': None,
            'performance_update': None,
            'news_check': None,
        }

        # Event tracking
        self.event_timeline = []
        self.signal_history = []

        # State variables
        self.market_state = {
            'trend': 'NEUTRAL',
            'volatility': 'NORMAL',
            'liquidity': 'GOOD',
        }

        # Performance metrics
        self.performance_metrics = {
            'daily_pnl': Decimal('0'),
            'max_drawdown': Decimal('0'),
            'win_rate': Decimal('0'),
            'sharpe_ratio': Decimal('0'),
        }

    def on_start(self):
        """Strategy start"""
        self.log.info("Comprehensive strategy starting")

        # Set up timers
        self._setup_timers()

        # Publish startup event
        self._publish_system_event("STRATEGY_START", "INFO", "Strategy started")

    def _setup_timers(self):
        """Set up all timers"""

        # 1. Market analysis (every 5 minutes)
        self.timers['market_analysis'] = self.clock.set_timer(
            name="market_analysis",
            interval_ns=5 * 60 * 1_000_000_000,
            callback=self._analyze_market,
        )

        # 2. Risk monitoring (every minute)
        self.timers['risk_monitor'] = self.clock.set_timer(
            name="risk_monitor",
            interval_ns=60 * 1_000_000_000,
            callback=self._monitor_risk,
        )

        # 3. Performance update (every hour)
        self.timers['performance_update'] = self.clock.set_timer(
            name="performance_update",
            interval_ns=3600 * 1_000_000_000,
            callback=self._update_performance,
        )

        # 4. News check (every 15 minutes)
        self.timers['news_check'] = self.clock.set_timer(
            name="news_check",
            interval_ns=15 * 60 * 1_000_000_000,
            callback=self._check_news,
        )

    def _analyze_market(self):
        """Market analysis"""
        current_time = self.clock.timestamp_ns()

        # Get latest market data
        # Should get from cache or data engine here
        # price = self.cache.get_last_price(...)

        # Simulate market analysis
        import random
        trend_change = random.choice(['UP', 'DOWN', 'NEUTRAL'])

        if trend_change != self.market_state['trend']:
            old_trend = self.market_state['trend']
            self.market_state['trend'] = trend_change

            # Publish trend change event
            self._publish_market_event(
                "TREND_CHANGE",
                f"Trend changed from {old_trend} to {trend_change}",
                {
                    'old_trend': old_trend,
                    'new_trend': trend_change,
                }
            )

            # Generate trading signal
            if trend_change == 'UP':
                self._generate_signal('BUY', 0.75)
            elif trend_change == 'DOWN':
                self._generate_signal('SELL', 0.75)

    def _monitor_risk(self):
        """Monitor risk"""
        # Check position risk
        total_exposure = self._calculate_total_exposure()
        max_allowed = Decimal('100000')  # Maximum exposure 100k

        if total_exposure > max_allowed:
            self._publish_risk_event(
                "EXPOSURE_LIMIT",
                "HIGH",
                f"Total exposure exceeded: {total_exposure} > {max_allowed}",
            )

        # Check drawdown
        current_drawdown = self._calculate_drawdown()
        if current_drawdown > Decimal('0.1'):  # 10%
            self._publish_risk_event(
                "DRAWDOWN",
                "CRITICAL",
                f"Drawdown too large: {current_drawdown:.1%}",
            )

    def _update_performance(self):
        """Update performance metrics"""
        # Calculate various metrics
        self.performance_metrics['daily_pnl'] = self._calculate_daily_pnl()
        self.performance_metrics['max_drawdown'] = self._calculate_drawdown()
        self.performance_metrics['win_rate'] = self._calculate_win_rate()
        self.performance_metrics['sharpe_ratio'] = self._calculate_sharpe_ratio()

        # Publish performance events
        for metric, value in self.performance_metrics.items():
            self._publish_performance_event(
                metric.upper(),
                float(value),
                "HOURLY"
            )

    def _check_news(self):
        """Check news (simulated)"""
        # Simulate news events
        import random
        news_impact = random.choices(
            ['POSITIVE', 'NEGATIVE', 'NEUTRAL'],
            weights=[0.2, 0.1, 0.7]
        )[0]

        if news_impact != 'NEUTRAL':
            self._publish_market_event(
                "NEWS_IMPACT",
                f"News impact: {news_impact}",
                {'impact': news_impact}
            )

    def on_bar(self, bar: Bar):
        """Process bar data"""
        # Update market state
        self._update_market_state(bar)

        # Generate real-time signals
        signal = self._analyze_bar_pattern(bar)
        if signal:
            self._generate_signal(signal['type'], signal['confidence'])

    def _update_market_state(self, bar: Bar):
        """Update market state"""
        # Calculate volatility
        price_change = abs(bar.close - bar.open) / bar.open
        if price_change > Decimal('0.02'):
            self.market_state['volatility'] = 'HIGH'
        elif price_change > Decimal('0.01'):
            self.market_state['volatility'] = 'MEDIUM'
        else:
            self.market_state['volatility'] = 'LOW'

    def _analyze_bar_pattern(self, bar: Bar) -> dict:
        """Analyze bar patterns"""
        # Simple pattern recognition
        body_size = abs(bar.close - bar.open)
        upper_shadow = bar.high - max(bar.open, bar.close)
        lower_shadow = min(bar.open, bar.close) - bar.low

        # Doji
        if body_size < (bar.high - bar.low) * Decimal('0.1'):
            return {'type': 'HOLD', 'confidence': 0.5}

        # Big bullish candle
        if bar.close > bar.open and body_size > (bar.high - bar.low) * Decimal('0.7'):
            return {'type': 'BUY', 'confidence': 0.6}

        # Big bearish candle
        if bar.close < bar.open and body_size > (bar.high - bar.low) * Decimal('0.7'):
            return {'type': 'SELL', 'confidence': 0.6}

        return None

    def _generate_signal(self, signal_type: str, confidence: float):
        """Generate trading signal"""
        signal = {
            'type': signal_type,
            'confidence': confidence,
            'timestamp': self.clock.timestamp_ns(),
            'state': self.market_state.copy(),
        }

        self.signal_history.append(signal)

        # Publish signal event
        event = SignalEvent(
            trader_id=self.trader_id,
            strategy_id=self.id,
            instrument_id="BTCUSDT.BINANCE",
            signal_type=signal_type,
            confidence=confidence,
            metadata=self.market_state.copy(),
        )

        self._handle_signal_event(event)

    def _calculate_total_exposure(self) -> Decimal:
        """Calculate total exposure"""
        # Simplified calculation
        return Decimal('50000')  # Simulated value

    def _calculate_drawdown(self) -> Decimal:
        """Calculate drawdown"""
        # Simplified calculation
        return Decimal('0.05')  # Simulated 5% drawdown

    def _calculate_daily_pnl(self) -> Decimal:
        """Calculate daily P&L"""
        # Simplified calculation
        return Decimal('1000')  # Simulated 1000 USDT

    def _calculate_win_rate(self) -> Decimal:
        """Calculate win rate"""
        # Simplified calculation
        return Decimal('0.6')  # Simulated 60%

    def _calculate_sharpe_ratio(self) -> Decimal:
        """Calculate Sharpe ratio"""
        # Simplified calculation
        return Decimal('1.5')  # Simulated 1.5

    def _publish_market_event(self, event_type: str, message: str, data: dict):
        """Publish market event"""
        event = CustomEvent(
            trader_id=self.trader_id,
            strategy_id=self.id,
            event_type=event_type,
            message=message,
            data=data,
        )
        self.event_timeline.append(event)

    def _publish_risk_event(self, risk_type: str, level: str, message: str):
        """Publish risk event"""
        event = RiskEvent(
            trader_id=self.trader_id,
            risk_type=risk_type,
            risk_level=level,
            message=message,
        )
        self.event_timeline.append(event)

    def _publish_performance_event(self, metric_type: str, value: float, period: str):
        """Publish performance event"""
        event = PerformanceEvent(
            trader_id=self.trader_id,
            strategy_id=self.id,
            metric_type=metric_type,
            metric_value=value,
            period=period,
        )
        self.event_timeline.append(event)

    def _publish_system_event(self, system_type: str, severity: str, message: str):
        """Publish system event"""
        event = SystemEvent(
            trader_id=self.trader_id,
            system_type=system_type,
            severity=severity,
            message=message,
        )
        self.event_timeline.append(event)

    def on_stop(self):
        """Strategy stop"""
        self.log.info("Comprehensive strategy stopping")

        # Cancel all timers
        for timer_name, timer_id in self.timers.items():
            if timer_id:
                self.clock.cancel_timer(timer_id)
                self.log.info(f"Cancelled timer: {timer_name}")

        # Generate final report
        self._generate_final_report()

    def _generate_final_report(self):
        """Generate final report"""
        report = f"""
Strategy Execution Report
================

Runtime: {self.clock.timestamp_ns() / 1_000_000_000:.0f} seconds
Total events: {len(self.event_timeline)}
Total signals: {len(self.signal_history)}

Market State
--------
Trend: {self.market_state['trend']}
Volatility: {self.market_state['volatility']}
Liquidity: {self.market_state['liquidity']}

Performance Metrics
--------
Daily P&L: {self.performance_metrics['daily_pnl']} USDT
Max Drawdown: {self.performance_metrics['max_drawdown']:.1%}
Win Rate: {self.performance_metrics['win_rate']:.1%}
Sharpe Ratio: {self.performance_metrics['sharpe_ratio']}

Recent Events
--------
"""

        # Add last 5 events
        for event in self.event_timeline[-5:]:
            report += f"- {event}\n"

        self.log.info(report)
```

## 6.5 Debugging and Monitoring

### 6.5.1 Event Tracking Tools

```python
"""
Event tracking and debugging tools
"""

import json
from datetime import datetime
from typing import Dict, List, Optional
from collections import defaultdict, deque


class EventTracker:
    """Event tracker"""

    def __init__(self, max_events: int = 10000):
        """
        Initialize tracker

        Parameters
        ----------
        max_events : int
            Maximum number of events
        """
        self.max_events = max_events
        self.events = deque(maxlen=max_events)
        self.event_stats = defaultdict(int)
        self.event_timeline = []

    def track_event(self, event: Event, source: str = "unknown"):
        """
        Track event

        Parameters
        ----------
        event : Event
            Event to track
        source : str
            Event source
        """
        event_info = {
            'id': str(event.id) if hasattr(event, 'id') else str(id(event)),
            'type': type(event).__name__,
            'timestamp': event.ts_event,
            'source': source,
            'data': self._serialize_event(event),
        }

        self.events.append(event_info)
        self.event_stats[event_info['type']] += 1

        # Add to timeline
        self.event_timeline.append({
            'timestamp': event.ts_event,
            'event_type': event_info['type'],
            'source': source,
        })

    def _serialize_event(self, event: Event) -> Dict:
        """Serialize event data"""
        data = {}

        # Extract common attributes
        if hasattr(event, 'trader_id'):
            data['trader_id'] = str(event.trader_id)
        if hasattr(event, 'strategy_id'):
            data['strategy_id'] = str(event.strategy_id)
        if hasattr(event, 'instrument_id'):
            data['instrument_id'] = str(event.instrument_id)

        # For custom events, add extra attributes
        if hasattr(event, 'metadata'):
            data['metadata'] = event.metadata

        return data

    def get_events_by_type(self, event_type: str) -> List[Dict]:
        """Get events of specified type"""
        return [e for e in self.events if e['type'] == event_type]

    def get_events_by_timerange(self, start_ns: int, end_ns: int) -> List[Dict]:
        """Get events within specified time range"""
        return [
            e for e in self.events
            if start_ns <= e['timestamp'] <= end_ns
        ]

    def get_event_sequence(self, event_id: str) -> List[Dict]:
        """Get event sequence"""
        sequence = []
        found = False

        for event in self.events:
            if event['id'] == event_id:
                found = True
            if found:
                sequence.append(event)

        return sequence

    def analyze_performance(self) -> Dict:
        """Analyze performance"""
        total_events = len(self.events)
        event_types = list(self.event_stats.keys())

        # Calculate event frequency
        if len(self.event_timeline) > 1:
            duration = (self.event_timeline[-1]['timestamp'] -
                      self.event_timeline[0]['timestamp'])
            frequency = total_events / (duration / 1_000_000_000) if duration > 0 else 0
        else:
            frequency = 0

        return {
            'total_events': total_events,
            'unique_types': len(event_types),
            'event_frequency': frequency,
            'type_distribution': dict(self.event_stats),
            'memory_usage': len(self.events) * 100 / self.max_events,  # Percentage
        }

    def export_to_file(self, filename: str):
        """Export to file"""
        data = {
            'export_time': datetime.now().isoformat(),
            'stats': self.analyze_performance(),
            'events': list(self.events),
            'timeline': self.event_timeline,
        }

        with open(filename, 'w') as f:
            json.dump(data, f, indent=2, default=str)

    def visualize_timeline(self):
        """Visualize timeline (simplified version)"""
        print("\nEvent Timeline:")
        print("-" * 80)

        last_time = None
        for event in self.event_timeline[-20:]:  # Show last 20 events
            time_str = datetime.fromtimestamp(
                event['timestamp'] / 1_000_000_000
            ).strftime('%H:%M:%S')

            if last_time:
                time_diff = (event['timestamp'] - last_time) / 1_000_000_000
                time_str += f" (+{time_diff:.1f}s)"

            print(f"{time_str}: {event['event_type']} from {event['source']}")
            last_time = event['timestamp']


class PerformanceProfiler:
    """Performance profiler"""

    def __init__(self):
        """Initialize profiler"""
        self.function_times = defaultdict(list)
        self.event_processing_times = defaultdict(list)

    def profile_function(self, func_name: str, execution_time_ns: int):
        """Record function execution time"""
        self.function_times[func_name].append(execution_time_ns)

    def profile_event_processing(
        self,
        event_type: str,
        processing_time_ns: int
    ):
        """Record event processing time"""
        self.event_processing_times[event_type].append(processing_time_ns)

    def get_function_stats(self) -> Dict:
        """Get function statistics"""
        stats = {}

        for func, times in self.function_times.items():
            if times:
                stats[func] = {
                    'count': len(times),
                    'total_time_ns': sum(times),
                    'avg_time_ns': sum(times) / len(times),
                    'max_time_ns': max(times),
                    'min_time_ns': min(times),
                }

        return stats

    def get_event_processing_stats(self) -> Dict:
        """Get event processing statistics"""
        stats = {}

        for event_type, times in self.event_processing_times.items():
            if times:
                stats[event_type] = {
                    'count': len(times),
                    'total_time_ns': sum(times),
                    'avg_time_ns': sum(times) / len(times),
                    'max_time_ns': max(times),
                }

        return stats

    def find_bottlenecks(self) -> List[Dict]:
        """Find performance bottlenecks"""
        bottlenecks = []

        # Function bottlenecks
        func_stats = self.get_function_stats()
        for func, stats in func_stats.items():
            if stats['avg_time_ns'] > 1_000_000:  # Over 1ms
                bottlenecks.append({
                    'type': 'function',
                    'name': func,
                    'avg_time_ms': stats['avg_time_ns'] / 1_000_000,
                    'issue': 'Function execution time too long'
                })

        # Event processing bottlenecks
        event_stats = self.get_event_processing_stats()
        for event_type, stats in event_stats.items():
            if stats['avg_time_ns'] > 1_000_000:  # Over 1ms
                bottlenecks.append({
                    'type': 'event',
                    'name': event_type,
                    'avg_time_ms': stats['avg_time_ns'] / 1_000_000,
                    'issue': 'Event processing time too long'
                })

        # Sort by time
        bottlenecks.sort(key=lambda x: x['avg_time_ms'], reverse=True)

        return bottlenecks


# Usage example
def main():
    """Main function example"""

    # Create event tracker
    tracker = EventTracker()
    profiler = PerformanceProfiler()

    # Create strategy
    strategy = ComprehensiveStrategy()

    # Simulate run
    import time
    start_time = time.time()

    # Simulate event processing
    for i in range(100):
        # Create simulated event
        event = SignalEvent(
            trader_id=TraderId("TEST-001"),
            strategy_id=StrategyId("TEST-STRATEGY"),
            instrument_id="BTCUSDT.BINANCE",
            signal_type="BUY",
            confidence=0.8,
        )

        # Track event
        start = time.time_ns()
        strategy._handle_signal_event(event)
        end = time.time_ns()

        tracker.track_event(event, "simulator")
        profiler.profile_event_processing("SignalEvent", end - start)

        # Simulate function call
        profiler.profile_function("market_analysis", time.time_ns() // 100)

    # Generate report
    print("\nEvent Tracking Report:")
    print("=" * 50)

    stats = tracker.analyze_performance()
    print(f"Total events: {stats['total_events']}")
    print(f"Event frequency: {stats['event_frequency']:.2f} events/s")

    print("\nPerformance Analysis:")
    print("=" * 50)

    bottlenecks = profiler.find_bottlenecks()
    if bottlenecks:
        print("Performance bottlenecks found:")
        for bottleneck in bottlenecks[:5]:
            print(f"- {bottleneck['name']}: {bottleneck['avg_time_ms']:.2f}ms "
                  f"({bottleneck['issue']})")
    else:
        print("No obvious performance bottlenecks found")

    # Export data
    tracker.export_to_file("event_trace.json")

    # Show timeline
    tracker.visualize_timeline()


if __name__ == "__main__":
    main()
```

## 6.6 Next Steps

In this chapter, we learned in-depth:

1. NautilusTrader's time management mechanism
2. Timer usage and advanced patterns
3. Event-driven architecture and best practices
4. Event handling and performance optimization
5. Debugging and monitoring tools

In the next chapter, we will learn:

1. Technical indicator systems
2. Using built-in indicators
3. Custom indicator development
4. Indicator cascading and combination
5. Indicator performance optimization

## 6.7 Summary

### Key Points
1. NautilusTrader provides a flexible time management system
2. Timers support complex scheduling logic
3. Event-driven architecture implements loosely coupled design
4. Event bus supports high-performance asynchronous processing

### Best Practices
1. Use timers reasonably, avoid over-scheduling
2. Event handling should be fast, avoid blocking
3. Use event tracking tools to debug issues
4. Regularly analyze performance bottlenecks

## 6.8 References

- [NautilusTrader Event System Documentation](https://nautilustrader.io/docs/latest/concepts/message_bus/)
- [Python Async Programming Guide](https://docs.python.org/3/library/asyncio.html)
- [Event-Driven Architecture Pattern](https://en.wikipedia.org/wiki/Event-driven_architecture)