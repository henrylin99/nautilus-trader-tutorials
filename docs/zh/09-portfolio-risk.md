# 第9章：投资组合与风险管理

## 学习目标

通过本章学习，您将：
- 掌握投资组合管理的基本概念
- 学会实现多层风险控制
- 理解资金管理策略
- 构建风险监控系统
- 实现动态仓位管理

## 9.1 投资组合基础

### 9.1.1 核心概念

```python
"""
投资组合管理基础
"""

from decimal import Decimal
from typing import Dict, List, Optional, Tuple
from datetime import datetime
from dataclasses import dataclass

from nautilus_trader.model.objects import Money
from nautilus_trader.model.identifiers import InstrumentId
from nautilus_trader.model.position import Position
from nautilus_trader.model.enums import PositionSide


@dataclass
class AssetAllocation:
    """资产配置"""
    instrument_id: InstrumentId
    weight: Decimal  # 权重（0-1）
    target_quantity: Optional[Decimal] = None
    current_quantity: Decimal = Decimal('0')
    market_value: Decimal = Decimal('0')


class PortfolioManager:
    """投资组合管理器"""

    def __init__(
        self,
        initial_capital: Decimal,
        base_currency: str = "USDT",
    ):
        """
        初始化投资组合管理器

        Parameters
        ----------
        initial_capital : Decimal
            初始资金
        base_currency : str
            基础货币
        """
        self.initial_capital = initial_capital
        self.base_currency = base_currency

        # 当前状态
        self.current_capital = initial_capital
        self.unrealized_pnl = Decimal('0')
        self.realized_pnl = Decimal('0')

        # 持仓信息
        self.positions = {}  # instrument_id -> Position
        self.allocations = {}  # asset_name -> AssetAllocation

        # 历史记录
        self.transaction_history = []
        self.pnl_history = []

        # 风险指标
        self.max_drawdown = Decimal('0')
        self.current_drawdown = Decimal('0')
        self.sharpe_ratio = Decimal('0')

    def add_asset_allocation(
        self,
        instrument_id: InstrumentId,
        weight: Decimal,
    ):
        """
        添加资产配置

        Parameters
        ----------
        instrument_id : InstrumentId
            交易工具ID
        weight : Decimal
            权重（0-1）
        """
        asset_name = str(instrument_id)
        self.allocations[asset_name] = AssetAllocation(
            instrument_id=instrument_id,
            weight=weight
        )

    def update_position(self, position: Position):
        """
        更新持仓

        Parameters
        ----------
        position : Position
            持仓对象
        """
        instrument_id = position.instrument_id
        self.positions[instrument_id] = position

        # 更新资产配置
        asset_name = str(instrument_id)
        if asset_name in self.allocations:
            self.allocations[asset_name].current_quantity = position.quantity

    def calculate_total_value(self, prices: Dict[InstrumentId, Decimal]) -> Decimal:
        """
        计算投资组合总价值

        Parameters
        ----------
        prices : Dict[InstrumentId, Decimal]
            当前价格

        Returns
        -------
        Decimal
            总价值
        """
        total_value = self.current_capital

        for instrument_id, position in self.positions.items():
            if instrument_id in prices:
                market_value = abs(position.quantity) * prices[instrument_id]
                total_value += market_value

                # 更新资产配置
                asset_name = str(instrument_id)
                if asset_name in self.allocations:
                    self.allocations[asset_name].market_value = market_value

        return total_value

    def get_current_allocation(self) -> Dict[str, Decimal]:
        """获取当前资产配置"""
        total_value = self._get_current_market_value()

        allocations = {}
        for asset_name, allocation in self.allocations.items():
            if total_value > 0:
                allocations[asset_name] = allocation.market_value / total_value

        return allocations

    def get_target_allocation(self) -> Dict[str, Decimal]:
        """获取目标资产配置"""
        return {
            name: allocation.weight
            for name, allocation in self.allocations.items()
        }

    def calculate_rebalance_orders(
        self,
        current_prices: Dict[InstrumentId, Decimal],
    ) -> List[Dict]:
        """
        计算再平衡订单

        Parameters
        ----------
        current_prices : Dict[InstrumentId, Decimal]
            当前价格

        Returns
        -------
        List[Dict]
            再平衡订单列表
        """
        total_value = self.calculate_total_value(current_prices)
        rebalance_orders = []

        for allocation in self.allocations.values():
            instrument_id = allocation.instrument_id
            current_price = current_prices.get(instrument_id)

            if not current_price or current_price <= 0:
                continue

            # 计算目标价值
            target_value = total_value * allocation.weight

            # 计算当前价值
            current_value = allocation.market_value

            # 计算差异
            value_diff = target_value - current_value

            # 计算需要调整的数量
            if abs(value_diff) > total_value * Decimal('0.01'):  # 1%的阈值
                quantity_diff = value_diff / current_price

                # 确定订单方向
                if quantity_diff > 0:
                    order_side = 'BUY'
                    quantity = abs(quantity_diff)
                else:
                    order_side = 'SELL'
                    quantity = abs(quantity_diff)

                if quantity > 0:
                    rebalance_orders.append({
                        'instrument_id': instrument_id,
                        'side': order_side,
                        'quantity': quantity,
                        'price': current_price,
                        'reason': 'REBALANCE',
                    })

        return rebalance_orders

    def _get_current_market_value(self) -> Decimal:
        """获取当前市场价值"""
        return sum(
            allocation.market_value
            for allocation in self.allocations.values()
        )

    def calculate_performance_metrics(
        self,
        current_prices: Dict[InstrumentId, Decimal],
        start_time: datetime,
    ) -> Dict:
        """
        计算业绩指标

        Parameters
        ----------
        current_prices : Dict[InstrumentId, Decimal]
            当前价格
        start_time : datetime
            开始时间

        Returns
        -------
        Dict
            业绩指标
        """
        current_time = datetime.utcnow()
        duration_days = (current_time - start_time).days

        total_value = self.calculate_total_value(current_prices)
        total_return = (total_value - self.initial_capital) / self.initial_capital

        # 计算年化收益率
        annualized_return = (1 + total_return) ** (365 / duration_days) - 1

        # 更新最大回撤
        self._update_max_drawdown(total_value)

        # 计算夏普比率（假设无风险利率为2%）
        risk_free_rate = Decimal('0.02')
        excess_return = annualized_return - risk_free_rate
        volatility = self._calculate_volatility(current_prices)

        if volatility > 0:
            self.sharpe_ratio = excess_return / volatility

        return {
            'total_value': float(total_value),
            'total_return': float(total_return),
            'annualized_return': float(annualized_return),
            'max_drawdown': float(self.max_drawdown),
            'sharpe_ratio': float(self.sharpe_ratio),
            'current_drawdown': float(self.current_drawdown),
            'duration_days': duration_days,
        }

    def _update_max_drawdown(self, current_value: Decimal):
        """更新最大回撤"""
        # 这里需要维护价值的历史记录
        # 简化实现
        peak_value = self.initial_capital * Decimal('1.2')  # 假设峰值

        if current_value > peak_value:
            peak_value = current_value
        else:
            drawdown = (peak_value - current_value) / peak_value
            self.current_drawdown = drawdown
            self.max_drawdown = max(self.max_drawdown, drawdown)

    def _calculate_volatility(self, current_prices: Dict[InstrumentId, Decimal]) -> Decimal:
        """计算波动率"""
        # 简化实现，实际应该使用历史数据
        return Decimal('0.15')  # 15% 年化波动率

    def generate_portfolio_report(self) -> str:
        """生成投资组合报告"""
        report = f"""
投资组合报告
=================
初始资金: {self.initial_capital}
当前资金: {self.current_capital}
已实现盈亏: {self.realized_pnl}
未实现盈亏: {self.unrealized_pnl}

资产配置:
- 目标配置: {self.get_target_allocation()}
- 当前配置: {self.get_current_allocation()}

风险指标:
- 最大回撤: {self.max_drawdown:.2%}
- 夏普比率: {self.sharpe_ratio:.2f}

持仓数量: {len(self.positions)}
"""
        return report
```

### 9.1.2 多策略投资组合

```python
"""
多策略投资组合管理
"""

from enum import Enum
from typing import Set
from collections import defaultdict


class StrategyType(Enum):
    """策略类型"""
    MOMENTUM = "Momentum"
    MEAN_REVERSION = "Mean Reversion"
    ARBITRAGE = "Arbitrage"
    MARKET_MAKING = "Market Making"
    TREND_FOLLOWING = "Trend Following"


class MultiStrategyPortfolio:
    """多策略投资组合"""

    def __init__(self, initial_capital: Decimal):
        """
        初始化多策略投资组合

        Parameters
        ----------
        initial_capital : Decimal
            初始资金
        """
        self.initial_capital = initial_capital
        self.current_capital = initial_capital

        # 策略管理
        self.strategies = {}  # strategy_id -> strategy_info
        self.strategy_positions = defaultdict(dict)  # strategy_id -> instrument_id -> position

        # 资金分配
        self.strategy_allocations = {}
        self.update_allocations()

    def add_strategy(
        self,
        strategy_id: str,
        strategy_type: StrategyType,
        allocation_weight: Decimal,
        max_capital: Decimal = None,
    ):
        """
        添加策略

        Parameters
        ----------
        strategy_id : str
            策略ID
        strategy_type : StrategyType
            策略类型
        allocation_weight : Decimal
            资金分配权重
        max_capital : Decimal
            最大资金限制
        """
        self.strategies[strategy_id] = {
            'type': strategy_type,
            'allocation_weight': allocation_weight,
            'max_capital': max_capital,
            'allocated_capital': Decimal('0'),
            'current_pnl': Decimal('0'),
            'positions': {},
            'trade_count': 0,
            'last_trade_time': None,
        }

        self.update_allocations()

    def update_allocations(self):
        """更新资金分配"""
        total_weight = sum(
            info['allocation_weight']
            for info in self.strategies.values()
        )

        if total_weight > 0:
            for strategy_id, info in self.strategies.items():
                weight = info['allocation_weight'] / total_weight
                allocated_capital = self.initial_capital * weight

                # 检查最大资金限制
                if info['max_capital']:
                    allocated_capital = min(allocated_capital, info['max_capital'])

                self.strategy_allocations[strategy_id] = allocated_capital

    def update_strategy_position(
        self,
        strategy_id: str,
        position: Position,
    ):
        """
        更新策略持仓

        Parameters
        ----------
        strategy_id : str
            策略ID
        position : Position
            持仓
        """
        instrument_id = position.instrument_id
        self.strategy_positions[strategy_id][instrument_id] = position
        self.strategies[strategy_id]['positions'][instrument_id] = position

    def get_strategy_performance(self, strategy_id: str) -> dict:
        """
        获取策略表现

        Parameters
        ----------
        strategy_id : str
            策略ID

        Returns
        -------
        dict
            策略表现数据
        """
        if strategy_id not in self.strategies:
            return {}

        info = self.strategies[strategy_id]
        positions = info['positions']

        # 计算持仓价值
        position_value = Decimal('0')
        unrealized_pnl = Decimal('0')

        for position in positions.values():
            # 这里应该获取当前价格
            current_price = self._get_current_price(position.instrument_id)
            if current_price:
                position_value += abs(position.quantity) * current_price
                unrealized_pnl += position.unrealized_pnl(self.current_capital)

        # 计算收益率
        allocated_capital = info['allocated_capital']
        total_pnl = info['current_pnl'] + unrealized_pnl
        return_rate = total_pnl / allocated_capital if allocated_capital > 0 else 0

        return {
            'strategy_id': strategy_id,
            'type': info['type'].value,
            'allocated_capital': float(allocated_capital),
            'current_pnl': float(info['current_pnl']),
            'unrealized_pnl': float(unrealized_pnl),
            'total_pnl': float(total_pnl),
            'return_rate': float(return_rate),
            'position_count': len(positions),
            'trade_count': info['trade_count'],
            'last_trade_time': info['last_trade_time'],
        }

    def get_portfolio_summary(self) -> dict:
        """获取投资组合摘要"""
        total_allocated = sum(self.strategy_allocations.values())
        total_pnl = sum(
            info['current_pnl'] for info in self.strategies.values()
        )

        strategy_performance = {}
        for strategy_id in self.strategies:
            strategy_performance[strategy_id] = self.get_strategy_performance(strategy_id)

        # 按类型分组
        performance_by_type = defaultdict(list)
        for strategy_id, perf in strategy_performance.items():
            performance_by_type[perf['type']].append(perf)

        type_summary = {}
        for strategy_type, perfs in performance_by_type.items():
            if perfs:
                avg_return = sum(p['return_rate'] for p in perfs) / len(perfs)
                type_summary[strategy_type] = {
                    'strategy_count': len(perfs),
                    'avg_return': avg_return,
                    'total_pnl': sum(p['total_pnl'] for p in perfs),
                }

        return {
            'total_capital': float(self.initial_capital),
            'allocated_capital': float(total_allocated),
            'unallocated_capital': float(self.initial_capital - total_allocated),
            'total_pnl': float(total_pnl),
            'strategy_count': len(self.strategies),
            'strategy_performance': strategy_performance,
            'performance_by_type': type_summary,
        }

    def check_strategy_correlations(self) -> dict:
        """检查策略相关性"""
        # 简化实现 - 基于策略类型
        correlations = {}

        strategy_types = {}
        for strategy_id, info in self.strategies.items():
            strategy_types[strategy_id] = info['type']

        # 计算类型分布
        type_distribution = defaultdict(list)
        for strategy_id, strategy_type in strategy_types.items():
            type_distribution[strategy_type].append(strategy_id)

        # 相关性分析（简化）
        type_names = list(type_distribution.keys())
        for i, type1 in enumerate(type_names):
            correlations[type1] = {}
            for j, type2 in enumerate(type_names):
                if i == j:
                    correlations[type1][type2] = 1.0
                else:
                    # 不同类型策略通常相关性较低
                    correlations[type1][type2] = 0.3 if type1 != type2 else 1.0

        return correlations

    def optimize_strategy_allocations(self) -> dict:
        """
        优化策略资金分配

        Returns
        -------
        dict
            优化后的分配
        """
        # 基于历史表现优化分配
        strategy_performance = {
            sid: self.get_strategy_performance(sid)
            for sid in self.strategies
        }

        # 过滤有表现的策略
        active_strategies = {
            sid: perf for sid, perf in strategy_performance.items()
            if perf['trade_count'] > 0
        }

        if not active_strategies:
            return self.strategy_allocations

        # 计算夏普比率
        for sid, perf in active_strategies.items():
            # 简化的夏普比率计算
            return_rate = perf['return_rate']
            volatility = abs(return_rate) * 2  # 简化假设
            perf['sharpe_ratio'] = return_rate / volatility if volatility > 0 else 0

        # 根据夏普比率调整权重
        total_sharpe = sum(abs(p['sharpe_ratio']) for p in active_strategies.values())

        optimized_allocations = {}
        if total_sharpe > 0:
            for sid, perf in active_strategies.items():
                weight = abs(perf['sharpe_ratio']) / total_sharpe
                # 设置最小权重为5%
                weight = max(weight, Decimal('0.05'))

                # 重新归一化
                optimized_allocations[sid] = weight

        # 归一化权重
        total_weight = sum(optimized_allocations.values())
        for sid in optimized_allocations:
            optimized_allocations[sid] /= total_weight

        return optimized_allocations

    def _get_current_price(self, instrument_id: InstrumentId) -> Optional[Decimal]:
        """获取当前价格（示例）"""
        # 这里应该从缓存或市场数据获取
        return Decimal("50000.00")  # 示例价格


# 使用多策略投资组合
class MultiStrategyCoordinator(Strategy):
    """多策略协调器"""

    def __init__(self):
        """初始化协调器"""
        super().__init__()

        # 创建多策略投资组合
        self.portfolio = MultiStrategyPortfolio(
            initial_capital=Decimal("100000")
        )

        # 注册策略
        self._register_strategies()

    def _register_strategies(self):
        """注册策略"""
        # 动量策略
        self.portfolio.add_strategy(
            "momentum_001",
            StrategyType.MOMENTUM,
            Decimal("0.4"),  # 40% 资金
            Decimal("50000")  # 最大5万
        )

        # 均值回归策略
        self.portfolio.add_strategy(
            "mean_rev_001",
            StrategyType.MEAN_REVERSION,
            Decimal("0.3"),  # 30% 资金
            Decimal("40000")  # 最大4万
        )

        # 套利策略
        self.portfolio.add_strategy(
            "arbitrage_001",
            StrategyType.ARBITRAGE,
            Decimal("0.2"),  # 20% 资金
            Decimal("30000")  # 最大3万
        )

        # 做市策略
        self.portfolio.add_strategy(
            "market_make_001",
            StrategyType.MARKET_MAKING,
            Decimal("0.1"),  # 10% 资金
            Decimal("20000")  # 最大2万
        )

    def on_start(self):
        """策略启动"""
        self.log.info("多策略协调器启动")
        self.log.info(f"注册了 {len(self.portfolio.strategies)} 个策略")

        # 输出初始分配
        self.log.info(f"初始资金分配: {self.portfolio.strategy_allocations}")

    def on_order_filled(self, event):
        """处理订单成交"""
        # 更新策略统计
        for strategy_id in self.portfolio.strategies:
            # 这里需要确定订单属于哪个策略
            # 简化处理：假设事件中包含策略ID
            if hasattr(event, 'strategy_id') and event.strategy_id.value == strategy_id:
                self.portfolio.strategies[strategy_id]['trade_count'] += 1
                self.portfolio.strategies[strategy_id]['last_trade_time'] = datetime.utcnow()
                break

    def on_bar(self, bar: Bar):
        """处理K线数据"""
        # 定期生成报告
        if self.count_bars() % 100 == 0:  # 每100根K线
            self._generate_portfolio_report()

    def _generate_portfolio_report(self):
        """生成投资组合报告"""
        summary = self.portfolio.get_portfolio_summary()

        self.log.info("=== 多策略投资组合报告 ===")
        self.log.info(f"总资金: {summary['total_capital']:,.2f}")
        self.log.info(f"已分配: {summary['allocated_capital']:,.2f}")
        self.log.info(f"未分配: {summary['unallocated_capital']:,.2f}")
        self.log.info(f"总盈亏: {summary['total_pnl']:,.2f}")

        self.log.info("\n各策略表现:")
        for strategy_id, perf in summary['strategy_performance'].items():
            self.log.info(
                f"  {strategy_id}: "
                f"收益率 {perf['return_rate']:.2%}, "
                f"交易次数 {perf['trade_count']}, "
                f"持仓数 {perf['position_count']}"
            )

        self.log.info("\n按类型统计:")
        for strategy_type, stats in summary['performance_by_type'].items():
            self.log.info(
                f"  {strategy_type}: "
                f"{stats['strategy_count']}个策略, "
                f"平均收益 {stats['avg_return']:.2%}, "
                f"总盈亏 {stats['total_pnl']:,.2f}"
            )

    def on_stop(self):
        """策略停止"""
        # 优化资金分配
        optimized_allocations = self.portfolio.optimize_strategy_allocations()
        self.log.info("\n优化后的资金分配:")
        for strategy_id, weight in optimized_allocations.items():
            self.log.info(f"  {strategy_id}: {weight:.2%}")
```

## 9.2 风险管理系统

### 9.2.1 多层风险控制

```python
"""
多层风险控制系统
"""

from enum import Enum
from typing import List, Dict, Optional, Callable
from dataclasses import dataclass
import time


class RiskLevel(Enum):
    """风险级别"""
    LOW = "LOW"
    MEDIUM = "MEDIUM"
    HIGH = "HIGH"
    CRITICAL = "CRITICAL"


class RiskType(Enum):
    """风险类型"""
    POSITION_LIMIT = "Position Limit"
    EXPOSURE_LIMIT = "Exposure Limit"
    DRAWDOWN_LIMIT = "Drawdown Limit"
    CORRELATION_RISK = "Correlation Risk"
    CONCENTRATION_RISK = "Concentration Risk"
    LIQUIDITY_RISK = "Liquidity Risk"
    VOLATILITY_RISK = "Volatility Risk"


@dataclass
class RiskEvent:
    """风险事件"""
    timestamp: datetime
    risk_type: RiskType
    risk_level: RiskLevel
    message: str
    strategy_id: Optional[str] = None
    instrument_id: Optional[str] = None
    current_value: Optional[float] = None
    limit_value: Optional[float] = None
    mitigation_action: Optional[str] = None


class RiskRule:
    """风险规则"""

    def __init__(
        self,
        name: str,
        risk_type: RiskType,
        risk_levels: Dict[RiskLevel, float],
        check_function: Callable,
        mitigation_actions: Dict[RiskLevel, str],
    ):
        """
        初始化风险规则

        Parameters
        ----------
        name : str
            规则名称
        risk_type : RiskType
            风险类型
        risk_levels : Dict[RiskLevel, float]
            风险级别阈值
        check_function : Callable
            检查函数
        mitigation_actions : Dict[RiskLevel, str]
            缓解措施
        """
        self.name = name
        self.risk_type = risk_type
        self.risk_levels = sorted(risk_levels.items(), key=lambda x: x[1])
        self.check_function = check_function
        self.mitigation_actions = mitigation_actions

    def evaluate(self, context: dict) -> Optional[RiskEvent]:
        """
        评估风险

        Parameters
        ----------
        context : dict
            上下文数据

        Returns
        -------
        Optional[RiskEvent]
            风险事件
        """
        current_value = self.check_function(context)
        if current_value is None:
            return None

        # 确定风险级别
        risk_level = None
        for level, threshold in self.risk_levels:
            if current_value >= threshold:
                risk_level = level
            else:
                break

        if risk_level:
            mitigation_action = self.mitigation_actions.get(risk_level)

            return RiskEvent(
                timestamp=datetime.utcnow(),
                risk_type=self.risk_type,
                risk_level=risk_level,
                message=f"{self.name}: {current_value:.2f}",
                current_value=current_value,
                limit_value=next((t for l, t in self.risk_levels if l == risk_level), None),
                mitigation_action=mitigation_action,
                **{k: v for k, v in context.items() if k in ['strategy_id', 'instrument_id']}
            )

        return None


class RiskEngine:
    """风险引擎"""

    def __init__(self):
        """初始化风险引擎"""
        self.rules: List[RiskRule] = []
        self.active_events: List[RiskEvent] = []
        self.event_history: List[RiskEvent] = []
        self.mandatory_stop = False

        # 风险计数器
        self.risk_counts = defaultdict(int)

    def add_rule(self, rule: RiskRule):
        """添加风险规则"""
        self.rules.append(rule)
        self.log.info(f"添加风险规则: {rule.name}")

    def evaluate_risks(self, context: dict) -> List[RiskEvent]:
        """
        评估所有风险

        Parameters
        ----------
        context : dict
            上下文数据

        Returns
        -------
        List[RiskEvent]
            风险事件列表
        """
        new_events = []

        for rule in self.rules:
            event = rule.evaluate(context)
            if event:
                new_events.append(event)
                self.active_events.append(event)
                self.event_history.append(event)
                self.risk_counts[event.risk_type] += 1

                # 检查是否需要强制停止
                if event.risk_level == RiskLevel.CRITICAL:
                    self.mandatory_stop = True
                    self.log.critical(f"关键风险触发: {event.message}")

        return new_events

    def get_active_events(self) -> List[RiskEvent]:
        """获取活跃风险事件"""
        return [e for e in self.active_events if e]

    def clear_expired_events(self, expire_minutes: int = 30):
        """清理过期事件"""
        cutoff_time = datetime.utcnow() - timedelta(minutes=expire_minutes)

        self.active_events = [
            e for e in self.active_events
            if e.timestamp > cutoff_time
        ]

    def get_risk_summary(self) -> dict:
        """获取风险摘要"""
        active_by_level = defaultdict(int)
        active_by_type = defaultdict(int)

        for event in self.active_events:
            active_by_level[event.risk_level] += 1
            active_by_type[event.risk_type] += 1

        return {
            'active_events_count': len(self.active_events),
            'total_events_count': len(self.event_history),
            'by_level': dict(active_by_level),
            'by_type': dict(active_by_type),
            'risk_counts': dict(self.risk_counts),
            'mandatory_stop': self.mandatory_stop,
        }


# 预定义风险规则
class PredefinedRiskRules:
    """预定义风险规则"""

    @staticmethod
    def position_limit_rule(max_position_value: float) -> RiskRule:
        """持仓限额规则"""
        def check_position(context):
            total_value = context.get('total_position_value', 0)
            return total_value

        return RiskRule(
            name="Position Limit",
            risk_type=RiskType.POSITION_LIMIT,
            risk_levels={
                RiskLevel.MEDIUM: max_position_value * 0.8,
                RiskLevel.HIGH: max_position_value * 0.9,
                RiskLevel.CRITICAL: max_position_value,
            },
            check_function=check_position,
            mitigation_actions={
                RiskLevel.MEDIUM: "Reduce position size",
                RiskLevel.HIGH: "Stop new positions",
                RiskLevel.CRITICAL: "Emergency: Close all positions",
            },
        )

    @staticmethod
    def drawdown_limit_rule(max_drawdown: float) -> RiskRule:
        """回撤限额规则"""
        def check_drawdown(context):
            current_drawdown = context.get('current_drawdown', 0)
            return current_drawdown

        return RiskRule(
            name="Drawdown Limit",
            risk_type=RiskType.DRAWDOWN_LIMIT,
            risk_levels={
                RiskLevel.MEDIUM: max_drawdown * 0.5,
                RiskLevel.HIGH: max_drawdown * 0.8,
                RiskLevel.CRITICAL: max_drawdown,
            },
            check_function=check_drawdown,
            mitigation_actions={
                RiskLevel.MEDIUM: "Reduce position sizes",
                RiskLevel.HIGH: "Stop new entries",
                RiskLevel.CRITICAL: "Close all positions",
            },
        )

    @staticmethod
    def concentration_limit_rule(max_concentration: float) -> RiskRule:
        """集中度风险规则"""
        def check_concentration(context):
            position_weights = context.get('position_weights', {})
            if not position_weights:
                return 0
            return max(position_weights.values())

        return RiskRule(
            name="Concentration Limit",
            risk_type=RiskType.CONCENTRATION_RISK,
            risk_levels={
                RiskLevel.MEDIUM: max_concentration * 0.6,
                RiskLevel.HIGH: max_concentration * 0.8,
                RiskLevel.CRITICAL: max_concentration,
            },
            check_function=check_concentration,
            mitigation_actions={
                RiskLevel.MEDIUM: "Diversify positions",
                RiskLevel.HIGH: "Stop new large positions",
                RiskLevel.CRITICAL: "Reduce largest position",
            },
        )

    @staticmethod
    def volatility_limit_rule(max_volatility: float) -> RiskRule:
        """波动率风险规则"""
        def check_volatility(context):
            current_volatility = context.get('current_volatility', 0)
            return current_volatility

        return RiskRule(
            name="Volatility Limit",
            risk_type=RiskType.VOLATILITY_RISK,
            risk_levels={
                RiskLevel.MEDIUM: max_volatility * 0.7,
                RiskLevel.HIGH: max_volatility * 0.85,
                RiskLevel.CRITICAL: max_volatility,
            },
            check_function=check_volatility,
            mitigation_actions={
                RiskLevel.MEDIUM: "Reduce position sizes",
                RiskLevel.HIGH: "Use stop losses",
                RiskLevel.CRITICAL: "Exit volatile positions",
            },
        )


# 集成风险管理的策略
class RiskManagedStrategy(Strategy):
    """风险管理策略"""

    def __init__(self):
        """初始化策略"""
        super().__init__()

        # 创建风险引擎
        self.risk_engine = RiskEngine()

        # 添加预定义规则
        self._setup_risk_rules()

        # 风险监控
        self.risk_monitoring_enabled = True
        self.last_risk_check = 0

    def _setup_risk_rules(self):
        """设置风险规则"""
        # 持仓限额（不超过总资金的80%）
        self.risk_engine.add_rule(
            PredefinedRiskRules.position_limit_rule(80000)
        )

        # 回撤限额（最大10%）
        self.risk_engine.add_rule(
            PredefinedRiskRules.drawdown_limit_rule(0.10)
        )

        # 集中度限制（单一持仓不超过40%）
        self.risk_engine.add_rule(
            PredefinedRiskRules.concentration_limit_rule(0.40)
        )

        # 波动率限制（年化波动率不超过50%）
        self.risk_engine.add_rule(
            PredefinedRiskRules.volatility_limit_rule(0.50)
        )

    def check_risk(self, context: dict = None) -> bool:
        """
        检查风险

        Parameters
        ----------
        context : dict
            上下文数据

        Returns
        -------
        bool
            是否通过风险检查
        """
        if not self.risk_monitoring_enabled:
            return True

        if context is None:
            context = self._build_risk_context()

        # 评估风险
        risk_events = self.risk_engine.evaluate_risks(context)

        # 处理风险事件
        for event in risk_events:
            self._handle_risk_event(event)

        # 检查是否需要强制停止
        if self.risk_engine.mandatory_stop:
            self._emergency_stop()
            return False

        return True

    def _build_risk_context(self) -> dict:
        """构建风险检查上下文"""
        context = {
            'strategy_id': self.id,
            'timestamp': self.clock.timestamp_ns(),
        }

        # 持仓价值
        total_position_value = 0
        position_weights = {}

        for position in self.portfolio.positions_open:
            value = abs(position.quantity) * self._get_current_price(position.instrument_id)
            total_position_value += value

            # 计算权重
            portfolio_value = self.portfolio.get_total_value()
            if portfolio_value > 0:
                position_weights[str(position.instrument_id)] = value / portfolio_value

        context.update({
            'total_position_value': total_position_value,
            'position_weights': position_weights,
        })

        # 回撤
        context['current_drawdown'] = self._calculate_current_drawdown()

        # 波动率
        context['current_volatility'] = self._calculate_current_volatility()

        return context

    def _handle_risk_event(self, event: RiskEvent):
        """处理风险事件"""
        log_color = {
            RiskLevel.LOW: LogColor.BLUE,
            RiskLevel.MEDIUM: LogColor.YELLOW,
            RiskLevel.HIGH: LogColor.ORANGE,
            RiskLevel.CRITICAL: LogColor.RED,
        }

        color = log_color.get(event.risk_level, LogColor.WHITE)

        self.log.warning(
            f"风险警报 [{event.risk_level.value}]: {event.message}",
            color=color
        )

        # 执行缓解措施
        if event.mitigation_action:
            self._execute_mitigation(event.mitigation_action)

    def _execute_mitigation(self, action: str):
        """执行缓解措施"""
        if action == "Reduce position size":
            self._reduce_positions()
        elif action == "Stop new positions":
            self._stop_new_positions()
        elif action == "Emergency: Close all positions":
            self._close_all_positions()
        elif action == "Diversify positions":
            self._diversify_positions()
        else:
            self.log.warning(f"未知的缓解措施: {action}")

    def _reduce_positions(self):
        """减少仓位"""
        for position in self.portfolio.positions_open:
            # 减少一半仓位
            if abs(position.quantity) > Decimal("0.001"):
                qty = abs(position.quantity) / 2
                side = OrderSide.SELL if position.side == PositionSide.LONG else OrderSide.BUY

                order = MarketOrder(
                    trader_id=self.trader_id,
                    strategy_id=self.id,
                    instrument_id=position.instrument_id,
                    order_side=side,
                    quantity=Quantity(qty),
                )
                self.submit_order(order)

    def _close_all_positions(self):
        """平掉所有仓位"""
        for position in self.portfolio.positions_open:
            if abs(position.quantity) > 0:
                side = OrderSide.SELL if position.side == PositionSide.LONG else OrderSide.BUY

                order = MarketOrder(
                    trader_id=self.trader_id,
                    strategy_id=self.id,
                    instrument_id=position.instrument_id,
                    order_side=side,
                    quantity=position.quantity,
                )
                self.submit_order(order)

    def _stop_new_positions(self):
        """停止开新仓"""
        # 这里可以设置一个标志，在策略逻辑中检查
        self.risk_monitoring_enabled = False
        self.log.warning("风险控制：停止开新仓")

    def _diversify_positions(self):
        """分散投资"""
        # 计算当前集中度
        context = self._build_risk_context()
        concentration = max(context.get('position_weights', {}).values()) if context.get('position_weights') else 0

        if concentration > 0.3:  # 超过30%
            self.log.warning(f"投资组合过于集中: {concentration:.1%}")

    def _emergency_stop(self):
        """紧急停止"""
        self.log.error("触发紧急停止！")
        self._close_all_positions()
        self._stop_new_positions()

    def _get_current_price(self, instrument_id: InstrumentId) -> Decimal:
        """获取当前价格"""
        # 这里应该从缓存获取
        return Decimal("50000.00")

    def _calculate_current_drawdown(self) -> float:
        """计算当前回撤"""
        # 简化实现
        return 0.05  # 5%

    def _calculate_current_volatility(self) -> float:
        """计算当前波动率"""
        # 简化实现
        return 0.15  # 15%

    def on_bar(self, bar: Bar):
        """处理K线数据"""
        # 定期风险检查
        if self.count_bars() % 10 == 0:  # 每10根K线检查一次
            self.check_risk()

    def on_stop(self):
        """策略停止"""
        self.log.info("风险统计:")
        summary = self.risk_engine.get_risk_summary()
        self.log.info(f"  活跃风险事件: {summary['active_events_count']}")
        self.log.info(f"  总风险事件: {summary['total_events_count']}")
        self.log.info(f"  按级别分布: {summary['by_level']}")
        self.log.info(f"  按类型分布: {summary['by_type']}")
```

## 9.3 资金管理策略

### 9.3.1 Kelly 公式

```python
"""
Kelly 资金管理策略
"""

import math
from decimal import Decimal
from typing import Optional


class KellyCriterion:
    """Kelly 公式资金管理"""

    def __init__(self, max_leverage: float = 2.0):
        """
        初始化 Kelly 公式

        Parameters
        ----------
        max_leverage : float
            最大杠杆倍数
        """
        self.max_leverage = max_leverage
        self.win_rate_history = []
        self.win_loss_history = []

    def add_trade_result(self, win_rate: float, win_loss_ratio: float):
        """
        添加交易结果

        Parameters
 ----------
        win_rate : float
            胜率 (0-1)
        win_loss_ratio : float
            胜负比
        """
        self.win_rate_history.append(win_rate)
        self.win_loss_history.append(win_loss_ratio)

        # 保留最近100次交易
        if len(self.win_rate_history) > 100:
            self.win_rate_history.pop(0)
            self.win_loss_history.pop(0)

    def calculate_kelly_fraction(self) -> Optional[float]:
        """
        计算 Kelly 分数

        Returns
        -------
        Optional[float]
            Kelly 分数
        """
        if len(self.win_rate_history) < 10:
            return None  # 需要足够的历史数据

        # 使用历史数据计算平均
        avg_win_rate = sum(self.win_rate_history) / len(self.win_rate_history)

        if avg_win_rate == 0:
            return 0

        # 计算平均胜负比
        avg_win_loss = sum(self.win_loss_history) / len(self.win_loss_history)

        # Kelly 公式: f = (bp - q) / b
        # 其中 b = 平均胜负比, p = 胜率, q = 败率 = 1-p
        kelly_fraction = (avg_win_rate * avg_win_loss - (1 - avg_win_rate)) / avg_win_loss

        # 应用安全系数（25%）
        kelly_fraction *= 0.25

        # 应用最大杠杆限制
        kelly_fraction = min(kelly_fraction, self.max_leverage)

        # 确保非负
        return max(0, kelly_fraction)

    def get_position_size(
        self,
        total_capital: Decimal,
        current_price: Decimal,
        risk_per_trade: Decimal = Decimal('0.02'),
    ) -> Decimal:
        """
        计算建议仓位大小

        Parameters
        ----------
        total_capital : Decimal
            总资金
        current_price : Decimal
            当前价格
        risk_per_trade : Decimal
            单次交易风险比例

        Returns
        -------
        Decimal
            建议仓位数量
        """
        kelly_fraction = self.calculate_kelly_fraction()

        if kelly_fraction is None:
            # 没有足够数据，使用固定比例
            kelly_fraction = 0.5

        # 基于风险调整
        risk_adjusted_fraction = min(kelly_fraction, float(risk_per_trade) * 10)

        # 计算金额
        trade_amount = Decimal(str(risk_adjusted_fraction)) * total_capital

        # 计算数量
        quantity = trade_amount / current_price

        # 四舍五入到6位小数
        return round(quantity, 6)


class FixedFractionalStrategy:
    """固定分数资金管理"""

    def __init__(self, fraction: float = 0.02):
        """
        初始化

        Parameters
        ----------
        fraction : float
            交易比例
        """
        self.fraction = fraction

    def get_position_size(
        self,
        total_capital: Decimal,
        current_price: Decimal,
    ) -> Decimal:
        """计算仓位大小"""
        trade_amount = Decimal(str(self.fraction)) * total_capital
        quantity = trade_amount / current_price
        return round(quantity, 6)


class VolatilityBasedSizing:
    """基于波动率的仓位管理"""

    def __init__(
        self,
        target_volatility: float = 0.15,  # 15% 年化波动率
        lookback_period: int = 20,
    ):
        """
        初始化

        Parameters
        ----------
        target_volatility : float
            目标波动率
        lookback_period : int
            回看周期
        """
        self.target_volatility = target_volatility
        self.lookback_period = lookback_period
        self.price_history = []

    def add_price(self, price: Decimal):
        """添加价格数据"""
        self.price_history.append(price)
        if len(self.price_history) > self.lookback_period:
            self.price_history.pop(0)

    def get_position_size(
        self,
        total_capital: Decimal,
        current_price: Decimal,
    ) -> Decimal:
        """计算仓位大小"""
        if len(self.price_history) < 2:
            return Decimal("0")

        # 计算历史波动率
        returns = []
        for i in range(1, len(self.price_history)):
            ret = (self.price_history[i] - self.price_history[i-1]) / self.price_history[i-1]
            returns.append(ret)

        if not returns:
            return Decimal("0")

        # 年化波动率
        daily_vol = np.std(returns)
        annualized_vol = daily_vol * math.sqrt(365)

        # 调整比例
        if annualized_vol > 0:
            volatility_adjustment = self.target_volatility / annualized_vol
        else:
            volatility_adjustment = 1.0

        # 限制调整范围
        volatility_adjustment = max(0.1, min(volatility_adjustment, 3.0))

        # 计算交易金额
        trade_amount = Decimal(str(volatility_adjustment * 0.05)) * total_capital
        quantity = trade_amount / current_price

        return round(quantity, 6)


# 资金管理策略使用示例
class CapitalManagedStrategy(Strategy):
    """资金管理策略"""

    def __init__(self):
        """初始化策略"""
        super().__init__()

        # 创建资金管理器
        self.kelly = KellyCriterion(max_leverage=1.5)
        self.fixed_fraction = FixedFractionalStrategy(fraction=0.02)
        self.volatility_based = VolatilityBasedSizing()

        # 选择资金管理方法
        self.position_sizing_method = "kelly"  # 可选: "fixed", "kelly", "volatility"

        # 资金
        self.initial_capital = Decimal("100000")
        self.current_capital = self.initial_capital

        # 交易统计
        self.trades_count = 0
        self.winning_trades = 0

    def calculate_position_size(self, current_price: Decimal) -> Decimal:
        """
        计算仓位大小

        Parameters
        ----------
        current_price : Decimal
            当前价格

        Returns
        -------
        Decimal
            仓位数量
        """
        if self.position_sizing_method == "fixed":
            return self.fixed_fraction.get_position_size(
                self.current_capital,
                current_price
            )
        elif self.position_sizing_method == "kelly":
            return self.kelly.get_position_size(
                self.current_capital,
                current_price
            )
        elif self.position_sizing_method == "volatility":
            return self.volatility_based.get_position_size(
                self.current_capital,
                current_price
            )
        else:
            return Decimal("0")

    def on_order_filled(self, event):
        """处理订单成交"""
        # 更新资金
        if event.order_side == OrderSide.SELL:
            # 卖出交易，更新盈亏
            # 这里需要知道开仓价格
            # 简化处理
            pnl = (event.last_px - Decimal("50000")) * event.last_qty
            self.current_capital += pnl

            # 更新交易统计
            self.trades_count += 1
            if pnl > 0:
                self.winning_trades += 1

            # 更新Kelly历史
            win_rate = self.winning_trades / self.trades_count
            win_loss_ratio = abs(pnl / (event.last_px * event.last_qty))
            self.kelly.add_trade_result(win_rate, float(win_loss_ratio))

    def on_bar(self, bar: Bar):
        """处理K线数据"""
        # 更新波动率历史
        self.volatility_based.add_price(bar.close)

        # 示例：每50根K线进行一次交易
        if self.count_bars() % 50 == 0:
            quantity = self.calculate_position_size(bar.close)
            if quantity > 0:
                # 根据信号决定买卖方向
                if self._generate_signal(bar):
                    order = MarketOrder(
                        trader_id=self.trader_id,
                        strategy_id=self.id,
                        instrument_id=self.bar_type.instrument_id,
                        order_side=OrderSide.BUY,
                        quantity=Quantity(quantity),
                    )
                    self.submit_order(order)

    def _generate_signal(self, bar: Bar) -> bool:
        """生成交易信号（示例）"""
        # 简单示例：每隔一段时间买入一次
        return True

    def on_stop(self):
        """策略停止"""
        total_return = (self.current_capital - self.initial_capital) / self.initial_capital
        win_rate = self.winning_trades / self.trades_count if self.trades_count > 0 else 0

        self.log.info("资金管理统计:")
        self.log.info(f"  初始资金: {self.initial_capital}")
        self.log.info(f"  最终资金: {self.current_capital}")
        self.log.info(f"  总收益率: {total_return:.2%}")
        self.log.info(f"  交易次数: {self.trades_count}")
        self.log.info(f"  胜率: {win_rate:.1%}")

        if self.position_sizing_method == "kelly":
            kelly_fraction = self.kelly.calculate_kelly_fraction()
            if kelly_fraction is not None:
                self.log.info(f"  Kelly分数: {kelly_fraction:.2f}")
```

## 9.4 下一步

在本章中，我们学习了：

1. 投资组合管理的基本概念和实现
2. 多策略协调和资金分配
3. 多层风险控制系统
4. 各种资金管理策略
5. 实际应用示例

在下一章中，我们将学习：

1. Parquet 数据目录的使用
2. 高效的数据存储和检索
3. 数据供应商集成
4. 大规模数据处理
5. 数据质量保证

## 9.5 总结

### 关键要点
1. 投资组合管理需要考虑资产配置和风险分散
2. 多层风险控制可以全面管理各种风险
3. 资金管理是长期盈利的关键
4. 动态调整和优化可以提高收益

### 最佳实践
1. 永远使用止损订单
2. 合理分配资金到不同策略
3. 定期检查和调整投资组合
4. 保持交易纪律

## 9.6 参考资料

- [现代投资组合理论](https://www.investopedia.com/terms/m/modernportfoliotheory.asp)
- [Kelly 公式详解](https://www.investopedia.com/articles/trading/041714-guide-kelly-criterion.asp)
- [风险管理框架](https://www.investopedia.com/articles/financial-theory/093015/6-basic-finance-risk-measures.asp)