# 第5章：基础回测引擎使用

## 学习目标

通过本章学习，您将：
- 掌握 BacktestEngine 的核心配置
- 理解交易所和账户配置
- 学会添加交易工具和订单类型
- 运行完整的回测流程
- 分析和理解回测结果

## 5.1 BacktestEngine 概述

BacktestEngine 是 NautilusTrader 的核心回测组件，提供了高度可配置的回测环境。

### 5.1.1 架构图

```
┌─────────────────────────────────────────┐
│               BacktestEngine            │
├─────────────────────────────────────────┤
│  - 配置管理                              │
│  - 时间管理                              │
│  - 事件调度                              │
│  - 状态管理                              │
└─────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────┐
│              核心组件                     │
├─────────────────────────────────────────┤
│  • ExecutionEngine (执行引擎)             │
│  • DataEngine (数据引擎)                 │
│  • RiskEngine (风险引擎)                 │
│  • Portfolio (投资组合)                  │
│  • Cache (缓存)                          │
│  • MessageBus (消息总线)                 │
└─────────────────────────────────────────┘
```

### 5.1.2 主要功能

1. **模拟交易所**：模拟真实交易所的撮合机制
2. **订单管理**：支持多种订单类型和执行算法
3. **费用计算**：精确计算手续费和滑点
4. **风险控制**：实时风险检查和限制
5. **时间管理**：支持不同时间模式和加速回测

## 5.2 基础配置

### 5.2.1 BacktestEngineConfig

```python
"""
BacktestEngine 配置示例
"""

from nautilus_trader.config import BacktestEngineConfig, LoggingConfig
from nautilus_trader.model.identifiers import TraderId


def create_engine_config():
    """创建引擎配置"""
    config = BacktestEngineConfig(
        # 交易者标识
        trader_id=TraderId("BACKTESTER-001"),

        # 日志配置
        logging=LoggingConfig(
            log_level="INFO",        # 日志级别
            log_colors=True,         # 彩色日志
            print_to_stdout=True,    # 输出到控制台
            filter_messages=False,   # 不过滤消息
        ),

        # 时间配置
        time_increment_ns=1_000_000_000,  # 1秒时间增量（纳秒）
        bar_build_interval_ms=100,         # K线构建间隔（毫秒）

        # 其他选项
        debug=False,                        # 调试模式
        debug_messages=False,               # 调试消息
        benchmark_fees=False,               # 基准费用

        # 冲突解决模式
        fill_at_knowledgeable=False,        # 在不知情时成交
        reject_terminal_orders=False,      # 拒绝终端订单
        use_position_ids=False,             # 使用仓位ID
    )

    return config
```

### 5.2.2 创建和配置引擎

```python
"""
完整的引擎配置示例
"""

from decimal import Decimal
from nautilus_trader.backtest.engine import BacktestEngine
from nautilus_trader.model.enums import OmsType, AccountType
from nautilus_trader.model.identifiers import Venue
from nautilus_trader.model.objects import Money
from nautilus_trader.model.currencies import USD, BTC
from nautilus_trader.test_kit.providers import TestInstrumentProvider


def create_and_configure_engine():
    """创建并配置回测引擎"""

    # 1. 创建配置
    config = BacktestEngineConfig(
        trader_id=TraderId("BACKTESTER-001"),
        logging=LoggingConfig(log_level="INFO"),
    )

    # 2. 创建引擎
    engine = BacktestEngine(config=config)

    # 3. 添加交易所（可以添加多个）
    add_venues(engine)

    # 4. 添加交易工具
    add_instruments(engine)

    return engine


def add_venues(engine: BacktestEngine):
    """添加交易所配置"""

    # Binance 配置
    BINANCE = Venue("BINANCE")
    engine.add_venue(
        venue=BINANCE,
        # 订单管理系统类型
        oms_type=OmsType.HEDGING,  # 对冲模式（允许同时持有多空）
        # 账户类型
        account_type=AccountType.MARGIN,  # 保证金账户
        # 起始资金
        starting_balances=[
            Money(1_000_000, USD),  # 100万USDT
            Money(20, BTC),         # 20 BTC
        ],
        # 基础货币
        base_currency=USD,
        # 杠杆配置
        leverage_real=False,       # 不使用真实杠杆
        # 费用配置（可选）
        account_id=None,           # 自动生成
        # 其他配置
        max_order_submit_rate=None,    # 最大订单提交速率
        max_order_cancel_rate=None,    # 最大订单取消速率
    )

    # 添加第二个交易所
    BYBIT = Venue("BYBIT")
    engine.add_venue(
        venue=BYBIT,
        oms_type=OmsType.NETTING,  # 净仓模式（单一方向）
        account_type=AccountType.CASH,  # 现金账户
        starting_balances=[Money(500_000, USD)],
        base_currency=USD,
    )

    print(f"已添加交易所: {BINANCE}, {BYBIT}")


def add_instruments(engine: BacktestEngine):
    """添加交易工具"""

    # BTC/USDT 永续合约
    btc_usdt_perp = TestInstrumentProvider.btcusdt_perp_binance()
    engine.add_instrument(btc_usdt_perp)

    # ETH/USDT 现货
    eth_usdt = TestInstrumentProvider.ethusdt_binance()
    engine.add_instrument(eth_usdt)

    # BTC/USD 期货
    btc_usd = TestInstrumentProvider.btcusdt_cme()
    engine.add_instrument(btc_usd)

    print(f"已添加交易工具: {len(engine.instruments())} 个")


def main():
    """主函数示例"""
    engine = create_and_configure_engine()
    print(f"引擎配置完成，支持 {len(cache.venues())} 个交易所")
    return engine


if __name__ == "__main__":
    main()
```

## 5.3 数据管理

### 5.3.1 添加历史数据

```python
"""
数据管理示例
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
    """添加K线数据"""

    # 获取交易工具
    btc_usdt = TestInstrumentProvider.btcusdt_binance()

    # 定义K线类型
    bar_type = BarType(
        instrument_id=btc_usdt.id,
        bar_spec=BarSpecification(
            step=1,                        # 1根K线
            aggregation=BarAggregation.MINUTE,  # 1分钟
            price_type=PriceType.LAST,     # 最后成交价
        ),
        aggregation_source="BINANCE",
    )

    # 加载历史数据（示例）
    bars = load_historical_bars(bar_type, days=30)

    # 添加到引擎
    engine.add_data(bars)

    print(f"添加了 {len(bars)} 根 {bar_type} K线")


def add_tick_data(engine: BacktestEngine):
    """添加Tick数据"""

    # 获取交易工具
    btc_usdt = TestInstrumentProvider.btcusdt_binance()

    # 加载报价数据
    quotes = load_quote_ticks(btc_usdt.id, days=1)
    engine.add_data(quotes)

    # 加载成交数据
    trades = load_trade_ticks(btc_usdt.id, days=1)
    engine.add_data(trades)

    print(f"添加了 {len(quotes)} 个报价，{len(trades)} 个成交")


def add_mixed_data(engine: BacktestEngine):
    """添加混合数据（K线 + Tick）"""

    # 添加K线数据作为基础数据
    add_bar_data(engine)

    # 在关键时间点添加Tick数据
    tick_events = generate_key_tick_events()
    engine.add_data(tick_events)


def load_historical_bars(bar_type: BarType, days: int = 30) -> list[Bar]:
    """加载历史K线数据"""

    # 示例：生成模拟数据
    # 实际应用中应该从数据库或文件加载

    bars = []
    start_time = datetime.utcnow() - timedelta(days=days)

    for i in range(days * 24 * 60):  # 每分钟
        timestamp = start_time + timedelta(minutes=i)

        # 生成模拟OHLCV
        base_price = Decimal("50000")
        random_walk = Decimal(str((i % 100 - 50) * 10))

        bar = Bar(
            bar_type=bar_type,
            open=base_price + random_walk,
            high=base_price + random_walk + Decimal("100"),
            low=base_price + random_walk - Decimal("100"),
            close=base_price + random_walk + Decimal("50"),
            volume=Decimal(str(100 + i % 50)),
            ts_event=int(timestamp.timestamp() * 1e9),  # 纳秒时间戳
            ts_init=int(timestamp.timestamp() * 1e9),
        )

        bars.append(bar)

    return bars


def load_quote_ticks(instrument_id: InstrumentId, days: int = 1) -> list[QuoteTick]:
    """加载报价Tick数据"""
    quotes = []

    # 模拟报价数据
    for i in range(1000):
        price = Decimal("50000") + Decimal(str(i % 100))

        quote = QuoteTick(
            instrument_id=instrument_id,
            bid_price=price - Decimal("1"),
            ask_price=price + Decimal("1"),
            bid_size=Decimal("10"),
            ask_size=Decimal("10"),
            ts_event=1640995200000000000 + i * 1000000000,  # 每秒一个tick
            ts_init=1640995200000000000 + i * 1000000000,
        )

        quotes.append(quote)

    return quotes


def load_trade_ticks(instrument_id: InstrumentId, days: int = 1) -> list[TradeTick]:
    """加载成交Tick数据"""
    trades = []

    # 模拟成交数据
    for i in range(500):
        price = Decimal("50000") + Decimal(str(i % 100))

        trade = TradeTick(
            instrument_id=instrument_id,
            price=price,
            size=Decimal("1"),
            aggressor_side=AggressorSide.BUYER if i % 2 == 0 else AggressorSide.SELLER,
            trade_id=str(i),
            ts_event=1640995200000000000 + i * 2000000000,  # 每2秒一个trade
            ts_init=1640995200000000000 + i * 2000000000,
        )

        trades.append(trade)

    return trades
```

### 5.3.2 数据序列化

```python
"""
数据序列化和持久化示例
"""

import pickle
import json
from pathlib import Path
from typing import Dict, Any

from nautilus_trader.serialization.serializert import Serializer


class DataSerializer:
    """数据序列化工具"""

    def __init__(self):
        """初始化序列化器"""
        self.serializer = Serializer()

    def serialize_to_pickle(self, data: list, file_path: Path):
        """序列化到Pickle文件"""
        with open(file_path, 'wb') as f:
            pickle.dump(data, f)

    def deserialize_from_pickle(self, file_path: Path) -> list:
        """从Pickle文件反序列化"""
        with open(file_path, 'rb') as f:
            return pickle.load(f)

    def serialize_to_json(self, bars: list[Bar], file_path: Path):
        """序列化K线到JSON文件"""
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
        """导出到DataFrame"""
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
    """主函数示例"""
    # 创建序列化器
    serializer = DataSerializer()

    # 加载数据
    bars = load_historical_bars(bar_type, days=30)

    # 序列化到文件
    serializer.serialize_to_pickle(
        bars,
        Path("bars_2024_01.pkl")
    )

    serializer.serialize_to_json(
        bars,
        Path("bars_2024_01.json")
    )

    # 导出到DataFrame
    df = serializer.export_to_dataframe(bars)
    df.to_csv("bars_2024_01.csv", index=False)

    print("数据序列化完成")
```

## 5.4 策略添加和管理

### 5.4.1 添加策略

```python
"""
策略管理示例
"""

from typing import List
from nautilus_trader.trading.strategy import Strategy
from nautilus_trader.model.enums import OrderSide
from nautilus_trader.model.orders import MarketOrder
from nautilus_trader.model.data import Bar
from nautilus_trader.model.data import BarType


class SimpleBuyStrategy(Strategy):
    """简单买入策略"""

    def __init__(self, bar_type: BarType, buy_threshold: float = 0.01):
        """
        初始化策略

        Parameters
        ----------
        bar_type : BarType
            K线类型
        buy_threshold : float
            买入阈值（1%）
        """
        super().__init__()
        self.bar_type = bar_type
        self.buy_threshold = buy_threshold
        self.last_price = None
        self.bought = False

    def on_start(self):
        """策略启动"""
        self.log.info(f"策略启动: {self.bar_type}")
        self.subscribe_bars(self.bar_type)

    def on_bar(self, bar: Bar):
        """处理K线"""
        current_price = float(bar.close)

        if self.last_price is None:
            self.last_price = current_price
            return

        # 计算涨跌幅
        change = (current_price - self.last_price) / self.last_price

        # 如果涨幅超过阈值且未买入，则买入
        if change > self.buy_threshold and not self.bought:
            self._buy(bar)
        elif change < -self.buy_threshold and self.bought:
            self._sell(bar)

        self.last_price = current_price

    def _buy(self, bar: Bar):
        """执行买入"""
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
        self.log.info(f"买入: {bar.close}")

    def _sell(self, bar: Bar):
        """执行卖出"""
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
        self.log.info(f"卖出: {bar.close}")


def add_strategies(engine: BacktestEngine):
    """添加策略到引擎"""

    # 策略1：BTC/USDT 1分钟
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

    # 策略2：ETH/USDT 5分钟
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

    print(f"添加了 2 个策略")


class MultiAssetStrategy(Strategy):
    """多资产策略"""

    def __init__(self, bar_types: List[BarType]):
        """
        初始化多资产策略

        Parameters
        ----------
        bar_types : List[BarType]
            多个K线类型
        """
        super().__init__()
        self.bar_types = bar_types
        self.prices = {}

    def on_start(self):
        """策略启动"""
        self.log.info("多资产策略启动")

        for bar_type in self.bar_types:
            self.subscribe_bars(bar_type)

    def on_bar(self, bar: Bar):
        """处理K线"""
        instrument_id = str(bar.bar_type.instrument_id)
        self.prices[instrument_id] = float(bar.close)

        # 如果有至少两个资产的价格
        if len(self.prices) >= 2:
            self._check_arbitrage()

    def _check_arbitrage(self):
        """检查套利机会"""
        # 这里可以实现简单的套利逻辑
        pass


def add_multi_asset_strategy(engine: BacktestEngine):
    """添加多资产策略"""

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

    print(f"添加了多资产策略，监控 {len(bar_types)} 个资产")
```

## 5.5 运行回测

### 5.5.1 基本回测流程

```python
"""
完整的回测执行示例
"""

from datetime import datetime, timedelta
from decimal import Decimal
import time


def run_simple_backtest():
    """运行简单回测"""

    print("开始回测...")
    start_time = time.time()

    # 1. 创建引擎
    config = BacktestEngineConfig(
        trader_id=TraderId("BACKTESTER-001"),
        logging=LoggingConfig(log_level="INFO"),
    )
    engine = BacktestEngine(config=config)

    # 2. 添加交易所
    engine.add_venue(
        venue=Venue("BINANCE"),
        oms_type=OmsType.HEDGING,
        account_type=AccountType.MARGIN,
        starting_balances=[Money(100_000, USDT)],
        base_currency=USDT,
    )

    # 3. 添加交易工具
    btc_usdt = TestInstrumentProvider.btcusdt_binance()
    engine.add_instrument(btc_usdt)

    # 4. 添加数据
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

    # 5. 添加策略
    strategy = SimpleBuyStrategy(bar_type)
    engine.add_strategy(strategy)

    # 6. 运行回测
    print("执行回测...")
    engine.run()

    # 7. 获取结果
    end_time = time.time()
    duration = end_time - start_time

    print(f"\n回测完成！")
    print(f"执行时间: {duration:.2f} 秒")
    print(f"处理K线: {len(bars)} 根")

    # 8. 分析结果
    analyze_results(engine)

    # 9. 清理资源
    engine.dispose()

    return engine


def run_parameter_sweep():
    """运行参数扫描"""

    # 参数列表
    buy_thresholds = [0.003, 0.005, 0.01, 0.02]

    results = []

    for threshold in buy_thresholds:
        print(f"\n测试参数: buy_threshold = {threshold}")

        # 创建引擎
        config = BacktestEngineConfig(
            trader_id=TraderId(f"BACKTESTER-{threshold}"),
            logging=LoggingConfig(log_level="ERROR"),  # 减少日志
        )
        engine = BacktestEngine(config=config)

        # 配置引擎（简化）
        configure_engine(engine)

        # 创建策略
        strategy = SimpleBuyStrategy(bar_type, buy_threshold=threshold)
        engine.add_strategy(strategy)

        # 运行回测
        engine.run()

        # 收集结果
        result = analyze_results_brief(engine)
        result['threshold'] = threshold
        results.append(result)

        # 清理
        engine.dispose()

    # 打印对比结果
    print("\n参数扫描结果:")
    print("-" * 80)
    for result in results:
        print(f"阈值: {result['threshold']:.3f} | "
              f"交易次数: {result['trade_count']:3d} | "
              f"胜率: {result['win_rate']:5.1%} | "
              f"总盈亏: {result['total_pnl']:8.2f}")

    return results


def run_multi_strategy_backtest():
    """运行多策略回测"""

    print("开始多策略回测...")

    # 创建引擎
    config = BacktestEngineConfig(
        trader_id=TraderId("MULTI-STRATEGY-001"),
        logging=LoggingConfig(log_level="INFO"),
    )
    engine = BacktestEngine(config=config)

    # 添加多个交易所
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

    # 添加多个交易工具
    instruments = [
        TestInstrumentProvider.btcusdt_binance(),
        TestInstrumentProvider.ethusdt_binance(),
    ]

    for instrument in instruments:
        engine.add_instrument(instrument)

    # 添加数据
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

    # 添加多个策略
    strategies = [
        SimpleBuyStrategy(bar_type_1, buy_threshold=0.005),
        SimpleBuyStrategy(bar_type_2, buy_threshold=0.003),
        MultiAssetStrategy([bar_type_1, bar_type_2]),
    ]

    for strategy in strategies:
        engine.add_strategy(strategy)

    # 运行回测
    start_time = time.time()
    engine.run()
    duration = time.time() - start_time

    print(f"\n多策略回测完成！")
    print(f"执行时间: {duration:.2f} 秒")
    print(f"策略数量: {len(strategies)}")

    # 分析结果
    analyze_multi_strategy_results(engine)

    # 清理
    engine.dispose()

    return engine
```

## 5.6 结果分析

### 5.6.1 获取和分析结果

```python
"""
回测结果分析
"""

from typing import Dict, Any
import pandas as pd


def analyze_results(engine: BacktestEngine):
    """分析回测结果"""

    # 获取结果对象
    result = engine.get_result()

    # 基本统计
    print("\n=== 基本统计 ===")
    print(f"运行时间: {result.run_duration_seconds:.2f} 秒")
    print(f"处理事件: {result.total_events}")
    print(f"执行命令: {result.total_commands}")

    # 交易统计
    print("\n=== 交易统计 ===")
    stats = result.stats_pnls()

    for key, value in stats.items():
        if isinstance(value, Decimal):
            print(f"{key}: {float(value):.2f}")
        else:
            print(f"{key}: {value}")

    # 仓位信息
    print("\n=== 仓位信息 ===")
    print(f"当前仓位: {len(result.positions_open)} 个")
    print(f"已平仓位: {len(result.positions_closed)} 个")

    # 已平仓位详情
    if result.positions_closed:
        print("\n已平仓位详情:")
        for position in result.positions_closed[:5]:  # 只显示前5个
            print(f"  {position}")
            print(f"    开仓: {position.opened_timestamp}")
            print(f"    平仓: {position.closed_timestamp}")
            print(f"    盈亏: {position.realized_pnl}")
            print(f"    手续费: {position.total_commission}")

    # 订单统计
    print("\n=== 订单统计 ===")
    print(f"总订单数: {len(result.orders)}")

    # 按状态统计订单
    order_status_counts = {}
    for order in result.orders:
        status = order.status
        order_status_counts[status] = order_status_counts.get(status, 0) + 1

    for status, count in order_status_counts.items():
        print(f"  {status}: {count}")


def analyze_results_brief(engine: BacktestEngine) -> Dict[str, Any]:
    """简要分析结果"""

    result = engine.get_result()
    stats = result.stats_pnls()

    # 计算交易统计
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
    """分析多策略结果"""

    result = engine.get_result()

    # 获取所有策略
    strategies = engine.trader.strategies()

    print("\n=== 策略表现 ===")
    for strategy_id, strategy in strategies.items():
        # 这里需要更详细的策略级别分析
        # 由于框架限制，我们使用整体统计
        pass

    # 按交易所统计
    print("\n=== 交易所统计 ===")
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
        print(f"  交易次数: {stats['count']}")
        print(f"  总盈亏: {stats['total_pnl']:.2f}")
        print(f"  胜率: {win_rate:.1%}")


def export_results_to_csv(engine: BacktestEngine, filename: str):
    """导出结果到CSV"""

    result = engine.get_result()

    # 导出交易记录
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

    # 导出统计摘要
    stats = result.stats_pnls()
    stats_df = pd.DataFrame([stats])
    stats_df.to_csv(f"{filename}_stats.csv", index=False)

    print(f"结果已导出到 {filename}_*.csv")


def create_performance_report(engine: BacktestEngine) -> str:
    """创建性能报告"""

    result = engine.get_result()
    stats = analyze_results_brief(engine)

    report = f"""
回测性能报告
================

基本信息
--------
运行时间: {result.run_duration_seconds:.2f} 秒
处理事件: {result.total_events:,}

交易表现
--------
总交易次数: {stats['trade_count']}
胜率: {stats['win_rate']:.1%}
总盈亏: {stats['total_pnl']:,.2f} USDT
已实现盈亏: {stats['realized_pnl']:,.2f} USDT
最大回撤: {stats['max_drawdown']:,.2f} USDT
夏普比率: {stats['sharpe_ratio']:.2f}

风险指标
--------
盈利交易: {sum(1 for pos in result.positions_closed if pos.realized_pnl > 0)}
亏损交易: {sum(1 for pos in result.positions_closed if pos.realized_pnl < 0)}
平均盈亏: {stats['realized_pnl'] / stats['trade_count'] if stats['trade_count'] > 0 else 0:.2f} USDT

仓位信息
--------
当前持仓: {len(result.positions_open)} 个
已平仓位: {len(result.positions_closed)} 个
"""

    return report


def main():
    """主函数示例"""

    # 运行单个回测
    engine = run_simple_backtest()

    # 创建报告
    report = create_performance_report(engine)
    print(report)

    # 导出结果
    export_results_to_csv(engine, "backtest_2024_01")

    # 运行参数扫描
    print("\n" + "=" * 50)
    print("运行参数扫描...")
    results = run_parameter_sweep()

    # 找出最佳参数
    best_result = max(results, key=lambda x: x['total_pnl'])
    print(f"\n最佳参数: buy_threshold = {best_result['threshold']}")
    print(f"最高盈亏: {best_result['total_pnl']:,.2f} USDT")


if __name__ == "__main__":
    main()
```

## 5.7 高级功能

### 5.7.1 自定义费用模型

```python
"""
自定义费用模型
"""

from nautilus_trader.model.fees import FeeModel
from nautilus_trader.model.objects import Money
from nautilus_trader.model.objects import Price
from nautilus_trader.model.objects import Quantity
from decimal import Decimal


class CustomFeeModel(FeeModel):
    """自定义费用模型"""

    def __init__(self, base_rate: Decimal = Decimal("0.001")):
        """
        初始化费用模型

        Parameters
        ----------
        base_rate : Decimal
            基础费率（0.1%）
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
        计算费用

        Parameters
        ----------
        price : Price
            价格
        quantity : Quantity
            数量
        side : str
            方向
        instrument_id : InstrumentId
            交易工具ID

        Returns
        -------
        Money
            费用金额
        """
        # 计算交易额
        notional_value = price * quantity

        # 查找适用的费率
        rate = self.base_rate

        for threshold, discount_rate in self.volume_discounts.items():
            if notional_value >= threshold:
                rate = discount_rate
                break

        # 计算费用
        fee_amount = notional_value * rate

        # VIP折扣（示例）
        if is_vip_customer(instrument_id.venue):
            fee_amount *= Decimal("0.9")  # 10% VIP折扣

        return Money(fee_amount, instrument_id.quote_currency)


def configure_custom_fees(engine: BacktestEngine):
    """配置自定义费用"""

    # 为特定交易所配置费用模型
    venue = Venue("BINANCE")
    engine.add_fee_model(
        venue=venue,
        instrument_id=None,  # 应用于所有交易工具
        fee_model=CustomFeeModel(Decimal("0.001")),
    )

    print("已配置自定义费用模型")
```

### 5.7.2 滑点模型

```python
"""
滑点模拟
"""

from nautilus_trader.model.slippage import SlippageModel
from random import uniform
from decimal import Decimal


class RandomSlippageModel(SlippageModel):
    """随机滑点模型"""

    def __init__(self, max_slippage: Decimal = Decimal("0.001")):
        """
        初始化滑点模型

        Parameters
        ----------
        max_slippage : Decimal
            最大滑点（0.1%）
        """
        self.max_slippage = max_slippage

    def calculate(
        self,
        order: Order,
        fill_price: Price,
        instrument_id: InstrumentId,
    ) -> Price:
        """
        计算滑点后的价格

        Parameters
        ----------
        order : Order
            订单
        fill_price : Price
            填充价格
        instrument_id : InstrumentId
            交易工具ID

        Returns
        -------
        Price
            滑点后的价格
        """
        # 生成随机滑点
        slippage_rate = Decimal(str(uniform(0, float(self.max_slippage))))

        # 买单价格上移，卖单价格下移
        if order.side == OrderSide.BUY:
            slippage_amount = fill_price * slippage_rate
            adjusted_price = fill_price + slippage_amount
        else:
            slippage_amount = fill_price * slippage_rate
            adjusted_price = fill_price - slippage_amount

        return Price(adjusted_price, fill_price.precision)


def configure_slippage(engine: BacktestEngine):
    """配置滑点模型"""

    # 为交易所添加滑点模型
    engine.add_slippage_model(
        venue=Venue("BINANCE"),
        slippage_model=RandomSlippageModel(Decimal("0.0005")),  # 0.05%
    )

    print("已配置滑点模型")
```

## 5.8 最佳实践

### 5.8.1 性能优化

```python
"""
性能优化技巧
"""

import multiprocessing
from typing import List
from concurrent.futures import ProcessPoolExecutor


def optimized_backtest(param_set: List[Dict]) -> List[Dict]:
    """优化的批量回测"""

    # 使用进程池并行运行
    with ProcessPoolExecutor(max_workers=multiprocessing.cpu_count()) as executor:
        results = list(executor.map(run_single_backtest, param_set))

    return results


def run_single_backtest(params: Dict) -> Dict:
    """运行单个回测（用于并行）"""

    # 创建最小日志配置
    config = BacktestEngineConfig(
        trader_id=TraderId(f"BACKTEST-{params['id']}"),
        logging=LoggingConfig(log_level="ERROR"),
    )

    engine = BacktestEngine(config=config)

    # 最小配置
    engine.add_venue(
        venue=Venue("BINANCE"),
        oms_type=OmsType.NETTING,
        account_type=AccountType.CASH,
        starting_balances=[Money(params['balance'], USDT)],
        base_currency=USDT,
    )

    # 添加数据和策略
    # ...

    # 运行
    engine.run()

    # 收集结果
    result = analyze_results_brief(engine)
    result['params'] = params

    # 清理
    engine.dispose()

    return result


def memory_efficient_backtest():
    """内存高效的回测"""

    config = BacktestEngineConfig(
        trader_id=TraderId("EFFICIENT-001"),
        # 限制日志数量
        logging=LoggingConfig(
            log_level="WARNING",
            log_colors=False,
            print_to_stdout=False,
        ),
        # 启用内存优化
        debug=False,
        debug_messages=False,
    )

    engine = BacktestEngine(config=config)

    # 分批加载数据
    for day in range(30):  # 30天数据
        bars = load_bars_for_day(day)
        engine.add_data(bars)

        # 可选：定期清理内存
        if day % 7 == 0:
            engine.cache._flush()

    return engine
```

### 5.8.2 调试技巧

```python
"""
回测调试工具
"""

import logging
from datetime import datetime


class BacktestDebugger:
    """回测调试器"""

    def __init__(self, engine: BacktestEngine):
        """初始化调试器"""
        self.engine = engine
        self.order_log = []
        self.fill_log = []

    def enable_verbose_logging(self):
        """启用详细日志"""

        # 创建文件处理器
        file_handler = logging.FileHandler("backtest_debug.log")
        file_handler.setLevel(logging.DEBUG)

        # 添加到引擎日志器
        engine.logger.addHandler(file_handler)
        engine.logger.setLevel(logging.DEBUG)

    def log_order_event(self, event):
        """记录订单事件"""
        self.order_log.append({
            'time': datetime.utcnow(),
            'type': type(event).__name__,
            'order_id': str(event.order_id),
            'data': str(event),
        })

    def analyze_order_flow(self):
        """分析订单流"""

        print("\n=== 订单流分析 ===")

        # 订单状态变化
        order_states = {}
        for log in self.order_log:
            order_id = log['order_id']
            if order_id not in order_states:
                order_states[order_id] = []
            order_states[order_id].append({
                'time': log['time'],
                'event': log['type'],
            })

        # 找出异常订单
        for order_id, events in order_states.items():
            if len(events) > 10:  # 太多状态变化
                print(f"异常订单: {order_id}")
                for event in events:
                    print(f"  {event['time']}: {event['event']}")

    def save_debug_data(self):
        """保存调试数据"""

        import json
        import pandas as pd

        # 保存订单日志
        df_orders = pd.DataFrame(self.order_log)
        df_orders.to_csv("debug_orders.csv", index=False)

        # 保存填充日志
        df_fills = pd.DataFrame(self.fill_log)
        df_fills.to_csv("debug_fills.csv", index=False)

        print("调试数据已保存")
```

## 5.9 下一步

在本章中，我们学习了：

1. BacktestEngine 的配置和使用
2. 交易所和账户管理
3. 数据添加和管理
4. 策略集成
5. 结果分析
6. 性能优化技巧

在下一章中，我们将学习：

1. 时间管理机制
2. 事件驱动编程
3. 定时器和调度
4. 并发处理
5. 高级事件模式

## 5.10 总结

### 关键要点
1. BacktestEngine 提供了完整的回测环境
2. 支持多交易所、多资产、多策略回测
3. 高度可配置，支持自定义费用和滑点
4. 提供详细的结果分析工具

### 最佳实践
1. 合理配置日志级别以提高性能
2. 使用批量处理处理大量数据
3. 定期清理内存
4. 保存和分析调试信息

## 5.11 参考资料

- [BacktestEngine API 文档](https://nautilustrader.io/docs/latest/api_reference/backtest/)
- [回测最佳实践](https://nautilustrader.io/docs/latest/concepts/backtesting/)
- [性能优化指南](https://nautilustrader.io/docs/latest/developer_guide/performance/)