---
title: "Chapter 16: Algorithmic Trading Strategies"
---

# Chapter 16: Algorithmic Trading Strategies

## Learning Objectives

- Master the principles and implementation of common algorithmic trading strategies
- Learn to build complex trading algorithms using NautilusTrader
- Understand strategy design under different market microstructures
- Implement high-performance, low-latency trading strategies

## 16.1 Market Making Strategies

### 16.1.1 Basic Market Making Strategy

```python
from nautilus_trader.core.message import Event
from nautilus_trader.model.data import QuoteTick
from nautilus_trader.model.enums import OrderSide, OrderType
from nautilus_trader.model.objects import Price, Quantity
from nautilus_trader.strategy import Strategy
from decimal import Decimal
import numpy as np

class MarketMakingStrategy(Strategy):
    """Basic Market Making Strategy"""

    def __init__(self, config):
        super().__init__()
        self.symbol = config['symbol']
        self.spread_bps = config['spread_bps']  # Spread in basis points
        self.order_size = config['order_size']  # Order size
        self.max_position = config['max_position']  # Maximum position
        self.inventory_target = config.get('inventory_target', 0)  # Target inventory
        self.rebalance_threshold = config.get('rebalance_threshold', 0.1)  # Rebalance threshold

        # State variables
        self.current_position = 0
        self.mid_price = None
        self.current_orders = {}  # Active orders
        self.last_quote_time = None

    def on_start(self):
        """Called when strategy starts"""
        self.subscribe_quote_ticks(self.symbol)
        self.log.info(f"Market making strategy started for {self.symbol}")

    def on_quote_tick(self, tick: QuoteTick):
        """Process quote data"""
        # Update mid price
        self.mid_price = (tick.bid + tick.ask) / 2
        self.last_quote_time = tick.ts_event

        # Get current position
        self._update_position()

        # Cancel existing orders
        await self._cancel_all_orders()

        # Calculate new orders
        bid_price, ask_price, bid_size, ask_size = self._calculate_quotes()

        # Place new orders
        if bid_price and bid_size > 0:
            await self._place_order(OrderSide.BUY, bid_price, bid_size)
        if ask_price and ask_size > 0:
            await self._place_order(OrderSide.SELL, ask_price, ask_size)

    def _update_position(self):
        """Update current position"""
        portfolio = self.portfolio
        position = portfolio.position(self.symbol)
        if position:
            self.current_position = position.quantity

    def _calculate_quotes(self):
        """Calculate bid and ask quotes"""
        if not self.mid_price:
            return None, None, None, None

        # Base spread
        spread = self.mid_price * self.spread_bps / 10000

        # Adjust spread and quotes based on inventory
        inventory_skew = self._calculate_inventory_skew()

        # Calculate bid and ask prices
        bid_price = self.mid_price - spread * (1 + inventory_skew)
        ask_price = self.mid_price + spread * (1 - inventory_skew)

        # Calculate order sizes
        bid_size, ask_size = self._calculate_order_sizes(inventory_skew)

        return bid_price, ask_price, bid_size, ask_size

    def _calculate_inventory_skew(self) -> float:
        """Calculate inventory skew"""
        if self.max_position == 0:
            return 0

        # Current position relative to target position ratio
        inventory_ratio = (self.current_position - self.inventory_target) / self.max_position

        # Limit to [-1, 1] range
        inventory_skew = max(-1, min(1, inventory_ratio))

        return inventory_skew

    def _calculate_order_sizes(self, inventory_skew: float):
        """Calculate order sizes"""
        # Base order size
        base_size = self.order_size

        # Adjust size based on inventory skew
        if inventory_skew > 0:  # Too long, reduce bid size, increase ask size
            bid_size = base_size * (1 - abs(inventory_skew))
            ask_size = base_size * (1 + abs(inventory_skew))
        elif inventory_skew < 0:  # Too short, increase bid size, reduce ask size
            bid_size = base_size * (1 + abs(inventory_skew))
            ask_size = base_size * (1 - abs(inventory_skew))
        else:  # Balanced inventory
            bid_size = ask_size = base_size

        # Ensure not exceeding maximum position limit
        if self.current_position + bid_size > self.max_position:
            bid_size = max(0, self.max_position - self.current_position)
        if self.current_position - ask_size < -self.max_position:
            ask_size = max(0, self.current_position + self.max_position)

        return bid_size, ask_size

    async def _place_order(self, side: OrderSide, price: Price, size: Quantity):
        """Place order"""
        order = self.order_factory.market(
            symbol=self.symbol,
            side=side,
            quantity=size
        )

        # Change to limit order
        order = self.order_factory.limit(
            symbol=self.symbol,
            side=side,
            price=price,
            quantity=size
        )

        order_id = await self.submit_order(order)
        self.current_orders[order_id] = {
            'side': side,
            'price': price,
            'size': size
        }

    async def _cancel_all_orders(self):
        """Cancel all orders"""
        for order_id in list(self.current_orders.keys()):
            await self.cancel_order(order_id)
            del self.current_orders[order_id]

    def on_order_filled(self, event):
        """Order fill notification"""
        self.log.info(f"Order filled: {event}")
        # Position update will be done on next tick

    def on_order_rejected(self, event):
        """Order rejection notification"""
        self.log.warning(f"Order rejected: {event}")
        if event.order_id in self.current_orders:
            del self.current_orders[event.order_id]

    def on_order_canceled(self, event):
        """Order cancellation notification"""
        if event.order_id in self.current_orders:
            del self.current_orders[event.order_id]

    def on_stop(self):
        """Called when strategy stops"""
        # Cancel all orders
        asyncio.create_task(self._cancel_all_orders())
        self.log.info("Market making strategy stopped")
```

### 16.1.2 High-Frequency Market Making Strategy

```python
import time
from numba import jit
from collections import deque

class HFTMarketMakingStrategy(Strategy):
    """High-Frequency Market Making Strategy"""

    def __init__(self, config):
        super().__init__()
        self.symbol = config['symbol']
        self.order_size = config['order_size']
        self.max_orders_per_side = config.get('max_orders_per_side', 5)
        self.min_spread_bps = config.get('min_spread_bps', 1)
        self.max_spread_bps = config.get('max_spread_bps', 10)
        self.volatility_window = config.get('volatility_window', 100)
        self.microprice_alpha = config.get('microprice_alpha', 0.2)

        # High-performance data structures
        self.price_history = deque(maxlen=self.volatility_window)
        self.order_book_levels = 10
        self.current_orders = {}
        self.last_microprice = None
        self.volatility = 0

        # Performance optimization
        self._setup_jit_functions()

    def _setup_jit_functions(self):
        """Setup JIT accelerated functions"""
        # Use numba to accelerate key computations
        self.calculate_microprice = jit(self._calculate_microprice_impl)
        self.calculate_spread = jit(self._calculate_spread_impl)

    @staticmethod
    @jit(nopython=True)
    def _calculate_microprice_impl(bids, asks, alpha):
        """Calculate microprice (JIT accelerated)"""
        if len(bids) == 0 or len(asks) == 0:
            return 0.0

        best_bid = bids[0, 0]  # Price
        best_ask = asks[0, 0]
        bid_size = bids[0, 1]  # Size
        ask_size = asks[0, 1]

        # Microprice considers imbalance
        total_size = bid_size + ask_size
        if total_size == 0:
            return (best_bid + best_ask) / 2

        microprice = (best_bid * ask_size + best_ask * bid_size) / total_size

        # Use exponentially weighted moving average
        return alpha * microprice + (1 - alpha) * (best_bid + best_ask) / 2

    @staticmethod
    @jit(nopython=True)
    def _calculate_spread_impl(microprice, volatility, min_spread, max_spread):
        """Calculate dynamic spread (JIT accelerated)"""
        # Volatility-based spread
        spread = microprice * volatility * 2

        # Limit between minimum and maximum spread
        return max(min_spread, min(max_spread, spread))

    def on_quote_tick(self, tick: QuoteTick):
        """Process quote data"""
        # Update price history
        mid_price = float((tick.bid + tick.ask) / 2)
        self.price_history.append(mid_price)

        # Calculate volatility
        if len(self.price_history) > 10:
            self._update_volatility()

        # Get order book snapshot
        order_book = self.get_order_book(self.symbol)
        if not order_book:
            return

        # Extract bid and ask data
        bids = np.array([[float(level.price), float(level.size)]
                        for level in order_book.bids[:self.order_book_levels]])
        asks = np.array([[float(level.price), float(level.size)]
                        for level in order_book.asks[:self.order_book_levels]])

        # Calculate microprice
        microprice = self.calculate_microprice(
            bids, asks, self.microprice_alpha
        )

        # Calculate dynamic spread
        min_spread = microprice * self.min_spread_bps / 10000
        max_spread = microprice * self.max_spread_bps / 10000
        spread = self.calculate_spread(
            microprice, self.volatility, min_spread, max_spread
        )

        # Update orders
        self._update_order_book(microprice, spread, order_book)

    def _update_volatility(self):
        """Update volatility estimate"""
        prices = np.array(self.price_history)
        returns = np.diff(np.log(prices))
        self.volatility = np.std(returns) * np.sqrt(252)  # Annualized volatility

    def _update_order_book(self, microprice: float, spread: float, order_book):
        """Update order book"""
        # Cancel existing orders
        asyncio.create_task(self._cancel_all_orders())

        # Get current position
        current_position = self._get_current_position()

        # Calculate multi-level quotes
        levels = self._calculate_quote_levels(
            microprice, spread, current_position
        )

        # Place new orders
        for level in levels:
            asyncio.create_task(self._place_quote_level(level))

    def _calculate_quote_levels(self, mid_price: float, spread: float,
                               position: float) -> List[dict]:
        """Calculate multi-level quote levels"""
        levels = []
        inventory_skew = position / self.max_position if self.max_position else 0

        for i in range(self.max_orders_per_side):
            # Incremental spread
            level_spread = spread * (1 + i * 0.2)

            # Bid price
            bid_price = mid_price - level_spread * (1 + inventory_skew * 0.5)
            bid_size = self.order_size * (1 - i * 0.1)  # Decrecremental size

            # Ask price
            ask_price = mid_price + level_spread * (1 - inventory_skew * 0.5)
            ask_size = self.order_size * (1 - i * 0.1)  # Decrecremental size

            # Adjust based on position
            if position > 0:  # Long position, reduce bids
                bid_size *= (1 - abs(inventory_skew))
            elif position < 0:  # Short position, reduce asks
                ask_size *= (1 - abs(inventory_skew))

            levels.append({
                'side': OrderSide.BUY,
                'price': Price.from_str(f"{bid_price:.{self.price_precision}f}"),
                'size': Quantity.from_int(int(bid_size))
            })
            levels.append({
                'side': OrderSide.SELL,
                'price': Price.from_str(f"{ask_price:.{self.price_precision}f}"),
                'size': Quantity.from_int(int(ask_size))
            })

        return levels

    async def _place_quote_level(self, level: dict):
        """Place quote level"""
        order = self.order_factory.limit(
            symbol=self.symbol,
            side=level['side'],
            price=level['price'],
            quantity=level['size']
        )

        order_id = await self.submit_order(order)
        self.current_orders[order_id] = {
            'level': level,
            'timestamp': time.time()
        }

    async def _cancel_all_orders(self):
        """Cancel all orders"""
        for order_id in list(self.current_orders.keys()):
            await self.cancel_order(order_id)
            del self.current_orders[order_id]
```

## 16.2 Arbitrage Strategies

### 16.2.1 Statistical Arbitrage

```python
from sklearn.linear_model import LinearRegression
from scipy import stats
import pandas as pd

class StatisticalArbitrageStrategy(Strategy):
    """Statistical Arbitrage Strategy"""

    def __init__(self, config):
        super().__init__()
        self.pair1 = config['pair1']  # Trading pair 1
        self.pair2 = config['pair2']  # Trading pair 2
        self.lookback_period = config['lookback_period']  # Lookback period
        self.z_entry_threshold = config.get('z_entry', 2.0)  # Entry Z-score threshold
        self.z_exit_threshold = config.get('z_exit', 0.5)    # Exit Z-score threshold
        self.position_size = config['position_size']

        # State variables
        self.price_history = {self.pair1: [], self.pair2: []}
        self.spread_history = []
        self.hedge_ratio = None
        self.mean_spread = None
        self.std_spread = None
        self.current_position = None  # long, short, or None

    def on_start(self):
        """Strategy start"""
        self.subscribe_quote_ticks(self.pair1)
        self.subscribe_quote_ticks(self.pair2)
        self.log.info(f"Statistical arbitrage started for {self.pair1}-{self.pair2}")

    def on_quote_tick(self, tick: QuoteTick):
        """Process quote data"""
        symbol = tick.symbol
        mid_price = float((tick.bid + tick.ask) / 2)

        # Update price history
        self.price_history[symbol].append(mid_price)

        # Maintain fixed length
        if len(self.price_history[symbol]) > self.lookback_period:
            self.price_history[symbol].pop(0)

        # If enough data, update statistics
        if self._has_enough_data():
            self._update_statistics()

            # Check trading signals
            signal = self._check_signal()
            if signal:
                self._execute_signal(signal)

    def _has_enough_data(self) -> bool:
        """Check if there is enough data"""
        return (len(self.price_history[self.pair1]) >= self.lookback_period and
                len(self.price_history[self.pair2]) >= self.lookback_period)

    def _update_statistics(self):
        """Update statistical indicators"""
        # Prepare data
        prices1 = np.array(self.price_history[self.pair1])
        prices2 = np.array(self.price_history[self.pair2])

        # Calculate log prices
        log_prices1 = np.log(prices1)
        log_prices2 = np.log(prices2)

        # Linear regression to calculate hedge ratio
        X = log_prices2.reshape(-1, 1)
        y = log_prices1
        model = LinearRegression()
        model.fit(X, y)
        self.hedge_ratio = model.coef_[0]

        # Calculate spread
        spread = log_prices1 - self.hedge_ratio * log_prices2
        self.spread_history = spread.tolist()
        self.mean_spread = np.mean(spread)
        self.std_spread = np.std(spread)

    def _check_signal(self) -> Optional[dict]:
        """Check trading signals"""
        if not self.hedge_ratio:
            return None

        # Calculate current spread
        current_price1 = self.price_history[self.pair1][-1]
        current_price2 = self.price_history[self.pair2][-1]
        current_spread = np.log(current_price1) - self.hedge_ratio * np.log(current_price2)

        # Calculate Z-score
        if self.std_spread > 0:
            z_score = (current_spread - self.mean_spread) / self.std_spread
        else:
            z_score = 0

        # Generate signals
        signal = None

        if self.current_position is None:
            # No position, check entry opportunities
            if z_score > self.z_entry_threshold:
                # Spread too high, short spread (short pair1, long pair2)
                signal = {
                    'action': 'enter_short_spread',
                    'z_score': z_score,
                    'spread': current_spread
                }
                self.current_position = 'short'
            elif z_score < -self.z_entry_threshold:
                # Spread too low, long spread (long pair1, short pair2)
                signal = {
                    'action': 'enter_long_spread',
                    'z_score': z_score,
                    'spread': current_spread
                }
                self.current_position = 'long'

        elif self.current_position == 'short':
            # Short spread position, check exit
            if z_score < self.z_exit_threshold:
                signal = {
                    'action': 'exit_short_spread',
                    'z_score': z_score,
                    'spread': current_spread
                }
                self.current_position = None

        elif self.current_position == 'long':
            # Long spread position, check exit
            if z_score > -self.z_exit_threshold:
                signal = {
                    'action': 'exit_long_spread',
                    'z_score': z_score,
                    'spread': current_spread
                }
                self.current_position = None

        return signal

    def _execute_signal(self, signal: dict):
        """Execute trading signal"""
        action = signal['action']
        current_price1 = self.price_history[self.pair1][-1]
        current_price2 = self.price_history[self.pair2][-1]

        if action == 'enter_long_spread':
            # Long spread: buy pair1, sell pair2*hedge_ratio
            size1 = self.position_size
            size2 = int(self.position_size * self.hedge_ratio * current_price1 / current_price2)

            asyncio.create_task(self._place_order(self.pair1, OrderSide.BUY, size1))
            asyncio.create_task(self._place_order(self.pair2, OrderSide.SELL, size2))

        elif action == 'enter_short_spread':
            # Short spread: sell pair1, buy pair2*hedge_ratio
            size1 = self.position_size
            size2 = int(self.position_size * self.hedge_ratio * current_price1 / current_price2)

            asyncio.create_task(self._place_order(self.pair1, OrderSide.SELL, size1))
            asyncio.create_task(self._place_order(self.pair2, OrderSide.BUY, size2))

        elif action in ['exit_long_spread', 'exit_short_spread']:
            # Close positions: close all related positions
            self._close_positions()

        self.log.info(f"Executing signal: {signal}")

    async def _place_order(self, symbol: str, side: OrderSide, size: int):
        """Place limit order"""
        order = self.order_factory.market(
            symbol=symbol,
            side=side,
            quantity=Quantity.from_int(size)
        )
        await self.submit_order(order)

    def _close_positions(self):
        """Close all positions"""
        # Get current positions
        portfolio = self.portfolio

        # Close position in pair1
        position1 = portfolio.position(self.pair1)
        if position1 and position1.quantity != 0:
            side = OrderSide.SELL if position1.quantity > 0 else OrderSide.BUY
            size = abs(int(position1.quantity))
            asyncio.create_task(self._place_order(self.pair1, side, size))

        # Close position in pair2
        position2 = portfolio.position(self.pair2)
        if position2 and position2.quantity != 0:
            side = OrderSide.SELL if position2.quantity > 0 else OrderSide.BUY
            size = abs(int(position2.quantity))
            asyncio.create_task(self._place_order(self.pair2, side, size))

    def get_performance_metrics(self) -> dict:
        """Get strategy performance metrics"""
        if not self.spread_history:
            return {}

        # Calculate statistical indicators
        spreads = np.array(self.spread_history)
        half_life = self._calculate_half_life(spreads)
        hurst_exponent = self._calculate_hurst_exponent(spreads)

        return {
            'hedge_ratio': self.hedge_ratio,
            'mean_spread': self.mean_spread,
            'std_spread': self.std_spread,
            'half_life': half_life,
            'hurst_exponent': hurst_exponent,
            'current_position': self.current_position
        }

    def _calculate_half_life(self, spreads: np.ndarray) -> float:
        """Calculate half-life of spread"""
        # Use AR(1) model to calculate half-life
        if len(spreads) < 2:
            return 0

        spreads_lag = spreads[:-1]
        spreads_lead = spreads[1:]

        # Linear regression
        model = LinearRegression()
        model.fit(spreads_lag.reshape(-1, 1), spreads_lead)
        beta = model.coef_[0]

        # Half-life = ln(0.5) / ln(beta)
        if abs(beta) < 1:
            half_life = np.log(0.5) / np.log(abs(beta))
            return max(0, half_life)
        return 0

    def _calculate_hurst_exponent(self, spreads: np.ndarray) -> float:
        """Calculate Hurst exponent"""
        if len(spreads) < 100:
            return 0.5  # Default value

        # Simplified Hurst exponent calculation
        lags = range(2, min(20, len(spreads) // 2))
        tau = [np.sqrt(np.std(np.subtract(spreads[lag:], spreads[:-lag]))) for lag in lags]

        # Linear regression fit
        poly = np.polyfit(np.log(lags), np.log(tau), 1)
        hurst = poly[0] * 2.0

        return hurst
```

## 16.3 Momentum Strategies

### 16.3.1 Time Series Momentum

```python
from ta.momentum import RSIIndicator, MACD
from ta.trend import SMAIndicator, EMAIndicator

class TimeSeriesMomentumStrategy(Strategy):
    """Time Series Momentum Strategy"""

    def __init__(self, config):
        super().__init__()
        self.symbol = config['symbol']
        self.lookback_period = config['lookback_period']  # Momentum lookback period
        self.fast_ma = config.get('fast_ma', 10)         # Fast moving average
        self.slow_ma = config.get('slow_ma', 50)         # Slow moving average
        self.rsi_period = config.get('rsi_period', 14)   # RSI period
        self.rsi_overbought = config.get('rsi_overbought', 70)
        self.rsi_oversold = config.get('rsi_oversold', 30)
        self.position_size = config['position_size']
        self.max_position = config.get('max_position', 3)

        # Technical indicators
        self.price_history = []
        self.indicators = {}

    def on_start(self):
        """Strategy start"""
        self.subscribe_quote_ticks(self.symbol)
        self.log.info(f"Time series momentum strategy started for {self.symbol}")

    def on_quote_tick(self, tick: QuoteTick):
        """Process quote data"""
        # Update price history
        close_price = float((tick.bid + tick.ask) / 2)
        self.price_history.append(close_price)

        # Maintain fixed length
        max_history = max(self.lookback_period, self.slow_ma * 2)
        if len(self.price_history) > max_history:
            self.price_history.pop(0)

        # Calculate technical indicators
        if len(self.price_history) >= self.slow_ma:
            self._calculate_indicators()

            # Generate trading signals
            signal = self._generate_signal()
            if signal:
                self._execute_signal(signal)

    def _calculate_indicators(self):
        """Calculate technical indicators"""
        prices = np.array(self.price_history)

        # Moving averages
        self.indicators['sma_fast'] = SMAIndicator(
            close=pd.Series(prices), window=self.fast_ma
        ).sma_indicator().iloc[-1]

        self.indicators['sma_slow'] = SMAIndicator(
            close=pd.Series(prices), window=self.slow_ma
        ).sma_indicator().iloc[-1]

        # Exponential moving averages
        self.indicators['ema_fast'] = EMAIndicator(
            close=pd.Series(prices), window=self.fast_ma
        ).ema_indicator().iloc[-1]

        self.indicators['ema_slow'] = EMAIndicator(
            close=pd.Series(prices), window=self.slow_ma
        ).ema_indicator().iloc[-1]

        # RSI
        self.indicators['rsi'] = RSIIndicator(
            close=pd.Series(prices), window=self.rsi_period
        ).rsi().iloc[-1]

        # MACD
        macd_indicator = MACD(
            close=pd.Series(prices),
            window_fast=12,
            window_slow=26,
            window_sign=9
        )
        self.indicators['macd'] = macd_indicator.macd().iloc[-1]
        self.indicators['macd_signal'] = macd_indicator.macd_signal().iloc[-1]
        self.indicators['macd_histogram'] = macd_indicator.macd_diff().iloc[-1]

        # Momentum indicator
        if len(self.price_history) >= self.lookback_period:
            momentum = (prices[-1] - prices[-self.lookback_period]) / prices[-self.lookback_period]
            self.indicators['momentum'] = momentum
            self.indicators['momentum_pct'] = momentum * 100

    def _generate_signal(self) -> Optional[dict]:
        """Generate trading signals"""
        # Get current position
        current_position = self._get_current_position()

        # Multiple condition confirmation
        long_conditions = []
        short_conditions = []

        # 1. Moving average crossover
        if (self.indicators['sma_fast'] > self.indicators['sma_slow'] and
            self.indicators['ema_fast'] > self.indicators['ema_slow']):
            long_conditions.append('ma_bullish')
        else:
            short_conditions.append('ma_bearish')

        # 2. RSI
        if self.indicators['rsi'] < self.rsi_oversold:
            long_conditions.append('rsi_oversold')
        elif self.indicators['rsi'] > self.rsi_overbought:
            short_conditions.append('rsi_overbought')

        # 3. MACD
        if (self.indicators['macd'] > self.indicators['macd_signal'] and
            self.indicators['macd_histogram'] > 0):
            long_conditions.append('macd_bullish')
        else:
            short_conditions.append('macd_bearish')

        # 4. Momentum
        if 'momentum' in self.indicators:
            if self.indicators['momentum'] > 0:
                long_conditions.append('momentum_positive')
            else:
                short_conditions.append('momentum_negative')

        # Generate signals
        signal = None

        if len(long_conditions) >= 3 and current_position <= 0:  # At least 3 bullish conditions
            signal = {
                'action': 'buy',
                'conditions': long_conditions,
                'strength': len(long_conditions)
            }

        elif len(short_conditions) >= 3 and current_position >= 0:  # At least 3 bearish conditions
            signal = {
                'action': 'sell',
                'conditions': short_conditions,
                'strength': len(short_conditions)
            }

        # Stop loss conditions
        elif current_position > 0 and self.indicators['rsi'] > self.rsi_overbought:
            signal = {
                'action': 'close_long',
                'reason': 'rsi_overbought'
            }

        elif current_position < 0 and self.indicators['rsi'] < self.rsi_oversold:
            signal = {
                'action': 'close_short',
                'reason': 'rsi_oversold'
            }

        return signal

    def _execute_signal(self, signal: dict):
        """Execute trading signal"""
        action = signal['action']
        current_position = self._get_current_position()

        if action == 'buy' and current_position < self.max_position:
            # Calculate position size
            position_to_add = min(
                self.position_size,
                self.max_position - current_position
            )

            # Adjust position based on signal strength
            if 'strength' in signal:
                position_to_add = int(position_to_add * signal['strength'] / 4)

            if position_to_add > 0:
                asyncio.create_task(
                    self._place_order(OrderSide.BUY, position_to_add)
                )

        elif action == 'sell' and current_position > -self.max_position:
            # Calculate position size
            position_to_add = min(
                self.position_size,
                self.max_position + current_position
            )

            # Adjust position based on signal strength
            if 'strength' in signal:
                position_to_add = int(position_to_add * signal['strength'] / 4)

            if position_to_add > 0:
                asyncio.create_task(
                    self._place_order(OrderSide.SELL, position_to_add)
                )

        elif action in ['close_long', 'close_short']:
            # Close position
            if current_position != 0:
                asyncio.create_task(
                    self._place_order(
                        OrderSide.SELL if current_position > 0 else OrderSide.BUY,
                        abs(current_position)
                    )
                )

        self.log.info(f"Executing signal: {signal}")

    async def _place_order(self, side: OrderSide, size: int):
        """Place order"""
        order = self.order_factory.market(
            symbol=self.symbol,
            side=side,
            quantity=Quantity.from_int(size)
        )
        await self.submit_order(order)

    def _get_current_position(self) -> int:
        """Get current position"""
        position = self.portfolio.position(self.symbol)
        return int(position.quantity) if position else 0

    def get_strategy_state(self) -> dict:
        """Get strategy state"""
        return {
            'indicators': self.indicators,
            'position': self._get_current_position(),
            'price_history_length': len(self.price_history)
        }
```

## 16.4 Machine Learning Strategies

### 16.4.1 Reinforcement Learning Trading

```python
import gym
from gym import spaces
import torch
import torch.nn as nn
from collections import deque
import random

class TradingEnvironment(gym.Env):
    """Trading Environment (OpenAI Gym Interface)"""

    def __init__(self, data: pd.DataFrame, initial_balance: float = 100000):
        super().__init__()

        self.data = data
        self.initial_balance = initial_balance
        self.current_step = 0
        self.balance = initial_balance
        self.position = 0
        self.position_price = 0

        # Action space: 0=hold, 1=buy, 2=sell
        self.action_space = spaces.Discrete(3)

        # State space: price, technical indicators, position info
        self.observation_space = spaces.Box(
            low=-np.inf,
            high=np.inf,
            shape=(10,),  # Can be adjusted as needed
            dtype=np.float32
        )

        # History
        self.trade_history = []
        self.portfolio_history = []

    def reset(self):
        """Reset environment"""
        self.current_step = 0
        self.balance = self.initial_balance
        self.position = 0
        self.position_price = 0
        self.trade_history = []
        self.portfolio_history = []
        return self._get_observation()

    def step(self, action):
        """Execute one step"""
        # Get current price
        current_price = self.data.iloc[self.current_step]['close']

        # Execute action
        reward = 0
        done = False

        if action == 1:  # Buy
            if self.position == 0:  # Can only buy when no position
                self.position = self.balance / current_price
                self.position_price = current_price
                self.balance = 0

        elif action == 2:  # Sell
            if self.position > 0:  # Can only sell when have position
                self.balance = self.position * current_price
                profit = (current_price - self.position_price) * self.position
                reward = profit / self.initial_balance  # Normalized reward
                self.position = 0
                self.position_price = 0

        # Calculate total portfolio value
        total_value = self.balance + self.position * current_price
        self.portfolio_history.append(total_value)

        # Record trade
        self.trade_history.append({
            'step': self.current_step,
            'action': action,
            'price': current_price,
            'position': self.position,
            'balance': self.balance,
            'total_value': total_value
        })

        # Move to next step
        self.current_step += 1

        # Check if done
        if self.current_step >= len(self.data) - 1:
            done = True
            # Force close position at the end
            if self.position > 0:
                self.balance = self.position * current_price
                self.position = 0

        # Get new state
        observation = self._get_observation()

        # Calculate additional reward (based on portfolio change)
        if len(self.portfolio_history) > 1:
            portfolio_return = (total_value - self.portfolio_history[-2]) / self.portfolio_history[-2]
            reward += portfolio_return * 0.1  # Smaller weight

        return observation, reward, done, {}

    def _get_observation(self):
        """Get current state"""
        if self.current_step >= len(self.data):
            return np.zeros(10)

        current_data = self.data.iloc[self.current_step]

        # Build state vector
        observation = np.array([
            current_data['close'] / self.data['close'].iloc[0] - 1,  # Normalized price
            current_data['volume'] / self.data['volume'].mean() - 1,  # Normalized volume
            current_data.get('rsi', 50) / 100,  # RSI
            current_data.get('macd', 0),  # MACD
            current_data.get('bb_upper', 0) / current_data['close'] - 1,  # Bollinger upper band
            current_data.get('bb_lower', 0) / current_data['close'] - 1,  # Bollinger lower band
            self.position / self.initial_balance if self.position > 0 else 0,  # Position ratio
            (current_data['close'] - self.position_price) / self.position_price if self.position_price > 0 else 0,  # Position return
            self.balance / self.initial_balance,  # Cash ratio
            len(self.trade_history) / len(self.data)  # Time progress
        ])

        return observation.astype(np.float32)

class DQNNetwork(nn.Module):
    """Deep Q Network"""

    def __init__(self, state_size: int, action_size: int):
        super().__init__()
        self.network = nn.Sequential(
            nn.Linear(state_size, 128),
            nn.ReLU(),
            nn.Dropout(0.2),
            nn.Linear(128, 64),
            nn.ReLU(),
            nn.Dropout(0.2),
            nn.Linear(64, action_size)
        )

    def forward(self, x):
        return self.network(x)

class RLTradingStrategy(Strategy):
    """Reinforcement Learning Trading Strategy"""

    def __init__(self, config):
        super().__init__()
        self.symbol = config['symbol']
        self.model_path = config.get('model_path', 'models/dqn_model.pth')
        self.lookback_window = config.get('lookback_window', 100)
        self.position_size = config['position_size']
        self.max_position = config.get('max_position', 10)

        # DQN parameters
        self.state_size = 10
        self.action_size = 3  # hold, buy, sell
        self.epsilon = config.get('epsilon', 0.1)  # Exploration rate
        self.memory = deque(maxlen=10000)

        # Load trained model
        self.device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
        self.model = DQNNetwork(self.state_size, self.action_size).to(self.device)
        self._load_model()

        # Data cache
        self.price_history = []
        self.feature_cache = {}

    def _load_model(self):
        """Load trained model"""
        try:
            self.model.load_state_dict(torch.load(self.model_path, map_location=self.device))
            self.model.eval()
            self.log.info(f"Loaded RL model from {self.model_path}")
        except FileNotFoundError:
            self.log.warning(f"Model file not found: {self.model_path}, using random policy")
        except Exception as e:
            self.log.error(f"Failed to load model: {e}")

    def on_start(self):
        """Strategy start"""
        self.subscribe_quote_ticks(self.symbol)
        self.log.info(f"RL trading strategy started for {self.symbol}")

    def on_quote_tick(self, tick: QuoteTick):
        """Process quote data"""
        # Update price history
        close_price = float((tick.bid + tick.ask) / 2)
        self.price_history.append(close_price)

        # Maintain fixed length
        if len(self.price_history) > self.lookback_window * 2:
            self.price_history.pop(0)

        # Calculate technical indicators
        if len(self.price_history) >= self.lookback_window:
            self._update_features()

            # Get current state
            state = self._get_current_state()

            # Select action
            action = self._select_action(state)

            # Execute action
            self._execute_action(action, close_price)

    def _update_features(self):
        """Update technical indicators"""
        prices = np.array(self.price_history)

        # Calculate various technical indicators
        df = pd.DataFrame({'close': prices})

        # RSI
        delta = df['close'].diff()
        gain = (delta.where(delta > 0, 0)).rolling(window=14).mean()
        loss = (-delta.where(delta < 0, 0)).rolling(window=14).mean()
        rs = gain / loss
        df['rsi'] = 100 - (100 / (1 + rs))

        # MACD
        exp1 = df['close'].ewm(span=12).mean()
        exp2 = df['close'].ewm(span=26).mean()
        df['macd'] = exp1 - exp2

        # Bollinger Bands
        df['sma'] = df['close'].rolling(window=20).mean()
        df['std'] = df['close'].rolling(window=20).std()
        df['bb_upper'] = df['sma'] + 2 * df['std']
        df['bb_lower'] = df['sma'] - 2 * df['std']

        # Volume (if available)
        df['volume'] = np.random.randn(len(df))  # Simulated volume

        # Cache latest feature values
        self.feature_cache = df.iloc[-1].to_dict()

    def _get_current_state(self) -> np.ndarray:
        """Get current state"""
        if not self.feature_cache:
            return np.zeros(self.state_size)

        # Build state vector
        current_price = self.price_history[-1]
        base_price = self.price_history[0]

        state = np.array([
            (current_price / base_price) - 1,  # Normalized price
            (self.feature_cache.get('volume', 0) / 100) - 1,  # Normalized volume
            self.feature_cache.get('rsi', 50) / 100,  # RSI
            self.feature_cache.get('macd', 0),  # MACD
            (self.feature_cache.get('bb_upper', current_price) / current_price) - 1,
            (self.feature_cache.get('bb_lower', current_price) / current_price) - 1,
            self._get_position_ratio(),
            self._get_position_return(),
            self._get_cash_ratio(),
            len(self.price_history) / (self.lookback_window * 2)
        ])

        return state.astype(np.float32)

    def _select_action(self, state: np.ndarray) -> int:
        """Select action"""
        # Epsilon-greedy strategy
        if random.random() < self.epsilon:
            return random.randrange(self.action_size)

        # Use model prediction
        with torch.no_grad():
            state_tensor = torch.FloatTensor(state).unsqueeze(0).to(self.device)
            q_values = self.model(state_tensor)
            return q_values.argmax().item()

    def _execute_action(self, action: int, price: float):
        """Execute action"""
        current_position = self._get_current_position()

        if action == 1 and current_position < self.max_position:  # Buy
            # Calculate buy quantity
            cash_available = self._get_available_cash()
            quantity = min(self.position_size, int(cash_available / price))

            if quantity > 0:
                asyncio.create_task(
                    self._place_order(OrderSide.BUY, quantity)
                )

        elif action == 2 and current_position > 0:  # Sell
            # Sell some or all
            sell_quantity = min(self.position_size, current_position)

            if sell_quantity > 0:
                asyncio.create_task(
                    self._place_order(OrderSide.SELL, sell_quantity)
                )

        # Record state transition (for possible online learning)
        self._remember_transition(state, action, price)

    def _get_position_ratio(self) -> float:
        """Get position ratio"""
        position = self._get_current_position()
        return position / self.max_position if self.max_position > 0 else 0

    def _get_position_return(self) -> float:
        """Get position return"""
        # Need to track average position cost here
        # Simplified implementation
        return 0.0

    def _get_cash_ratio(self) -> float:
        """Get cash ratio"""
        # Simplified implementation
        return 0.5

    def _get_available_cash(self) -> float:
        """Get available cash"""
        # Simplified implementation
        return 100000  # Assumed available funds

    async def _place_order(self, side: OrderSide, quantity: int):
        """Place order"""
        order = self.order_factory.market(
            symbol=self.symbol,
            side=side,
            quantity=Quantity.from_int(quantity)
        )
        await self.submit_order(order)

    def _remember_transition(self, state: np.ndarray, action: int, price: float):
        """Remember state transition (for experience replay)"""
        if len(self.memory) < self.memory.maxlen:
            self.memory.append((state, action, price))

    def get_model_confidence(self, state: np.ndarray) -> float:
        """Get model prediction confidence"""
        with torch.no_grad():
            state_tensor = torch.FloatTensor(state).unsqueeze(0).to(self.device)
            q_values = self.model(state_tensor)

            # Use difference between max Q-value and other Q-values as confidence
            q_values_array = q_values.cpu().numpy()[0]
            max_q = np.max(q_values_array)
            mean_q = np.mean(q_values_array)

            confidence = (max_q - mean_q) / (np.std(q_values_array) + 1e-6)
            return float(confidence)

    def update_model(self, new_model_path: str):
        """Hot update model"""
        try:
            self.model.load_state_dict(torch.load(new_model_path, map_location=self.device))
            self.model.eval()
            self.model_path = new_model_path
            self.log.info(f"Model updated to: {new_model_path}")
        except Exception as e:
            self.log.error(f"Failed to update model: {e}")
```

## Summary

This chapter detailed the implementation of various algorithmic trading strategies, covering:

1. **Market Making Strategies**
   - Basic market making strategies
   - High-frequency market making strategies
   - Inventory management and spread adjustment

2. **Arbitrage Strategies**
   - Statistical arbitrage
   - Pair trading
   - Cointegration analysis

3. **Momentum Strategies**
   - Time series momentum
   - Technical indicator combinations
   - Multi-condition confirmation

4. **Machine Learning Strategies**
   - Reinforcement learning
   - Deep Q Networks
   - Online learning

5. **Performance Optimization**
   - JIT compilation
   - Vectorized computation
   - High-frequency data processing

Through this chapter, readers can build various complex algorithmic trading strategies and select appropriate strategy types based on different market environments.

## References

- [Algorithmic Trading: Winning Strategies and Their Rationale](https://www.wiley.com/en-us/Algorithmic+Trading%3A+Winning+Strategies+and+Their+Rationale)
- [Advances in Financial Machine Learning](https://www.wiley.com/en-us/Advances+in+Financial+Machine+Learning)
- [Reinforcement Learning for Finance](https://rlfinance.io/)

## Runnable Code Entry (Source Build)

- Entry script: docs/tutorials_cn/code/chapter16/run_algorithmic_trading.py
- Key examples: docs/tutorials_cn/code/chapter16/mean_reversion.py, docs/tutorials_cn/code/chapter16/momentum.py, docs/tutorials_cn/code/chapter16/pairs_trading.py

Dependency strategy: All strategies share a unified signal interface and degrade safely to HOLD on insufficient data.

Verified command:

```bash
uv run --python 3.13 python docs/tutorials_cn/code/chapter16/run_algorithmic_trading.py
```
