# 第18章：自定义组件开发

## 学习目标

- 理解NautilusTrader的组件架构和扩展机制
- 学会开发自定义策略、执行客户端和数据适配器
- 掌握插件的开发和使用方法
- 实现可复用的交易组件

## 18.1 自定义策略开发

### 18.1.1 策略基类和生命周期

```python
from nautilus_trader.strategy import Strategy
from nautilus_trader.model.data import QuoteTick, TradeTick, Bar
from nautilus_trader.model.events import OrderFilled, PositionClosed
from nautilus_trader.model.identifiers import InstrumentId
from nautilus_trader.model.enums import OrderSide, OrderType
from nautilus_trader.model.objects import Quantity, Price
from nautilus_trader.core.message import Event
from decimal import Decimal
import asyncio
import logging

class CustomStrategyTemplate(Strategy):
    """自定义策略模板"""

    # 参数配置
    def __init__(self, config):
        super().__init__()

        # 必需配置
        self.instrument_id = InstrumentId.from_str(config['instrument_id'])

        # 可选配置
        self.order_size = Quantity.from_int(config.get('order_size', 100))
        self.max_orders = config.get('max_orders', 5)
        self.order_timeout_seconds = config.get('order_timeout', 60)

        # 状态变量
        self.is_running = False
        self.active_orders = {}
        self.position = None
        self.last_price = None

        # 性能统计
        self.order_count = 0
        self.filled_count = 0
        self.total_pnl = Decimal('0')

    # ==================== 生命周期方法 ====================

    def on_start(self):
        """策略启动时调用"""
        self.is_running = True

        # 订阅数据流
        self.subscribe_quote_ticks(self.instrument_id)
        self.subscribe_trade_ticks(self.instrument_id)

        # 订阅K线（如果需要）
        if hasattr(self, 'bar_type'):
            self.subscribe_bars(self.bar_type)

        # 初始化定时器
        self._setup_timers()

        self.log.info(f"Strategy {self.__class__.__name__} started for {self.instrument_id}")

    def on_stop(self):
        """策略停止时调用"""
        self.is_running = False

        # 取消所有活动订单
        asyncio.create_task(self._cancel_all_orders())

        self.log.info(f"Strategy {self.__class__.__name__} stopped")

    # ==================== 数据处理方法 ====================

    def on_quote_tick(self, tick: QuoteTick):
        """处理报价数据"""
        self.last_price = (tick.bid + tick.ask) / 2

        # 实现策略逻辑
        if self.is_running:
            self._process_quote_tick(tick)

    def on_trade_tick(self, tick: TradeTick):
        """处理成交数据"""
        # 实现策略逻辑
        if self.is_running:
            self._process_trade_tick(tick)

    def on_bar(self, bar: Bar):
        """处理K线数据"""
        # 实现策略逻辑
        if self.is_running:
            self._process_bar(bar)

    # ==================== 订单事件处理 ====================

    def on_order_submitted(self, event):
        """订单提交事件"""
        order_id = event.order_id
        self.active_orders[order_id] = {
            'status': 'submitted',
            'submitted_at': event.ts_event
        }
        self.order_count += 1

    def on_order_accepted(self, event):
        """订单接受事件"""
        order_id = event.order_id
        if order_id in self.active_orders:
            self.active_orders[order_id]['status'] = 'accepted'
            self.active_orders[order_id]['accepted_at'] = event.ts_event

    def on_order_rejected(self, event):
        """订单拒绝事件"""
        order_id = event.order_id
        if order_id in self.active_orders:
            del self.active_orders[order_id]

        self.log.warning(f"Order rejected: {event.order_id}")

    def on_order_filled(self, event: OrderFilled):
        """订单成交通知"""
        order_id = event.order_id
        if order_id in self.active_orders:
            self.active_orders[order_id]['status'] = 'filled'
            self.active_orders[order_id]['filled_at'] = event.ts_event
            self.active_orders[order_id]['fill_price'] = event.last_px

        self.filled_count += 1

        # 处理成交
        self._handle_order_filled(event)

    def on_order_canceled(self, event):
        """订单取消事件"""
        order_id = event.order_id
        if order_id in self.active_orders:
            del self.active_orders[order_id]

    # ==================== 持仓事件处理 ====================

    def on_position_opened(self, event):
        """持仓开启事件"""
        self.position = event.position
        self.log.info(f"Position opened: {self.position}")

    def on_position_changed(self, event):
        """持仓变化事件"""
        self.position = event.position

    def on_position_closed(self, event: PositionClosed):
        """持仓关闭事件"""
        # 更新PnL
        self.total_pnl += event.pnl
        self.position = None

        self.log.info(f"Position closed. PnL: {event.pnl}")

    # ==================== 自定义方法 ====================

    def _setup_timers(self):
        """设置定时器"""
        # 示例：每分钟检查一次
        self.set_timer("strategy_check", 60.0, self._strategy_check)

    async def _strategy_check(self):
        """策略定期检查"""
        # 清理超时订单
        await self._cleanup_expired_orders()

        # 其他定期检查逻辑
        pass

    def _process_quote_tick(self, tick: QuoteTick):
        """处理报价数据的子类实现"""
        raise NotImplementedError("Subclass must implement _process_quote_tick")

    def _process_trade_tick(self, tick: TradeTick):
        """处理成交数据的子类实现"""
        pass

    def _process_bar(self, bar: Bar):
        """处理K线数据的子类实现"""
        pass

    def _handle_order_filled(self, event: OrderFilled):
        """处理订单成交的子类实现"""
        pass

    async def _place_order(self, side: OrderSide, quantity: Quantity,
                           order_type: OrderType = OrderType.MARKET,
                           price: Price = None):
        """下订单的通用方法"""
        # 检查订单限制
        if len(self.active_orders) >= self.max_orders:
            self.log.warning(f"Max orders limit reached: {self.max_orders}")
            return None

        # 创建订单
        if order_type == OrderType.MARKET:
            order = self.order_factory.market(
                instrument_id=self.instrument_id,
                side=side,
                quantity=quantity
            )
        else:  # LIMIT
            order = self.order_factory.limit(
                instrument_id=self.instrument_id,
                side=side,
                price=price,
                quantity=quantity
            )

        # 提交订单
        order_id = await self.submit_order(order)
        return order_id

    async def _cancel_order(self, order_id):
        """取消订单"""
        await self.cancel_order(order_id)

    async def _cancel_all_orders(self):
        """取消所有订单"""
        for order_id in list(self.active_orders.keys()):
            await self._cancel_order(order_id)

    async def _cleanup_expired_orders(self):
        """清理过期订单"""
        current_time = self.clock.timestamp()

        expired_orders = []
        for order_id, order_info in self.active_orders.items():
            if (current_time - order_info['submitted_at']) > self.order_timeout_seconds * 1e9:
                expired_orders.append(order_id)

        for order_id in expired_orders:
            self.log.info(f"Cancelling expired order: {order_id}")
            await self._cancel_order(order_id)

    def get_strategy_stats(self):
        """获取策略统计信息"""
        return {
            'is_running': self.is_running,
            'active_orders': len(self.active_orders),
            'total_orders': self.order_count,
            'filled_orders': self.filled_count,
            'fill_rate': self.filled_count / max(self.order_count, 1),
            'total_pnl': float(self.total_pnl),
            'current_position': self.position.quantity if self.position else 0
        }

# 使用示例
class MovingAverageCrossStrategy(CustomStrategyTemplate):
    """移动平均交叉策略示例"""

    def __init__(self, config):
        super().__init__(config)

        # 策略参数
        self.fast_period = config.get('fast_period', 10)
        self.slow_period = config.get('slow_period', 30)

        # 数据存储
        self.price_history = []
        self.fast_ma = None
        self.slow_ma = None
        self.last_signal = None  # 1: 多头, -1: 空头, 0: 无

    def _process_quote_tick(self, tick: QuoteTick):
        """处理报价数据"""
        # 更新价格历史
        price = float(tick.mid)
        self.price_history.append(price)

        # 保持固定长度
        max_history = self.slow_period * 2
        if len(self.price_history) > max_history:
            self.price_history.pop(0)

        # 计算移动平均
        if len(self.price_history) >= self.slow_period:
            self.fast_ma = sum(self.price_history[-self.fast_period:]) / self.fast_period
            self.slow_ma = sum(self.price_history[-self.slow_period:]) / self.slow_period

            # 生成交易信号
            self._generate_signals()

    def _generate_signals(self):
        """生成交易信号"""
        if self.fast_ma is None or self.slow_ma is None:
            return

        # 检查金叉（快线上穿慢线）
        if self.fast_ma > self.slow_ma and self.last_signal != 1:
            self._execute_buy_signal()
            self.last_signal = 1

        # 检查死叉（快线下穿慢线）
        elif self.fast_ma < self.slow_ma and self.last_signal != -1:
            self._execute_sell_signal()
            self.last_signal = -1

    async def _execute_buy_signal(self):
        """执行买入信号"""
        # 检查当前持仓
        if self.position and self.position.quantity > 0:
            return  # 已有多头持仓

        # 平掉空头持仓（如果有）
        if self.position and self.position.quantity < 0:
            close_size = Quantity.from_int(abs(self.position.quantity))
            await self._place_order(OrderSide.BUY, close_size)

        # 开多头仓位
        await self._place_order(OrderSide.BUY, self.order_size)
        self.log.info(f"Buy signal executed at price {self.last_price}")

    async def _execute_sell_signal(self):
        """执行卖出信号"""
        # 检查当前持仓
        if self.position and self.position.quantity < 0:
            return  # 已有空头持仓

        # 平掉多头持仓（如果有）
        if self.position and self.position.quantity > 0:
            close_size = Quantity.from_int(self.position.quantity)
            await self._place_order(OrderSide.SELL, close_size)

        # 开空头仓位
        await self._place_order(OrderSide.SELL, self.order_size)
        self.log.info(f"Sell signal executed at price {self.last_price}")
```

### 18.1.2 高级策略功能

```python
from abc import ABC, abstractmethod
from typing import Dict, List, Callable
import pandas as pd
import numpy as np

class StrategyState:
    """策略状态管理"""

    def __init__(self):
        self.data = {}
        self.history = []

    def set(self, key: str, value):
        """设置状态值"""
        old_value = self.data.get(key)
        self.data[key] = value

        # 记录变化历史
        if old_value != value:
            self.history.append({
                'timestamp': pd.Timestamp.now(),
                'key': key,
                'old_value': old_value,
                'new_value': value
            })

    def get(self, key: str, default=None):
        """获取状态值"""
        return self.data.get(key, default)

    def save_snapshot(self):
        """保存状态快照"""
        return {
            'data': self.data.copy(),
            'timestamp': pd.Timestamp.now()
        }

    def load_snapshot(self, snapshot: dict):
        """加载状态快照"""
        self.data = snapshot['data'].copy()

class RiskManager:
    """风险管理器"""

    def __init__(self, config: dict):
        self.max_position_size = config.get('max_position_size', 1000)
        self.max_daily_loss = config.get('max_daily_loss', 10000)
        self.max_drawdown = config.get('max_drawdown', 0.1)
        self.stop_loss_pct = config.get('stop_loss_pct', 0.02)
        self.take_profit_pct = config.get('take_profit_pct', 0.05)

        self.daily_pnl = 0
        self.max_portfolio_value = 0

    def check_order_risk(self, order_side: OrderSide, quantity: Quantity,
                         current_position: int) -> bool:
        """检查订单风险"""
        # 检查持仓限制
        if order_side == OrderSide.BUY:
            new_position = current_position + quantity.as_double()
        else:
            new_position = current_position - quantity.as_double()

        if abs(new_position) > self.max_position_size:
            return False

        return True

    def check_position_risk(self, position) -> dict:
        """检查持仓风险"""
        if not position:
            return {'action': 'none'}

        unrealized_pnl = position.unrealized_pnl(self.clock.timestamp())
        portfolio_value = self._get_portfolio_value()

        # 更新最高组合价值
        self.max_portfolio_value = max(self.max_portfolio_value, portfolio_value)

        # 计算回撤
        drawdown = (self.max_portfolio_value - portfolio_value) / self.max_portfolio_value

        # 检查止损
        if unrealized_pnl < -self.stop_loss_pct * portfolio_value:
            return {'action': 'stop_loss', 'reason': 'Stop loss triggered'}

        # 检查止盈
        if unrealized_pnl > self.take_profit_pct * portfolio_value:
            return {'action': 'take_profit', 'reason': 'Take profit triggered'}

        # 检查最大回撤
        if drawdown > self.max_drawdown:
            return {'action': 'reduce_position', 'reason': 'Max drawdown exceeded'}

        return {'action': 'none'}

    def _get_portfolio_value(self):
        """获取组合价值"""
        # 实现组合价值计算
        return 100000  # 示例值

class AdvancedStrategy(Strategy):
    """高级策略基类"""

    def __init__(self, config):
        super().__init__(config)

        # 状态管理
        self.state = StrategyState()

        # 风险管理
        self.risk_manager = RiskManager(config.get('risk', {}))

        # 信号生成器
        self.signal_generators = []

        # 订单管理器
        self.order_manager = OrderManager(self)

        # 性能跟踪
        self.performance_tracker = PerformanceTracker()

    def add_signal_generator(self, generator: 'SignalGenerator'):
        """添加信号生成器"""
        self.signal_generators.append(generator)

    def generate_signals(self) -> List[dict]:
        """生成所有信号"""
        signals = []
        for generator in self.signal_generators:
            signal = generator.generate(self)
            if signal:
                signals.append(signal)
        return signals

    def execute_signals(self, signals: List[dict]):
        """执行信号"""
        for signal in signals:
            # 风险检查
            if not self.risk_manager.check_order_risk(
                signal['side'],
                signal['quantity'],
                self.position.quantity if self.position else 0
            ):
                continue

            # 执行订单
            self.order_manager.place_order(signal)

class SignalGenerator(ABC):
    """信号生成器基类"""

    @abstractmethod
    def generate(self, strategy: AdvancedStrategy) -> dict:
        """生成信号"""
        pass

class MeanReversionSignal(SignalGenerator):
    """均值回归信号生成器"""

    def __init__(self, lookback: int = 20, threshold: float = 2.0):
        self.lookback = lookback
        self.threshold = threshold

    def generate(self, strategy: AdvancedStrategy) -> dict:
        """生成均值回归信号"""
        if not hasattr(strategy, 'price_history') or len(strategy.price_history) < self.lookback:
            return None

        prices = strategy.price_history[-self.lookback:]
        mean_price = np.mean(prices)
        std_price = np.std(prices)
        current_price = strategy.price_history[-1]

        # 计算Z-score
        zscore = (current_price - mean_price) / std_price if std_price > 0 else 0

        # 生成信号
        if zscore > self.threshold:  # 价格过高，卖出
            return {
                'type': 'sell',
                'strength': min(zscore / self.threshold, 3.0),
                'reason': f'Price {zscore:.2f} std above mean'
            }
        elif zscore < -self.threshold:  # 价格过低，买入
            return {
                'type': 'buy',
                'strength': min(abs(zscore) / self.threshold, 3.0),
                'reason': f'Price {abs(zscore):.2f} std below mean'
            }

        return None

class OrderManager:
    """订单管理器"""

    def __init__(self, strategy: Strategy):
        self.strategy = strategy
        self.orders = {}
        self.order_groups = {}  # 订单组管理

    def place_order(self, signal: dict):
        """下订单"""
        order_config = {
            'side': OrderSide.BUY if signal['type'] == 'buy' else OrderSide.SELL,
            'quantity': self._calculate_quantity(signal),
            'order_type': OrderType.MARKET
        }

        # 创建并提交订单
        order = self.strategy.order_factory.market(
            instrument_id=self.strategy.instrument_id,
            side=order_config['side'],
            quantity=order_config['quantity']
        )

        # 添加到订单组
        group_id = signal.get('group_id', 'default')
        if group_id not in self.order_groups:
            self.order_groups[group_id] = []
        self.order_groups[group_id].append(order)

        # 提交订单
        asyncio.create_task(self.strategy.submit_order(order))

    def _calculate_quantity(self, signal: dict) -> Quantity:
        """计算订单数量"""
        # 基于信号强度调整数量
        base_quantity = self.strategy.order_size.as_double()
        strength = signal.get('strength', 1.0)

        adjusted_quantity = int(base_quantity * strength)
        return Quantity.from_int(max(1, adjusted_quantity))

class PerformanceTracker:
    """性能跟踪器"""

    def __init__(self):
        self.trades = []
        self.pnl_history = []
        self.metrics = {}

    def record_trade(self, trade: dict):
        """记录交易"""
        self.trades.append(trade)

    def update_metrics(self):
        """更新性能指标"""
        if not self.trades:
            return

        # 计算各种指标
        win_rate = len([t for t in self.trades if t['pnl'] > 0]) / len(self.trades)
        avg_win = np.mean([t['pnl'] for t in self.trades if t['pnl'] > 0]) if any(t['pnl'] > 0 for t in self.trades) else 0
        avg_loss = np.mean([t['pnl'] for t in self.trades if t['pnl'] < 0]) if any(t['pnl'] < 0 for t in self.trades) else 0
        profit_factor = avg_win / abs(avg_loss) if avg_loss != 0 else float('inf')

        self.metrics = {
            'total_trades': len(self.trades),
            'win_rate': win_rate,
            'avg_win': avg_win,
            'avg_loss': avg_loss,
            'profit_factor': profit_factor
        }
```

## 18.2 自定义执行客户端

### 18.2.1 执行客户端基类

```python
from nautilus_trader.execution.client import ExecutionClient
from nautilus_trader.model.enums import LiquiditySide, OrderSide
from nautilus_trader.model.objects import Quantity, Price
from nautilus_trader.core.uuid import UUID4

class CustomExecutionClient(ExecutionClient):
    """自定义执行客户端基类"""

    def __init__(self, config):
        super().__init__(
            name=config.get('name', 'CustomExecClient'),
            config=config
        )

        # 客户端配置
        self.base_url = config.get('base_url')
        self.api_key = config.get('api_key')
        self.api_secret = config.get('api_secret')

        # 状态
        self.connected = False
        self.account_id = None
        self.order_id_map = {}  # 映射内部订单ID到交易所订单ID

    async def connect(self):
        """连接到交易所"""
        try:
            # 实现连接逻辑
            await self._authenticate()

            # 获取账户信息
            account_info = await self._get_account_info()
            self.account_id = account_info['id']

            # 启动数据流
            await self._start_data_streams()

            self.connected = True
            self.log.info(f"Connected to {self.__class__.__name__}")

        except Exception as e:
            self.log.error(f"Failed to connect: {e}")
            raise

    async def disconnect(self):
        """断开连接"""
        if self.connected:
            await self._stop_data_streams()
            self.connected = False
            self.log.info("Disconnected from exchange")

    async def submit_order(self, order):
        """提交订单"""
        if not self.connected:
            raise RuntimeError("Not connected to exchange")

        # 转换订单格式
        exchange_order = self._convert_order(order)

        try:
            # 提交到交易所
            response = await self._send_order_to_exchange(exchange_order)

            # 映射订单ID
            self.order_id_map[order.client_order_id] = response['order_id']

            # 生成事件
            self._generate_order_submitted_event(order)

        except Exception as e:
            self._generate_order_rejected_event(order, str(e))
            raise

    async def cancel_order(self, order):
        """取消订单"""
        if not self.connected:
            raise RuntimeError("Not connected to exchange")

        exchange_order_id = self.order_id_map.get(order.client_order_id)
        if not exchange_order_id:
            raise ValueError(f"Order {order.client_order_id} not found")

        try:
            await self._cancel_order_on_exchange(exchange_order_id)
            self._generate_order_canceled_event(order)
        except Exception as e:
            self.log.error(f"Failed to cancel order {order.client_order_id}: {e}")
            raise

    async def modify_order(self, order):
        """修改订单（如果支持）"""
        # 大部分交易所不支持直接修改，需要先取消再新建
        await self.cancel_order(order)
        await self.submit_order(order)

    # ==================== 抽象方法（子类实现） ====================

    @abstractmethod
    async def _authenticate(self):
        """认证"""
        pass

    @abstractmethod
    async def _get_account_info(self) -> dict:
        """获取账户信息"""
        pass

    @abstractmethod
    async def _start_data_streams(self):
        """启动数据流"""
        pass

    @abstractmethod
    async def _stop_data_streams(self):
        """停止数据流"""
        pass

    @abstractmethod
    def _convert_order(self, order) -> dict:
        """转换订单格式"""
        pass

    @abstractmethod
    async def _send_order_to_exchange(self, order: dict) -> dict:
        """发送订单到交易所"""
        pass

    @abstractmethod
    async def _cancel_order_on_exchange(self, order_id: str):
        """在交易所取消订单"""
        pass

    # ==================== 辅助方法 ====================

    def _generate_order_submitted_event(self, order):
        """生成订单提交事件"""
        from nautilus_trader.model.events import OrderSubmitted

        event = OrderSubmitted(
            trader_id=self.trader_id,
            strategy_id=self.strategy_id,
            account_id=self.account_id,
            order_id=order.client_order_id,
            ts_event=self.clock.timestamp_ns()
        )
        self.cache.add(event)
        self.msgbus.publish(topic=f"events.order.{order.client_order_id}", msg=event)

    def _generate_order_rejected_event(self, order, reason: str):
        """生成订单拒绝事件"""
        from nautilus_trader.model.events import OrderRejected

        event = OrderRejected(
            trader_id=self.trader_id,
            strategy_id=self.strategy_id,
            account_id=self.account_id,
            order_id=order.client_order_id,
            reason=reason,
            ts_event=self.clock.timestamp_ns()
        )
        self.cache.add(event)
        self.msgbus.publish(topic=f"events.order.{order.client_order_id}", msg=event)

    def _generate_order_canceled_event(self, order):
        """生成订单取消事件"""
        from nautilus_trader.model.events import OrderCanceled

        event = OrderCanceled(
            trader_id=self.trader_id,
            strategy_id=self.strategy_id,
            account_id=self.account_id,
            order_id=order.client_order_id,
            ts_event=self.clock.timestamp_ns()
        )
        self.cache.add(event)
        self.msgbus.publish(topic=f"events.order.{order.client_order_id}", msg=event)

class WebSocketExecutionClient(CustomExecutionClient):
    """WebSocket执行客户端"""

    def __init__(self, config):
        super().__init__(config)

        self.ws_url = config.get('ws_url')
        self.websocket = None
        self.ws_connected = False

    async def _start_data_streams(self):
        """启动WebSocket数据流"""
        import websockets

        self.websocket = await websockets.connect(self.ws_url)
        self.ws_connected = True

        # 启动消息处理任务
        asyncio.create_task(self._handle_ws_messages())

    async def _stop_data_streams(self):
        """停止WebSocket数据流"""
        if self.websocket:
            await self.websocket.close()
            self.ws_connected = False

    async def _handle_ws_messages(self):
        """处理WebSocket消息"""
        while self.ws_connected:
            try:
                message = await self.websocket.recv()
                await self._process_ws_message(message)
            except Exception as e:
                self.log.error(f"WebSocket message error: {e}")
                break

    async def _process_ws_message(self, message: str):
        """处理WebSocket消息"""
        # 解析消息
        data = json.loads(message)

        # 根据消息类型处理
        if data['type'] == 'execution':
            await self._handle_execution_report(data)
        elif data['type'] == 'account':
            await self._handle_account_update(data)

    async def _handle_execution_report(self, data: dict):
        """处理执行报告"""
        order_id = self._find_internal_order_id(data['exchange_order_id'])

        if data['status'] == 'filled':
            await self._handle_order_filled(order_id, data)
        elif data['status'] == 'canceled':
            await self._handle_order_canceled(order_id, data)

    async def _handle_order_filled(self, order_id: str, data: dict):
        """处理订单成交"""
        # 创建OrderFilled事件
        from nautilus_trader.model.events import OrderFilled

        event = OrderFilled(
            trader_id=self.trader_id,
            strategy_id=self.strategy_id,
            account_id=self.account_id,
            order_id=UUID4(order_id),
            venue_order_id=data['exchange_order_id'],
            symbol=self.instrument_id,
            side=OrderSide[data['side'].upper()],
            order_type=OrderType[data['type'].upper()],
            filled_qty=Quantity.from_str(data['quantity']),
            last_px=Price.from_str(data['price']),
            last_qty=Quantity.from_str(data['quantity']),
            ts_event=pd.Timestamp(data['timestamp']).value,
            ts_init=self.clock.timestamp_ns()
        )

        self.cache.add(event)
        self.msgbus.publish(topic=f"events.order.{order_id}", msg=event)

    def _find_internal_order_id(self, exchange_order_id: str) -> str:
        """根据交易所订单ID查找内部订单ID"""
        for internal_id, ex_id in self.order_id_map.items():
            if ex_id == exchange_order_id:
                return internal_id
        return None
```

## 18.3 数据适配器开发

### 18.3.1 自定义数据源

```python
from nautilus_trader.adapters.betfair.providers import BetfairInstrumentProvider
from nautilus_trader.model.data import QuoteTick, TradeTick, BarType, Bar
from nautilus_trader.model.identifiers import InstrumentId, Symbol, Venue
from nautilus_trader.core.datetime import dt_to_unix_nanos
import asyncio
import websockets

class CustomDataAdapter:
    """自定义数据适配器基类"""

    def __init__(self, name: str, config: dict):
        self.name = name
        self.config = config

        # 状态
        self.is_connected = False
        self.subscribed_symbols = set()
        self.callbacks = {}

        # 数据缓存
        self.quote_cache = {}
        self.trade_cache = {}
        self.bar_cache = {}

    async def connect(self):
        """连接数据源"""
        try:
            await self._connect_to_source()
            self.is_connected = True
            self.log.info(f"Connected to {self.name} data source")
        except Exception as e:
            self.log.error(f"Failed to connect: {e}")
            raise

    async def disconnect(self):
        """断开连接"""
        if self.is_connected:
            await self._disconnect_from_source()
            self.is_connected = False

    async def subscribe_quotes(self, symbols: List[str], callback):
        """订阅报价数据"""
        for symbol in symbols:
            self.subscribed_symbols.add(symbol)
            if 'quote' not in self.callbacks:
                self.callbacks['quote'] = []
            self.callbacks['quote'].append((symbol, callback))

        await self._subscribe_to_quotes(symbols)

    async def subscribe_trades(self, symbols: List[str], callback):
        """订阅成交数据"""
        for symbol in symbols:
            self.subscribed_symbols.add(symbol)
            if 'trade' not in self.callbacks:
                self.callbacks['trade'] = []
            self.callbacks['trade'].append((symbol, callback))

        await self._subscribe_to_trades(symbols)

    async def subscribe_bars(self, bar_type: BarType, callback):
        """订阅K线数据"""
        symbol = str(bar_type.instrument_id)
        self.subscribed_symbols.add(symbol)
        if 'bar' not in self.callbacks:
            self.callbacks['bar'] = []
        self.callbacks['bar'].append((bar_type, callback))

        await self._subscribe_to_bars(bar_type)

    # ==================== 抽象方法 ====================

    @abstractmethod
    async def _connect_to_source(self):
        """连接到数据源"""
        pass

    @abstractmethod
    async def _disconnect_from_source(self):
        """断开连接"""
        pass

    @abstractmethod
    async def _subscribe_to_quotes(self, symbols: List[str]):
        """订阅报价数据"""
        pass

    @abstractmethod
    async def _subscribe_to_trades(self, symbols: List[str]):
        """订阅成交数据"""
        pass

    @abstractmethod
    async def _subscribe_to_bars(self, bar_type: BarType):
        """订阅K线数据"""
        pass

class WebSocketDataAdapter(CustomDataAdapter):
    """WebSocket数据适配器"""

    def __init__(self, name: str, config: dict):
        super().__init__(name, config)

        self.ws_url = config.get('ws_url')
        self.websocket = None
        self.data_handlers = {
            'quote': self._handle_quote_data,
            'trade': self._handle_trade_data,
            'bar': self._handle_bar_data
        }

    async def _connect_to_source(self):
        """连接WebSocket"""
        self.websocket = await websockets.connect(self.ws_url)

        # 启动消息处理循环
        asyncio.create_task(self._message_loop())

    async def _disconnect_from_source(self):
        """断开WebSocket连接"""
        if self.websocket:
            await self.websocket.close()

    async def _message_loop(self):
        """消息循环"""
        while True:
            try:
                message = await self.websocket.recv()
                data = json.loads(message)
                await self._process_message(data)
            except Exception as e:
                self.log.error(f"Message processing error: {e}")
                break

    async def _process_message(self, data: dict):
        """处理消息"""
        message_type = data.get('type')

        if message_type in self.data_handlers:
            await self.data_handlers[message_type](data)

    async def _handle_quote_data(self, data: dict):
        """处理报价数据"""
        # 创建QuoteTick
        quote_tick = self._create_quote_tick(data)

        # 调用回调
        for symbol, callback in self.callbacks.get('quote', []):
            if symbol == data['symbol']:
                await callback(quote_tick)

    async def _handle_trade_data(self, data: dict):
        """处理成交数据"""
        # 创建TradeTick
        trade_tick = self._create_trade_tick(data)

        # 调用回调
        for symbol, callback in self.callbacks.get('trade', []):
            if symbol == data['symbol']:
                await callback(trade_tick)

    async def _handle_bar_data(self, data: dict):
        """处理K线数据"""
        # 创建Bar
        bar = self._create_bar(data)

        # 调用回调
        for bar_type, callback in self.callbacks.get('bar', []):
            if str(bar_type.instrument_id) == data['symbol']:
                await callback(bar)

class FileDataAdapter(CustomDataAdapter):
    """文件数据适配器（用于回测）"""

    def __init__(self, name: str, config: dict):
        super().__init__(name, config)

        self.file_path = config.get('file_path')
        self.data_type = config.get('data_type', 'csv')  # csv, parquet, json
        self.speed_factor = config.get('speed_factor', 1.0)  # 回放速度

        self.data_iterator = None
        self.start_time = None

    async def _connect_to_source(self):
        """连接数据源"""
        # 加载数据
        if self.data_type == 'csv':
            df = pd.read_csv(self.file_path)
        elif self.data_type == 'parquet':
            df = pd.read_parquet(self.file_path)
        else:
            raise ValueError(f"Unsupported data type: {self.data_type}")

        # 创建迭代器
        self.data_iterator = df.iterrows()
        self.start_time = pd.Timestamp.now()

    async def _disconnect_from_source(self):
        """断开连接"""
        self.data_iterator = None

    async def play_data(self):
        """回放数据"""
        if not self.data_iterator:
            return

        previous_timestamp = None

        for _, row in self.data_iterator:
            # 解析时间戳
            timestamp = pd.Timestamp(row['timestamp'])

            # 计算等待时间（模拟实时）
            if previous_timestamp:
                time_diff = (timestamp - previous_timestamp).total_seconds()
                await asyncio.sleep(time_diff / self.speed_factor)

            # 处理数据行
            if 'bid' in row and 'ask' in row:
                await self._process_quote_row(row)
            elif 'price' in row and 'size' in row:
                await self._process_trade_row(row)

            previous_timestamp = timestamp

    async def _process_quote_row(self, row):
        """处理报价行"""
        quote_tick = QuoteTick(
            instrument_id=InstrumentId.from_str(f"{row['symbol']}.{self.name}"),
            bid=Price.from_str(str(row['bid'])),
            ask=Price.from_str(str(row['ask'])),
            bid_size=Quantity.from_str(str(row.get('bid_size', 1))),
            ask_size=Quantity.from_str(str(row.get('ask_size', 1))),
            ts_event=dt_to_unix_nanos(row['timestamp']),
            ts_init=self.clock.timestamp_ns()
        )

        # 触发回调
        for symbol, callback in self.callbacks.get('quote', []):
            if symbol == row['symbol']:
                await callback(quote_tick)
```

## 18.4 插件开发

### 18.4.1 插件系统

```python
from abc import ABC, abstractmethod
import importlib
import inspect

class Plugin(ABC):
    """插件基类"""

    def __init__(self, name: str, version: str):
        self.name = name
        self.version = version
        self.enabled = True
        self.config = {}

    @abstractmethod
    async def initialize(self, config: dict):
        """初始化插件"""
        pass

    @abstractmethod
    async def start(self):
        """启动插件"""
        pass

    @abstractmethod
    async def stop(self):
        """停止插件"""
        pass

    async def cleanup(self):
        """清理资源"""
        pass

class PluginManager:
    """插件管理器"""

    def __init__(self):
        self.plugins = {}
        self.plugin_configs = {}
        self.event_handlers = {}

    def load_plugin(self, plugin_path: str, config: dict = None):
        """加载插件"""
        try:
            # 动态导入插件模块
            spec = importlib.util.spec_from_file_location("plugin", plugin_path)
            module = importlib.util.module_from_spec(spec)
            spec.loader.exec_module(module)

            # 查找插件类
            plugin_class = None
            for name, obj in inspect.getmembers(module):
                if inspect.isclass(obj) and issubclass(obj, Plugin) and obj != Plugin:
                    plugin_class = obj
                    break

            if not plugin_class:
                raise ValueError(f"No plugin class found in {plugin_path}")

            # 创建插件实例
            plugin = plugin_class(
                name=config.get('name', plugin_class.__name__),
                version=config.get('version', '1.0.0')
            )

            # 初始化插件
            asyncio.create_task(plugin.initialize(config or {}))

            self.plugins[plugin.name] = plugin
            self.plugin_configs[plugin.name] = config or {}

            self.log.info(f"Plugin loaded: {plugin.name}")

        except Exception as e:
            self.log.error(f"Failed to load plugin {plugin_path}: {e}")
            raise

    async def start_plugin(self, plugin_name: str):
        """启动插件"""
        if plugin_name not in self.plugins:
            raise ValueError(f"Plugin {plugin_name} not found")

        plugin = self.plugins[plugin_name]
        if plugin.enabled:
            await plugin.start()
            self.log.info(f"Plugin started: {plugin_name}")

    async def stop_plugin(self, plugin_name: str):
        """停止插件"""
        if plugin_name not in self.plugins:
            raise ValueError(f"Plugin {plugin_name} not found")

        plugin = self.plugins[plugin_name]
        await plugin.stop()
        self.log.info(f"Plugin stopped: {plugin_name}")

    async def start_all_plugins(self):
        """启动所有插件"""
        for plugin_name in self.plugins:
            await self.start_plugin(plugin_name)

    async def stop_all_plugins(self):
        """停止所有插件"""
        for plugin_name in self.plugins:
            await self.stop_plugin(plugin_name)

    def get_plugin(self, plugin_name: str) -> Plugin:
        """获取插件"""
        return self.plugins.get(plugin_name)

    def register_event_handler(self, event_type: str, handler: Callable):
        """注册事件处理器"""
        if event_type not in self.event_handlers:
            self.event_handlers[event_type] = []
        self.event_handlers[event_type].append(handler)

    async def emit_event(self, event_type: str, data: dict):
        """发送事件"""
        if event_type in self.event_handlers:
            for handler in self.event_handlers[event_type]:
                try:
                    await handler(data)
                except Exception as e:
                    self.log.error(f"Event handler error: {e}")

# 示例插件
class RiskMonitorPlugin(Plugin):
    """风险监控插件"""

    def __init__(self, name: str, version: str):
        super().__init__(name, version)
        self.risk_limits = {}
        self.current_metrics = {}
        self.alerts = []

    async def initialize(self, config: dict):
        """初始化"""
        self.config = config
        self.risk_limits = config.get('risk_limits', {})

        # 注册事件处理器
        self.plugin_manager.register_event_handler('order_filled', self.on_order_filled)
        self.plugin_manager.register_event_handler('position_updated', self.on_position_updated)

    async def start(self):
        """启动"""
        # 启动监控任务
        asyncio.create_task(self.monitor_loop())

    async def stop(self):
        """停止"""
        pass

    async def monitor_loop(self):
        """监控循环"""
        while True:
            await self.check_risk_metrics()
            await asyncio.sleep(1)  # 每秒检查一次

    async def check_risk_metrics(self):
        """检查风险指标"""
        # 实现风险检查逻辑
        pass

    async def on_order_filled(self, data: dict):
        """订单成交事件处理"""
        # 更新风险指标
        pass

    async def on_position_updated(self, data: dict):
        """持仓更新事件处理"""
        # 更新风险指标
        pass

class PerformanceAnalyticsPlugin(Plugin):
    """性能分析插件"""

    def __init__(self, name: str, version: str):
        super().__init__(name, version)
        self.trade_history = []
        self.performance_metrics = {}

    async def initialize(self, config: dict):
        """初始化"""
        self.config = config

        # 注册事件处理器
        self.plugin_manager.register_event_handler('trade_executed', self.on_trade_executed)

    async def start(self):
        """启动"""
        pass

    async def stop(self):
        """停止"""
        # 保存分析报告
        await self.save_report()

    async def on_trade_executed(self, data: dict):
        """交易执行事件处理"""
        self.trade_history.append(data)

        # 实时更新性能指标
        self.update_performance_metrics()

    def update_performance_metrics(self):
        """更新性能指标"""
        if not self.trade_history:
            return

        # 计算各种指标
        total_trades = len(self.trade_history)
        profitable_trades = len([t for t in self.trade_history if t['pnl'] > 0])
        win_rate = profitable_trades / total_trades

        total_pnl = sum(t['pnl'] for t in self.trade_history)

        self.performance_metrics = {
            'total_trades': total_trades,
            'win_rate': win_rate,
            'total_pnl': total_pnl,
            'avg_pnl_per_trade': total_pnl / total_trades
        }

    async def save_report(self):
        """保存分析报告"""
        report_path = self.config.get('report_path', 'performance_report.json')

        report = {
            'timestamp': pd.Timestamp.now().isoformat(),
            'metrics': self.performance_metrics,
            'trade_history': self.trade_history
        }

        with open(report_path, 'w') as f:
            json.dump(report, f, indent=2)
```

## 小结

本章详细介绍了NautilusTrader的自定义组件开发，涵盖了：

1. **自定义策略开发**
   - 策略生命周期管理
   - 事件处理机制
   - 高级策略功能

2. **自定义执行客户端**
   - 执行客户端架构
   - WebSocket连接
   - 订单管理

3. **数据适配器开发**
   - 数据源适配
   - WebSocket数据处理
   - 文件数据回放

4. **插件系统**
   - 插件架构设计
   - 插件管理器
   - 示例插件实现

通过本章的学习，读者可以扩展NautilusTrader的功能，开发满足特定需求的自定义组件。

## 参考资料

- [NautilusTrader Extending Guide](https://docs.nautilustrader.io/extending.html)
- [Python Plugin Architecture](https://packaging.python.org/guides/creating-and-discovering-plugins/)
- [Design Patterns in Python](https://refactoring.guru/design-patterns/python)
## 本章可运行代码入口（源码版）

- 入口脚本：`docs/tutorials_cn/code/chapter18/run_custom_components.py`
- 关键示例：`docs/tutorials_cn/code/chapter18/custom_data_source.py`、`docs/tutorials_cn/code/chapter18/custom_execution.py`、`docs/tutorials_cn/code/chapter18/custom_indicators.py`

依赖策略说明：优先对齐 Nautilus Data/DataType 与内置指标思路，缺失能力时自动回退到纯 Python 实现。

已验证命令：

```bash
uv run --python 3.13 python docs/tutorials_cn/code/chapter18/run_custom_components.py
```
