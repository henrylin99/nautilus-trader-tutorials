# 第16章：算法交易策略

## 学习目标

- 掌握常见算法交易策略的原理和实现
- 学会使用NautilusTrader构建复杂的交易算法
- 理解不同市场微观结构下的策略设计
- 实现高性能、低延迟的交易策略

## 16.1 做市策略

### 16.1.1 基础做市策略

```python
from nautilus_trader.core.message import Event
from nautilus_trader.model.data import QuoteTick
from nautilus_trader.model.enums import OrderSide, OrderType
from nautilus_trader.model.objects import Price, Quantity
from nautilus_trader.strategy import Strategy
from decimal import Decimal
import numpy as np

class MarketMakingStrategy(Strategy):
    """基础做市策略"""

    def __init__(self, config):
        super().__init__()
        self.symbol = config['symbol']
        self.spread_bps = config['spread_bps']  # 价差(基点)
        self.order_size = config['order_size']  # 订单大小
        self.max_position = config['max_position']  # 最大持仓
        self.inventory_target = config.get('inventory_target', 0)  # 目标持仓
        self.rebalance_threshold = config.get('rebalance_threshold', 0.1)  # 再平衡阈值

        # 状态变量
        self.current_position = 0
        self.mid_price = None
        self.current_orders = {}  # 活跃订单
        self.last_quote_time = None

    def on_start(self):
        """策略启动时调用"""
        self.subscribe_quote_ticks(self.symbol)
        self.log.info(f"Market making strategy started for {self.symbol}")

    def on_quote_tick(self, tick: QuoteTick):
        """处理报价数据"""
        # 更新中间价
        self.mid_price = (tick.bid + tick.ask) / 2
        self.last_quote_time = tick.ts_event

        # 获取当前持仓
        self._update_position()

        # 取消现有订单
        await self._cancel_all_orders()

        # 计算新订单
        bid_price, ask_price, bid_size, ask_size = self._calculate_quotes()

        # 下新单
        if bid_price and bid_size > 0:
            await self._place_order(OrderSide.BUY, bid_price, bid_size)
        if ask_price and ask_size > 0:
            await self._place_order(OrderSide.SELL, ask_price, ask_size)

    def _update_position(self):
        """更新当前持仓"""
        portfolio = self.portfolio
        position = portfolio.position(self.symbol)
        if position:
            self.current_position = position.quantity

    def _calculate_quotes(self):
        """计算买卖报价"""
        if not self.mid_price:
            return None, None, None, None

        # 基础价差
        spread = self.mid_price * self.spread_bps / 10000

        # 根据持仓调整价差和报价
        inventory_skew = self._calculate_inventory_skew()

        # 计算买卖价格
        bid_price = self.mid_price - spread * (1 + inventory_skew)
        ask_price = self.mid_price + spread * (1 - inventory_skew)

        # 计算订单大小
        bid_size, ask_size = self._calculate_order_sizes(inventory_skew)

        return bid_price, ask_price, bid_size, ask_size

    def _calculate_inventory_skew(self) -> float:
        """计算库存偏斜度"""
        if self.max_position == 0:
            return 0

        # 当前持仓相对于目标持仓的比例
        inventory_ratio = (self.current_position - self.inventory_target) / self.max_position

        # 限制在[-1, 1]范围内
        inventory_skew = max(-1, min(1, inventory_ratio))

        return inventory_skew

    def _calculate_order_sizes(self, inventory_skew: float):
        """计算订单大小"""
        # 基础订单大小
        base_size = self.order_size

        # 根据库存偏斜调整大小
        if inventory_skew > 0:  # 持仓过多，减少买单，增加卖单
            bid_size = base_size * (1 - abs(inventory_skew))
            ask_size = base_size * (1 + abs(inventory_skew))
        elif inventory_skew < 0:  # 持仓过少，增加买单，减少卖单
            bid_size = base_size * (1 + abs(inventory_skew))
            ask_size = base_size * (1 - abs(inventory_skew))
        else:  # 库存平衡
            bid_size = ask_size = base_size

        # 确保不超过最大持仓限制
        if self.current_position + bid_size > self.max_position:
            bid_size = max(0, self.max_position - self.current_position)
        if self.current_position - ask_size < -self.max_position:
            ask_size = max(0, self.current_position + self.max_position)

        return bid_size, ask_size

    async def _place_order(self, side: OrderSide, price: Price, size: Quantity):
        """下单"""
        order = self.order_factory.market(
            symbol=self.symbol,
            side=side,
            quantity=size
        )

        # 修改为限价单
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
        """取消所有订单"""
        for order_id in list(self.current_orders.keys()):
            await self.cancel_order(order_id)
            del self.current_orders[order_id]

    def on_order_filled(self, event):
        """订单成交通知"""
        self.log.info(f"Order filled: {event}")
        # 更新持仓会在下次tick时进行

    def on_order_rejected(self, event):
        """订单被拒通知"""
        self.log.warning(f"Order rejected: {event}")
        if event.order_id in self.current_orders:
            del self.current_orders[event.order_id]

    def on_order_canceled(self, event):
        """订单取消通知"""
        if event.order_id in self.current_orders:
            del self.current_orders[event.order_id]

    def on_stop(self):
        """策略停止时调用"""
        # 取消所有订单
        asyncio.create_task(self._cancel_all_orders())
        self.log.info("Market making strategy stopped")
```

### 16.1.2 高频做市策略

```python
import time
from numba import jit
from collections import deque

class HFTMarketMakingStrategy(Strategy):
    """高频做市策略"""

    def __init__(self, config):
        super().__init__()
        self.symbol = config['symbol']
        self.order_size = config['order_size']
        self.max_orders_per_side = config.get('max_orders_per_side', 5)
        self.min_spread_bps = config.get('min_spread_bps', 1)
        self.max_spread_bps = config.get('max_spread_bps', 10)
        self.volatility_window = config.get('volatility_window', 100)
        self.microprice_alpha = config.get('microprice_alpha', 0.2)

        # 高性能数据结构
        self.price_history = deque(maxlen=self.volatility_window)
        self.order_book_levels = 10
        self.current_orders = {}
        self.last_microprice = None
        self.volatility = 0

        # 性能优化
        self._setup_jit_functions()

    def _setup_jit_functions(self):
        """设置JIT加速函数"""
        # 使用numba加速关键计算
        self.calculate_microprice = jit(self._calculate_microprice_impl)
        self.calculate_spread = jit(self._calculate_spread_impl)

    @staticmethod
    @jit(nopython=True)
    def _calculate_microprice_impl(bids, asks, alpha):
        """计算微价格（JIT加速）"""
        if len(bids) == 0 or len(asks) == 0:
            return 0.0

        best_bid = bids[0, 0]  # 价格
        best_ask = asks[0, 0]
        bid_size = bids[0, 1]  # 数量
        ask_size = asks[0, 1]

        # 微价格考虑不平衡
        total_size = bid_size + ask_size
        if total_size == 0:
            return (best_bid + best_ask) / 2

        microprice = (best_bid * ask_size + best_ask * bid_size) / total_size

        # 使用指数加权移动平均
        return alpha * microprice + (1 - alpha) * (best_bid + best_ask) / 2

    @staticmethod
    @jit(nopython=True)
    def _calculate_spread_impl(microprice, volatility, min_spread, max_spread):
        """计算动态价差（JIT加速）"""
        # 基于波动率的价差
        spread = microprice * volatility * 2

        # 限制在最小和最大价差之间
        return max(min_spread, min(max_spread, spread))

    def on_quote_tick(self, tick: QuoteTick):
        """处理报价数据"""
        # 更新价格历史
        mid_price = float((tick.bid + tick.ask) / 2)
        self.price_history.append(mid_price)

        # 计算波动率
        if len(self.price_history) > 10:
            self._update_volatility()

        # 获取订单簿快照
        order_book = self.get_order_book(self.symbol)
        if not order_book:
            return

        # 提取买卖盘数据
        bids = np.array([[float(level.price), float(level.size)]
                        for level in order_book.bids[:self.order_book_levels]])
        asks = np.array([[float(level.price), float(level.size)]
                        for level in order_book.asks[:self.order_book_levels]])

        # 计算微价格
        microprice = self.calculate_microprice(
            bids, asks, self.microprice_alpha
        )

        # 计算动态价差
        min_spread = microprice * self.min_spread_bps / 10000
        max_spread = microprice * self.max_spread_bps / 10000
        spread = self.calculate_spread(
            microprice, self.volatility, min_spread, max_spread
        )

        # 更新订单
        self._update_order_book(microprice, spread, order_book)

    def _update_volatility(self):
        """更新波动率估计"""
        prices = np.array(self.price_history)
        returns = np.diff(np.log(prices))
        self.volatility = np.std(returns) * np.sqrt(252)  # 年化波动率

    def _update_order_book(self, microprice: float, spread: float, order_book):
        """更新订单簿"""
        # 取消现有订单
        asyncio.create_task(self._cancel_all_orders())

        # 获取当前持仓
        current_position = self._get_current_position()

        # 计算多层报价
        levels = self._calculate_quote_levels(
            microprice, spread, current_position
        )

        # 下新单
        for level in levels:
            asyncio.create_task(self._place_quote_level(level))

    def _calculate_quote_levels(self, mid_price: float, spread: float,
                               position: float) -> List[dict]:
        """计算多层报价水平"""
        levels = []
        inventory_skew = position / self.max_position if self.max_position else 0

        for i in range(self.max_orders_per_side):
            # 递增的价差
            level_spread = spread * (1 + i * 0.2)

            # 买价
            bid_price = mid_price - level_spread * (1 + inventory_skew * 0.5)
            bid_size = self.order_size * (1 - i * 0.1)  # 递减的量

            # 卖价
            ask_price = mid_price + level_spread * (1 - inventory_skew * 0.5)
            ask_size = self.order_size * (1 - i * 0.1)  # 递减的量

            # 根据持仓调整
            if position > 0:  # 多头持仓，减少买单
                bid_size *= (1 - abs(inventory_skew))
            elif position < 0:  # 空头持仓，减少卖单
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
        """放置报价水平"""
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
        """取消所有订单"""
        for order_id in list(self.current_orders.keys()):
            await self.cancel_order(order_id)
            del self.current_orders[order_id]
```

## 16.2 套利策略

### 16.2.1 统计套利

```python
from sklearn.linear_model import LinearRegression
from scipy import stats
import pandas as pd

class StatisticalArbitrageStrategy(Strategy):
    """统计套利策略"""

    def __init__(self, config):
        super().__init__()
        self.pair1 = config['pair1']  # 交易对1
        self.pair2 = config['pair2']  # 交易对2
        self.lookback_period = config['lookback_period']  # 回看期
        self.z_entry_threshold = config.get('z_entry', 2.0)  # 入场Z-score阈值
        self.z_exit_threshold = config.get('z_exit', 0.5)    # 出场Z-score阈值
        self.position_size = config['position_size']

        # 状态变量
        self.price_history = {self.pair1: [], self.pair2: []}
        self.spread_history = []
        self.hedge_ratio = None
        self.mean_spread = None
        self.std_spread = None
        self.current_position = None  # long, short, or None

    def on_start(self):
        """策略启动"""
        self.subscribe_quote_ticks(self.pair1)
        self.subscribe_quote_ticks(self.pair2)
        self.log.info(f"Statistical arbitrage started for {self.pair1}-{self.pair2}")

    def on_quote_tick(self, tick: QuoteTick):
        """处理报价数据"""
        symbol = tick.symbol
        mid_price = float((tick.bid + tick.ask) / 2)

        # 更新价格历史
        self.price_history[symbol].append(mid_price)

        # 保持固定长度
        if len(self.price_history[symbol]) > self.lookback_period:
            self.price_history[symbol].pop(0)

        # 如果有足够的数据，更新统计
        if self._has_enough_data():
            self._update_statistics()

            # 检查交易信号
            signal = self._check_signal()
            if signal:
                self._execute_signal(signal)

    def _has_enough_data(self) -> bool:
        """检查是否有足够的数据"""
        return (len(self.price_history[self.pair1]) >= self.lookback_period and
                len(self.price_history[self.pair2]) >= self.lookback_period)

    def _update_statistics(self):
        """更新统计指标"""
        # 准备数据
        prices1 = np.array(self.price_history[self.pair1])
        prices2 = np.array(self.price_history[self.pair2])

        # 计算对数价格
        log_prices1 = np.log(prices1)
        log_prices2 = np.log(prices2)

        # 线性回归计算对冲比率
        X = log_prices2.reshape(-1, 1)
        y = log_prices1
        model = LinearRegression()
        model.fit(X, y)
        self.hedge_ratio = model.coef_[0]

        # 计算价差
        spread = log_prices1 - self.hedge_ratio * log_prices2
        self.spread_history = spread.tolist()
        self.mean_spread = np.mean(spread)
        self.std_spread = np.std(spread)

    def _check_signal(self) -> Optional[dict]:
        """检查交易信号"""
        if not self.hedge_ratio:
            return None

        # 计算当前价差
        current_price1 = self.price_history[self.pair1][-1]
        current_price2 = self.price_history[self.pair2][-1]
        current_spread = np.log(current_price1) - self.hedge_ratio * np.log(current_price2)

        # 计算Z-score
        if self.std_spread > 0:
            z_score = (current_spread - self.mean_spread) / self.std_spread
        else:
            z_score = 0

        # 生成信号
        signal = None

        if self.current_position is None:
            # 没有持仓，检查入场机会
            if z_score > self.z_entry_threshold:
                # 价差过高，做空价差（做空pair1，做多pair2）
                signal = {
                    'action': 'enter_short_spread',
                    'z_score': z_score,
                    'spread': current_spread
                }
                self.current_position = 'short'
            elif z_score < -self.z_entry_threshold:
                # 价差过低，做多价差（做多pair1，做空pair2）
                signal = {
                    'action': 'enter_long_spread',
                    'z_score': z_score,
                    'spread': current_spread
                }
                self.current_position = 'long'

        elif self.current_position == 'short':
            # 空头价差持仓，检查平仓
            if z_score < self.z_exit_threshold:
                signal = {
                    'action': 'exit_short_spread',
                    'z_score': z_score,
                    'spread': current_spread
                }
                self.current_position = None

        elif self.current_position == 'long':
            # 多头价差持仓，检查平仓
            if z_score > -self.z_exit_threshold:
                signal = {
                    'action': 'exit_long_spread',
                    'z_score': z_score,
                    'spread': current_spread
                }
                self.current_position = None

        return signal

    def _execute_signal(self, signal: dict):
        """执行交易信号"""
        action = signal['action']
        current_price1 = self.price_history[self.pair1][-1]
        current_price2 = self.price_history[self.pair2][-1]

        if action == 'enter_long_spread':
            # 做多价差：买入pair1，卖出pair2*hedge_ratio
            size1 = self.position_size
            size2 = int(self.position_size * self.hedge_ratio * current_price1 / current_price2)

            asyncio.create_task(self._place_order(self.pair1, OrderSide.BUY, size1))
            asyncio.create_task(self._place_order(self.pair2, OrderSide.SELL, size2))

        elif action == 'enter_short_spread':
            # 做空价差：卖出pair1，买入pair2*hedge_ratio
            size1 = self.position_size
            size2 = int(self.position_size * self.hedge_ratio * current_price1 / current_price2)

            asyncio.create_task(self._place_order(self.pair1, OrderSide.SELL, size1))
            asyncio.create_task(self._place_order(self.pair2, OrderSide.BUY, size2))

        elif action in ['exit_long_spread', 'exit_short_spread']:
            # 平仓：关闭所有相关持仓
            self._close_positions()

        self.log.info(f"Executing signal: {signal}")

    async def _place_order(self, symbol: str, side: OrderSide, size: int):
        """下限价单"""
        order = self.order_factory.market(
            symbol=symbol,
            side=side,
            quantity=Quantity.from_int(size)
        )
        await self.submit_order(order)

    def _close_positions(self):
        """关闭所有持仓"""
        # 获取当前持仓
        portfolio = self.portfolio

        # 关闭pair1的持仓
        position1 = portfolio.position(self.pair1)
        if position1 and position1.quantity != 0:
            side = OrderSide.SELL if position1.quantity > 0 else OrderSide.BUY
            size = abs(int(position1.quantity))
            asyncio.create_task(self._place_order(self.pair1, side, size))

        # 关闭pair2的持仓
        position2 = portfolio.position(self.pair2)
        if position2 and position2.quantity != 0:
            side = OrderSide.SELL if position2.quantity > 0 else OrderSide.BUY
            size = abs(int(position2.quantity))
            asyncio.create_task(self._place_order(self.pair2, side, size))

    def get_performance_metrics(self) -> dict:
        """获取策略性能指标"""
        if not self.spread_history:
            return {}

        # 计算统计指标
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
        """计算价差的半衰期"""
        # 使用AR(1)模型计算半衰期
        if len(spreads) < 2:
            return 0

        spreads_lag = spreads[:-1]
        spreads_lead = spreads[1:]

        # 线性回归
        model = LinearRegression()
        model.fit(spreads_lag.reshape(-1, 1), spreads_lead)
        beta = model.coef_[0]

        # 半衰期 = ln(0.5) / ln(beta)
        if abs(beta) < 1:
            half_life = np.log(0.5) / np.log(abs(beta))
            return max(0, half_life)
        return 0

    def _calculate_hurst_exponent(self, spreads: np.ndarray) -> float:
        """计算Hurst指数"""
        if len(spreads) < 100:
            return 0.5  # 默认值

        # 简化的Hurst指数计算
        lags = range(2, min(20, len(spreads) // 2))
        tau = [np.sqrt(np.std(np.subtract(spreads[lag:], spreads[:-lag]))) for lag in lags]

        # 线性回归拟合
        poly = np.polyfit(np.log(lags), np.log(tau), 1)
        hurst = poly[0] * 2.0

        return hurst
```

## 16.3 动量策略

### 16.3.1 时间序列动量

```python
from ta.momentum import RSIIndicator, MACD
from ta.trend import SMAIndicator, EMAIndicator

class TimeSeriesMomentumStrategy(Strategy):
    """时间序列动量策略"""

    def __init__(self, config):
        super().__init__()
        self.symbol = config['symbol']
        self.lookback_period = config['lookback_period']  # 动量回看期
        self.fast_ma = config.get('fast_ma', 10)         # 快速均线
        self.slow_ma = config.get('slow_ma', 50)         # 慢速均线
        self.rsi_period = config.get('rsi_period', 14)   # RSI周期
        self.rsi_overbought = config.get('rsi_overbought', 70)
        self.rsi_oversold = config.get('rsi_oversold', 30)
        self.position_size = config['position_size']
        self.max_position = config.get('max_position', 3)

        # 技术指标
        self.price_history = []
        self.indicators = {}

    def on_start(self):
        """策略启动"""
        self.subscribe_quote_ticks(self.symbol)
        self.log.info(f"Time series momentum strategy started for {self.symbol}")

    def on_quote_tick(self, tick: QuoteTick):
        """处理报价数据"""
        # 更新价格历史
        close_price = float((tick.bid + tick.ask) / 2)
        self.price_history.append(close_price)

        # 保持固定长度
        max_history = max(self.lookback_period, self.slow_ma * 2)
        if len(self.price_history) > max_history:
            self.price_history.pop(0)

        # 计算技术指标
        if len(self.price_history) >= self.slow_ma:
            self._calculate_indicators()

            # 生成交易信号
            signal = self._generate_signal()
            if signal:
                self._execute_signal(signal)

    def _calculate_indicators(self):
        """计算技术指标"""
        prices = np.array(self.price_history)

        # 移动平均线
        self.indicators['sma_fast'] = SMAIndicator(
            close=pd.Series(prices), window=self.fast_ma
        ).sma_indicator().iloc[-1]

        self.indicators['sma_slow'] = SMAIndicator(
            close=pd.Series(prices), window=self.slow_ma
        ).sma_indicator().iloc[-1]

        # 指数移动平均线
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

        # 动量指标
        if len(self.price_history) >= self.lookback_period:
            momentum = (prices[-1] - prices[-self.lookback_period]) / prices[-self.lookback_period]
            self.indicators['momentum'] = momentum
            self.indicators['momentum_pct'] = momentum * 100

    def _generate_signal(self) -> Optional[dict]:
        """生成交易信号"""
        # 获取当前持仓
        current_position = self._get_current_position()

        # 多重条件确认
        long_conditions = []
        short_conditions = []

        # 1. 均线交叉
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

        # 4. 动量
        if 'momentum' in self.indicators:
            if self.indicators['momentum'] > 0:
                long_conditions.append('momentum_positive')
            else:
                short_conditions.append('momentum_negative')

        # 生成信号
        signal = None

        if len(long_conditions) >= 3 and current_position <= 0:  # 至少3个看多条件
            signal = {
                'action': 'buy',
                'conditions': long_conditions,
                'strength': len(long_conditions)
            }

        elif len(short_conditions) >= 3 and current_position >= 0:  # 至少3个看空条件
            signal = {
                'action': 'sell',
                'conditions': short_conditions,
                'strength': len(short_conditions)
            }

        # 止损条件
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
        """执行交易信号"""
        action = signal['action']
        current_position = self._get_current_position()

        if action == 'buy' and current_position < self.max_position:
            # 计算仓位大小
            position_to_add = min(
                self.position_size,
                self.max_position - current_position
            )

            # 基于信号强度调整仓位
            if 'strength' in signal:
                position_to_add = int(position_to_add * signal['strength'] / 4)

            if position_to_add > 0:
                asyncio.create_task(
                    self._place_order(OrderSide.BUY, position_to_add)
                )

        elif action == 'sell' and current_position > -self.max_position:
            # 计算仓位大小
            position_to_add = min(
                self.position_size,
                self.max_position + current_position
            )

            # 基于信号强度调整仓位
            if 'strength' in signal:
                position_to_add = int(position_to_add * signal['strength'] / 4)

            if position_to_add > 0:
                asyncio.create_task(
                    self._place_order(OrderSide.SELL, position_to_add)
                )

        elif action in ['close_long', 'close_short']:
            # 平仓
            if current_position != 0:
                asyncio.create_task(
                    self._place_order(
                        OrderSide.SELL if current_position > 0 else OrderSide.BUY,
                        abs(current_position)
                    )
                )

        self.log.info(f"Executing signal: {signal}")

    async def _place_order(self, side: OrderSide, size: int):
        """下单"""
        order = self.order_factory.market(
            symbol=self.symbol,
            side=side,
            quantity=Quantity.from_int(size)
        )
        await self.submit_order(order)

    def _get_current_position(self) -> int:
        """获取当前持仓"""
        position = self.portfolio.position(self.symbol)
        return int(position.quantity) if position else 0

    def get_strategy_state(self) -> dict:
        """获取策略状态"""
        return {
            'indicators': self.indicators,
            'position': self._get_current_position(),
            'price_history_length': len(self.price_history)
        }
```

## 16.4 机器学习策略

### 16.4.1 强化学习交易

```python
import gym
from gym import spaces
import torch
import torch.nn as nn
from collections import deque
import random

class TradingEnvironment(gym.Env):
    """交易环境（OpenAI Gym接口）"""

    def __init__(self, data: pd.DataFrame, initial_balance: float = 100000):
        super().__init__()

        self.data = data
        self.initial_balance = initial_balance
        self.current_step = 0
        self.balance = initial_balance
        self.position = 0
        self.position_price = 0

        # 动作空间：0=持有, 1=买入, 2=卖出
        self.action_space = spaces.Discrete(3)

        # 状态空间：价格、技术指标、持仓信息
        self.observation_space = spaces.Box(
            low=-np.inf,
            high=np.inf,
            shape=(10,),  # 可根据需要调整
            dtype=np.float32
        )

        # 历史记录
        self.trade_history = []
        self.portfolio_history = []

    def reset(self):
        """重置环境"""
        self.current_step = 0
        self.balance = self.initial_balance
        self.position = 0
        self.position_price = 0
        self.trade_history = []
        self.portfolio_history = []
        return self._get_observation()

    def step(self, action):
        """执行一步"""
        # 获取当前价格
        current_price = self.data.iloc[self.current_step]['close']

        # 执行动作
        reward = 0
        done = False

        if action == 1:  # 买入
            if self.position == 0:  # 无持仓时才能买入
                self.position = self.balance / current_price
                self.position_price = current_price
                self.balance = 0

        elif action == 2:  # 卖出
            if self.position > 0:  # 有持仓时才能卖出
                self.balance = self.position * current_price
                profit = (current_price - self.position_price) * self.position
                reward = profit / self.initial_balance  # 归一化奖励
                self.position = 0
                self.position_price = 0

        # 计算总资产价值
        total_value = self.balance + self.position * current_price
        self.portfolio_history.append(total_value)

        # 记录交易
        self.trade_history.append({
            'step': self.current_step,
            'action': action,
            'price': current_price,
            'position': self.position,
            'balance': self.balance,
            'total_value': total_value
        })

        # 前进一步
        self.current_step += 1

        # 检查是否结束
        if self.current_step >= len(self.data) - 1:
            done = True
            # 最后强制平仓
            if self.position > 0:
                self.balance = self.position * current_price
                self.position = 0

        # 获取新状态
        observation = self._get_observation()

        # 计算额外奖励（基于总资产变化）
        if len(self.portfolio_history) > 1:
            portfolio_return = (total_value - self.portfolio_history[-2]) / self.portfolio_history[-2]
            reward += portfolio_return * 0.1  # 较小的权重

        return observation, reward, done, {}

    def _get_observation(self):
        """获取当前状态"""
        if self.current_step >= len(self.data):
            return np.zeros(10)

        current_data = self.data.iloc[self.current_step]

        # 构建状态向量
        observation = np.array([
            current_data['close'] / self.data['close'].iloc[0] - 1,  # 归一化价格
            current_data['volume'] / self.data['volume'].mean() - 1,  # 归一化成交量
            current_data.get('rsi', 50) / 100,  # RSI
            current_data.get('macd', 0),  # MACD
            current_data.get('bb_upper', 0) / current_data['close'] - 1,  # 布林带上轨
            current_data.get('bb_lower', 0) / current_data['close'] - 1,  # 布林带下轨
            self.position / self.initial_balance if self.position > 0 else 0,  # 持仓比例
            (current_data['close'] - self.position_price) / self.position_price if self.position_price > 0 else 0,  # 持仓收益率
            self.balance / self.initial_balance,  # 现金比例
            len(self.trade_history) / len(self.data)  # 时间进度
        ])

        return observation.astype(np.float32)

class DQNNetwork(nn.Module):
    """深度Q网络"""

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
    """强化学习交易策略"""

    def __init__(self, config):
        super().__init__()
        self.symbol = config['symbol']
        self.model_path = config.get('model_path', 'models/dqn_model.pth')
        self.lookback_window = config.get('lookback_window', 100)
        self.position_size = config['position_size']
        self.max_position = config.get('max_position', 10)

        # DQN参数
        self.state_size = 10
        self.action_size = 3  # 持有、买入、卖出
        self.epsilon = config.get('epsilon', 0.1)  # 探索率
        self.memory = deque(maxlen=10000)

        # 加载训练好的模型
        self.device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
        self.model = DQNNetwork(self.state_size, self.action_size).to(self.device)
        self._load_model()

        # 数据缓存
        self.price_history = []
        self.feature_cache = {}

    def _load_model(self):
        """加载训练好的模型"""
        try:
            self.model.load_state_dict(torch.load(self.model_path, map_location=self.device))
            self.model.eval()
            self.log.info(f"Loaded RL model from {self.model_path}")
        except FileNotFoundError:
            self.log.warning(f"Model file not found: {self.model_path}, using random policy")
        except Exception as e:
            self.log.error(f"Failed to load model: {e}")

    def on_start(self):
        """策略启动"""
        self.subscribe_quote_ticks(self.symbol)
        self.log.info(f"RL trading strategy started for {self.symbol}")

    def on_quote_tick(self, tick: QuoteTick):
        """处理报价数据"""
        # 更新价格历史
        close_price = float((tick.bid + tick.ask) / 2)
        self.price_history.append(close_price)

        # 保持固定长度
        if len(self.price_history) > self.lookback_window * 2:
            self.price_history.pop(0)

        # 计算技术指标
        if len(self.price_history) >= self.lookback_window:
            self._update_features()

            # 获取当前状态
            state = self._get_current_state()

            # 选择动作
            action = self._select_action(state)

            # 执行动作
            self._execute_action(action, close_price)

    def _update_features(self):
        """更新技术指标"""
        prices = np.array(self.price_history)

        # 计算各种技术指标
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

        # 布林带
        df['sma'] = df['close'].rolling(window=20).mean()
        df['std'] = df['close'].rolling(window=20).std()
        df['bb_upper'] = df['sma'] + 2 * df['std']
        df['bb_lower'] = df['sma'] - 2 * df['std']

        # 成交量（如果有）
        df['volume'] = np.random.randn(len(df))  # 模拟成交量

        # 缓存最新的特征值
        self.feature_cache = df.iloc[-1].to_dict()

    def _get_current_state(self) -> np.ndarray:
        """获取当前状态"""
        if not self.feature_cache:
            return np.zeros(self.state_size)

        # 构建状态向量
        current_price = self.price_history[-1]
        base_price = self.price_history[0]

        state = np.array([
            (current_price / base_price) - 1,  # 归一化价格
            (self.feature_cache.get('volume', 0) / 100) - 1,  # 归一化成交量
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
        """选择动作"""
        # Epsilon-greedy策略
        if random.random() < self.epsilon:
            return random.randrange(self.action_size)

        # 使用模型预测
        with torch.no_grad():
            state_tensor = torch.FloatTensor(state).unsqueeze(0).to(self.device)
            q_values = self.model(state_tensor)
            return q_values.argmax().item()

    def _execute_action(self, action: int, price: float):
        """执行动作"""
        current_position = self._get_current_position()

        if action == 1 and current_position < self.max_position:  # 买入
            # 计算买入数量
            cash_available = self._get_available_cash()
            quantity = min(self.position_size, int(cash_available / price))

            if quantity > 0:
                asyncio.create_task(
                    self._place_order(OrderSide.BUY, quantity)
                )

        elif action == 2 and current_position > 0:  # 卖出
            # 卖出一部分或全部
            sell_quantity = min(self.position_size, current_position)

            if sell_quantity > 0:
                asyncio.create_task(
                    self._place_order(OrderSide.SELL, sell_quantity)
                )

        # 记录状态转移（用于可能的在线学习）
        self._remember_transition(state, action, price)

    def _get_position_ratio(self) -> float:
        """获取持仓比例"""
        position = self._get_current_position()
        return position / self.max_position if self.max_position > 0 else 0

    def _get_position_return(self) -> float:
        """获取持仓收益率"""
        # 这里需要跟踪平均持仓成本
        # 简化实现
        return 0.0

    def _get_cash_ratio(self) -> float:
        """获取现金比例"""
        # 简化实现
        return 0.5

    def _get_available_cash(self) -> float:
        """获取可用现金"""
        # 简化实现
        return 100000  # 假设的可用资金

    async def _place_order(self, side: OrderSide, quantity: int):
        """下单"""
        order = self.order_factory.market(
            symbol=self.symbol,
            side=side,
            quantity=Quantity.from_int(quantity)
        )
        await self.submit_order(order)

    def _remember_transition(self, state: np.ndarray, action: int, price: float):
        """记住状态转移（用于经验回放）"""
        if len(self.memory) < self.memory.maxlen:
            self.memory.append((state, action, price))

    def get_model_confidence(self, state: np.ndarray) -> float:
        """获取模型预测的置信度"""
        with torch.no_grad():
            state_tensor = torch.FloatTensor(state).unsqueeze(0).to(self.device)
            q_values = self.model(state_tensor)

            # 使用最大Q值与其他Q值的差异作为置信度
            q_values_array = q_values.cpu().numpy()[0]
            max_q = np.max(q_values_array)
            mean_q = np.mean(q_values_array)

            confidence = (max_q - mean_q) / (np.std(q_values_array) + 1e-6)
            return float(confidence)

    def update_model(self, new_model_path: str):
        """热更新模型"""
        try:
            self.model.load_state_dict(torch.load(new_model_path, map_location=self.device))
            self.model.eval()
            self.model_path = new_model_path
            self.log.info(f"Model updated to: {new_model_path}")
        except Exception as e:
            self.log.error(f"Failed to update model: {e}")
```

## 小结

本章详细介绍了多种算法交易策略的实现，涵盖了：

1. **做市策略**
   - 基础做市策略
   - 高频做市策略
   - 库存管理和价差调整

2. **套利策略**
   - 统计套利
   - 配对交易
   - 协整分析

3. **动量策略**
   - 时间序列动量
   - 技术指标组合
   - 多条件确认

4. **机器学习策略**
   - 强化学习
   - 深度Q网络
   - 在线学习

5. **性能优化**
   - JIT编译
   - 向量化计算
   - 高频数据处理

通过本章的学习，读者可以构建各种复杂的算法交易策略，并根据不同的市场环境选择合适的策略类型。

## 参考资料

- [Algorithmic Trading: Winning Strategies and Their Rationale](https://www.wiley.com/en-us/Algorithmic+Trading%3A+Winning+Strategies+and+Their+Rationale)
- [Advances in Financial Machine Learning](https://www.wiley.com/en-us/Advances+in+Financial+Machine+Learning)
- [Reinforcement Learning for Finance](https://rlfinance.io/)
## 本章可运行代码入口（源码版）

- 入口脚本：`docs/tutorials_cn/code/chapter16/run_algorithmic_trading.py`
- 关键示例：`docs/tutorials_cn/code/chapter16/mean_reversion.py`、`docs/tutorials_cn/code/chapter16/momentum.py`、`docs/tutorials_cn/code/chapter16/pairs_trading.py`

依赖策略说明：三类策略统一信号接口，数据不足时自动降级为 `HOLD`，不抛异常中断。

已验证命令：

```bash
uv run --python 3.13 python docs/tutorials_cn/code/chapter16/run_algorithmic_trading.py
```
