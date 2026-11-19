# Chapter 9: Portfolio and Risk Management

## Learning Objectives

After completing this chapter, you will:
- Master the basic concepts of portfolio management
- Learn to implement multi-layer risk control
- Understand capital management strategies
- Build risk monitoring systems
- Implement dynamic position management

## 9.1 Portfolio Basics

### 9.1.1 Core Concepts

```python
"""
Portfolio Management Fundamentals
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
    """Asset allocation"""
    instrument_id: InstrumentId
    weight: Decimal  # Weight (0-1)
    target_quantity: Optional[Decimal] = None
    current_quantity: Decimal = Decimal('0')
    market_value: Decimal = Decimal('0')


class PortfolioManager:
    """Portfolio manager"""

    def __init__(
        self,
        initial_capital: Decimal,
        base_currency: str = "USDT",
    ):
        """
        Initialize portfolio manager

        Parameters
        ----------
        initial_capital : Decimal
            Initial capital
        base_currency : str
            Base currency
        """
        self.initial_capital = initial_capital
        self.base_currency = base_currency

        # Current status
        self.current_capital = initial_capital
        self.unrealized_pnl = Decimal('0')
        self.realized_pnl = Decimal('0')

        # Position information
        self.positions = {}  # instrument_id -> Position
        self.allocations = {}  # asset_name -> AssetAllocation

        # History
        self.transaction_history = []
        self.pnl_history = []

        # Risk metrics
        self.max_drawdown = Decimal('0')
        self.current_drawdown = Decimal('0')
        self.sharpe_ratio = Decimal('0')

    def add_asset_allocation(
        self,
        instrument_id: InstrumentId,
        weight: Decimal,
    ):
        """
        Add asset allocation

        Parameters
        ----------
        instrument_id : InstrumentId
            Instrument ID
        weight : Decimal
            Weight (0-1)
        """
        asset_name = str(instrument_id)
        self.allocations[asset_name] = AssetAllocation(
            instrument_id=instrument_id,
            weight=weight
        )

    def update_position(self, position: Position):
        """
        Update position

        Parameters
        ----------
        position : Position
            Position object
        """
        instrument_id = position.instrument_id
        self.positions[instrument_id] = position

        # Update asset allocation
        asset_name = str(instrument_id)
        if asset_name in self.allocations:
            self.allocations[asset_name].current_quantity = position.quantity

    def calculate_total_value(self, prices: Dict[InstrumentId, Decimal]) -> Decimal:
        """
        Calculate total portfolio value

        Parameters
        ----------
        prices : Dict[InstrumentId, Decimal]
            Current prices

        Returns
        -------
        Decimal
            Total value
        """
        total_value = self.current_capital

        for instrument_id, position in self.positions.items():
            if instrument_id in prices:
                market_value = abs(position.quantity) * prices[instrument_id]
                total_value += market_value

                # Update asset allocation
                asset_name = str(instrument_id)
                if asset_name in self.allocations:
                    self.allocations[asset_name].market_value = market_value

        return total_value

    def get_current_allocation(self) -> Dict[str, Decimal]:
        """Get current asset allocation"""
        total_value = self._get_current_market_value()

        allocations = {}
        for asset_name, allocation in self.allocations.items():
            if total_value > 0:
                allocations[asset_name] = allocation.market_value / total_value

        return allocations

    def get_target_allocation(self) -> Dict[str, Decimal]:
        """Get target asset allocation"""
        return {
            name: allocation.weight
            for name, allocation in self.allocations.items()
        }

    def calculate_rebalance_orders(
        self,
        current_prices: Dict[InstrumentId, Decimal],
    ) -> List[Dict]:
        """
        Calculate rebalancing orders

        Parameters
        ----------
        current_prices : Dict[InstrumentId, Decimal]
            Current prices

        Returns
        -------
        List[Dict]
            List of rebalancing orders
        """
        total_value = self.calculate_total_value(current_prices)
        rebalance_orders = []

        for allocation in self.allocations.values():
            instrument_id = allocation.instrument_id
            current_price = current_prices.get(instrument_id)

            if not current_price or current_price <= 0:
                continue

            # Calculate target value
            target_value = total_value * allocation.weight

            # Calculate current value
            current_value = allocation.market_value

            # Calculate difference
            value_diff = target_value - current_value

            # Calculate quantity to adjust
            if abs(value_diff) > total_value * Decimal('0.01'):  # 1% threshold
                quantity_diff = value_diff / current_price

                # Determine order direction
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
        """Get current market value"""
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
        Calculate performance metrics

        Parameters
        ----------
        current_prices : Dict[InstrumentId, Decimal]
            Current prices
        start_time : datetime
            Start time

        Returns
        -------
        Dict
            Performance metrics
        """
        current_time = datetime.utcnow()
        duration_days = (current_time - start_time).days

        total_value = self.calculate_total_value(current_prices)
        total_return = (total_value - self.initial_capital) / self.initial_capital

        # Calculate annualized return
        annualized_return = (1 + total_return) ** (365 / duration_days) - 1

        # Update max drawdown
        self._update_max_drawdown(total_value)

        # Calculate Sharpe ratio (assuming 2% risk-free rate)
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
        """Update maximum drawdown"""
        # Need to maintain historical value records here
        # Simplified implementation
        peak_value = self.initial_capital * Decimal('1.2')  # Assumed peak

        if current_value > peak_value:
            peak_value = current_value
        else:
            drawdown = (peak_value - current_value) / peak_value
            self.current_drawdown = drawdown
            self.max_drawdown = max(self.max_drawdown, drawdown)

    def _calculate_volatility(self, current_prices: Dict[InstrumentId, Decimal]) -> Decimal:
        """Calculate volatility"""
        # Simplified implementation, should use historical data in practice
        return Decimal('0.15')  # 15% annualized volatility

    def generate_portfolio_report(self) -> str:
        """Generate portfolio report"""
        report = f"""
Portfolio Report
=================
Initial Capital: {self.initial_capital}
Current Capital: {self.current_capital}
Realized P&L: {self.realized_pnl}
Unrealized P&L: {self.unrealized_pnl}

Asset Allocation:
- Target Allocation: {self.get_target_allocation()}
- Current Allocation: {self.get_current_allocation()}

Risk Metrics:
- Max Drawdown: {self.max_drawdown:.2%}
- Sharpe Ratio: {self.sharpe_ratio:.2f}

Position Count: {len(self.positions)}
"""
        return report
```

### 9.1.2 Multi-Strategy Portfolio

```python
"""
Multi-Strategy Portfolio Management
"""

from enum import Enum
from typing import Set
from collections import defaultdict


class StrategyType(Enum):
    """Strategy type"""
    MOMENTUM = "Momentum"
    MEAN_REVERSION = "Mean Reversion"
    ARBITRAGE = "Arbitrage"
    MARKET_MAKING = "Market Making"
    TREND_FOLLOWING = "Trend Following"


class MultiStrategyPortfolio:
    """Multi-strategy portfolio"""

    def __init__(self, initial_capital: Decimal):
        """
        Initialize multi-strategy portfolio

        Parameters
        ----------
        initial_capital : Decimal
            Initial capital
        """
        self.initial_capital = initial_capital
        self.current_capital = initial_capital

        # Strategy management
        self.strategies = {}  # strategy_id -> strategy_info
        self.strategy_positions = defaultdict(dict)  # strategy_id -> instrument_id -> position

        # Capital allocation
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
        Add strategy

        Parameters
        ----------
        strategy_id : str
            Strategy ID
        strategy_type : StrategyType
            Strategy type
        allocation_weight : Decimal
            Capital allocation weight
        max_capital : Decimal
            Maximum capital limit
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
        """Update capital allocation"""
        total_weight = sum(
            info['allocation_weight']
            for info in self.strategies.values()
        )

        if total_weight > 0:
            for strategy_id, info in self.strategies.items():
                weight = info['allocation_weight'] / total_weight
                allocated_capital = self.initial_capital * weight

                # Check maximum capital limit
                if info['max_capital']:
                    allocated_capital = min(allocated_capital, info['max_capital'])

                self.strategy_allocations[strategy_id] = allocated_capital

    def update_strategy_position(
        self,
        strategy_id: str,
        position: Position,
    ):
        """
        Update strategy position

        Parameters
        ----------
        strategy_id : str
            Strategy ID
        position : Position
            Position
        """
        instrument_id = position.instrument_id
        self.strategy_positions[strategy_id][instrument_id] = position
        self.strategies[strategy_id]['positions'][instrument_id] = position

    def get_strategy_performance(self, strategy_id: str) -> dict:
        """
        Get strategy performance

        Parameters
        ----------
        strategy_id : str
            Strategy ID

        Returns
        -------
        dict
            Strategy performance data
        """
        if strategy_id not in self.strategies:
            return {}

        info = self.strategies[strategy_id]
        positions = info['positions']

        # Calculate position value
        position_value = Decimal('0')
        unrealized_pnl = Decimal('0')

        for position in positions.values():
            # Should get current price here
            current_price = self._get_current_price(position.instrument_id)
            if current_price:
                position_value += abs(position.quantity) * current_price
                unrealized_pnl += position.unrealized_pnl(self.current_capital)

        # Calculate return rate
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
        """Get portfolio summary"""
        total_allocated = sum(self.strategy_allocations.values())
        total_pnl = sum(
            info['current_pnl'] for info in self.strategies.values()
        )

        strategy_performance = {}
        for strategy_id in self.strategies:
            strategy_performance[strategy_id] = self.get_strategy_performance(strategy_id)

        # Group by type
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
        """Check strategy correlations"""
        # Simplified implementation - based on strategy types
        correlations = {}

        strategy_types = {}
        for strategy_id, info in self.strategies.items():
            strategy_types[strategy_id] = info['type']

        # Calculate type distribution
        type_distribution = defaultdict(list)
        for strategy_id, strategy_type in strategy_types.items():
            type_distribution[strategy_type].append(strategy_id)

        # Correlation analysis (simplified)
        type_names = list(type_distribution.keys())
        for i, type1 in enumerate(type_names):
            correlations[type1] = {}
            for j, type2 in enumerate(type_names):
                if i == j:
                    correlations[type1][type2] = 1.0
                else:
                    # Different strategy types usually have lower correlation
                    correlations[type1][type2] = 0.3 if type1 != type2 else 1.0

        return correlations

    def optimize_strategy_allocations(self) -> dict:
        """
        Optimize strategy capital allocation

        Returns
        -------
        dict
            Optimized allocation
        """
        # Optimize allocation based on historical performance
        strategy_performance = {
            sid: self.get_strategy_performance(sid)
            for sid in self.strategies
        }

        # Filter strategies with performance
        active_strategies = {
            sid: perf for sid, perf in strategy_performance.items()
            if perf['trade_count'] > 0
        }

        if not active_strategies:
            return self.strategy_allocations

        # Calculate Sharpe ratios
        for sid, perf in active_strategies.items():
            # Simplified Sharpe ratio calculation
            return_rate = perf['return_rate']
            volatility = abs(return_rate) * 2  # Simplified assumption
            perf['sharpe_ratio'] = return_rate / volatility if volatility > 0 else 0

        # Adjust weights based on Sharpe ratios
        total_sharpe = sum(abs(p['sharpe_ratio']) for p in active_strategies.values())

        optimized_allocations = {}
        if total_sharpe > 0:
            for sid, perf in active_strategies.items():
                weight = abs(perf['sharpe_ratio']) / total_sharpe
                # Set minimum weight to 5%
                weight = max(weight, Decimal('0.05'))

                # Renormalize
                optimized_allocations[sid] = weight

        # Normalize weights
        total_weight = sum(optimized_allocations.values())
        for sid in optimized_allocations:
            optimized_allocations[sid] /= total_weight

        return optimized_allocations

    def _get_current_price(self, instrument_id: InstrumentId) -> Optional[Decimal]:
        """Get current price (example)"""
        # Should get from cache or market data here
        return Decimal("50000.00")  # Example price


# Using multi-strategy portfolio
class MultiStrategyCoordinator(Strategy):
    """Multi-strategy coordinator"""

    def __init__(self):
        """Initialize coordinator"""
        super().__init__()

        # Create multi-strategy portfolio
        self.portfolio = MultiStrategyPortfolio(
            initial_capital=Decimal("100000")
        )

        # Register strategies
        self._register_strategies()

    def _register_strategies(self):
        """Register strategies"""
        # Momentum strategy
        self.portfolio.add_strategy(
            "momentum_001",
            StrategyType.MOMENTUM,
            Decimal("0.4"),  # 40% capital
            Decimal("50000")  # Max 50k
        )

        # Mean reversion strategy
        self.portfolio.add_strategy(
            "mean_rev_001",
            StrategyType.MEAN_REVERSION,
            Decimal("0.3"),  # 30% capital
            Decimal("40000")  # Max 40k
        )

        # Arbitrage strategy
        self.portfolio.add_strategy(
            "arbitrage_001",
            StrategyType.ARBITRAGE,
            Decimal("0.2"),  # 20% capital
            Decimal("30000")  # Max 30k
        )

        # Market making strategy
        self.portfolio.add_strategy(
            "market_make_001",
            StrategyType.MARKET_MAKING,
            Decimal("0.1"),  # 10% capital
            Decimal("20000")  # Max 20k
        )

    def on_start(self):
        """Strategy start"""
        self.log.info("Multi-strategy coordinator started")
        self.log.info(f"Registered {len(self.portfolio.strategies)} strategies")

        # Output initial allocation
        self.log.info(f"Initial capital allocation: {self.portfolio.strategy_allocations}")

    def on_order_filled(self, event):
        """Handle order filled"""
        # Update strategy statistics
        for strategy_id in self.portfolio.strategies:
            # Need to determine which strategy the order belongs to
            # Simplified: assume event contains strategy ID
            if hasattr(event, 'strategy_id') and event.strategy_id.value == strategy_id:
                self.portfolio.strategies[strategy_id]['trade_count'] += 1
                self.portfolio.strategies[strategy_id]['last_trade_time'] = datetime.utcnow()
                break

    def on_bar(self, bar: Bar):
        """Handle bar data"""
        # Generate reports periodically
        if self.count_bars() % 100 == 0:  # Every 100 bars
            self._generate_portfolio_report()

    def _generate_portfolio_report(self):
        """Generate portfolio report"""
        summary = self.portfolio.get_portfolio_summary()

        self.log.info("=== Multi-Strategy Portfolio Report ===")
        self.log.info(f"Total Capital: {summary['total_capital']:,.2f}")
        self.log.info(f"Allocated: {summary['allocated_capital']:,.2f}")
        self.log.info(f"Unallocated: {summary['unallocated_capital']:,.2f}")
        self.log.info(f"Total P&L: {summary['total_pnl']:,.2f}")

        self.log.info("\nStrategy Performance:")
        for strategy_id, perf in summary['strategy_performance'].items():
            self.log.info(
                f"  {strategy_id}: "
                f"Return {perf['return_rate']:.2%}, "
                f"Trade Count {perf['trade_count']}, "
                f"Position Count {perf['position_count']}"
            )

        self.log.info("\nStatistics by Type:")
        for strategy_type, stats in summary['performance_by_type'].items():
            self.log.info(
                f"  {strategy_type}: "
                f"{stats['strategy_count']} strategies, "
                f"Average Return {stats['avg_return']:.2%}, "
                f"Total P&L {stats['total_pnl']:,.2f}"
            )

    def on_stop(self):
        """Strategy stop"""
        # Optimize capital allocation
        optimized_allocations = self.portfolio.optimize_strategy_allocations()
        self.log.info("\nOptimized Capital Allocation:")
        for strategy_id, weight in optimized_allocations.items():
            self.log.info(f"  {strategy_id}: {weight:.2%}")
```

## 9.2 Risk Management System

### 9.2.1 Multi-Layer Risk Control

```python
"""
Multi-Layer Risk Control System
"""

from enum import Enum
from typing import List, Dict, Optional, Callable
from dataclasses import dataclass
import time


class RiskLevel(Enum):
    """Risk level"""
    LOW = "LOW"
    MEDIUM = "MEDIUM"
    HIGH = "HIGH"
    CRITICAL = "CRITICAL"


class RiskType(Enum):
    """Risk type"""
    POSITION_LIMIT = "Position Limit"
    EXPOSURE_LIMIT = "Exposure Limit"
    DRAWDOWN_LIMIT = "Drawdown Limit"
    CORRELATION_RISK = "Correlation Risk"
    CONCENTRATION_RISK = "Concentration Risk"
    LIQUIDITY_RISK = "Liquidity Risk"
    VOLATILITY_RISK = "Volatility Risk"


@dataclass
class RiskEvent:
    """Risk event"""
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
    """Risk rule"""

    def __init__(
        self,
        name: str,
        risk_type: RiskType,
        risk_levels: Dict[RiskLevel, float],
        check_function: Callable,
        mitigation_actions: Dict[RiskLevel, str],
    ):
        """
        Initialize risk rule

        Parameters
        ----------
        name : str
            Rule name
        risk_type : RiskType
            Risk type
        risk_levels : Dict[RiskLevel, float]
            Risk level thresholds
        check_function : Callable
            Check function
        mitigation_actions : Dict[RiskLevel, str]
            Mitigation actions
        """
        self.name = name
        self.risk_type = risk_type
        self.risk_levels = sorted(risk_levels.items(), key=lambda x: x[1])
        self.check_function = check_function
        self.mitigation_actions = mitigation_actions

    def evaluate(self, context: dict) -> Optional[RiskEvent]:
        """
        Evaluate risk

        Parameters
        ----------
        context : dict
            Context data

        Returns
        -------
        Optional[RiskEvent]
            Risk event
        """
        current_value = self.check_function(context)
        if current_value is None:
            return None

        # Determine risk level
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
    """Risk engine"""

    def __init__(self):
        """Initialize risk engine"""
        self.rules: List[RiskRule] = []
        self.active_events: List[RiskEvent] = []
        self.event_history: List[RiskEvent] = []
        self.mandatory_stop = False

        # Risk counters
        self.risk_counts = defaultdict(int)

    def add_rule(self, rule: RiskRule):
        """Add risk rule"""
        self.rules.append(rule)
        self.log.info(f"Added risk rule: {rule.name}")

    def evaluate_risks(self, context: dict) -> List[RiskEvent]:
        """
        Evaluate all risks

        Parameters
        ----------
        context : dict
            Context data

        Returns
        -------
        List[RiskEvent]
            List of risk events
        """
        new_events = []

        for rule in self.rules:
            event = rule.evaluate(context)
            if event:
                new_events.append(event)
                self.active_events.append(event)
                self.event_history.append(event)
                self.risk_counts[event.risk_type] += 1

                # Check if mandatory stop is needed
                if event.risk_level == RiskLevel.CRITICAL:
                    self.mandatory_stop = True
                    self.log.critical(f"Critical risk triggered: {event.message}")

        return new_events

    def get_active_events(self) -> List[RiskEvent]:
        """Get active risk events"""
        return [e for e in self.active_events if e]

    def clear_expired_events(self, expire_minutes: int = 30):
        """Clear expired events"""
        cutoff_time = datetime.utcnow() - timedelta(minutes=expire_minutes)

        self.active_events = [
            e for e in self.active_events
            if e.timestamp > cutoff_time
        ]

    def get_risk_summary(self) -> dict:
        """Get risk summary"""
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


# Predefined risk rules
class PredefinedRiskRules:
    """Predefined risk rules"""

    @staticmethod
    def position_limit_rule(max_position_value: float) -> RiskRule:
        """Position limit rule"""
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
        """Drawdown limit rule"""
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
        """Concentration risk rule"""
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
        """Volatility risk rule"""
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


# Strategy with integrated risk management
class RiskManagedStrategy(Strategy):
    """Risk management strategy"""

    def __init__(self):
        """Initialize strategy"""
        super().__init__()

        # Create risk engine
        self.risk_engine = RiskEngine()

        # Add predefined rules
        self._setup_risk_rules()

        # Risk monitoring
        self.risk_monitoring_enabled = True
        self.last_risk_check = 0

    def _setup_risk_rules(self):
        """Set up risk rules"""
        # Position limit (not exceeding 80% of total capital)
        self.risk_engine.add_rule(
            PredefinedRiskRules.position_limit_rule(80000)
        )

        # Drawdown limit (maximum 10%)
        self.risk_engine.add_rule(
            PredefinedRiskRules.drawdown_limit_rule(0.10)
        )

        # Concentration limit (single position not exceeding 40%)
        self.risk_engine.add_rule(
            PredefinedRiskRules.concentration_limit_rule(0.40)
        )

        # Volatility limit (annualized volatility not exceeding 50%)
        self.risk_engine.add_rule(
            PredefinedRiskRules.volatility_limit_rule(0.50)
        )

    def check_risk(self, context: dict = None) -> bool:
        """
        Check risk

        Parameters
        ----------
        context : dict
            Context data

        Returns
        -------
        bool
            Whether risk check passed
        """
        if not self.risk_monitoring_enabled:
            return True

        if context is None:
            context = self._build_risk_context()

        # Evaluate risks
        risk_events = self.risk_engine.evaluate_risks(context)

        # Handle risk events
        for event in risk_events:
            self._handle_risk_event(event)

        # Check if mandatory stop is needed
        if self.risk_engine.mandatory_stop:
            self._emergency_stop()
            return False

        return True

    def _build_risk_context(self) -> dict:
        """Build risk check context"""
        context = {
            'strategy_id': self.id,
            'timestamp': self.clock.timestamp_ns(),
        }

        # Position values
        total_position_value = 0
        position_weights = {}

        for position in self.portfolio.positions_open:
            value = abs(position.quantity) * self._get_current_price(position.instrument_id)
            total_position_value += value

            # Calculate weights
            portfolio_value = self.portfolio.get_total_value()
            if portfolio_value > 0:
                position_weights[str(position.instrument_id)] = value / portfolio_value

        context.update({
            'total_position_value': total_position_value,
            'position_weights': position_weights,
        })

        # Drawdown
        context['current_drawdown'] = self._calculate_current_drawdown()

        # Volatility
        context['current_volatility'] = self._calculate_current_volatility()

        return context

    def _handle_risk_event(self, event: RiskEvent):
        """Handle risk event"""
        log_color = {
            RiskLevel.LOW: LogColor.BLUE,
            RiskLevel.MEDIUM: LogColor.YELLOW,
            RiskLevel.HIGH: LogColor.ORANGE,
            RiskLevel.CRITICAL: LogColor.RED,
        }

        color = log_color.get(event.risk_level, LogColor.WHITE)

        self.log.warning(
            f"Risk Alert [{event.risk_level.value}]: {event.message}",
            color=color
        )

        # Execute mitigation action
        if event.mitigation_action:
            self._execute_mitigation(event.mitigation_action)

    def _execute_mitigation(self, action: str):
        """Execute mitigation action"""
        if action == "Reduce position size":
            self._reduce_positions()
        elif action == "Stop new positions":
            self._stop_new_positions()
        elif action == "Emergency: Close all positions":
            self._close_all_positions()
        elif action == "Diversify positions":
            self._diversify_positions()
        else:
            self.log.warning(f"Unknown mitigation action: {action}")

    def _reduce_positions(self):
        """Reduce positions"""
        for position in self.portfolio.positions_open:
            # Reduce position by half
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
        """Close all positions"""
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
        """Stop opening new positions"""
        # Can set a flag here to check in strategy logic
        self.risk_monitoring_enabled = False
        self.log.warning("Risk Control: Stop opening new positions")

    def _diversify_positions(self):
        """Diversify investments"""
        # Calculate current concentration
        context = self._build_risk_context()
        concentration = max(context.get('position_weights', {}).values()) if context.get('position_weights') else 0

        if concentration > 0.3:  # Exceeds 30%
            self.log.warning(f"Portfolio too concentrated: {concentration:.1%}")

    def _emergency_stop(self):
        """Emergency stop"""
        self.log.error("Emergency stop triggered!")
        self._close_all_positions()
        self._stop_new_positions()

    def _get_current_price(self, instrument_id: InstrumentId) -> Decimal:
        """Get current price"""
        # Should get from cache
        return Decimal("50000.00")

    def _calculate_current_drawdown(self) -> float:
        """Calculate current drawdown"""
        # Simplified implementation
        return 0.05  # 5%

    def _calculate_current_volatility(self) -> float:
        """Calculate current volatility"""
        # Simplified implementation
        return 0.15  # 15%

    def on_bar(self, bar: Bar):
        """Handle bar data"""
        # Periodic risk check
        if self.count_bars() % 10 == 0:  # Check every 10 bars
            self.check_risk()

    def on_stop(self):
        """Strategy stop"""
        self.log.info("Risk Statistics:")
        summary = self.risk_engine.get_risk_summary()
        self.log.info(f"  Active risk events: {summary['active_events_count']}")
        self.log.info(f"  Total risk events: {summary['total_events_count']}")
        self.log.info(f"  Distribution by level: {summary['by_level']}")
        self.log.info(f"  Distribution by type: {summary['by_type']}")
```

## 9.3 Capital Management Strategies

### 9.3.1 Kelly Formula

```python
"""
Kelly Capital Management Strategy
"""

import math
from decimal import Decimal
from typing import Optional


class KellyCriterion:
    """Kelly formula capital management"""

    def __init__(self, max_leverage: float = 2.0):
        """
        Initialize Kelly formula

        Parameters
        ----------
        max_leverage : float
            Maximum leverage multiplier
        """
        self.max_leverage = max_leverage
        self.win_rate_history = []
        self.win_loss_history = []

    def add_trade_result(self, win_rate: float, win_loss_ratio: float):
        """
        Add trade result

        Parameters
        ----------
        win_rate : float
            Win rate (0-1)
        win_loss_ratio : float
            Win-loss ratio
        """
        self.win_rate_history.append(win_rate)
        self.win_loss_history.append(win_loss_ratio)

        # Keep last 100 trades
        if len(self.win_rate_history) > 100:
            self.win_rate_history.pop(0)
            self.win_loss_history.pop(0)

    def calculate_kelly_fraction(self) -> Optional[float]:
        """
        Calculate Kelly fraction

        Returns
        -------
        Optional[float]
            Kelly fraction
        """
        if len(self.win_rate_history) < 10:
            return None  # Need sufficient historical data

        # Use historical data to calculate average
        avg_win_rate = sum(self.win_rate_history) / len(self.win_rate_history)

        if avg_win_rate == 0:
            return 0

        # Calculate average win-loss ratio
        avg_win_loss = sum(self.win_loss_history) / len(self.win_loss_history)

        # Kelly formula: f = (bp - q) / b
        # where b = average win-loss ratio, p = win rate, q = loss rate = 1-p
        kelly_fraction = (avg_win_rate * avg_win_loss - (1 - avg_win_rate)) / avg_win_loss

        # Apply safety factor (25%)
        kelly_fraction *= 0.25

        # Apply maximum leverage limit
        kelly_fraction = min(kelly_fraction, self.max_leverage)

        # Ensure non-negative
        return max(0, kelly_fraction)

    def get_position_size(
        self,
        total_capital: Decimal,
        current_price: Decimal,
        risk_per_trade: Decimal = Decimal('0.02'),
    ) -> Decimal:
        """
        Calculate recommended position size

        Parameters
        ----------
        total_capital : Decimal
            Total capital
        current_price : Decimal
            Current price
        risk_per_trade : Decimal
            Risk ratio per trade

        Returns
        -------
        Decimal
            Recommended position quantity
        """
        kelly_fraction = self.calculate_kelly_fraction()

        if kelly_fraction is None:
            # No sufficient data, use fixed ratio
            kelly_fraction = 0.5

        # Risk-based adjustment
        risk_adjusted_fraction = min(kelly_fraction, float(risk_per_trade) * 10)

        # Calculate amount
        trade_amount = Decimal(str(risk_adjusted_fraction)) * total_capital

        # Calculate quantity
        quantity = trade_amount / current_price

        # Round to 6 decimal places
        return round(quantity, 6)


class FixedFractionalStrategy:
    """Fixed fractional capital management"""

    def __init__(self, fraction: float = 0.02):
        """
        Initialize

        Parameters
        ----------
        fraction : float
            Trading fraction
        """
        self.fraction = fraction

    def get_position_size(
        self,
        total_capital: Decimal,
        current_price: Decimal,
    ) -> Decimal:
        """Calculate position size"""
        trade_amount = Decimal(str(self.fraction)) * total_capital
        quantity = trade_amount / current_price
        return round(quantity, 6)


class VolatilityBasedSizing:
    """Volatility-based position management"""

    def __init__(
        self,
        target_volatility: float = 0.15,  # 15% annualized volatility
        lookback_period: int = 20,
    ):
        """
        Initialize

        Parameters
        ----------
        target_volatility : float
            Target volatility
        lookback_period : int
            Lookback period
        """
        self.target_volatility = target_volatility
        self.lookback_period = lookback_period
        self.price_history = []

    def add_price(self, price: Decimal):
        """Add price data"""
        self.price_history.append(price)
        if len(self.price_history) > self.lookback_period:
            self.price_history.pop(0)

    def get_position_size(
        self,
        total_capital: Decimal,
        current_price: Decimal,
    ) -> Decimal:
        """Calculate position size"""
        if len(self.price_history) < 2:
            return Decimal("0")

        # Calculate historical volatility
        returns = []
        for i in range(1, len(self.price_history)):
            ret = (self.price_history[i] - self.price_history[i-1]) / self.price_history[i-1]
            returns.append(ret)

        if not returns:
            return Decimal("0")

        # Annualized volatility
        daily_vol = np.std(returns)
        annualized_vol = daily_vol * math.sqrt(365)

        # Adjustment ratio
        if annualized_vol > 0:
            volatility_adjustment = self.target_volatility / annualized_vol
        else:
            volatility_adjustment = 1.0

        # Limit adjustment range
        volatility_adjustment = max(0.1, min(volatility_adjustment, 3.0))

        # Calculate trade amount
        trade_amount = Decimal(str(volatility_adjustment * 0.05)) * total_capital
        quantity = trade_amount / current_price

        return round(quantity, 6)


# Capital management strategy usage example
class CapitalManagedStrategy(Strategy):
    """Capital management strategy"""

    def __init__(self):
        """Initialize strategy"""
        super().__init__()

        # Create capital manager
        self.kelly = KellyCriterion(max_leverage=1.5)
        self.fixed_fraction = FixedFractionalStrategy(fraction=0.02)
        self.volatility_based = VolatilityBasedSizing()

        # Select capital management method
        self.position_sizing_method = "kelly"  # Options: "fixed", "kelly", "volatility"

        # Capital
        self.initial_capital = Decimal("100000")
        self.current_capital = self.initial_capital

        # Trading statistics
        self.trades_count = 0
        self.winning_trades = 0

    def calculate_position_size(self, current_price: Decimal) -> Decimal:
        """
        Calculate position size

        Parameters
        ----------
        current_price : Decimal
            Current price

        Returns
        -------
        Decimal
            Position quantity
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
        """Handle order filled"""
        # Update capital
        if event.order_side == OrderSide.SELL:
            # Sell trade, update P&L
            # Need to know opening price here
            # Simplified handling
            pnl = (event.last_px - Decimal("50000")) * event.last_qty
            self.current_capital += pnl

            # Update trade statistics
            self.trades_count += 1
            if pnl > 0:
                self.winning_trades += 1

            # Update Kelly history
            win_rate = self.winning_trades / self.trades_count
            win_loss_ratio = abs(pnl / (event.last_px * event.last_qty))
            self.kelly.add_trade_result(win_rate, float(win_loss_ratio))

    def on_bar(self, bar: Bar):
        """Handle bar data"""
        # Update volatility history
        self.volatility_based.add_price(bar.close)

        # Example: trade every 50 bars
        if self.count_bars() % 50 == 0:
            quantity = self.calculate_position_size(bar.close)
            if quantity > 0:
                # Determine buy/sell direction based on signal
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
        """Generate trading signal (example)"""
        # Simple example: buy periodically
        return True

    def on_stop(self):
        """Strategy stop"""
        total_return = (self.current_capital - self.initial_capital) / self.initial_capital
        win_rate = self.winning_trades / self.trades_count if self.trades_count > 0 else 0

        self.log.info("Capital Management Statistics:")
        self.log.info(f"  Initial Capital: {self.initial_capital}")
        self.log.info(f"  Final Capital: {self.current_capital}")
        self.log.info(f"  Total Return: {total_return:.2%}")
        self.log.info(f"  Trade Count: {self.trades_count}")
        self.log.info(f"  Win Rate: {win_rate:.1%}")

        if self.position_sizing_method == "kelly":
            kelly_fraction = self.kelly.calculate_kelly_fraction()
            if kelly_fraction is not None:
                self.log.info(f"  Kelly Fraction: {kelly_fraction:.2f}")
```

## 9.4 Next Steps

In this chapter, we learned:

1. Basic concepts and implementation of portfolio management
2. Multi-strategy coordination and capital allocation
3. Multi-layer risk control system
4. Various capital management strategies
5. Practical application examples

In the next chapter, we will learn:

1. Using Parquet data directories
2. Efficient data storage and retrieval
3. Data provider integration
4. Large-scale data processing
5. Data quality assurance

## 9.5 Summary

### Key Points
1. Portfolio management needs to consider asset allocation and risk diversification
2. Multi-layer risk control can comprehensively manage various risks
3. Capital management is key to long-term profitability
4. Dynamic adjustment and optimization can improve returns

### Best Practices
1. Always use stop-loss orders
2. Allocate capital reasonably to different strategies
3. Regularly check and adjust the portfolio
4. Maintain trading discipline

## 9.6 References

- [Modern Portfolio Theory](https://www.investopedia.com/terms/m/modernportfoliotheory.asp)
- [Kelly Formula Explained](https://www.investopedia.com/articles/trading/041714-guide-kelly-criterion.asp)
- [Risk Management Framework](https://www.investopedia.com/articles/financial-theory/093015/6-basic-finance-risk-measures.asp)