# Chapter 18: Custom Component Development

## Learning Objectives

- Understand NautilusTrader's component architecture and extension mechanisms
- Learn to develop custom strategies, execution clients, and data adapters
- Master plugin development and usage methods
- Implement reusable trading components

## 18.1 Custom Strategy Development

### 18.1.1 Strategy Base Class and Lifecycle

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
    """Custom Strategy Template"""

    # Parameter configuration
    def __init__(self, config):
        super().__init__()

        # Required configuration
        self.instrument_id = InstrumentId.from_str(config['instrument_id'])

        # Optional configuration
        self.order_size = Quantity.from_int(config.get('order_size', 100))
        self.max_orders = config.get('max_orders', 5)
        self.order_timeout_seconds = config.get('order_timeout', 60)

        # State variables
        self.is_running = False
        self.active_orders = {}
        self.position = None
        self.last_price = None

        # Performance statistics
        self.order_count = 0
        self.filled_count = 0
        self.total_pnl = Decimal('0')

    # ==================== Lifecycle Methods ====================

    def on_start(self):
        """Called when strategy starts"""
        self.is_running = True

        # Subscribe to data streams
        self.subscribe_quote_ticks(self.instrument_id)
        self.subscribe_trade_ticks(self.instrument_id)

        # Subscribe to bars (if needed)
        if hasattr(self, 'bar_type'):
            self.subscribe_bars(self.bar_type)

        # Initialize timers
        self._setup_timers()

        self.log.info(f"Strategy {self.__class__.__name__} started for {self.instrument_id}")

    def on_stop(self):
        """Called when strategy stops"""
        self.is_running = False

        # Cancel all active orders
        asyncio.create_task(self._cancel_all_orders())

        self.log.info(f"Strategy {self.__class__.__name__} stopped")

    # ==================== Data Processing Methods ====================

    def on_quote_tick(self, tick: QuoteTick):
        """Process quote data"""
        self.last_price = (tick.bid + tick.ask) / 2

        # Implement strategy logic
        if self.is_running:
            self._process_quote_tick(tick)

    def on_trade_tick(self, tick: TradeTick):
        """Process trade data"""
        # Implement strategy logic
        if self.is_running:
            self._process_trade_tick(tick)

    def on_bar(self, bar: Bar):
        """Process bar data"""
        # Implement strategy logic
        if self.is_running:
            self._process_bar(bar)

    # ==================== Order Event Handling ====================

    def on_order_submitted(self, event):
        """Order submitted event"""
        order_id = event.order_id
        self.active_orders[order_id] = {
            'status': 'submitted',
            'submitted_at': event.ts_event
        }
        self.order_count += 1

    def on_order_accepted(self, event):
        """Order accepted event"""
        order_id = event.order_id
        if order_id in self.active_orders:
            self.active_orders[order_id]['status'] = 'accepted'
            self.active_orders[order_id]['accepted_at'] = event.ts_event

    def on_order_rejected(self, event):
        """Order rejected event"""
        order_id = event.order_id
        if order_id in self.active_orders:
            del self.active_orders[order_id]

        self.log.warning(f"Order rejected: {event.order_id}")

    def on_order_filled(self, event: OrderFilled):
        """Order filled notification"""
        order_id = event.order_id
        if order_id in self.active_orders:
            self.active_orders[order_id]['status'] = 'filled'
            self.active_orders[order_id]['filled_at'] = event.ts_event
            self.active_orders[order_id]['fill_price'] = event.last_px

        self.filled_count += 1

        # Handle fill
        self._handle_order_filled(event)

    def on_order_canceled(self, event):
        """Order canceled event"""
        order_id = event.order_id
        if order_id in self.active_orders:
            del self.active_orders[order_id]

    # ==================== Position Event Handling ====================

    def on_position_opened(self, event):
        """Position opened event"""
        self.position = event.position
        self.log.info(f"Position opened: {self.position}")

    def on_position_changed(self, event):
        """Position changed event"""
        self.position = event.position

    def on_position_closed(self, event: PositionClosed):
        """Position closed event"""
        # Update PnL
        self.total_pnl += event.pnl
        self.position = None

        self.log.info(f"Position closed. PnL: {event.pnl}")

    # ==================== Custom Methods ====================

    def _setup_timers(self):
        """Setup timers"""
        # Example: check every minute
        self.set_timer("strategy_check", 60.0, self._strategy_check)

    async def _strategy_check(self):
        """Strategy periodic check"""
        # Clean up expired orders
        await self._cleanup_expired_orders()

        # Other periodic check logic
        pass

    def _process_quote_tick(self, tick: QuoteTick):
        """Subclass implementation for processing quote data"""
        raise NotImplementedError("Subclass must implement _process_quote_tick")

    def _process_trade_tick(self, tick: TradeTick):
        """Subclass implementation for processing trade data"""
        pass

    def _process_bar(self, bar: Bar):
        """Subclass implementation for processing bar data"""
        pass

    def _handle_order_filled(self, event: OrderFilled):
        """Subclass implementation for handling order fills"""
        pass

    async def _place_order(self, side: OrderSide, quantity: Quantity,
                           order_type: OrderType = OrderType.MARKET,
                           price: Price = None):
        """Common method for placing orders"""
        # Check order limit
        if len(self.active_orders) >= self.max_orders:
            self.log.warning(f"Max orders limit reached: {self.max_orders}")
            return None

        # Create order
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

        # Submit order
        order_id = await self.submit_order(order)
        return order_id

    async def _cancel_order(self, order_id):
        """Cancel order"""
        await self.cancel_order(order_id)

    async def _cancel_all_orders(self):
        """Cancel all orders"""
        for order_id in list(self.active_orders.keys()):
            await self._cancel_order(order_id)

    async def _cleanup_expired_orders(self):
        """Clean up expired orders"""
        current_time = self.clock.timestamp()

        expired_orders = []
        for order_id, order_info in self.active_orders.items():
            if (current_time - order_info['submitted_at']) > self.order_timeout_seconds * 1e9:
                expired_orders.append(order_id)

        for order_id in expired_orders:
            self.log.info(f"Cancelling expired order: {order_id}")
            await self._cancel_order(order_id)

    def get_strategy_stats(self):
        """Get strategy statistics"""
        return {
            'is_running': self.is_running,
            'active_orders': len(self.active_orders),
            'total_orders': self.order_count,
            'filled_orders': self.filled_count,
            'fill_rate': self.filled_count / max(self.order_count, 1),
            'total_pnl': float(self.total_pnl),
            'current_position': self.position.quantity if self.position else 0
        }

# Usage example
class MovingAverageCrossStrategy(CustomStrategyTemplate):
    """Moving Average Crossover Strategy Example"""

    def __init__(self, config):
        super().__init__(config)

        # Strategy parameters
        self.fast_period = config.get('fast_period', 10)
        self.slow_period = config.get('slow_period', 30)

        # Data storage
        self.price_history = []
        self.fast_ma = None
        self.slow_ma = None
        self.last_signal = None  # 1: long, -1: short, 0: none

    def _process_quote_tick(self, tick: QuoteTick):
        """Process quote data"""
        # Update price history
        price = float(tick.mid)
        self.price_history.append(price)

        # Maintain fixed length
        max_history = self.slow_period * 2
        if len(self.price_history) > max_history:
            self.price_history.pop(0)

        # Calculate moving averages
        if len(self.price_history) >= self.slow_period:
            self.fast_ma = sum(self.price_history[-self.fast_period:]) / self.fast_period
            self.slow_ma = sum(self.price_history[-self.slow_period:]) / self.slow_period

            # Generate trading signals
            self._generate_signals()

    def _generate_signals(self):
        """Generate trading signals"""
        if self.fast_ma is None or self.slow_ma is None:
            return

        # Check for golden cross (fast line crosses above slow line)
        if self.fast_ma > self.slow_ma and self.last_signal != 1:
            self._execute_buy_signal()
            self.last_signal = 1

        # Check for death cross (fast line crosses below slow line)
        elif self.fast_ma < self.slow_ma and self.last_signal != -1:
            self._execute_sell_signal()
            self.last_signal = -1

    async def _execute_buy_signal(self):
        """Execute buy signal"""
        # Check current position
        if self.position and self.position.quantity > 0:
            return  # Already have long position

        # Close short position (if any)
        if self.position and self.position.quantity < 0:
            close_size = Quantity.from_int(abs(self.position.quantity))
            await self._place_order(OrderSide.BUY, close_size)

        # Open long position
        await self._place_order(OrderSide.BUY, self.order_size)
        self.log.info(f"Buy signal executed at price {self.last_price}")

    async def _execute_sell_signal(self):
        """Execute sell signal"""
        # Check current position
        if self.position and self.position.quantity < 0:
            return  # Already have short position

        # Close long position (if any)
        if self.position and self.position.quantity > 0:
            close_size = Quantity.from_int(self.position.quantity)
            await self._place_order(OrderSide.SELL, close_size)

        # Open short position
        await self._place_order(OrderSide.SELL, self.order_size)
        self.log.info(f"Sell signal executed at price {self.last_price}")
```

### 18.1.2 Advanced Strategy Features

```python
from abc import ABC, abstractmethod
from typing import Dict, List, Callable
import pandas as pd
import numpy as np

class StrategyState:
    """Strategy State Management"""

    def __init__(self):
        self.data = {}
        self.history = []

    def set(self, key: str, value):
        """Set state value"""
        old_value = self.data.get(key)
        self.data[key] = value

        # Record change history
        if old_value != value:
            self.history.append({
                'timestamp': pd.Timestamp.now(),
                'key': key,
                'old_value': old_value,
                'new_value': value
            })

    def get(self, key: str, default=None):
        """Get state value"""
        return self.data.get(key, default)

    def save_snapshot(self):
        """Save state snapshot"""
        return {
            'data': self.data.copy(),
            'timestamp': pd.Timestamp.now()
        }

    def load_snapshot(self, snapshot: dict):
        """Load state snapshot"""
        self.data = snapshot['data'].copy()

class RiskManager:
    """Risk Manager"""

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
        """Check order risk"""
        # Check position limit
        if order_side == OrderSide.BUY:
            new_position = current_position + quantity.as_double()
        else:
            new_position = current_position - quantity.as_double()

        if abs(new_position) > self.max_position_size:
            return False

        return True

    def check_position_risk(self, position) -> dict:
        """Check position risk"""
        if not position:
            return {'action': 'none'}

        unrealized_pnl = position.unrealized_pnl(self.clock.timestamp())
        portfolio_value = self._get_portfolio_value()

        # Update max portfolio value
        self.max_portfolio_value = max(self.max_portfolio_value, portfolio_value)

        # Calculate drawdown
        drawdown = (self.max_portfolio_value - portfolio_value) / self.max_portfolio_value

        # Check stop loss
        if unrealized_pnl < -self.stop_loss_pct * portfolio_value:
            return {'action': 'stop_loss', 'reason': 'Stop loss triggered'}

        # Check take profit
        if unrealized_pnl > self.take_profit_pct * portfolio_value:
            return {'action': 'take_profit', 'reason': 'Take profit triggered'}

        # Check max drawdown
        if drawdown > self.max_drawdown:
            return {'action': 'reduce_position', 'reason': 'Max drawdown exceeded'}

        return {'action': 'none'}

    def _get_portfolio_value(self):
        """Get portfolio value"""
        # Implement portfolio value calculation
        return 100000  # Example value

class AdvancedStrategy(Strategy):
    """Advanced Strategy Base Class"""

    def __init__(self, config):
        super().__init__(config)

        # State management
        self.state = StrategyState()

        # Risk management
        self.risk_manager = RiskManager(config.get('risk', {}))

        # Signal generators
        self.signal_generators = []

        # Order manager
        self.order_manager = OrderManager(self)

        # Performance tracking
        self.performance_tracker = PerformanceTracker()

    def add_signal_generator(self, generator: 'SignalGenerator'):
        """Add signal generator"""
        self.signal_generators.append(generator)

    def generate_signals(self) -> List[dict]:
        """Generate all signals"""
        signals = []
        for generator in self.signal_generators:
            signal = generator.generate(self)
            if signal:
                signals.append(signal)
        return signals

    def execute_signals(self, signals: List[dict]):
        """Execute signals"""
        for signal in signals:
            # Risk check
            if not self.risk_manager.check_order_risk(
                signal['side'],
                signal['quantity'],
                self.position.quantity if self.position else 0
            ):
                continue

            # Execute order
            self.order_manager.place_order(signal)

class SignalGenerator(ABC):
    """Signal Generator Base Class"""

    @abstractmethod
    def generate(self, strategy: AdvancedStrategy) -> dict:
        """Generate signal"""
        pass

class MeanReversionSignal(SignalGenerator):
    """Mean Reversion Signal Generator"""

    def __init__(self, lookback: int = 20, threshold: float = 2.0):
        self.lookback = lookback
        self.threshold = threshold

    def generate(self, strategy: AdvancedStrategy) -> dict:
        """Generate mean reversion signal"""
        if not hasattr(strategy, 'price_history') or len(strategy.price_history) < self.lookback:
            return None

        prices = strategy.price_history[-self.lookback:]
        mean_price = np.mean(prices)
        std_price = np.std(prices)
        current_price = strategy.price_history[-1]

        # Calculate Z-score
        zscore = (current_price - mean_price) / std_price if std_price > 0 else 0

        # Generate signal
        if zscore > self.threshold:  # Price too high, sell
            return {
                'type': 'sell',
                'strength': min(zscore / self.threshold, 3.0),
                'reason': f'Price {zscore:.2f} std above mean'
            }
        elif zscore < -self.threshold:  # Price too low, buy
            return {
                'type': 'buy',
                'strength': min(abs(zscore) / self.threshold, 3.0),
                'reason': f'Price {abs(zscore):.2f} std below mean'
            }

        return None

class OrderManager:
    """Order Manager"""

    def __init__(self, strategy: Strategy):
        self.strategy = strategy
        self.orders = {}
        self.order_groups = {}  # Order group management

    def place_order(self, signal: dict):
        """Place order"""
        order_config = {
            'side': OrderSide.BUY if signal['type'] == 'buy' else OrderSide.SELL,
            'quantity': self._calculate_quantity(signal),
            'order_type': OrderType.MARKET
        }

        # Create and submit order
        order = self.strategy.order_factory.market(
            instrument_id=self.strategy.instrument_id,
            side=order_config['side'],
            quantity=order_config['quantity']
        )

        # Add to order group
        group_id = signal.get('group_id', 'default')
        if group_id not in self.order_groups:
            self.order_groups[group_id] = []
        self.order_groups[group_id].append(order)

        # Submit order
        asyncio.create_task(self.strategy.submit_order(order))

    def _calculate_quantity(self, signal: dict) -> Quantity:
        """Calculate order quantity"""
        # Adjust quantity based on signal strength
        base_quantity = self.strategy.order_size.as_double()
        strength = signal.get('strength', 1.0)

        adjusted_quantity = int(base_quantity * strength)
        return Quantity.from_int(max(1, adjusted_quantity))

class PerformanceTracker:
    """Performance Tracker"""

    def __init__(self):
        self.trades = []
        self.pnl_history = []
        self.metrics = {}

    def record_trade(self, trade: dict):
        """Record trade"""
        self.trades.append(trade)

    def update_metrics(self):
        """Update performance metrics"""
        if not self.trades:
            return

        # Calculate various metrics
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

## 18.2 Custom Execution Client

### 18.2.1 Execution Client Base Class

```python
from nautilus_trader.execution.client import ExecutionClient
from nautilus_trader.model.enums import LiquiditySide, OrderSide
from nautilus_trader.model.objects import Quantity, Price
from nautilus_trader.core.uuid import UUID4

class CustomExecutionClient(ExecutionClient):
    """Custom Execution Client Base Class"""

    def __init__(self, config):
        super().__init__(
            name=config.get('name', 'CustomExecClient'),
            config=config
        )

        # Client configuration
        self.base_url = config.get('base_url')
        self.api_key = config.get('api_key')
        self.api_secret = config.get('api_secret')

        # State
        self.connected = False
        self.account_id = None
        self.order_id_map = {}  # Map internal order IDs to exchange order IDs

    async def connect(self):
        """Connect to exchange"""
        try:
            # Implement connection logic
            await self._authenticate()

            # Get account information
            account_info = await self._get_account_info()
            self.account_id = account_info['id']

            # Start data streams
            await self._start_data_streams()

            self.connected = True
            self.log.info(f"Connected to {self.__class__.__name__}")

        except Exception as e:
            self.log.error(f"Failed to connect: {e}")
            raise

    async def disconnect(self):
        """Disconnect"""
        if self.connected:
            await self._stop_data_streams()
            self.connected = False
            self.log.info("Disconnected from exchange")

    async def submit_order(self, order):
        """Submit order"""
        if not self.connected:
            raise RuntimeError("Not connected to exchange")

        # Convert order format
        exchange_order = self._convert_order(order)

        try:
            # Submit to exchange
            response = await self._send_order_to_exchange(exchange_order)

            # Map order IDs
            self.order_id_map[order.client_order_id] = response['order_id']

            # Generate event
            self._generate_order_submitted_event(order)

        except Exception as e:
            self._generate_order_rejected_event(order, str(e))
            raise

    async def cancel_order(self, order):
        """Cancel order"""
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
        """Modify order (if supported)"""
        # Most exchanges don't support direct modification, need to cancel and create new
        await self.cancel_order(order)
        await self.submit_order(order)

    # ==================== Abstract Methods (subclass implementation) ====================

    @abstractmethod
    async def _authenticate(self):
        """Authenticate"""
        pass

    @abstractmethod
    async def _get_account_info(self) -> dict:
        """Get account information"""
        pass

    @abstractmethod
    async def _start_data_streams(self):
        """Start data streams"""
        pass

    @abstractmethod
    async def _stop_data_streams(self):
        """Stop data streams"""
        pass

    @abstractmethod
    def _convert_order(self, order) -> dict:
        """Convert order format"""
        pass

    @abstractmethod
    async def _send_order_to_exchange(self, order: dict) -> dict:
        """Send order to exchange"""
        pass

    @abstractmethod
    async def _cancel_order_on_exchange(self, order_id: str):
        """Cancel order on exchange"""
        pass

    # ==================== Helper Methods ====================

    def _generate_order_submitted_event(self, order):
        """Generate order submitted event"""
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
        """Generate order rejected event"""
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
        """Generate order canceled event"""
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
    """WebSocket Execution Client"""

    def __init__(self, config):
        super().__init__(config)

        self.ws_url = config.get('ws_url')
        self.websocket = None
        self.ws_connected = False

    async def _start_data_streams(self):
        """Start WebSocket data stream"""
        import websockets

        self.websocket = await websockets.connect(self.ws_url)
        self.ws_connected = True

        # Start message processing task
        asyncio.create_task(self._handle_ws_messages())

    async def _stop_data_streams(self):
        """Stop WebSocket data stream"""
        if self.websocket:
            await self.websocket.close()
            self.ws_connected = False

    async def _handle_ws_messages(self):
        """Handle WebSocket messages"""
        while self.ws_connected:
            try:
                message = await self.websocket.recv()
                await self._process_ws_message(message)
            except Exception as e:
                self.log.error(f"WebSocket message error: {e}")
                break

    async def _process_ws_message(self, message: str):
        """Process WebSocket message"""
        # Parse message
        data = json.loads(message)

        # Handle based on message type
        if data['type'] == 'execution':
            await self._handle_execution_report(data)
        elif data['type'] == 'account':
            await self._handle_account_update(data)

    async def _handle_execution_report(self, data: dict):
        """Handle execution report"""
        order_id = self._find_internal_order_id(data['exchange_order_id'])

        if data['status'] == 'filled':
            await self._handle_order_filled(order_id, data)
        elif data['status'] == 'canceled':
            await self._handle_order_canceled(order_id, data)

    async def _handle_order_filled(self, order_id: str, data: dict):
        """Handle order fill"""
        # Create OrderFilled event
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
        """Find internal order ID by exchange order ID"""
        for internal_id, ex_id in self.order_id_map.items():
            if ex_id == exchange_order_id:
                return internal_id
        return None
```

## 18.3 Data Adapter Development

### 18.3.1 Custom Data Source

```python
from nautilus_trader.adapters.betfair.providers import BetfairInstrumentProvider
from nautilus_trader.model.data import QuoteTick, TradeTick, BarType, Bar
from nautilus_trader.model.identifiers import InstrumentId, Symbol, Venue
from nautilus_trader.core.datetime import dt_to_unix_nanos
import asyncio
import websockets

class CustomDataAdapter:
    """Custom Data Adapter Base Class"""

    def __init__(self, name: str, config: dict):
        self.name = name
        self.config = config

        # State
        self.is_connected = False
        self.subscribed_symbols = set()
        self.callbacks = {}

        # Data cache
        self.quote_cache = {}
        self.trade_cache = {}
        self.bar_cache = {}

    async def connect(self):
        """Connect to data source"""
        try:
            await self._connect_to_source()
            self.is_connected = True
            self.log.info(f"Connected to {self.name} data source")
        except Exception as e:
            self.log.error(f"Failed to connect: {e}")
            raise

    async def disconnect(self):
        """Disconnect"""
        if self.is_connected:
            await self._disconnect_from_source()
            self.is_connected = False

    async def subscribe_quotes(self, symbols: List[str], callback):
        """Subscribe to quote data"""
        for symbol in symbols:
            self.subscribed_symbols.add(symbol)
            if 'quote' not in self.callbacks:
                self.callbacks['quote'] = []
            self.callbacks['quote'].append((symbol, callback))

        await self._subscribe_to_quotes(symbols)

    async def subscribe_trades(self, symbols: List[str], callback):
        """Subscribe to trade data"""
        for symbol in symbols:
            self.subscribed_symbols.add(symbol)
            if 'trade' not in self.callbacks:
                self.callbacks['trade'] = []
            self.callbacks['trade'].append((symbol, callback))

        await self._subscribe_to_trades(symbols)

    async def subscribe_bars(self, bar_type: BarType, callback):
        """Subscribe to bar data"""
        symbol = str(bar_type.instrument_id)
        self.subscribed_symbols.add(symbol)
        if 'bar' not in self.callbacks:
            self.callbacks['bar'] = []
        self.callbacks['bar'].append((bar_type, callback))

        await self._subscribe_to_bars(bar_type)

    # ==================== Abstract Methods ====================

    @abstractmethod
    async def _connect_to_source(self):
        """Connect to data source"""
        pass

    @abstractmethod
    async def _disconnect_from_source(self):
        """Disconnect"""
        pass

    @abstractmethod
    async def _subscribe_to_quotes(self, symbols: List[str]):
        """Subscribe to quote data"""
        pass

    @abstractmethod
    async def _subscribe_to_trades(self, symbols: List[str]):
        """Subscribe to trade data"""
        pass

    @abstractmethod
    async def _subscribe_to_bars(self, bar_type: BarType):
        """Subscribe to bar data"""
        pass

class WebSocketDataAdapter(CustomDataAdapter):
    """WebSocket Data Adapter"""

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
        """Connect WebSocket"""
        self.websocket = await websockets.connect(self.ws_url)

        # Start message processing loop
        asyncio.create_task(self._message_loop())

    async def _disconnect_from_source(self):
        """Disconnect WebSocket"""
        if self.websocket:
            await self.websocket.close()

    async def _message_loop(self):
        """Message loop"""
        while True:
            try:
                message = await self.websocket.recv()
                data = json.loads(message)
                await self._process_message(data)
            except Exception as e:
                self.log.error(f"Message processing error: {e}")
                break

    async def _process_message(self, data: dict):
        """Process message"""
        message_type = data.get('type')

        if message_type in self.data_handlers:
            await self.data_handlers[message_type](data)

    async def _handle_quote_data(self, data: dict):
        """Handle quote data"""
        # Create QuoteTick
        quote_tick = self._create_quote_tick(data)

        # Call callbacks
        for symbol, callback in self.callbacks.get('quote', []):
            if symbol == data['symbol']:
                await callback(quote_tick)

    async def _handle_trade_data(self, data: dict):
        """Handle trade data"""
        # Create TradeTick
        trade_tick = self._create_trade_tick(data)

        # Call callbacks
        for symbol, callback in self.callbacks.get('trade', []):
            if symbol == data['symbol']:
                await callback(trade_tick)

    async def _handle_bar_data(self, data: dict):
        """Handle bar data"""
        # Create Bar
        bar = self._create_bar(data)

        # Call callbacks
        for bar_type, callback in self.callbacks.get('bar', []):
            if str(bar_type.instrument_id) == data['symbol']:
                await callback(bar)

class FileDataAdapter(CustomDataAdapter):
    """File Data Adapter (for backtesting)"""

    def __init__(self, name: str, config: dict):
        super().__init__(name, config)

        self.file_path = config.get('file_path')
        self.data_type = config.get('data_type', 'csv')  # csv, parquet, json
        self.speed_factor = config.get('speed_factor', 1.0)  # Playback speed

        self.data_iterator = None
        self.start_time = None

    async def _connect_to_source(self):
        """Connect to data source"""
        # Load data
        if self.data_type == 'csv':
            df = pd.read_csv(self.file_path)
        elif self.data_type == 'parquet':
            df = pd.read_parquet(self.file_path)
        else:
            raise ValueError(f"Unsupported data type: {self.data_type}")

        # Create iterator
        self.data_iterator = df.iterrows()
        self.start_time = pd.Timestamp.now()

    async def _disconnect_from_source(self):
        """Disconnect"""
        self.data_iterator = None

    async def play_data(self):
        """Play back data"""
        if not self.data_iterator:
            return

        previous_timestamp = None

        for _, row in self.data_iterator:
            # Parse timestamp
            timestamp = pd.Timestamp(row['timestamp'])

            # Calculate wait time (simulate real-time)
            if previous_timestamp:
                time_diff = (timestamp - previous_timestamp).total_seconds()
                await asyncio.sleep(time_diff / self.speed_factor)

            # Process data row
            if 'bid' in row and 'ask' in row:
                await self._process_quote_row(row)
            elif 'price' in row and 'size' in row:
                await self._process_trade_row(row)

            previous_timestamp = timestamp

    async def _process_quote_row(self, row):
        """Process quote row"""
        quote_tick = QuoteTick(
            instrument_id=InstrumentId.from_str(f"{row['symbol']}.{self.name}"),
            bid=Price.from_str(str(row['bid'])),
            ask=Price.from_str(str(row['ask'])),
            bid_size=Quantity.from_str(str(row.get('bid_size', 1))),
            ask_size=Quantity.from_str(str(row.get('ask_size', 1))),
            ts_event=dt_to_unix_nanos(row['timestamp']),
            ts_init=self.clock.timestamp_ns()
        )

        # Trigger callbacks
        for symbol, callback in self.callbacks.get('quote', []):
            if symbol == row['symbol']:
                await callback(quote_tick)
```

## 18.4 Plugin Development

### 18.4.1 Plugin System

```python
from abc import ABC, abstractmethod
import importlib
import inspect

class Plugin(ABC):
    """Plugin Base Class"""

    def __init__(self, name: str, version: str):
        self.name = name
        self.version = version
        self.enabled = True
        self.config = {}

    @abstractmethod
    async def initialize(self, config: dict):
        """Initialize plugin"""
        pass

    @abstractmethod
    async def start(self):
        """Start plugin"""
        pass

    @abstractmethod
    async def stop(self):
        """Stop plugin"""
        pass

    async def cleanup(self):
        """Clean up resources"""
        pass

class PluginManager:
    """Plugin Manager"""

    def __init__(self):
        self.plugins = {}
        self.plugin_configs = {}
        self.event_handlers = {}

    def load_plugin(self, plugin_path: str, config: dict = None):
        """Load plugin"""
        try:
            # Dynamically import plugin module
            spec = importlib.util.spec_from_file_location("plugin", plugin_path)
            module = importlib.util.module_from_spec(spec)
            spec.loader.exec_module(module)

            # Find plugin class
            plugin_class = None
            for name, obj in inspect.getmembers(module):
                if inspect.isclass(obj) and issubclass(obj, Plugin) and obj != Plugin:
                    plugin_class = obj
                    break

            if not plugin_class:
                raise ValueError(f"No plugin class found in {plugin_path}")

            # Create plugin instance
            plugin = plugin_class(
                name=config.get('name', plugin_class.__name__),
                version=config.get('version', '1.0.0')
            )

            # Initialize plugin
            asyncio.create_task(plugin.initialize(config or {}))

            self.plugins[plugin.name] = plugin
            self.plugin_configs[plugin.name] = config or {}

            self.log.info(f"Plugin loaded: {plugin.name}")

        except Exception as e:
            self.log.error(f"Failed to load plugin {plugin_path}: {e}")
            raise

    async def start_plugin(self, plugin_name: str):
        """Start plugin"""
        if plugin_name not in self.plugins:
            raise ValueError(f"Plugin {plugin_name} not found")

        plugin = self.plugins[plugin_name]
        if plugin.enabled:
            await plugin.start()
            self.log.info(f"Plugin started: {plugin_name}")

    async def stop_plugin(self, plugin_name: str):
        """Stop plugin"""
        if plugin_name not in self.plugins:
            raise ValueError(f"Plugin {plugin_name} not found")

        plugin = self.plugins[plugin_name]
        await plugin.stop()
        self.log.info(f"Plugin stopped: {plugin_name}")

    async def start_all_plugins(self):
        """Start all plugins"""
        for plugin_name in self.plugins:
            await self.start_plugin(plugin_name)

    async def stop_all_plugins(self):
        """Stop all plugins"""
        for plugin_name in self.plugins:
            await self.stop_plugin(plugin_name)

    def get_plugin(self, plugin_name: str) -> Plugin:
        """Get plugin"""
        return self.plugins.get(plugin_name)

    def register_event_handler(self, event_type: str, handler: Callable):
        """Register event handler"""
        if event_type not in self.event_handlers:
            self.event_handlers[event_type] = []
        self.event_handlers[event_type].append(handler)

    async def emit_event(self, event_type: str, data: dict):
        """Emit event"""
        if event_type in self.event_handlers:
            for handler in self.event_handlers[event_type]:
                try:
                    await handler(data)
                except Exception as e:
                    self.log.error(f"Event handler error: {e}")

# Example plugins
class RiskMonitorPlugin(Plugin):
    """Risk Monitoring Plugin"""

    def __init__(self, name: str, version: str):
        super().__init__(name, version)
        self.risk_limits = {}
        self.current_metrics = {}
        self.alerts = []

    async def initialize(self, config: dict):
        """Initialize"""
        self.config = config
        self.risk_limits = config.get('risk_limits', {})

        # Register event handlers
        self.plugin_manager.register_event_handler('order_filled', self.on_order_filled)
        self.plugin_manager.register_event_handler('position_updated', self.on_position_updated)

    async def start(self):
        """Start"""
        # Start monitoring task
        asyncio.create_task(self.monitor_loop())

    async def stop(self):
        """Stop"""
        pass

    async def monitor_loop(self):
        """Monitoring loop"""
        while True:
            await self.check_risk_metrics()
            await asyncio.sleep(1)  # Check every second

    async def check_risk_metrics(self):
        """Check risk metrics"""
        # Implement risk checking logic
        pass

    async def on_order_filled(self, data: dict):
        """Order filled event handler"""
        # Update risk metrics
        pass

    async def on_position_updated(self, data: dict):
        """Position update event handler"""
        # Update risk metrics
        pass

class PerformanceAnalyticsPlugin(Plugin):
    """Performance Analytics Plugin"""

    def __init__(self, name: str, version: str):
        super().__init__(name, version)
        self.trade_history = []
        self.performance_metrics = {}

    async def initialize(self, config: dict):
        """Initialize"""
        self.config = config

        # Register event handlers
        self.plugin_manager.register_event_handler('trade_executed', self.on_trade_executed)

    async def start(self):
        """Start"""
        pass

    async def stop(self):
        """Stop"""
        # Save analysis report
        await self.save_report()

    async def on_trade_executed(self, data: dict):
        """Trade execution event handler"""
        self.trade_history.append(data)

        # Update performance metrics in real-time
        self.update_performance_metrics()

    def update_performance_metrics(self):
        """Update performance metrics"""
        if not self.trade_history:
            return

        # Calculate various metrics
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
        """Save analysis report"""
        report_path = self.config.get('report_path', 'performance_report.json')

        report = {
            'timestamp': pd.Timestamp.now().isoformat(),
            'metrics': self.performance_metrics,
            'trade_history': self.trade_history
        }

        with open(report_path, 'w') as f:
            json.dump(report, f, indent=2)
```

## Summary

This chapter provides a detailed introduction to custom component development in NautilusTrader, covering:

1. **Custom Strategy Development**
   - Strategy lifecycle management
   - Event handling mechanisms
   - Advanced strategy features

2. **Custom Execution Clients**
   - Execution client architecture
   - WebSocket connections
   - Order management

3. **Data Adapter Development**
   - Data source adaptation
   - WebSocket data processing
   - File data playback

4. **Plugin System**
   - Plugin architecture design
   - Plugin manager
   - Example plugin implementations

Through this chapter, readers can extend NautilusTrader's functionality and develop custom components to meet specific requirements.

## References

- [NautilusTrader Extending Guide](https://docs.nautilustrader.io/extending.html)
- [Python Plugin Architecture](https://packaging.python.org/guides/creating-and-discovering-plugins/)
- [Design Patterns in Python](https://refactoring.guru/design-patterns/python)

## Runnable Code Entry (Source Build)

- Entry script: docs/tutorials_cn/code/chapter18/run_custom_components.py
- Key examples: docs/tutorials_cn/code/chapter18/custom_data_source.py, docs/tutorials_cn/code/chapter18/custom_execution.py, docs/tutorials_cn/code/chapter18/custom_indicators.py

Dependency strategy: Prioritizes Nautilus Data/DataType and built-in indicator integration, with deterministic pure-Python fallback.

Verified command:

```bash
uv run --python 3.13 python docs/tutorials_cn/code/chapter18/run_custom_components.py
```
