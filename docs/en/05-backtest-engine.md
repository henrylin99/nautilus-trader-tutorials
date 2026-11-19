---
title: "Chapter 5: Basic Backtest Engine Usage"
---

# Chapter 5: Basic Backtest Engine Usage

## Learning Objectives

Through this chapter, you will learn:
- Master core configuration of BacktestEngine
- Understand exchange and account configuration
- Learn to add trading instruments and order types
- Run complete backtest workflows
- Analyze and understand backtest results

## 5.1 BacktestEngine Overview

BacktestEngine is the core backtesting component of NautilusTrader, providing a highly configurable backtesting environment.

### 5.1.1 Architecture Diagram

```
┌─────────────────────────────────────────┐
│               BacktestEngine            │
├─────────────────────────────────────────┤
│  - Configuration Management             │
│  - Time Management                      │
│  - Event Scheduling                     │
│  - State Management                     │
└─────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────┐
│              Core Components             │
├─────────────────────────────────────────┤
│  • ExecutionEngine (Execution Engine)   │
│  • DataEngine (Data Engine)             │
│  • RiskEngine (Risk Engine)             │
│  • Portfolio (Portfolio)                │
│  • Cache (Cache)                        │
│  • MessageBus (Message Bus)             │
└─────────────────────────────────────────┘
```

### 5.1.2 Main Features

1. **Simulated Exchange**: Simulates real exchange matching mechanisms
2. **Order Management**: Supports multiple order types and execution algorithms
3. **Fee Calculation**: Accurate calculation of fees and slippage
4. **Risk Control**: Real-time risk checks and limits
5. **Time Management**: Supports different time modes and accelerated backtesting

## 5.2 Basic Configuration

### 5.2.1 BacktestEngineConfig

```python
"""
BacktestEngine Configuration Example
"""

from nautilus_trader.config import BacktestEngineConfig, LoggingConfig
from nautilus_trader.model.identifiers import TraderId


def create_engine_config():
    """Create engine configuration"""
    config = BacktestEngineConfig(
        # Trader identifier
        trader_id=TraderId("BACKTESTER-001"),

        # Logging configuration
        logging=LoggingConfig(
            log_level="INFO",        # Log level
            log_colors=True,         # Colored logs
            print_to_stdout=True,    # Output to console
            filter_messages=False,   # Do not filter messages
        ),

        # Time configuration
        time_increment_ns=1_000_000_000,  # 1-second time increment (nanoseconds)
        bar_build_interval_ms=100,         # Bar building interval (milliseconds)

        # Other options
        debug=False,                        # Debug mode
        debug_messages=False,               # Debug messages
        benchmark_fees=False,               # Benchmark fees

        # Conflict resolution modes
        fill_at_knowledgeable=False,        # Fill when not knowledgeable
        reject_terminal_orders=False,      # Reject terminal orders
        use_position_ids=False,             # Use position IDs
    )

    return config
```

### 5.2.2 Creating and Configuring the Engine

```python
"""
Complete Engine Configuration Example
"""

from decimal import Decimal
from nautilus_trader.backtest.engine import BacktestEngine
from nautilus_trader.model.enums import OmsType, AccountType
from nautilus_trader.model.identifiers import Venue
from nautilus_trader.model.objects import Money
from nautilus_trader.model.currencies import USD, BTC
from nautilus_trader.test_kit.providers import TestInstrumentProvider


def create_and_configure_engine():
    """Create and configure backtest engine"""

    # 1. Create configuration
    config = BacktestEngineConfig(
        trader_id=TraderId("BACKTESTER-001"),
        logging=LoggingConfig(log_level="INFO"),
    )

    # 2. Create engine
    engine = BacktestEngine(config=config)

    # 3. Add exchanges (can add multiple)
    add_venues(engine)

    # 4. Add trading instruments
    add_instruments(engine)

    return engine


def add_venues(engine: BacktestEngine):
    """Add exchange configurations"""

    # Binance configuration
    BINANCE = Venue("BINANCE")
    engine.add_venue(
        venue=BINANCE,
        # Order management system type
        oms_type=OmsType.HEDGING,  # Hedging mode (allows simultaneous long and short)
        # Account type
        account_type=AccountType.MARGIN,  # Margin account
        # Starting balance
        starting_balances=[
            Money(1_000_000, USD),  # 1 million USDT
            Money(20, BTC),         # 20 BTC
        ],
        # Base currency
        base_currency=USD,
        # Leverage configuration
        leverage_real=False,       # Do not use real leverage
        # Fee configuration (optional)
        account_id=None,           # Auto-generate
        # Other configuration
        max_order_submit_rate=None,    # Maximum order submission rate
        max_order_cancel_rate=None,    # Maximum order cancellation rate
    )

    # Add second exchange
    BYBIT = Venue("BYBIT")
    engine.add_venue(
        venue=BYBIT,
        oms_type=OmsType.NETTING,  # Netting mode (single direction)
        account_type=AccountType.CASH,  # Cash account
        starting_balances=[Money(500_000, USD)],
        base_currency=USD,
    )

    print(f"Added exchanges: {BINANCE}, {BYBIT}")


def add_instruments(engine: BacktestEngine):
    """Add trading instruments"""

    # BTC/USDT perpetual contract
    btc_usdt_perp = TestInstrumentProvider.btcusdt_perp_binance()
    engine.add_instrument(btc_usdt_perp)

    # ETH/USDT spot
    eth_usdt = TestInstrumentProvider.ethusdt_binance()
    engine.add_instrument(eth_usdt)

    # BTC/USD futures
    btc_usd = TestInstrumentProvider.btcusdt_cme()
    engine.add_instrument(btc_usd)

    print(f"Added trading instruments: {len(engine.instruments())}")


def main():
    """Main function example"""
    engine = create_and_configure_engine()
    print(f"Engine configuration complete, supports {len(engine.cache.venues())} exchanges")
    return engine


if __name__ == "__main__":
    main()
```

## 5.3 Data Management

### 5.3.1 Adding Historical Data

```python
"""
Data Management Example
"""

from datetime import datetime, timedelta
from decimal import Decimal
import pandas as pd
from pathlib import Path

from nautilus_trader.model.data import Bar, QuoteTick, TradeTick
from nautilus_trader.model.data import BarType, BarSpecification
from nautilus_trader.model.enums import BarAggregation, PriceType
from nautilus_trader.model.identifiers import InstrumentId
from nautilus_trader.persistence.wranglers import BarDataWrangler
from nautilus_trader.test_kit.providers import TestInstrumentProvider


def add_bar_data(engine: BacktestEngine):
    """Add bar data"""

    # Get trading instrument
    btc_usdt = TestInstrumentProvider.btcusdt_binance()

    # Define bar type
    bar_type = BarType(
        instrument_id=btc_usdt.id,
        bar_spec=BarSpecification(
            step=1,                        # 1 bar
            aggregation=BarAggregation.MINUTE,  # 1 minute
            price_type=PriceType.LAST,     # Last price
        ),
        aggregation_source="BINANCE",
    )

    # Load historical data (example)
    bars = load_historical_bars(bar_type, days=30)

    # Add to engine
    engine.add_data(bars)

    print(f"Added {len(bars)} {bar_type} bars")


def add_tick_data(engine: BacktestEngine):
    """Add tick data"""

    # Get trading instrument
    btc_usdt = TestInstrumentProvider.btcusdt_binance()

    # Load quote data
    quotes = load_quote_ticks(btc_usdt.id, days=1)
    engine.add_data(quotes)

    # Load trade data
    trades = load_trade_ticks(btc_usdt.id, days=1)
    engine.add_data(trades)

    print(f"Added {len(quotes)} quotes, {len(trades)} trades")


def add_mixed_data(engine: BacktestEngine):
    """Add mixed data (bars + ticks)"""

    # Add bar data as base data
    add_bar_data(engine)

    # Add tick data at key time points
    tick_events = generate_key_tick_events()
    engine.add_data(tick_events)


def load_historical_bars(bar_type: BarType, days: int = 30) -> list[Bar]:
    """Load historical bar data"""

    # Example: Generate simulated data
    # In practice, should load from database or files

    bars = []
    start_time = datetime.utcnow() - timedelta(days=days)

    for i in range(days * 24 * 60):  # Every minute
        timestamp = start_time + timedelta(minutes=i)

        # Generate simulated OHLCV
        base_price = Decimal("50000")
        random_walk = Decimal(str((i % 100 - 50) * 10))

        bar = Bar(
            bar_type=bar_type,
            open=base_price + random_walk,
            high=base_price + random_walk + Decimal("100"),
            low=base_price + random_walk - Decimal("100"),
            close=base_price + random_walk + Decimal("50"),
            volume=Decimal(str(100 + i % 50)),
            ts_event=int(timestamp.timestamp() * 1e9),  # Nanosecond timestamp
            ts_init=int(timestamp.timestamp() * 1e9),
        )

        bars.append(bar)

    return bars


def load_quote_ticks(instrument_id: InstrumentId, days: int = 1) -> list[QuoteTick]:
    """Load quote tick data"""
    quotes = []

    # Simulate quote data
    for i in range(1000):
        price = Decimal("50000") + Decimal(str(i % 100))

        quote = QuoteTick(
            instrument_id=instrument_id,
            bid_price=price - Decimal("1"),
            ask_price=price + Decimal("1"),
            bid_size=Decimal("10"),
            ask_size=Decimal("10"),
            ts_event=1640995200000000000 + i * 1000000000,  # One tick per second
            ts_init=1640995200000000000 + i * 1000000000,
        )

        quotes.append(quote)

    return quotes


def load_trade_ticks(instrument_id: InstrumentId, days: int = 1) -> list[TradeTick]:
    """Load trade tick data"""
    trades = []

    # Simulate trade data
    for i in range(500):
        price = Decimal("50000") + Decimal(str(i % 100))

        trade = TradeTick(
            instrument_id=instrument_id,
            price=price,
            size=Decimal("1"),
            aggressor_side=AggressorSide.BUYER if i % 2 == 0 else AggressorSide.SELLER,
            trade_id=str(i),
            ts_event=1640995200000000000 + i * 2000000000,  # One trade every 2 seconds
            ts_init=1640995200000000000 + i * 2000000000,
        )

        trades.append(trade)

    return trades
```

### 5.3.2 Data Serialization

```python
"""
Data Serialization and Persistence Example
"""

import pickle
import json
from pathlib import Path
from typing import Dict, Any

from nautilus_trader.serialization.serializert import Serializer


class DataSerializer:
    """Data serialization utility"""

    def __init__(self):
        """Initialize serializer"""
        self.serializer = Serializer()

    def serialize_to_pickle(self, data: list, file_path: Path):
        """Serialize to Pickle file"""
        with open(file_path, 'wb') as f:
            pickle.dump(data, f)

    def deserialize_from_pickle(self, file_path: Path) -> list:
        """Deserialize from Pickle file"""
        with open(file_path, 'rb') as f:
            return pickle.load(f)

    def serialize_to_json(self, bars: list[Bar], file_path: Path):
        """Serialize bars to JSON file"""
        data = []
        for bar in bars:
            data.append({
                'timestamp': bar.ts_event,
                'open': str(bar.open),
                'high': str(bar.high),
                'low': str(bar.low),
                'close': str(bar.close),
                'volume': str(bar.volume),
            })

        with open(file_path, 'w') as f:
            json.dump(data, f, indent=2)

    def export_to_dataframe(self, bars: list[Bar]) -> pd.DataFrame:
        """Export to DataFrame"""
        data = []
        for bar in bars:
            data.append({
                'timestamp': pd.to_datetime(bar.ts_event, unit='ns'),
                'open': float(bar.open),
                'high': float(bar.high),
                'low': float(bar.low),
                'close': float(bar.close),
                'volume': float(bar.volume),
            })

        return pd.DataFrame(data)


def main():
    """Main function example"""
    # Create serializer
    serializer = DataSerializer()

    # Load data
    bars = load_historical_bars(bar_type, days=30)

    # Serialize to file
    serializer.serialize_to_pickle(
        bars,
        Path("bars_2024_01.pkl")
    )

    serializer.serialize_to_json(
        bars,
        Path("bars_2024_01.json")
    )

    # Export to DataFrame
    df = serializer.export_to_dataframe(bars)
    df.to_csv("bars_2024_01.csv", index=False)

    print("Data serialization complete")
```

## 5.4 Strategy Addition and Management

### 5.4.1 Adding Strategies

```python
"""
Strategy Management Example
"""

from typing import List
from nautilus_trader.trading.strategy import Strategy
from nautilus_trader.model.enums import OrderSide
from nautilus_trader.model.orders import MarketOrder
from nautilus_trader.model.data import Bar
from nautilus_trader.model.data import BarType


class SimpleBuyStrategy(Strategy):
    """Simple buy strategy"""

    def __init__(self, bar_type: BarType, buy_threshold: float = 0.01):
        """
        Initialize strategy

        Parameters
        ----------
        bar_type : BarType
            Bar type
        buy_threshold : float
            Buy threshold (1%)
        """
        super().__init__()
        self.bar_type = bar_type
        self.buy_threshold = buy_threshold
        self.last_price = None
        self.bought = False

    def on_start(self):
        """Strategy start"""
        self.log.info(f"Strategy started: {self.bar_type}")
        self.subscribe_bars(self.bar_type)

    def on_bar(self, bar: Bar):
        """Handle bar"""
        current_price = float(bar.close)

        if self.last_price is None:
            self.last_price = current_price
            return

        # Calculate percentage change
        change = (current_price - self.last_price) / self.last_price

        # Buy if change exceeds threshold and not already bought
        if change > self.buy_threshold and not self.bought:
            self._buy(bar)
        elif change < -self.buy_threshold and self.bought:
            self._sell(bar)

        self.last_price = current_price

    def _buy(self, bar: Bar):
        """Execute buy"""
        order = MarketOrder(
            trader_id=self.trader_id,
            strategy_id=self.id,
            instrument_id=self.bar_type.instrument_id,
            order_side=OrderSide.BUY,
            quantity=Decimal("0.1"),
            tags="SIMPLE_BUY",
        )

        self.submit_order(order)
        self.bought = True
        self.log.info(f"Buy: {bar.close}")

    def _sell(self, bar: Bar):
        """Execute sell"""
        order = MarketOrder(
            trader_id=self.trader_id,
            strategy_id=self.id,
            instrument_id=self.bar_type.instrument_id,
            order_side=OrderSide.SELL,
            quantity=Decimal("0.1"),
            tags="SIMPLE_SELL",
        )

        self.submit_order(order)
        self.bought = False
        self.log.info(f"Sell: {bar.close}")


def add_strategies(engine: BacktestEngine):
    """Add strategies to engine"""

    # Strategy 1: BTC/USDT 1-minute
    strategy1 = SimpleBuyStrategy(
        bar_type=BarType(
            instrument_id=InstrumentId.from_str("BTCUSDT.BINANCE"),
            bar_spec=BarSpecification(
                step=1,
                aggregation=BarAggregation.MINUTE,
                price_type=PriceType.LAST,
            ),
            aggregation_source="BINANCE",
        ),
        buy_threshold=0.005,  # 0.5%
    )
    engine.add_strategy(strategy1)

    # Strategy 2: ETH/USDT 5-minute
    strategy2 = SimpleBuyStrategy(
        bar_type=BarType(
            instrument_id=InstrumentId.from_str("ETHUSDT.BINANCE"),
            bar_spec=BarSpecification(
                step=5,
                aggregation=BarAggregation.MINUTE,
                price_type=PriceType.LAST,
            ),
            aggregation_source="BINANCE",
        ),
        buy_threshold=0.003,  # 0.3%
    )
    engine.add_strategy(strategy2)

    print(f"Added 2 strategies")


class MultiAssetStrategy(Strategy):
    """Multi-asset strategy"""

    def __init__(self, bar_types: List[BarType]):
        """
        Initialize multi-asset strategy

        Parameters
        ----------
        bar_types : List[BarType]
            Multiple bar types
        """
        super().__init__()
        self.bar_types = bar_types
        self.prices = {}

    def on_start(self):
        """Strategy start"""
        self.log.info("Multi-asset strategy started")

        for bar_type in self.bar_types:
            self.subscribe_bars(bar_type)

    def on_bar(self, bar: Bar):
        """Handle bar"""
        instrument_id = str(bar.bar_type.instrument_id)
        self.prices[instrument_id] = float(bar.close)

        # If have at least two asset prices
        if len(self.prices) >= 2:
            self._check_arbitrage()

    def _check_arbitrage(self):
        """Check arbitrage opportunity"""
        # Here implement simple arbitrage logic
        pass


def add_multi_asset_strategy(engine: BacktestEngine):
    """Add multi-asset strategy"""

    bar_types = [
        BarType(
            instrument_id=InstrumentId.from_str("BTCUSDT.BINANCE"),
            bar_spec=BarSpecification(
                step=1,
                aggregation=BarAggregation.MINUTE,
                price_type=PriceType.LAST,
            ),
            aggregation_source="BINANCE",
        ),
        BarType(
            instrument_id=InstrumentId.from_str("ETHUSDT.BINANCE"),
            bar_spec=BarSpecification(
                step=1,
                aggregation=BarAggregation.MINUTE,
                price_type=PriceType.LAST,
            ),
            aggregation_source="BINANCE",
        ),
    ]

    strategy = MultiAssetStrategy(bar_types)
    engine.add_strategy(strategy)

    print(f"Added multi-asset strategy, monitoring {len(bar_types)} assets")
```

## 5.5 Running Backtests

### 5.5.1 Basic Backtest Workflow

```python
"""
Complete Backtest Execution Example
"""

from datetime import datetime, timedelta
from decimal import Decimal
import time


def run_simple_backtest():
    """Run simple backtest"""

    print("Starting backtest...")
    start_time = time.time()

    # 1. Create engine
    config = BacktestEngineConfig(
        trader_id=TraderId("BACKTESTER-001"),
        logging=LoggingConfig(log_level="INFO"),
    )
    engine = BacktestEngine(config=config)

    # 2. Add exchange
    engine.add_venue(
        venue=Venue("BINANCE"),
        oms_type=OmsType.HEDGING,
        account_type=AccountType.MARGIN,
        starting_balances=[Money(100_000, USDT)],
        base_currency=USDT,
    )

    # 3. Add trading instrument
    btc_usdt = TestInstrumentProvider.btcusdt_binance()
    engine.add_instrument(btc_usdt)

    # 4. Add data
    bar_type = BarType(
        instrument_id=btc_usdt.id,
        bar_spec=BarSpecification(
            step=1,
            aggregation=BarAggregation.MINUTE,
            price_type=PriceType.LAST,
        ),
        aggregation_source="BINANCE",
    )
    bars = load_historical_bars(bar_type, days=30)
    engine.add_data(bars)

    # 5. Add strategy
    strategy = SimpleBuyStrategy(bar_type)
    engine.add_strategy(strategy)

    # 6. Run backtest
    print("Executing backtest...")
    engine.run()

    # 7. Get results
    end_time = time.time()
    duration = end_time - start_time

    print(f"\nBacktest complete!")
    print(f"Execution time: {duration:.2f} seconds")
    print(f"Processed bars: {len(bars)}")

    # 8. Analyze results
    analyze_results(engine)

    # 9. Clean up resources
    engine.dispose()

    return engine


def run_parameter_sweep():
    """Run parameter sweep"""

    # Parameter list
    buy_thresholds = [0.003, 0.005, 0.01, 0.02]

    results = []

    for threshold in buy_thresholds:
        print(f"\nTesting parameter: buy_threshold = {threshold}")

        # Create engine
        config = BacktestEngineConfig(
            trader_id=TraderId(f"BACKTESTER-{threshold}"),
            logging=LoggingConfig(log_level="ERROR"),  # Reduce logs
        )
        engine = BacktestEngine(config=config)

        # Configure engine (simplified)
        configure_engine(engine)

        # Create strategy
        strategy = SimpleBuyStrategy(bar_type, buy_threshold=threshold)
        engine.add_strategy(strategy)

        # Run backtest
        engine.run()

        # Collect results
        result = analyze_results_brief(engine)
        result['threshold'] = threshold
        results.append(result)

        # Clean up
        engine.dispose()

    # Print comparison results
    print("\nParameter sweep results:")
    print("-" * 80)
    for result in results:
        print(f"Threshold: {result['threshold']:.3f} | "
              f"Trade count: {result['trade_count']:3d} | "
              f"Win rate: {result['win_rate']:5.1%} | "
              f"Total PnL: {result['total_pnl']:8.2f}")

    return results


def run_multi_strategy_backtest():
    """Run multi-strategy backtest"""

    print("Starting multi-strategy backtest...")

    # Create engine
    config = BacktestEngineConfig(
        trader_id=TraderId("MULTI-STRATEGY-001"),
        logging=LoggingConfig(log_level="INFO"),
    )
    engine = BacktestEngine(config=config)

    # Add multiple exchanges
    engine.add_venue(
        venue=Venue("BINANCE"),
        oms_type=OmsType.HEDGING,
        account_type=AccountType.MARGIN,
        starting_balances=[Money(100_000, USDT)],
        base_currency=USDT,
    )

    engine.add_venue(
        venue=Venue("BYBIT"),
        oms_type=OmsType.NETTING,
        account_type=AccountType.CASH,
        starting_balances=[Money(50_000, USDT)],
        base_currency=USDT,
    )

    # Add multiple trading instruments
    instruments = [
        TestInstrumentProvider.btcusdt_binance(),
        TestInstrumentProvider.ethusdt_binance(),
    ]

    for instrument in instruments:
        engine.add_instrument(instrument)

    # Add data
    for instrument in instruments:
        bar_type = BarType(
            instrument_id=instrument.id,
            bar_spec=BarSpecification(
                step=1,
                aggregation=BarAggregation.MINUTE,
                price_type=PriceType.LAST,
            ),
            aggregation_source=instrument.id.venue.value,
        )
        bars = load_historical_bars(bar_type, days=30)
        engine.add_data(bars)

    # Add multiple strategies
    strategies = [
        SimpleBuyStrategy(bar_type_1, buy_threshold=0.005),
        SimpleBuyStrategy(bar_type_2, buy_threshold=0.003),
        MultiAssetStrategy([bar_type_1, bar_type_2]),
    ]

    for strategy in strategies:
        engine.add_strategy(strategy)

    # Run backtest
    start_time = time.time()
    engine.run()
    duration = time.time() - start_time

    print(f"\nMulti-strategy backtest complete!")
    print(f"Execution time: {duration:.2f} seconds")
    print(f"Strategy count: {len(strategies)}")

    # Analyze results
    analyze_multi_strategy_results(engine)

    # Clean up
    engine.dispose()

    return engine
```

## 5.6 Result Analysis

### 5.6.1 Getting and Analyzing Results

```python
"""
Backtest Result Analysis
"""

from typing import Dict, Any
import pandas as pd


def analyze_results(engine: BacktestEngine):
    """Analyze backtest results"""

    # Get result object
    result = engine.get_result()

    # Basic statistics
    print("\n=== Basic Statistics ===")
    print(f"Runtime: {result.run_duration_seconds:.2f} seconds")
    print(f"Processed events: {result.total_events}")
    print(f"Executed commands: {result.total_commands}")

    # Trading statistics
    print("\n=== Trading Statistics ===")
    stats = result.stats_pnls()

    for key, value in stats.items():
        if isinstance(value, Decimal):
            print(f"{key}: {float(value):.2f}")
        else:
            print(f"{key}: {value}")

    # Position information
    print("\n=== Position Information ===")
    print(f"Open positions: {len(result.positions_open)}")
    print(f"Closed positions: {len(result.positions_closed)}")

    # Closed position details
    if result.positions_closed:
        print("\nClosed position details:")
        for position in result.positions_closed[:5]:  # Show only first 5
            print(f"  {position}")
            print(f"    Opened: {position.opened_timestamp}")
            print(f"    Closed: {position.closed_timestamp}")
            print(f"    PnL: {position.realized_pnl}")
            print(f"    Commission: {position.total_commission}")

    # Order statistics
    print("\n=== Order Statistics ===")
    print(f"Total orders: {len(result.orders)}")

    # Count orders by status
    order_status_counts = {}
    for order in result.orders:
        status = order.status
        order_status_counts[status] = order_status_counts.get(status, 0) + 1

    for status, count in order_status_counts.items():
        print(f"  {status}: {count}")


def analyze_results_brief(engine: BacktestEngine) -> Dict[str, Any]:
    """Brief analysis of results"""

    result = engine.get_result()
    stats = result.stats_pnls()

    # Calculate trading statistics
    trade_count = len(result.positions_closed)
    win_count = sum(1 for pos in result.positions_closed
                    if pos.realized_pnl > 0)

    return {
        'trade_count': trade_count,
        'win_rate': win_count / trade_count if trade_count > 0 else 0,
        'total_pnl': float(stats.get('Total PnL', 0)),
        'realized_pnl': float(stats.get('Realized PnL', 0)),
        'max_drawdown': float(stats.get('Max Drawdown', 0)),
        'sharpe_ratio': float(stats.get('Sharpe Ratio', 0)),
    }


def analyze_multi_strategy_results(engine: BacktestEngine):
    """Analyze multi-strategy results"""

    result = engine.get_result()

    # Get all strategies
    strategies = engine.trader.strategies()

    print("\n=== Strategy Performance ===")
    for strategy_id, strategy in strategies.items():
        # Need more detailed strategy-level analysis here
        # Due to framework limitations, we use overall statistics
        pass

    # Statistics by exchange
    print("\n=== Exchange Statistics ===")
    venue_stats = {}

    for position in result.positions_closed:
        venue = position.instrument_id.venue.value
        if venue not in venue_stats:
            venue_stats[venue] = {
                'count': 0,
                'total_pnl': 0,
                'trades': []
            }

        venue_stats[venue]['count'] += 1
        venue_stats[venue]['total_pnl'] += float(position.realized_pnl)
        venue_stats[venue]['trades'].append(float(position.realized_pnl))

    for venue, stats in venue_stats.items():
        win_rate = sum(1 for pnl in stats['trades'] if pnl > 0) / len(stats['trades'])
        print(f"{venue}:")
        print(f"  Trade count: {stats['count']}")
        print(f"  Total PnL: {stats['total_pnl']:.2f}")
        print(f"  Win rate: {win_rate:.1%}")


def export_results_to_csv(engine: BacktestEngine, filename: str):
    """Export results to CSV"""

    result = engine.get_result()

    # Export trading records
    trades_data = []

    for position in result.positions_closed:
        trades_data.append({
            'instrument': str(position.instrument_id),
            'opened': position.opened_timestamp,
            'closed': position.closed_timestamp,
            'duration': str(position.open_duration),
            'side': 'LONG' if position.is_long else 'SHORT',
            'entry_price': float(position.avg_px_open),
            'exit_price': float(position.avg_px_close),
            'quantity': float(position.quantity),
            'pnl': float(position.realized_pnl),
            'commissions': float(position.total_commission),
            'return_pct': (float(position.realized_pnl) /
                         float(position.quantities_executed) * 100),
        })

    df = pd.DataFrame(trades_data)
    df.to_csv(f"{filename}_trades.csv", index=False)

    # Export statistics summary
    stats = result.stats_pnls()
    stats_df = pd.DataFrame([stats])
    stats_df.to_csv(f"{filename}_stats.csv", index=False)

    print(f"Results exported to {filename}_*.csv")


def create_performance_report(engine: BacktestEngine) -> str:
    """Create performance report"""

    result = engine.get_result()
    stats = analyze_results_brief(engine)

    report = f"""
Backtest Performance Report
================

Basic Information
--------
Runtime: {result.run_duration_seconds:.2f} seconds
Processed events: {result.total_events:,}

Trading Performance
--------
Total trades: {stats['trade_count']}
Win rate: {stats['win_rate']:.1%}
Total PnL: {stats['total_pnl']:,.2f} USDT
Realized PnL: {stats['realized_pnl']:,.2f} USDT
Max drawdown: {stats['max_drawdown']:,.2f} USDT
Sharpe ratio: {stats['sharpe_ratio']:.2f}

Risk Metrics
--------
Profitable trades: {sum(1 for pos in result.positions_closed if pos.realized_pnl > 0)}
Losing trades: {sum(1 for pos in result.positions_closed if pos.realized_pnl < 0)}
Average PnL: {stats['realized_pnl'] / stats['trade_count'] if stats['trade_count'] > 0 else 0:.2f} USDT

Position Information
--------
Open positions: {len(result.positions_open)}
Closed positions: {len(result.positions_closed)}
"""

    return report


def main():
    """Main function example"""

    # Run single backtest
    engine = run_simple_backtest()

    # Create report
    report = create_performance_report(engine)
    print(report)

    # Export results
    export_results_to_csv(engine, "backtest_2024_01")

    # Run parameter sweep
    print("\n" + "=" * 50)
    print("Running parameter sweep...")
    results = run_parameter_sweep()

    # Find best parameters
    best_result = max(results, key=lambda x: x['total_pnl'])
    print(f"\nBest parameters: buy_threshold = {best_result['threshold']}")
    print(f"Highest PnL: {best_result['total_pnl']:,.2f} USDT")


if __name__ == "__main__":
    main()
```

## 5.7 Advanced Features

### 5.7.1 Custom Fee Model

```python
"""
Custom Fee Model
"""

from nautilus_trader.model.fees import FeeModel
from nautilus_trader.model.objects import Money
from nautilus_trader.model.objects import Price
from nautilus_trader.model.objects import Quantity
from decimal import Decimal


class CustomFeeModel(FeeModel):
    """Custom fee model"""

    def __init__(self, base_rate: Decimal = Decimal("0.001")):
        """
        Initialize fee model

        Parameters
        ----------
        base_rate : Decimal
            Base rate (0.1%)
        """
        self.base_rate = base_rate
        self.volume_discounts = {
            Decimal("1000"): Decimal("0.0008"),    # 1000+ USDT: 0.08%
            Decimal("10000"): Decimal("0.0006"),   # 10000+ USDT: 0.06%
            Decimal("100000"): Decimal("0.0004"),  # 100000+ USDT: 0.04%
        }

    def calculate(
        self,
        price: Price,
        quantity: Quantity,
        side: str,
        instrument_id: InstrumentId,
    ) -> Money:
        """
        Calculate fee

        Parameters
        ----------
        price : Price
            Price
        quantity : Quantity
            Quantity
        side : str
            Side
        instrument_id : InstrumentId
            Instrument ID

        Returns
        -------
        Money
            Fee amount
        """
        # Calculate notional value
        notional_value = price * quantity

        # Find applicable rate
        rate = self.base_rate

        for threshold, discount_rate in self.volume_discounts.items():
            if notional_value >= threshold:
                rate = discount_rate
                break

        # Calculate fee
        fee_amount = notional_value * rate

        # VIP discount (example)
        if is_vip_customer(instrument_id.venue):
            fee_amount *= Decimal("0.9")  # 10% VIP discount

        return Money(fee_amount, instrument_id.quote_currency)


def configure_custom_fees(engine: BacktestEngine):
    """Configure custom fees"""

    # Configure fee model for specific exchange
    venue = Venue("BINANCE")
    engine.add_fee_model(
        venue=venue,
        instrument_id=None,  # Apply to all instruments
        fee_model=CustomFeeModel(Decimal("0.001")),
    )

    print("Custom fee model configured")
```

### 5.7.2 Slippage Model

```python
"""
Slippage Simulation
"""

from nautilus_trader.model.slippage import SlippageModel
from random import uniform
from decimal import Decimal


class RandomSlippageModel(SlippageModel):
    """Random slippage model"""

    def __init__(self, max_slippage: Decimal = Decimal("0.001")):
        """
        Initialize slippage model

        Parameters
        ----------
        max_slippage : Decimal
            Maximum slippage (0.1%)
        """
        self.max_slippage = max_slippage

    def calculate(
        self,
        order: Order,
        fill_price: Price,
        instrument_id: InstrumentId,
    ) -> Price:
        """
        Calculate price after slippage

        Parameters
        ----------
        order : Order
            Order
        fill_price : Price
            Fill price
        instrument_id : InstrumentId
            Instrument ID

        Returns
        -------
        Price
            Price after slippage
        """
        # Generate random slippage
        slippage_rate = Decimal(str(uniform(0, float(self.max_slippage))))

        # Buy price moves up, sell price moves down
        if order.side == OrderSide.BUY:
            slippage_amount = fill_price * slippage_rate
            adjusted_price = fill_price + slippage_amount
        else:
            slippage_amount = fill_price * slippage_rate
            adjusted_price = fill_price - slippage_amount

        return Price(adjusted_price, fill_price.precision)


def configure_slippage(engine: BacktestEngine):
    """Configure slippage model"""

    # Add slippage model to exchange
    engine.add_slippage_model(
        venue=Venue("BINANCE"),
        slippage_model=RandomSlippageModel(Decimal("0.0005")),  # 0.05%
    )

    print("Slippage model configured")
```

## 5.8 Best Practices

### 5.8.1 Performance Optimization

```python
"""
Performance Optimization Tips
"""

import multiprocessing
from typing import List
from concurrent.futures import ProcessPoolExecutor


def optimized_backtest(param_set: List[Dict]) -> List[Dict]:
    """Optimized batch backtesting"""

    # Use process pool for parallel execution
    with ProcessPoolExecutor(max_workers=multiprocessing.cpu_count()) as executor:
        results = list(executor.map(run_single_backtest, param_set))

    return results


def run_single_backtest(params: Dict) -> Dict:
    """Run single backtest (for parallel execution)"""

    # Create minimal logging configuration
    config = BacktestEngineConfig(
        trader_id=TraderId(f"BACKTEST-{params['id']}"),
        logging=LoggingConfig(log_level="ERROR"),
    )

    engine = BacktestEngine(config=config)

    # Minimal configuration
    engine.add_venue(
        venue=Venue("BINANCE"),
        oms_type=OmsType.NETTING,
        account_type=AccountType.CASH,
        starting_balances=[Money(params['balance'], USDT)],
        base_currency=USDT,
    )

    # Add data and strategies
    # ...

    # Run
    engine.run()

    # Collect results
    result = analyze_results_brief(engine)
    result['params'] = params

    # Clean up
    engine.dispose()

    return result


def memory_efficient_backtest():
    """Memory-efficient backtesting"""

    config = BacktestEngineConfig(
        trader_id=TraderId("EFFICIENT-001"),
        # Limit log quantity
        logging=LoggingConfig(
            log_level="WARNING",
            log_colors=False,
            print_to_stdout=False,
        ),
        # Enable memory optimization
        debug=False,
        debug_messages=False,
    )

    engine = BacktestEngine(config=config)

    # Load data in batches
    for day in range(30):  # 30 days of data
        bars = load_bars_for_day(day)
        engine.add_data(bars)

        # Optional: periodically clear memory
        if day % 7 == 0:
            engine.cache._flush()

    return engine
```

### 5.8.2 Debugging Tips

```python
"""
Backtest Debugging Tools
"""

import logging
from datetime import datetime


class BacktestDebugger:
    """Backtest debugger"""

    def __init__(self, engine: BacktestEngine):
        """Initialize debugger"""
        self.engine = engine
        self.order_log = []
        self.fill_log = []

    def enable_verbose_logging(self):
        """Enable verbose logging"""

        # Create file handler
        file_handler = logging.FileHandler("backtest_debug.log")
        file_handler.setLevel(logging.DEBUG)

        # Add to engine logger
        engine.logger.addHandler(file_handler)
        engine.logger.setLevel(logging.DEBUG)

    def log_order_event(self, event):
        """Log order event"""
        self.order_log.append({
            'time': datetime.utcnow(),
            'type': type(event).__name__,
            'order_id': str(event.order_id),
            'data': str(event),
        })

    def analyze_order_flow(self):
        """Analyze order flow"""

        print("\n=== Order Flow Analysis ===")

        # Order state changes
        order_states = {}
        for log in self.order_log:
            order_id = log['order_id']
            if order_id not in order_states:
                order_states[order_id] = []
            order_states[order_id].append({
                'time': log['time'],
                'event': log['type'],
            })

        # Find abnormal orders
        for order_id, events in order_states.items():
            if len(events) > 10:  # Too many state changes
                print(f"Abnormal order: {order_id}")
                for event in events:
                    print(f"  {event['time']}: {event['event']}")

    def save_debug_data(self):
        """Save debug data"""

        import json
        import pandas as pd

        # Save order log
        df_orders = pd.DataFrame(self.order_log)
        df_orders.to_csv("debug_orders.csv", index=False)

        # Save fill log
        df_fills = pd.DataFrame(self.fill_log)
        df_fills.to_csv("debug_fills.csv", index=False)

        print("Debug data saved")
```

## 5.9 Next Steps

In this chapter, we learned:

1. Configuration and usage of BacktestEngine
2. Exchange and account management
3. Data addition and management
4. Strategy integration
5. Result analysis
6. Performance optimization tips

In the next chapter, we will learn:

1. Time management mechanisms
2. Event-driven programming
3. Timers and scheduling
4. Concurrent processing
5. Advanced event patterns

## 5.10 Summary

### Key Points
1. BacktestEngine provides a complete backtesting environment
2. Supports multi-exchange, multi-asset, multi-strategy backtesting
3. Highly configurable, supports custom fees and slippage
4. Provides detailed result analysis tools

### Best Practices
1. Configure log levels appropriately to improve performance
2. Use batch processing for large amounts of data
3. Clear memory regularly
4. Save and analyze debugging information

## 5.11 References

- [BacktestEngine API Documentation](https://nautilustrader.io/docs/latest/api_reference/backtest/)
- [Backtesting Best Practices](https://nautilustrader.io/docs/latest/concepts/backtesting/)
- [Performance Optimization Guide](https://nautilustrader.io/docs/latest/developer_guide/performance/)