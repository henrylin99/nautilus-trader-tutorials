# 第8章：订单管理系统

## 学习目标

通过本章学习，您将：
- 掌握各种订单类型的使用方法
- 理解订单生命周期和状态管理
- 学会高级订单策略（OCO、OTO等）
- 实现智能订单管理
- 掌握订单执行和风险管理

## 8.1 订单类型概览

### 8.1.1 基础订单类型

NautilusTrader 支持所有常见的订单类型：

```python
"""
基础订单类型示例
"""

from decimal import Decimal
from nautilus_trader.model.orders import (
    MarketOrder,
    LimitOrder,
    StopMarketOrder,
    StopLimitOrder,
)
from nautilus_trader.model.enums import (
    OrderSide,
    TimeInForce,
    OrderStatus,
)
from nautilus_trader.model.objects import Price, Quantity
from nautilus_trader.model.identifiers import (
    InstrumentId,
    OrderId,
    StrategyId,
    TraderId,
)
from nautilus_trader.common.enums import LogColor
from nautilus_trader.trading.strategy import Strategy


class OrderTypesExample:
    """订单类型示例类"""

    def __init__(self, trader_id: TraderId, strategy_id: StrategyId):
        """初始化"""
        self.trader_id = trader_id
        self.strategy_id = strategy_id
        self.instrument_id = InstrumentId.from_str("BTCUSDT.BINANCE")

    def create_market_order(self, side: OrderSide, quantity: Decimal) -> MarketOrder:
        """创建市价单"""
        return MarketOrder(
            trader_id=self.trader_id,
            strategy_id=self.strategy_id,
            instrument_id=self.instrument_id,
            order_side=side,
            quantity=Quantity(quantity),
            time_in_force=TimeInForce.GTC,  # Good Till Cancelled
            post_only=False,  # 允许吃单
            reduce_only=False,  # 非减仓单
        )

    def create_limit_order(
        self,
        side: OrderSide,
        quantity: Decimal,
        price: Price,
        **kwargs
    ) -> LimitOrder:
        """创建限价单"""
        # 默认参数
        default_kwargs = {
            'time_in_force': TimeInForce.GTC,
            'post_only': False,
            'reduce_only': False,
            'display_qty': None,
            'emulation_trigger': None,
            'trigger_instruction': None,
            'expire_time': None,
        }

        # 合并自定义参数
        default_kwargs.update(kwargs)

        return LimitOrder(
            trader_id=self.trader_id,
            strategy_id=self.strategy_id,
            instrument_id=self.instrument_id,
            order_side=side,
            quantity=Quantity(quantity),
            price=price,
            **default_kwargs
        )

    def create_stop_market_order(
        self,
        side: OrderSide,
        quantity: Decimal,
        trigger_price: Price,
        **kwargs
    ) -> StopMarketOrder:
        """创建止损市价单"""
        return StopMarketOrder(
            trader_id=self.trader_id,
            strategy_id=self.strategy_id,
            instrument_id=self.instrument_id,
            order_side=side,
            quantity=Quantity(quantity),
            trigger_price=trigger_price,
            time_in_force=TimeInForce.GTC,
            reduce_only=True,  # 止损单通常是减仓
            **kwargs
        )

    def create_stop_limit_order(
        self,
        side: OrderSide,
        quantity: Decimal,
        trigger_price: Price,
        limit_price: Price,
        **kwargs
    ) -> StopLimitOrder:
        """创建止损限价单"""
        return StopLimitOrder(
            trader_id=self.trader_id,
            strategy_id=self.strategy_id,
            instrument_id=self.instrument_id,
            order_side=side,
            quantity=Quantity(quantity),
            trigger_price=trigger_price,
            price=limit_price,
            time_in_force=TimeInForce.GTC,
            reduce_only=True,
            **kwargs
        )


# 订单使用示例
class OrderSubmissionExample(Strategy):
    """订单提交示例策略"""

    def __init__(self):
        """初始化策略"""
        super().__init__()
        self.example_orders = OrderTypesExample(self.trader_id, self.id)
        self.submitted_orders = {}

    def on_start(self):
        """策略启动"""
        self.log.info("订单提交示例策略启动")
        self.subscribe_bars(self.bar_type)

    def on_bar(self, bar: Bar):
        """处理K线数据"""
        if self.count_bars() % 10 == 0:  # 每10根K线测试不同订单类型
            self._test_order_types(bar)

    def _test_order_types(self, bar: Bar):
        """测试不同订单类型"""
        current_price = bar.close

        # 1. 市价单买入
        if self.count_bars() % 50 == 0:
            market_order = self.example_orders.create_market_order(
                OrderSide.BUY,
                Decimal("0.01")
            )
            self.submit_order(market_order)
            self.log.info("提交市价买单", color=LogColor.BLUE)

        # 2. 限价单卖出
        elif self.count_bars() % 30 == 0:
            limit_price = Price.from_str(str(current_price * Decimal("1.001")))
            limit_order = self.example_orders.create_limit_order(
                OrderSide.SELL,
                Decimal("0.01"),
                limit_price
            )
            self.submit_order(limit_order)
            self.log.info(f"提交限价卖单 - 价格: {limit_price}", color=LogColor.ORANGE)

        # 3. 止损单
        elif self.count_bars() % 20 == 0:
            stop_price = Price.from_str(str(current_price * Decimal("0.99")))
            stop_order = self.example_orders.create_stop_market_order(
                OrderSide.SELL,
                Decimal("0.01"),
                stop_price
            )
            self.submit_order(stop_order)
            self.log.info(f"提交止损单 - 触发价: {stop_price}", color=LogColor.RED)
```

### 8.1.2 高级订单类型

```python
"""
高级订单类型示例
"""

from nautilus_trader.model.orders import (
    MarketIfTouchedOrder,
    LimitIfTouchedOrder,
    TrailingStopMarketOrder,
    TrailingStopLimitOrder,
)
from nautilus_trader.model.enums import (
    TriggerType,
    TrailingOffset,
)


class AdvancedOrderTypesExample(OrderTypesExample):
    """高级订单类型示例"""

    def create_market_if_touched_order(
        self,
        side: OrderSide,
        quantity: Decimal,
        trigger_price: Price,
        **kwargs
    ) -> MarketIfTouchedOrder:
        """创建市价触发单"""
        return MarketIfTouchedOrder(
            trader_id=self.trader_id,
            strategy_id=self.strategy_id,
            instrument_id=self.instrument_id,
            order_side=side,
            quantity=Quantity(quantity),
            trigger_price=trigger_price,
            trigger_type=TriggerType.DEFAULT,
            time_in_force=TimeInForce.GTC,
            **kwargs
        )

    def create_limit_if_touched_order(
        self,
        side: OrderSide,
        quantity: Decimal,
        trigger_price: Price,
        limit_price: Price,
        **kwargs
    ) -> LimitIfTouchedOrder:
        """创建限价触发单"""
        return LimitIfTouchedOrder(
            trader_id=self.trader_id,
            strategy_id=self.strategy_id,
            instrument_id=self.instrument_id,
            order_side=side,
            quantity=Quantity(quantity),
            trigger_price=trigger_price,
            price=limit_price,
            trigger_type=TriggerType.DEFAULT,
            time_in_force=TimeInForce.GTC,
            **kwargs
        )

    def create_trailing_stop_market_order(
        self,
        side: OrderSide,
        quantity: Decimal,
        trailing_offset: Decimal,
        **kwargs
    ) -> TrailingStopMarketOrder:
        """创建追踪止损市价单"""
        return TrailingStopMarketOrder(
            trader_id=self.trader_id,
            strategy_id=self.strategy_id,
            instrument_id=self.instrument_id,
            order_side=side,
            quantity=Quantity(quantity),
            trailing_offset=TrailingOffset(
                trailing_offset=trailing_offset,
                offset_type='PRICE',
            ),
            trigger_type=TriggerType.DEFAULT,
            time_in_force=TimeInForce.GTC,
            reduce_only=True,
            **kwargs
        )

    def create_trailing_stop_limit_order(
        self,
        side: OrderSide,
        quantity: Decimal,
        trailing_offset: Decimal,
        limit_offset: Decimal,
        **kwargs
    ) -> TrailingStopLimitOrder:
        """创建追踪止损限价单"""
        return TrailingStopLimitOrder(
            trader_id=self.trader_id,
            strategy_id=self.strategy_id,
            instrument_id=self.instrument_id,
            order_side=side,
            quantity=Quantity(quantity),
            trailing_offset=TrailingOffset(
                trailing_offset=trailing_offset,
                offset_type='PRICE',
            ),
            limit_offset=TrailingOffset(
                trailing_offset=limit_offset,
                offset_type='PRICE',
            ),
            trigger_type=TriggerType.DEFAULT,
            time_in_force=TimeInForce.GTC,
            reduce_only=True,
            **kwargs
        )


# 高级订单使用策略
class AdvancedOrderStrategy(Strategy):
    """高级订单策略"""

    def __init__(self):
        """初始化策略"""
        super().__init__()
        self.example_orders = AdvancedOrderTypesExample(self.trader_id, self.id)

        # 状态跟踪
        self.position_open = False
        self.entry_price = None
        self.trailing_order_id = None

    def on_start(self):
        """策略启动"""
        self.log.info("高级订单策略启动")
        self.subscribe_bars(self.bar_type)

    def on_bar(self, bar: Bar):
        """处理K线数据"""
        current_price = bar.close

        if not self.position_open:
            # 尝试开仓
            self._try_open_position(current_price)
        else:
            # 更新追踪止损
            self._update_trailing_stop(current_price)

    def _try_open_position(self, current_price: Decimal):
        """尝试开仓"""
        # 使用触发单
        trigger_price = Price.from_str(str(current_price * Decimal("1.001")))

        mit_order = self.example_orders.create_market_if_touched_order(
            OrderSide.BUY,
            Decimal("0.01"),
            trigger_price
        )

        self.submit_order(mit_order)
        self.log.info(
            f"提交MIT买单 - 触发价: {trigger_price}, "
            f"当前价: {current_price}",
            color=LogColor.GREEN
        )

    def on_order_filled(self, event):
        """处理订单成交"""
        if event.order.side == OrderSide.BUY and not self.position_open:
            # 开仓成功
            self.position_open = True
            self.entry_price = event.last_px
            self.log.info(f"开仓成功 - 价格: {self.entry_price}")

            # 设置追踪止损
            self._set_initial_trailing_stop()

    def _set_initial_trailing_stop(self):
        """设置初始追踪止损"""
        if self.position_open:
            # 2%的追踪距离
            trailing_offset = Decimal("0.02")
            current_price = self.cache.get_last_price(self.example_orders.instrument_id)

            if current_price:
                trailing_stop = self.example_orders.create_trailing_stop_market_order(
                    OrderSide.SELL,
                    Decimal("0.01"),
                    trailing_offset
                )

                self.submit_order(trailing_stop)
                self.trailing_order_id = trailing_stop.client_order_id
                self.log.info(
                    f"设置追踪止损 - 距离: {trailing_offset*100}%, "
                    f"当前价: {current_price}"
                )

    def _update_trailing_stop(self, current_price: Decimal):
        """更新追踪止损"""
        if self.position_open and self.trailing_order_id:
            # 检查是否需要调整追踪距离
            unrealized_pnl = self._calculate_unrealized_pnl(current_price)

            # 如果盈利超过3%，紧化追踪距离
            if unrealized_pnl > Decimal("0.03"):
                self._tighten_trailing_stop(current_price)

    def _tighten_trailing_stop(self, current_price: Decimal):
        """紧化追踪止损"""
        # 取消旧订单
        if self.trailing_order_id:
            self.cancel_order(self.trailing_order_id)
            self.log.info("取消旧追踪止损订单")

        # 设置新的追踪止损（1%距离）
        new_trailing_offset = Decimal("0.01")
        new_trailing_stop = self.example_orders.create_trailing_stop_market_order(
            OrderSide.SELL,
            Decimal("0.01"),
            new_trailing_offset
        )

        self.submit_order(new_trailing_stop)
        self.trailing_order_id = new_trailing_stop.client_order_id
        self.log.info(
            f"紧化追踪止损 - 新距离: {new_trailing_offset*100}%, "
            f"当前价: {current_price}"
        )

    def _calculate_unrealized_pnl(self, current_price: Decimal) -> Decimal:
        """计算未实现盈亏"""
        if not self.entry_price:
            return Decimal("0")

        return (current_price - self.entry_price) / self.entry_price
```

## 8.2 订单生命周期管理

### 8.2.1 订单状态和事件

```python
"""
订单状态管理
"""

from nautilus_trader.model.events import (
    OrderInitialized,
    OrderSubmitted,
    OrderAccepted,
    OrderRejected,
    OrderCanceled,
    OrderExpired,
    OrderTriggered,
    OrderPendingUpdate,
    OrderPendingCancel,
    OrderFilled,
    OrderPartiallyFilled,
)
from typing import Dict, List, Optional
from datetime import datetime


class OrderLifecycleManager:
    """订单生命周期管理器"""

    def __init__(self):
        """初始化生命周期管理器"""
        self.order_states = {}  # order_id -> state info
        self.order_history = {}  # order_id -> event list
        self.active_orders = set()  # active order IDs
        self.filled_orders = {}  # order_id -> fill info

    def track_order(self, order_id: str):
        """开始追踪订单"""
        self.order_states[order_id] = {
            'created_at': datetime.utcnow(),
            'last_updated': datetime.utcnow(),
            'status': None,
            'filled_qty': Decimal('0'),
            'filled_amount': Decimal('0'),
            'commission': Decimal('0'),
        }
        self.order_history[order_id] = []
        self.active_orders.add(order_id)

    def update_order_state(self, event):
        """更新订单状态"""
        order_id = str(event.order_id)

        if order_id not in self.order_states:
            self.track_order(order_id)

        # 更新历史
        self.order_history[order_id].append({
            'timestamp': datetime.utcnow(),
            'event_type': type(event).__name__,
            'event': event,
        })

        # 更新状态
        state = self.order_states[order_id]
        state['last_updated'] = datetime.utcnow()

        # 处理特定事件
        if isinstance(event, OrderFilled):
            self._handle_fill(event, order_id, state)
        elif isinstance(event, OrderPartiallyFilled):
            self._handle_partial_fill(event, order_id, state)
        elif isinstance(event, OrderCanceled):
            self._handle_cancellation(event, order_id, state)
        elif isinstance(event, OrderRejected):
            self._handle_rejection(event, order_id, state)
        elif isinstance(event, OrderExpired):
            self._handle_expiry(event, order_id, state)

    def _handle_fill(self, event, order_id: str, state: dict):
        """处理完全成交"""
        state['status'] = 'FILLED'
        state['filled_qty'] = event.last_qty
        state['filled_amount'] = event.last_qty * event.last_px
        state['commission'] = event.commission
        state['filled_at'] = datetime.fromtimestamp(event.ts_event / 1e9)

        self.active_orders.remove(order_id)
        self.filled_orders[order_id] = {
            'order': event.order,
            'fill': event,
            'state': state.copy(),
        }

        self.log.info(f"订单完全成交: {order_id}")

    def _handle_partial_fill(self, event, order_id: str, state: dict):
        """处理部分成交"""
        state['status'] = 'PARTIALLY_FILLED'
        state['filled_qty'] += event.last_qty
        state['filled_amount'] += event.last_qty * event.last_px
        state['commission'] += event.commission

        self.log.info(f"订单部分成交: {order_id}, "
                     f"成交数量: {event.last_qty}, "
                     f"已成交总量: {state['filled_qty']}")

    def _handle_cancellation(self, event, order_id: str, state: dict):
        """处理取消"""
        state['status'] = 'CANCELED'
        state['canceled_at'] = datetime.fromtimestamp(event.ts_event / 1e9)

        if order_id in self.active_orders:
            self.active_orders.remove(order_id)

        self.log.info(f"订单已取消: {order_id}")

    def _handle_rejection(self, event, order_id: str, state: dict):
        """处理拒绝"""
        state['status'] = 'REJECTED'
        state['rejected_at'] = datetime.fromtimestamp(event.ts_event / 1e9)
        state['rejection_reason'] = event.reason

        if order_id in self.active_orders:
            self.active_orders.remove(order_id)

        self.log.warning(f"订单被拒绝: {order_id}, 原因: {event.reason}")

    def _handle_expiry(self, event, order_id: str, state: dict):
        """处理过期"""
        state['status'] = 'EXPIRED'
        state['expired_at'] = datetime.fromtimestamp(event.ts_event / 1e9)

        if order_id in self.active_orders:
            self.active_orders.remove(order_id)

        self.log.warning(f"订单已过期: {order_id}")

    def get_order_state(self, order_id: str) -> Optional[dict]:
        """获取订单状态"""
        return self.order_states.get(order_id)

    def get_active_orders(self) -> List[str]:
        """获取活跃订单"""
        return list(self.active_orders)

    def get_order_history(self, order_id: str) -> List[dict]:
        """获取订单历史"""
        return self.order_history.get(order_id, [])

    def get_order_stats(self) -> dict:
        """获取订单统计"""
        total_orders = len(self.order_states)
        active_count = len(self.active_orders)
        filled_count = len(self.filled_orders)

        # 计算成功率
        submitted_count = sum(1 for state in self.order_states.values()
                             if state['status'] in ['FILLED', 'CANCELED', 'REJECTED', 'EXPIRED'])

        success_rate = (filled_count / submitted_count) if submitted_count > 0 else 0

        return {
            'total_orders': total_orders,
            'active_orders': active_count,
            'filled_orders': filled_count,
            'success_rate': success_rate,
            'rejection_rate': sum(1 for state in self.order_states.values()
                                 if state['status'] == 'REJECTED') / total_orders if total_orders > 0 else 0,
        }


# 使用生命周期管理的策略
class LifecycleManagedStrategy(Strategy):
    """生命周期管理策略"""

    def __init__(self):
        """初始化策略"""
        super().__init__()
        self.lifecycle_manager = OrderLifecycleManager()

    def on_order_initialized(self, event: OrderInitialized):
        """处理订单初始化"""
        self.lifecycle_manager.update_order_state(event)

    def on_order_submitted(self, event: OrderSubmitted):
        """处理订单提交"""
        self.lifecycle_manager.update_order_state(event)

    def on_order_accepted(self, event: OrderAccepted):
        """处理订单接受"""
        self.lifecycle_manager.update_order_state(event)

    def on_order_rejected(self, event: OrderRejected):
        """处理订单拒绝"""
        self.lifecycle_manager.update_order_state(event)
        self._handle_rejection(event)

    def on_order_canceled(self, event: OrderCanceled):
        """处理订单取消"""
        self.lifecycle_manager.update_order_state(event)

    def on_order_expired(self, event: OrderExpired):
        """处理订单过期"""
        self.lifecycle_manager.update_order_state(event)

    def on_order_triggered(self, event: OrderTriggered):
        """处理订单触发"""
        self.lifecycle_manager.update_order_state(event)

    def on_order_pending_update(self, event: OrderPendingUpdate):
        """处理订单待更新"""
        self.lifecycle_manager.update_order_state(event)

    def on_order_pending_cancel(self, event: OrderPendingCancel):
        """处理订单待取消"""
        self.lifecycle_manager.update_order_state(event)

    def on_order_filled(self, event: OrderFilled):
        """处理订单成交"""
        self.lifecycle_manager.update_order_state(event)
        self._handle_fill(event)

    def on_order_partially_filled(self, event: OrderPartiallyFilled):
        """处理部分成交"""
        self.lifecycle_manager.update_order_state(event)

    def _handle_rejection(self, event: OrderRejected):
        """处理订单拒绝"""
        self.log.error(
            f"订单被拒绝: {event.order_id}, "
            f"原因: {event.reason}, "
            f"策略: {event.strategy_id}"
        )

        # 这里可以实现重试逻辑
        self._maybe_retry_order(event.order)

    def _handle_fill(self, event: OrderFilled):
        """处理订单成交"""
        self.log.info(
            f"订单成交: {event.order_id}, "
            f"数量: {event.last_qty}, "
            f"价格: {event.last_px}, "
            f"手续费: {event.commission}"
        )

        # 更新策略状态
        self._update_strategy_after_fill(event)

    def _update_strategy_after_fill(self, event: OrderFilled):
        """更新策略状态（子类实现）"""
        pass

    def _maybe_retry_order(self, rejected_order):
        """可能重试被拒绝的订单（子类实现）"""
        pass

    def get_order_stats(self):
        """获取订单统计"""
        return self.lifecycle_manager.get_order_stats()
```

## 8.3 订单组合策略

### 8.3.1 OCO（One Cancels Other）

```python
"""
OCO订单策略
"""

from nautilus_trader.model.orders import OrderList
from nautilus_trader.model.enums import OrderType


class OCOOrderStrategy(Strategy):
    """OCO订单策略"""

    def __init__(self):
        """初始化策略"""
        super().__init__()

        self.oco_order_lists = {}  # oco_id -> order_list
        self.active_oco_sets = {}  # oco_id -> orders

    def create_oco_orders(
        self,
        take_profit_price: Price,
        stop_loss_price: Price,
        quantity: Decimal,
    ) -> OrderList:
        """创建OCO订单组"""

        # 止盈限价单
        take_profit_order = LimitOrder(
            trader_id=self.trader_id,
            strategy_id=self.id,
            instrument_id=self.bar_type.instrument_id,
            order_side=OrderSide.SELL,
            quantity=Quantity(quantity),
            price=take_profit_price,
            time_in_force=TimeInForce.GTC,
            reduce_only=True,
        )

        # 止损市价单
        stop_loss_order = StopMarketOrder(
            trader_id=self.trader_id,
            strategy_id=self.id,
            instrument_id=self.bar_type.instrument_id,
            order_side=OrderSide.SELL,
            quantity=Quantity(quantity),
            trigger_price=stop_loss_price,
            time_in_force=TimeInForce.GTC,
            reduce_only=True,
        )

        # 创建OCO订单列表
        oco_orders = OrderList(
            orders=[take_profit_order, stop_loss_order],
            oco=True,  # 一成交另一取消
        )

        self.oco_order_lists[oco_orders.id] = oco_orders
        self.active_oco_sets[oco_orders.id] = {
            'created_at': datetime.utcnow(),
            'take_profit_id': take_profit_order.client_order_id,
            'stop_loss_id': stop_loss_order.client_order_id,
            'filled': False,
            'canceled': False,
        }

        self.log.info(
            f"创建OCO订单组 - ID: {oco_orders.id}, "
            f"止盈: {take_profit_price}, "
            f"止损: {stop_loss_price}"
        )

        return oco_orders

    def submit_oco_orders(self, oco_orders: OrderList):
        """提交OCO订单"""
        self.submit_order_list(oco_orders)

    def on_order_filled(self, event: OrderFilled):
        """处理订单成交"""
        # 检查是否是OCO订单
        for oco_id, oco_info in self.active_oco_sets.items():
            if event.order_id in [oco_info['take_profit_id'], oco_info['stop_loss_id']]:
                # 标记OCO组已成交
                oco_info['filled'] = True
                oco_info['filled_price'] = event.last_px
                oco_info['filled_time'] = datetime.fromtimestamp(event.ts_event / 1e9)

                # 另一个订单应该被自动取消
                self._cancel_oco_partner(oco_id, event.order_id)

                # 记录OCO执行结果
                self._log_oco_result(oco_id, event.order_id, event)

                break

    def on_order_canceled(self, event: OrderCanceled):
        """处理订单取消"""
        # 检查是否是OCO订单的伙伴被取消
        for oco_id, oco_info in self.active_oco_sets.items():
            if event.order_id in [oco_info['take_profit_id'], oco_info['stop_loss_id']]:
                if not oco_info['filled']:
                    oco_info['canceled'] = True
                    oco_info['canceled_reason'] = 'PARTNER_CANCELED'
                    self._log_oco_cancellation(oco_id, event)
                break

    def _cancel_oco_partner(self, oco_id: str, filled_order_id: str):
        """取消OCO伙伴订单"""
        oco_info = self.active_oco_sets[oco_id]

        # 确定要取消的订单
        if filled_order_id == oco_info['take_profit_id']:
            partner_id = oco_info['stop_loss_id']
            partner_type = '止损单'
        else:
            partner_id = oco_info['take_profit_id']
            partner_type = '止盈单'

        # 取消伙伴订单
        if partner_id:
            self.cancel_order(partner_id)
            self.log.info(f"OCO执行 - 自动取消{partner_type}: {partner_id}")

    def _log_oco_result(self, oco_id: str, filled_order_id: str, fill_event):
        """记录OCO执行结果"""
        oco_info = self.active_oco_sets[oco_id]

        filled_type = '止盈' if filled_order_id == oco_info['take_profit_id'] else '止损'
        duration = oco_info['filled_time'] - oco_info['created_at']

        self.log.info(
            f"OCO订单组{oco_id}执行完成 - "
            f"{filled_type}成交: {fill_event.last_px}, "
            f"执行时间: {duration.total_seconds():.1f}秒"
        )

    def _log_oco_cancellation(self, oco_id: str, cancel_event):
        """记录OCO取消"""
        oco_info = self.active_oco_sets[oco_id]

        partner_order_id = (oco_info['take_profit_id'] if cancel_event.order_id == oco_info['stop_loss_id']
                        else oco_info['stop_loss_id'])

        partner_type = '止损单' if partner_order_id == oco_info['stop_loss_id'] else '止盈单'

        self.log.info(
            f"OCO订单组{oco_id}部分执行 - "
            f"取消{partner_type}: {partner_order_id}"
        )

    def cleanup_completed_oco_sets(self):
        """清理已完成的OCO组"""
        completed_oco = []

        for oco_id, oco_info in self.active_oco_sets.items():
            if oco_info['filled']:
                completed_oco.append(oco_id)
                continue

            # 检查所有订单是否都已完成
            all_completed = True
            for order_id in [oco_info['take_profit_id'], oco_info['stop_loss_id']]:
                state = self.cache.order(order_id)
                if state and state.status not in ['FILLED', 'CANCELED']:
                    all_completed = False
                    break

            if all_completed:
                completed_oco.append(oco_id)

        # 清理
        for oco_id in completed_oco:
            del self.active_oco_sets[oco_id]
            if oco_id in self.oco_order_lists:
                del self.oco_order_lists[oco_id]

        if completed_oco:
            self.log.info(f"清理 {len(completed_oco)} 个OCO订单组")


# 使用OCO策略的示例
class OCOTradingStrategy(Strategy):
    """使用OCO的交易策略"""

    def __init__(self):
        """初始化策略"""
        super().__init__()
        self.oco_manager = OCOOrderStrategy()
        self.current_position = None

    def on_order_filled(self, event: OrderFilled):
        """处理订单成交"""
        self.oco_manager.on_order_filled(event)

        # 检查是否是开仓
        if event.order_side == OrderSide.BUY:
            self._handle_position_open(event)
        # OCO成交应该是平仓
        else:
            self._handle_position_close(event)

    def _handle_position_open(self, event: OrderFilled):
        """处理开仓"""
        self.current_position = {
            'entry_price': event.last_px,
            'quantity': event.last_qty,
            'entry_time': datetime.fromtimestamp(event.ts_event / 1e9),
        }

        # 计算止盈止损价格
        take_profit_price = self._calculate_take_profit(event.last_px)
        stop_loss_price = self._calculate_stop_loss(event.last_px)

        # 创建OCO订单
        oco_orders = self.oco_manager.create_oco_orders(
            take_profit_price,
            stop_loss_price,
            event.last_qty,
        )

        # 提交OCO订单
        self.oco_manager.submit_oco_orders(oco_orders)

        self.log.info(
            f"开仓成功 - 价格: {event.last_px}, "
            f"设置OCO止盈: {take_profit_price}, "
            f"止损: {stop_loss_price}"
        )

    def _calculate_take_profit(self, entry_price: Decimal) -> Price:
        """计算止盈价格"""
        risk_reward_ratio = 2.0  # 1:2 的风险收益比
        risk_amount = entry_price * Decimal("0.02")  # 2% 风险
        take_profit_amount = risk_amount * risk_reward_ratio
        take_profit_price = entry_price + take_profit_price

        return Price(take_profit_price)

    def _calculate_stop_loss(self, entry_price: Decimal) -> Price:
        """计算止损价格"""
        stop_distance = entry_price * Decimal("0.02")  # 2% 止损
        stop_loss_price = entry_price - stop_distance

        return Price(stop_loss_price)

    def _handle_position_close(self, event: OrderFilled):
        """处理平仓"""
        if self.current_position:
            duration = datetime.fromtimestamp(event.ts_event / 1e9) - self.current_position['entry_time']
            pnl = (event.last_px - self.current_position['entry_price']) * self.current_position['quantity']

            self.log.info(
                f"平仓完成 - 入场价: {self.current_position['entry_price']}, "
                f"出场价: {event.last_px}, "
                f"盈亏: {pnl}, "
                f"持仓时间: {duration.total_seconds():.1f}秒"
            )

            self.current_position = None

            # 清理OCO订单
            self.oco_manager.cleanup_completed_oco_sets()
```

### 8.3.2 OTO（One Triggers Other）

```python
"""
OTO订单策略
"""

from nautilus_trader.model.orders import OrderList
from nautilus_trader.model.enums import OrderType


class OTOOrderStrategy(Strategy):
    """OTO订单策略"""

    def __init__(self):
        """初始化策略"""
        super().__init__()

        self.oto_order_lists = {}  # oto_id -> order_list
        self.active_oto_sets = {}  # oto_id -> orders

    def create_oto_orders(
        self,
        primary_order,  # 主要订单（限价单）
        secondary_order,  # 触发订单（止损单）
    ) -> OrderList:
        """创建OTO订单组"""

        # 创建OTO订单列表
        oto_orders = OrderList(
            orders=[primary_order, secondary_order],
            oto=True,  # 一触发另一个
        )

        self.oto_order_lists[oto_orders.id] = oto_orders
        self.active_oto_sets[oto_orders.id] = {
            'created_at': datetime.utcnow(),
            'primary_id': primary_order.client_order_id,
            'secondary_id': secondary_order.client_order_id,
            'triggered': False,
            'canceled': False,
        }

        self.log.info(
            f"创建OTO订单组 - ID: {oto_orders.id}, "
            f"主要订单: {primary_order}, "
            f"触发订单: {secondary_order}"
        )

        return oto_orders

    def submit_oto_orders(self, oto_orders: OrderList):
        """提交OTO订单"""
        self.submit_order_list(oto_orders)

    def on_order_filled(self, event: OrderFilled):
        """处理订单成交"""
        # 检查是否是OTO订单的触发条件
        for oto_id, oto_info in self.active_oto_sets.items():
            if event.order_id == oto_info['primary_id']:
                # 主要订单成交，触发次要订单
                self._trigger_secondary_order(oto_id, event)
                break

    def on_order_canceled(self, event: OrderCanceled):
        """处理订单取消"""
        # 检查是否是OTO订单
        for oto_id, oto_info in self.active_oto_sets.items():
            if event.order_id == oto_info['primary_id']:
                # 主要订单取消，自动取消次要订单
                self._cancel_secondary_order(oto_id, event)
                break

    def _trigger_secondary_order(self, oto_id: str, trigger_event):
        """触发次要订单"""
        oto_info = self.active_oto_sets[oto_id]

        if not oto_info['triggered']:
            oto_info['triggered'] = True
            oto_info['triggered_at'] = datetime.fromtimestamp(trigger_event.ts_event / 1e9)

            # 获取次要订单并提交
            secondary_order = self.cache.order(oto_info['secondary_id'])
            if secondary_order and secondary_order.status == OrderStatus.INITIALIZED:
                self.submit_order(secondary_order)
                self.log.info(
                    f"OTO触发 - 提交次要订单: {secondary_order}, "
                    f"触发价格: {trigger_event.last_px}"
                )

    def _cancel_secondary_order(self, oto_id: str, cancel_event):
        """取消次要订单"""
        oto_info = self.active_oto_sets[oto_id]

        if not oto_info['triggered']:
            # 取消未触发的次要订单
            secondary_order_id = oto_info['secondary_id']
            secondary_order = self.cache.order(secondary_order_id)

            if secondary_order and secondary_order.status in [OrderStatus.INITIALIZED, OrderStatus.SUBMITTED]:
                self.cancel_order(secondary_order_id)
                self.log.warning(
                    f"OTO取消 - 主要订单取消，自动取消次要订单: {secondary_order}"
                )

            oto_info['canceled'] = True
            oto_info['canceled_at'] = datetime.utcnow()
            oto_info['cancel_reason'] = 'PRIMARY_CANCELLED'


# 使用OTO策略的示例
class OTOMomentumStrategy(Strategy):
    """使用OTO的动量策略"""

    def __init__(self):
        """初始化策略"""
        super().__init__()
        self.oto_manager = OTOOrderStrategy()

        # 指标
        self.rsi = RSI(period=14)
        self.ema = EMA(period=20)

        # 状态
        self.position = None
        self.entry_oto_id = None

    def on_bar(self, bar: Bar):
        """处理K线数据"""
        # 更新指标
        self.rsi.update_raw(float(bar.close))
        self.ema.update_raw(float(bar.close))

        if not (self.rsi.initialized and self.ema.initialized):
            return

        # 检查交易信号
        if not self.position:
            self._check_entry_signal(bar)
        else:
            self._check_exit_signal(bar)

    def _check_entry_signal(self, bar: Bar):
        """检查入场信号"""
        current_price = bar.close
        ema_value = self.ema.value
        rsi_value = self.rsi.value

        # 买入条件：价格 > EMA 且 RSI > 50
        if (current_price > ema_value and rsi_value > 50 and
            not self.entry_oto_id):

            # 设置买入限价单（低于当前价格）
            entry_price = current_price * Decimal("0.995")

            primary_order = LimitOrder(
                trader_id=self.trader_id,
                strategy_id=self.id,
                instrument_id=self.bar_type.instrument_id,
                order_side=OrderSide.BUY,
                quantity=Quantity(Decimal("0.01")),
                price=Price(entry_price),
                time_in_force=TimeInForce.GTC,
            )

            # 设置止损单作为触发订单
            stop_loss_price = entry_price * Decimal("0.98")

            secondary_order = StopMarketOrder(
                trader_id=self.trader_id,
                strategy_id=self.id,
                instrument_id=self.bar_type.instrument_id,
                order_side=OrderSide.SELL,
                quantity=Quantity(Decimal("0.01")),
                trigger_price=Price(stop_loss_price),
                time_in_force=TimeInForce.GTC,
                reduce_only=True,
            )

            # 创建OTO订单
            oto_orders = self.oto_manager.create_oto_orders(
                primary_order,
                secondary_order,
            )

            # 提交OTO
            self.oto_manager.submit_oto_orders(oto_orders)
            self.entry_oto_id = oto_orders.id

            self.log.info(
                f"提交OTO - 买单: {entry_price}, "
                f"止损触发: {stop_loss_price}"
            )

    def _check_exit_signal(self, bar: Bar):
        """检查出场信号"""
        current_price = bar.close
        entry_price = self.position['entry_price']

        # 简单的止盈策略
        if current_price > entry_price * Decimal("1.03"):
            # 市价卖出
            exit_order = MarketOrder(
                trader_id=self.trader_id,
                strategy_id=self.id,
                instrument_id=self.bar_type.instrument_id,
                order_side=OrderSide.SELL,
                quantity=self.position['quantity'],
            )
            self.submit_order(exit_order)
            self.log.info(f"止盈出场 - 价格: {current_price}")

    def on_order_filled(self, event: OrderFilled):
        """处理订单成交"""
        self.oto_manager.on_order_filled(event)

        if event.order_side == OrderSide.BUY and not self.position:
            # 开仓
            self.position = {
                'entry_price': event.last_px,
                'quantity': event.last_qty,
                'entry_time': datetime.fromtimestamp(event.ts_event / 1e9),
            }
            self.log.info(f"开仓 - 价格: {event.last_px}")

        elif event.order_side == OrderSide.SELL and self.position:
            # 平仓
            pnl = (event.last_px - self.position['entry_price']) * self.position['quantity']
            self.log.info(
                f"平仓 - 盈亏: {pnl}, "
                f"入场价: {self.position['entry_price']}, "
                f"出场价: {event.last_px}"
            )
            self.position = None
            self.entry_oto_id = None

    def on_order_rejected(self, event: OrderRejected):
        """处理订单拒绝"""
        self.oto_manager.on_order_rejected(event)

        # 如果主要订单被拒绝，重试
        if self.entry_oto_id:
            for oto_id, oto_info in self.oto_manager.active_oto_sets.items():
                if (oto_id == self.entry_oto_id and
                    event.order_id == oto_info['primary_id']):
                    # 清理OTO并重试
                    del self.oto_manager.active_oto_sets[oto_id]
                    self.entry_oto_id = None
                    break
```

## 8.4 订单执行算法

### 8.4.1 TWAP（时间加权平均价格）

```python
"""
TWAP执行算法
"""

from decimal import Decimal
from typing import List, Dict
from datetime import datetime, timedelta
import numpy as np


class TWAPExecutor:
    """TWAP执行器"""

    def __init__(
        self,
        trader_id,
        strategy_id,
        instrument_id,
        total_quantity: Decimal,
        duration_seconds: int,
        num_slices: int = 10,
    ):
        """
        初始化TWAP执行器

        Parameters
        ----------
        trader_id : TraderId
            交易者ID
        strategy_id : StrategyId
            策略ID
        instrument_id : InstrumentId
            交易工具ID
        total_quantity : Decimal
            总执行数量
        duration_seconds : int
            执行持续时间（秒）
        num_slices : int
            分片数量
        """
        self.trader_id = trader_id
        self.strategy_id = strategy_id
        self.instrument_id = instrument_id

        self.total_quantity = total_quantity
        self.remaining_quantity = total_quantity

        self.duration = duration_seconds
        self.num_slices = num_slices
        self.slice_interval = duration_seconds / num_slices

        self.slice_quantities = self._calculate_slice_quantities()

        self.orders = []  # 已提交的订单
        self.filled_orders = []  # 已成交的订单
        self.start_time = None
        self.end_time = None

        self.is_active = False
        self.is_completed = False

    def _calculate_slice_quantities(self) -> List[Decimal]:
        """计算分片数量"""
        # 均匀分配
        base_quantity = self.total_quantity / self.num_slices

        # 添加随机扰动
        random_factors = np.random.uniform(0.8, 1.2, self.num_slices)
        random_factors /= np.mean(random_factors)  # 归一化

        quantities = [base_quantity * Decimal(str(f)) for f in random_factors]
        quantities = [round(q, 6) for q in quantities]

        # 确保总和等于总量
        total = sum(quantities)
        if total != self.total_quantity:
            # 调整最后一个分片
            quantities[-1] += (self.total_quantity - total)

        return quantities

    def start_execution(self, start_time_ns: int):
        """开始执行"""
        self.start_time = datetime.fromtimestamp(start_time_ns / 1e9)
        self.end_time = self.start_time + timedelta(seconds=self.duration)
        self.is_active = True

        # 创建并提交所有订单
        self._create_and_submit_orders()

    def _create_and_submit_orders(self):
        """创建并提交所有订单"""
        current_time = self.start_time
        remaining = self.remaining_quantity

        for i, (slice_qty, slice_num) in enumerate(
            zip(self.slice_quantities, range(self.num_slices))
        ):
            if slice_qty <= 0:
                continue

            # 计算订单时间
            order_time = current_time + timedelta(
                seconds=(slice_num + 0.5) * self.slice_interval
            )

            # 使用限价单，价格基于当前市场价
            # 这里应该获取当前价格
            current_price = self._get_current_price()

            # 添加随机价格扰动
            price_adjustment = np.random.uniform(-0.0005, 0.0005)  # ±0.05%
            limit_price = current_price * (1 + price_adjustment)

            # 创建限价单
            order = LimitOrder(
                trader_id=self.trader_id,
                strategy_id=self.strategy_id,
                instrument_id=self.instrument_id,
                order_side=OrderSide.BUY,  # 可以根据需要设置
                quantity=Quantity(slice_qty),
                price=Price(limit_price),
                time_in_force=TimeInForce.GTC,
                expire_time=int(order_time.timestamp() * 1e9),
                post_only=True,  # 只做maker，避免手续费
            )

            # 这里应该提交订单
            self.orders.append({
                'order': order,
                'slice_number': slice_num,
                'quantity': slice_qty,
                'scheduled_time': order_time,
                'status': 'SCHEDULED',
            })

    def _get_current_price(self) -> Decimal:
        """获取当前价格（示例实现）"""
        # 这里应该从缓存或数据引擎获取
        # 返回模拟值
        return Decimal("50000.00")

    def update_execution(self, current_time_ns: int):
        """更新执行状态"""
        if not self.is_active or self.is_completed:
            return

        current_time = datetime.fromtimestamp(current_time_ns / 1e9)

        # 检查是否超时
        if current_time > self.end_time:
            self._complete_execution("TIMEOUT")
            return

        # 检查是否已完成
        if self.remaining_quantity <= 0:
            self._complete_execution("COMPLETED")
            return

        # 这里可以添加动态调整逻辑
        # 例如基于市场条件调整剩余订单

    def on_order_filled(self, event):
        """处理订单成交"""
        order_id = event.order_id

        # 找到对应的订单信息
        for order_info in self.orders:
            if order_info['order'].client_order_id == order_id:
                order_info['status'] = 'FILLED'
                order_info['fill_time'] = datetime.fromtimestamp(event.ts_event / 1e9)
                order_info['fill_price'] = event.last_px
                order_info['fill_qty'] = event.last_qty

                self.filled_orders.append(order_info)
                self.remaining_quantity -= event.last_qty

                break

        # 检查是否需要取消剩余订单
        if self.remaining_quantity <= 0:
            self._cancel_remaining_orders()

    def _cancel_remaining_orders(self):
        """取消剩余订单"""
        for order_info in self.orders:
            if order_info['status'] == 'SCHEDULED':
                # 这里应该取消订单
                order_info['status'] = 'CANCELED'

    def _complete_execution(self, reason: str):
        """完成执行"""
        self.is_active = False
        self.is_completed = True
        self.completion_time = datetime.utcnow()
        self.completion_reason = reason

        # 取消所有未成交的订单
        self._cancel_remaining_orders()

    def get_execution_stats(self) -> dict:
        """获取执行统计"""
        total_qty = float(self.total_quantity)
        filled_qty = sum(float(o['fill_qty']) for o in self.filled_orders)

        if filled_qty > 0:
            avg_price = sum(
                float(o['fill_price']) * float(o['fill_qty'])
                for o in self.filled_orders
            ) / filled_qty
        else:
            avg_price = 0

        filled_rate = filled_qty / total_qty if total_qty > 0 else 0

        return {
            'total_quantity': total_qty,
            'filled_quantity': filled_qty,
            'remaining_quantity': float(self.remaining_quantity),
            'filled_rate': filled_rate,
            'average_fill_price': avg_price,
            'total_orders': len(self.orders),
            'filled_orders': len(self.filled_orders),
            'is_completed': self.is_completed,
            'completion_reason': getattr(self, 'completion_reason', None),
        }


# 使用TWAP的策略
class TWAPStrategy(Strategy):
    """使用TWAP执行算法的策略"""

    def __init__(self):
        """初始化策略"""
        super().__init__()

        self.twap_executors = {}  # execution_id -> executor
        self.next_execution_id = 1

    def create_twap_execution(
        self,
        quantity: Decimal,
        duration_minutes: int,
        num_slices: int = 20,
    ) -> str:
        """创建TWAP执行"""

        execution_id = f"TWAP_{self.next_execution_id}"
        self.next_execution_id += 1

        executor = TWAPExecutor(
            trader_id=self.trader_id,
            strategy_id=self.id,
            instrument_id=self.bar_type.instrument_id,
            total_quantity=quantity,
            duration_seconds=duration_minutes * 60,
            num_slices=num_slices,
        )

        self.twap_executors[execution_id] = executor
        return execution_id

    def start_twap_execution(self, execution_id: str):
        """开始TWAP执行"""
        if execution_id not in self.twap_executors:
            self.log.error(f"TWAP执行器不存在: {execution_id}")
            return

        executor = self.twap_executors[execution_id]
        executor.start_execution(self.clock.timestamp_ns())

        # 提交所有订单
        for order_info in executor.orders:
            self.submit_order(order_info['order'])

        self.log.info(f"开始TWAP执行: {execution_id}")

    def update_twap_executions(self):
        """更新TWAP执行状态"""
        current_time = self.clock.timestamp_ns()

        for executor in self.twap_executors.values():
            executor.update_execution(current_time)

    def on_order_filled(self, event):
        """处理订单成交"""
        # 检查是否是TWAP订单
        for executor in self.twap_executors.values():
            executor.on_order_filled(event)

    def get_twap_stats(self, execution_id: str) -> dict:
        """获取TWAP统计"""
        if execution_id not in self.twap_executors:
            return {}

        return self.twap_executors[execution_id].get_execution_stats()

    def on_bar(self, bar: Bar):
        """处理K线数据"""
        # 更新TWAP执行
        self.update_twap_executions()

        # 简单的示例：每1小时创建一个新的TWAP执行
        if self.count_bars() % 60 == 0:  # 假设1小时K线
            # 创建0.1 BTC的TWAP执行，持续30分钟
            execution_id = self.create_twap_execution(
                quantity=Decimal("0.1"),
                duration_minutes=30,
                num_slices=30,
            )

            # 立即开始执行
            self.start_twap_execution(execution_id)

    def on_stop(self):
        """策略停止"""
        # 完成所有TWAP执行
        for executor in self.twap_executors.values():
            if executor.is_active:
                executor._complete_execution("STRATEGY_STOP")

        # 输出统计
        self._log_twap_stats()

    def _log_twap_stats(self):
        """记录TWAP统计"""
        self.log.info("TWAP执行统计:")

        for execution_id, executor in self.twap_executors.items():
            stats = executor.get_execution_stats()
            if stats:
                self.log.info(
                    f"执行器 {execution_id}: "
                    f"执行率: {stats['filled_rate']:.1%}, "
                    f"平均价格: {stats['average_fill_price']:.2f}"
                )
```

## 8.5 下一步

在本章中，我们学习了：

1. NautilusTrader 订单系统的核心概念
2. 各种订单类型的使用方法
3. 订单生命周期管理
4. 高级订单策略（OCO、OTO）
5. 执行算法（TWAP）

在下一章中，我们将学习：

1. 投资组合管理
2. 风险控制机制
3. 资金管理策略
4. 多策略协调
5. 绩效分析

## 8.6 总结

### 关键要点
1. NautilusTrader 支持所有常见的订单类型
2. 订单生命周期管理提供了完整的追踪能力
3. OCO 和 OTO 等高级订单策略便于风险控制
4. 执行算法可以优化大额订单的执行效果

### 最佳实践
1. 始终设置止损订单管理风险
2. 使用 OCO 策略实现止盈止损
3. 大额订单使用 TWAP 等算法避免市场冲击
4. 监控订单状态并及时处理异常

## 8.7 参考资料

- [订单管理 API 文档](https://nautilustrader.io/docs/latest/api_reference/orders/)
- [执行算法最佳实践](https://nautilustrader.io/docs/latest/concepts/execution/)
- [风险管理指南](https://nautilustrader.io/docs/latest/concepts/risk/)
- [OCO/OTO 订单策略](https://www.investopedia.com/terms/o/onecancelsother.asp)