---
title: "Chapter 8: Order Management System"
---

# Chapter 8: Order Management System

## Learning Objectives

Through this chapter, you will learn:
- How to use various order types
- Understanding order lifecycle and state management
- Advanced order strategies (OCO, OTO, etc.)
- Implementing intelligent order management
- Order execution and risk management

## 8.1 Order Types Overview

### 8.1.1 Basic Order Types

NautilusTrader supports all common order types:

```python
"""
Basic Order Types Example
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
    """Order Types Example Class"""

    def __init__(self, trader_id: TraderId, strategy_id: StrategyId):
        """Initialize"""
        self.trader_id = trader_id
        self.strategy_id = strategy_id
        self.instrument_id = InstrumentId.from_str("BTCUSDT.BINANCE")

    def create_market_order(self, side: OrderSide, quantity: Decimal) -> MarketOrder:
        """Create a market order"""
        return MarketOrder(
            trader_id=self.trader_id,
            strategy_id=self.strategy_id,
            instrument_id=self.instrument_id,
            order_side=side,
            quantity=Quantity(quantity),
            time_in_force=TimeInForce.GTC,  # Good Till Cancelled
            post_only=False,  # Allow taker
            reduce_only=False,  # Not a reduce-only order
        )

    def create_limit_order(
        self,
        side: OrderSide,
        quantity: Decimal,
        price: Price,
        **kwargs
    ) -> LimitOrder:
        """Create a limit order"""
        # Default parameters
        default_kwargs = {
            'time_in_force': TimeInForce.GTC,
            'post_only': False,
            'reduce_only': False,
            'display_qty': None,
            'emulation_trigger': None,
            'trigger_instruction': None,
            'expire_time': None,
        }

        # Merge custom parameters
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
        """Create a stop market order"""
        return StopMarketOrder(
            trader_id=self.trader_id,
            strategy_id=self.strategy_id,
            instrument_id=self.instrument_id,
            order_side=side,
            quantity=Quantity(quantity),
            trigger_price=trigger_price,
            time_in_force=TimeInForce.GTC,
            reduce_only=True,  # Stop orders are usually reduce-only
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
        """Create a stop limit order"""
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


# Order Usage Example
class OrderSubmissionExample(Strategy):
    """Order Submission Example Strategy"""

    def __init__(self):
        """Initialize strategy"""
        super().__init__()
        self.example_orders = OrderTypesExample(self.trader_id, self.id)
        self.submitted_orders = {}

    def on_start(self):
        """Strategy start"""
        self.log.info("Order submission example strategy started")
        self.subscribe_bars(self.bar_type)

    def on_bar(self, bar: Bar):
        """Process bar data"""
        if self.count_bars() % 10 == 0:  # Test different order types every 10 bars
            self._test_order_types(bar)

    def _test_order_types(self, bar: Bar):
        """Test different order types"""
        current_price = bar.close

        # 1. Market order buy
        if self.count_bars() % 50 == 0:
            market_order = self.example_orders.create_market_order(
                OrderSide.BUY,
                Decimal("0.01")
            )
            self.submit_order(market_order)
            self.log.info("Submitted market buy order", color=LogColor.BLUE)

        # 2. Limit order sell
        elif self.count_bars() % 30 == 0:
            limit_price = Price.from_str(str(current_price * Decimal("1.001")))
            limit_order = self.example_orders.create_limit_order(
                OrderSide.SELL,
                Decimal("0.01"),
                limit_price
            )
            self.submit_order(limit_order)
            self.log.info(f"Submitted limit sell order - Price: {limit_price}", color=LogColor.ORANGE)

        # 3. Stop order
        elif self.count_bars() % 20 == 0:
            stop_price = Price.from_str(str(current_price * Decimal("0.99")))
            stop_order = self.example_orders.create_stop_market_order(
                OrderSide.SELL,
                Decimal("0.01"),
                stop_price
            )
            self.submit_order(stop_order)
            self.log.info(f"Submitted stop order - Trigger price: {stop_price}", color=LogColor.RED)
```

### 8.1.2 Advanced Order Types

```python
"""
Advanced Order Types Example
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
    """Advanced Order Types Example"""

    def create_market_if_touched_order(
        self,
        side: OrderSide,
        quantity: Decimal,
        trigger_price: Price,
        **kwargs
    ) -> MarketIfTouchedOrder:
        """Create a market if touched order"""
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
        """Create a limit if touched order"""
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
        """Create a trailing stop market order"""
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
        """Create a trailing stop limit order"""
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


# Advanced Order Usage Strategy
class AdvancedOrderStrategy(Strategy):
    """Advanced Order Strategy"""

    def __init__(self):
        """Initialize strategy"""
        super().__init__()
        self.example_orders = AdvancedOrderTypesExample(self.trader_id, self.id)

        # State tracking
        self.position_open = False
        self.entry_price = None
        self.trailing_order_id = None

    def on_start(self):
        """Strategy start"""
        self.log.info("Advanced order strategy started")
        self.subscribe_bars(self.bar_type)

    def on_bar(self, bar: Bar):
        """Process bar data"""
        current_price = bar.close

        if not self.position_open:
            # Try to open position
            self._try_open_position(current_price)
        else:
            # Update trailing stop
            self._update_trailing_stop(current_price)

    def _try_open_position(self, current_price: Decimal):
        """Try to open position"""
        # Use trigger order
        trigger_price = Price.from_str(str(current_price * Decimal("1.001")))

        mit_order = self.example_orders.create_market_if_touched_order(
            OrderSide.BUY,
            Decimal("0.01"),
            trigger_price
        )

        self.submit_order(mit_order)
        self.log.info(
            f"Submitted MIT buy order - Trigger price: {trigger_price}, "
            f"Current price: {current_price}",
            color=LogColor.GREEN
        )

    def on_order_filled(self, event):
        """Handle order fill"""
        if event.order.side == OrderSide.BUY and not self.position_open:
            # Position opened successfully
            self.position_open = True
            self.entry_price = event.last_px
            self.log.info(f"Position opened - Price: {self.entry_price}")

            # Set trailing stop
            self._set_initial_trailing_stop()

    def _set_initial_trailing_stop(self):
        """Set initial trailing stop"""
        if self.position_open:
            # 2% trailing distance
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
                    f"Set trailing stop - Distance: {trailing_offset*100}%, "
                    f"Current price: {current_price}"
                )

    def _update_trailing_stop(self, current_price: Decimal):
        """Update trailing stop"""
        if self.position_open and self.trailing_order_id:
            # Check if need to adjust trailing distance
            unrealized_pnl = self._calculate_unrealized_pnl(current_price)

            # If profit exceeds 3%, tighten trailing stop
            if unrealized_pnl > Decimal("0.03"):
                self._tighten_trailing_stop(current_price)

    def _tighten_trailing_stop(self, current_price: Decimal):
        """Tighten trailing stop"""
        # Cancel old order
        if self.trailing_order_id:
            self.cancel_order(self.trailing_order_id)
            self.log.info("Canceled old trailing stop order")

        # Set new trailing stop (1% distance)
        new_trailing_offset = Decimal("0.01")
        new_trailing_stop = self.example_orders.create_trailing_stop_market_order(
            OrderSide.SELL,
            Decimal("0.01"),
            new_trailing_offset
        )

        self.submit_order(new_trailing_stop)
        self.trailing_order_id = new_trailing_stop.client_order_id
        self.log.info(
            f"Tightened trailing stop - New distance: {new_trailing_offset*100}%, "
            f"Current price: {current_price}"
        )

    def _calculate_unrealized_pnl(self, current_price: Decimal) -> Decimal:
        """Calculate unrealized P&L"""
        if not self.entry_price:
            return Decimal("0")

        return (current_price - self.entry_price) / self.entry_price
```

## 8.2 Order Lifecycle Management

### 8.2.1 Order States and Events

```python
"""
Order State Management
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
    """Order Lifecycle Manager"""

    def __init__(self):
        """Initialize lifecycle manager"""
        self.order_states = {}  # order_id -> state info
        self.order_history = {}  # order_id -> event list
        self.active_orders = set()  # active order IDs
        self.filled_orders = {}  # order_id -> fill info

    def track_order(self, order_id: str):
        """Start tracking order"""
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
        """Update order state"""
        order_id = str(event.order_id)

        if order_id not in self.order_states:
            self.track_order(order_id)

        # Update history
        self.order_history[order_id].append({
            'timestamp': datetime.utcnow(),
            'event_type': type(event).__name__,
            'event': event,
        })

        # Update state
        state = self.order_states[order_id]
        state['last_updated'] = datetime.utcnow()

        # Handle specific events
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
        """Handle complete fill"""
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

        self.log.info(f"Order completely filled: {order_id}")

    def _handle_partial_fill(self, event, order_id: str, state: dict):
        """Handle partial fill"""
        state['status'] = 'PARTIALLY_FILLED'
        state['filled_qty'] += event.last_qty
        state['filled_amount'] += event.last_qty * event.last_px
        state['commission'] += event.commission

        self.log.info(f"Order partially filled: {order_id}, "
                     f"Fill quantity: {event.last_qty}, "
                     f"Total filled quantity: {state['filled_qty']}")

    def _handle_cancellation(self, event, order_id: str, state: dict):
        """Handle cancellation"""
        state['status'] = 'CANCELED'
        state['canceled_at'] = datetime.fromtimestamp(event.ts_event / 1e9)

        if order_id in self.active_orders:
            self.active_orders.remove(order_id)

        self.log.info(f"Order canceled: {order_id}")

    def _handle_rejection(self, event, order_id: str, state: dict):
        """Handle rejection"""
        state['status'] = 'REJECTED'
        state['rejected_at'] = datetime.fromtimestamp(event.ts_event / 1e9)
        state['rejection_reason'] = event.reason

        if order_id in self.active_orders:
            self.active_orders.remove(order_id)

        self.log.warning(f"Order rejected: {order_id}, Reason: {event.reason}")

    def _handle_expiry(self, event, order_id: str, state: dict):
        """Handle expiry"""
        state['status'] = 'EXPIRED'
        state['expired_at'] = datetime.fromtimestamp(event.ts_event / 1e9)

        if order_id in self.active_orders:
            self.active_orders.remove(order_id)

        self.log.warning(f"Order expired: {order_id}")

    def get_order_state(self, order_id: str) -> Optional[dict]:
        """Get order state"""
        return self.order_states.get(order_id)

    def get_active_orders(self) -> List[str]:
        """Get active orders"""
        return list(self.active_orders)

    def get_order_history(self, order_id: str) -> List[dict]:
        """Get order history"""
        return self.order_history.get(order_id, [])

    def get_order_stats(self) -> dict:
        """Get order statistics"""
        total_orders = len(self.order_states)
        active_count = len(self.active_orders)
        filled_count = len(self.filled_orders)

        # Calculate success rate
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


# Strategy Using Lifecycle Management
class LifecycleManagedStrategy(Strategy):
    """Lifecycle Managed Strategy"""

    def __init__(self):
        """Initialize strategy"""
        super().__init__()
        self.lifecycle_manager = OrderLifecycleManager()

    def on_order_initialized(self, event: OrderInitialized):
        """Handle order initialization"""
        self.lifecycle_manager.update_order_state(event)

    def on_order_submitted(self, event: OrderSubmitted):
        """Handle order submission"""
        self.lifecycle_manager.update_order_state(event)

    def on_order_accepted(self, event: OrderAccepted):
        """Handle order acceptance"""
        self.lifecycle_manager.update_order_state(event)

    def on_order_rejected(self, event: OrderRejected):
        """Handle order rejection"""
        self.lifecycle_manager.update_order_state(event)
        self._handle_rejection(event)

    def on_order_canceled(self, event: OrderCanceled):
        """Handle order cancellation"""
        self.lifecycle_manager.update_order_state(event)

    def on_order_expired(self, event: OrderExpired):
        """Handle order expiry"""
        self.lifecycle_manager.update_order_state(event)

    def on_order_triggered(self, event: OrderTriggered):
        """Handle order trigger"""
        self.lifecycle_manager.update_order_state(event)

    def on_order_pending_update(self, event: OrderPendingUpdate):
        """Handle order pending update"""
        self.lifecycle_manager.update_order_state(event)

    def on_order_pending_cancel(self, event: OrderPendingCancel):
        """Handle order pending cancel"""
        self.lifecycle_manager.update_order_state(event)

    def on_order_filled(self, event: OrderFilled):
        """Handle order fill"""
        self.lifecycle_manager.update_order_state(event)
        self._handle_fill(event)

    def on_order_partially_filled(self, event: OrderPartiallyFilled):
        """Handle partial fill"""
        self.lifecycle_manager.update_order_state(event)

    def _handle_rejection(self, event: OrderRejected):
        """Handle order rejection"""
        self.log.error(
            f"Order rejected: {event.order_id}, "
            f"Reason: {event.reason}, "
            f"Strategy: {event.strategy_id}"
        )

        # Implement retry logic here
        self._maybe_retry_order(event.order)

    def _handle_fill(self, event: OrderFilled):
        """Handle order fill"""
        self.log.info(
            f"Order filled: {event.order_id}, "
            f"Quantity: {event.last_qty}, "
            f"Price: {event.last_px}, "
            f"Commission: {event.commission}"
        )

        # Update strategy state
        self._update_strategy_after_fill(event)

    def _update_strategy_after_fill(self, event: OrderFilled):
        """Update strategy state (to be implemented by subclasses)"""
        pass

    def _maybe_retry_order(self, rejected_order):
        """Possibly retry rejected order (to be implemented by subclasses)"""
        pass

    def get_order_stats(self):
        """Get order statistics"""
        return self.lifecycle_manager.get_order_stats()
```

## 8.3 Order Combination Strategies

### 8.3.1 OCO (One Cancels Other)

```python
"""
OCO Order Strategy
"""

from nautilus_trader.model.orders import OrderList
from nautilus_trader.model.enums import OrderType


class OCOOrderStrategy(Strategy):
    """OCO Order Strategy"""

    def __init__(self):
        """Initialize strategy"""
        super().__init__()

        self.oco_order_lists = {}  # oco_id -> order_list
        self.active_oco_sets = {}  # oco_id -> orders

    def create_oco_orders(
        self,
        take_profit_price: Price,
        stop_loss_price: Price,
        quantity: Decimal,
    ) -> OrderList:
        """Create OCO order group"""

        # Take profit limit order
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

        # Stop loss market order
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

        # Create OCO order list
        oco_orders = OrderList(
            orders=[take_profit_order, stop_loss_order],
            oco=True,  # One cancels the other
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
            f"Created OCO order group - ID: {oco_orders.id}, "
            f"Take profit: {take_profit_price}, "
            f"Stop loss: {stop_loss_price}"
        )

        return oco_orders

    def submit_oco_orders(self, oco_orders: OrderList):
        """Submit OCO orders"""
        self.submit_order_list(oco_orders)

    def on_order_filled(self, event: OrderFilled):
        """Handle order fill"""
        # Check if it's an OCO order
        for oco_id, oco_info in self.active_oco_sets.items():
            if event.order_id in [oco_info['take_profit_id'], oco_info['stop_loss_id']]:
                # Mark OCO group as filled
                oco_info['filled'] = True
                oco_info['filled_price'] = event.last_px
                oco_info['filled_time'] = datetime.fromtimestamp(event.ts_event / 1e9)

                # The other order should be automatically canceled
                self._cancel_oco_partner(oco_id, event.order_id)

                # Record OCO execution result
                self._log_oco_result(oco_id, event.order_id, event)

                break

    def on_order_canceled(self, event: OrderCanceled):
        """Handle order cancellation"""
        # Check if it's an OCO partner order being canceled
        for oco_id, oco_info in self.active_oco_sets.items():
            if event.order_id in [oco_info['take_profit_id'], oco_info['stop_loss_id']]:
                if not oco_info['filled']:
                    oco_info['canceled'] = True
                    oco_info['canceled_reason'] = 'PARTNER_CANCELED'
                    self._log_oco_cancellation(oco_id, event)
                break

    def _cancel_oco_partner(self, oco_id: str, filled_order_id: str):
        """Cancel OCO partner order"""
        oco_info = self.active_oco_sets[oco_id]

        # Determine which order to cancel
        if filled_order_id == oco_info['take_profit_id']:
            partner_id = oco_info['stop_loss_id']
            partner_type = 'Stop loss order'
        else:
            partner_id = oco_info['take_profit_id']
            partner_type = 'Take profit order'

        # Cancel partner order
        if partner_id:
            self.cancel_order(partner_id)
            self.log.info(f"OCO execution - Auto-cancel {partner_type}: {partner_id}")

    def _log_oco_result(self, oco_id: str, filled_order_id: str, fill_event):
        """Log OCO execution result"""
        oco_info = self.active_oco_sets[oco_id]

        filled_type = 'Take profit' if filled_order_id == oco_info['take_profit_id'] else 'Stop loss'
        duration = oco_info['filled_time'] - oco_info['created_at']

        self.log.info(
            f"OCO order group {oco_id} execution complete - "
            f"{filled_type} filled: {fill_event.last_px}, "
            f"Execution time: {duration.total_seconds():.1f} seconds"
        )

    def _log_oco_cancellation(self, oco_id: str, cancel_event):
        """Log OCO cancellation"""
        oco_info = self.active_oco_sets[oco_id]

        partner_order_id = (oco_info['take_profit_id'] if cancel_event.order_id == oco_info['stop_loss_id']
                        else oco_info['stop_loss_id'])

        partner_type = 'Stop loss order' if partner_order_id == oco_info['stop_loss_id'] else 'Take profit order'

        self.log.info(
            f"OCO order group {oco_id} partially executed - "
            f"Cancel {partner_type}: {partner_order_id}"
        )

    def cleanup_completed_oco_sets(self):
        """Clean up completed OCO groups"""
        completed_oco = []

        for oco_id, oco_info in self.active_oco_sets.items():
            if oco_info['filled']:
                completed_oco.append(oco_id)
                continue

            # Check if all orders are completed
            all_completed = True
            for order_id in [oco_info['take_profit_id'], oco_info['stop_loss_id']]:
                state = self.cache.order(order_id)
                if state and state.status not in ['FILLED', 'CANCELED']:
                    all_completed = False
                    break

            if all_completed:
                completed_oco.append(oco_id)

        # Cleanup
        for oco_id in completed_oco:
            del self.active_oco_sets[oco_id]
            if oco_id in self.oco_order_lists:
                del self.oco_order_lists[oco_id]

        if completed_oco:
            self.log.info(f"Cleaned up {len(completed_oco)} OCO order groups")


# Example of Using OCO Strategy
class OCOTradingStrategy(Strategy):
    """Trading Strategy Using OCO"""

    def __init__(self):
        """Initialize strategy"""
        super().__init__()
        self.oco_manager = OCOOrderStrategy()
        self.current_position = None

    def on_order_filled(self, event: OrderFilled):
        """Handle order fill"""
        self.oco_manager.on_order_filled(event)

        # Check if it's opening a position
        if event.order_side == OrderSide.BUY:
            self._handle_position_open(event)
        # OCO fill should be closing a position
        else:
            self._handle_position_close(event)

    def _handle_position_open(self, event: OrderFilled):
        """Handle position open"""
        self.current_position = {
            'entry_price': event.last_px,
            'quantity': event.last_qty,
            'entry_time': datetime.fromtimestamp(event.ts_event / 1e9),
        }

        # Calculate take profit and stop loss prices
        take_profit_price = self._calculate_take_profit(event.last_px)
        stop_loss_price = self._calculate_stop_loss(event.last_px)

        # Create OCO orders
        oco_orders = self.oco_manager.create_oco_orders(
            take_profit_price,
            stop_loss_price,
            event.last_qty,
        )

        # Submit OCO orders
        self.oco_manager.submit_oco_orders(oco_orders)

        self.log.info(
            f"Position opened - Price: {event.last_px}, "
            f"Set OCO take profit: {take_profit_price}, "
            f"Stop loss: {stop_loss_price}"
        )

    def _calculate_take_profit(self, entry_price: Decimal) -> Price:
        """Calculate take profit price"""
        risk_reward_ratio = 2.0  # 1:2 risk-reward ratio
        risk_amount = entry_price * Decimal("0.02")  # 2% risk
        take_profit_amount = risk_amount * risk_reward_ratio
        take_profit_price = entry_price + take_profit_amount

        return Price(take_profit_price)

    def _calculate_stop_loss(self, entry_price: Decimal) -> Price:
        """Calculate stop loss price"""
        stop_distance = entry_price * Decimal("0.02")  # 2% stop loss
        stop_loss_price = entry_price - stop_distance

        return Price(stop_loss_price)

    def _handle_position_close(self, event: OrderFilled):
        """Handle position close"""
        if self.current_position:
            duration = datetime.fromtimestamp(event.ts_event / 1e9) - self.current_position['entry_time']
            pnl = (event.last_px - self.current_position['entry_price']) * self.current_position['quantity']

            self.log.info(
                f"Position closed - Entry price: {self.current_position['entry_price']}, "
                f"Exit price: {event.last_px}, "
                f"P&L: {pnl}, "
                f"Holding time: {duration.total_seconds():.1f} seconds"
            )

            self.current_position = None

            # Clean up OCO orders
            self.oco_manager.cleanup_completed_oco_sets()
```

### 8.3.2 OTO (One Triggers Other)

```python
"""
OTO Order Strategy
"""

from nautilus_trader.model.orders import OrderList
from nautilus_trader.model.enums import OrderType


class OTOOrderStrategy(Strategy):
    """OTO Order Strategy"""

    def __init__(self):
        """Initialize strategy"""
        super().__init__()

        self.oto_order_lists = {}  # oto_id -> order_list
        self.active_oto_sets = {}  # oto_id -> orders

    def create_oto_orders(
        self,
        primary_order,  # Primary order (limit order)
        secondary_order,  # Trigger order (stop order)
    ) -> OrderList:
        """Create OTO order group"""

        # Create OTO order list
        oto_orders = OrderList(
            orders=[primary_order, secondary_order],
            oto=True,  # One triggers the other
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
            f"Created OTO order group - ID: {oto_orders.id}, "
            f"Primary order: {primary_order}, "
            f"Trigger order: {secondary_order}"
        )

        return oto_orders

    def submit_oto_orders(self, oto_orders: OrderList):
        """Submit OTO orders"""
        self.submit_order_list(oto_orders)

    def on_order_filled(self, event: OrderFilled):
        """Handle order fill"""
        # Check if it's an OTO trigger condition
        for oto_id, oto_info in self.active_oto_sets.items():
            if event.order_id == oto_info['primary_id']:
                # Primary order filled, trigger secondary order
                self._trigger_secondary_order(oto_id, event)
                break

    def on_order_canceled(self, event: OrderCanceled):
        """Handle order cancellation"""
        # Check if it's an OTO order
        for oto_id, oto_info in self.active_oto_sets.items():
            if event.order_id == oto_info['primary_id']:
                # Primary order canceled, auto-cancel secondary order
                self._cancel_secondary_order(oto_id, event)
                break

    def _trigger_secondary_order(self, oto_id: str, trigger_event):
        """Trigger secondary order"""
        oto_info = self.active_oto_sets[oto_id]

        if not oto_info['triggered']:
            oto_info['triggered'] = True
            oto_info['triggered_at'] = datetime.fromtimestamp(trigger_event.ts_event / 1e9)

            # Get secondary order and submit
            secondary_order = self.cache.order(oto_info['secondary_id'])
            if secondary_order and secondary_order.status == OrderStatus.INITIALIZED:
                self.submit_order(secondary_order)
                self.log.info(
                    f"OTO triggered - Submit secondary order: {secondary_order}, "
                    f"Trigger price: {trigger_event.last_px}"
                )

    def _cancel_secondary_order(self, oto_id: str, cancel_event):
        """Cancel secondary order"""
        oto_info = self.active_oto_sets[oto_id]

        if not oto_info['triggered']:
            # Cancel untriggered secondary order
            secondary_order_id = oto_info['secondary_id']
            secondary_order = self.cache.order(secondary_order_id)

            if secondary_order and secondary_order.status in [OrderStatus.INITIALIZED, OrderStatus.SUBMITTED]:
                self.cancel_order(secondary_order_id)
                self.log.warning(
                    f"OTO canceled - Primary order canceled, auto-cancel secondary order: {secondary_order}"
                )

            oto_info['canceled'] = True
            oto_info['canceled_at'] = datetime.utcnow()
            oto_info['cancel_reason'] = 'PRIMARY_CANCELLED'


# Example of Using OTO Strategy
class OTOMomentumStrategy(Strategy):
    """Momentum Strategy Using OTO"""

    def __init__(self):
        """Initialize strategy"""
        super().__init__()
        self.oto_manager = OTOOrderStrategy()

        # Indicators
        self.rsi = RSI(period=14)
        self.ema = EMA(period=20)

        # State
        self.position = None
        self.entry_oto_id = None

    def on_bar(self, bar: Bar):
        """Process bar data"""
        # Update indicators
        self.rsi.update_raw(float(bar.close))
        self.ema.update_raw(float(bar.close))

        if not (self.rsi.initialized and self.ema.initialized):
            return

        # Check trading signals
        if not self.position:
            self._check_entry_signal(bar)
        else:
            self._check_exit_signal(bar)

    def _check_entry_signal(self, bar: Bar):
        """Check entry signal"""
        current_price = bar.close
        ema_value = self.ema.value
        rsi_value = self.rsi.value

        # Buy condition: price > EMA and RSI > 50
        if (current_price > ema_value and rsi_value > 50 and
            not self.entry_oto_id):

            # Set buy limit order (below current price)
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

            # Set stop loss order as trigger order
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

            # Create OTO order
            oto_orders = self.oto_manager.create_oto_orders(
                primary_order,
                secondary_order,
            )

            # Submit OTO
            self.oto_manager.submit_oto_orders(oto_orders)
            self.entry_oto_id = oto_orders.id

            self.log.info(
                f"Submit OTO - Buy order: {entry_price}, "
                f"Stop loss trigger: {stop_loss_price}"
            )

    def _check_exit_signal(self, bar: Bar):
        """Check exit signal"""
        current_price = bar.close
        entry_price = self.position['entry_price']

        # Simple take profit strategy
        if current_price > entry_price * Decimal("1.03"):
            # Market sell
            exit_order = MarketOrder(
                trader_id=self.trader_id,
                strategy_id=self.id,
                instrument_id=self.bar_type.instrument_id,
                order_side=OrderSide.SELL,
                quantity=self.position['quantity'],
            )
            self.submit_order(exit_order)
            self.log.info(f"Take profit exit - Price: {current_price}")

    def on_order_filled(self, event: OrderFilled):
        """Handle order fill"""
        self.oto_manager.on_order_filled(event)

        if event.order_side == OrderSide.BUY and not self.position:
            # Open position
            self.position = {
                'entry_price': event.last_px,
                'quantity': event.last_qty,
                'entry_time': datetime.fromtimestamp(event.ts_event / 1e9),
            }
            self.log.info(f"Position opened - Price: {event.last_px}")

        elif event.order_side == OrderSide.SELL and self.position:
            # Close position
            pnl = (event.last_px - self.position['entry_price']) * self.position['quantity']
            self.log.info(
                f"Position closed - P&L: {pnl}, "
                f"Entry price: {self.position['entry_price']}, "
                f"Exit price: {event.last_px}"
            )
            self.position = None
            self.entry_oto_id = None

    def on_order_rejected(self, event: OrderRejected):
        """Handle order rejection"""
        self.oto_manager.on_order_rejected(event)

        # If primary order is rejected, retry
        if self.entry_oto_id:
            for oto_id, oto_info in self.oto_manager.active_oto_sets.items():
                if (oto_id == self.entry_oto_id and
                    event.order_id == oto_info['primary_id']):
                    # Clean up OTO and retry
                    del self.oto_manager.active_oto_sets[oto_id]
                    self.entry_oto_id = None
                    break
```

## 8.4 Order Execution Algorithms

### 8.4.1 TWAP (Time Weighted Average Price)

```python
"""
TWAP Execution Algorithm
"""

from decimal import Decimal
from typing import List, Dict
from datetime import datetime, timedelta
import numpy as np


class TWAPExecutor:
    """TWAP Executor"""

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
        Initialize TWAP Executor

        Parameters
        ----------
        trader_id : TraderId
            Trader ID
        strategy_id : StrategyId
            Strategy ID
        instrument_id : InstrumentId
            Instrument ID
        total_quantity : Decimal
            Total execution quantity
        duration_seconds : int
            Execution duration (seconds)
        num_slices : int
            Number of slices
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

        self.orders = []  # Submitted orders
        self.filled_orders = []  # Filled orders
        self.start_time = None
        self.end_time = None

        self.is_active = False
        self.is_completed = False

    def _calculate_slice_quantities(self) -> List[Decimal]:
        """Calculate slice quantities"""
        # Even distribution
        base_quantity = self.total_quantity / self.num_slices

        # Add random perturbation
        random_factors = np.random.uniform(0.8, 1.2, self.num_slices)
        random_factors /= np.mean(random_factors)  # Normalize

        quantities = [base_quantity * Decimal(str(f)) for f in random_factors]
        quantities = [round(q, 6) for q in quantities]

        # Ensure sum equals total
        total = sum(quantities)
        if total != self.total_quantity:
            # Adjust the last slice
            quantities[-1] += (self.total_quantity - total)

        return quantities

    def start_execution(self, start_time_ns: int):
        """Start execution"""
        self.start_time = datetime.fromtimestamp(start_time_ns / 1e9)
        self.end_time = self.start_time + timedelta(seconds=self.duration)
        self.is_active = True

        # Create and submit all orders
        self._create_and_submit_orders()

    def _create_and_submit_orders(self):
        """Create and submit all orders"""
        current_time = self.start_time
        remaining = self.remaining_quantity

        for i, (slice_qty, slice_num) in enumerate(
            zip(self.slice_quantities, range(self.num_slices))
        ):
            if slice_qty <= 0:
                continue

            # Calculate order time
            order_time = current_time + timedelta(
                seconds=(slice_num + 0.5) * self.slice_interval
            )

            # Use limit order, price based on current market price
            # Should get current price here
            current_price = self._get_current_price()

            # Add random price perturbation
            price_adjustment = np.random.uniform(-0.0005, 0.0005)  # ±0.05%
            limit_price = current_price * (1 + price_adjustment)

            # Create limit order
            order = LimitOrder(
                trader_id=self.trader_id,
                strategy_id=self.strategy_id,
                instrument_id=self.instrument_id,
                order_side=OrderSide.BUY,  # Can be set as needed
                quantity=Quantity(slice_qty),
                price=Price(limit_price),
                time_in_force=TimeInForce.GTC,
                expire_time=int(order_time.timestamp() * 1e9),
                post_only=True,  # Maker only, avoid fees
            )

            # Should submit order here
            self.orders.append({
                'order': order,
                'slice_number': slice_num,
                'quantity': slice_qty,
                'scheduled_time': order_time,
                'status': 'SCHEDULED',
            })

    def _get_current_price(self) -> Decimal:
        """Get current price (example implementation)"""
        # Should get from cache or data engine here
        # Return simulated value
        return Decimal("50000.00")

    def update_execution(self, current_time_ns: int):
        """Update execution status"""
        if not self.is_active or self.is_completed:
            return

        current_time = datetime.fromtimestamp(current_time_ns / 1e9)

        # Check for timeout
        if current_time > self.end_time:
            self._complete_execution("TIMEOUT")
            return

        # Check if completed
        if self.remaining_quantity <= 0:
            self._complete_execution("COMPLETED")
            return

        # Can add dynamic adjustment logic here
        # For example, adjust remaining orders based on market conditions

    def on_order_filled(self, event):
        """Handle order fill"""
        order_id = event.order_id

        # Find corresponding order info
        for order_info in self.orders:
            if order_info['order'].client_order_id == order_id:
                order_info['status'] = 'FILLED'
                order_info['fill_time'] = datetime.fromtimestamp(event.ts_event / 1e9)
                order_info['fill_price'] = event.last_px
                order_info['fill_qty'] = event.last_qty

                self.filled_orders.append(order_info)
                self.remaining_quantity -= event.last_qty

                break

        # Check if need to cancel remaining orders
        if self.remaining_quantity <= 0:
            self._cancel_remaining_orders()

    def _cancel_remaining_orders(self):
        """Cancel remaining orders"""
        for order_info in self.orders:
            if order_info['status'] == 'SCHEDULED':
                # Should cancel order here
                order_info['status'] = 'CANCELED'

    def _complete_execution(self, reason: str):
        """Complete execution"""
        self.is_active = False
        self.is_completed = True
        self.completion_time = datetime.utcnow()
        self.completion_reason = reason

        # Cancel all unfilled orders
        self._cancel_remaining_orders()

    def get_execution_stats(self) -> dict:
        """Get execution statistics"""
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


# Strategy Using TWAP
class TWAPStrategy(Strategy):
    """Strategy Using TWAP Execution Algorithm"""

    def __init__(self):
        """Initialize strategy"""
        super().__init__()

        self.twap_executors = {}  # execution_id -> executor
        self.next_execution_id = 1

    def create_twap_execution(
        self,
        quantity: Decimal,
        duration_minutes: int,
        num_slices: int = 20,
    ) -> str:
        """Create TWAP execution"""

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
        """Start TWAP execution"""
        if execution_id not in self.twap_executors:
            self.log.error(f"TWAP executor does not exist: {execution_id}")
            return

        executor = self.twap_executors[execution_id]
        executor.start_execution(self.clock.timestamp_ns())

        # Submit all orders
        for order_info in executor.orders:
            self.submit_order(order_info['order'])

        self.log.info(f"Started TWAP execution: {execution_id}")

    def update_twap_executions(self):
        """Update TWAP execution status"""
        current_time = self.clock.timestamp_ns()

        for executor in self.twap_executors.values():
            executor.update_execution(current_time)

    def on_order_filled(self, event):
        """Handle order fill"""
        # Check if it's a TWAP order
        for executor in self.twap_executors.values():
            executor.on_order_filled(event)

    def get_twap_stats(self, execution_id: str) -> dict:
        """Get TWAP statistics"""
        if execution_id not in self.twap_executors:
            return {}

        return self.twap_executors[execution_id].get_execution_stats()

    def on_bar(self, bar: Bar):
        """Process bar data"""
        # Update TWAP executions
        self.update_twap_executions()

        # Simple example: create a new TWAP execution every hour
        if self.count_bars() % 60 == 0:  # Assuming hourly bars
            # Create 0.1 BTC TWAP execution, lasting 30 minutes
            execution_id = self.create_twap_execution(
                quantity=Decimal("0.1"),
                duration_minutes=30,
                num_slices=30,
            )

            # Start execution immediately
            self.start_twap_execution(execution_id)

    def on_stop(self):
        """Strategy stop"""
        # Complete all TWAP executions
        for executor in self.twap_executors.values():
            if executor.is_active:
                executor._complete_execution("STRATEGY_STOP")

        # Output statistics
        self._log_twap_stats()

    def _log_twap_stats(self):
        """Log TWAP statistics"""
        self.log.info("TWAP execution statistics:")

        for execution_id, executor in self.twap_executors.items():
            stats = executor.get_execution_stats()
            if stats:
                self.log.info(
                    f"Executor {execution_id}: "
                    f"Execution rate: {stats['filled_rate']:.1%}, "
                    f"Average price: {stats['average_fill_price']:.2f}"
                )
```

## 8.5 Next Steps

In this chapter, we learned:

1. Core concepts of the NautilusTrader order system
2. How to use various order types
3. Order lifecycle management
4. Advanced order strategies (OCO, OTO)
5. Execution algorithms (TWAP)

In the next chapter, we will learn:

1. Portfolio management
2. Risk control mechanisms
3. Capital management strategies
4. Multi-strategy coordination
5. Performance analysis

## 8.6 Summary

### Key Points
1. NautilusTrader supports all common order types
2. Order lifecycle management provides complete tracking capabilities
3. Advanced order strategies like OCO and OTO facilitate risk control
4. Execution algorithms can optimize the execution of large orders

### Best Practices
1. Always set stop orders to manage risk
2. Use OCO strategy to implement take profit and stop loss
3. Use algorithms like TWAP for large orders to avoid market impact
4. Monitor order status and handle exceptions promptly

## 8.7 References

- [Order Management API Documentation](https://nautilustrader.io/docs/latest/api_reference/orders/)
- [Execution Algorithm Best Practices](https://nautilustrader.io/docs/latest/concepts/execution/)
- [Risk Management Guide](https://nautilustrader.io/docs/latest/concepts/risk/)
- [OCO/OTO Order Strategies](https://www.investopedia.com/terms/o/onecancelsother.asp)