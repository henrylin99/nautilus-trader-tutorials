# 第7章：技术指标系统

## 学习目标

通过本章学习，您将：
- 掌握 NautilusTrader 内置技术指标的使用
- 理解指标的更新机制和生命周期
- 学会创建自定义技术指标
- 实现指标级联和组合
- 优化指标计算性能

## 7.1 技术指标概览

### 7.1.1 指标基类

NautilusTrader 的所有指标都继承自 `Indicator` 基类：

```python
"""
技术指标基类说明
"""

from abc import ABC, abstractmethod
from decimal import Decimal
from typing import Any, Optional

from nautilus_trader.model.data import Bar
from nautilus_trader.core.message import Event


class Indicator(ABC):
    """技术指标基类"""

    def __init__(
        self,
        params: Optional[dict] = None,
        name: Optional[str] = None,
    ):
        """
        初始化指标

        Parameters
        ----------
        params : dict, optional
            指标参数
        name : str, optional
            指标名称
        """
        self._params = params or {}
        self._name = name or self.__class__.__name__
        self._initialized = False
        self._has_inputs = False
        self._inputs = 0

    @property
    def name(self) -> str:
        """获取指标名称"""
        return self._name

    @property
    def initialized(self) -> bool:
        """是否已初始化"""
        return self._initialized

    @property
    def has_inputs(self) -> bool:
        """是否有输入数据"""
        return self._has_inputs

    @property
    def count_inputs(self) -> int:
        """输入数据数量"""
        return self._inputs

    @abstractmethod
    def handle_bar(self, bar: Bar) -> None:
        """
        处理K线数据（抽象方法）

        Parameters
        ----------
        bar : Bar
            K线数据
        """
        pass

    @abstractmethod
    def handle_quote_tick(self, quote) -> None:
        """
        处理报价数据（抽象方法）

        Parameters
        ----------
        quote : QuoteTick
            报价数据
        """
        pass

    @abstractmethod
    def handle_trade_tick(self, trade) -> None:
        """
        处理成交数据（抽象方法）

        Parameters
        ----------
        trade : TradeTick
            成交数据
        """
        pass

    @abstractmethod
    def reset(self) -> None:
        """重置指标状态"""
        pass

    @abstractmethod
    def _update(self, value: float) -> None:
        """更新指标值（内部方法）"""
        pass

    def update_raw(self, value: float) -> None:
        """
        更新原始值

        Parameters
        ----------
        value : float
            原始值
        """
        if not isinstance(value, (int, float, Decimal)):
            raise TypeError(f"Invalid input type: {type(value)}")

        self._has_inputs = True
        self._inputs += 1
        self._update(float(value))

    def to_dict(self) -> dict:
        """转换为字典"""
        return {
            'name': self.name,
            'params': self._params,
            'initialized': self.initialized,
            'inputs': self._inputs,
            'has_inputs': self.has_inputs,
        }


# 自定义指标示例
class CustomIndicator(Indicator):
    """自定义指标示例"""

    def __init__(self, period: int = 20):
        """
        初始化指标

        Parameters
        ----------
        period : int
            计算周期
        """
        super().__init__(
            params={'period': period},
            name="CustomIndicator"
        )

        self.period = period
        self.values = []  # 存储历史值
        self._value = 0.0

    @property
    def value(self) -> float:
        """获取指标值"""
        return self._value

    @property
    def value_1(self) -> float:
        """获取前一个值"""
        return self.values[-2] if len(self.values) >= 2 else 0.0

    def handle_bar(self, bar: Bar) -> None:
        """处理K线数据"""
        self.update_raw(float(bar.close))

    def handle_quote_tick(self, quote) -> None:
        """处理报价数据"""
        # 使用中间价
        mid_price = (float(quote.bid_price) + float(quote.ask_price)) / 2
        self.update_raw(mid_price)

    def handle_trade_tick(self, trade) -> None:
        """处理成交数据"""
        self.update_raw(float(trade.price))

    def reset(self) -> None:
        """重置指标"""
        self.values.clear()
        self._value = 0.0
        self._initialized = False
        self._has_inputs = False
        self._inputs = 0

    def _update(self, value: float) -> None:
        """更新指标值"""
        self.values.append(value)

        if len(self.values) > self.period:
            self.values.pop(0)

        # 计算简单移动平均
        if len(self.values) >= self.period:
            self._value = sum(self.values) / self.period
            self._initialized = True
```

### 7.1.2 内置指标库

NautilusTrader 提供了丰富的内置指标：

```python
"""
内置指标库导览
"""

# 移动平均线指标
from nautilus_trader.indicators.average import (
    SMA,      # 简单移动平均
    EMA,      # 指数移动平均
    WMA,      # 加权移动平均
    DEMA,     # 双指数移动平均
    TEMA,     # 三指数移动平均
    KAMA,     # 考夫曼自适应移动平均
    MAMA,     # MESA自适应移动平均
    FRAMA,    # 分形自适应移动平均
    HMA,      # 赫尔移动平均
)

# 动量指标
from nautilus_trader.indicators.momentum import (
    RSI,      # 相对强弱指数
    MACD,     # MACD指标
    Stochastic, # 随机指标
    CCI,      # 商品通道指数
    ROC,      # 变化率
    MOMENTUM, # 动量指标
    WILLIAMS_R, # 威廉指标
)

# 趋势指标
from nautilus_trader.indicators.trend import (
    ADX,      # 平均趋向指数
    Aroon,    # 阿隆指标
    ATR,      # 平均真实波幅
    ParabolicSAR, # 抛物线SAR
    TRIX,     # 三重指数平滑平均
    Vortex,   # 涡流指标
)

# 波动率指标
from nautilus_trader.indicators.volatility import (
    BollingerBands, # 布林带
    KeltnerChannels, # 肯特纳通道
    DonchianChannels, # 唐奇安通道
    StandardDeviation, # 标准差
)

# 成交量指标
from nautilus_trader.indicators.volume import (
    OBV,      # 能量潮
    VWAP,     # 成交量加权平均价
    MoneyFlowIndex, # 资金流量指数
    AccumulationDistribution, # 累积派发线
    EaseOfMovement, # 简易波动指标
)

# 模糊逻辑指标
from nautilus_trader.indicators.fuzzy_candlesticks import (
    FuzzyCandlestickPattern, # 模糊K线形态
)
```

## 7.2 移动平均线指标

### 7.2.1 基础移动平均

```python
"""
移动平均线使用示例
"""

from decimal import Decimal
from nautilus_trader.model.data import Bar
from nautilus_trader.model.data import BarType
from nautilus_trader.model.enums import BarAggregation, PriceType
from nautilus_trader.model.identifiers import InstrumentId
from nautilus_trader.trading.strategy import Strategy
from nautilus_trader.common.enums import LogColor


class MovingAverageStrategy(Strategy):
    """移动平均线策略"""

    def __init__(
        self,
        bar_type: BarType,
        fast_period: int = 10,
        slow_period: int = 30,
    ):
        """
        初始化策略

        Parameters
        ----------
        bar_type : BarType
            K线类型
        fast_period : int
            快速均线周期
        slow_period : int
            慢速均线周期
        """
        super().__init__()

        self.bar_type = bar_type

        # 创建移动平均线指标
        self.sma_fast = SMA(period=fast_period)
        self.sma_slow = SMA(period=slow_period)

        # 也可以使用EMA
        # self.ema_fast = EMA(period=fast_period)
        # self.ema_slow = EMA(period=slow_period)

        # 状态跟踪
        self.last_signal = None
        self.position = None

    def on_start(self):
        """策略启动"""
        self.log.info("移动平均线策略启动")
        self.log.info(f"快线周期: {self.sma_fast.period}, "
                     f"慢线周期: {self.sma_slow.period}")

        # 订阅K线数据
        self.subscribe_bars(self.bar_type)

    def on_bar(self, bar: Bar):
        """处理K线数据"""
        # 更新指标
        self.sma_fast.update_raw(float(bar.close))
        self.sma_slow.update_raw(float(bar.close))

        # 等待指标初始化
        if not (self.sma_fast.initialized and self.sma_slow.initialized):
            return

        # 获取指标值
        fast_value = self.sma_fast.value
        slow_value = self.sma_slow.value

        # 检查金叉/死叉
        self._check_crossover(fast_value, slow_value, bar)

        # 记录指标值
        if self.count_bars() % 10 == 0:  # 每10根K线记录一次
            self.log.info(
                f"MA - 快线: {fast_value:.2f}, "
                f"慢线: {slow_value:.2f}, "
                f"收盘: {bar.close:.2f}",
                color=LogColor.BLUE
            )

    def _check_crossover(self, fast: float, slow: float, bar: Bar):
        """检查均线交叉"""
        fast_prev = self.sma_fast.value_1
        slow_prev = self.sma_slow.value_1

        # 金叉：快线上穿慢线
        if (fast > slow and fast_prev <= slow_prev and
            self.position != "LONG"):
            self._enter_long(bar)
            self.last_signal = "GOLDEN_CROSS"

        # 死叉：快线下穿慢线
        elif (fast < slow and fast_prev >= slow_prev and
              self.position == "LONG"):
            self._exit_long(bar)
            self.last_signal = "DEATH_CROSS"

    def _enter_long(self, bar: Bar):
        """开多头仓位"""
        self.position = "LONG"
        self.log.info(
            f"金叉信号 - 买入价: {bar.close:.2f}",
            color=LogColor.GREEN
        )
        # 这里应该执行实际的买入操作
        # buy_order = MarketOrder(...)
        # self.submit_order(buy_order)

    def _exit_long(self, bar: Bar):
        """平多头仓位"""
        self.position = None
        self.log.info(
            f"死叉信号 - 卖出价: {bar.close:.2f}",
            color=LogColor.RED
        )
        # 这里应该执行实际的卖出操作
        # sell_order = MarketOrder(...)
        # self.submit_order(sell_order)


# 高级移动平均线策略示例
class AdvancedMAStrategy(Strategy):
    """高级移动平均线策略"""

    def __init__(self, bar_type: BarType):
        """初始化策略"""
        super().__init__()
        self.bar_type = bar_type

        # 多种移动平均线
        self.sma_short = SMA(period=5)
        self.sma_medium = SMA(period=20)
        self.sma_long = SMA(period=50)

        # EMA用于趋势确认
        self.ema_trend = EMA(period=100)

        # 加权移动平均
        self.wma_smooth = WMA(period=10)

        # 自适应移动平均
        self.kama_fast = KAMA(
            fast_period=2,
            slow_period=30,
            period=10
        )

        # 状态变量
        self.trend_strength = 0
        self.market_state = "NEUTRAL"

    def on_start(self):
        """策略启动"""
        self.log.info("高级MA策略启动")
        self.subscribe_bars(self.bar_type)

    def on_bar(self, bar: Bar):
        """处理K线数据"""
        price = float(bar.close)

        # 更新所有指标
        self.sma_short.update_raw(price)
        self.sma_medium.update_raw(price)
        self.sma_long.update_raw(price)
        self.ema_trend.update_raw(price)
        self.wma_smooth.update_raw(price)
        self.kama_fast.update_raw(price)

        # 等待指标初始化
        if not (self.sma_long.initialized and self.ema_trend.initialized):
            return

        # 分析市场状态
        self._analyze_market_state()

        # 生成交易信号
        signal = self._generate_signal()

        if signal:
            self.log.info(
                f"交易信号: {signal['type']} "
                f"(强度: {signal['strength']:.2f})",
                color=LogColor.CYAN
            )

    def _analyze_market_state(self):
        """分析市场状态"""
        short = self.sma_short.value
        medium = self.sma_medium.value
        long = self.sma_long.value

        # 计算均线排列
        if short > medium > long:
            self.market_state = "BULLISH"
        elif short < medium < long:
            self.market_state = "BEARISH"
        else:
            self.market_state = "NEUTRAL"

        # 计算趋势强度
        spread_long_short = abs(short - long) / long
        self.trend_strength = min(spread_long_short * 10, 1.0)

    def _generate_signal(self) -> Optional[dict]:
        """生成交易信号"""
        if self.market_state == "NEUTRAL":
            return None

        # 使用KAMA作为确认
        kama_value = self.kama_fast.value
        current_price = self.sma_short.value

        # 多重确认
        conditions = []

        # 1. 市场状态
        if self.market_state == "BULLISH":
            conditions.append(kama_value > current_price * 0.995)  # 接近或高于
        elif self.market_state == "BEARISH":
            conditions.append(kama_value < current_price * 1.005)  # 接近或低于

        # 2. 趋势强度
        conditions.append(self.trend_strength > 0.3)

        # 3. EMA趋势确认
        if self.market_state == "BULLISH":
            conditions.append(self.ema_trend.value < current_price)
        elif self.market_state == "BEARISH":
            conditions.append(self.ema_trend.value > current_price)

        # 生成信号
        if all(conditions):
            return {
                'type': "BUY" if self.market_state == "BULLISH" else "SELL",
                'strength': self.trend_strength,
                'confidence': len(conditions) / 3,
            }

        return None
```

### 7.2.2 自定义移动平均

```python
"""
自定义移动平均线指标
"""

from decimal import Decimal
import math


class HullMovingAverage(Indicator):
    """赫尔移动平均线 (HMA)"""

    def __init__(self, period: int = 20):
        """
        初始化HMA

        Parameters
        ----------
        period : int
            周期
        """
        super().__init__(params={'period': period}, name="HMA")
        self.period = period
        self.half_period = int(period / 2)
        self.sqrt_period = int(math.sqrt(period))

        # 内部WMA指标
        self.wma_full = WMA(period=period)
        self.wma_half = WMA(period=self.half_period)
        self.wma_diff = WMA(period=self.sqrt_period)

        self._value = 0.0

    @property
    def value(self) -> float:
        """获取HMA值"""
        return self._value

    def handle_bar(self, bar: Bar):
        """处理K线数据"""
        self.update_raw(float(bar.close))

    def handle_quote_tick(self, quote):
        """处理报价数据"""
        mid_price = (float(quote.bid_price) + float(quote.ask_price)) / 2
        self.update_raw(mid_price)

    def handle_trade_tick(self, trade):
        """处理成交数据"""
        self.update_raw(float(trade.price))

    def reset(self):
        """重置指标"""
        self.wma_full.reset()
        self.wma_half.reset()
        self.wma_diff.reset()
        self._value = 0.0
        self._initialized = False
        self._has_inputs = False
        self._inputs = 0

    def _update(self, value: float):
        """更新HMA值"""
        # 更新内部WMA
        self.wma_full.update_raw(value)
        self.wma_half.update_raw(value)

        if self.wma_full.initialized and self.wma_half.initialized:
            # 计算2 * WMA(n/2) - WMA(n)
            diff_value = 2 * self.wma_half.value - self.wma_full.value
            self.wma_diff.update_raw(diff_value)

            if self.wma_diff.initialized:
                self._value = self.wma_diff.value
                self._initialized = True


class JurikMovingAverage(Indicator):
    """Jurik移动平均线 (JMA) - 平滑性优化的移动平均"""

    def __init__(
        self,
        period: int = 14,
        phase: float = 0.0,
        power: float = 2.0,
    ):
        """
        初始化JMA

        Parameters
        ----------
        period : int
            周期
        phase : float
            相位 (-100 to 100)
        power : float
            幂次
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

        # JMA参数
        self.phase_ratio = phase / 100.0 + 1.5
        self.beta = 0.45 * (period - 1) / (0.45 * (period - 1) + 2)
        self.alpha = self.beta

        # 内部变量
        self.prev_jma = 0.0
        self.prev_det0 = 0.0
        self.prev_det1 = 0.0

        self._value = 0.0
        self._initialized = False

    @property
    def value(self) -> float:
        """获取JMA值"""
        return self._value

    def handle_bar(self, bar: Bar):
        """处理K线数据"""
        self.update_raw(float(bar.close))

    def handle_quote_tick(self, quote):
        """处理报价数据"""
        mid_price = (float(quote.bid_price) + float(quote.ask_price)) / 2
        self.update_raw(mid_price)

    def handle_trade_tick(self, trade):
        """处理成交数据"""
        self.update_raw(float(trade.price))

    def reset(self):
        """重置指标"""
        self.prev_jma = 0.0
        self.prev_det0 = 0.0
        self.prev_det1 = 0.0
        self._value = 0.0
        self._initialized = False
        self._has_inputs = False
        self._inputs = 0

    def _update(self, value: float):
        """更新JMA值"""
        if not self.has_inputs:
            # 第一次输入
            self.prev_jma = value
            self.prev_det0 = 0.0
            self.prev_det1 = 0.0
            self._initialized = True
        else:
            # JMA计算
            det0 = (1 - self.alpha) * (value + value - self.prev_jma)
            det1 = self.alpha * self.prev_det0 + det0

            # 幂次变换
            if self.power != 1:
                abs_det1 = abs(det1) ** self.power
                if det1 < 0:
                    det1 = -abs_det1
                else:
                    det1 = abs_det1

            # 更新JMA
            self._value = self.prev_jma + self.phase_ratio * det1

            # 保存历史值
            self.prev_jma = self._value
            self.prev_det0 = det0
            self.prev_det1 = det1


# 使用自定义MA的策略
class CustomMAStrategy(Strategy):
    """使用自定义移动平均线的策略"""

    def __init__(self, bar_type: BarType):
        """初始化策略"""
        super().__init__()
        self.bar_type = bar_type

        # 自定义移动平均线
        self.hma_fast = HullMovingAverage(period=10)
        self.hma_slow = HullMovingAverage(period=30)
        self.jma_smooth = JurikMovingAverage(period=20, phase=0)

        # 状态跟踪
        self.hma_spread = 0.0
        self.jma_trend = 0.0

    def on_start(self):
        """策略启动"""
        self.log.info("自定义MA策略启动")
        self.subscribe_bars(self.bar_type)

    def on_bar(self, bar: Bar):
        """处理K线数据"""
        price = float(bar.close)

        # 更新指标
        self.hma_fast.update_raw(price)
        self.hma_slow.update_raw(price)
        self.jma_smooth.update_raw(price)

        # 等待初始化
        if not (self.hma_fast.initialized and
                self.hma_slow.initialized and
                self.jma_smooth.initialized):
            return

        # 计算指标差异
        self.hma_spread = (self.hma_fast.value - self.hma_slow.value) / self.hma_slow.value
        self.jma_trend = self.jma_smooth.value - self.jma_smooth.value_1

        # 生成信号
        self._check_signals(bar)

    def _check_signals(self, bar: Bar):
        """检查交易信号"""
        # HMA交叉信号
        hma_cross = (
            (self.hma_fast.value > self.hma_slow.value) and
            (self.hma_fast.value_1 <= self.hma_slow.value_1)
        ) or (
            (self.hma_fast.value < self.hma_slow.value) and
            (self.hma_fast.value_1 >= self.hma_slow.value_1)
        )

        # JMA趋势确认
        trend_confirm = abs(self.jma_trend) > 0.001

        if hma_cross and trend_confirm:
            signal = "BUY" if self.hma_fast.value > self.hma_slow.value else "SELL"
            self.log.info(
                f"信号: {signal}, HMA价差: {self.hma_spread:.4f}, "
                f"JMA趋势: {self.jma_trend:.4f}",
                color=LogColor.YELLOW
            )
```

## 7.3 动量指标

### 7.3.1 RSI 指标

```python
"""
RSI (相对强弱指数) 使用示例
"""

from decimal import Decimal
from nautilus_trader.indicators.momentum import RSI


class RSIStrategy(Strategy):
    """RSI策略"""

    def __init__(
        self,
        bar_type: BarType,
        rsi_period: int = 14,
        oversold: float = 30,
        overbought: float = 70,
    ):
        """
        初始化RSI策略

        Parameters
        ----------
        bar_type : BarType
            K线类型
        rsi_period : int
            RSI周期
        oversold : float
            超卖线
        overbought : float
            超买线
        """
        super().__init__()
        self.bar_type = bar_type

        # 创建RSI指标
        self.rsi = RSI(period=rsi_period)

        # 参数
        self.oversold = oversold
        self.overbought = overbought

        # 状态
        self.last_rsi = 0.0
        self.divergences = []

    def on_start(self):
        """策略启动"""
        self.log.info(
            f"RSI策略启动 - 周期: {self.rsi.period}, "
            f"超卖: {self.oversold}, 超买: {self.overbought}"
        )
        self.subscribe_bars(self.bar_type)

    def on_bar(self, bar: Bar):
        """处理K线数据"""
        # 更新RSI
        self.rsi.update_raw(float(bar.close))

        if not self.rsi.initialized:
            return

        current_rsi = self.rsi.value
        price = float(bar.close)

        # 检查RSI信号
        self._check_rsi_signals(current_rsi, price)

        # 检查背离
        self._check_divergences(current_rsi, price)

        # 记录RSI值
        self.last_rsi = current_rsi

        # 定期输出
        if self.count_bars() % 20 == 0:
            self.log.info(
                f"RSI: {current_rsi:.2f}, 价格: {price:.2f}",
                color=LogColor.MAGENTA
            )

    def _check_rsi_signals(self, rsi: float, price: float):
        """检查RSI信号"""
        # 超卖区域反弹
        if self.last_rsi < self.oversold and rsi > self.oversold:
            self.log.info(
                f"RSI超卖反弹 - RSI: {rsi:.2f}",
                color=LogColor.GREEN
            )
            self._generate_buy_signal(rsi, price, "OVERSOLD_RECOVERY")

        # 超买区域回调
        elif self.last_rsi > self.overbought and rsi < self.overbought:
            self.log.info(
                f"RSI超买回调 - RSI: {rsi:.2f}",
                color=LogColor.RED
            )
            self._generate_sell_signal(rsi, price, "OVERBOUGHT_PULLBACK")

        # 中轴突破
        elif self.last_rsi < 50 and rsi > 50:
            self.log.info(
                f"RSI突破中轴 - RSI: {rsi:.2f}",
                color=LogColor.BLUE
            )
            self._generate_buy_signal(rsi, price, "CENTER_CROSS_UP")

        elif self.last_rsi > 50 and rsi < 50:
            self.log.info(
                f"RSI跌破中轴 - RSI: {rsi:.2f}",
                color=LogColor.ORANGE
            )
            self._generate_sell_signal(rsi, price, "CENTER_CROSS_DOWN")

    def _check_divergences(self, rsi: float, price: float):
        """检查背离"""
        # 需要维护价格和RSI的历史数据
        # 这里简化实现
        pass

    def _generate_buy_signal(self, rsi: float, price: float, reason: str):
        """生成买入信号"""
        signal = {
            'type': 'BUY',
            'rsi': rsi,
            'price': price,
            'reason': reason,
            'strength': self._calculate_signal_strength(rsi, 'BUY')
        }
        self.log.info(f"买入信号: {signal}", color=LogColor.GREEN)

    def _generate_sell_signal(self, rsi: float, price: float, reason: str):
        """生成卖出信号"""
        signal = {
            'type': 'SELL',
            'rsi': rsi,
            'price': price,
            'reason': reason,
            'strength': self._calculate_signal_strength(rsi, 'SELL')
        }
        self.log.info(f"卖出信号: {signal}", color=LogColor.RED)

    def _calculate_signal_strength(self, rsi: float, signal_type: str) -> float:
        """计算信号强度"""
        if signal_type == 'BUY':
            # 超卖程度越深，信号越强
            return min((50 - rsi) / 20, 1.0)
        else:
            # 超买程度越高，信号越强
            return min((rsi - 50) / 20, 1.0)


# RSI + MA 组合策略
class RSIWithMAStrategy(Strategy):
    """RSI与移动平均线组合策略"""

    def __init__(self, bar_type: BarType):
        """初始化策略"""
        super().__init__()
        self.bar_type = bar_type

        # 指标
        self.rsi = RSI(period=14)
        self.ema_fast = EMA(period=20)
        self.ema_slow = EMA(period=50)

        # 状态
        self.trend = "NEUTRAL"
        self.rsi_state = "NEUTRAL"

    def on_start(self):
        """策略启动"""
        self.log.info("RSI+MA组合策略启动")
        self.subscribe_bars(self.bar_type)

    def on_bar(self, bar: Bar):
        """处理K线数据"""
        price = float(bar.close)

        # 更新指标
        self.rsi.update_raw(price)
        self.ema_fast.update_raw(price)
        self.ema_slow.update_raw(price)

        if not (self.rsi.initialized and
                self.ema_fast.initialized and
                self.ema_slow.initialized):
            return

        # 判断趋势
        self._determine_trend()

        # 判断RSI状态
        self._determine_rsi_state()

        # 生成信号
        self._generate_combined_signal(price)

    def _determine_trend(self):
        """判断趋势"""
        if self.ema_fast.value > self.ema_slow.value:
            self.trend = "BULLISH"
        else:
            self.trend = "BEARISH"

    def _determine_rsi_state(self):
        """判断RSI状态"""
        if self.rsi.value < 30:
            self.rsi_state = "OVERSOLD"
        elif self.rsi.value > 70:
            self.rsi_state = "OVERBOUGHT"
        else:
            self.rsi_state = "NEUTRAL"

    def _generate_combined_signal(self, price: float):
        """生成组合信号"""
        # 牛市 + RSI超卖 = 买入
        if (self.trend == "BULLISH" and
            self.rsi_state == "OVERSOLD"):
            self.log.info(
                f"组合买入信号 - 趋势: {self.trend}, RSI: {self.rsi.value:.1f}",
                color=LogColor.GREEN
            )

        # 熊市 + RSI超买 = 卖出
        elif (self.trend == "BEARISH" and
              self.rsi_state == "OVERBOUGHT"):
            self.log.info(
                f"组合卖出信号 - 趋势: {self.trend}, RSI: {self.rsi.value:.1f}",
                color=LogColor.RED
            )
```

### 7.3.2 MACD 指标

```python
"""
MACD (移动平均收敛散度) 使用示例
"""

from nautilus_trader.indicators.momentum import MACD


class MACDStrategy(Strategy):
    """MACD策略"""

    def __init__(
        self,
        bar_type: BarType,
        fast_period: int = 12,
        slow_period: int = 26,
        signal_period: int = 9,
    ):
        """
        初始化MACD策略

        Parameters
        ----------
        bar_type : BarType
            K线类型
        fast_period : int
            快线周期
        slow_period : int
            慢线周期
        signal_period : int
            信号线周期
        """
        super().__init__()
        self.bar_type = bar_type

        # 创建MACD指标
        self.macd = MACD(
            fast_period=fast_period,
            slow_period=slow_period,
            signal_period=signal_period,
        )

        # 状态
        self.last_signal = 0  # MACD柱状图的符号
        self.histogram_trend = []  # 柱状图趋势

    def on_start(self):
        """策略启动"""
        self.log.info(
            f"MACD策略启动 - "
            f"快线: {self.macd.fast_period}, "
            f"慢线: {self.macd.slow_period}, "
            f"信号: {self.macd.signal_period}"
        )
        self.subscribe_bars(self.bar_type)

    def on_bar(self, bar: Bar):
        """处理K线数据"""
        # 更新MACD
        self.macd.update_raw(float(bar.close))

        if not self.macd.initialized:
            return

        # 获取MACD值
        macd_line = self.macd.value
        signal_line = self.macd.signal
        histogram = self.macd.histogram

        # 检查信号
        self._check_macd_signals(macd_line, signal_line, histogram, bar)

        # 记录数值
        if self.count_bars() % 10 == 0:
            self.log.info(
                f"MACD - 线: {macd_line:.4f}, "
                f"信号: {signal_line:.4f}, "
                f"柱: {histogram:.4f}",
                color=LogColor.CYAN
            )

    def _check_macd_signals(
        self,
        macd_line: float,
        signal_line: float,
        histogram: float,
        bar: Bar
    ):
        """检查MACD信号"""
        # 记录柱状图趋势
        self.histogram_trend.append(histogram)
        if len(self.histogram_trend) > 3:
            self.histogram_trend.pop(0)

        # 1. MACD线与信号线交叉
        if (macd_line > signal_line and self.last_signal <= 0):
            self._on_macd_bull_cross(bar)
        elif (macd_line < signal_line and self.last_signal >= 0):
            self._on_macd_bear_cross(bar)

        # 2. 柱状图背离
        if len(self.histogram_trend) >= 3:
            self._check_histogram_divergence(bar)

        # 3. 零轴突破
        if self.last_signal * histogram < 0:  # 符号变化
            if histogram > 0:
                self._on_zero_cross_up(bar)
            else:
                self._on_zero_cross_down(bar)

        self.last_signal = histogram

    def _on_macd_bull_cross(self, bar: Bar):
        """MACD牛市交叉"""
        self.log.info(
            f"MACD牛市交叉 - 价格: {bar.close:.2f}",
            color=LogColor.GREEN
        )

    def _on_macd_bear_cross(self, bar: Bar):
        """MACD熊市交叉"""
        self.log.info(
            f"MACD熊市交叉 - 价格: {bar.close:.2f}",
            color=LogColor.RED
        )

    def _on_zero_cross_up(self, bar: Bar):
        """MACD上穿零轴"""
        self.log.info(
            f"MACD上穿零轴 - 价格: {bar.close:.2f}",
            color=LogColor.BLUE
        )

    def _on_zero_cross_down(self, bar: Bar):
        """MACD下穿零轴"""
        self.log.info(
            f"MACD下穿零轴 - 价格: {bar.close:.2f}",
            color=LogColor.ORANGE
        )

    def _check_histogram_divergence(self, bar: Bar):
        """检查柱状图背离"""
        # 简化的背离检测
        h0, h1, h2 = self.histogram_trend[-3:]
        current_price = float(bar.close)

        # 看涨背离：价格新低但柱状图谷底抬高
        if (h2 < h1 < h0 and h0 < 0):  # 上升的谷底
            # 需要结合价格数据进行完整实现
            pass

        # 看跌背离：价格新高但柱状图峰顶降低
        elif (h2 > h1 > h0 and h0 > 0):  # 下降的峰顶
            # 需要结合价格数据进行完整实现
            pass
```

## 7.4 趋势和波动率指标

### 7.4.1 布林带 (Bollinger Bands)

```python
"""
布林带使用示例
"""

from nautilus_trader.indicators.volatility import BollingerBands
from nautilus_trader.indicators.volatility import KeltnerChannels
from nautilus_trader.indicators.volatility import DonchianChannels


class BollingerBandsStrategy(Strategy):
    """布林带策略"""

    def __init__(
        self,
        bar_type: BarType,
        period: int = 20,
        std_dev: float = 2.0,
    ):
        """
        初始化布林带策略

        Parameters
        ----------
        bar_type : BarType
            K线类型
        period : int
            均线周期
        std_dev : float
            标准差倍数
        """
        super().__init__()
        self.bar_type = bar_type

        # 布林带指标
        self.bbands = BollingerBands(
            period=period,
            std_dev=std_dev,
        )

        # 辅助指标
        self.keltner = KeltnerChannels(
            period=period,
            multiplier=2.0,
        )
        self.donchian = DonchianChannels(period=period)

        # 状态
        self.price_position = "MIDDLE"  # UPPER, MIDDLE, LOWER
        self.band_width = 0.0
        self.squeeze_detected = False

    def on_start(self):
        """策略启动"""
        self.log.info(
            f"布林带策略启动 - "
            f"周期: {self.bbands.period}, "
            f"标准差: {self.bbands.std_dev}"
        )
        self.subscribe_bars(self.bar_type)

    def on_bar(self, bar: Bar):
        """处理K线数据"""
        price = float(bar.close)
        high = float(bar.high)
        low = float(bar.low)

        # 更新指标
        self.bbands.update_raw(price)
        self.keltner.update_raw(price)
        self.donchian.update_high(high)
        self.donchian.update_low(low)

        if not (self.bbands.initialized and
                self.keltner.initialized and
                self.donchian.initialized):
            return

        # 获取布林带值
        bb_upper = self.bbands.upper
        bb_middle = self.bbands.middle
        bb_lower = self.bbands.lower

        # 计算价格位置
        if price > bb_upper:
            self.price_position = "ABOVE_UPPER"
        elif price < bb_lower:
            self.price_position = "BELOW_LOWER"
        elif price > bb_middle:
            self.price_position = "UPPER_HALF"
        else:
            self.price_position = "LOWER_HALF"

        # 计算带宽
        self.band_width = (bb_upper - bb_lower) / bb_middle

        # 检测挤压
        self._detect_squeeze()

        # 生成信号
        self._check_signals(price, bar)

        # 记录信息
        if self.count_bars() % 10 == 0:
            self.log.info(
                f"BB - 上轨: {bb_upper:.2f}, "
                f"中轨: {bb_middle:.2f}, "
                f"下轨: {bb_lower:.2f}, "
                f"价格: {price:.2f}",
                color=LogColor.MAGENTA
            )

    def _detect_squeeze(self):
        """检测布林带挤压"""
        # 布林带宽度小于某个阈值
        min_width = 0.02  # 2%

        if self.band_width < min_width:
            if not self.squeeze_detected:
                self.squeeze_detected = True
                self.log.warning(
                    f"布林带挤压检测 - 带宽: {self.band_width:.4f}",
                    color=LogColor.YELLOW
                )
        else:
            if self.squeeze_detected:
                self.squeeze_detected = False
                self.log.info("布林带挤压释放", color=LogColor.CYAN)

    def _check_signals(self, price: float, bar: Bar):
        """检查交易信号"""
        bb_upper = self.bbands.upper
        bb_lower = self.bbands.lower

        # 回归交易（均值回归）
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

        # 突破交易
        elif self.price_position == "UPPER_HALF":
            # 检查是否突破
            if bar.close > bar.open and self.band_width > 0.04:
                self._generate_buy_signal(
                    price=price,
                    reason="UPPER_BAND_BREAKOUT",
                    strength=self._calculate_strength("breakout")
                )
        elif self.price_position == "LOWER_HALF":
            # 检查是否跌破
            if bar.close < bar.open and self.band_width > 0.04:
                self._generate_sell_signal(
                    price=price,
                    reason="LOWER_BAND_BREAKOUT",
                    strength=self._calculate_strength("breakout")
                )

    def _calculate_strength(self, signal_type: str) -> float:
        """计算信号强度"""
        if signal_type == "mean_reversion":
            # 带宽越窄，回归信号越强
            return max(1.0 - self.band_width * 10, 0.3)
        elif signal_type == "breakout":
            # 带宽越宽，突破信号越强
            return min(self.band_width * 15, 1.0)
        return 0.5

    def _generate_buy_signal(self, price: float, reason: str, strength: float):
        """生成买入信号"""
        self.log.info(
            f"买入信号 - 价格: {price:.2f}, "
            f"原因: {reason}, 强度: {strength:.2f}",
            color=LogColor.GREEN
        )

    def _generate_sell_signal(self, price: float, reason: str, strength: float):
        """生成卖出信号"""
        self.log.info(
            f"卖出信号 - 价格: {price:.2f}, "
            f"原因: {reason}, 强度: {strength:.2f}",
            color=LogColor.RED
        )
```

## 7.5 自定义指标开发

### 7.5.1 创建复合指标

```python
"""
自定义复合指标示例
"""

from typing import Optional
import numpy as np


class AdaptiveRSI(Indicator):
    """自适应RSI指标"""

    def __init__(
        self,
        base_period: int = 14,
        min_period: int = 5,
        max_period: int = 30,
        sensitivity: float = 0.02,
    ):
        """
        初始化自适应RSI

        Parameters
        ----------
        base_period : int
            基础周期
        min_period : int
            最小周期
        max_period : int
            最大周期
        sensitivity : float
            敏感度
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

        # 内部RSI
        self.rsi = RSI(period=base_period)

        # 波动率计算
        self.price_changes = []
        self.volatility = 0.0

        # 自适应周期
        self.current_period = base_period

        self._value = 0.0

    @property
    def value(self) -> float:
        """获取指标值"""
        return self._value

    @property
    def adaptive_period(self) -> int:
        """获取自适应周期"""
        return self.current_period

    def handle_bar(self, bar: Bar):
        """处理K线数据"""
        self.update_raw(float(bar.close))

    def handle_quote_tick(self, quote):
        """处理报价数据"""
        mid_price = (float(quote.bid_price) + float(quote.ask_price)) / 2
        self.update_raw(mid_price)

    def handle_trade_tick(self, trade):
        """处理成交数据"""
        self.update_raw(float(trade.price))

    def reset(self):
        """重置指标"""
        self.rsi.reset()
        self.price_changes.clear()
        self.volatility = 0.0
        self.current_period = self.base_period
        self._value = 0.0
        self._initialized = False
        self._has_inputs = False
        self._inputs = 0

    def _update(self, value: float):
        """更新指标值"""
        # 计算价格变化
        if self.has_inputs:
            price_change = abs(value - self.price_changes[-1])
            self.price_changes.append(price_change)

            # 保留足够的历史数据
            if len(self.price_changes) > self.max_period * 2:
                self.price_changes.pop(0)

            # 计算波动率
            if len(self.price_changes) >= self.base_period:
                self.volatility = np.std(self.price_changes[-self.base_period:])

                # 自适应调整周期
                self._adjust_period()

        else:
            self.price_changes.append(value)

        # 更新内部RSI
        self.rsi.update_raw(value)

        # 调整RSI周期
        if self.rsi.period != self.current_period:
            self.rsi = RSI(period=self.current_period)
            # 重新计算历史数据
            for price in self.price_changes[-self.current_period:]:
                self.rsi.update_raw(price)

        self._value = self.rsi.value

        if self.rsi.initialized:
            self._initialized = True

    def _adjust_period(self):
        """调整RSI周期"""
        # 基于波动率调整周期
        # 高波动率 -> 短周期
        # 低波动率 -> 长周期

        volatility_ratio = self.volatility / (np.mean(self.price_changes) + 1e-8)

        # 计算新周期
        period_adjustment = int(volatility_ratio * self.sensitivity * self.base_period)
        new_period = self.base_period - period_adjustment

        # 限制周期范围
        self.current_period = max(self.min_period, min(self.max_period, new_period))


class MultiTimeframeRSI(Indicator):
    """多时间框架RSI指标"""

    def __init__(
        self,
        short_period: int = 7,
        medium_period: int = 14,
        long_period: int = 28,
    ):
        """
        初始化多时间框架RSI

        Parameters
        ----------
        short_period : int
            短周期
        medium_period : int
            中周期
        long_period : int
            长周期
        """
        super().__init__(
            params={
                'short_period': short_period,
                'medium_period': medium_period,
                'long_period': long_period
            },
            name="MultiTimeframeRSI"
        )

        # 三个不同周期的RSI
        self.rsi_short = RSI(period=short_period)
        self.rsi_medium = RSI(period=medium_period)
        self.rsi_long = RSI(period=long_period)

        # 综合信号
        self._value = 50.0
        self.signal_strength = 0.0

    @property
    def value(self) -> float:
        """获取综合RSI值"""
        return self._value

    @property
    def strength(self) -> float:
        """获取信号强度"""
        return self.signal_strength

    @property
    def rsi_values(self) -> dict:
        """获取各周期RSI值"""
        return {
            'short': self.rsi_short.value if self.rsi_short.initialized else None,
            'medium': self.rsi_medium.value if self.rsi_medium.initialized else None,
            'long': self.rsi_long.value if self.rsi_long.initialized else None,
        }

    def handle_bar(self, bar: Bar):
        """处理K线数据"""
        self.update_raw(float(bar.close))

    def handle_quote_tick(self, quote):
        """处理报价数据"""
        mid_price = (float(quote.bid_price) + float(quote.ask_price)) / 2
        self.update_raw(mid_price)

    def handle_trade_tick(self, trade):
        """处理成交数据"""
        self.update_raw(float(trade.price))

    def reset(self):
        """重置指标"""
        self.rsi_short.reset()
        self.rsi_medium.reset()
        self.rsi_long.reset()
        self._value = 50.0
        self.signal_strength = 0.0
        self._initialized = False
        self._has_inputs = False
        self._inputs = 0

    def _update(self, value: float):
        """更新指标值"""
        # 更新所有RSI
        self.rsi_short.update_raw(value)
        self.rsi_medium.update_raw(value)
        self.rsi_long.update_raw(value)

        # 等待所有RSI初始化
        if not (self.rsi_short.initialized and
                self.rsi_medium.initialized and
                self.rsi_long.initialized):
            return

        # 计算加权平均
        weights = [0.5, 0.3, 0.2]  # 短期权重最大
        rsi_values = [
            self.rsi_short.value,
            self.rsi_medium.value,
            self.rsi_long.value
        ]

        self._value = sum(w * r for w, r in zip(weights, rsi_values))

        # 计算信号强度（基于RSI的一致性）
        rsi_mean = np.mean(rsi_values)
        rsi_std = np.std(rsi_values)
        self.signal_strength = max(0, 1 - rsi_std / 50)  # 标准化到0-1

        self._initialized = True
```

## 7.6 性能优化

### 7.6.1 批量处理和缓存

```python
"""
指标性能优化
"""

from typing import List, Tuple
import numpy as np
from functools import lru_cache


class OptimizedIndicator(Indicator):
    """优化后的指标基类"""

    def __init__(self, batch_size: int = 100):
        """
        初始化优化指标

        Parameters
        ----------
        batch_size : int
            批处理大小
        """
        super().__init__()
        self.batch_size = batch_size
        self.buffer = []
        self._cache_enabled = True
        self._update_count = 0

    @lru_cache(maxsize=1024)
    def _cached_calculation(self, value: float) -> float:
        """缓存的计算（适用于纯函数）"""
        # 子类实现具体的计算逻辑
        return value

    def update_batch(self, values: List[float]):
        """
        批量更新

        Parameters
        ----------
        values : List[float]
            值列表
        """
        self.buffer.extend(values)

        # 达到批处理大小时进行批量处理
        if len(self.buffer) >= self.batch_size:
            self._process_batch()
            self.buffer.clear()

    def _process_batch(self):
        """处理批量数据"""
        # 使用numpy进行向量化计算
        np_values = np.array(self.buffer)

        # 子类实现具体的批量处理逻辑
        self._batch_update(np_values)

    def _batch_update(self, values: np.ndarray):
        """批量更新（子类实现）"""
        pass


class OptimizedEMA(OptimizedIndicator):
    """优化的EMA指标"""

    def __init__(
        self,
        period: int = 20,
        batch_size: int = 100,
        cache_size: int = 1024,
    ):
        """
        初始化优化的EMA

        Parameters
        ----------
        period : int
            EMA周期
        batch_size : int
            批处理大小
        cache_size : int
            缓存大小
        """
        super().__init__(batch_size= batch_size)
        self.period = period
        self.alpha = 2.0 / (period + 1)
        self._value = 0.0
        self._initialized = False

        # 启用缓存
        self._cache_enabled = cache_size > 0
        if self._cache_enabled:
            self._cached_calculation = lru_cache(maxsize=cache_size)(self._calculate_ema)

    @property
    def value(self) -> float:
        """获取EMA值"""
        return self._value

    def handle_bar(self, bar: Bar):
        """处理K线数据"""
        self.update_raw(float(bar.close))

    def handle_quote_tick(self, quote):
        """处理报价数据"""
        mid_price = (float(quote.bid_price) + float(quote.ask_price)) / 2
        self.update_raw(mid_price)

    def handle_trade_tick(self, trade):
        """处理成交数据"""
        self.update_raw(float(trade.price))

    def reset(self):
        """重置指标"""
        self._value = 0.0
        self._initialized = False
        self.buffer.clear()
        self._update_count = 0

        # 清除缓存
        if hasattr(self._cached_calculation, 'cache_clear'):
            self._cached_calculation.cache_clear()

    def _update(self, value: float):
        """更新EMA值"""
        if self._cache_enabled:
            # 使用缓存的计算
            self._value = self._cached_calculation(value)
        else:
            # 直接计算
            self._value = self._calculate_ema(value)

        if not self._initialized:
            self._initialized = True

        self._update_count += 1

    def _calculate_ema(self, value: float) -> float:
        """计算EMA值"""
        if self._update_count == 0:
            # 第一个值
            return value
        else:
            # EMA公式
            return (value * self.alpha) + (self._value * (1 - self.alpha))

    def _batch_update(self, values: np.ndarray):
        """批量更新EMA"""
        # 使用numpy进行向量化计算
        for value in values:
            self._update(float(value))


class IndicatorManager:
    """指标管理器"""

    def __init__(self):
        """初始化管理器"""
        self.indicators = {}
        self.update_queue = []

    def add_indicator(self, name: str, indicator: Indicator):
        """
        添加指标

        Parameters
        ----------
        name : str
            指标名称
        indicator : Indicator
            指标实例
        """
        self.indicators[name] = indicator

    def update_all(self, value: float):
        """
        更新所有指标

        Parameters
        ----------
        value : float
            更新值
        """
        for indicator in self.indicators.values():
            indicator.update_raw(value)

    def update_indicators(self, updates: List[Tuple[str, float]]):
        """
        更新指定指标

        Parameters
        ----------
        updates : List[Tuple[str, float]]
            (指标名称, 值) 列表
        """
        for name, value in updates:
            if name in self.indicators:
                self.indicators[name].update_raw(value)

    def get_values(self) -> dict:
        """获取所有指标的值"""
        values = {}
        for name, indicator in self.indicators.items():
            if indicator.initialized:
                values[name] = indicator.value
        return values

    def reset_all(self):
        """重置所有指标"""
        for indicator in self.indicators.values():
            indicator.reset()

    def get_initialized_indicators(self) -> List[str]:
        """获取已初始化的指标列表"""
        return [
            name for name, indicator in self.indicators.items()
            if indicator.initialized
        ]


# 性能分析工具
class IndicatorProfiler:
    """指标性能分析器"""

    def __init__(self):
        """初始化分析器"""
        self.timings = {}
        self.call_counts = {}

    def profile_update(self, indicator_name: str, func, *args, **kwargs):
        """
        分析指标更新

        Parameters
        ----------
        indicator_name : str
            指标名称
        func : Callable
            更新函数
        *args, **kwargs : Any
            函数参数
        """
        import time

        start_time = time.perf_counter()
        result = func(*args, **kwargs)
        end_time = time.perf_counter()

        elapsed = (end_time - start_time) * 1000  # 转换为毫秒

        # 记录时间
        if indicator_name not in self.timings:
            self.timings[indicator_name] = []
        self.timings[indicator_name].append(elapsed)

        # 记录调用次数
        self.call_counts[indicator_name] = self.call_counts.get(indicator_name, 0) + 1

        return result

    def get_stats(self) -> dict:
        """获取性能统计"""
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
        """找出慢速指标"""
        slow_indicators = []
        stats = self.get_stats()

        for name, stat in stats.items():
            if stat['avg_ms'] > threshold_ms:
                slow_indicators.append(name)

        return slow_indicators


# 使用示例
def main():
    """性能优化示例"""
    from time import time

    # 创建指标管理器
    manager = IndicatorManager()

    # 添加指标
    manager.add_indicator("EMA_20", OptimizedEMA(period=20, batch_size=100))
    manager.add_indicator("RSI_14", RSI(period=14))
    manager.add_indicator("AdaptiveRSI", AdaptiveRSI(base_period=14))

    # 创建性能分析器
    profiler = IndicatorProfiler()

    # 模拟数据更新
    data = [100 + i * 0.1 + (i % 10) for i in range(1000)]

    # 测试性能
    start_time = time()
    for i, value in enumerate(data):
        # 使用分析器包装更新
        for name in ["EMA_20", "RSI_14", "AdaptiveRSI"]:
            indicator = manager.indicators[name]
            profiler.profile_update(name, indicator.update_raw, value)

    end_time = time()

    # 输出结果
    print(f"更新 {len(data)} 个数据点耗时: {end_time - start_time:.3f} 秒")

    # 性能统计
    stats = profiler.get_stats()
    print("\n指标性能统计:")
    for name, stat in stats.items():
        print(f"{name}:")
        print(f"  调用次数: {stat['calls']}")
        print(f"  平均耗时: {stat['avg_ms']:.3f} ms")
        print(f"  最大耗时: {stat['max_ms']:.3f} ms")

    # 找出慢速指标
    slow_indicators = profiler.find_slow_indicators()
    if slow_indicators:
        print(f"\n慢速指标 (>1ms): {slow_indicators}")
    else:
        print("\n所有指标性能良好")


if __name__ == "__main__":
    main()
```

## 7.7 下一步

在本章中，我们学习了：

1. NautilusTrader 技术指标系统的架构
2. 内置指标的使用方法
3. 移动平均线、动量、趋势指标的应用
4. 自定义指标开发
5. 性能优化技巧

在下一章中，我们将学习：

1. 订单管理系统
2. 各种订单类型的使用
3. 订单状态管理
4. 执行算法
5. 风险控制

## 7.8 总结

### 关键要点
1. NautilusTrader 提供了丰富的内置指标库
2. 指标系统采用统一的接口设计
3. 支持自定义指标和复合指标
4. 提供性能优化工具和批量处理能力

### 最佳实践
1. 合理选择指标周期，避免过度优化
2. 使用指标管理器统一管理多个指标
3. 定期分析指标性能，找出瓶颈
4. 结合多个指标进行信号确认

## 7.9 参考资料

- [技术指标 API 文档](https://nautilustrader.io/docs/latest/api_reference/indicators/)
- [量化交易常用指标](https://www.investopedia.com/terms/t/technicalindicator.asp)
- [Python 性能优化指南](https://docs.python.org/3/library/profile.html)