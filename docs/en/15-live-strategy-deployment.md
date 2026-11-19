# Chapter 15: Live Strategy Deployment

## Learning Objectives

- Master the complete workflow from backtesting to live trading
- Learn to build a secure live trading environment
- Understand strategy lifecycle management
- Implement comprehensive monitoring and risk control

## 15.1 Strategy Deployment Process

### 15.1.1 Deployment Preparation

```python
from dataclasses import dataclass
from typing import Dict, List, Optional
from enum import Enum
import yaml
import json
from datetime import datetime

class DeploymentStage(Enum):
    """Deployment stage enumeration"""
    PREPARATION = "preparation"       # Preparation stage
    VALIDATION = "validation"         # Validation stage
    STAGING = "staging"              # Staging environment
    PRODUCTION = "production"         # Production environment
    MONITORING = "monitoring"         # Monitoring stage

@dataclass
class StrategyConfig:
    """Strategy configuration"""
    name: str
    version: str
    description: str
    instruments: List[str]
    parameters: Dict
    risk_limits: Dict
    schedule: Optional[Dict] = None
    dependencies: List[str] = None

@dataclass
class DeploymentConfig:
    """Deployment configuration"""
    environment: str  # staging, production
    exchange: str
    account: str
    initial_capital: float
    max_position_size: float
    max_daily_loss: float
    max_drawdown: float
    restart_on_failure: bool = True
    auto_rebalance: bool = False

class DeploymentManager:
    """Deployment manager"""

    def __init__(self, config_path: str = "deploy_config.yaml"):
        self.config_path = config_path
        self.config = self._load_config()
        self.deployment_history = []
        self.active_deployments = {}

    def _load_config(self) -> dict:
        """Load deployment configuration"""
        with open(self.config_path, 'r') as f:
            return yaml.safe_load(f)

    async def deploy_strategy(self, strategy_config: StrategyConfig,
                             deployment_config: DeploymentConfig) -> dict:
        """Deploy strategy"""
        deployment_id = self._generate_deployment_id()

        logging.info(f"Starting deployment {deployment_id} for strategy {strategy_config.name}")

        try:
            # Stage 1: Preparation
            await self._stage_preparation(deployment_id, strategy_config, deployment_config)

            # Stage 2: Validation
            await self._stage_validation(deployment_id, strategy_config)

            # Stage 3: Staging test
            await self._stage_staging(deployment_id, strategy_config)

            # Stage 4: Production deployment
            if deployment_config.environment == "production":
                await self._stage_production(deployment_id, strategy_config, deployment_config)

            # Record deployment information
            deployment_info = {
                'id': deployment_id,
                'strategy': strategy_config.name,
                'version': strategy_config.version,
                'environment': deployment_config.environment,
                'status': 'deployed',
                'timestamp': datetime.utcnow().isoformat(),
                'config': deployment_config.__dict__
            }
            self.deployment_history.append(deployment_info)
            self.active_deployments[deployment_id] = deployment_info

            logging.info(f"Deployment {deployment_id} completed successfully")
            return deployment_info

        except Exception as e:
            logging.error(f"Deployment {deployment_id} failed: {e}")
            # Rollback operation
            await self._rollback_deployment(deployment_id)
            raise

    def _generate_deployment_id(self) -> str:
        """Generate deployment ID"""
        timestamp = datetime.utcnow().strftime("%Y%m%d_%H%M%S")
        random_str = ''.join(random.choices(string.ascii_lowercase + string.digits, k=6))
        return f"deploy_{timestamp}_{random_str}"

    async def _stage_preparation(self, deployment_id: str,
                                strategy_config: StrategyConfig,
                                deployment_config: DeploymentConfig):
        """Preparation stage"""
        logging.info(f"Stage 1: Preparation for {deployment_id}")

        # Check strategy file
        strategy_file = f"strategies/{strategy_config.name}.py"
        if not os.path.exists(strategy_file):
            raise FileNotFoundError(f"Strategy file not found: {strategy_file}")

        # Check dependencies
        if strategy_config.dependencies:
            for dep in strategy_config.dependencies:
                if not self._check_dependency(dep):
                    raise ImportError(f"Missing dependency: {dep}")

        # Validate configuration
        self._validate_config(strategy_config, deployment_config)

        # Create deployment directory
        deploy_dir = f"deployments/{deployment_id}"
        os.makedirs(deploy_dir, exist_ok=True)

        # Copy strategy file
        shutil.copy(strategy_file, f"{deploy_dir}/strategy.py")

        # Save configuration
        with open(f"{deploy_dir}/config.yaml", 'w') as f:
            yaml.dump({
                'strategy': strategy_config.__dict__,
                'deployment': deployment_config.__dict__
            }, f)

    async def _stage_validation(self, deployment_id: str, strategy_config: StrategyConfig):
        """Validation stage"""
        logging.info(f"Stage 2: Validation for {deployment_id}")

        # Load strategy class
        strategy_class = self._load_strategy_class(strategy_config.name)

        # Instantiate strategy (using simulation environment)
        from nautilus_trader.model.data import QuoteTick
        from nautilus_trader.model.identifiers import InstrumentId
        from nautilus_trader.test_kit.stubs import TestStubs

        # Create test environment
        strategy = strategy_class(config=strategy_config.parameters)

        # Validate strategy methods
        required_methods = ['on_quote_tick', 'on_trade_tick', 'on_bar']
        for method in required_methods:
            if not hasattr(strategy, method):
                raise ValueError(f"Strategy missing required method: {method}")

        # Test strategy initialization
        try:
            # Simulate initialization
            pass
        except Exception as e:
            raise ValueError(f"Strategy initialization failed: {e}")

    async def _stage_staging(self, deployment_id: str, strategy_config: StrategyConfig):
        """Staging stage"""
        logging.info(f"Stage 3: Staging for {deployment_id}")

        # Run strategy in test environment
        test_trader = self._create_test_trader()
        strategy = self._load_strategy_class(strategy_config.name)(config=strategy_config.parameters)

        # Run short duration test
        test_duration = 60  # 60 seconds
        start_time = time.time()

        while time.time() - start_time < test_duration:
            # Simulate market data
            await self._simulate_market_data(strategy)
            await asyncio.sleep(0.1)

        # Check strategy status
        if not self._validate_strategy_state(strategy):
            raise ValueError("Strategy validation failed in staging")

    async def _stage_production(self, deployment_id: str,
                               strategy_config: StrategyConfig,
                               deployment_config: DeploymentConfig):
        """Production deployment stage"""
        logging.info(f"Stage 4: Production deployment for {deployment_id}")

        # Create production environment trading node
        trading_node = await self._create_production_node(deployment_config)

        # Load strategy
        strategy = self._load_strategy_class(strategy_config.name)(config=strategy_config.parameters)

        # Configure risk control
        risk_engine = self._create_risk_engine(deployment_config)
        trading_node.add_risk_engine(risk_engine)

        # Deploy strategy
        await trading_node.add_strategy(strategy)

        # Start monitoring
        monitor = StrategyMonitor(deployment_id, strategy, deployment_config)
        await monitor.start()

        # Save monitor reference
        self.active_deployments[deployment_id]['monitor'] = monitor

        logging.info(f"Strategy {strategy_config.name} deployed successfully")

    def _validate_config(self, strategy_config: StrategyConfig, deployment_config: DeploymentConfig):
        """Validate configuration"""
        # Check required parameters
        if not strategy_config.instruments:
            raise ValueError("Strategy must specify at least one instrument")

        if deployment_config.initial_capital <= 0:
            raise ValueError("Initial capital must be positive")

        # Check risk limits
        if deployment_config.max_daily_loss <= 0:
            raise ValueError("Max daily loss must be positive")

    def _check_dependency(self, dependency: str) -> bool:
        """Check if dependency is satisfied"""
        try:
            __import__(dependency)
            return True
        except ImportError:
            return False

    def _load_strategy_class(self, strategy_name: str):
        """Dynamically load strategy class"""
        module = importlib.import_module(f"strategies.{strategy_name}")
        # Assume strategy class uses camel case naming
        class_name = ''.join(word.capitalize() for word in strategy_name.split('_'))
        return getattr(module, class_name)

    async def _create_production_node(self, deployment_config: DeploymentConfig):
        """Create production trading node"""
        from nautilus_trader.live.node import TradingNode
        from nautilus_trader.config import LiveExecEngineConfig

        # Configure execution engine
        exec_config = LiveExecEngineConfig(
            debug=False,
            barrier_init_timeout=30.0
        )

        # Create trading node
        node = TradingNode(config=exec_config)

        # Configure exchange adapters
        await self._setup_exchange_adapters(node, deployment_config)

        return node

    async def _setup_exchange_adapters(self, node, deployment_config: DeploymentConfig):
        """Setup exchange adapters"""
        from nautilus_trader.adapters.binance.factories import BinanceLiveExecClientFactory
        from nautilus_trader.adapters.binance.config import BinanceExecClientConfig

        # Configure Binance adapter
        if deployment_config.exchange.lower() == 'binance':
            config = BinanceExecClientConfig(
                api_key=os.getenv('BINANCE_API_KEY'),
                api_secret=os.getenv('BINANCE_SECRET_KEY'),
                testnet=deployment_config.environment == 'staging'
            )

            factory = BinanceLiveExecClientFactory(config=config)
            await node.add_client_factory(factory)

    def _create_risk_engine(self, deployment_config: DeploymentConfig):
        """Create risk engine"""
        from nautilus_trader.execution.risk import RiskEngine
        from nautilus_trader.model.enums import OrderSide

        risk_engine = RiskEngine(
            max_order_size=deployment_config.max_position_size,
            max_notional_per_order=deployment_config.max_position_size * 100,  # Assume price
        )

        # Add risk rules
        risk_engine.add_rule(
            self._create_max_position_rule(deployment_config.max_position_size)
        )
        risk_engine.add_rule(
            self._create_daily_loss_rule(deployment_config.max_daily_loss)
        )
        risk_engine.add_rule(
            self._create_drawdown_rule(deployment_config.max_drawdown)
        )

        return risk_engine

    async def rollback_deployment(self, deployment_id: str):
        """Rollback deployment"""
        if deployment_id in self.active_deployments:
            deployment_info = self.active_deployments[deployment_id]
            monitor = deployment_info.get('monitor')

            if monitor:
                await monitor.stop()

            # Stop strategy
            await self._stop_strategy(deployment_id)

            # Update status
            deployment_info['status'] = 'rolled_back'
            deployment_info['rollback_time'] = datetime.utcnow().isoformat()

            del self.active_deployments[deployment_id]

            logging.info(f"Deployment {deployment_id} rolled back successfully")

    async def _stop_strategy(self, deployment_id: str):
        """Stop strategy"""
        # Implement strategy stop logic
        pass
```

### 15.1.2 Configuration Management

```python
class ConfigManager:
    """Configuration manager"""

    def __init__(self, config_dir: str = "configs"):
        self.config_dir = Path(config_dir)
        self.configs = {}
        self.encryption_key = self._get_or_create_key()

    def _get_or_create_key(self) -> bytes:
        """Get or create encryption key"""
        key_file = self.config_dir / ".encryption_key"
        if key_file.exists():
            return key_file.read_bytes()
        else:
            from cryptography.fernet import Fernet
            key = Fernet.generate_key()
            key_file.write_bytes(key)
            return key

    def save_config(self, name: str, config: dict, encrypt: bool = False):
        """Save configuration"""
        config_path = self.config_dir / f"{name}.json"

        if encrypt:
            from cryptography.fernet import Fernet
            f = Fernet(self.encryption_key)
            config_json = json.dumps(config).encode()
            encrypted = f.encrypt(config_json)
            config_path.write_bytes(encrypted)
        else:
            with open(config_path, 'w') as f:
                json.dump(config, f, indent=2)

        self.configs[name] = config

    def load_config(self, name: str, encrypted: bool = False) -> dict:
        """Load configuration"""
        if name in self.configs:
            return self.configs[name]

        config_path = self.config_dir / f"{name}.json"

        if not config_path.exists():
            raise FileNotFoundError(f"Config not found: {name}")

        if encrypted:
            from cryptography.fernet import Fernet
            f = Fernet(self.encryption_key)
            encrypted = config_path.read_bytes()
            decrypted = f.decrypt(encrypted)
            config = json.loads(decrypted.decode())
        else:
            with open(config_path, 'r') as f:
                config = json.load(f)

        self.configs[name] = config
        return config

    def validate_strategy_config(self, config: dict) -> bool:
        """Validate strategy configuration"""
        required_fields = ['name', 'version', 'instruments', 'parameters']
        for field in required_fields:
            if field not in config:
                raise ValueError(f"Missing required field: {field}")

        # Validate parameter types
        if not isinstance(config['instruments'], list):
            raise ValueError("Instruments must be a list")

        if not isinstance(config['parameters'], dict):
            raise ValueError("Parameters must be a dict")

        return True

    def merge_configs(self, base_config: dict, override_config: dict) -> dict:
        """Merge configurations"""
        merged = base_config.copy()

        def deep_merge(base: dict, override: dict):
            for key, value in override.items():
                if key in base and isinstance(base[key], dict) and isinstance(value, dict):
                    deep_merge(base[key], value)
                else:
                    base[key] = value

        deep_merge(merged, override_config)
        return merged
```

## 15.2 Strategy Lifecycle Management

### 15.2.1 Lifecycle States

```python
from enum import Enum
import asyncio

class StrategyState(Enum):
    """Strategy state enumeration"""
    INITIALIZING = "initializing"      # Initializing
    IDLE = "idle"                     # Idle
    RUNNING = "running"               # Running
    PAUSED = "paused"                 # Paused
    STOPPING = "stopping"            # Stopping
    STOPPED = "stopped"              # Stopped
    ERROR = "error"                  # Error state
    MAINTENANCE = "maintenance"       # Maintenance

class StrategyLifecycleManager:
    """Strategy lifecycle manager"""

    def __init__(self):
        self.strategies = {}
        self.state_transitions = {
            StrategyState.INITIALIZING: [StrategyState.IDLE, StrategyState.ERROR],
            StrategyState.IDLE: [StrategyState.RUNNING, StrategyState.STOPPED],
            StrategyState.RUNNING: [StrategyState.PAUSED, StrategyState.STOPPING, StrategyState.ERROR],
            StrategyState.PAUSED: [StrategyState.RUNNING, StrategyState.STOPPING],
            StrategyState.STOPPING: [StrategyState.STOPPED, StrategyState.ERROR],
            StrategyState.STOPPED: [StrategyState.INITIALIZING],
            StrategyState.ERROR: [StrategyState.INITIALIZING, StrategyState.STOPPED],
            StrategyState.MAINTENANCE: [StrategyState.RUNNING, StrategyState.STOPPED]
        }

    async def register_strategy(self, strategy_id: str, strategy, initial_state: StrategyState = StrategyState.INITIALIZING):
        """Register strategy"""
        self.strategies[strategy_id] = {
            'strategy': strategy,
            'state': initial_state,
            'state_history': [(initial_state, datetime.utcnow())],
            'metrics': {},
            'last_heartbeat': datetime.utcnow()
        }

        # Trigger state change callback
        await self._on_state_change(strategy_id, None, initial_state)

    async def transition_state(self, strategy_id: str, new_state: StrategyState):
        """Transition strategy state"""
        if strategy_id not in self.strategies:
            raise ValueError(f"Strategy not found: {strategy_id}")

        current_state = self.strategies[strategy_id]['state']

        # Validate if state transition is legal
        if new_state not in self.state_transitions.get(current_state, []):
            raise ValueError(f"Invalid state transition: {current_state} -> {new_state}")

        # Execute state transition
        old_state = current_state
        self.strategies[strategy_id]['state'] = new_state
        self.strategies[strategy_id]['state_history'].append((new_state, datetime.utcnow()))

        # Execute state entry/exit logic
        await self._exit_state(strategy_id, old_state)
        await self._enter_state(strategy_id, new_state)

        # Trigger callback
        await self._on_state_change(strategy_id, old_state, new_state)

        logging.info(f"Strategy {strategy_id} transitioned: {old_state} -> {new_state}")

    async def _enter_state(self, strategy_id: str, state: StrategyState):
        """Handle when entering state"""
        strategy_info = self.strategies[strategy_id]
        strategy = strategy_info['strategy']

        if state == StrategyState.INITIALIZING:
            await strategy.initialize()
        elif state == StrategyState.RUNNING:
            await strategy.start()
        elif state == StrategyState.PAUSED:
            await strategy.pause()
        elif state == StrategyState.STOPPING:
            await strategy.stop()
        elif state == StrategyState.STOPPED:
            await strategy.cleanup()

    async def _exit_state(self, strategy_id: str, state: StrategyState):
        """Handle when exiting state"""
        # Cleanup work when exiting state
        pass

    async def _on_state_change(self, strategy_id: str, old_state: Optional[StrategyState], new_state: StrategyState):
        """State change callback"""
        # Notify monitoring system
        # Update UI
        # Send alerts, etc.
        pass

    async def start_strategy(self, strategy_id: str):
        """Start strategy"""
        await self.transition_state(strategy_id, StrategyState.RUNNING)

    async def stop_strategy(self, strategy_id: str):
        """Stop strategy"""
        await self.transition_state(strategy_id, StrategyState.STOPPING)
        # Wait for strategy to completely stop
        while self.get_strategy_state(strategy_id) != StrategyState.STOPPED:
            await asyncio.sleep(0.1)

    async def pause_strategy(self, strategy_id: str):
        """Pause strategy"""
        await self.transition_state(strategy_id, StrategyState.PAUSED)

    async def resume_strategy(self, strategy_id: str):
        """Resume strategy"""
        await self.transition_state(strategy_id, StrategyState.RUNNING)

    def get_strategy_state(self, strategy_id: str) -> Optional[StrategyState]:
        """Get strategy state"""
        if strategy_id in self.strategies:
            return self.strategies[strategy_id]['state']
        return None

    def get_strategy_metrics(self, strategy_id: str) -> dict:
        """Get strategy metrics"""
        if strategy_id not in self.strategies:
            return {}

        return self.strategies[strategy_id]['metrics']

    def update_strategy_metrics(self, strategy_id: str, metrics: dict):
        """Update strategy metrics"""
        if strategy_id in self.strategies:
            self.strategies[strategy_id]['metrics'].update(metrics)

    async def heartbeat_check(self):
        """Heartbeat check"""
        current_time = datetime.utcnow()
        for strategy_id, info in self.strategies.items():
            last_heartbeat = info['last_heartbeat']
            state = info['state']

            # If strategy is in running state but no heartbeat for more than 5 minutes
            if state == StrategyState.RUNNING and (current_time - last_heartbeat).seconds > 300:
                logging.warning(f"Strategy {strategy_id} heartbeat timeout")
                await self.transition_state(strategy_id, StrategyState.ERROR)

    def update_heartbeat(self, strategy_id: str):
        """Update heartbeat"""
        if strategy_id in self.strategies:
            self.strategies[strategy_id]['last_heartbeat'] = datetime.utcnow()
```

### 15.2.2 Health Checks

```python
import psutil
import GPUtil

class StrategyHealthChecker:
    """Strategy health checker"""

    def __init__(self, check_interval: float = 30.0):
        self.check_interval = check_interval
        self.health_checks = {}
        self.alert_thresholds = {
            'cpu_percent': 80.0,
            'memory_percent': 85.0,
            'latency_ms': 1000.0,
            'error_rate': 0.05
        }

    def register_health_check(self, strategy_id: str, check_func):
        """Register health check function"""
        if strategy_id not in self.health_checks:
            self.health_checks[strategy_id] = []
        self.health_checks[strategy_id].append(check_func)

    async def run_health_checks(self, strategy_id: str) -> dict:
        """Run health checks"""
        if strategy_id not in self.health_checks:
            return {'status': 'unknown', 'checks': []}

        checks = []
        overall_healthy = True

        for check_func in self.health_checks[strategy_id]:
            try:
                result = await check_func()
                checks.append(result)
                if not result.get('healthy', True):
                    overall_healthy = False
            except Exception as e:
                checks.append({
                    'name': check_func.__name__,
                    'healthy': False,
                    'message': str(e)
                })
                overall_healthy = False

        return {
            'status': 'healthy' if overall_healthy else 'unhealthy',
            'timestamp': datetime.utcnow().isoformat(),
            'checks': checks
        }

# Predefined health checks
class PredefinedHealthChecks:
    """Predefined health checks"""

    @staticmethod
    def cpu_usage_check(strategy):
        """CPU usage check"""
        process = psutil.Process()
        cpu_percent = process.cpu_percent()

        return {
            'name': 'cpu_usage',
            'healthy': cpu_percent < 80.0,
            'value': cpu_percent,
            'unit': 'percent',
            'message': f'CPU usage: {cpu_percent:.1f}%'
        }

    @staticmethod
    def memory_usage_check(strategy):
        """Memory usage check"""
        process = psutil.Process()
        memory_info = process.memory_info()
        memory_mb = memory_info.rss / 1024 / 1024
        memory_percent = process.memory_percent()

        return {
            'name': 'memory_usage',
            'healthy': memory_percent < 85.0,
            'value': memory_mb,
            'unit': 'MB',
            'message': f'Memory usage: {memory_mb:.1f}MB ({memory_percent:.1f}%)'
        }

    @staticmethod
    def order_latency_check(strategy):
        """Order latency check"""
        if hasattr(strategy, 'get_order_latency'):
            latency_ms = strategy.get_order_latency()
            return {
                'name': 'order_latency',
                'healthy': latency_ms < 1000.0,
                'value': latency_ms,
                'unit': 'ms',
                'message': f'Order latency: {latency_ms:.2f}ms'
            }
        else:
            return {
                'name': 'order_latency',
                'healthy': True,
                'message': 'Latency data not available'
            }

    @staticmethod
    def position_check(strategy):
        """Position check"""
        if hasattr(strategy, 'portfolio'):
            positions = strategy.portfolio.positions()
            # Check for abnormal positions
            for pos in positions:
                if pos.quantity < 0 and not strategy.config.get('allow_shorting', False):
                    return {
                        'name': 'position_check',
                        'healthy': False,
                        'message': f'Unexpected short position in {pos.symbol}'
                    }
            return {
                'name': 'position_check',
                'healthy': True,
                'message': f'Positions OK ({len(positions)} positions)'
            }
        else:
            return {
                'name': 'position_check',
                'healthy': True,
                'message': 'Position data not available'
            }

    @staticmethod
    def error_rate_check(strategy):
        """Error rate check"""
        if hasattr(strategy, 'error_stats'):
            total_requests = strategy.error_stats.get('total_requests', 0)
            errors = strategy.error_stats.get('errors', 0)
            error_rate = errors / total_requests if total_requests > 0 else 0

            return {
                'name': 'error_rate',
                'healthy': error_rate < 0.05,
                'value': error_rate,
                'unit': 'ratio',
                'message': f'Error rate: {error_rate:.2%} ({errors}/{total_requests})'
            }
        else:
            return {
                'name': 'error_rate',
                'healthy': True,
                'message': 'Error stats not available'
            }
```

## 15.3 Real-time Monitoring

### 15.3.1 Monitoring System

```python
import asyncio
from datetime import datetime, timedelta

class StrategyMonitor:
    """Strategy monitor"""

    def __init__(self, strategy_id: str, strategy, deployment_config):
        self.strategy_id = strategy_id
        self.strategy = strategy
        self.deployment_config = deployment_config
        self.metrics_collector = MetricsCollector()
        self.alert_manager = AlertManager()
        self.dashboard = Dashboard()
        self.running = False

    async def start(self):
        """Start monitoring"""
        self.running = True
        logging.info(f"Starting monitor for strategy {self.strategy_id}")

        # Start monitoring tasks
        asyncio.create_task(self._monitor_loop())
        asyncio.create_task(self._collect_metrics_loop())
        asyncio.create_task(self._check_alerts_loop())

    async def stop(self):
        """Stop monitoring"""
        self.running = False
        logging.info(f"Stopping monitor for strategy {self.strategy_id}")

    async def _monitor_loop(self):
        """Main monitoring loop"""
        while self.running:
            try:
                # Basic health check
                health = await self._check_strategy_health()
                if not health['healthy']:
                    await self._handle_unhealthy(health)

                # Performance check
                performance = await self._check_performance()
                if not performance['acceptable']:
                    await self._handle_poor_performance(performance)

                # Risk check
                risk = await self._check_risk_metrics()
                if risk['breached']:
                    await self._handle_risk_breach(risk)

                await asyncio.sleep(10)  # Check every 10 seconds

            except Exception as e:
                logging.error(f"Monitor loop error: {e}")
                await asyncio.sleep(5)

    async def _collect_metrics_loop(self):
        """Metrics collection loop"""
        while self.running:
            try:
                metrics = await self._collect_metrics()
                await self.metrics_collector.store(self.strategy_id, metrics)
                await self.dashboard.update_metrics(self.strategy_id, metrics)
                await asyncio.sleep(5)  # Collect every 5 seconds
            except Exception as e:
                logging.error(f"Metrics collection error: {e}")
                await asyncio.sleep(5)

    async def _check_alerts_loop(self):
        """Alert check loop"""
        while self.running:
            try:
                alerts = await self._check_alert_conditions()
                for alert in alerts:
                    await self.alert_manager.send_alert(alert)
                await asyncio.sleep(30)  # Check every 30 seconds
            except Exception as e:
                logging.error(f"Alert check error: {e}")
                await asyncio.sleep(30)

    async def _check_strategy_health(self) -> dict:
        """Check strategy health status"""
        checks = []

        # CPU usage
        cpu_percent = psutil.cpu_percent(interval=1)
        checks.append({
            'metric': 'cpu',
            'value': cpu_percent,
            'threshold': 80.0,
            'healthy': cpu_percent < 80.0
        })

        # Memory usage
        memory = psutil.virtual_memory()
        checks.append({
            'metric': 'memory',
            'value': memory.percent,
            'threshold': 85.0,
            'healthy': memory.percent < 85.0
        })

        # Order latency
        if hasattr(self.strategy, 'get_order_latency'):
            latency = self.strategy.get_order_latency()
            checks.append({
                'metric': 'latency',
                'value': latency,
                'threshold': 1000.0,
                'healthy': latency < 1000.0
            })

        all_healthy = all(check['healthy'] for check in checks)
        return {'healthy': all_healthy, 'checks': checks}

    async def _check_performance(self) -> dict:
        """Check performance metrics"""
        performance = {'acceptable': True, 'metrics': {}}

        # Check PnL
        if hasattr(self.strategy, 'get_pnl'):
            pnl = self.strategy.get_pnl()
            performance['metrics']['pnl'] = pnl

            # Check if daily loss limit exceeded
            daily_loss = -min(pnl, 0)
            if daily_loss > self.deployment_config.max_daily_loss:
                performance['acceptable'] = False
                performance['reason'] = 'Daily loss exceeded'

        # Check Sharpe ratio
        if hasattr(self.strategy, 'get_sharpe_ratio'):
            sharpe = self.strategy.get_sharpe_ratio()
            performance['metrics']['sharpe'] = sharpe

            if sharpe < 0.5:  # Minimum Sharpe ratio requirement
                performance['acceptable'] = False
                performance['reason'] = 'Sharpe ratio too low'

        return performance

    async def _check_risk_metrics(self) -> dict:
        """Check risk metrics"""
        risk = {'breached': False, 'metrics': {}}

        # Check position size
        if hasattr(self.strategy, 'get_total_position'):
            total_position = abs(self.strategy.get_total_position())
            risk['metrics']['position_size'] = total_position

            if total_position > self.deployment_config.max_position_size:
                risk['breached'] = True
                risk['reason'] = 'Position size exceeded'

        # Check drawdown
        if hasattr(self.strategy, 'get_max_drawdown'):
            drawdown = self.strategy.get_max_drawdown()
            risk['metrics']['drawdown'] = drawdown

            if drawdown > self.deployment_config.max_drawdown:
                risk['breached'] = True
                risk['reason'] = 'Max drawdown exceeded'

        return risk

    async def _collect_metrics(self) -> dict:
        """Collect strategy metrics"""
        metrics = {
            'timestamp': datetime.utcnow().isoformat(),
            'strategy_id': self.strategy_id
        }

        # Basic metrics
        if hasattr(self.strategy, 'portfolio'):
            portfolio = self.strategy.portfolio
            metrics.update({
                'total_pnl': portfolio.total_pnl(),
                'unrealized_pnl': portfolio.unrealized_pnl(),
                'realized_pnl': portfolio.realized_pnl(),
                'position_count': len(portfolio.positions())
            })

        # Order statistics
        if hasattr(self.strategy, 'order_stats'):
            metrics.update(self.strategy.order_stats)

        # System metrics
        metrics.update({
            'cpu_percent': psutil.cpu_percent(),
            'memory_percent': psutil.virtual_memory().percent
        })

        return metrics

    async def _handle_unhealthy(self, health: dict):
        """Handle unhealthy state"""
        logging.warning(f"Strategy {self.strategy_id} unhealthy: {health}")

        # Send alert
        alert = {
            'type': 'health',
            'severity': 'warning',
            'message': f"Strategy health check failed",
            'details': health
        }
        await self.alert_manager.send_alert(alert)

    async def _handle_poor_performance(self, performance: dict):
        """Handle poor performance"""
        logging.warning(f"Strategy {self.strategy_id} poor performance: {performance}")

        # Send alert
        alert = {
            'type': 'performance',
            'severity': 'warning',
            'message': f"Strategy performance below threshold: {performance.get('reason')}",
            'details': performance['metrics']
        }
        await self.alert_manager.send_alert(alert)

    async def _handle_risk_breach(self, risk: dict):
        """Handle risk breach"""
        logging.error(f"Strategy {self.strategy_id} risk breach: {risk}")

        # Send critical alert
        alert = {
            'type': 'risk',
            'severity': 'critical',
            'message': f"Risk limit breached: {risk['reason']}",
            'details': risk['metrics']
        }
        await self.alert_manager.send_alert(alert)

        # Consider automatic stop loss
        if risk['metrics'].get('drawdown', 0) > self.deployment_config.max_drawdown * 1.5:
            logging.critical(f"Emergency stop triggered for strategy {self.strategy_id}")
            await self._emergency_stop()

    async def _emergency_stop(self):
        """Emergency stop"""
        # Stop new orders
        await self.strategy.stop_new_orders()

        # Try to close positions
        await self.strategy.close_all_positions()

        # Completely stop strategy
        await self.strategy.stop()

    async def _check_alert_conditions(self) -> List[dict]:
        """Check alert conditions"""
        alerts = []

        # Check for no trades for a long time
        if hasattr(self.strategy, 'last_trade_time'):
            if (datetime.utcnow() - self.strategy.last_trade_time).hours > 4:
                alerts.append({
                    'type': 'no_trades',
                    'severity': 'info',
                    'message': 'No trades in the last 4 hours'
                })

        # Check error rate
        if hasattr(self.strategy, 'error_stats'):
            error_rate = self.strategy.error_rate
            if error_rate > 0.05:
                alerts.append({
                    'type': 'high_error_rate',
                    'severity': 'warning',
                    'message': f'High error rate: {error_rate:.2%}'
                })

        return alerts
```

### 15.3.2 Alert System

```python
from enum import Enum
import smtplib
from email.mime.text import MIMEText
import requests

class AlertSeverity(Enum):
    INFO = "info"
    WARNING = "warning"
    CRITICAL = "critical"

class AlertChannel:
    """Alert channel base class"""

    async def send(self, alert: dict):
        raise NotImplementedError

class EmailAlertChannel(AlertChannel):
    """Email alert channel"""

    def __init__(self, smtp_config: dict, recipients: List[str]):
        self.smtp_config = smtp_config
        self.recipients = recipients

    async def send(self, alert: dict):
        """Send email alert"""
        subject = f"[Nautilus Alert] {alert['type'].upper()}"
        body = self._format_alert(alert)

        msg = MIMEText(body)
        msg['Subject'] = subject
        msg['From'] = self.smtp_config['from']
        msg['To'] = ', '.join(self.recipients)

        try:
            with smtplib.SMTP(self.smtp_config['host'], self.smtp_config['port']) as server:
                if self.smtp_config.get('use_tls'):
                    server.starttls()
                if self.smtp_config.get('username'):
                    server.login(self.smtp_config['username'], self.smtp_config['password'])
                server.send_message(msg)
        except Exception as e:
            logging.error(f"Failed to send email alert: {e}")

    def _format_alert(self, alert: dict) -> str:
        """Format alert message"""
        return f"""
Alert Type: {alert['type']}
Severity: {alert['severity']}
Message: {alert['message']}
Time: {datetime.utcnow().strftime('%Y-%m-%d %H:%M:%S UTC')}
Details: {alert.get('details', 'N/A')}
        """

class SlackAlertChannel(AlertChannel):
    """Slack alert channel"""

    def __init__(self, webhook_url: str):
        self.webhook_url = webhook_url

    async def send(self, alert: dict):
        """Send Slack alert"""
        color = {
            AlertSeverity.INFO: "#36a64f",
            AlertSeverity.WARNING: "#ff9500",
            AlertSeverity.CRITICAL: "#ff0000"
        }.get(alert['severity'], "#36a64f")

        payload = {
            "attachments": [{
                "color": color,
                "title": f"Strategy Alert: {alert['type'].upper()}",
                "text": alert['message'],
                "fields": [
                    {"title": "Severity", "value": alert['severity'], "short": True},
                    {"title": "Time", "value": datetime.utcnow().strftime('%Y-%m-%d %H:%M:%S UTC'), "short": True}
                ],
                "footer": "Nautilus Trader",
                "ts": int(datetime.utcnow().timestamp())
            }]
        }

        if 'details' in alert:
            payload["attachments"][0]["fields"].append({
                "title": "Details",
                "value": str(alert['details']),
                "short": False
            })

        try:
            response = requests.post(self.webhook_url, json=payload)
            if response.status_code != 200:
                logging.error(f"Failed to send Slack alert: {response.text}")
        except Exception as e:
            logging.error(f"Failed to send Slack alert: {e}")

class AlertManager:
    """Alert manager"""

    def __init__(self):
        self.channels = []
        self.alert_history = []
        self.rate_limits = {}  # Prevent alert flooding

    def add_channel(self, channel: AlertChannel):
        """Add alert channel"""
        self.channels.append(channel)

    async def send_alert(self, alert: dict):
        """Send alert"""
        # Check rate limiting
        if self._is_rate_limited(alert):
            return

        # Record alert
        alert['timestamp'] = datetime.utcnow()
        self.alert_history.append(alert)

        # Send to all channels
        for channel in self.channels:
            try:
                await channel.send(alert)
            except Exception as e:
                logging.error(f"Failed to send alert via channel: {e}")

    def _is_rate_limited(self, alert: dict) -> bool:
        """Check if rate limited"""
        key = f"{alert['type']}_{alert.get('severity', 'info')}"
        now = datetime.utcnow()

        if key in self.rate_limits:
            last_sent = self.rate_limits[key]
            # Only send same type alert once within 5 minutes
            if (now - last_sent).seconds < 300:
                return True

        self.rate_limits[key] = now
        return False
```

## 15.4 Automated Operations

### 15.4.1 Auto Recovery

```python
class AutoRecoveryManager:
    """Auto recovery manager"""

    def __init__(self, deployment_manager: DeploymentManager):
        self.deployment_manager = deployment_manager
        self.recovery_strategies = {
            'strategy_crash': self._recover_from_crash,
            'exchange_disconnect': self._recover_from_disconnect,
            'high_latency': self._recover_from_high_latency,
            'memory_leak': self._recover_from_memory_leak
        }

    async def handle_failure(self, deployment_id: str, failure_type: str, details: dict):
        """Handle failure"""
        logging.info(f"Handling failure {failure_type} for deployment {deployment_id}")

        if failure_type in self.recovery_strategies:
            await self.recovery_strategies[failure_type](deployment_id, details)
        else:
            logging.warning(f"No recovery strategy for failure type: {failure_type}")

    async def _recover_from_crash(self, deployment_id: str, details: dict):
        """Recover from strategy crash"""
        # Stop crashed instance
        await self.deployment_manager.stop_strategy(deployment_id)

        # Wait for some time
        await asyncio.sleep(5)

        # Restart strategy
        deployment_info = self.deployment_manager.active_deployments.get(deployment_id)
        if deployment_info:
            await self._restart_deployment(deployment_info)

    async def _recover_from_disconnect(self, deployment_id: str, details: dict):
        """Recover from exchange connection disconnect"""
        logging.info(f"Attempting to reconnect to exchange for {deployment_id}")

        # Try to reconnect
        max_attempts = 5
        for attempt in range(max_attempts):
            try:
                # Reset connection
                await self._reset_exchange_connection(deployment_id)
                logging.info(f"Successfully reconnected for {deployment_id}")
                return
            except Exception as e:
                logging.error(f"Reconnection attempt {attempt + 1} failed: {e}")
                if attempt < max_attempts - 1:
                    await asyncio.sleep(2 ** attempt)  # Exponential backoff

        # If reconnection fails, restart strategy
        logging.error(f"Failed to reconnect, restarting deployment {deployment_id}")
        await self._restart_deployment(
            self.deployment_manager.active_deployments[deployment_id]
        )

    async def _recover_from_high_latency(self, deployment_id: str, details: dict):
        """Recover from high latency"""
        # Try to optimize strategy
        await self._optimize_strategy_performance(deployment_id)

        # If latency still high, consider restart
        if await self._check_latency(deployment_id) > 2000:  # 2 seconds
            logging.warning(f"High latency persists, restarting {deployment_id}")
            await self._restart_deployment(
                self.deployment_manager.active_deployments[deployment_id]
            )

    async def _recover_from_memory_leak(self, deployment_id: str, details: dict):
        """Recover from memory leak"""
        # Record memory usage
        memory_info = psutil.virtual_memory()
        logging.info(f"Memory usage before recovery: {memory_info.percent}%")

        # Try to clean cache
        await self._cleanup_strategy_cache(deployment_id)

        # If memory usage still high, restart strategy
        if memory_info.percent > 85:
            logging.warning(f"High memory usage persists, restarting {deployment_id}")
            await self._restart_deployment(
                self.deployment_manager.active_deployments[deployment_id]
            )

    async def _restart_deployment(self, deployment_info: dict):
        """Restart deployment"""
        deployment_id = deployment_info['id']
        logging.info(f"Restarting deployment {deployment_id}")

        try:
            # Stop current deployment
            await self.deployment_manager.stop_strategy(deployment_id)

            # Redeploy from configuration
            strategy_config = StrategyConfig(**deployment_info['config'])
            deployment_config = DeploymentConfig(**deployment_info['config'])

            # Redeploy with same configuration
            new_deployment = await self.deployment_manager.deploy_strategy(
                strategy_config,
                deployment_config
            )

            # Migrate monitor
            if 'monitor' in deployment_info:
                old_monitor = deployment_info['monitor']
                new_monitor = self.deployment_manager.active_deployments[new_deployment['id']]['monitor']
                await new_monitor.copy_state(old_monitor)

            logging.info(f"Deployment {deployment_id} restarted successfully as {new_deployment['id']}")

        except Exception as e:
            logging.error(f"Failed to restart deployment {deployment_id}: {e}")
            raise
```

## 15.5 Deployment Best Practices

### 15.5.1 Deployment Checklist

```python
class DeploymentChecklist:
    """Deployment checklist"""

    @staticmethod
    def pre_deployment_checklist():
        """Pre-deployment checklist"""
        return {
            "Code Quality": [
                "✓ Code has passed code review",
                "✓ Unit test coverage > 90%",
                "✓ Integration tests passed",
                "✓ Performance tests passed"
            ],
            "Configuration Check": [
                "✓ Production environment configuration ready",
                "✓ API keys configured",
                "✓ Risk parameters set",
                "✓ Monitoring rules configured"
            ],
            "Environment Preparation": [
                "✓ Sufficient server resources",
                "✓ Stable network connection",
                "✓ Firewall rules configured",
                "✓ Backup strategy established"
            ],
            "Risk Control": [
                "✓ Maximum position limits set",
                "✓ Daily loss limits set",
                "✓ Emergency stop button tested",
                "✓ Auto-recovery mechanism enabled"
            ]
        }

    @staticmethod
    def post_deployment_checklist():
        """Post-deployment checklist"""
        return {
            "Functionality Verification": [
                "□ Strategy started normally",
                "□ Receiving market data",
                "□ Orders can be submitted normally",
                "□ Risk control effective"
            ],
            "Monitoring Verification": [
                "□ Monitoring system normal",
                "□ Alert channels clear",
                "□ Complete log records",
                "□ Performance metrics normal"
            ],
            "Business Verification": [
                "□ Trading logic correct",
                "□ Position calculation accurate",
                "□ PnL calculation correct",
                "□ Risk indicators normal"
            ]
        }

    @staticmethod
    def run_checklist(checklist: dict) -> dict:
        """Run checklist"""
        results = {}
        for category, items in checklist.items():
            results[category] = []
            for item in items:
                status = "✓" if item.startswith("✓") else "□"
                results[category].append({
                    'item': item[2:],  # Remove prefix
                    'checked': status == "✓"
                })
        return results
```

## Summary

This chapter detailed the complete workflow for NautilusTrader strategy live deployment, covering:

1. **Deployment Process Management**
   - Multi-stage deployment process
   - Configuration management
   - Environment isolation

2. **Strategy Lifecycle**
   - State management
   - Lifecycle control
   - Health checks

3. **Real-time Monitoring System**
   - Multi-dimensional monitoring
   - Alert management
   - Performance analysis

4. **Automated Operations**
   - Automatic failure recovery
   - Intelligent restart
   - Resource optimization

5. **Best Practices**
   - Deployment checklist
   - Security measures
   - Operations specifications

Through this chapter, readers can safely and reliably deploy strategies to production environments and achieve comprehensive monitoring and automated operations.

## References

- [NautilusTrader Deployment Guide](https://docs.nautilustrader.io/)
- [Continuous Deployment Best Practices](https://docs.microsoft.com/en-us/azure/devops/learn/what-is-cicd)
- [Trading System Operations](https://www.oreilly.com/library/view/trading-system-operations/9781492048539/)

## Runnable Code Entry (Source Build)

- Entry script: docs/tutorials_cn/code/chapter15/run_hf_trading.py
- Key examples: docs/tutorials_cn/code/chapter15/market_making.py, docs/tutorials_cn/code/chapter15/latency_arbitrage.py

Dependency strategy: Includes Nautilus strategy shape where possible and always keeps a microstructure-oriented pure-Python fallback.

Verified command:

```bash
uv run --python 3.13 python docs/tutorials_cn/code/chapter15/run_hf_trading.py
```
