---
title: "Chapter 20: Complete Project Example 2 - Multi-Strategy Quantitative Fund"
---

# Chapter 20: Complete Project Example 2 - Multi-Strategy Quantitative Fund

## Learning Objectives

- Build a multi-strategy quantitative trading fund system
- Implement portfolio management and capital allocation for strategies
- Master fund-level risk control and management
- Learn to build a complete quantitative investment management system

## 20.1 Project Overview

### 20.1.1 Fund Architecture Design

This project is a quantitative trading fund system designed for institutional investors with the following features:

- **Multi-Strategy Architecture**: Supports parallel operation of various strategies including trend-following, arbitrage, and market-making
- **Intelligent Capital Allocation**: Dynamic capital allocation based on risk budgeting and Kelly criterion
- **Multi-Level Risk Management**: Three-tier risk management at strategy, portfolio, and fund levels
- **Performance Attribution**: Detailed return attribution and risk analysis
- **Institutional-Grade Reporting**: Regulatory-compliant report generation

```mermaid
graph TB
    subgraph "Fund Manager"
        A[Fund Manager] --> B[Capital Allocator]
        B --> C[Strategy Pool]
    end

    subgraph "Strategy Layer"
        C --> D[Trend Strategy]
        C --> E[Arbitrage Strategy]
        C --> F[Market Making Strategy]
        C --> G[Event-Driven Strategy]
    end

    subgraph "Risk Management"
        H[Strategy Risk Control] --> I[Portfolio Risk Control]
        I --> J[Fund Risk Control]
    end

    subgraph "Execution Layer"
        K[Execution Engine] --> L[Multi-Exchange Connection]
        L --> M[Smart Order Routing]
    end

    subgraph "Analysis & Reporting"
        N[Performance Analysis] --> O[Risk Analysis]
        O --> P[Attribution Analysis]
        P --> Q[Regulatory Reports]
    end
```

## 20.2 Core Architecture Implementation

### 20.2.1 Fund Manager

```python
# src/fund/fund_manager.py
from typing import Dict, List, Optional, Tuple
from dataclasses import dataclass, field
from decimal import Decimal
from datetime import datetime, timedelta
import asyncio
import numpy as np

@dataclass
class StrategyAllocation:
    """Strategy capital allocation"""
    strategy_id: str
    strategy_name: str
    allocated_capital: Decimal
    weight: Decimal  # Allocation weight
    target_weight: Decimal  # Target weight
    min_weight: Decimal    # Minimum weight
    max_weight: Decimal    # Maximum weight
    risk_budget: Decimal   # Risk budget
    last_rebalance: datetime = field(default_factory=datetime.now)

@dataclass
class FundMetrics:
    """Fund metrics"""
    total_aum: Decimal  # Total assets under management
    total_pnl: Decimal  # Total profit and loss
    daily_pnl: Decimal  # Daily profit and loss
    mtd_return: Decimal  # Month-to-date return
    ytd_return: Decimal  # Year-to-date return
    sharpe_ratio: float  # Sharpe ratio
    max_drawdown: Decimal  # Maximum drawdown
    var_95: Decimal  # 95% VaR
    total_exposure: Decimal  # Total exposure
    leverage: float  # Leverage ratio

class FundManager:
    """Fund Manager"""

    def __init__(self, config: dict):
        self.config = config

        # Fund parameters
        self.initial_capital = Decimal(str(config['initial_capital']))
        self.max_leverage = Decimal(str(config.get('max_leverage', '3.0')))
        self.rebalance_frequency = config.get('rebalance_frequency', 'daily')  # daily, weekly, monthly
        self.min_cash_reserve = Decimal(str(config.get('min_cash_reserve', '0.05')))  # 5% cash reserve

        # Strategy management
        self.strategies = {}  # {strategy_id: strategy_instance}
        self.allocations = {}  # {strategy_id: StrategyAllocation}
        self.strategy_performance = {}  # {strategy_id: performance_metrics}

        # Capital management
        self.cash = self.initial_capital
        self.reserved_cash = self.initial_capital * self.min_cash_reserve
        self.available_cash = self.cash - self.reserved_cash

        # Risk management
        self.risk_manager = FundRiskManager(config.get('risk', {}))
        self.portfolio_optimizer = PortfolioOptimizer(config.get('optimization', {}))

        # Performance tracking
        self.performance_tracker = FundPerformanceTracker()
        self.attribution_analyzer = AttributionAnalyzer()

        # Status
        self.is_running = False
        self.last_rebalance = None

    async def initialize(self):
        """Initialize the fund"""
        # Initialize strategies
        await self._initialize_strategies()

        # Initial capital allocation
        await self._initialize_allocations()

        # Start monitoring
        asyncio.create_task(self._monitoring_loop())

        self.is_running = True
        self.log.info(f"Fund initialized with AUM: {self.initial_capital}")

    async def _initialize_strategies(self):
        """Initialize all strategies"""
        strategy_configs = self.config.get('strategies', {})

        for strategy_id, config in strategy_configs.items():
            strategy = await self._create_strategy(strategy_id, config)
            self.strategies[strategy_id] = strategy

            # Initialize strategy performance tracking
            self.strategy_performance[strategy_id] = {
                'total_pnl': Decimal('0'),
                'daily_pnl': Decimal('0'),
                'positions': {},
                'trades': [],
                'risk_metrics': {}
            }

    async def _create_strategy(self, strategy_id: str, config: dict):
        """Create strategy instance"""
        strategy_type = config['type']
        strategy_config = config['config']

        if strategy_type == 'trend_following':
            from src.strategies.trend_following import TrendFollowingStrategy
            strategy = TrendFollowingStrategy(strategy_config)
        elif strategy_type == 'statistical_arbitrage':
            from src.strategies.stat_arb import StatisticalArbitrageStrategy
            strategy = StatisticalArbitrageStrategy(strategy_config)
        elif strategy_type == 'market_making':
            from src.strategies.market_making import MarketMakingStrategy
            strategy = MarketMakingStrategy(strategy_config)
        else:
            raise ValueError(f"Unknown strategy type: {strategy_type}")

        await strategy.initialize()
        return strategy

    async def _initialize_allocations(self):
        """Initialize capital allocation"""
        strategy_configs = self.config.get('strategies', {})
        total_weight = sum(config.get('weight', 1) for config in strategy_configs.values())

        for strategy_id, config in strategy_configs.items():
            weight = Decimal(str(config.get('weight', 1))) / Decimal(str(total_weight))
            target_weight = Decimal(str(config.get('target_weight', weight)))
            min_weight = Decimal(str(config.get('min_weight', '0')))
            max_weight = Decimal(str(config.get('max_weight', '1')))

            allocated_capital = self.available_cash * weight
            risk_budget = Decimal(str(config.get('risk_budget', '0.02')))  # Default 2%

            allocation = StrategyAllocation(
                strategy_id=strategy_id,
                strategy_name=config.get('name', strategy_id),
                allocated_capital=allocated_capital,
                weight=weight,
                target_weight=target_weight,
                min_weight=min_weight,
                max_weight=max_weight,
                risk_budget=risk_budget
            )

            self.allocations[strategy_id] = allocation

            # Allocate capital to strategy
            await self._allocate_capital_to_strategy(strategy_id, allocated_capital)

    async def _allocate_capital_to_strategy(self, strategy_id: str, capital: Decimal):
        """Allocate capital to strategy"""
        if strategy_id in self.strategies:
            strategy = self.strategies[strategy_id]
            await strategy.set_capital(capital)
            self.log.info(f"Allocated {capital} to strategy {strategy_id}")

    async def start_trading(self):
        """Start trading"""
        if not self.is_running:
            raise RuntimeError("Fund not initialized")

        # Start all strategies
        for strategy_id, strategy in self.strategies.items():
            await strategy.start()
            self.log.info(f"Started strategy: {strategy_id}")

        # Start rebalancing task
        asyncio.create_task(self._rebalance_loop())

        self.log.info("Fund trading started")

    async def stop_trading(self):
        """Stop trading"""
        # Stop all strategies
        for strategy_id, strategy in self.strategies.items():
            await strategy.stop()
            self.log.info(f"Stopped strategy: {strategy_id}")

        self.is_running = False
        self.log.info("Fund trading stopped")

    async def _rebalance_loop(self):
        """Rebalancing loop"""
        while self.is_running:
            try:
                # Check if rebalancing is needed
                if self._should_rebalance():
                    await self._perform_rebalance()

                # Wait for next check
                if self.rebalance_frequency == 'daily':
                    await asyncio.sleep(3600)  # Check every hour
                elif self.rebalance_frequency == 'weekly':
                    await asyncio.sleep(3600 * 6)  # Check every 6 hours
                else:  # monthly
                    await asyncio.sleep(3600 * 24)  # Check every day

            except Exception as e:
                self.log.error(f"Rebalance error: {e}")
                await asyncio.sleep(300)  # Retry after 5 minutes

    def _should_rebalance(self) -> bool:
        """Determine if rebalancing is needed"""
        now = datetime.now()

        # Check if it's time to rebalance
        if self.last_rebalance is None:
            return True

        if self.rebalance_frequency == 'daily':
            # Daily after market close
            return now.hour >= 16 and now.date() != self.last_rebalance.date()
        elif self.rebalance_frequency == 'weekly':
            # Every Friday after market close
            return now.weekday() == 4 and now.hour >= 16 and now.date() != self.last_rebalance.date()
        elif self.rebalance_frequency == 'monthly':
            # Last trading day of month
            return now.day >= 28 and now.date() != self.last_rebalance.date()

        return False

    async def _perform_rebalance(self):
        """Perform rebalancing"""
        self.log.info("Starting portfolio rebalance...")

        # Calculate optimal weights
        optimal_weights = await self._calculate_optimal_weights()

        # Calculate current weights
        current_weights = self._calculate_current_weights()

        # Generate rebalancing trades
        rebalance_trades = self._generate_rebalance_trades(current_weights, optimal_weights)

        # Risk check
        if await self.risk_manager.validate_rebalance(rebalance_trades):
            # Execute rebalancing
            await self._execute_rebalance_trades(rebalance_trades)
            self.last_rebalance = datetime.now()
            self.log.info("Rebalance completed successfully")
        else:
            self.log.warning("Rebalance failed risk validation")

    async def _calculate_optimal_weights(self) -> Dict[str, Decimal]:
        """Calculate optimal weights"""
        # Get strategy expected returns and risks
        expected_returns = {}
        covariance_matrix = {}

        for strategy_id in self.strategies.keys():
            perf = self.strategy_performance[strategy_id]
            # Use historical returns to calculate expected returns
            returns = self._calculate_strategy_returns(strategy_id)
            expected_returns[strategy_id] = np.mean(returns) if returns else 0.01

        # Build covariance matrix
        for id1 in self.strategies.keys():
            covariance_matrix[id1] = {}
            for id2 in self.strategies.keys():
                covariance_matrix[id1][id2] = self._calculate_covariance(id1, id2)

        # Use mean-variance optimization
        return self.portfolio_optimizer.optimize(
            expected_returns=expected_returns,
            covariance_matrix=covariance_matrix,
            constraints={
                'sum_to_one': True,
                'long_only': True,
                'min_weights': {sid: self.allocations[sid].min_weight for sid in self.strategies.keys()},
                'max_weights': {sid: self.allocations[sid].max_weight for sid in self.strategies.keys()}
            }
        )

    def _calculate_current_weights(self) -> Dict[str, Decimal]:
        """Calculate current weights"""
        current_values = {}
        total_value = self.get_total_aum()

        for strategy_id in self.strategies.keys():
            strategy_value = self._get_strategy_value(strategy_id)
            current_weights[strategy_id] = strategy_value / total_value if total_value > 0 else Decimal('0')

        return current_weights

    def _generate_rebalance_trades(self, current_weights: Dict[str, Decimal],
                                   target_weights: Dict[str, Decimal]) -> List[dict]:
        """Generate rebalancing trades"""
        trades = []
        total_value = self.get_total_aum()

        for strategy_id in self.strategies.keys():
            current_weight = current_weights.get(strategy_id, Decimal('0'))
            target_weight = target_weights.get(strategy_id, Decimal('0'))

            # Calculate weight difference
            weight_diff = target_weight - current_weight

            # Generate trade if difference exceeds threshold (0.5%)
            if abs(weight_diff) > Decimal('0.005'):
                trade_value = total_value * weight_diff
                trades.append({
                    'strategy_id': strategy_id,
                    'action': 'allocate' if weight_diff > 0 else 'deallocate',
                    'value': abs(trade_value),
                    'weight_diff': weight_diff
                })

        return trades

    async def _execute_rebalance_trades(self, trades: List[dict]):
        """Execute rebalancing trades"""
        for trade in trades:
            strategy_id = trade['strategy_id']
            strategy = self.strategies[strategy_id]

            if trade['action'] == 'allocate':
                # Add capital
                await strategy.add_capital(trade['value'])
            else:
                # Reduce capital
                await strategy.reduce_capital(trade['value'])

    def get_total_aum(self) -> Decimal:
        """Get total assets under management"""
        total = self.cash  # Cash

        # Add value of all strategies
        for strategy_id in self.strategies.keys():
            total += self._get_strategy_value(strategy_id)

        return total

    def _get_strategy_value(self, strategy_id: str) -> Decimal:
        """Get strategy value"""
        if strategy_id in self.strategies:
            return self.strategies[strategy_id].get_portfolio_value()
        return Decimal('0')

    def get_fund_metrics(self) -> FundMetrics:
        """Get fund metrics"""
        total_aum = self.get_total_aum()

        # Calculate returns
        total_pnl = total_aum - self.initial_capital
        daily_pnl = self._calculate_daily_pnl()

        # Calculate return metrics
        mtd_return = self._calculate_mtd_return()
        ytd_return = self._calculate_ytd_return()

        # Calculate risk metrics
        returns = self._calculate_fund_returns()
        sharpe_ratio = self._calculate_sharpe_ratio(returns)
        max_drawdown = self._calculate_max_drawdown()
        var_95 = self._calculate_var(returns, 0.05)

        # Calculate exposure and leverage
        total_exposure = self._calculate_total_exposure()
        leverage = float(total_exposure / total_aum) if total_aum > 0 else 1.0

        return FundMetrics(
            total_aum=total_aum,
            total_pnl=total_pnl,
            daily_pnl=daily_pnl,
            mtd_return=mtd_return,
            ytd_return=ytd_return,
            sharpe_ratio=sharpe_ratio,
            max_drawdown=max_drawdown,
            var_95=var_95,
            total_exposure=total_exposure,
            leverage=leverage
        )

    def get_strategy_performance(self) -> Dict[str, dict]:
        """Get performance of all strategies"""
        performance = {}

        for strategy_id in self.strategies.keys():
            strategy_perf = self.strategy_performance[strategy_id]
            strategy = self.strategies[strategy_id]

            performance[strategy_id] = {
                'name': self.allocations[strategy_id].strategy_name,
                'allocated_capital': float(self.allocations[strategy_id].allocated_capital),
                'current_weight': float(self._get_strategy_value(strategy_id) / self.get_total_aum()),
                'target_weight': float(self.allocations[strategy_id].target_weight),
                'total_pnl': float(strategy_perf['total_pnl']),
                'daily_pnl': float(strategy_perf['daily_pnl']),
                'sharpe_ratio': strategy.get_sharpe_ratio(),
                'max_drawdown': strategy.get_max_drawdown(),
                'position_count': len(strategy.get_positions()),
                'trade_count': len(strategy_perf['trades'])
            }

        return performance

    async def _monitoring_loop(self):
        """Monitoring loop"""
        while self.is_running:
            try:
                # Update performance metrics
                await self._update_performance_metrics()

                # Risk monitoring
                await self.risk_manager.monitor_risk(self)

                # Generate reports (hourly)
                if datetime.now().minute == 0:
                    await self._generate_hourly_report()

                await asyncio.sleep(60)  # Update every minute

            except Exception as e:
                self.log.error(f"Monitoring error: {e}")
                await asyncio.sleep(60)

    async def _update_performance_metrics(self):
        """Update performance metrics"""
        for strategy_id, strategy in self.strategies.items():
            # Update strategy performance
            perf = self.strategy_performance[strategy_id]
            perf['total_pnl'] = Decimal(str(strategy.get_total_pnl()))
            perf['daily_pnl'] = Decimal(str(strategy.get_daily_pnl()))
            perf['positions'] = strategy.get_positions()
            perf['risk_metrics'] = strategy.get_risk_metrics()

    async def _generate_hourly_report(self):
        """Generate hourly report"""
        fund_metrics = self.get_fund_metrics()
        strategy_perf = self.get_strategy_performance()

        report = {
            'timestamp': datetime.now().isoformat(),
            'fund_metrics': {
                'total_aum': float(fund_metrics.total_aum),
                'total_pnl': float(fund_metrics.total_pnl),
                'daily_pnl': float(fund_metrics.daily_pnl),
                'leverage': fund_metrics.leverage,
                'max_drawdown': float(fund_metrics.max_drawdown)
            },
            'strategy_performance': strategy_perf
        }

        # Save report
        await self._save_report(report, 'hourly')

        # Check alert conditions
        await self._check_alert_conditions(fund_metrics)

    async def _check_alert_conditions(self, metrics: FundMetrics):
        """Check alert conditions"""
        # Check leverage
        if metrics.leverage > float(self.max_leverage):
            await self._send_alert('HIGH_LEVERAGE', f"Leverage {metrics.leverage:.2f} exceeds limit {self.max_leverage}")

        # Check drawdown
        if metrics.max_drawdown > Decimal('0.1'):  # 10%
            await self._send_alert('HIGH_DRAWDOWN', f"Max drawdown {metrics.max_drawdown:.2%}")

        # Check daily loss
        if metrics.daily_pnl < -self.initial_capital * Decimal('0.02'):  # 2%
            await self._send_alert('LARGE_DAILY_LOSS', f"Daily loss {metrics.daily_pnl:.2f}")

    async def _send_alert(self, alert_type: str, message: str):
        """Send alert"""
        alert = {
            'type': alert_type,
            'message': message,
            'timestamp': datetime.now(),
            'fund_aum': float(self.get_total_aum()),
            'leverage': float(self._calculate_total_exposure() / self.get_total_aum())
        }

        # Various alert channels can be integrated here
        self.log.warning(f"ALERT [{alert_type}]: {message}")

    async def _save_report(self, report: dict, report_type: str):
        """Save report"""
        import json
        from pathlib import Path

        reports_dir = Path('reports')
        reports_dir.mkdir(exist_ok=True)

        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        filename = f"{report_type}_report_{timestamp}.json"
        filepath = reports_dir / filename

        with open(filepath, 'w') as f:
            json.dump(report, f, indent=2)
```

### 20.2.2 Portfolio Optimizer

```python
# src/portfolio/portfolio_optimizer.py
import cvxpy as cp
import numpy as np
from typing import Dict, List, Optional
from decimal import Decimal

class PortfolioOptimizer:
    """Portfolio Optimizer"""

    def __init__(self, config: dict):
        self.config = config
        self.risk_free_rate = config.get('risk_free_rate', 0.02)
        self.optimization_method = config.get('method', 'mean_variance')  # mean_variance, risk_parity, equal_weight

    def optimize(self, expected_returns: Dict[str, float],
                 covariance_matrix: Dict[str, Dict[str, float]],
                 constraints: dict) -> Dict[str, Decimal]:
        """Optimize portfolio"""
        if self.optimization_method == 'mean_variance':
            return self._mean_variance_optimization(expected_returns, covariance_matrix, constraints)
        elif self.optimization_method == 'risk_parity':
            return self._risk_parity_optimization(covariance_matrix, constraints)
        elif self.optimization_method == 'equal_weight':
            return self._equal_weight_optimization(constraints)
        else:
            raise ValueError(f"Unknown optimization method: {self.optimization_method}")

    def _mean_variance_optimization(self, expected_returns: Dict[str, float],
                                    covariance_matrix: Dict[str, Dict[str, float]],
                                    constraints: dict) -> Dict[str, Decimal]:
        """Mean-variance optimization"""
        assets = list(expected_returns.keys())
        n = len(assets)

        # Convert to numpy arrays
        mu = np.array([expected_returns[asset] for asset in assets])
        Sigma = np.array([[covariance_matrix[asset1][asset2] for asset2 in assets] for asset1 in assets])

        # Optimization variables
        w = cp.Variable(n)

        # Objective: maximize Sharpe ratio
        portfolio_return = mu.T @ w
        portfolio_variance = cp.quad_form(w, Sigma)
        objective = cp.Maximize((portfolio_return - self.risk_free_rate) / cp.sqrt(portfolio_variance))

        # Constraints
        constraints_list = []

        # Weights sum to 1
        if constraints.get('sum_to_one', True):
            constraints_list.append(cp.sum(w) == 1)

        # Non-negative weights (no short selling)
        if constraints.get('long_only', True):
            constraints_list.append(w >= 0)

        # Min/max weights
        if 'min_weights' in constraints:
            min_weights = [constraints['min_weights'].get(asset, 0) for asset in assets]
            constraints_list.append(w >= min_weights)

        if 'max_weights' in constraints:
            max_weights = [constraints['max_weights'].get(asset, 1) for asset in assets]
            constraints_list.append(w <= max_weights)

        # Risk budget constraints
        if 'risk_budgets' in constraints:
            marginal_contributions = Sigma @ w
            for i, asset in enumerate(assets):
                if asset in constraints['risk_budgets']:
                    constraints_list.append(
                        w[i] * marginal_contributions[i] <= constraints['risk_budgets'][asset]
                    )

        # Solve optimization problem
        problem = cp.Problem(objective, constraints_list)
        problem.solve()

        if problem.status not in ["infeasible", "unbounded"]:
            optimal_weights = w.value
            return {asset: Decimal(str(weight)) for asset, weight in zip(assets, optimal_weights)}
        else:
            # Return equal weights if optimization fails
            return self._equal_weight_optimization(constraints)

    def _risk_parity_optimization(self, covariance_matrix: Dict[str, Dict[str, float]],
                                  constraints: dict) -> Dict[str, Decimal]:
        """Risk parity optimization"""
        assets = list(covariance_matrix.keys())
        n = len(assets)

        # Convert to numpy arrays
        Sigma = np.array([[covariance_matrix[asset1][asset2] for asset2 in assets] for asset1 in assets])

        # Optimization variables
        w = cp.Variable(n)

        # Calculate portfolio variance and marginal risk contributions
        portfolio_variance = cp.quad_form(w, Sigma)
        marginal_contributions = Sigma @ w

        # Objective: minimize sum of squared differences in risk contributions
        risk_contributions = cp.multiply(w, marginal_contributions)
        avg_risk_contribution = cp.sum(risk_contributions) / n
        objective = cp.Minimize(cp.sum_squares(risk_contributions - avg_risk_contribution))

        # Constraints
        constraints_list = [
            cp.sum(w) == 1,
            w >= 0
        ]

        # Add min/max weight constraints
        if 'min_weights' in constraints:
            min_weights = [constraints['min_weights'].get(asset, 0.01) for asset in assets]
            constraints_list.append(w >= min_weights)

        if 'max_weights' in constraints:
            max_weights = [constraints['max_weights'].get(asset, 0.4) for asset in assets]
            constraints_list.append(w <= max_weights)

        # Solve
        problem = cp.Problem(objective, constraints_list)
        problem.solve()

        if problem.status not in ["infeasible", "unbounded"]:
            optimal_weights = w.value
            return {asset: Decimal(str(weight)) for asset, weight in zip(assets, optimal_weights)}
        else:
            return self._equal_weight_optimization(constraints)

    def _equal_weight_optimization(self, constraints: dict) -> Dict[str, Decimal]:
        """Equal weight optimization"""
        assets = list(constraints.get('min_weights', {}).keys())
        if not assets:
            # Return empty dict if no asset list specified
            return {}

        n = len(assets)
        equal_weight = Decimal('1') / Decimal(str(n))

        weights = {}
        for asset in assets:
            # Apply minimum weight constraints
            min_weight = Decimal(str(constraints['min_weights'].get(asset, 0)))
            max_weight = Decimal(str(constraints['max_weights'].get(asset, 1)))

            weight = max(equal_weight, min_weight)
            weight = min(weight, max_weight)
            weights[asset] = weight

        # Renormalize to ensure sum equals 1
        total_weight = sum(weights.values())
        if total_weight > 0:
            weights = {k: v / total_weight for k, v in weights.items()}

        return weights

    def black_litterman_optimization(self, views: Dict[str, float],
                                     view_uncertainties: Dict[str, float],
                                     tau: float = 0.025,
                                     risk_aversion: float = 3.0) -> Dict[str, Decimal]:
        """Black-Litterman optimization"""
        # This is a simplified Black-Litterman implementation
        # A more complete implementation is needed in practice

        # Need market implied returns and covariance matrix
        # Return mean-variance optimization result as placeholder here
        return {}

    def robust_optimization(self, expected_returns: Dict[str, float],
                            uncertainty_sets: Dict[str, float],
                            covariance_matrix: Dict[str, Dict[str, float]],
                            constraints: dict) -> Dict[str, Decimal]:
        """Robust optimization (worst-case optimization)"""
        # Consider uncertainty in expected returns
        adjusted_returns = {}
        for asset, exp_ret in expected_returns.items():
            uncertainty = uncertainty_sets.get(asset, 0.01)
            # Use worst-case returns
            adjusted_returns[asset] = exp_ret - uncertainty

        return self._mean_variance_optimization(adjusted_returns, covariance_matrix, constraints)
```

## 20.3 Advanced Risk Management

### 20.3.1 Multi-Level Risk Control

```python
# src/risk/fund_risk_manager.py
from typing import Dict, List, Optional, Tuple
from dataclasses import dataclass
from decimal import Decimal
import numpy as np
from datetime import datetime

@dataclass
class RiskLimit:
    """Risk limit"""
    name: str
    limit_type: str  # position, leverage, var, drawdown, concentration
    limit_value: Decimal
    current_value: Decimal
    utilization: Decimal  # current/limit
    breach_count: int = 0
    last_breach: Optional[datetime] = None

@dataclass
class StressTestResult:
    """Stress test result"""
    scenario_name: str
    portfolio_pnl: Decimal
    max_loss: Decimal
    var_estimate: Decimal
    worst_drawdown: Decimal
    breaching_positions: List[str]

class FundRiskManager:
    """Fund Risk Manager"""

    def __init__(self, config: dict):
        self.config = config

        # Risk limits
        self.limits = self._initialize_limits()

        # Stress test scenarios
        self.stress_scenarios = self._initialize_stress_scenarios()

        # Risk monitoring
        self.risk_monitoring_active = True
        self.last_risk_check = None

    def _initialize_limits(self) -> Dict[str, RiskLimit]:
        """Initialize risk limits"""
        limits_config = self.config.get('limits', {})
        limits = {}

        # Position limits
        limits['max_position'] = RiskLimit(
            name='max_position',
            limit_type='position',
            limit_value=Decimal(str(limits_config.get('max_position', '0.2'))),  # Max 20% per strategy
            current_value=Decimal('0'),
            utilization=Decimal('0')
        )

        # Leverage limits
        limits['max_leverage'] = RiskLimit(
            name='max_leverage',
            limit_type='leverage',
            limit_value=Decimal(str(limits_config.get('max_leverage', '3.0'))),
            current_value=Decimal('0'),
            utilization=Decimal('0')
        )

        # VaR limits
        limits['max_var'] = RiskLimit(
            name='max_var',
            limit_type='var',
            limit_value=Decimal(str(limits_config.get('max_var', '0.02'))),  # 2% VaR
            current_value=Decimal('0'),
            utilization=Decimal('0')
        )

        # Drawdown limits
        limits['max_drawdown'] = RiskLimit(
            name='max_drawdown',
            limit_type='drawdown',
            limit_value=Decimal(str(limits_config.get('max_drawdown', '0.15'))),  # 15%
            current_value=Decimal('0'),
            utilization=Decimal('0')
        )

        # Concentration limits
        limits['max_concentration'] = RiskLimit(
            name='max_concentration',
            limit_type='concentration',
            limit_value=Decimal(str(limits_config.get('max_concentration', '0.4'))),  # 40%
            current_value=Decimal('0'),
            utilization=Decimal('0')
        )

        return limits

    def _initialize_stress_scenarios(self) -> List[dict]:
        """Initialize stress test scenarios"""
        scenarios_config = self.config.get('stress_scenarios', [])
        scenarios = []

        # Default scenarios
        default_scenarios = [
            {
                'name': 'market_crash',
                'shocks': {
                    'equity': -0.30,  # 30% equity market drop
                    'volatility': 2.0,  # Volatility doubles
                    'correlation': 0.8   # Correlation rises to 0.8
                }
            },
            {
                'name': 'volatility_spike',
                'shocks': {
                    'volatility': 3.0,  # Volatility triples
                    'correlation': 0.9   # Correlation rises to 0.9
                }
            },
            {
                'name': 'liquidity_crisis',
                'shocks': {
                    'bid_ask_spread': 5.0,  # Bid-ask spread widens 5x
                    'market_impact': 2.0    # Market impact doubles
                }
            }
        ]

        # Add configured scenarios
        scenarios.extend(default_scenarios)
        scenarios.extend(scenarios_config)

        return scenarios

    async def monitor_risk(self, fund_manager):
        """Monitor risk"""
        if not self.risk_monitoring_active:
            return

        try:
            # Check all risk limits
            await self._check_all_limits(fund_manager)

            # Run stress tests
            await self._run_stress_tests(fund_manager)

            # Update risk report
            await self._update_risk_report(fund_manager)

            self.last_risk_check = datetime.now()

        except Exception as e:
            fund_manager.log.error(f"Risk monitoring error: {e}")

    async def _check_all_limits(self, fund_manager):
        """Check all risk limits"""
        # Check position limits
        await self._check_position_limits(fund_manager)

        # Check leverage limits
        await self._check_leverage_limits(fund_manager)

        # Check VaR limits
        await self._check_var_limits(fund_manager)

        # Check drawdown limits
        await self._check_drawdown_limits(fund_manager)

        # Check concentration limits
        await self._check_concentration_limits(fund_manager)

    async def _check_position_limits(self, fund_manager):
        """Check position limits"""
        total_aum = fund_manager.get_total_aum()
        strategy_performance = fund_manager.get_strategy_performance()

        for strategy_id, perf in strategy_performance.items():
            strategy_value = Decimal(str(perf['allocated_capital'])) + Decimal(str(perf['total_pnl']))
            position_ratio = strategy_value / total_aum if total_aum > 0 else Decimal('0')

            limit = self.limits['max_position']
            limit.current_value = position_ratio
            limit.utilization = position_ratio / limit.limit_value if limit.limit_value > 0 else Decimal('0')

            if position_ratio > limit.limit_value:
                limit.breach_count += 1
                limit.last_breach = datetime.now()

                await self._handle_limit_breach(limit, {
                    'strategy_id': strategy_id,
                    'position_ratio': float(position_ratio),
                    'limit': float(limit.limit_value)
                })

    async def _check_leverage_limits(self, fund_manager):
        """Check leverage limits"""
        total_exposure = self._calculate_total_exposure(fund_manager)
        total_aum = fund_manager.get_total_aum()
        leverage = total_exposure / total_aum if total_aum > 0 else Decimal('0')

        limit = self.limits['max_leverage']
        limit.current_value = leverage
        limit.utilization = leverage / limit.limit_value if limit.limit_value > 0 else Decimal('0')

        if leverage > limit.limit_value:
            limit.breach_count += 1
            limit.last_breach = datetime.now()

            await self._handle_limit_breach(limit, {
                'current_leverage': float(leverage),
                'limit': float(limit.limit_value)
            })

    async def _check_var_limits(self, fund_manager):
        """Check VaR limits"""
        # Calculate portfolio VaR
        returns = self._calculate_portfolio_returns(fund_manager)
        var_95 = self._calculate_var(returns, 0.05)

        limit = self.limits['max_var']
        limit.current_value = var_95
        limit.utilization = var_95 / limit.limit_value if limit.limit_value > 0 else Decimal('0')

        if var_95 > limit.limit_value:
            limit.breach_count += 1
            limit.last_breach = datetime.now()

            await self._handle_limit_breach(limit, {
                'var_95': float(var_95),
                'limit': float(limit.limit_value)
            })

    async def _check_drawdown_limits(self, fund_manager):
        """Check drawdown limits"""
        max_drawdown = self._calculate_max_drawdown(fund_manager)

        limit = self.limits['max_drawdown']
        limit.current_value = max_drawdown
        limit.utilization = max_drawdown / limit.limit_value if limit.limit_value > 0 else Decimal('0')

        if max_drawdown > limit.limit_value:
            limit.breach_count += 1
            limit.last_breach = datetime.now()

            await self._handle_limit_breach(limit, {
                'max_drawdown': float(max_drawdown),
                'limit': float(limit.limit_value)
            })

    async def _check_concentration_limits(self, fund_manager):
        """Check concentration limits"""
        # Calculate asset concentration (Herfindahl index)
        strategy_performance = fund_manager.get_strategy_performance()
        weights = [Decimal(str(p['current_weight'])) for p in strategy_performance.values()]
        herfindahl = sum(w**2 for w in weights)

        limit = self.limits['max_concentration']
        limit.current_value = herfindahl
        limit.utilization = herfindahl / limit.limit_value if limit.limit_value > 0 else Decimal('0')

        if herfindahl > limit.limit_value:
            limit.breach_count += 1
            limit.last_breach = datetime.now()

            await self._handle_limit_breach(limit, {
                'herfindahl_index': float(herfindahl),
                'limit': float(limit.limit_value)
            })

    async def _handle_limit_breach(self, limit: RiskLimit, details: dict):
        """Handle risk limit breach"""
        breach_level = self._determine_breach_level(limit)

        if breach_level == 'critical':
            # Critical breach: take immediate action
            await self._handle_critical_breach(limit, details)
        elif breach_level == 'warning':
            # Warning level: log and monitor
            await self._handle_warning_breach(limit, details)

    def _determine_breach_level(self, limit: RiskLimit) -> str:
        """Determine breach level"""
        if limit.utilization > Decimal('1.5'):  # Over 50%
            return 'critical'
        elif limit.utilization > Decimal('1.0'):  # Over limit
            return 'warning'
        elif limit.utilization > Decimal('0.8'):  # Approaching limit
            return 'info'
        else:
            return 'normal'

    async def _handle_critical_breach(self, limit: RiskLimit, details: dict):
        """Handle critical breach"""
        # Trigger emergency measures
        if limit.limit_type == 'leverage':
            await self._emergency_deleverage()
        elif limit.limit_type == 'var':
            await self._reduce_risk_exposure()
        elif limit.limit_type == 'drawdown':
            await self._emergency_stop_loss()

    async def _handle_warning_breach(self, limit: RiskLimit, details: dict):
        """Handle warning level breach"""
        # Log and potentially notify
        pass

    async def _run_stress_tests(self, fund_manager):
        """Run stress tests"""
        stress_results = {}

        for scenario in self.stress_scenarios:
            result = await self._run_single_stress_test(fund_manager, scenario)
            stress_results[scenario['name']] = result

        # Check stress test results
        for scenario_name, result in stress_results.items():
            if result.max_loss < -self.limits['max_var'].limit_value:
                await self._handle_stress_test_failure(scenario_name, result)

        return stress_results

    async def _run_single_stress_test(self, fund_manager, scenario: dict) -> StressTestResult:
        """Run single stress test"""
        # Get current positions
        positions = self._get_all_positions(fund_manager)

        # Apply shocks
        shocked_returns = {}
        for asset in positions.keys():
            # Apply different shocks based on asset type
            if 'equity' in asset.lower():
                shocked_returns[asset] = scenario['shocks'].get('equity', 0)
            elif 'bond' in asset.lower():
                shocked_returns[asset] = scenario['shocks'].get('bond', 0)
            else:
                shocked_returns[asset] = 0

        # Calculate portfolio PnL
        portfolio_pnl = Decimal('0')
        for asset, position in positions.items():
            pnl = position * Decimal(str(shocked_returns.get(asset, 0)))
            portfolio_pnl += pnl

        # Calculate maximum loss
        max_loss = min(portfolio_pnl, Decimal('0'))

        # Estimate stressed VaR
        var_estimate = abs(max_loss)

        # Calculate worst drawdown
        worst_drawdown = abs(max_loss) / fund_manager.get_total_aum()

        # Find breaching positions
        breaching_positions = []
        for asset, position in positions.items():
            if abs(position * Decimal(str(shocked_returns.get(asset, 0)))) > fund_manager.get_total_aum() * Decimal('0.01'):
                breaching_positions.append(asset)

        return StressTestResult(
            scenario_name=scenario['name'],
            portfolio_pnl=portfolio_pnl,
            max_loss=max_loss,
            var_estimate=var_estimate,
            worst_drawdown=worst_drawdown,
            breaching_positions=breaching_positions
        )

    async def _handle_stress_test_failure(self, scenario_name: str, result: StressTestResult):
        """Handle stress test failure"""
        # Log failure
        # May need to adjust risk parameters
        pass

    def calculate_risk_metrics(self, fund_manager) -> dict:
        """Calculate risk metrics"""
        returns = self._calculate_portfolio_returns(fund_manager)

        return {
            'volatility': self._calculate_volatility(returns),
            'var_95': self._calculate_var(returns, 0.05),
            'var_99': self._calculate_var(returns, 0.01),
            'cvar_95': self._calculate_cvar(returns, 0.05),
            'max_drawdown': self._calculate_max_drawdown(fund_manager),
            'sharpe_ratio': self._calculate_sharpe_ratio(returns),
            'sortino_ratio': self._calculate_sortino_ratio(returns),
            'calmar_ratio': self._calculate_calmar_ratio(fund_manager),
            'information_ratio': self._calculate_information_ratio(fund_manager),
            'beta': self._calculate_beta(fund_manager),
            'tracking_error': self._calculate_tracking_error(fund_manager)
        }

    def _calculate_volatility(self, returns: List[float]) -> float:
        """Calculate volatility"""
        if len(returns) < 2:
            return 0.0
        return np.std(returns) * np.sqrt(252)  # Annualized volatility

    def _calculate_var(self, returns: List[float], alpha: float) -> Decimal:
        """Calculate VaR"""
        if len(returns) < 10:
            return Decimal('0')
        var = np.percentile(returns, alpha * 100)
        return Decimal(str(var))

    def _calculate_cvar(self, returns: List[float], alpha: float) -> Decimal:
        """Calculate Conditional VaR"""
        if len(returns) < 10:
            return Decimal('0')
        var = np.percentile(returns, alpha * 100)
        cvar = np.mean([r for r in returns if r <= var])
        return Decimal(str(cvar))

    def _calculate_max_drawdown(self, fund_manager) -> Decimal:
        """Calculate maximum drawdown"""
        # Need historical NAV data
        # Return placeholder here
        return Decimal('0.05')  # Example: 5%

    def _calculate_sharpe_ratio(self, returns: List[float]) -> float:
        """Calculate Sharpe ratio"""
        if len(returns) < 2:
            return 0.0
        excess_return = np.mean(returns) - self.config.get('risk_free_rate', 0.02)
        volatility = self._calculate_volatility(returns)
        return excess_return / volatility if volatility > 0 else 0.0

    def _calculate_sortino_ratio(self, returns: List[float]) -> float:
        """Calculate Sortino ratio"""
        if len(returns) < 2:
            return 0.0
        downside_returns = [r for r in returns if r < 0]
        if not downside_returns:
            return float('inf')
        excess_return = np.mean(returns) - self.config.get('risk_free_rate', 0.02)
        downside_volatility = np.std(downside_returns)
        return excess_return / downside_volatility if downside_volatility > 0 else 0.0

    # Other helper methods...
    def _get_all_positions(self, fund_manager) -> Dict[str, Decimal]:
        """Get all positions"""
        positions = {}
        for strategy_id, strategy in fund_manager.strategies.items():
            strategy_positions = strategy.get_positions()
            for asset, position in strategy_positions.items():
                positions[asset] = positions.get(asset, Decimal('0')) + Decimal(str(position))
        return positions

    def _calculate_total_exposure(self, fund_manager) -> Decimal:
        """Calculate total exposure"""
        total_exposure = Decimal('0')
        for strategy in fund_manager.strategies.values():
            total_exposure += strategy.get_exposure()
        return total_exposure

    def _calculate_portfolio_returns(self, fund_manager) -> List[float]:
        """Calculate portfolio returns"""
        # Need historical data
        # Return sample data here
        return [0.001, -0.002, 0.003, -0.001, 0.002]

    async def validate_rebalance(self, trades: List[dict]) -> bool:
        """Validate rebalancing trades"""
        # Check if trades comply with risk limits
        for trade in trades:
            if not await self._validate_trade(trade):
                return False
        return True

    async def _validate_trade(self, trade: dict) -> bool:
        """Validate single trade"""
        # Check trade size, concentration, etc.
        return True

    # Emergency measure methods
    async def _emergency_deleverage(self):
        """Emergency deleveraging"""
        self.log.critical("Emergency deleveraging triggered!")
        # Implement emergency deleveraging logic

    async def _reduce_risk_exposure(self):
        """Reduce risk exposure"""
        self.log.warning("Reducing risk exposure...")
        # Implement risk exposure reduction logic

    async def _emergency_stop_loss(self):
        """Emergency stop loss"""
        self.log.critical("Emergency stop loss triggered!")
        # Implement emergency stop loss logic
```

## 20.4 Performance Attribution System

### 20.4.1 Return Attribution Analysis

```python
# src/attribution/attribution_analyzer.py
from typing import Dict, List, Tuple
from dataclasses import dataclass
from decimal import Decimal
import numpy as np
from datetime import datetime, timedelta

@dataclass
class AttributionResult:
    """Attribution result"""
    total_return: Decimal
    benchmark_return: Decimal
    active_return: Decimal  # Excess return
    allocation_effect: Decimal  # Allocation effect
    selection_effect: Decimal  # Selection effect
    interaction_effect: Decimal  # Interaction effect
    trading_cost: Decimal  # Trading cost
    currency_effect: Decimal  # Currency effect

@dataclass
class FactorAttribution:
    """Factor attribution"""
    factor_name: str
    factor_return: Decimal
    factor_exposure: Decimal
    factor_contribution: Decimal

class AttributionAnalyzer:
    """Performance Attribution Analyzer"""

    def __init__(self, config: dict):
        self.config = config
        self.benchmark = config.get('benchmark', 'SPY')
        self.factors = config.get('factors', [
            'momentum',
            'value',
            'size',
            'quality',
            'volatility',
            'liquidity'
        ])

    def calculate_attribution(self, portfolio_returns: Dict[str, float],
                             benchmark_returns: Dict[str, float],
                             portfolio_weights: Dict[str, float],
                             benchmark_weights: Dict[str, float],
                             period: str = 'monthly') -> AttributionResult:
        """Calculate performance attribution"""
        # Calculate total returns
        total_return = self._calculate_portfolio_return(portfolio_returns, portfolio_weights)
        benchmark_return = self._calculate_portfolio_return(benchmark_returns, benchmark_weights)
        active_return = total_return - benchmark_return

        # Calculate effect decomposition (Brinson model)
        allocation_effect = self._calculate_allocation_effect(
            portfolio_returns, benchmark_returns,
            portfolio_weights, benchmark_weights
        )

        selection_effect = self._calculate_selection_effect(
            portfolio_returns, benchmark_returns,
            portfolio_weights, benchmark_weights
        )

        interaction_effect = self._calculate_interaction_effect(
            portfolio_returns, benchmark_returns,
            portfolio_weights, benchmark_weights
        )

        # Other effects
        trading_cost = self._estimate_trading_cost()
        currency_effect = self._estimate_currency_effect()

        return AttributionResult(
            total_return=total_return,
            benchmark_return=benchmark_return,
            active_return=active_return,
            allocation_effect=allocation_effect,
            selection_effect=selection_effect,
            interaction_effect=interaction_effect,
            trading_cost=trading_cost,
            currency_effect=currency_effect
        )

    def _calculate_portfolio_return(self, returns: Dict[str, float], weights: Dict[str, float]) -> Decimal:
        """Calculate portfolio return"""
        portfolio_return = 0.0
        total_weight = sum(weights.values())

        for asset, weight in weights.items():
            if asset in returns:
                portfolio_return += weight * returns[asset] / total_weight

        return Decimal(str(portfolio_return))

    def _calculate_allocation_effect(self, portfolio_returns: Dict[str, float],
                                   benchmark_returns: Dict[str, float],
                                   portfolio_weights: Dict[str, float],
                                   benchmark_weights: Dict[str, float]) -> Decimal:
        """Calculate allocation effect"""
        allocation_effect = 0.0

        for asset in portfolio_weights.keys():
            if asset in benchmark_weights:
                # Excess weight × benchmark return
                weight_diff = portfolio_weights[asset] - benchmark_weights[asset]
                benchmark_return = benchmark_returns.get(asset, 0)
                allocation_effect += weight_diff * benchmark_return

        return Decimal(str(allocation_effect))

    def _calculate_selection_effect(self, portfolio_returns: Dict[str, float],
                                   benchmark_returns: Dict[str, float],
                                   portfolio_weights: Dict[str, float],
                                   benchmark_weights: Dict[str, float]) -> Decimal:
        """Calculate selection effect"""
        selection_effect = 0.0

        for asset in portfolio_weights.keys():
            if asset in benchmark_weights:
                # Benchmark weight × excess return
                weight = benchmark_weights[asset]
                excess_return = portfolio_returns.get(asset, 0) - benchmark_returns.get(asset, 0)
                selection_effect += weight * excess_return

        return Decimal(str(selection_effect))

    def _calculate_interaction_effect(self, portfolio_returns: Dict[str, float],
                                     benchmark_returns: Dict[str, float],
                                     portfolio_weights: Dict[str, float],
                                     benchmark_weights: Dict[str, float]) -> Decimal:
        """Calculate interaction effect"""
        interaction_effect = 0.0

        for asset in portfolio_weights.keys():
            if asset in benchmark_weights:
                # Excess weight × excess return
                weight_diff = portfolio_weights[asset] - benchmark_weights[asset]
                excess_return = portfolio_returns.get(asset, 0) - benchmark_returns.get(asset, 0)
                interaction_effect += weight_diff * excess_return

        return Decimal(str(interaction_effect))

    def calculate_factor_attribution(self, portfolio: Dict,
                                    factor_returns: Dict[str, float],
                                    factor_loadings: Dict[str, Dict[str, float]]) -> List[FactorAttribution]:
        """Calculate factor attribution"""
        attributions = []

        for factor in self.factors:
            if factor in factor_returns:
                factor_return = factor_returns[factor]

                # Calculate portfolio's average exposure to this factor
                portfolio_exposure = self._calculate_average_factor_exposure(
                    portfolio, factor_loadings.get(factor, {})
                )

                # Calculate factor contribution
                factor_contribution = factor_return * portfolio_exposure

                attributions.append(FactorAttribution(
                    factor_name=factor,
                    factor_return=Decimal(str(factor_return)),
                    factor_exposure=Decimal(str(portfolio_exposure)),
                    factor_contribution=Decimal(str(factor_contribution))
                ))

        return attributions

    def _calculate_average_factor_exposure(self, portfolio: Dict,
                                           factor_loadings: Dict[str, float]) -> float:
        """Calculate average factor exposure"""
        total_exposure = 0.0
        total_weight = 0.0

        for asset, weight in portfolio.items():
            exposure = factor_loadings.get(asset, 0)
            total_exposure += weight * exposure
            total_weight += weight

        return total_exposure / total_weight if total_weight > 0 else 0.0

    def generate_attribution_report(self, attribution: AttributionResult,
                                  factor_attributions: List[FactorAttribution] = None) -> dict:
        """Generate attribution report"""
        report = {
            'period': datetime.now().strftime('%Y-%m'),
            'total_return': float(attribution.total_return),
            'benchmark_return': float(attribution.benchmark_return),
            'active_return': float(attribution.active_return),
            'attribution_breakdown': {
                'allocation_effect': float(attribution.allocation_effect),
                'selection_effect': float(attribution.selection_effect),
                'interaction_effect': float(attribution.interaction_effect),
                'trading_cost': float(attribution.trading_cost),
                'currency_effect': float(attribution.currency_effect)
            },
            'attribution_percentages': {}
        }

        # Calculate contribution percentages
        if attribution.active_return != 0:
            report['attribution_percentages'] = {
                'allocation': float(attribution.allocation_effect / attribution.active_return * 100),
                'selection': float(attribution.selection_effect / attribution.active_return * 100),
                'interaction': float(attribution.interaction_effect / attribution.active_return * 100),
                'costs': float(attribution.trading_cost / attribution.active_return * 100)
            }

        # Add factor attribution
        if factor_attributions:
            report['factor_attribution'] = {
                factor.name: {
                    'return': float(factor.factor_return),
                    'exposure': float(factor.factor_exposure),
                    'contribution': float(factor.factor_contribution)
                }
                for factor in factor_attributions
            }

        return report

    def rolling_attribution_analysis(self, returns_history: Dict[str, List[Tuple[datetime, float]]],
                                    weights_history: Dict[str, List[Tuple[datetime, float]]],
                                    window: int = 12) -> Dict[str, List[dict]]:
        """Rolling attribution analysis"""
        rolling_results = {}

        for asset in returns_history.keys():
            asset_returns = returns_history[asset]
            asset_weights = weights_history.get(asset, [])

            if len(asset_returns) < window:
                continue

            rolling_attributions = []
            for i in range(len(asset_returns) - window + 1):
                window_returns = {asset: r for (_, r) in asset_returns[i:i+window]}
                window_weights = {asset: w for (_, w) in asset_weights[i:i+window]}

                # Calculate window attribution
                # Simplified implementation needed here
                attribution = AttributionResult(
                    total_return=Decimal('0'),
                    benchmark_return=Decimal('0'),
                    active_return=Decimal('0'),
                    allocation_effect=Decimal('0'),
                    selection_effect=Decimal('0'),
                    interaction_effect=Decimal('0'),
                    trading_cost=Decimal('0'),
                    currency_effect=Decimal('0')
                )

                rolling_attributions.append({
                    'end_date': asset_returns[i + window - 1][0],
                    'attribution': self.generate_attribution_report(attribution)
                })

            rolling_results[asset] = rolling_attributions

        return rolling_results

    def _estimate_trading_cost(self) -> Decimal:
        """Estimate trading cost"""
        # Estimate based on trading volume and market conditions
        return Decimal('0.001')  # Example: 10 basis points

    def _estimate_currency_effect(self) -> Decimal:
        """Estimate currency effect"""
        # For multi-currency portfolios
        return Decimal('0')  # Example value

    def analyze_style_attribution(self, portfolio_positions: Dict,
                                 style_factors: Dict[str, Dict[str, float]]) -> dict:
        """Analyze style attribution"""
        style_analysis = {}

        # Calculate portfolio's exposure to various style factors
        portfolio_exposures = {}
        for style, loadings in style_factors.items():
            exposure = self._calculate_style_exposure(portfolio_positions, loadings)
            portfolio_exposures[style] = exposure

        # Calculate style return contributions
        style_returns = {
            'value': 0.015,      # Value factor return
            'growth': 0.010,     # Growth factor return
            'momentum': 0.020,   # Momentum factor return
            'quality': 0.008,    # Quality factor return
            'size': 0.005        # Size factor return
        }

        style_contributions = {}
        for style, exposure in portfolio_exposures.items():
            if style in style_returns:
                contribution = exposure * style_returns[style]
                style_contributions[style] = contribution

        style_analysis = {
            'exposures': {k: float(v) for k, v in portfolio_exposures.items()},
            'returns': style_returns,
            'contributions': {k: float(v) for k, v in style_contributions.items()},
            'total_contribution': float(sum(style_contributions.values()))
        }

        return style_analysis

    def _calculate_style_exposure(self, positions: Dict, loadings: Dict[str, float]) -> float:
        """Calculate style exposure"""
        total_exposure = 0.0
        total_weight = 0.0

        for asset, weight in positions.items():
            loading = loadings.get(asset, 0)
            total_exposure += weight * loading
            total_weight += abs(weight)

        return total_exposure / total_weight if total_weight > 0 else 0.0
```

## 20.5 Main Program Implementation

### 20.5.1 Fund Main Program

```python
# src/main_fund.py
import asyncio
import signal
import sys
import logging
from pathlib import Path
from datetime import datetime

from src.config.config_manager import ConfigManager
from src.fund.fund_manager import FundManager
from src.risk.fund_risk_manager import FundRiskManager
from src.attribution.attribution_analyzer import AttributionAnalyzer
from src.reporting.fund_reporter import FundReporter

class QuantitativeFund:
    """Quantitative Fund Main Program"""

    def __init__(self, config_path: str):
        # Setup logging
        self._setup_logging()
        self.log = logging.getLogger(__name__)

        # Load configuration
        self.config_manager = ConfigManager(config_path)

        # Core components
        self.fund_manager = None
        self.risk_manager = None
        self.attribution_analyzer = None
        self.reporter = None

        # Running status
        self.running = False

    def _setup_logging(self):
        """Setup logging"""
        logging.basicConfig(
            level=logging.INFO,
            format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
            handlers=[
                logging.FileHandler('logs/fund.log'),
                logging.StreamHandler()
            ]
        )

    async def initialize(self):
        """Initialize the fund"""
        self.log.info("Initializing quantitative fund...")

        try:
            # Initialize fund manager
            fund_config = self.config_manager.get('fund')
            self.fund_manager = FundManager(fund_config)
            await self.fund_manager.initialize()

            # Initialize risk manager
            risk_config = self.config_manager.get('risk')
            self.risk_manager = FundRiskManager(risk_config)

            # Initialize attribution analyzer
            attribution_config = self.config_manager.get('attribution', {})
            self.attribution_analyzer = AttributionAnalyzer(attribution_config)

            # Initialize report generator
            reporting_config = self.config_manager.get('reporting', {})
            self.reporter = FundReporter(reporting_config)

            self.log.info("Quantitative fund initialized successfully")

        except Exception as e:
            self.log.error(f"Failed to initialize fund: {e}")
            raise

    async def start(self):
        """Start the fund"""
        self.log.info("Starting quantitative fund...")

        try:
            self.running = True

            # Setup signal handlers
            self._setup_signal_handlers()

            # Start trading
            await self.fund_manager.start_trading()

            # Start periodic tasks
            await self._start_periodic_tasks()

            # Main loop
            await self._main_loop()

        except Exception as e:
            self.log.error(f"Error in main loop: {e}")
            raise

    async def stop(self):
        """Stop the fund"""
        self.log.info("Stopping quantitative fund...")
        self.running = False

        try:
            # Generate final report
            await self._generate_final_report()

            # Stop trading
            await self.fund_manager.stop_trading()

            self.log.info("Quantitative fund stopped successfully")

        except Exception as e:
            self.log.error(f"Error during shutdown: {e}")

    def _setup_signal_handlers(self):
        """Setup signal handlers"""
        def signal_handler(signum, frame):
            self.log.info(f"Received signal {signum}, shutting down...")
            asyncio.create_task(self.stop())

        signal.signal(signal.SIGINT, signal_handler)
        signal.signal(signal.SIGTERM, signal_handler)

    async def _start_periodic_tasks(self):
        """Start periodic tasks"""
        # Daily risk report
        asyncio.create_task(self._daily_report_loop())

        # Weekly performance analysis
        asyncio.create_task(self._weekly_analysis_loop())

        # Monthly attribution analysis
        asyncio.create_task(self._monthly_attribution_loop())

    async def _daily_report_loop(self):
        """Daily report loop"""
        while self.running:
            try:
                # Wait until 16:00 (after market close)
                now = datetime.now()
                next_report = now.replace(hour=16, minute=0, second=0, microsecond=0)
                if next_report <= now:
                    next_report += timedelta(days=1)

                sleep_seconds = (next_report - now).total_seconds()
                await asyncio.sleep(sleep_seconds)

                # Generate daily report
                await self._generate_daily_report()

            except Exception as e:
                self.log.error(f"Daily report error: {e}")
                await asyncio.sleep(3600)  # Retry after 1 hour

    async def _weekly_analysis_loop(self):
        """Weekly analysis loop"""
        while self.running:
            try:
                # Wait until Friday 16:00
                now = datetime.now()
                days_until_friday = (4 - now.weekday()) % 7
                if days_until_friday == 0 and now.hour >= 16:
                    days_until_friday = 7

                next_friday = now + timedelta(days=days_until_friday)
                next_friday = next_friday.replace(hour=16, minute=0, second=0, microsecond=0)

                sleep_seconds = (next_friday - now).total_seconds()
                await asyncio.sleep(sleep_seconds)

                # Generate weekly report
                await self._generate_weekly_report()

            except Exception as e:
                self.log.error(f"Weekly analysis error: {e}")
                await asyncio.sleep(86400)  # Retry after 1 day

    async def _monthly_attribution_loop(self):
        """Monthly attribution loop"""
        while self.running:
            try:
                # Wait until end of month
                now = datetime.now()
                next_month = (now.replace(day=1) + timedelta(days=32)).replace(day=1)
                sleep_seconds = (next_month - now).total_seconds()

                await asyncio.sleep(sleep_seconds)

                # Generate monthly attribution report
                await self._generate_monthly_attribution()

            except Exception as e:
                self.log.error(f"Monthly attribution error: {e}")
                await asyncio.sleep(86400 * 7)  # Retry after 1 week

    async def _main_loop(self):
        """Main loop"""
        while self.running:
            try:
                # Execute every minute
                await self._perform_regular_checks()
                await asyncio.sleep(60)

            except Exception as e:
                self.log.error(f"Main loop error: {e}")
                await asyncio.sleep(60)

    async def _perform_regular_checks(self):
        """Perform regular checks"""
        # Risk monitoring
        await self.risk_manager.monitor_risk(self.fund_manager)

        # Check if temporary report needs to be generated
        now = datetime.now()
        if now.minute == 0:  # On the hour
            await self._generate_hourly_snapshot()

    async def _generate_hourly_snapshot(self):
        """Generate hourly snapshot"""
        fund_metrics = self.fund_manager.get_fund_metrics()
        strategy_performance = self.fund_manager.get_strategy_performance()
        risk_metrics = self.risk_manager.calculate_risk_metrics(self.fund_manager)

        snapshot = {
            'timestamp': now.isoformat(),
            'fund_metrics': {
                'aum': float(fund_metrics.total_aum),
                'pnl': float(fund_metrics.total_pnl),
                'leverage': fund_metrics.leverage
            },
            'strategy_count': len(strategy_performance),
            'risk_metrics': {
                'var': float(risk_metrics.get('var_95', 0)),
                'sharpe': risk_metrics.get('sharpe_ratio', 0)
            }
        }

        # Save snapshot
        import json
        with open(f'snapshots/snapshot_{now.strftime("%Y%m%d_%H%M")}.json', 'w') as f:
            json.dump(snapshot, f, indent=2)

    async def _generate_daily_report(self):
        """Generate daily report"""
        self.log.info("Generating daily report...")
        await self.reporter.generate_daily_report(self.fund_manager, self.risk_manager)

    async def _generate_weekly_report(self):
        """Generate weekly report"""
        self.log.info("Generating weekly report...")
        await self.reporter.generate_weekly_report(self.fund_manager, self.risk_manager, self.attribution_analyzer)

    async def _generate_monthly_attribution(self):
        """Generate monthly attribution report"""
        self.log.info("Generating monthly attribution report...")
        await self.reporter.generate_monthly_attribution(self.fund_manager, self.attribution_analyzer)

    async def _generate_final_report(self):
        """Generate final report"""
        self.log.info("Generating final report...")
        await self.reporter.generate_final_report(self.fund_manager, self.risk_manager, self.attribution_analyzer)

async def main():
    """Main function"""
    if len(sys.argv) < 2:
        print("Usage: python main_fund.py <config_path>")
        sys.exit(1)

    config_path = sys.argv[1]

    # Create fund instance
    fund = QuantitativeFund(config_path)

    try:
        # Initialize
        await fund.initialize()

        # Start
        await fund.start()

    except KeyboardInterrupt:
        print("\nReceived keyboard interrupt, shutting down...")
    except Exception as e:
        print(f"Fatal error: {e}")
        return 1
    finally:
        # Cleanup
        await fund.stop()

    return 0

if __name__ == "__main__":
    # Run main function
    exit_code = asyncio.run(main())
    sys.exit(exit_code)
```

## Summary

This chapter demonstrates through a multi-strategy quantitative fund system project how to:

1. **Build Institutional-Grade Systems**
   - Multi-strategy capital allocation
   - Intelligent portfolio optimization
   - Multi-level risk control

2. **Implement Advanced Features**
   - Performance attribution analysis
   - Stress testing
   - Risk budget management

3. **Meet Regulatory Requirements**
   - Complete reporting system
   - Trade record preservation
   - Risk disclosure

4. **Apply Best Practices**
   - Modular design
   - Asynchronous programming
   - Fault tolerance

This project showcases a complete framework for building professional quantitative investment funds, providing reliable solutions for institutional investors.

## References

- [Inside the Black Box: The Simple Truth About Quantitative Trading](https://www.wiley.com/en-us/Inside+the+Black+Box:+The+Simple+Truth+About+Quantitative+Trading)
- [Quantitative Equity Portfolio Management](https://www.mhprofessional.com/quantitative-equity-portfolio-management-9780071833393/)
- [Active Portfolio Management](https://www.mhprofessional.com/active-portfolio-management-9780071798920/)

## Runnable Code Entry (Source Build)

- Entry script: docs/tutorials_cn/code/chapter20/run_quant_fund.py
- Key examples: docs/tutorials_cn/code/chapter20/strategy_runner.py, docs/tutorials_cn/code/chapter20/quant_fund_portfolio.py, docs/tutorials_cn/code/chapter20/risk_allocator.py

Dependency strategy: Uses returns matrix as a shared contract for strategy run, allocation, and portfolio construction.

Verified command:

```bash
uv run --python 3.13 python docs/tutorials_cn/code/chapter20/run_quant_fund.py
```
