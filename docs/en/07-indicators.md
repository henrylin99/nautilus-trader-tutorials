# Chapter 7: Technical Indicators System

## Learning Objectives

After completing this chapter, you will:
- Master the use of NautilusTrader's built-in technical indicators
- Understand the update mechanism and lifecycle of indicators
- Learn to create custom technical indicators
- Implement indicator cascading and combinations
- Optimize indicator calculation performance

## 7.1 Technical Indicators Overview

### 7.1.1 Indicator Base Class

All indicators in NautilusTrader inherit from the `Indicator` base class:

```python
"""
Technical Indicator Base Class Documentation
"""

from abc import ABC, abstractmethod
from decimal import Decimal
from typing import Any, Optional

from nautilus_trader.model.data import Bar
from nautilus_trader.core.message import Event


class Indicator(ABC):
    """Technical indicator base class"""

    def __init__(
        self,
        params: Optional[dict] = None,
        name: Optional[str] = None,
    ):
        """
        Initialize indicator

        Parameters
        ----------
        params : dict, optional
            Indicator parameters
        name : str, optional
            Indicator name
        """
        self._params = params or {}
        self._name = name or self.__class__.__name__
        self._initialized = False
        self._has_inputs = False
        self._inputs = 0

    @property
    def name(self) -> str:
        """Get indicator name"""
        return self._name

    @property
    def initialized(self) -> bool:
        """Whether initialized"""
        return self._initialized

    @property
    def has_inputs(self) -> bool:
        """Whether has input data"""
        return self._has_inputs

    @property
    def count_inputs(self) -> int:
        """Number of input data"""
        return self._inputs

    @abstractmethod
    def handle_bar(self, bar: Bar) -> None:
        """
        Handle bar data (abstract method)

        Parameters
        ----------
        bar : Bar
            Bar data
        """
        pass

    @abstractmethod
    def handle_quote_tick(self, quote) -> None:
        """
        Handle quote data (abstract method)

        Parameters
        ----------
        quote : QuoteTick
            Quote data
        """
        pass

    @abstractmethod
    def handle_trade_tick(self, trade) -> None:
        """
        Handle trade data (abstract method)

        Parameters
        ----------
        trade : TradeTick
            Trade data
        """
        pass

    @abstractmethod
    def reset(self) -> None:
        """Reset indicator state"""
        pass

    @abstractmethod
    def _update(self, value: float) -> None:
        """Update indicator value (internal method)"""
        pass

    def update_raw(self, value: float) -> None:
        """
        Update raw value

        Parameters
        ----------
        value : float
            Raw value
        """
        if not isinstance(value, (int, float, Decimal)):
            raise TypeError(f"Invalid input type: {type(value)}")

        self._has_inputs = True
        self._inputs += 1
        self._update(float(value))

    def to_dict(self) -> dict:
        """Convert to dictionary"""
        return {
            'name': self.name,
            'params': self._params,
            'initialized': self.initialized,
            'inputs': self._inputs,
            'has_inputs': self.has_inputs,
        }


# Custom indicator example
class CustomIndicator(Indicator):
    """Custom indicator example"""

    def __init__(self, period: int = 20):
        """
        Initialize indicator

        Parameters
        ----------
        period : int
            Calculation period
        """
        super().__init__(
            params={'period': period},
            name="CustomIndicator"
        )

        self.period = period
        self.values = []  # Store historical values
        self._value = 0.0

    @property
    def value(self) -> float:
        """Get indicator value"""
        return self._value

    @property
    def value_1(self) -> float:
        """Get previous value"""
        return self.values[-2] if len(self.values) >= 2 else 0.0

    def handle_bar(self, bar: Bar) -> None:
        """Handle bar data"""
        self.update_raw(float(bar.close))

    def handle_quote_tick(self, quote) -> None:
        """Handle quote data"""
        # Use mid price
        mid_price = (float(quote.bid_price) + float(quote.ask_price)) / 2
        self.update_raw(mid_price)

    def handle_trade_tick(self, trade) -> None:
        """Handle trade data"""
        self.update_raw(float(trade.price))

    def reset(self) -> None:
        """Reset indicator"""
        self.values.clear()
        self._value = 0.0
        self._initialized = False
        self._has_inputs = False
        self._inputs = 0

    def _update(self, value: float) -> None:
        """Update indicator value"""
        self.values.append(value)

        if len(self.values) > self.period:
            self.values.pop(0)

        # Calculate simple moving average
        if len(self.values) >= self.period:
            self._value = sum(self.values) / self.period
            self._initialized = True
```

### 7.1.2 Built-in Indicator Library

NautilusTrader provides a rich set of built-in indicators:

```python
"""
Built-in Indicator Library Overview
"""

# Moving average indicators
from nautilus_trader.indicators.average import (
    SMA,      # Simple Moving Average
    EMA,      # Exponential Moving Average
    WMA,      # Weighted Moving Average
    DEMA,     # Double Exponential Moving Average
    TEMA,     # Triple Exponential Moving Average
    KAMA,     # Kaufman's Adaptive Moving Average
    MAMA,     # MESA Adaptive Moving Average
    FRAMA,    # Fractal Adaptive Moving Average
    HMA,      # Hull Moving Average
)

# Momentum indicators
from nautilus_trader.indicators.momentum import (
    RSI,      # Relative Strength Index
    MACD,     # MACD Indicator
    Stochastic, # Stochastic Oscillator
    CCI,      # Commodity Channel Index
    ROC,      # Rate of Change
    MOMENTUM, # Momentum Indicator
    WILLIAMS_R, # Williams %R
)

# Trend indicators
from nautilus_trader.indicators.trend import (
    ADX,      # Average Directional Index
    Aroon,    # Aroon Indicator
    ATR,      # Average True Range
    ParabolicSAR, # Parabolic SAR
    TRIX,     # Triple Exponential Moving Average
    Vortex,   # Vortex Indicator
)

# Volatility indicators
from nautilus_trader.indicators.volatility import (
    BollingerBands, # Bollinger Bands
    KeltnerChannels, # Keltner Channels
    DonchianChannels, # Donchian Channels
    StandardDeviation, # Standard Deviation
)

# Volume indicators
from nautilus_trader.indicators.volume import (
    OBV,      # On-Balance Volume
    VWAP,     # Volume Weighted Average Price
    MoneyFlowIndex, # Money Flow Index
    AccumulationDistribution, # Accumulation/Distribution Line
    EaseOfMovement, # Ease of Movement
)

# Fuzzy logic indicators
from nautilus_trader.indicators.fuzzy_candlesticks import (
    FuzzyCandlestickPattern, # Fuzzy Candlestick Pattern
)
```

## 7.2 Moving Average Indicators

### 7.2.1 Basic Moving Averages

```python
"""
Moving Average Usage Example
"""

from decimal import Decimal
from nautilus_trader.model.data import Bar
from nautilus_trader.model.data import BarType
from nautilus_trader.model.enums import BarAggregation, PriceType
from nautilus_trader.model.identifiers import InstrumentId
from nautilus_trader.trading.strategy import Strategy
from nautilus_trader.common.enums import LogColor


class MovingAverageStrategy(Strategy):
    """Moving average strategy"""

    def __init__(
        self,
        bar_type: BarType,
        fast_period: int = 10,
        slow_period: int = 30,
    ):
        """
        Initialize strategy

        Parameters
        ----------
        bar_type : BarType
            Bar type
        fast_period : int
            Fast MA period
        slow_period : int
            Slow MA period
        """
        super().__init__()

        self.bar_type = bar_type

        # Create moving average indicators
        self.sma_fast = SMA(period=fast_period)
        self.sma_slow = SMA(period=slow_period)

        # Can also use EMA
        # self.ema_fast = EMA(period=fast_period)
        # self.ema_slow = EMA(period=slow_period)

        # State tracking
        self.last_signal = None
        self.position = None

    def on_start(self):
        """Strategy start"""
        self.log.info("Moving average strategy started")
        self.log.info(f"Fast period: {self.sma_fast.period}, "
                     f"Slow period: {self.sma_slow.period}")

        # Subscribe to bar data
        self.subscribe_bars(self.bar_type)

    def on_bar(self, bar: Bar):
        """Process bar data"""
        # Update indicators
        self.sma_fast.update_raw(float(bar.close))
        self.sma_slow.update_raw(float(bar.close))

        # Wait for indicators to initialize
        if not (self.sma_fast.initialized and self.sma_slow.initialized):
            return

        # Get indicator values
        fast_value = self.sma_fast.value
        slow_value = self.sma_slow.value

        # Check for golden cross/death cross
        self._check_crossover(fast_value, slow_value, bar)

        # Log indicator values
        if self.count_bars() % 10 == 0:  # Log every 10 bars
            self.log.info(
                f"MA - Fast: {fast_value:.2f}, "
                f"Slow: {slow_value:.2f}, "
                f"Close: {bar.close:.2f}",
                color=LogColor.BLUE
            )

    def _check_crossover(self, fast: float, slow: float, bar: Bar):
        """Check moving average crossover"""
        fast_prev = self.sma_fast.value_1
        slow_prev = self.sma_slow.value_1

        # Golden cross: fast MA crosses above slow MA
        if (fast > slow and fast_prev <= slow_prev and
            self.position != "LONG"):
            self._enter_long(bar)
            self.last_signal = "GOLDEN_CROSS"

        # Death cross: fast MA crosses below slow MA
        elif (fast < slow and fast_prev >= slow_prev and
              self.position == "LONG"):
            self._exit_long(bar)
            self.last_signal = "DEATH_CROSS"

    def _enter_long(self, bar: Bar):
        """Enter long position"""
        self.position = "LONG"
        self.log.info(
            f"Golden cross signal - Buy price: {bar.close:.2f}",
            color=LogColor.GREEN
        )
        # Actual buy order should be placed here
        # buy_order = MarketOrder(...)
        # self.submit_order(buy_order)

    def _exit_long(self, bar: Bar):
        """Exit long position"""
        self.position = None
        self.log.info(
            f"Death cross signal - Sell price: {bar.close:.2f}",
            color=LogColor.RED
        )
        # Actual sell order should be placed here
        # sell_order = MarketOrder(...)
        # self.submit_order(sell_order)


# Advanced moving average strategy example
class AdvancedMAStrategy(Strategy):
    """Advanced moving average strategy"""

    def __init__(self, bar_type: BarType):
        """Initialize strategy"""
        super().__init__()
        self.bar_type = bar_type

        # Multiple moving averages
        self.sma_short = SMA(period=5)
        self.sma_medium = SMA(period=20)
        self.sma_long = SMA(period=50)

        # EMA for trend confirmation
        self.ema_trend = EMA(period=100)

        # Weighted moving average
        self.wma_smooth = WMA(period=10)

        # Adaptive moving average
        self.kama_fast = KAMA(
            fast_period=2,
            slow_period=30,
            period=10
        )

        # State variables
        self.trend_strength = 0
        self.market_state = "NEUTRAL"

    def on_start(self):
        """Strategy start"""
        self.log.info("Advanced MA strategy started")
        self.subscribe_bars(self.bar_type)

    def on_bar(self, bar: Bar):
        """Process bar data"""
        price = float(bar.close)

        # Update all indicators
        self.sma_short.update_raw(price)
        self.sma_medium.update_raw(price)
        self.sma_long.update_raw(price)
        self.ema_trend.update_raw(price)
        self.wma_smooth.update_raw(price)
        self.kama_fast.update_raw(price)

        # Wait for indicators to initialize
        if not (self.sma_long.initialized and self.ema_trend.initialized):
            return

        # Analyze market state
        self._analyze_market_state()

        # Generate trading signal
        signal = self._generate_signal()

        if signal:
            self.log.info(
                f"Trading signal: {signal['type']} "
                f"(Strength: {signal['strength']:.2f})",
                color=LogColor.CYAN
            )

    def _analyze_market_state(self):
        """Analyze market state"""
        short = self.sma_short.value
        medium = self.sma_medium.value
        long = self.sma_long.value

        # Calculate moving average alignment
        if short > medium > long:
            self.market_state = "BULLISH"
        elif short < medium < long:
            self.market_state = "BEARISH"
        else:
            self.market_state = "NEUTRAL"

        # Calculate trend strength
        spread_long_short = abs(short - long) / long
        self.trend_strength = min(spread_long_short * 10, 1.0)

    def _generate_signal(self) -> Optional[dict]:
        """Generate trading signal"""
        if self.market_state == "NEUTRAL":
            return None

        # Use KAMA for confirmation
        kama_value = self.kama_fast.value
        current_price = self.sma_short.value

        # Multiple confirmations
        conditions = []

        # 1. Market state
        if self.market_state == "BULLISH":
            conditions.append(kama_value > current_price * 0.995)  # Near or above
        elif self.market_state == "BEARISH":
            conditions.append(kama_value < current_price * 1.005)  # Near or below

        # 2. Trend strength
        conditions.append(self.trend_strength > 0.3)

        # 3. EMA trend confirmation
        if self.market_state == "BULLISH":
            conditions.append(self.ema_trend.value < current_price)
        elif self.market_state == "BEARISH":
            conditions.append(self.ema_trend.value > current_price)

        # Generate signal
        if all(conditions):
            return {
                'type': "BUY" if self.market_state == "BULLISH" else "SELL",
                'strength': self.trend_strength,
                'confidence': len(conditions) / 3,
            }

        return None
```

### 7.2.2 Custom Moving Averages

```python
"""
Custom Moving Average Indicators
"""

from decimal import Decimal
import math


class HullMovingAverage(Indicator):
    """Hull Moving Average (HMA)"""

    def __init__(self, period: int = 20):
        """
        Initialize HMA

        Parameters
        ----------
        period : int
            Period
        """
        super().__init__(params={'period': period}, name="HMA")
        self.period = period
        self.half_period = int(period / 2)
        self.sqrt_period = int(math.sqrt(period))

        # Internal WMA indicators
        self.wma_full = WMA(period=period)
        self.wma_half = WMA(period=self.half_period)
        self.wma_diff = WMA(period=self.sqrt_period)

        self._value = 0.0

    @property
    def value(self) -> float:
        """Get HMA value"""
        return self._value

    def handle_bar(self, bar: Bar):
        """Handle bar data"""
        self.update_raw(float(bar.close))

    def handle_quote_tick(self, quote):
        """Handle quote data"""
        mid_price = (float(quote.bid_price) + float(quote.ask_price)) / 2
        self.update_raw(mid_price)

    def handle_trade_tick(self, trade):
        """Handle trade data"""
        self.update_raw(float(trade.price))

    def reset(self):
        """Reset indicator"""
        self.wma_full.reset()
        self.wma_half.reset()
        self.wma_diff.reset()
        self._value = 0.0
        self._initialized = False
        self._has_inputs = False
        self._inputs = 0

    def _update(self, value: float):
        """Update HMA value"""
        # Update internal WMAs
        self.wma_full.update_raw(value)
        self.wma_half.update_raw(value)

        if self.wma_full.initialized and self.wma_half.initialized:
            # Calculate 2 * WMA(n/2) - WMA(n)
            diff_value = 2 * self.wma_half.value - self.wma_full.value
            self.wma_diff.update_raw(diff_value)

            if self.wma_diff.initialized:
                self._value = self.wma_diff.value
                self._initialized = True


class JurikMovingAverage(Indicator):
    """Jurik Moving Average (JMA) - Smoothness optimized moving average"""

    def __init__(
        self,
        period: int = 14,
        phase: float = 0.0,
        power: float = 2.0,
    ):
        """
        Initialize JMA

        Parameters
        ----------
        period : int
            Period
        phase : float
            Phase (-100 to 100)
        power : float
            Power
        """
        super().__init__(
            params={
                'period': period,
                'phase': phase,
                'power': power
            },
            name="JMA"
        )

        self.period = period
        self.phase = phase
        self.power = power

        # JMA parameters
        self.phase_ratio = phase / 100.0 + 1.5
        self.beta = 0.45 * (period - 1) / (0.45 * (period - 1) + 2)
        self.alpha = self.beta

        # Internal variables
        self.prev_jma = 0.0
        self.prev_det0 = 0.0
        self.prev_det1 = 0.0

        self._value = 0.0
        self._initialized = False

    @property
    def value(self) -> float:
        """Get JMA value"""
        return self._value

    def handle_bar(self, bar: Bar):
        """Handle bar data"""
        self.update_raw(float(bar.close))

    def handle_quote_tick(self, quote):
        """Handle quote data"""
        mid_price = (float(quote.bid_price) + float(quote.ask_price)) / 2
        self.update_raw(mid_price)

    def handle_trade_tick(self, trade):
        """Handle trade data"""
        self.update_raw(float(trade.price))

    def reset(self):
        """Reset indicator"""
        self.prev_jma = 0.0
        self.prev_det0 = 0.0
        self.prev_det1 = 0.0
        self._value = 0.0
        self._initialized = False
        self._has_inputs = False
        self._inputs = 0

    def _update(self, value: float):
        """Update JMA value"""
        if not self.has_inputs:
            # First input
            self.prev_jma = value
            self.prev_det0 = 0.0
            self.prev_det1 = 0.0
            self._initialized = True
        else:
            # JMA calculation
            det0 = (1 - self.alpha) * (value + value - self.prev_jma)
            det1 = self.alpha * self.prev_det0 + det0

            # Power transformation
            if self.power != 1:
                abs_det1 = abs(det1) ** self.power
                if det1 < 0:
                    det1 = -abs_det1
                else:
                    det1 = abs_det1

            # Update JMA
            self._value = self.prev_jma + self.phase_ratio * det1

            # Save historical values
            self.prev_jma = self._value
            self.prev_det0 = det0
            self.prev_det1 = det1


# Strategy using custom MAs
class CustomMAStrategy(Strategy):
    """Strategy using custom moving averages"""

    def __init__(self, bar_type: BarType):
        """Initialize strategy"""
        super().__init__()
        self.bar_type = bar_type

        # Custom moving averages
        self.hma_fast = HullMovingAverage(period=10)
        self.hma_slow = HullMovingAverage(period=30)
        self.jma_smooth = JurikMovingAverage(period=20, phase=0)

        # State tracking
        self.hma_spread = 0.0
        self.jma_trend = 0.0

    def on_start(self):
        """Strategy start"""
        self.log.info("Custom MA strategy started")
        self.subscribe_bars(self.bar_type)

    def on_bar(self, bar: Bar):
        """Process bar data"""
        price = float(bar.close)

        # Update indicators
        self.hma_fast.update_raw(price)
        self.hma_slow.update_raw(price)
        self.jma_smooth.update_raw(price)

        # Wait for initialization
        if not (self.hma_fast.initialized and
                self.hma_slow.initialized and
                self.jma_smooth.initialized):
            return

        # Calculate indicator differences
        self.hma_spread = (self.hma_fast.value - self.hma_slow.value) / self.hma_slow.value
        self.jma_trend = self.jma_smooth.value - self.jma_smooth.value_1

        # Generate signals
        self._check_signals(bar)

    def _check_signals(self, bar: Bar):
        """Check trading signals"""
        # HMA crossover signal
        hma_cross = (
            (self.hma_fast.value > self.hma_slow.value) and
            (self.hma_fast.value_1 <= self.hma_slow.value_1)
        ) or (
            (self.hma_fast.value < self.hma_slow.value) and
            (self.hma_fast.value_1 >= self.hma_slow.value_1)
        )

        # JMA trend confirmation
        trend_confirm = abs(self.jma_trend) > 0.001

        if hma_cross and trend_confirm:
            signal = "BUY" if self.hma_fast.value > self.hma_slow.value else "SELL"
            self.log.info(
                f"Signal: {signal}, HMA spread: {self.hma_spread:.4f}, "
                f"JMA trend: {self.jma_trend:.4f}",
                color=LogColor.YELLOW
            )
```

## 7.3 Momentum Indicators

### 7.3.1 RSI Indicator

```python
"""
RSI (Relative Strength Index) Usage Example
"""

from decimal import Decimal
from nautilus_trader.indicators.momentum import RSI


class RSIStrategy(Strategy):
    """RSI strategy"""

    def __init__(
        self,
        bar_type: BarType,
        rsi_period: int = 14,
        oversold: float = 30,
        overbought: float = 70,
    ):
        """
        Initialize RSI strategy

        Parameters
        ----------
        bar_type : BarType
            Bar type
        rsi_period : int
            RSI period
        oversold : float
            Oversold level
        overbought : float
            Overbought level
        """
        super().__init__()
        self.bar_type = bar_type

        # Create RSI indicator
        self.rsi = RSI(period=rsi_period)

        # Parameters
        self.oversold = oversold
        self.overbought = overbought

        # State
        self.last_rsi = 0.0
        self.divergences = []

    def on_start(self):
        """Strategy start"""
        self.log.info(
            f"RSI strategy started - Period: {self.rsi.period}, "
            f"Oversold: {self.oversold}, Overbought: {self.overbought}"
        )
        self.subscribe_bars(self.bar_type)

    def on_bar(self, bar: Bar):
        """Process bar data"""
        # Update RSI
        self.rsi.update_raw(float(bar.close))

        if not self.rsi.initialized:
            return

        current_rsi = self.rsi.value
        price = float(bar.close)

        # Check RSI signals
        self._check_rsi_signals(current_rsi, price)

        # Check divergences
        self._check_divergences(current_rsi, price)

        # Record RSI value
        self.last_rsi = current_rsi

        # Periodic output
        if self.count_bars() % 20 == 0:
            self.log.info(
                f"RSI: {current_rsi:.2f}, Price: {price:.2f}",
                color=LogColor.MAGENTA
            )

    def _check_rsi_signals(self, rsi: float, price: float):
        """Check RSI signals"""
        # Oversold bounce
        if self.last_rsi < self.oversold and rsi > self.oversold:
            self.log.info(
                f"RSI oversold bounce - RSI: {rsi:.2f}",
                color=LogColor.GREEN
            )
            self._generate_buy_signal(rsi, price, "OVERSOLD_RECOVERY")

        # Overbought pullback
        elif self.last_rsi > self.overbought and rsi < self.overbought:
            self.log.info(
                f"RSI overbought pullback - RSI: {rsi:.2f}",
                color=LogColor.RED
            )
            self._generate_sell_signal(rsi, price, "OVERBOUGHT_PULLBACK")

        # Center line cross
        elif self.last_rsi < 50 and rsi > 50:
            self.log.info(
                f"RSI crosses center line up - RSI: {rsi:.2f}",
                color=LogColor.BLUE
            )
            self._generate_buy_signal(rsi, price, "CENTER_CROSS_UP")

        elif self.last_rsi > 50 and rsi < 50:
            self.log.info(
                f"RSI crosses center line down - RSI: {rsi:.2f}",
                color=LogColor.ORANGE
            )
            self._generate_sell_signal(rsi, price, "CENTER_CROSS_DOWN")

    def _check_divergences(self, rsi: float, price: float):
        """Check divergences"""
        # Need to maintain historical price and RSI data
        # Simplified implementation here
        pass

    def _generate_buy_signal(self, rsi: float, price: float, reason: str):
        """Generate buy signal"""
        signal = {
            'type': 'BUY',
            'rsi': rsi,
            'price': price,
            'reason': reason,
            'strength': self._calculate_signal_strength(rsi, 'BUY')
        }
        self.log.info(f"Buy signal: {signal}", color=LogColor.GREEN)

    def _generate_sell_signal(self, rsi: float, price: float, reason: str):
        """Generate sell signal"""
        signal = {
            'type': 'SELL',
            'rsi': rsi,
            'price': price,
            'reason': reason,
            'strength': self._calculate_signal_strength(rsi, 'SELL')
        }
        self.log.info(f"Sell signal: {signal}", color=LogColor.RED)

    def _calculate_signal_strength(self, rsi: float, signal_type: str) -> float:
        """Calculate signal strength"""
        if signal_type == 'BUY':
            # Deeper oversold = stronger signal
            return min((50 - rsi) / 20, 1.0)
        else:
            # Higher overbought = stronger signal
            return min((rsi - 50) / 20, 1.0)


# RSI + MA combination strategy
class RSIWithMAStrategy(Strategy):
    """RSI and moving average combination strategy"""

    def __init__(self, bar_type: BarType):
        """Initialize strategy"""
        super().__init__()
        self.bar_type = bar_type

        # Indicators
        self.rsi = RSI(period=14)
        self.ema_fast = EMA(period=20)
        self.ema_slow = EMA(period=50)

        # State
        self.trend = "NEUTRAL"
        self.rsi_state = "NEUTRAL"

    def on_start(self):
        """Strategy start"""
        self.log.info("RSI+MA combination strategy started")
        self.subscribe_bars(self.bar_type)

    def on_bar(self, bar: Bar):
        """Process bar data"""
        price = float(bar.close)

        # Update indicators
        self.rsi.update_raw(price)
        self.ema_fast.update_raw(price)
        self.ema_slow.update_raw(price)

        if not (self.rsi.initialized and
                self.ema_fast.initialized and
                self.ema_slow.initialized):
            return

        # Determine trend
        self._determine_trend()

        # Determine RSI state
        self._determine_rsi_state()

        # Generate signal
        self._generate_combined_signal(price)

    def _determine_trend(self):
        """Determine trend"""
        if self.ema_fast.value > self.ema_slow.value:
            self.trend = "BULLISH"
        else:
            self.trend = "BEARISH"

    def _determine_rsi_state(self):
        """Determine RSI state"""
        if self.rsi.value < 30:
            self.rsi_state = "OVERSOLD"
        elif self.rsi.value > 70:
            self.rsi_state = "OVERBOUGHT"
        else:
            self.rsi_state = "NEUTRAL"

    def _generate_combined_signal(self, price: float):
        """Generate combined signal"""
        # Bull market + RSI oversold = Buy
        if (self.trend == "BULLISH" and
            self.rsi_state == "OVERSOLD"):
            self.log.info(
                f"Combined buy signal - Trend: {self.trend}, RSI: {self.rsi.value:.1f}",
                color=LogColor.GREEN
            )

        # Bear market + RSI overbought = Sell
        elif (self.trend == "BEARISH" and
              self.rsi_state == "OVERBOUGHT"):
            self.log.info(
                f"Combined sell signal - Trend: {self.trend}, RSI: {self.rsi.value:.1f}",
                color=LogColor.RED
            )
```

### 7.3.2 MACD Indicator

```python
"""
MACD (Moving Average Convergence Divergence) Usage Example
"""

from nautilus_trader.indicators.momentum import MACD


class MACDStrategy(Strategy):
    """MACD strategy"""

    def __init__(
        self,
        bar_type: BarType,
        fast_period: int = 12,
        slow_period: int = 26,
        signal_period: int = 9,
    ):
        """
        Initialize MACD strategy

        Parameters
        ----------
        bar_type : BarType
            Bar type
        fast_period : int
            Fast line period
        slow_period : int
            Slow line period
        signal_period : int
            Signal line period
        """
        super().__init__()
        self.bar_type = bar_type

        # Create MACD indicator
        self.macd = MACD(
            fast_period=fast_period,
            slow_period=slow_period,
            signal_period=signal_period,
        )

        # State
        self.last_signal = 0  # MACD histogram sign
        self.histogram_trend = []  # Histogram trend

    def on_start(self):
        """Strategy start"""
        self.log.info(
            f"MACD strategy started - "
            f"Fast: {self.macd.fast_period}, "
            f"Slow: {self.macd.slow_period}, "
            f"Signal: {self.macd.signal_period}"
        )
        self.subscribe_bars(self.bar_type)

    def on_bar(self, bar: Bar):
        """Process bar data"""
        # Update MACD
        self.macd.update_raw(float(bar.close))

        if not self.macd.initialized:
            return

        # Get MACD values
        macd_line = self.macd.value
        signal_line = self.macd.signal
        histogram = self.macd.histogram

        # Check signals
        self._check_macd_signals(macd_line, signal_line, histogram, bar)

        # Log values
        if self.count_bars() % 10 == 0:
            self.log.info(
                f"MACD - Line: {macd_line:.4f}, "
                f"Signal: {signal_line:.4f}, "
                f"Histogram: {histogram:.4f}",
                color=LogColor.CYAN
            )

    def _check_macd_signals(
        self,
        macd_line: float,
        signal_line: float,
        histogram: float,
        bar: Bar
    ):
        """Check MACD signals"""
        # Record histogram trend
        self.histogram_trend.append(histogram)
        if len(self.histogram_trend) > 3:
            self.histogram_trend.pop(0)

        # 1. MACD line and signal line crossover
        if (macd_line > signal_line and self.last_signal <= 0):
            self._on_macd_bull_cross(bar)
        elif (macd_line < signal_line and self.last_signal >= 0):
            self._on_macd_bear_cross(bar)

        # 2. Histogram divergence
        if len(self.histogram_trend) >= 3:
            self._check_histogram_divergence(bar)

        # 3. Zero line cross
        if self.last_signal * histogram < 0:  # Sign change
            if histogram > 0:
                self._on_zero_cross_up(bar)
            else:
                self._on_zero_cross_down(bar)

        self.last_signal = histogram

    def _on_macd_bull_cross(self, bar: Bar):
        """MACD bullish cross"""
        self.log.info(
            f"MACD bullish cross - Price: {bar.close:.2f}",
            color=LogColor.GREEN
        )

    def _on_macd_bear_cross(self, bar: Bar):
        """MACD bearish cross"""
        self.log.info(
            f"MACD bearish cross - Price: {bar.close:.2f}",
            color=LogColor.RED
        )

    def _on_zero_cross_up(self, bar: Bar):
        """MACD crosses zero line up"""
        self.log.info(
            f"MACD crosses zero line up - Price: {bar.close:.2f}",
            color=LogColor.BLUE
        )

    def _on_zero_cross_down(self, bar: Bar):
        """MACD crosses zero line down"""
        self.log.info(
            f"MACD crosses zero line down - Price: {bar.close:.2f}",
            color=LogColor.ORANGE
        )

    def _check_histogram_divergence(self, bar: Bar):
        """Check histogram divergence"""
        # Simplified divergence detection
        h0, h1, h2 = self.histogram_trend[-3:]
        current_price = float(bar.close)

        # Bullish divergence: price new low but histogram trough rising
        if (h2 < h1 < h0 and h0 < 0):  # Rising troughs
            # Need price data for complete implementation
            pass

        # Bearish divergence: price new high but histogram peak falling
        elif (h2 > h1 > h0 and h0 > 0):  # Falling peaks
            # Need price data for complete implementation
            pass
```

## 7.4 Trend and Volatility Indicators

### 7.4.1 Bollinger Bands

```python
"""
Bollinger Bands Usage Example
"""

from nautilus_trader.indicators.volatility import BollingerBands
from nautilus_trader.indicators.volatility import KeltnerChannels
from nautilus_trader.indicators.volatility import DonchianChannels


class BollingerBandsStrategy(Strategy):
    """Bollinger Bands strategy"""

    def __init__(
        self,
        bar_type: BarType,
        period: int = 20,
        std_dev: float = 2.0,
    ):
        """
        Initialize Bollinger Bands strategy

        Parameters
        ----------
        bar_type : BarType
            Bar type
        period : int
            Moving average period
        std_dev : float
            Standard deviation multiplier
        """
        super().__init__()
        self.bar_type = bar_type

        # Bollinger Bands indicator
        self.bbands = BollingerBands(
            period=period,
            std_dev=std_dev,
        )

        # Auxiliary indicators
        self.keltner = KeltnerChannels(
            period=period,
            multiplier=2.0,
        )
        self.donchian = DonchianChannels(period=period)

        # State
        self.price_position = "MIDDLE"  # UPPER, MIDDLE, LOWER
        self.band_width = 0.0
        self.squeeze_detected = False

    def on_start(self):
        """Strategy start"""
        self.log.info(
            f"Bollinger Bands strategy started - "
            f"Period: {self.bbands.period}, "
            f"Std Dev: {self.bbands.std_dev}"
        )
        self.subscribe_bars(self.bar_type)

    def on_bar(self, bar: Bar):
        """Process bar data"""
        price = float(bar.close)
        high = float(bar.high)
        low = float(bar.low)

        # Update indicators
        self.bbands.update_raw(price)
        self.keltner.update_raw(price)
        self.donchian.update_high(high)
        self.donchian.update_low(low)

        if not (self.bbands.initialized and
                self.keltner.initialized and
                self.donchian.initialized):
            return

        # Get Bollinger Bands values
        bb_upper = self.bbands.upper
        bb_middle = self.bbands.middle
        bb_lower = self.bbands.lower

        # Calculate price position
        if price > bb_upper:
            self.price_position = "ABOVE_UPPER"
        elif price < bb_lower:
            self.price_position = "BELOW_LOWER"
        elif price > bb_middle:
            self.price_position = "UPPER_HALF"
        else:
            self.price_position = "LOWER_HALF"

        # Calculate band width
        self.band_width = (bb_upper - bb_lower) / bb_middle

        # Detect squeeze
        self._detect_squeeze()

        # Generate signals
        self._check_signals(price, bar)

        # Log information
        if self.count_bars() % 10 == 0:
            self.log.info(
                f"BB - Upper: {bb_upper:.2f}, "
                f"Middle: {bb_middle:.2f}, "
                f"Lower: {bb_lower:.2f}, "
                f"Price: {price:.2f}",
                color=LogColor.MAGENTA
            )

    def _detect_squeeze(self):
        """Detect Bollinger Bands squeeze"""
        # Band width less than certain threshold
        min_width = 0.02  # 2%

        if self.band_width < min_width:
            if not self.squeeze_detected:
                self.squeeze_detected = True
                self.log.warning(
                    f"Bollinger Bands squeeze detected - Width: {self.band_width:.4f}",
                    color=LogColor.YELLOW
                )
        else:
            if self.squeeze_detected:
                self.squeeze_detected = False
                self.log.info("Bollinger Bands squeeze released", color=LogColor.CYAN)

    def _check_signals(self, price: float, bar: Bar):
        """Check trading signals"""
        bb_upper = self.bbands.upper
        bb_lower = self.bbands.lower

        # Mean reversion trading
        if self.price_position == "ABOVE_UPPER":
            self._generate_sell_signal(
                price=price,
                reason="UPPER_BAND_REJECTION",
                strength=self._calculate_strength("mean_reversion")
            )
        elif self.price_position == "BELOW_LOWER":
            self._generate_buy_signal(
                price=price,
                reason="LOWER_BAND_SUPPORT",
                strength=self._calculate_strength("mean_reversion")
            )

        # Breakout trading
        elif self.price_position == "UPPER_HALF":
            # Check for breakout
            if bar.close > bar.open and self.band_width > 0.04:
                self._generate_buy_signal(
                    price=price,
                    reason="UPPER_BAND_BREAKOUT",
                    strength=self._calculate_strength("breakout")
                )
        elif self.price_position == "LOWER_HALF":
            # Check for breakdown
            if bar.close < bar.open and self.band_width > 0.04:
                self._generate_sell_signal(
                    price=price,
                    reason="LOWER_BAND_BREAKOUT",
                    strength=self._calculate_strength("breakout")
                )

    def _calculate_strength(self, signal_type: str) -> float:
        """Calculate signal strength"""
        if signal_type == "mean_reversion":
            # Narrower band = stronger reversion signal
            return max(1.0 - self.band_width * 10, 0.3)
        elif signal_type == "breakout":
            # Wider band = stronger breakout signal
            return min(self.band_width * 15, 1.0)
        return 0.5

    def _generate_buy_signal(self, price: float, reason: str, strength: float):
        """Generate buy signal"""
        self.log.info(
            f"Buy signal - Price: {price:.2f}, "
            f"Reason: {reason}, Strength: {strength:.2f}",
            color=LogColor.GREEN
        )

    def _generate_sell_signal(self, price: float, reason: str, strength: float):
        """Generate sell signal"""
        self.log.info(
            f"Sell signal - Price: {price:.2f}, "
            f"Reason: {reason}, Strength: {strength:.2f}",
            color=LogColor.RED
        )
```

## 7.5 Custom Indicator Development

### 7.5.1 Creating Composite Indicators

```python
"""
Custom Composite Indicator Examples
"""

from typing import Optional
import numpy as np


class AdaptiveRSI(Indicator):
    """Adaptive RSI indicator"""

    def __init__(
        self,
        base_period: int = 14,
        min_period: int = 5,
        max_period: int = 30,
        sensitivity: float = 0.02,
    ):
        """
        Initialize Adaptive RSI

        Parameters
        ----------
        base_period : int
            Base period
        min_period : int
            Minimum period
        max_period : int
            Maximum period
        sensitivity : float
            Sensitivity
        """
        super().__init__(
            params={
                'base_period': base_period,
                'min_period': min_period,
                'max_period': max_period,
                'sensitivity': sensitivity
            },
            name="AdaptiveRSI"
        )

        self.base_period = base_period
        self.min_period = min_period
        self.max_period = max_period
        self.sensitivity = sensitivity

        # Internal RSI
        self.rsi = RSI(period=base_period)

        # Volatility calculation
        self.price_changes = []
        self.volatility = 0.0

        # Adaptive period
        self.current_period = base_period

        self._value = 0.0

    @property
    def value(self) -> float:
        """Get indicator value"""
        return self._value

    @property
    def adaptive_period(self) -> int:
        """Get adaptive period"""
        return self.current_period

    def handle_bar(self, bar: Bar):
        """Handle bar data"""
        self.update_raw(float(bar.close))

    def handle_quote_tick(self, quote):
        """Handle quote data"""
        mid_price = (float(quote.bid_price) + float(quote.ask_price)) / 2
        self.update_raw(mid_price)

    def handle_trade_tick(self, trade):
        """Handle trade data"""
        self.update_raw(float(trade.price))

    def reset(self):
        """Reset indicator"""
        self.rsi.reset()
        self.price_changes.clear()
        self.volatility = 0.0
        self.current_period = self.base_period
        self._value = 0.0
        self._initialized = False
        self._has_inputs = False
        self._inputs = 0

    def _update(self, value: float):
        """Update indicator value"""
        # Calculate price change
        if self.has_inputs:
            price_change = abs(value - self.price_changes[-1])
            self.price_changes.append(price_change)

            # Keep enough historical data
            if len(self.price_changes) > self.max_period * 2:
                self.price_changes.pop(0)

            # Calculate volatility
            if len(self.price_changes) >= self.base_period:
                self.volatility = np.std(self.price_changes[-self.base_period:])

                # Adaptive period adjustment
                self._adjust_period()

        else:
            self.price_changes.append(value)

        # Update internal RSI
        self.rsi.update_raw(value)

        # Adjust RSI period
        if self.rsi.period != self.current_period:
            self.rsi = RSI(period=self.current_period)
            # Recalculate historical data
            for price in self.price_changes[-self.current_period:]:
                self.rsi.update_raw(price)

        self._value = self.rsi.value

        if self.rsi.initialized:
            self._initialized = True

    def _adjust_period(self):
        """Adjust RSI period"""
        # Adjust period based on volatility
        # High volatility -> Short period
        # Low volatility -> Long period

        volatility_ratio = self.volatility / (np.mean(self.price_changes) + 1e-8)

        # Calculate new period
        period_adjustment = int(volatility_ratio * self.sensitivity * self.base_period)
        new_period = self.base_period - period_adjustment

        # Limit period range
        self.current_period = max(self.min_period, min(self.max_period, new_period))


class MultiTimeframeRSI(Indicator):
    """Multi-timeframe RSI indicator"""

    def __init__(
        self,
        short_period: int = 7,
        medium_period: int = 14,
        long_period: int = 28,
    ):
        """
        Initialize Multi-timeframe RSI

        Parameters
        ----------
        short_period : int
            Short period
        medium_period : int
            Medium period
        long_period : int
            Long period
        """
        super().__init__(
            params={
                'short_period': short_period,
                'medium_period': medium_period,
                'long_period': long_period
            },
            name="MultiTimeframeRSI"
        )

        # Three RSIs with different periods
        self.rsi_short = RSI(period=short_period)
        self.rsi_medium = RSI(period=medium_period)
        self.rsi_long = RSI(period=long_period)

        # Composite signal
        self._value = 50.0
        self.signal_strength = 0.0

    @property
    def value(self) -> float:
        """Get composite RSI value"""
        return self._value

    @property
    def strength(self) -> float:
        """Get signal strength"""
        return self.signal_strength

    @property
    def rsi_values(self) -> dict:
        """Get RSI values for each period"""
        return {
            'short': self.rsi_short.value if self.rsi_short.initialized else None,
            'medium': self.rsi_medium.value if self.rsi_medium.initialized else None,
            'long': self.rsi_long.value if self.rsi_long.initialized else None,
        }

    def handle_bar(self, bar: Bar):
        """Handle bar data"""
        self.update_raw(float(bar.close))

    def handle_quote_tick(self, quote):
        """Handle quote data"""
        mid_price = (float(quote.bid_price) + float(quote.ask_price)) / 2
        self.update_raw(mid_price)

    def handle_trade_tick(self, trade):
        """Handle trade data"""
        self.update_raw(float(trade.price))

    def reset(self):
        """Reset indicator"""
        self.rsi_short.reset()
        self.rsi_medium.reset()
        self.rsi_long.reset()
        self._value = 50.0
        self.signal_strength = 0.0
        self._initialized = False
        self._has_inputs = False
        self._inputs = 0

    def _update(self, value: float):
        """Update indicator value"""
        # Update all RSIs
        self.rsi_short.update_raw(value)
        self.rsi_medium.update_raw(value)
        self.rsi_long.update_raw(value)

        # Wait for all RSIs to initialize
        if not (self.rsi_short.initialized and
                self.rsi_medium.initialized and
                self.rsi_long.initialized):
            return

        # Calculate weighted average
        weights = [0.5, 0.3, 0.2]  # Short-term has highest weight
        rsi_values = [
            self.rsi_short.value,
            self.rsi_medium.value,
            self.rsi_long.value
        ]

        self._value = sum(w * r for w, r in zip(weights, rsi_values))

        # Calculate signal strength (based on RSI consistency)
        rsi_mean = np.mean(rsi_values)
        rsi_std = np.std(rsi_values)
        self.signal_strength = max(0, 1 - rsi_std / 50)  # Normalize to 0-1

        self._initialized = True
```

## 7.6 Performance Optimization

### 7.6.1 Batch Processing and Caching

```python
"""
Indicator Performance Optimization
"""

from typing import List, Tuple
import numpy as np
from functools import lru_cache


class OptimizedIndicator(Indicator):
    """Optimized indicator base class"""

    def __init__(self, batch_size: int = 100):
        """
        Initialize optimized indicator

        Parameters
        ----------
        batch_size : int
            Batch processing size
        """
        super().__init__()
        self.batch_size = batch_size
        self.buffer = []
        self._cache_enabled = True
        self._update_count = 0

    @lru_cache(maxsize=1024)
    def _cached_calculation(self, value: float) -> float:
        """Cached calculation (suitable for pure functions)"""
        # Subclass implements specific calculation logic
        return value

    def update_batch(self, values: List[float]):
        """
        Batch update

        Parameters
        ----------
        values : List[float]
            List of values
        """
        self.buffer.extend(values)

        # Process in batch when batch size is reached
        if len(self.buffer) >= self.batch_size:
            self._process_batch()
            self.buffer.clear()

    def _process_batch(self):
        """Process batch data"""
        # Use numpy for vectorized calculations
        np_values = np.array(self.buffer)

        # Subclass implements specific batch processing logic
        self._batch_update(np_values)

    def _batch_update(self, values: np.ndarray):
        """Batch update (subclass implementation)"""
        pass


class OptimizedEMA(OptimizedIndicator):
    """Optimized EMA indicator"""

    def __init__(
        self,
        period: int = 20,
        batch_size: int = 100,
        cache_size: int = 1024,
    ):
        """
        Initialize optimized EMA

        Parameters
        ----------
        period : int
            EMA period
        batch_size : int
            Batch processing size
        cache_size : int
            Cache size
        """
        super().__init__(batch_size= batch_size)
        self.period = period
        self.alpha = 2.0 / (period + 1)
        self._value = 0.0
        self._initialized = False

        # Enable caching
        self._cache_enabled = cache_size > 0
        if self._cache_enabled:
            self._cached_calculation = lru_cache(maxsize=cache_size)(self._calculate_ema)

    @property
    def value(self) -> float:
        """Get EMA value"""
        return self._value

    def handle_bar(self, bar: Bar):
        """Handle bar data"""
        self.update_raw(float(bar.close))

    def handle_quote_tick(self, quote):
        """Handle quote data"""
        mid_price = (float(quote.bid_price) + float(quote.ask_price)) / 2
        self.update_raw(mid_price)

    def handle_trade_tick(self, trade):
        """Handle trade data"""
        self.update_raw(float(trade.price))

    def reset(self):
        """Reset indicator"""
        self._value = 0.0
        self._initialized = False
        self.buffer.clear()
        self._update_count = 0

        # Clear cache
        if hasattr(self._cached_calculation, 'cache_clear'):
            self._cached_calculation.cache_clear()

    def _update(self, value: float):
        """Update EMA value"""
        if self._cache_enabled:
            # Use cached calculation
            self._value = self._cached_calculation(value)
        else:
            # Direct calculation
            self._value = self._calculate_ema(value)

        if not self._initialized:
            self._initialized = True

        self._update_count += 1

    def _calculate_ema(self, value: float) -> float:
        """Calculate EMA value"""
        if self._update_count == 0:
            # First value
            return value
        else:
            # EMA formula
            return (value * self.alpha) + (self._value * (1 - self.alpha))

    def _batch_update(self, values: np.ndarray):
        """Batch update EMA"""
        # Use numpy for vectorized calculations
        for value in values:
            self._update(float(value))


class IndicatorManager:
    """Indicator manager"""

    def __init__(self):
        """Initialize manager"""
        self.indicators = {}
        self.update_queue = []

    def add_indicator(self, name: str, indicator: Indicator):
        """
        Add indicator

        Parameters
        ----------
        name : str
            Indicator name
        indicator : Indicator
            Indicator instance
        """
        self.indicators[name] = indicator

    def update_all(self, value: float):
        """
        Update all indicators

        Parameters
        ----------
        value : float
            Update value
        """
        for indicator in self.indicators.values():
            indicator.update_raw(value)

    def update_indicators(self, updates: List[Tuple[str, float]]):
        """
        Update specific indicators

        Parameters
        ----------
        updates : List[Tuple[str, float]]
            List of (indicator name, value)
        """
        for name, value in updates:
            if name in self.indicators:
                self.indicators[name].update_raw(value)

    def get_values(self) -> dict:
        """Get values of all indicators"""
        values = {}
        for name, indicator in self.indicators.items():
            if indicator.initialized:
                values[name] = indicator.value
        return values

    def reset_all(self):
        """Reset all indicators"""
        for indicator in self.indicators.values():
            indicator.reset()

    def get_initialized_indicators(self) -> List[str]:
        """Get list of initialized indicators"""
        return [
            name for name, indicator in self.indicators.items()
            if indicator.initialized
        ]


# Performance analysis tool
class IndicatorProfiler:
    """Indicator performance profiler"""

    def __init__(self):
        """Initialize profiler"""
        self.timings = {}
        self.call_counts = {}

    def profile_update(self, indicator_name: str, func, *args, **kwargs):
        """
        Profile indicator update

        Parameters
        ----------
        indicator_name : str
            Indicator name
        func : Callable
            Update function
        *args, **kwargs : Any
            Function arguments
        """
        import time

        start_time = time.perf_counter()
        result = func(*args, **kwargs)
        end_time = time.perf_counter()

        elapsed = (end_time - start_time) * 1000  # Convert to milliseconds

        # Record timing
        if indicator_name not in self.timings:
            self.timings[indicator_name] = []
        self.timings[indicator_name].append(elapsed)

        # Record call count
        self.call_counts[indicator_name] = self.call_counts.get(indicator_name, 0) + 1

        return result

    def get_stats(self) -> dict:
        """Get performance statistics"""
        stats = {}

        for name, times in self.timings.items():
            count = self.call_counts.get(name, 0)
            if count > 0:
                stats[name] = {
                    'calls': count,
                    'total_ms': sum(times),
                    'avg_ms': np.mean(times),
                    'max_ms': max(times),
                    'min_ms': min(times),
                    'std_ms': np.std(times),
                }

        return stats

    def find_slow_indicators(self, threshold_ms: float = 1.0) -> List[str]:
        """Find slow indicators"""
        slow_indicators = []
        stats = self.get_stats()

        for name, stat in stats.items():
            if stat['avg_ms'] > threshold_ms:
                slow_indicators.append(name)

        return slow_indicators


# Usage example
def main():
    """Performance optimization example"""
    from time import time

    # Create indicator manager
    manager = IndicatorManager()

    # Add indicators
    manager.add_indicator("EMA_20", OptimizedEMA(period=20, batch_size=100))
    manager.add_indicator("RSI_14", RSI(period=14))
    manager.add_indicator("AdaptiveRSI", AdaptiveRSI(base_period=14))

    # Create performance profiler
    profiler = IndicatorProfiler()

    # Simulate data updates
    data = [100 + i * 0.1 + (i % 10) for i in range(1000)]

    # Test performance
    start_time = time()
    for i, value in enumerate(data):
        # Use profiler to wrap updates
        for name in ["EMA_20", "RSI_14", "AdaptiveRSI"]:
            indicator = manager.indicators[name]
            profiler.profile_update(name, indicator.update_raw, value)

    end_time = time()

    # Output results
    print(f"Updating {len(data)} data points took: {end_time - start_time:.3f} seconds")

    # Performance statistics
    stats = profiler.get_stats()
    print("\nIndicator performance statistics:")
    for name, stat in stats.items():
        print(f"{name}:")
        print(f"  Calls: {stat['calls']}")
        print(f"  Avg time: {stat['avg_ms']:.3f} ms")
        print(f"  Max time: {stat['max_ms']:.3f} ms")

    # Find slow indicators
    slow_indicators = profiler.find_slow_indicators()
    if slow_indicators:
        print(f"\nSlow indicators (>1ms): {slow_indicators}")
    else:
        print("\nAll indicators perform well")


if __name__ == "__main__":
    main()
```

## 7.7 Next Steps

In this chapter, we learned:

1. NautilusTrader's technical indicator system architecture
2. How to use built-in indicators
3. Applications of moving averages, momentum, and trend indicators
4. Custom indicator development
5. Performance optimization techniques

In the next chapter, we will learn:

1. Order management system
2. Usage of various order types
3. Order state management
4. Execution algorithms
5. Risk control

## 7.8 Summary

### Key Points
1. NautilusTrader provides a rich built-in indicator library
2. The indicator system adopts a unified interface design
3. Supports custom indicators and composite indicators
4. Provides performance optimization tools and batch processing capabilities

### Best Practices
1. Choose indicator periods reasonably to avoid over-optimization
2. Use an indicator manager to centrally manage multiple indicators
3. Regularly analyze indicator performance to identify bottlenecks
4. Combine multiple indicators for signal confirmation

## 7.9 References

- [Technical Indicators API Documentation](https://nautilustrader.io/docs/latest/api_reference/indicators/)
- [Common Quantitative Trading Indicators](https://www.investopedia.com/terms/t/technicalindicator.asp)
- [Python Performance Optimization Guide](https://docs.python.org/3/library/profile.html)