# 第6章：时间管理和事件处理

## 学习目标

通过本章学习，您将：
- 理解 NautilusTrader 的时间管理机制
- 掌握定时器的使用方法
- 学会事件驱动编程模式
- 了解消息总线和事件路由
- 实现复杂的调度逻辑

## 6.1 时间管理架构

### 6.1.1 时钟类型

NautilusTrader 提供了多种时钟实现，以适应不同的需求：

```python
"""
时钟类型和使用场景
"""

from nautilus_trader.core.datetime import dt_to_unix_nanos
from nautilus_trader.test_kit.stubs.events import TestEventStubs
from datetime import datetime, timezone


# 1. LiveClock - 实时时钟（用于实盘交易）
from nautilus_trader.core.datetime import nanos_to_unix_nanos

class LiveClockExample:
    """实时时钟示例"""

    def __init__(self):
        from nautilus_trader.core.datetime import get_atomic_clock
        self.clock = get_atomic_clock()

    def get_current_time(self):
        """获取当前时间（纳秒）"""
        return self.clock.timestamp_ns()

    def wait_until(self, target_time_ns: int):
        """等待到指定时间"""
        import time
        current_time = self.clock.timestamp_ns()
        if current_time < target_time_ns:
            sleep_seconds = (target_time_ns - current_time) / 1_000_000_000
            time.sleep(sleep_seconds)

# 2. TestClock - 测试时钟（用于回测）
class TestClockExample:
    """测试时钟示例"""

    def __init__(self):
        from nautilus_trader.test_kit.stubs.events import TestClockStub
        # 创建测试时钟
        self.clock = TestClockStub()

        # 设置初始时间
        self.clock.set_time(dt_to_unix_nanos(datetime(2024, 1, 1)))

    def advance_time(self, seconds: int):
        """推进时间"""
        self.clock.advance_time(seconds * 1_000_000_000)

    def schedule_event(self, delay_ns: int, callback):
        """调度事件"""
        self.clock.set_time_ns_callback(
            target_time=self.clock.timestamp_ns() + delay_ns,
            callback=callback
        )


# 3. MonotonicClock - 单调时钟
class MonotonicClockExample:
    """单调时钟示例"""

    def __init__(self):
        from nautilus_trader.core.datetime import MonotonicClock
        self.clock = MonotonicClock()

    def measure_duration(self, func):
        """测量函数执行时间"""
        start = self.clock.timestamp_ns()
        func()
        end = self.clock.timestamp_ns()
        return end - start
```

### 6.1.2 时间格式和转换

```python
"""
时间处理工具
"""

from datetime import datetime, timedelta, timezone
from decimal import Decimal
import pytz


class TimeUtils:
    """时间处理工具类"""

    @staticmethod
    def to_utc_naive(dt: datetime) -> datetime:
        """转换为UTC时间（无时区）"""
        if dt.tzinfo is None:
            return dt
        return dt.astimezone(timezone.utc).replace(tzinfo=None)

    @staticmethod
    def from_utc_naive(dt: datetime, tz_str: str = "UTC") -> datetime:
        """从UTC时间转换为指定时区"""
        tz = pytz.timezone(tz_str)
        return tz.localize(dt)

    @staticmethod
    def to_nanos(dt: datetime) -> int:
        """转换为纳秒时间戳"""
        return int(dt.timestamp() * 1_000_000_000)

    @staticmethod
    def from_nanos(nanos: int) -> datetime:
        """从纳秒时间戳转换"""
        return datetime.fromtimestamp(nanos / 1_000_000_000, tz=timezone.utc)

    @staticmethod
    def format_duration(nanos: int) -> str:
        """格式化时间间隔"""
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
        获取交易时段

        Returns:
            tuple: (session_start, session_end, is_trading_time)
        """
        tz = pytz.timezone(timezone_str)
        local_dt = dt.astimezone(tz)

        # 标准交易时间：9:30 - 16:00
        session_start = local_dt.replace(
            hour=9, minute=30, second=0, microsecond=0
        )
        session_end = local_dt.replace(
            hour=16, minute=0, second=0, microsecond=0
        )

        is_trading = session_start <= local_dt <= session_end

        return session_start, session_end, is_trading


# 使用示例
def main():
    """时间处理示例"""

    utils = TimeUtils()

    # 当前时间
    now = datetime.now()
    print(f"本地时间: {now}")

    # 转换为UTC
    utc_time = utils.to_utc_naive(now)
    print(f"UTC时间: {utc_time}")

    # 转换为纳秒
    nanos = utils.to_nanos(utc_time)
    print(f"纳秒时间戳: {nanos}")

    # 转换回来
    dt_back = utils.from_nanos(nanos)
    print(f"转换回来: {dt_back}")

    # 交易时间检查
    session_start, session_end, is_trading = utils.get_trading_session_time(now)
    print(f"交易时段: {session_start} - {session_end}")
    print(f"当前是否交易时间: {is_trading}")


if __name__ == "__main__":
    main()
```

## 6.2 定时器系统

### 6.2.1 基础定时器

```python
"""
定时器使用示例
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
    """使用定时器的策略示例"""

    def __init__(self):
        """初始化策略"""
        super().__init__()

        # 定时器ID管理
        self.timers = {
            'daily_reset': None,
            'hourly_update': None,
            'minute_check': None,
            'exit_check': None,
        }

        # 状态跟踪
        self.daily_high = Decimal('0')
        self.daily_low = Decimal('999999')
        self.hourly_volume = Decimal('0')
        self.last_bar_time = None
        self.positions_held_time = {}

    def on_start(self):
        """策略启动时设置定时器"""
        self.log.info("定时器策略启动")

        # 每日重置定时器（UTC 00:00）
        self._schedule_daily_reset()

        # 每小时更新
        self._schedule_hourly_update()

        # 每分钟检查
        self._schedule_minute_check()

        # 每30秒检查退出条件
        self._schedule_exit_check()

    def _schedule_daily_reset(self):
        """调度每日重置定时器"""
        # 计算下一个UTC 00:00
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

        self.log.info(f"每日重置定时器设置，下次执行: {next_midnight}")

    def _on_daily_reset(self):
        """每日重置回调"""
        self.log.info("执行每日重置")

        # 重置统计
        self.daily_high = Decimal('0')
        self.daily_low = Decimal('999999')
        self.hourly_volume = Decimal('0')

        # 清空持仓时间记录
        self.positions_held_time.clear()

        # 重新调度下次重置
        self._schedule_daily_reset()

    def _schedule_hourly_update(self):
        """调度每小时更新"""
        # 计算到下一个整小时的延迟
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
        """每小时更新回调"""
        self.log.info(f"每小时更新 - 当前小时成交量: {self.hourly_volume}")

        # 重置小时成交量
        self.hourly_volume = Decimal('0')

        # 重新调度
        self._schedule_hourly_update()

        # 这里可以执行其他小时级任务
        self._analyze_hourly_performance()

    def _schedule_minute_check(self):
        """调度每分钟检查"""
        self.timers['minute_check'] = self.clock.set_timer(
            name="minute_check",
            interval_ns=60 * 1_000_000_000,  # 1分钟
            callback=self._on_minute_check,
        )

    def _on_minute_check(self):
        """每分钟检查回调"""
        current_time = self.clock.timestamp_ns()

        # 检查是否有长时间未更新的数据
        if self.last_bar_time:
            time_diff = current_time - self.last_bar_time
            if time_diff > 2 * 60 * 1_000_000_000:  # 超过2分钟
                self.log.warning("数据更新异常 - 可能存在连接问题")

    def _schedule_exit_check(self):
        """调度退出检查"""
        self.timers['exit_check'] = self.clock.set_timer(
            name="exit_check",
            interval_ns=30 * 1_000_000_000,  # 30秒
            callback=self._on_exit_check,
        )

    def _on_exit_check(self):
        """退出条件检查"""
        # 检查持仓时间
        current_time = self.clock.timestamp_ns()

        for position_id, entry_time in list(self.positions_held_time.items()):
            hold_duration = current_time - entry_time
            hold_hours = hold_duration / (1_000_000_000 * 3600)

            # 持仓超过24小时强制平仓
            if hold_hours > 24:
                self.log.warning(
                    f"持仓时间过长 - 强制平仓: {position_id}, "
                    f"持仓时间: {hold_hours:.1f}小时"
                )
                self._force_close_position(position_id)

    def on_bar(self, bar: Bar):
        """处理K线数据"""
        # 更新最后K线时间
        self.last_bar_time = self.clock.timestamp_ns()

        # 更新日内高低点
        if bar.close > self.daily_high:
            self.daily_high = bar.close

        if bar.close < self.daily_low:
            self.daily_low = bar.close

        # 累计成交量
        self.hourly_volume += bar.volume

        # 检查交易信号
        self._check_trading_signal(bar)

    def on_order_filled(self, event: OrderFilled):
        """处理订单成交"""
        # 记录持仓时间
        self.positions_held_time[event.position_id] = self.clock.timestamp_ns()

    def _check_trading_signal(self, bar: Bar):
        """检查交易信号"""
        # 基于高低点突破的策略
        if bar.close > self.daily_high * Decimal('1.001'):  # 突破高点0.1%
            self._enter_long(bar)
        elif bar.close < self.daily_low * Decimal('0.999'):  # 跌破低点0.1%
            self._enter_short(bar)

    def _analyze_hourly_performance(self):
        """分析每小时表现"""
        # 这里可以实现每小时的性能分析
        pass

    def _force_close_position(self, position_id):
        """强制平仓"""
        # 实现强制平仓逻辑
        pass

    def on_stop(self):
        """策略停止时清理定时器"""
        self.log.info("停止定时器策略")

        # 取消所有定时器
        for timer_name, timer_id in self.timers.items():
            if timer_id:
                self.clock.cancel_timer(timer_id)
                self.log.info(f"取消定时器: {timer_name}")
```

### 6.2.2 高级定时器模式

```python
"""
高级定时器模式
"""

from typing import Callable, Dict, List
from enum import Enum
import heapq


class TimerPriority(Enum):
    """定时器优先级"""
    CRITICAL = 1    # 关键任务（如风险检查）
    HIGH = 2        # 高优先级（如交易信号）
    NORMAL = 3      # 正常优先级（如统计更新）
    LOW = 4         # 低优先级（如清理任务）


class AdvancedTimerManager:
    """高级定时器管理器"""

    def __init__(self, clock):
        """初始化定时器管理器"""
        self.clock = clock
        self.timers = {}  # timer_id -> timer_info
        self.timer_heap = []  # 优先队列

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
        调度定时器

        Parameters
        ----------
        name : str
            定时器名称
        callback : Callable
            回调函数
        delay_ns : int
            延迟时间（纳秒）
        interval_ns : int
            重复间隔（纳秒），None表示单次
        priority : TimerPriority
            优先级
        metadata : Dict
            元数据

        Returns
        -------
        str
            定时器ID
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

        # 添加到优先队列
        heapq.heappush(
            self.timer_heap,
            (timer_info['next_run'], priority.value, timer_info['id'])
        )

        return timer_info['id']

    def cancel(self, timer_id: str):
        """取消定时器"""
        if timer_id in self.timers:
            self.timers[timer_id]['active'] = False
            del self.timers[timer_id]

    def process_pending(self):
        """处理待执行的定时器"""
        current_time = self.clock.timestamp_ns()

        while self.timer_heap:
            next_time, priority, timer_id = self.timer_heap[0]

            if next_time > current_time:
                break

            # 弹出定时器
            heapq.heappop(self.timer_heap)

            if timer_id not in self.timers:
                continue

            timer = self.timers[timer_id]
            if not timer['active']:
                continue

            # 执行回调
            try:
                timer['callback']()
                timer['run_count'] += 1
            except Exception as e:
                self.log.error(f"定时器执行错误: {timer_id}, 错误: {e}")

            # 如果是重复定时器，调度下次执行
            if timer['interval_ns']:
                timer['next_run'] = current_time + timer['interval_ns']
                heapq.heappush(
                    self.timer_heap,
                    (timer['next_run'], timer['priority'].value, timer_id)
                )
            else:
                # 单次定时器，删除
                del self.timers[timer_id]

    def get_stats(self) -> Dict:
        """获取统计信息"""
        total_timers = len(self.timers)

        # 按优先级统计
        priority_stats = {}
        for priority in TimerPriority:
            count = sum(
                1 for t in self.timers.values()
                if t['priority'] == priority and t['active']
            )
            priority_stats[priority.name] = count

        # 下次执行时间
        next_run_time = None
        if self.timer_heap:
            next_run_time = self.timer_heap[0][0]

        return {
            'total_active': total_timers,
            'by_priority': priority_stats,
            'next_run_ns': next_run_time,
        }


class ScheduledTaskStrategy(Strategy):
    """使用高级定时器的策略"""

    def __init__(self):
        """初始化策略"""
        super().__init__()

        # 任务调度器
        self.scheduler = AdvancedTimerManager(self.clock)

        # 统计数据
        self.stats = {
            'signals_detected': 0,
            'orders_submitted': 0,
            'risk_checks': 0,
            'data_updates': 0,
        }

    def on_start(self):
        """策略启动"""
        self.log.info("定时任务策略启动")

        # 调度各种任务

        # 1. 数据分析任务（每5分钟）
        self.scheduler.schedule(
            name="data_analysis",
            callback=self._analyze_market_data,
            delay_ns=5 * 60 * 1_000_000_000,  # 5分钟后开始
            interval_ns=5 * 60 * 1_000_000_000,  # 每5分钟
            priority=TimerPriority.NORMAL,
        )

        # 2. 风险检查任务（每分钟）
        self.scheduler.schedule(
            name="risk_check",
            callback=self._check_risk,
            delay_ns=60 * 1_000_000_000,  # 1分钟后开始
            interval_ns=60 * 1_000_000_000,  # 每分钟
            priority=TimerPriority.CRITICAL,
        )

        # 3. 信号扫描任务（每10秒）
        self.scheduler.schedule(
            name="signal_scan",
            callback=self._scan_signals,
            delay_ns=10 * 1_000_000_000,  # 10秒后开始
            interval_ns=10 * 1_000_000_000,  # 每10秒
            priority=TimerPriority.HIGH,
        )

        # 4. 性能统计任务（每小时）
        self.scheduler.schedule(
            name="performance_stats",
            callback=self._update_performance_stats,
            delay_ns=3600 * 1_000_000_000,  # 1小时后开始
            interval_ns=3600 * 1_000_000_000,  # 每小时
            priority=TimerPriority.LOW,
        )

        # 5. 清理任务（每天凌晨）
        self.scheduler.schedule(
            name="daily_cleanup",
            callback=self._daily_cleanup,
            delay_ns=self._get_time_to_next_midnight(),
            interval_ns=24 * 3600 * 1_000_000_000,  # 每天
            priority=TimerPriority.LOW,
        )

        # 启动定时器处理循环
        self.scheduler.schedule(
            name="timer_processor",
            callback=self._process_timers,
            delay_ns=100_000_000,  # 100ms后开始
            interval_ns=100_000_000,  # 每100ms
            priority=TimerPriority.CRITICAL,
        )

    def _process_timers(self):
        """处理待执行的定时器"""
        self.scheduler.process_pending()

    def _analyze_market_data(self):
        """分析市场数据"""
        self.stats['data_updates'] += 1
        self.log.debug(f"执行市场数据分析 - 更新次数: {self.stats['data_updates']}")

        # 获取最新数据
        # ... 分析逻辑 ...

    def _check_risk(self):
        """检查风险"""
        self.stats['risk_checks'] += 1
        self.log.debug(f"执行风险检查 - 检查次数: {self.stats['risk_checks']}")

        # 检查持仓风险
        # ... 风险逻辑 ...

    def _scan_signals(self):
        """扫描交易信号"""
        self.stats['signals_detected'] += 1
        self.log.debug(f"扫描交易信号 - 扫描次数: {self.stats['signals_detected']}")

        # 扫描市场信号
        # ... 信号逻辑 ...

    def _update_performance_stats(self):
        """更新性能统计"""
        self.log.info(
            f"性能统计 - "
            f"信号: {self.stats['signals_detected']}, "
            f"订单: {self.stats['orders_submitted']}, "
            f"风控: {self.stats['risk_checks']}"
        )

    def _daily_cleanup(self):
        """每日清理"""
        self.log.info("执行每日清理")

        # 清理过期数据
        # 重置统计
        # ... 清理逻辑 ...

    def _get_time_to_next_midnight(self) -> int:
        """获取到下一个午夜的时间"""
        now = datetime.now()
        next_midnight = (now + timedelta(days=1)).replace(
            hour=0, minute=0, second=0, microsecond=0
        )
        return int((next_midnight - now).total_seconds() * 1_000_000_000)

    def on_bar(self, bar: Bar):
        """处理K线数据"""
        # 定时器会自动处理，这里只处理实时数据
        pass

    def on_stop(self):
        """策略停止"""
        self.log.info("定时任务策略停止")

        # 显示最终统计
        stats = self.scheduler.get_stats()
        self.log.info(f"定时器统计: {stats}")
        self.log.info(f"任务统计: {self.stats}")
```

## 6.3 事件驱动架构

### 6.3.1 事件类型和继承

```python
"""
事件系统架构
"""

from abc import ABC
from typing import Any, Dict, Optional
from datetime import datetime

from nautilus_trader.core.message import Event
from nautilus_trader.model.identifiers import TraderId, StrategyId


class CustomEvent(Event, ABC):
    """自定义事件基类"""

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
        初始化自定义事件

        Parameters
        ----------
        trader_id : TraderId
            交易者ID
        strategy_id : StrategyId, optional
            策略ID
        event_id : str, optional
            事件ID
        ts_init : int, optional
            初始化时间戳
        ts_event : int, optional
            事件时间戳
        **kwargs : Any
            其他参数
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
    """交易信号事件"""

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
        初始化信号事件

        Parameters
        ----------
        instrument_id : str
            交易工具ID
        signal_type : str
            信号类型
        confidence : float
            置信度
        price : float, optional
            建议价格
        quantity : float, optional
            建议数量
        metadata : Dict, optional
            额外元数据
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
        """转换为字典"""
        return {
            'instrument_id': self.instrument_id,
            'signal_type': self.signal_type,
            'confidence': self.confidence,
            'price': self.price,
            'quantity': self.quantity,
            'timestamp': self.ts_event,
        }


class RiskEvent(CustomEvent):
    """风险事件"""

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
        初始化风险事件

        Parameters
        ----------
        risk_type : str
            风险类型
        risk_level : str
            风险级别
        message : str
            风险描述
        affected_positions : list, optional
            受影响的仓位
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
        """是否需要立即处理"""
        return self.risk_level in ['HIGH', 'CRITICAL']


class PerformanceEvent(CustomEvent):
    """性能统计事件"""

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
        初始化性能事件

        Parameters
        ----------
        metric_type : str
            指标类型
        metric_value : float
            指标值
        period : str
            统计周期
        benchmark_value : float, optional
            基准值
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
        """是否跑赢基准"""
        if self.benchmark_value is None:
            return None
        return self.metric_value > self.benchmark_value


class SystemEvent(CustomEvent):
    """系统事件"""

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
        初始化系统事件

        Parameters
        ----------
        system_type : str
            系统类型
        severity : str
            严重程度
        message : str
            事件消息
        component : str, optional
            组件名称
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
        """是否需要告警"""
        return self.severity in ['ERROR', 'CRITICAL']
```

### 6.3.2 事件处理器和调度器

```python
"""
事件处理系统
"""

from typing import Callable, Dict, List, Type
from collections import defaultdict
from concurrent.futures import ThreadPoolExecutor
import threading
import queue
import time


class EventBus:
    """事件总线"""

    def __init__(self, max_workers: int = 4):
        """
        初始化事件总线

        Parameters
        ----------
        max_workers : int
            最大工作线程数
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
        订阅事件

        Parameters
        ----------
        event_type : Type[Event]
            事件类型
        handler : Callable
            处理函数
        priority : int
            优先级（数字越大优先级越高）
        """
        self.handlers[event_type].append((handler, priority))
        # 按优先级排序
        self.handlers[event_type].sort(key=lambda x: x[1], reverse=True)

    def unsubscribe(self, event_type: Type[Event], handler: Callable):
        """取消订阅"""
        handlers = self.handlers.get(event_type, [])
        self.handlers[event_type] = [
            (h, p) for h, p in handlers if h != handler
        ]

    def publish(self, event: Event):
        """
        发布事件（异步）

        Parameters
        ----------
        event : Event
            要发布的事件
        """
        self.event_queue.put(event)

    def publish_sync(self, event: Event):
        """
        发布事件（同步）

        Parameters
        ----------
        event : Event
            要发布的事件
        """
        self._process_event(event)

    def start(self):
        """启动事件处理"""
        if not self.running:
            self.running = True
            self.processing_thread = threading.Thread(
                target=self._process_loop,
                daemon=True
            )
            self.processing_thread.start()

    def stop(self):
        """停止事件处理"""
        self.running = False
        if self.processing_thread:
            self.processing_thread.join()

    def _process_loop(self):
        """事件处理循环"""
        while self.running:
            try:
                # 从队列获取事件
                event = self.event_queue.get(timeout=1.0)
                self._process_event(event)
                self.event_queue.task_done()
            except queue.Empty:
                continue
            except Exception as e:
                print(f"事件处理错误: {e}")

    def _process_event(self, event: Event):
        """处理单个事件"""
        event_type = type(event)
        handlers = self.handlers.get(event_type, [])

        # 并行处理所有订阅者
        futures = []
        for handler, priority in handlers:
            future = self.executor.submit(self._safe_handle, handler, event)
            futures.append(future)

        # 等待所有处理完成
        for future in futures:
            try:
                future.result(timeout=5.0)
            except Exception as e:
                print(f"处理器执行错误: {e}")

    def _safe_handle(self, handler: Callable, event: Event):
        """安全地调用处理器"""
        try:
            handler(event)
        except Exception as e:
            print(f"事件处理异常: {e}")

    def get_stats(self) -> Dict:
        """获取统计信息"""
        return {
            'queue_size': self.event_queue.qsize(),
            'subscriptions': {
                str(event_type): len(handlers)
                for event_type, handlers in self.handlers.items()
            },
            'running': self.running,
        }


class EventDrivenStrategy(Strategy):
    """事件驱动策略基类"""

    def __init__(self):
        """初始化策略"""
        super().__init__()

        # 事件总线
        self.event_bus = EventBus()

        # 事件处理器注册
        self._register_event_handlers()

        # 统计信息
        self.event_stats = defaultdict(int)

    def _register_event_handlers(self):
        """注册事件处理器"""
        # 注册自定义事件处理器
        self.event_bus.subscribe(SignalEvent, self._handle_signal_event)
        self.event_bus.subscribe(RiskEvent, self._handle_risk_event)
        self.event_bus.subscribe(PerformanceEvent, self._handle_performance_event)
        self.event_bus.subscribe(SystemEvent, self._handle_system_event)

    def on_start(self):
        """策略启动"""
        self.log.info("事件驱动策略启动")
        self.event_bus.start()

        # 发布启动事件
        self._publish_system_event("STARTUP", "INFO", "策略已启动")

    def on_stop(self):
        """策略停止"""
        self.log.info("事件驱动策略停止")
        self.event_bus.stop()

        # 发布停止事件
        self._publish_system_event("SHUTDOWN", "INFO", "策略已停止")

        # 显示事件统计
        self.log.info(f"事件统计: {dict(self.event_stats)}")

    def _handle_signal_event(self, event: SignalEvent):
        """处理信号事件"""
        self.event_stats['signals'] += 1

        self.log.info(
            f"收到信号事件: {event.signal_type} "
            f"(置信度: {event.confidence:.2f})"
        )

        # 根据信号执行交易
        if event.signal_type == 'BUY':
            self._execute_buy_signal(event)
        elif event.signal_type == 'SELL':
            self._execute_sell_signal(event)

    def _handle_risk_event(self, event: RiskEvent):
        """处理风险事件"""
        self.event_stats['risk_events'] += 1

        self.log.warning(
            f"风险事件: {event.risk_type} - {event.message}"
        )

        # 根据风险级别采取措施
        if event.requires_immediate_action():
            self._handle_critical_risk(event)
        else:
            self._handle_normal_risk(event)

    def _handle_performance_event(self, event: PerformanceEvent):
        """处理性能事件"""
        self.event_stats['performance_events'] += 1

        self.log.info(
            f"性能更新: {event.metric_type} = {event.metric_value} "
            f"({event.period})"
        )

        # 检查性能表现
        if event.is_outperforming() is False:
            self.log.warning("策略表现不佳，考虑调整参数")

    def _handle_system_event(self, event: SystemEvent):
        """处理系统事件"""
        self.event_stats['system_events'] += 1

        if event.requires_alert():
            self.log.error(f"系统告警: {event.message}")
        else:
            self.log.info(f"系统事件: {event.message}")

    def _execute_buy_signal(self, event: SignalEvent):
        """执行买入信号"""
        # 实现买入逻辑
        pass

    def _execute_sell_signal(self, event: SignalEvent):
        """执行卖出信号"""
        # 实现卖出逻辑
        pass

    def _handle_critical_risk(self, event: RiskEvent):
        """处理关键风险"""
        # 立即采取措施，如停止交易或平仓
        pass

    def _handle_normal_risk(self, event: RiskEvent):
        """处理普通风险"""
        # 记录日志，监控风险发展
        pass

    def _publish_signal_event(self, signal_type: str, confidence: float):
        """发布信号事件"""
        event = SignalEvent(
            trader_id=self.trader_id,
            strategy_id=self.id,
            instrument_id="BTCUSDT.BINANCE",
            signal_type=signal_type,
            confidence=confidence,
        )
        self.event_bus.publish(event)

    def _publish_risk_event(self, risk_type: str, risk_level: str, message: str):
        """发布风险事件"""
        event = RiskEvent(
            trader_id=self.trader_id,
            risk_type=risk_type,
            risk_level=risk_level,
            message=message,
        )
        self.event_bus.publish(event)

    def _publish_performance_event(self, metric_type: str, value: float, period: str):
        """发布性能事件"""
        event = PerformanceEvent(
            trader_id=self.trader_id,
            strategy_id=self.id,
            metric_type=metric_type,
            metric_value=value,
            period=period,
        )
        self.event_bus.publish(event)

    def _publish_system_event(self, system_type: str, severity: str, message: str):
        """发布系统事件"""
        event = SystemEvent(
            trader_id=self.trader_id,
            system_type=system_type,
            severity=severity,
            message=message,
        )
        self.event_bus.publish(event)


# 使用示例
def main():
    """主函数示例"""

    # 创建事件总线
    event_bus = EventBus()

    # 创建策略
    strategy = EventDrivenStrategy()

    # 模拟事件
    signal = SignalEvent(
        trader_id=TraderId("TRADER-001"),
        strategy_id=StrategyId("STRATEGY-001"),
        instrument_id="BTCUSDT.BINANCE",
        signal_type="BUY",
        confidence=0.85,
    )

    # 发布事件
    event_bus.publish(signal)

    # 运行一段时间后停止
    time.sleep(1)
    event_bus.stop()


if __name__ == "__main__":
    main()
```

## 6.4 实际应用示例

### 6.4.1 综合定时器和事件系统

```python
"""
综合定时器和事件系统示例
"""

from nautilus_trader.trading.strategy import Strategy
from nautilus_trader.model.data import Bar
from nautilus_trader.model.events import OrderFilled
from nautilus_trader.model.objects import Money
from decimal import Decimal
from datetime import datetime, timedelta
import asyncio


class ComprehensiveStrategy(Strategy):
    """综合使用定时器和事件的策略"""

    def __init__(self):
        """初始化策略"""
        super().__init__()

        # 定时器管理
        self.timers = {
            'market_analysis': None,
            'risk_monitor': None,
            'performance_update': None,
            'news_check': None,
        }

        # 事件跟踪
        self.event_timeline = []
        self.signal_history = []

        # 状态变量
        self.market_state = {
            'trend': 'NEUTRAL',
            'volatility': 'NORMAL',
            'liquidity': 'GOOD',
        }

        # 性能指标
        self.performance_metrics = {
            'daily_pnl': Decimal('0'),
            'max_drawdown': Decimal('0'),
            'win_rate': Decimal('0'),
            'sharpe_ratio': Decimal('0'),
        }

    def on_start(self):
        """策略启动"""
        self.log.info("综合策略启动")

        # 设置定时器
        self._setup_timers()

        # 发布启动事件
        self._publish_system_event("STRATEGY_START", "INFO", "策略已启动")

    def _setup_timers(self):
        """设置所有定时器"""

        # 1. 市场分析（每5分钟）
        self.timers['market_analysis'] = self.clock.set_timer(
            name="market_analysis",
            interval_ns=5 * 60 * 1_000_000_000,
            callback=self._analyze_market,
        )

        # 2. 风险监控（每分钟）
        self.timers['risk_monitor'] = self.clock.set_timer(
            name="risk_monitor",
            interval_ns=60 * 1_000_000_000,
            callback=self._monitor_risk,
        )

        # 3. 性能更新（每小时）
        self.timers['performance_update'] = self.clock.set_timer(
            name="performance_update",
            interval_ns=3600 * 1_000_000_000,
            callback=self._update_performance,
        )

        # 4. 新闻检查（每15分钟）
        self.timers['news_check'] = self.clock.set_timer(
            name="news_check",
            interval_ns=15 * 60 * 1_000_000_000,
            callback=self._check_news,
        )

    def _analyze_market(self):
        """市场分析"""
        current_time = self.clock.timestamp_ns()

        # 获取最新市场数据
        # 这里应该从缓存或数据引擎获取
        # price = self.cache.get_last_price(...)

        # 模拟市场分析
        import random
        trend_change = random.choice(['UP', 'DOWN', 'NEUTRAL'])

        if trend_change != self.market_state['trend']:
            old_trend = self.market_state['trend']
            self.market_state['trend'] = trend_change

            # 发布趋势变化事件
            self._publish_market_event(
                "TREND_CHANGE",
                f"趋势从 {old_trend} 变为 {trend_change}",
                {
                    'old_trend': old_trend,
                    'new_trend': trend_change,
                }
            )

            # 生成交易信号
            if trend_change == 'UP':
                self._generate_signal('BUY', 0.75)
            elif trend_change == 'DOWN':
                self._generate_signal('SELL', 0.75)

    def _monitor_risk(self):
        """监控风险"""
        # 检查持仓风险
        total_exposure = self._calculate_total_exposure()
        max_allowed = Decimal('100000')  # 最大敞口10万

        if total_exposure > max_allowed:
            self._publish_risk_event(
                "EXPOSURE_LIMIT",
                "HIGH",
                f"总敞口超限: {total_exposure} > {max_allowed}",
            )

        # 检查回撤
        current_drawdown = self._calculate_drawdown()
        if current_drawdown > Decimal('0.1'):  # 10%
            self._publish_risk_event(
                "DRAWDOWN",
                "CRITICAL",
                f"回撤过大: {current_drawdown:.1%}",
            )

    def _update_performance(self):
        """更新性能指标"""
        # 计算各项指标
        self.performance_metrics['daily_pnl'] = self._calculate_daily_pnl()
        self.performance_metrics['max_drawdown'] = self._calculate_drawdown()
        self.performance_metrics['win_rate'] = self._calculate_win_rate()
        self.performance_metrics['sharpe_ratio'] = self._calculate_sharpe_ratio()

        # 发布性能事件
        for metric, value in self.performance_metrics.items():
            self._publish_performance_event(
                metric.upper(),
                float(value),
                "HOURLY"
            )

    def _check_news(self):
        """检查新闻（模拟）"""
        # 模拟新闻事件
        import random
        news_impact = random.choices(
            ['POSITIVE', 'NEGATIVE', 'NEUTRAL'],
            weights=[0.2, 0.1, 0.7]
        )[0]

        if news_impact != 'NEUTRAL':
            self._publish_market_event(
                "NEWS_IMPACT",
                f"新闻影响: {news_impact}",
                {'impact': news_impact}
            )

    def on_bar(self, bar: Bar):
        """处理K线数据"""
        # 更新市场状态
        self._update_market_state(bar)

        # 生成实时信号
        signal = self._analyze_bar_pattern(bar)
        if signal:
            self._generate_signal(signal['type'], signal['confidence'])

    def _update_market_state(self, bar: Bar):
        """更新市场状态"""
        # 计算波动率
        price_change = abs(bar.close - bar.open) / bar.open
        if price_change > Decimal('0.02'):
            self.market_state['volatility'] = 'HIGH'
        elif price_change > Decimal('0.01'):
            self.market_state['volatility'] = 'MEDIUM'
        else:
            self.market_state['volatility'] = 'LOW'

    def _analyze_bar_pattern(self, bar: Bar) -> dict:
        """分析K线模式"""
        # 简单的模式识别
        body_size = abs(bar.close - bar.open)
        upper_shadow = bar.high - max(bar.open, bar.close)
        lower_shadow = min(bar.open, bar.close) - bar.low

        # 十字星
        if body_size < (bar.high - bar.low) * Decimal('0.1'):
            return {'type': 'HOLD', 'confidence': 0.5}

        # 大阳线
        if bar.close > bar.open and body_size > (bar.high - bar.low) * Decimal('0.7'):
            return {'type': 'BUY', 'confidence': 0.6}

        # 大阴线
        if bar.close < bar.open and body_size > (bar.high - bar.low) * Decimal('0.7'):
            return {'type': 'SELL', 'confidence': 0.6}

        return None

    def _generate_signal(self, signal_type: str, confidence: float):
        """生成交易信号"""
        signal = {
            'type': signal_type,
            'confidence': confidence,
            'timestamp': self.clock.timestamp_ns(),
            'state': self.market_state.copy(),
        }

        self.signal_history.append(signal)

        # 发布信号事件
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
        """计算总敞口"""
        # 简化计算
        return Decimal('50000')  # 模拟值

    def _calculate_drawdown(self) -> Decimal:
        """计算回撤"""
        # 简化计算
        return Decimal('0.05')  # 模拟5%回撤

    def _calculate_daily_pnl(self) -> Decimal:
        """计算日盈亏"""
        # 简化计算
        return Decimal('1000')  # 模拟1000 USDT

    def _calculate_win_rate(self) -> Decimal:
        """计算胜率"""
        # 简化计算
        return Decimal('0.6')  # 模拟60%

    def _calculate_sharpe_ratio(self) -> Decimal:
        """计算夏普比率"""
        # 简化计算
        return Decimal('1.5')  # 模拟1.5

    def _publish_market_event(self, event_type: str, message: str, data: dict):
        """发布市场事件"""
        event = CustomEvent(
            trader_id=self.trader_id,
            strategy_id=self.id,
            event_type=event_type,
            message=message,
            data=data,
        )
        self.event_timeline.append(event)

    def _publish_risk_event(self, risk_type: str, level: str, message: str):
        """发布风险事件"""
        event = RiskEvent(
            trader_id=self.trader_id,
            risk_type=risk_type,
            risk_level=level,
            message=message,
        )
        self.event_timeline.append(event)

    def _publish_performance_event(self, metric_type: str, value: float, period: str):
        """发布性能事件"""
        event = PerformanceEvent(
            trader_id=self.trader_id,
            strategy_id=self.id,
            metric_type=metric_type,
            metric_value=value,
            period=period,
        )
        self.event_timeline.append(event)

    def _publish_system_event(self, system_type: str, severity: str, message: str):
        """发布系统事件"""
        event = SystemEvent(
            trader_id=self.trader_id,
            system_type=system_type,
            severity=severity,
            message=message,
        )
        self.event_timeline.append(event)

    def on_stop(self):
        """策略停止"""
        self.log.info("综合策略停止")

        # 取消所有定时器
        for timer_name, timer_id in self.timers.items():
            if timer_id:
                self.clock.cancel_timer(timer_id)
                self.log.info(f"取消定时器: {timer_name}")

        # 生成最终报告
        self._generate_final_report()

    def _generate_final_report(self):
        """生成最终报告"""
        report = f"""
策略执行报告
================

运行时间: {self.clock.timestamp_ns() / 1_000_000_000:.0f} 秒
事件总数: {len(self.event_timeline)}
信号总数: {len(self.signal_history)}

市场状态
--------
趋势: {self.market_state['trend']}
波动率: {self.market_state['volatility']}
流动性: {self.market_state['liquidity']}

性能指标
--------
日盈亏: {self.performance_metrics['daily_pnl']} USDT
最大回撤: {self.performance_metrics['max_drawdown']:.1%}
胜率: {self.performance_metrics['win_rate']:.1%}
夏普比率: {self.performance_metrics['sharpe_ratio']}

最近事件
--------
"""

        # 添加最近5个事件
        for event in self.event_timeline[-5:]:
            report += f"- {event}\n"

        self.log.info(report)
```

## 6.5 调试和监控

### 6.5.1 事件追踪工具

```python
"""
事件追踪和调试工具
"""

import json
from datetime import datetime
from typing import Dict, List, Optional
from collections import defaultdict, deque


class EventTracker:
    """事件追踪器"""

    def __init__(self, max_events: int = 10000):
        """
        初始化追踪器

        Parameters
        ----------
        max_events : int
            最大事件数
        """
        self.max_events = max_events
        self.events = deque(maxlen=max_events)
        self.event_stats = defaultdict(int)
        self.event_timeline = []

    def track_event(self, event: Event, source: str = "unknown"):
        """
        追踪事件

        Parameters
        ----------
        event : Event
            要追踪的事件
        source : str
            事件来源
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

        # 添加到时间线
        self.event_timeline.append({
            'timestamp': event.ts_event,
            'event_type': event_info['type'],
            'source': source,
        })

    def _serialize_event(self, event: Event) -> Dict:
        """序列化事件数据"""
        data = {}

        # 提取常用属性
        if hasattr(event, 'trader_id'):
            data['trader_id'] = str(event.trader_id)
        if hasattr(event, 'strategy_id'):
            data['strategy_id'] = str(event.strategy_id)
        if hasattr(event, 'instrument_id'):
            data['instrument_id'] = str(event.instrument_id)

        # 对于自定义事件，添加额外属性
        if hasattr(event, 'metadata'):
            data['metadata'] = event.metadata

        return data

    def get_events_by_type(self, event_type: str) -> List[Dict]:
        """获取指定类型的事件"""
        return [e for e in self.events if e['type'] == event_type]

    def get_events_by_timerange(self, start_ns: int, end_ns: int) -> List[Dict]:
        """获取指定时间范围内的事件"""
        return [
            e for e in self.events
            if start_ns <= e['timestamp'] <= end_ns
        ]

    def get_event_sequence(self, event_id: str) -> List[Dict]:
        """获取事件序列"""
        sequence = []
        found = False

        for event in self.events:
            if event['id'] == event_id:
                found = True
            if found:
                sequence.append(event)

        return sequence

    def analyze_performance(self) -> Dict:
        """分析性能"""
        total_events = len(self.events)
        event_types = list(self.event_stats.keys())

        # 计算事件频率
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
            'memory_usage': len(self.events) * 100 / self.max_events,  # 百分比
        }

    def export_to_file(self, filename: str):
        """导出到文件"""
        data = {
            'export_time': datetime.now().isoformat(),
            'stats': self.analyze_performance(),
            'events': list(self.events),
            'timeline': self.event_timeline,
        }

        with open(filename, 'w') as f:
            json.dump(data, f, indent=2, default=str)

    def visualize_timeline(self):
        """可视化时间线（简化版）"""
        print("\n事件时间线:")
        print("-" * 80)

        last_time = None
        for event in self.event_timeline[-20:]:  # 显示最近20个事件
            time_str = datetime.fromtimestamp(
                event['timestamp'] / 1_000_000_000
            ).strftime('%H:%M:%S')

            if last_time:
                time_diff = (event['timestamp'] - last_time) / 1_000_000_000
                time_str += f" (+{time_diff:.1f}s)"

            print(f"{time_str}: {event['event_type']} from {event['source']}")
            last_time = event['timestamp']


class PerformanceProfiler:
    """性能分析器"""

    def __init__(self):
        """初始化分析器"""
        self.function_times = defaultdict(list)
        self.event_processing_times = defaultdict(list)

    def profile_function(self, func_name: str, execution_time_ns: int):
        """记录函数执行时间"""
        self.function_times[func_name].append(execution_time_ns)

    def profile_event_processing(
        self,
        event_type: str,
        processing_time_ns: int
    ):
        """记录事件处理时间"""
        self.event_processing_times[event_type].append(processing_time_ns)

    def get_function_stats(self) -> Dict:
        """获取函数统计"""
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
        """获取事件处理统计"""
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
        """找出性能瓶颈"""
        bottlenecks = []

        # 函数瓶颈
        func_stats = self.get_function_stats()
        for func, stats in func_stats.items():
            if stats['avg_time_ns'] > 1_000_000:  # 超过1ms
                bottlenecks.append({
                    'type': 'function',
                    'name': func,
                    'avg_time_ms': stats['avg_time_ns'] / 1_000_000,
                    'issue': '函数执行时间过长'
                })

        # 事件处理瓶颈
        event_stats = self.get_event_processing_stats()
        for event_type, stats in event_stats.items():
            if stats['avg_time_ns'] > 1_000_000:  # 超过1ms
                bottlenecks.append({
                    'type': 'event',
                    'name': event_type,
                    'avg_time_ms': stats['avg_time_ns'] / 1_000_000,
                    'issue': '事件处理时间过长'
                })

        # 按时间排序
        bottlenecks.sort(key=lambda x: x['avg_time_ms'], reverse=True)

        return bottlenecks


# 使用示例
def main():
    """主函数示例"""

    # 创建事件追踪器
    tracker = EventTracker()
    profiler = PerformanceProfiler()

    # 创建策略
    strategy = ComprehensiveStrategy()

    # 模拟运行
    import time
    start_time = time.time()

    # 模拟事件处理
    for i in range(100):
        # 创建模拟事件
        event = SignalEvent(
            trader_id=TraderId("TEST-001"),
            strategy_id=StrategyId("TEST-STRATEGY"),
            instrument_id="BTCUSDT.BINANCE",
            signal_type="BUY",
            confidence=0.8,
        )

        # 追踪事件
        start = time.time_ns()
        strategy._handle_signal_event(event)
        end = time.time_ns()

        tracker.track_event(event, "simulator")
        profiler.profile_event_processing("SignalEvent", end - start)

        # 模拟函数调用
        profiler.profile_function("market_analysis", time.time_ns() // 100)

    # 生成报告
    print("\n事件追踪报告:")
    print("=" * 50)

    stats = tracker.analyze_performance()
    print(f"总事件数: {stats['total_events']}")
    print(f"事件频率: {stats['event_frequency']:.2f} events/s")

    print("\n性能分析:")
    print("=" * 50)

    bottlenecks = profiler.find_bottlenecks()
    if bottlenecks:
        print("发现性能瓶颈:")
        for bottleneck in bottlenecks[:5]:
            print(f"- {bottleneck['name']}: {bottleneck['avg_time_ms']:.2f}ms "
                  f"({bottleneck['issue']})")
    else:
        print("未发现明显性能瓶颈")

    # 导出数据
    tracker.export_to_file("event_trace.json")

    # 显示时间线
    tracker.visualize_timeline()


if __name__ == "__main__":
    main()
```

## 6.6 下一步

在本章中，我们深入学习了：

1. NautilusTrader 的时间管理机制
2. 定时器的使用和高级模式
3. 事件驱动架构和最佳实践
4. 事件处理和性能优化
5. 调试和监控工具

在下一章中，我们将学习：

1. 技术指标系统
2. 内置指标的使用
3. 自定义指标开发
4. 指标级联和组合
5. 指标性能优化

## 6.7 总结

### 关键要点
1. NautilusTrader 提供了灵活的时间管理系统
2. 定时器支持复杂调度逻辑
3. 事件驱动架构实现了松耦合设计
4. 事件总线支持高性能的异步处理

### 最佳实践
1. 合理使用定时器，避免过度调度
2. 事件处理要快速，避免阻塞
3. 使用事件追踪工具调试问题
4. 定期分析性能瓶颈

## 6.8 参考资料

- [NautilusTrader 事件系统文档](https://nautilustrader.io/docs/latest/concepts/message_bus/)
- [Python 异步编程指南](https://docs.python.org/3/library/asyncio.html)
- [事件驱动架构模式](https://en.wikipedia.org/wiki/Event-driven_architecture)