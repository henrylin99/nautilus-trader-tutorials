# 第3章：快速开始

## 学习目标

通过本章学习，您将：
- 理解 NautilusTrader 的核心概念和术语
- 运行第一个简单的策略示例
- 掌握基本的工作流程
- 学会查看和分析回测结果
- 了解两种 API 层级的区别和用途

## 3.1 核心概念

### 3.1.1 基本术语

NautilusTrader 使用了一套精确的术语来描述交易系统的各个组件：

#### 数据类型 (Data Types)
- **QuoteTick**：报价数据（买卖价）
- **TradeTick**：成交数据
- **Bar**：K线数据（OHLCV）
- **OrderBookDelta**：订单簿增量更新
- **OrderBookSnapshot**：订单簿快照

#### 交易对象 (Trading Objects)
- **Instrument**：交易工具（如 BTCUSDT）
- **Order**：订单指令
- **Position**：持仓
- **Account**：账户
- **Venue**：交易场所（如 BINANCE）

#### 策略组件 (Strategy Components)
- **Strategy**：交易策略
- **Actor**：独立执行的组件
- **Clock**：时钟管理
- **Cache**：数据缓存

### 3.1.2 API 层级

NautilusTrader 提供了两个主要 API 层级：

#### High-Level API（推荐）
使用 `BacktestNode` 和 `TradingNode`，适合大多数用户：

```python
# 优点：
# - 简单易用
# - 配置驱动
# - 易于从回测切换到实盘
# - 生产环境就绪

from nautilus_trader.config import BacktestRunConfig
from nautilus_trader.test_kit.providers import TestInstrumentProvider

# 配置驱动的方式
config = BacktestRunConfig(
    venues=[...],
    instruments=[...],
    data=[...],
    strategies=[...],
)

# 运行回测
from nautilus_trader.live.runner import BacktestRunner
result = BacktestRunner.run(config)
```

#### Low-Level API
使用 `BacktestEngine`，适合库开发者：

```python
# 优点：
# - 更灵活
# - 细粒度控制
# - 适合研究和实验
# - 可深度定制

from nautilus_trader.backtest.engine import BacktestEngine
from nautilus_trader.config import BacktestEngineConfig

# 编程式配置
engine = BacktestEngine(config=BacktestEngineConfig(...))
engine.add_venue(...)
engine.add_instrument(...)
engine.add_strategy(...)
engine.run()
```

**建议**：初学者从 High-Level API 开始，熟练后再使用 Low-Level API。

## 3.2 第一个策略：简单的计数器

让我们从一个最简单的策略开始，这个策略只计算并记录处理了多少根K线。

### 3.2.1 创建策略文件

创建 `counter_strategy.py`：
```python
"""
一个简单的计数器策略
统计处理的K线数量
"""

from decimal import Decimal
from typing import Optional

from nautilus_trader.trading.strategy import Strategy
from nautilus_trader.model.data import Bar
from nautilus_trader.model.data import BarType
from nautilus_trader.common.enums import LogColor


class CounterStrategy(Strategy):
    """
    简单的计数器策略

    这个策略不进行任何交易，只是：
    1. 订阅K线数据
    2. 统计处理的K线数量
    3. 定期输出统计信息
    """

    def __init__(self, bar_type: BarType):
        """
        初始化策略

        Parameters
        ----------
        bar_type : BarType
            要订阅的K线类型
        """
        super().__init__()

        self.bar_type = bar_type
        self.bar_count = 0
        self.last_price: Optional[Decimal] = None
        self.high: Optional[Decimal] = None
        self.low: Optional[Decimal] = None

        # 策略配置
        self.log_interval = 100  # 每处理100根K线输出一次日志

    def on_start(self) -> None:
        """
        策略启动时调用

        在这里进行初始化工作，如订阅数据
        """
        self.log.info(f"策略启动: {self.bar_type}")
        self.log.info(f"每处理 {self.log_interval} 根K线输出一次统计信息")

        # 订阅K线数据
        self.subscribe_bars(self.bar_type)

        self.log.info("已订阅K线数据")

    def on_bar(self, bar: Bar) -> None:
        """
        处理每一根K线

        Parameters
        ----------
        bar : Bar
            到达的K线数据
        """
        self.bar_count += 1

        # 更新统计信息
        self.last_price = bar.close
        if self.high is None or bar.high > self.high:
            self.high = bar.high
        if self.low is None or bar.low < self.low:
            self.low = bar.low

        # 定期输出统计信息
        if self.bar_count % self.log_interval == 0:
            self._log_statistics()

    def _log_statistics(self) -> None:
        """输出统计信息"""
        self.log.info(
            f"已处理 {self.bar_count} 根K线 | "
            f"当前价格: {self.last_price} | "
            f"最高价: {self.high} | "
            f"最低价: {self.low}",
            color=LogColor.YELLOW
        )

    def on_stop(self) -> None:
        """
        策略停止时调用

        在这里进行清理工作，如输出最终统计
        """
        self.log.info(f"策略停止")
        self.log.info(f"总共处理了 {self.bar_count} 根K线")

        if self.bar_count > 0:
            self._log_statistics()
```

### 3.2.2 使用 High-Level API 运行回测

创建 `run_counter_high_level.py`：
```python
"""
使用 High-Level API 运行计数器策略回测
"""

from datetime import datetime
from decimal import Decimal
from pathlib import Path

from nautilus_trader.config import BacktestRunConfig
from nautilus_trader.config import BacktestVenueConfig
from nautilus_trader.config import ImportableStrategyConfig
from nautilus_trader.config import LoggingConfig
from nautilus_trader.model.data import BarType
from nautilus_trader.model.enums import BarAggregation
from nautilus_trader.model.enums import PriceType
from nautilus_trader.model.identifiers import InstrumentId
from nautilus_trader.model.objects import Money
from nautilus_trader.model.currencies import USDT
from nautilus_trader.test_kit.providers import TestInstrumentProvider
from nautilus_trader.persistence.wranglers import BarDataWrangler

from counter_strategy import CounterStrategy


def main():
    """主函数"""

    # 1. 配置日志
    logging_config = LoggingConfig(
        log_level="INFO",
        log_colors=True,
        print_to_stdout=True,
    )

    # 2. 创建测试交易工具（BTC/USDT）
    btc_usdt = TestInstrumentProvider.btcusdt_binance()

    # 3. 定义K线类型（1分钟K线）
    bar_type = BarType(
        instrument_id=btc_usdt.id,
        bar_spec=BarSpecification(
            step=1,
            aggregation=BarAggregation.MINUTE,
            price_type=PriceType.LAST,
        ),
        aggregation_source="BINANCE",
    )

    # 4. 生成测试数据
    print("生成测试数据...")
    wrangler = BarDataWrangler(bar_type=bar_type, instrument=btc_usdt)

    # 创建一些模拟K线数据
    bars = []
    base_price = Decimal("50000")
    for i in range(500):
        # 简单的随机价格变动
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

    print(f"生成了 {len(bars)} 根1分钟K线数据")

    # 5. 配置回测
    config = BacktestRunConfig(
        logging=logging_config,

        # 配置交易所
        venues=[
            BacktestVenueConfig(
                name="BINANCE",
                oms_type="HEDGING",  # 对冲模式
                account_type="MARGIN",  # 保证金账户
                base_currency=USDT,
                starting_balances=[Money(100_000, USDT)],
            ),
        ],

        # 配置数据
        data={
            "catalog_path": None,  # 不使用数据目录
            "bar_types": [bar_type],
            "bars": bars,
        },

        # 配置策略
        strategies=[
            ImportableStrategyConfig(
                strategy_path=Path(__file__).parent / "counter_strategy.py",
                config_path=None,  # 使用默认配置
                config={
                    "bar_type": str(bar_type),
                },
            ),
        ],
    )

    # 6. 运行回测
    print("\n运行回测...")
    from nautilus_trader.live.runner import BacktestRunner

    result = BacktestRunner.run(config)

    # 7. 输出结果
    print("\n回测完成！")
    print(f"处理时间: {result.run_duration_seconds:.2f} 秒")
    print(f"总K线数: {result.total_events}")

    # 查看交易统计（虽然这个策略没有交易）
    stats = result.stats_pnls()
    print("\nPnL 统计:")
    for key, value in stats.items():
        print(f"  {key}: {value}")


if __name__ == "__main__":
    main()
```

### 3.2.3 使用 Low-Level API 运行回测

创建 `run_counter_low_level.py`：
```python
"""
使用 Low-Level API 运行计数器策略回测
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
    """主函数"""

    # 1. 配置回测引擎
    config = BacktestEngineConfig(
        trader_id=TraderId("BACKTESTER-001"),
        logging=LoggingConfig(
            log_level="INFO",
            log_colors=True,
        ),
    )

    # 创建回测引擎
    engine = BacktestEngine(config=config)

    # 2. 添加交易所
    BINANCE = Venue("BINANCE")
    engine.add_venue(
        venue=BINANCE,
        oms_type=OmsType.HEDGING,  # 对冲模式
        account_type=AccountType.MARGIN,  # 保证金账户
        base_currency=USDT,
        starting_balances=[Money(100_000, USDT)],
    )

    # 3. 创建并添加交易工具
    btc_usdt = TestInstrumentProvider.btcusdt_binance()
    engine.add_instrument(btc_usdt)

    # 4. 定义K线类型
    bar_type = BarType(
        instrument_id=btc_usdt.id,
        bar_spec=BarSpecification(
            step=1,
            aggregation=BarAggregation.MINUTE,
            price_type=PriceType.LAST,
        ),
        aggregation_source="BINANCE",
    )

    # 5. 生成测试数据
    print("生成测试数据...")
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

    print(f"生成了 {len(bars)} 根1分钟K线数据")

    # 6. 添加数据到引擎
    engine.add_data(bars)

    # 7. 创建并添加策略
    strategy = CounterStrategy(bar_type=bar_type)
    engine.add_strategy(strategy)

    # 8. 运行回测
    print("\n运行回测...")
    start_time = datetime.now()

    engine.run()

    end_time = datetime.now()
    duration = end_time - start_time

    # 9. 获取结果
    result = engine.get_result()

    # 10. 输出结果
    print("\n回测完成！")
    print(f"处理时间: {duration.total_seconds():.2f} 秒")

    # 清理资源
    engine.dispose()
    print("\n资源已清理")


if __name__ == "__main__":
    main()
```

### 3.2.4 运行示例

```bash
# 确保虚拟环境已激活
source .venv/bin/activate

# 运行 High-Level API 版本
python run_counter_high_level.py

# 或运行 Low-Level API 版本
python run_counter_low_level.py
```

### 3.2.5 预期输出

```
生成测试数据...
生成了 500 根1分钟K线数据

运行回测...

回测完成！
处理时间: 0.15 秒
总K线数: 500

PnL 统计:
  Total PnL: 0 USDT
  Realized PnL: 0 USDT
  Unrealized PnL: 0 USDT
```

## 3.3 第二个策略：简单的 EMA 交叉

让我们创建一个更实用的策略，基于指数移动平均线（EMA）交叉。

### 3.3.1 创建 EMA 交叉策略

创建 `ema_cross_strategy.py`：
```python
"""
EMA 交叉策略

当快速 EMA 上穿慢速 EMA 时买入
当快速 EMA 下穿慢速 EMA 时卖出
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
    EMA 交叉策略

    使用两条指数移动平均线：
    - 快速 EMA（10周期）
    - 慢速 EMA（30周期）

    交易逻辑：
    - 金叉（快线上穿慢线）时买入
    - 死叉（快线下穿慢线）时卖出
    """

    def __init__(
        self,
        bar_type: BarType,
        fast_ema_period: int = 10,
        slow_ema_period: int = 30,
        trade_size: Decimal = Decimal("0.01"),
    ):
        """
        初始化策略

        Parameters
        ----------
        bar_type : BarType
            要订阅的K线类型
        fast_ema_period : int, default 10
            快速 EMA 周期
        slow_ema_period : int, default 30
            慢速 EMA 周期
        trade_size : Decimal, default "0.01"
            交易数量
        """
        super().__init__()

        self.bar_type = bar_type
        self.fast_ema_period = fast_ema_period
        self.slow_ema_period = slow_ema_period
        self.trade_size = trade_size

        # 创建 EMA 指标
        self.fast_ema = EMA(self.fast_ema_period)
        self.slow_ema = EMA(self.slow_ema_period)

        # 状态跟踪
        self.in_position = False
        self.entry_price: Optional[Decimal] = None
        self.last_signal: Optional[str] = None

        # 统计信息
        self.total_trades = 0
        self.winning_trades = 0
        self.losing_trades = 0

        # 日志记录
        self.log_enabled = True

    def on_start(self) -> None:
        """策略启动时调用"""
        self.log.info(
            f"EMA 交叉策略启动 - "
            f"快速EMA: {self.fast_ema_period}, "
            f"慢速EMA: {self.slow_ema_period}, "
            f"交易数量: {self.trade_size}"
        )

        # 订阅K线数据
        self.subscribe_bars(self.bar_type)

        # 订阅账户事件（成交回报）
        self.subscribe_account_events()

    def on_bar(self, bar: Bar) -> None:
        """处理每一根K线"""
        # 更新 EMA
        self.fast_ema.update_raw(bar.close)
        self.slow_ema.update_raw(bar.close)

        # 等待两个 EMA 都有足够的数据
        if not self.fast_ema.initialized or not self.slow_ema.initialized:
            return

        # 检查交叉信号
        self._check_crossover(bar)

        # 输出当前状态
        if self.log_enabled and bar.ts_event.minute % 5 == 0:  # 每5分钟输出一次
            self._log_status(bar)

    def _check_crossover(self, bar: Bar) -> None:
        """检查 EMA 交叉"""
        fast_value = self.fast_ema.value
        slow_value = self.slow_ema.value
        fast_prev = self.fast_ema.value_1
        slow_prev = self.slow_ema.value_1

        # 金叉：快线上穿慢线
        if (fast_value > slow_value and
            fast_prev <= slow_prev and
            not self.in_position):

            self._enter_long(bar)

        # 死叉：快线下穿慢线
        elif (fast_value < slow_value and
              fast_prev >= slow_prev and
              self.in_position):

            self._exit_long(bar)

    def _enter_long(self, bar: Bar) -> None:
        """开多头仓位"""
        # 创建市价买单
        order = MarketOrder(
            trader_id=self.trader_id,
            strategy_id=self.id,
            instrument_id=self.bar_type.instrument_id,
            order_side=OrderSide.BUY,
            quantity=self.trade_size,
            tags="EMA_CROSS_ENTRY",
        )

        # 提交订单
        self.submit_order(order)

        # 记录信号
        self.last_signal = "BUY"
        self.log.info(
            f"买入信号 - 价格: {bar.close}, "
            f"快EMA: {self.fast_ema.value:.2f}, "
            f"慢EMA: {self.slow_ema.value:.2f}",
            color=LogColor.GREEN,
        )

    def _exit_long(self, bar: Bar) -> None:
        """平多头仓位"""
        # 创建市价卖单
        order = MarketOrder(
            trader_id=self.trader_id,
            strategy_id=self.id,
            instrument_id=self.bar_type.instrument_id,
            order_side=OrderSide.SELL,
            quantity=self.trade_size,
            tags="EMA_CROSS_EXIT",
        )

        # 提交订单
        self.submit_order(order)

        # 记录信号
        self.last_signal = "SELL"
        self.log.info(
            f"卖出信号 - 价格: {bar.close}, "
            f"快EMA: {self.fast_ema.value:.2f}, "
            f"慢EMA: {self.slow_ema.value:.2f}",
            color=LogColor.RED,
        )

    def on_order_filled(self, event: OrderFilled) -> None:
        """处理订单成交"""
        self.total_trades += 1

        if event.order_side == OrderSide.BUY:
            self.in_position = True
            self.entry_price = event.last_px
            self.log.info(
                f"买入成交 - 数量: {event.last_qty}, "
                f"价格: {event.last_px}, "
                f"手续费: {event.commission}"
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
                    f"卖出成交 - 数量: {event.last_qty}, "
                    f"价格: {event.last_px}, "
                    f"盈亏: {pnl:.2f}, "
                    f"手续费: {event.commission}"
                )

    def _log_status(self, bar: Bar) -> None:
        """输出当前状态"""
        position_str = "持仓" if self.in_position else "空仓"
        self.log.info(
            f"[{bar.ts_event.strftime('%H:%M')}] "
            f"价格: {bar.close:.2f} | "
            f"快EMA: {self.fast_ema.value:.2f} | "
            f"慢EMA: {self.slow_ema.value:.2f} | "
            f"状态: {position_str}",
            color=LogColor.BLUE,
        )

    def on_stop(self) -> None:
        """策略停止时调用"""
        self.log.info("策略停止")
        self.log.info(f"总交易次数: {self.total_trades}")
        self.log.info(f"盈利交易: {self.winning_trades}")
        self.log.info(f"亏损交易: {self.losing_trades}")

        if self.total_trades > 0:
            win_rate = self.winning_trades / self.total_trades * 100
            self.log.info(f"胜率: {win_rate:.2f}%")
```

### 3.3.2 运行 EMA 交叉策略

创建 `run_ema_cross.py`：
```python
"""
运行 EMA 交叉策略回测
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
    """生成具有趋势的测试数据"""

    # 创建 BTC/USDT 交易工具
    btc_usdt = TestInstrumentProvider.btcusdt_binance()

    # 定义K线类型
    bar_type = BarType(
        instrument_id=btc_usdt.id,
        bar_spec=BarSpecification(
            step=1,
            aggregation=BarAggregation.MINUTE,
            price_type=PriceType.LAST,
        ),
        aggregation_source="BINANCE",
    )

    # 创建数据处理器
    wrangler = BarDataWrangler(bar_type=bar_type, instrument=btc_usdt)

    # 生成数据
    bars = []
    base_price = Decimal("50000")
    trend = Decimal("0")  # 0 = 横盘, 1 = 上涨, -1 = 下跌
    trend_duration = 0

    for i in range(bars_needed):
        # 随机改变趋势
        if trend_duration <= 0:
            trend = Decimal(str([ -1, 0, 1 ][i % 3]))  # 循环：下跌、横盘、上涨
            trend_duration = 100 + (i % 100)  # 100-200的随机周期

        # 计算价格变化
        if trend != 0:
            price_change = trend * Decimal("50") + Decimal(str((i % 20 - 10) * 2))
        else:
            price_change = Decimal(str((i % 20 - 10) * 5))

        # 更新价格
        base_price += price_change
        if base_price < Decimal("45000"):
            base_price = Decimal("45000")
        elif base_price > Decimal("55000"):
            base_price = Decimal("55000")

        # 生成OHLCV
        open_price = base_price - Decimal("20")
        high_price = base_price + Decimal("50")
        low_price = base_price - Decimal("70")
        close_price = base_price + Decimal("10")
        volume = Decimal("100") + Decimal(str(i % 50))

        # 创建K线
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
    """主函数"""

    print("生成测试数据...")
    bars, bar_type, btc_usdt = generate_trend_data(1440)  # 1天的1分钟K线
    print(f"生成了 {len(bars)} 根K线数据")

    # 配置回测
    config = BacktestRunConfig(
        logging=LoggingConfig(
            log_level="WARNING",  # 减少日志输出
            log_colors=True,
        ),

        # 配置交易所
        venues=[
            BacktestVenueConfig(
                name="BINANCE",
                oms_type="HEDGING",
                account_type="MARGIN",
                base_currency=USDT,
                starting_balances=[Money(10_000, USDT)],
            ),
        ],

        # 配置数据
        data={
            "catalog_path": None,
            "bar_types": [bar_type],
            "bars": bars,
        },

        # 配置策略
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

    # 运行回测
    print("\n运行回测...")
    from nautilus_trader.live.runner import BacktestRunner

    result = BacktestRunner.run(config)

    # 输出结果
    print("\n回测完成！")
    print(f"处理时间: {result.run_duration_seconds:.2f} 秒")

    # 查看详细统计
    stats = result.stats_pnls()
    print("\n=== 交易统计 ===")
    print(f"总交易次数: {result.stats_returns_count()}")

    if stats:
        for key, value in stats.items():
            if isinstance(value, (int, float, Decimal)):
                print(f"{key}: {value}")
            else:
                print(f"{key}: {value}")

    # 查看仓位信息
    print("\n=== 仓位信息 ===")
    for position in result.positions_closed:
        print(f"已平仓: {position}")

    if result.positions_open:
        for position in result.positions_open:
            print(f"未平仓: {position}")


if __name__ == "__main__":
    main()
```

### 3.3.3 运行并查看结果

```bash
python run_ema_cross.py
```

预期输出示例：
```
生成测试数据...
生成了 1440 根K线数据

运行回测...

回测完成！
处理时间: 0.45 秒

=== 交易统计 ===
总交易次数: 20
Total PnL: 45.23 USDT
Realized PnL: 45.23 USDT
Unrealized PnL: 0 USDT

=== 仓位信息 ===
已平仓: [Position(...)]
已平仓: [Position(...)]
...
```

## 3.4 理解回测结果

### 3.4.1 基本统计指标

- **Total PnL**：总盈亏（包括已实现和未实现）
- **Realized PnL**：已实现盈亏
- **Unrealized PnL**：未实现盈亏
- **Max Drawdown**：最大回撤
- **Sharpe Ratio**：夏普比率
- **Win Rate**：胜率

### 3.4.2 分析回测报告

NautilusTrader 提供了丰富的分析工具，在后续章节中我们会详细介绍如何生成详细的分析报告，包括：

- 交易分析
- 业绩归因
- 风险指标
- 回测日志

## 3.5 最佳实践

### 3.5.1 策略开发原则

1. **保持简单**：从简单的逻辑开始
2. **充分测试**：在多个市场条件下测试
3. **风险管理**：始终包含风险控制
4. **日志记录**：记录关键决策点
5. **参数化**：使用配置参数而非硬编码

### 3.5.2 调试技巧

1. **使用日志**：
   ```python
   self.log.info("调试信息", color=LogColor.YELLOW)
   ```

2. **输出数据**：
   ```python
   print(f"当前价格: {bar.close}")
   ```

3. **条件断点**：
   ```python
   if bar.ts_event.hour == 9:
       import pdb; pdb.set_trace()
   ```

### 3.5.3 性能考虑

1. **避免重复计算**：缓存计算结果
2. **使用适当的数据结构**：选择合适的数据类型
3. **限制日志输出**：避免过多日志影响性能

## 3.6 下一步

恭喜！您已经成功运行了第一个 NautilusTrader 策略。在下一章中，我们将学习：

1. 如何加载真实的历史数据
2. 支持的各种数据格式
3. 数据预处理和清洗
4. 高效的数据存储方案

## 3.7 总结

本章介绍了 NautilusTrader 的基础概念和使用方法：

### 核心要点
1. NautilusTrader 使用精确的术语定义各种概念
2. 提供了 High-Level 和 Low-Level 两种 API
3. 事件驱动的架构使得策略开发直观
4. 相同的代码可用于回测和实盘

### 关键技能
- 理解核心概念和术语
- 创建简单的策略类
- 配置和运行回测
- 分析基本结果

## 3.8 参考资料

- [策略开发指南](https://nautilustrader.io/docs/latest/concepts/strategies/)
- [事件系统文档](https://nautilustrader.io/docs/latest/concepts/message_bus/)
- [BacktestRunner API](https://nautilustrader.io/docs/latest/api_reference/runner/)
- [技术指标列表](https://nautilustrader.io/docs/latest/api_reference/indicators/)