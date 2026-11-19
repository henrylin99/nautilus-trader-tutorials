---
title: "Chapter 3: Quick Start"
---

# Chapter 3: Quick Start

## Learning Objectives

By the end of this chapter, you will:
- Understand the core concepts and terminology of NautilusTrader
- Run your first simple strategy example
- Master the basic workflow
- Learn to view and analyze backtest results
- Understand the differences and use cases of the two API levels

## 3.1 Core Concepts

### 3.1.1 Basic Terminology

NautilusTrader uses a precise set of terminology to describe the various components of a trading system:

#### Data Types
- **QuoteTick**: Quote data (bid/ask prices)
- **TradeTick**: Trade data
- **Bar**: K-line data (OHLCV)
- **OrderBookDelta**: Order book incremental updates
- **OrderBookSnapshot**: Order book snapshot

#### Trading Objects
- **Instrument**: Trading instrument (e.g., BTCUSDT)
- **Order**: Order instruction
- **Position**: Position
- **Account**: Account
- **Venue**: Trading venue (e.g., BINANCE)

#### Strategy Components
- **Strategy**: Trading strategy
- **Actor**: Independently executing component
- **Clock**: Clock management
- **Cache**: Data cache

### 3.1.2 API Levels

NautilusTrader provides two main API levels:

#### High-Level API (Recommended)
Using `BacktestNode` and `TradingNode`, suitable for most users:

```python
# Advantages:
# - Simple and easy to use
# - Configuration driven
# - Easy to switch from backtest to live trading
# - Production ready

from nautilus_trader.config import BacktestRunConfig
from nautilus_trader.test_kit.providers import TestInstrumentProvider

# Configuration-driven approach
config = BacktestRunConfig(
    venues=[...],
    instruments=[...],
    data=[...],
    strategies=[...],
)

# Run backtest
from nautilus_trader.live.runner import BacktestRunner
result = BacktestRunner.run(config)
```

#### Low-Level API
Using `BacktestEngine`, suitable for library developers:

```python
# Advantages:
# - More flexible
# - Fine-grained control
# - Suitable for research and experimentation
# - Highly customizable

from nautilus_trader.backtest.engine import BacktestEngine
from nautilus_trader.config import BacktestEngineConfig

# Programmatic configuration
engine = BacktestEngine(config=BacktestEngineConfig(...))
engine.add_venue(...)
engine.add_instrument(...)
engine.add_strategy(...)
engine.run()
```

**Recommendation**: Beginners should start with the High-Level API and use the Low-Level API after gaining proficiency.

## 3.2 First Strategy: Simple Counter

Let's start with the simplest strategy that counts and logs how many K-lines it processes.

### 3.2.1 Creating the Strategy File

Create `counter_strategy.py`:
```python
"""
A simple counter strategy
Counts the number of processed K-lines
"""

from decimal import Decimal
from typing import Optional

from nautilus_trader.trading.strategy import Strategy
from nautilus_trader.model.data import Bar
from nautilus_trader.model.data import BarType
from nautilus_trader.common.enums import LogColor


class CounterStrategy(Strategy):
    """
    Simple counter strategy

    This strategy doesn't perform any trading, it just:
    1. Subscribes to K-line data
    2. Counts the number of processed K-lines
    3. Periodically outputs statistics
    """

    def __init__(self, bar_type: BarType):
        """
        Initialize the strategy

        Parameters
        ----------
        bar_type : BarType
            The K-line type to subscribe to
        """
        super().__init__()

        self.bar_type = bar_type
        self.bar_count = 0
        self.last_price: Optional[Decimal] = None
        self.high: Optional[Decimal] = None
        self.low: Optional[Decimal] = None

        # Strategy configuration
        self.log_interval = 100  # Log every 100 K-lines

    def on_start(self) -> None:
        """
        Called when the strategy starts

        Perform initialization work here, such as subscribing to data
        """
        self.log.info(f"Strategy started: {self.bar_type}")
        self.log.info(f"Logging statistics every {self.log_interval} K-lines")

        # Subscribe to K-line data
        self.subscribe_bars(self.bar_type)

        self.log.info("Subscribed to K-line data")

    def on_bar(self, bar: Bar) -> None:
        """
        Process each K-line

        Parameters
        ----------
        bar : Bar
            The arrived K-line data
        """
        self.bar_count += 1

        # Update statistics
        self.last_price = bar.close
        if self.high is None or bar.high > self.high:
            self.high = bar.high
        if self.low is None or bar.low < self.low:
            self.low = bar.low

        # Periodically output statistics
        if self.bar_count % self.log_interval == 0:
            self._log_statistics()

    def _log_statistics(self) -> None:
        """Output statistics"""
        self.log.info(
            f"Processed {self.bar_count} K-lines | "
            f"Current price: {self.last_price} | "
            f"Highest price: {self.high} | "
            f"Lowest price: {self.low}",
            color=LogColor.YELLOW
        )

    def on_stop(self) -> None:
        """
        Called when the strategy stops

        Perform cleanup work here, such as outputting final statistics
        """
        self.log.info(f"Strategy stopped")
        self.log.info(f"Total processed K-lines: {self.bar_count}")

        if self.bar_count > 0:
            self._log_statistics()
```

### 3.2.2 Running Backtest with High-Level API

Create `run_counter_high_level.py`:
```python
"""
Run the counter strategy backtest using High-Level API
"""

from datetime import datetime
from decimal import Decimal
from pathlib import Path

from nautilus_trader.config import BacktestRunConfig
from nautilus_trader.config import BacktestVenueConfig
from nautilus_trader.config import ImportableStrategyConfig
from nautilus_trader.config import LoggingConfig
from nautilus_trader.model.data import BarType
from nautilus_trader.model.data import BarSpecification
from nautilus_trader.model.enums import BarAggregation
from nautilus_trader.model.enums import PriceType
from nautilus_trader.model.identifiers import InstrumentId
from nautilus_trader.model.objects import Money
from nautilus_trader.model.currencies import USDT
from nautilus_trader.test_kit.providers import TestInstrumentProvider
from nautilus_trader.persistence.wranglers import BarDataWrangler

from counter_strategy import CounterStrategy


def main():
    """Main function"""

    # 1. Configure logging
    logging_config = LoggingConfig(
        log_level="INFO",
        log_colors=True,
        print_to_stdout=True,
    )

    # 2. Create test trading instrument (BTC/USDT)
    btc_usdt = TestInstrumentProvider.btcusdt_binance()

    # 3. Define K-line type (1-minute K-lines)
    bar_type = BarType(
        instrument_id=btc_usdt.id,
        bar_spec=BarSpecification(
            step=1,
            aggregation=BarAggregation.MINUTE,
            price_type=PriceType.LAST,
        ),
        aggregation_source="BINANCE",
    )

    # 4. Generate test data
    print("Generating test data...")
    wrangler = BarDataWrangler(bar_type=bar_type, instrument=btc_usdt)

    # Create some simulated K-line data
    bars = []
    base_price = Decimal("50000")
    for i in range(500):
        # Simple random price changes
        price_change = Decimal("100") * (i % 10 - 5) / 10
        close = base_price + price_change

        bar = wrangler.process_bar(
            ts_event=datetime.utcnow().timestamp() * 1e9 + i * 60 * 1e9,
            open=close,
            high=close + Decimal("50"),
            low=close - Decimal("50"),
            close=close,
            volume=Decimal("100"),
        )
        bars.append(bar)

    print(f"Generated {len(bars)} 1-minute K-line data")

    # 5. Configure backtest
    config = BacktestRunConfig(
        logging=logging_config,

        # Configure exchange
        venues=[
            BacktestVenueConfig(
                name="BINANCE",
                oms_type="HEDGING",  # Hedging mode
                account_type="MARGIN",  # Margin account
                base_currency=USDT,
                starting_balances=[Money(100_000, USDT)],
            ),
        ],

        # Configure data
        data={
            "catalog_path": None,  # Don't use data catalog
            "bar_types": [bar_type],
            "bars": bars,
        },

        # Configure strategy
        strategies=[
            ImportableStrategyConfig(
                strategy_path=Path(__file__).parent / "counter_strategy.py",
                config_path=None,  # Use default configuration
                config={
                    "bar_type": str(bar_type),
                },
            ),
        ],
    )

    # 6. Run backtest
    print("\nRunning backtest...")
    from nautilus_trader.live.runner import BacktestRunner

    result = BacktestRunner.run(config)

    # 7. Output results
    print("\nBacktest completed!")
    print(f"Processing time: {result.run_duration_seconds:.2f} seconds")
    print(f"Total K-lines: {result.total_events}")

    # View trading statistics (though this strategy has no trades)
    stats = result.stats_pnls()
    print("\nPnL Statistics:")
    for key, value in stats.items():
        print(f"  {key}: {value}")


if __name__ == "__main__":
    main()
```

### 3.2.3 Running Backtest with Low-Level API

Create `run_counter_low_level.py`:
```python
"""
Run the counter strategy backtest using Low-Level API
"""

from datetime import datetime
from decimal import Decimal

from nautilus_trader.backtest.engine import BacktestEngine
from nautilus_trader.config import BacktestEngineConfig
from nautilus_trader.config import LoggingConfig
from nautilus_trader.model.data import BarType
from nautilus_trader.model.data import BarSpecification
from nautilus_trader.model.enums import BarAggregation
from nautilus_trader.model.enums import PriceType
from nautilus_trader.model.enums import OmsType
from nautilus_trader.model.enums import AccountType
from nautilus_trader.model.identifiers import TraderId
from nautilus_trader.model.identifiers import Venue
from nautilus_trader.model.objects import Money
from nautilus_trader.model.currencies import USDT
from nautilus_trader.test_kit.providers import TestInstrumentProvider
from nautilus_trader.persistence.wranglers import BarDataWrangler

from counter_strategy import CounterStrategy


def main():
    """Main function"""

    # 1. Configure backtest engine
    config = BacktestEngineConfig(
        trader_id=TraderId("BACKTESTER-001"),
        logging=LoggingConfig(
            log_level="INFO",
            log_colors=True,
        ),
    )

    # Create backtest engine
    engine = BacktestEngine(config=config)

    # 2. Add venue
    BINANCE = Venue("BINANCE")
    engine.add_venue(
        venue=BINANCE,
        oms_type=OmsType.HEDGING,  # Hedging mode
        account_type=AccountType.MARGIN,  # Margin account
        base_currency=USDT,
        starting_balances=[Money(100_000, USDT)],
    )

    # 3. Create and add trading instrument
    btc_usdt = TestInstrumentProvider.btcusdt_binance()
    engine.add_instrument(btc_usdt)

    # 4. Define K-line type
    bar_type = BarType(
        instrument_id=btc_usdt.id,
        bar_spec=BarSpecification(
            step=1,
            aggregation=BarAggregation.MINUTE,
            price_type=PriceType.LAST,
        ),
        aggregation_source="BINANCE",
    )

    # 5. Generate test data
    print("Generating test data...")
    wrangler = BarDataWrangler(bar_type=bar_type, instrument=btc_usdt)

    bars = []
    base_price = Decimal("50000")
    for i in range(500):
        price_change = Decimal("100") * (i % 10 - 5) / 10
        close = base_price + price_change

        bar = wrangler.process_bar(
            ts_event=datetime.utcnow().timestamp() * 1e9 + i * 60 * 1e9,
            open=close,
            high=close + Decimal("50"),
            low=close - Decimal("50"),
            close=close,
            volume=Decimal("100"),
        )
        bars.append(bar)

    print(f"Generated {len(bars)} 1-minute K-line data")

    # 6. Add data to engine
    engine.add_data(bars)

    # 7. Create and add strategy
    strategy = CounterStrategy(bar_type=bar_type)
    engine.add_strategy(strategy)

    # 8. Run backtest
    print("\nRunning backtest...")
    start_time = datetime.now()

    engine.run()

    end_time = datetime.now()
    duration = end_time - start_time

    # 9. Get results
    result = engine.get_result()

    # 10. Output results
    print("\nBacktest completed!")
    print(f"Processing time: {duration.total_seconds():.2f} seconds")

    # Clean up resources
    engine.dispose()
    print("\nResources cleaned up")


if __name__ == "__main__":
    main()
```

### 3.2.4 Running the Example

```bash
# Make sure virtual environment is activated
source .venv/bin/activate

# Run High-Level API version
python run_counter_high_level.py

# Or run Low-Level API version
python run_counter_low_level.py
```

### 3.2.5 Expected Output

```
Generating test data...
Generated 500 1-minute K-line data

Running backtest...

Backtest completed!
Processing time: 0.15 seconds
Total K-lines: 500

PnL Statistics:
  Total PnL: 0 USDT
  Realized PnL: 0 USDT
  Unrealized PnL: 0 USDT
```

## 3.3 Second Strategy: Simple EMA Crossover

Let's create a more practical strategy based on Exponential Moving Average (EMA) crossover.

### 3.3.1 Creating the EMA Crossover Strategy

Create `ema_cross_strategy.py`:
```python
"""
EMA Crossover Strategy

Buy when fast EMA crosses above slow EMA
Sell when fast EMA crosses below slow EMA
"""

from decimal import Decimal
from typing import Optional

from nautilus_trader.trading.strategy import Strategy
from nautilus_trader.model.data import Bar
from nautilus_trader.model.data import BarType
from nautilus_trader.model.enums import OrderSide
from nautilus_trader.model.enums import OrderType
from nautilus_trader.model.events import OrderFilled
from nautilus_trader.model.orders import MarketOrder
from nautilus_trader.model.identifiers import InstrumentId
from nautilus_trader.indicators.average.ema import EMA
from nautilus_trader.common.enums import LogColor


class EMACrossStrategy(Strategy):
    """
    EMA Crossover Strategy

    Uses two exponential moving averages:
    - Fast EMA (10 period)
    - Slow EMA (30 period)

    Trading logic:
    - Buy on golden cross (fast line crosses above slow line)
    - Sell on death cross (fast line crosses below slow line)
    """

    def __init__(
        self,
        bar_type: BarType,
        fast_ema_period: int = 10,
        slow_ema_period: int = 30,
        trade_size: Decimal = Decimal("0.01"),
    ):
        """
        Initialize the strategy

        Parameters
        ----------
        bar_type : BarType
            The K-line type to subscribe to
        fast_ema_period : int, default 10
            Fast EMA period
        slow_ema_period : int, default 30
            Slow EMA period
        trade_size : Decimal, default "0.01"
            Trade size
        """
        super().__init__()

        self.bar_type = bar_type
        self.fast_ema_period = fast_ema_period
        self.slow_ema_period = slow_ema_period
        self.trade_size = trade_size

        # Create EMA indicators
        self.fast_ema = EMA(self.fast_ema_period)
        self.slow_ema = EMA(self.slow_ema_period)

        # State tracking
        self.in_position = False
        self.entry_price: Optional[Decimal] = None
        self.last_signal: Optional[str] = None

        # Statistics
        self.total_trades = 0
        self.winning_trades = 0
        self.losing_trades = 0

        # Logging
        self.log_enabled = True

    def on_start(self) -> None:
        """Called when strategy starts"""
        self.log.info(
            f"EMA Crossover Strategy Started - "
            f"Fast EMA: {self.fast_ema_period}, "
            f"Slow EMA: {self.slow_ema_period}, "
            f"Trade Size: {self.trade_size}"
        )

        # Subscribe to K-line data
        self.subscribe_bars(self.bar_type)

        # Subscribe to account events (trade confirmations)
        self.subscribe_account_events()

    def on_bar(self, bar: Bar) -> None:
        """Process each K-line"""
        # Update EMAs
        self.fast_ema.update_raw(bar.close)
        self.slow_ema.update_raw(bar.close)

        # Wait for both EMAs to have sufficient data
        if not self.fast_ema.initialized or not self.slow_ema.initialized:
            return

        # Check for crossover signals
        self._check_crossover(bar)

        # Output current status
        if self.log_enabled and bar.ts_event.minute % 5 == 0:  # Log every 5 minutes
            self._log_status(bar)

    def _check_crossover(self, bar: Bar) -> None:
        """Check for EMA crossover"""
        fast_value = self.fast_ema.value
        slow_value = self.slow_ema.value
        fast_prev = self.fast_ema.value_1
        slow_prev = self.slow_ema.value_1

        # Golden cross: fast line crosses above slow line
        if (fast_value > slow_value and
            fast_prev <= slow_prev and
            not self.in_position):

            self._enter_long(bar)

        # Death cross: fast line crosses below slow line
        elif (fast_value < slow_value and
              fast_prev >= slow_prev and
              self.in_position):

            self._exit_long(bar)

    def _enter_long(self, bar: Bar) -> None:
        """Enter long position"""
        # Create market buy order
        order = MarketOrder(
            trader_id=self.trader_id,
            strategy_id=self.id,
            instrument_id=self.bar_type.instrument_id,
            order_side=OrderSide.BUY,
            quantity=self.trade_size,
            tags="EMA_CROSS_ENTRY",
        )

        # Submit order
        self.submit_order(order)

        # Record signal
        self.last_signal = "BUY"
        self.log.info(
            f"Buy signal - Price: {bar.close}, "
            f"Fast EMA: {self.fast_ema.value:.2f}, "
            f"Slow EMA: {self.slow_ema.value:.2f}",
            color=LogColor.GREEN,
        )

    def _exit_long(self, bar: Bar) -> None:
        """Exit long position"""
        # Create market sell order
        order = MarketOrder(
            trader_id=self.trader_id,
            strategy_id=self.id,
            instrument_id=self.bar_type.instrument_id,
            order_side=OrderSide.SELL,
            quantity=self.trade_size,
            tags="EMA_CROSS_EXIT",
        )

        # Submit order
        self.submit_order(order)

        # Record signal
        self.last_signal = "SELL"
        self.log.info(
            f"Sell signal - Price: {bar.close}, "
            f"Fast EMA: {self.fast_ema.value:.2f}, "
            f"Slow EMA: {self.slow_ema.value:.2f}",
            color=LogColor.RED,
        )

    def on_order_filled(self, event: OrderFilled) -> None:
        """Handle order fill"""
        self.total_trades += 1

        if event.order_side == OrderSide.BUY:
            self.in_position = True
            self.entry_price = event.last_px
            self.log.info(
                f"Buy filled - Quantity: {event.last_qty}, "
                f"Price: {event.last_px}, "
                f"Commission: {event.commission}"
            )
        else:
            self.in_position = False
            if self.entry_price:
                pnl = (event.last_px - self.entry_price) * event.last_qty
                if pnl > 0:
                    self.winning_trades += 1
                else:
                    self.losing_trades += 1
                self.log.info(
                    f"Sell filled - Quantity: {event.last_qty}, "
                    f"Price: {event.last_px}, "
                    f"PnL: {pnl:.2f}, "
                    f"Commission: {event.commission}"
                )

    def _log_status(self, bar: Bar) -> None:
        """Output current status"""
        position_str = "In position" if self.in_position else "No position"
        self.log.info(
            f"[{bar.ts_event.strftime('%H:%M')}] "
            f"Price: {bar.close:.2f} | "
            f"Fast EMA: {self.fast_ema.value:.2f} | "
            f"Slow EMA: {self.slow_ema.value:.2f} | "
            f"Status: {position_str}",
            color=LogColor.BLUE,
        )

    def on_stop(self) -> None:
        """Called when strategy stops"""
        self.log.info("Strategy stopped")
        self.log.info(f"Total trades: {self.total_trades}")
        self.log.info(f"Winning trades: {self.winning_trades}")
        self.log.info(f"Losing trades: {self.losing_trades}")

        if self.total_trades > 0:
            win_rate = self.winning_trades / self.total_trades * 100
            self.log.info(f"Win rate: {win_rate:.2f}%")
```

### 3.3.2 Running the EMA Crossover Strategy

Create `run_ema_cross.py`:
```python
"""
Run EMA Crossover Strategy Backtest
"""

from datetime import datetime, timedelta
from decimal import Decimal
from pathlib import Path

from nautilus_trader.config import BacktestRunConfig
from nautilus_trader.config import BacktestVenueConfig
from nautilus_trader.config import ImportableStrategyConfig
from nautilus_trader.config import LoggingConfig
from nautilus_trader.model.data import BarType
from nautilus_trader.model.data import BarSpecification
from nautilus_trader.model.enums import BarAggregation
from nautilus_trader.model.enums import PriceType
from nautilus_trader.model.objects import Money
from nautilus_trader.model.currencies import USDT
from nautilus_trader.test_kit.providers import TestInstrumentProvider
from nautilus_trader.persistence.wranglers import BarDataWrangler


def generate_trend_data(bars_needed: int = 1000):
    """Generate test data with trends"""

    # Create BTC/USDT trading instrument
    btc_usdt = TestInstrumentProvider.btcusdt_binance()

    # Define K-line type
    bar_type = BarType(
        instrument_id=btc_usdt.id,
        bar_spec=BarSpecification(
            step=1,
            aggregation=BarAggregation.MINUTE,
            price_type=PriceType.LAST,
        ),
        aggregation_source="BINANCE",
    )

    # Create data processor
    wrangler = BarDataWrangler(bar_type=bar_type, instrument=btc_usdt)

    # Generate data
    bars = []
    base_price = Decimal("50000")
    trend = Decimal("0")  # 0 = sideways, 1 = up, -1 = down
    trend_duration = 0

    for i in range(bars_needed):
        # Randomly change trend
        if trend_duration <= 0:
            trend = Decimal(str([ -1, 0, 1 ][i % 3]))  # Cycle: down, sideways, up
            trend_duration = 100 + (i % 100)  # Random period of 100-200

        # Calculate price change
        if trend != 0:
            price_change = trend * Decimal("50") + Decimal(str((i % 20 - 10) * 2))
        else:
            price_change = Decimal(str((i % 20 - 10) * 5))

        # Update price
        base_price += price_change
        if base_price < Decimal("45000"):
            base_price = Decimal("45000")
        elif base_price > Decimal("55000"):
            base_price = Decimal("55000")

        # Generate OHLCV
        open_price = base_price - Decimal("20")
        high_price = base_price + Decimal("50")
        low_price = base_price - Decimal("70")
        close_price = base_price + Decimal("10")
        volume = Decimal("100") + Decimal(str(i % 50))

        # Create K-line
        bar = wrangler.process_bar(
            ts_event=(datetime.utcnow() - timedelta(minutes=bars_needed-i)).timestamp() * 1e9,
            open=open_price,
            high=high_price,
            low=low_price,
            close=close_price,
            volume=volume,
        )
        bars.append(bar)

        trend_duration -= 1

    return bars, bar_type, btc_usdt


def main():
    """Main function"""

    print("Generating test data...")
    bars, bar_type, btc_usdt = generate_trend_data(1440)  # 1 day of 1-minute K-lines
    print(f"Generated {len(bars)} K-line data")

    # Configure backtest
    config = BacktestRunConfig(
        logging=LoggingConfig(
            log_level="WARNING",  # Reduce log output
            log_colors=True,
        ),

        # Configure exchange
        venues=[
            BacktestVenueConfig(
                name="BINANCE",
                oms_type="HEDGING",
                account_type="MARGIN",
                base_currency=USDT,
                starting_balances=[Money(10_000, USDT)],
            ),
        ],

        # Configure data
        data={
            "catalog_path": None,
            "bar_types": [bar_type],
            "bars": bars,
        },

        # Configure strategy
        strategies=[
            ImportableStrategyConfig(
                strategy_path=Path(__file__).parent / "ema_cross_strategy.py",
                config={
                    "bar_type": str(bar_type),
                    "fast_ema_period": 10,
                    "slow_ema_period": 30,
                    "trade_size": "0.01",
                },
            ),
        ],
    )

    # Run backtest
    print("\nRunning backtest...")
    from nautilus_trader.live.runner import BacktestRunner

    result = BacktestRunner.run(config)

    # Output results
    print("\nBacktest completed!")
    print(f"Processing time: {result.run_duration_seconds:.2f} seconds")

    # View detailed statistics
    stats = result.stats_pnls()
    print("\n=== Trading Statistics ===")
    print(f"Total trades: {result.stats_returns_count()}")

    if stats:
        for key, value in stats.items():
            if isinstance(value, (int, float, Decimal)):
                print(f"{key}: {value}")
            else:
                print(f"{key}: {value}")

    # View position information
    print("\n=== Position Information ===")
    for position in result.positions_closed:
        print(f"Closed: {position}")

    if result.positions_open:
        for position in result.positions_open:
            print(f"Open: {position}")


if __name__ == "__main__":
    main()
```

### 3.3.3 Running and Viewing Results

```bash
python run_ema_cross.py
```

Expected output example:
```
Generating test data...
Generated 1440 K-line data

Running backtest...

Backtest completed!
Processing time: 0.45 seconds

=== Trading Statistics ===
Total trades: 20
Total PnL: 45.23 USDT
Realized PnL: 45.23 USDT
Unrealized PnL: 0 USDT

=== Position Information ===
Closed: [Position(...)]
Closed: [Position(...)]
...
```

## 3.4 Understanding Backtest Results

### 3.4.1 Basic Statistical Indicators

- **Total PnL**: Total profit and loss (including realized and unrealized)
- **Realized PnL**: Realized profit and loss
- **Unrealized PnL**: Unrealized profit and loss
- **Max Drawdown**: Maximum drawdown
- **Sharpe Ratio**: Sharpe ratio
- **Win Rate**: Win rate

### 3.4.2 Analyzing Backtest Reports

NautilusTrader provides rich analysis tools. In later chapters, we will detail how to generate detailed analysis reports, including:

- Trade analysis
- Performance attribution
- Risk metrics
- Backtest logs

## 3.5 Best Practices

### 3.5.1 Strategy Development Principles

1. **Keep it simple**: Start with simple logic
2. **Test thoroughly**: Test under various market conditions
3. **Risk management**: Always include risk controls
4. **Logging**: Record key decision points
5. **Parameterization**: Use configuration parameters instead of hardcoding

### 3.5.2 Debugging Tips

1. **Use logging**:
   ```python
   self.log.info("Debug information", color=LogColor.YELLOW)
   ```

2. **Output data**:
   ```python
   print(f"Current price: {bar.close}")
   ```

3. **Conditional breakpoints**:
   ```python
   if bar.ts_event.hour == 9:
       import pdb; pdb.set_trace()
   ```

### 3.5.3 Performance Considerations

1. **Avoid repeated calculations**: Cache calculation results
2. **Use appropriate data structures**: Choose suitable data types
3. **Limit log output**: Avoid excessive logging affecting performance

## 3.6 Next Steps

Congratulations! You have successfully run your first NautilusTrader strategy. In the next chapter, we will learn:

1. How to load real historical data
2. Supported data formats
3. Data preprocessing and cleaning
4. Efficient data storage solutions

## 3.7 Summary

This chapter introduced the basic concepts and usage of NautilusTrader:

### Key Points
1. NautilusTrader uses precise terminology to define various concepts
2. Provides both High-Level and Low-Level APIs
3. Event-driven architecture makes strategy development intuitive
4. Same code can be used for backtesting and live trading

### Key Skills
- Understanding core concepts and terminology
- Creating simple strategy classes
- Configuring and running backtests
- Analyzing basic results

## 3.8 References

- [Strategy Development Guide](https://nautilustrader.io/docs/latest/concepts/strategies/)
- [Event System Documentation](https://nautilustrader.io/docs/latest/concepts/message_bus/)
- [BacktestRunner API](https://nautilustrader.io/docs/latest/api_reference/runner/)
- [Technical Indicators List](https://nautilustrader.io/docs/latest/api_reference/indicators/)