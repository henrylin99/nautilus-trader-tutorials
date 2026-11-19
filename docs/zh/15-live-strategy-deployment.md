# 第15章：实盘策略部署

## 学习目标

- 掌握策略从回测到实盘的完整流程
- 学会构建安全的实盘交易环境
- 理解策略生命周期管理
- 实现完善的监控和风险控制

## 15.1 策略部署流程

### 15.1.1 部署准备

```python
from dataclasses import dataclass
from typing import Dict, List, Optional
from enum import Enum
import yaml
import json
from datetime import datetime

class DeploymentStage(Enum):
    """部署阶段枚举"""
    PREPARATION = "preparation"       # 准备阶段
    VALIDATION = "validation"         # 验证阶段
    STAGING = "staging"              # 预发布环境
    PRODUCTION = "production"         # 生产环境
    MONITORING = "monitoring"         # 监控阶段

@dataclass
class StrategyConfig:
    """策略配置"""
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
    """部署配置"""
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
    """部署管理器"""

    def __init__(self, config_path: str = "deploy_config.yaml"):
        self.config_path = config_path
        self.config = self._load_config()
        self.deployment_history = []
        self.active_deployments = {}

    def _load_config(self) -> dict:
        """加载部署配置"""
        with open(self.config_path, 'r') as f:
            return yaml.safe_load(f)

    async def deploy_strategy(self, strategy_config: StrategyConfig,
                             deployment_config: DeploymentConfig) -> dict:
        """部署策略"""
        deployment_id = self._generate_deployment_id()

        logging.info(f"Starting deployment {deployment_id} for strategy {strategy_config.name}")

        try:
            # 阶段1: 准备
            await self._stage_preparation(deployment_id, strategy_config, deployment_config)

            # 阶段2: 验证
            await self._stage_validation(deployment_id, strategy_config)

            # 阶段3: 预发布测试
            await self._stage_staging(deployment_id, strategy_config)

            # 阶段4: 生产部署
            if deployment_config.environment == "production":
                await self._stage_production(deployment_id, strategy_config, deployment_config)

            # 记录部署信息
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
            # 回滚操作
            await self._rollback_deployment(deployment_id)
            raise

    def _generate_deployment_id(self) -> str:
        """生成部署ID"""
        timestamp = datetime.utcnow().strftime("%Y%m%d_%H%M%S")
        random_str = ''.join(random.choices(string.ascii_lowercase + string.digits, k=6))
        return f"deploy_{timestamp}_{random_str}"

    async def _stage_preparation(self, deployment_id: str,
                                strategy_config: StrategyConfig,
                                deployment_config: DeploymentConfig):
        """准备阶段"""
        logging.info(f"Stage 1: Preparation for {deployment_id}")

        # 检查策略文件
        strategy_file = f"strategies/{strategy_config.name}.py"
        if not os.path.exists(strategy_file):
            raise FileNotFoundError(f"Strategy file not found: {strategy_file}")

        # 检查依赖
        if strategy_config.dependencies:
            for dep in strategy_config.dependencies:
                if not self._check_dependency(dep):
                    raise ImportError(f"Missing dependency: {dep}")

        # 验证配置
        self._validate_config(strategy_config, deployment_config)

        # 创建部署目录
        deploy_dir = f"deployments/{deployment_id}"
        os.makedirs(deploy_dir, exist_ok=True)

        # 复制策略文件
        shutil.copy(strategy_file, f"{deploy_dir}/strategy.py")

        # 保存配置
        with open(f"{deploy_dir}/config.yaml", 'w') as f:
            yaml.dump({
                'strategy': strategy_config.__dict__,
                'deployment': deployment_config.__dict__
            }, f)

    async def _stage_validation(self, deployment_id: str, strategy_config: StrategyConfig):
        """验证阶段"""
        logging.info(f"Stage 2: Validation for {deployment_id}")

        # 加载策略类
        strategy_class = self._load_strategy_class(strategy_config.name)

        # 实例化策略（使用模拟环境）
        from nautilus_trader.model.data import QuoteTick
        from nautilus_trader.model.identifiers import InstrumentId
        from nautilus_trader.test_kit.stubs import TestStubs

        # 创建测试环境
        strategy = strategy_class(config=strategy_config.parameters)

        # 验证策略方法
        required_methods = ['on_quote_tick', 'on_trade_tick', 'on_bar']
        for method in required_methods:
            if not hasattr(strategy, method):
                raise ValueError(f"Strategy missing required method: {method}")

        # 测试策略初始化
        try:
            # 模拟初始化
            pass
        except Exception as e:
            raise ValueError(f"Strategy initialization failed: {e}")

    async def _stage_staging(self, deployment_id: str, strategy_config: StrategyConfig):
        """预发布阶段"""
        logging.info(f"Stage 3: Staging for {deployment_id}")

        # 在测试环境运行策略
        test_trader = self._create_test_trader()
        strategy = self._load_strategy_class(strategy_config.name)(config=strategy_config.parameters)

        # 运行短时间测试
        test_duration = 60  # 60秒
        start_time = time.time()

        while time.time() - start_time < test_duration:
            # 模拟市场数据
            await self._simulate_market_data(strategy)
            await asyncio.sleep(0.1)

        # 检查策略状态
        if not self._validate_strategy_state(strategy):
            raise ValueError("Strategy validation failed in staging")

    async def _stage_production(self, deployment_id: str,
                               strategy_config: StrategyConfig,
                               deployment_config: DeploymentConfig):
        """生产部署阶段"""
        logging.info(f"Stage 4: Production deployment for {deployment_id}")

        # 创建生产环境交易节点
        trading_node = await self._create_production_node(deployment_config)

        # 加载策略
        strategy = self._load_strategy_class(strategy_config.name)(config=strategy_config.parameters)

        # 配置风险控制
        risk_engine = self._create_risk_engine(deployment_config)
        trading_node.add_risk_engine(risk_engine)

        # 部署策略
        await trading_node.add_strategy(strategy)

        # 启动监控
        monitor = StrategyMonitor(deployment_id, strategy, deployment_config)
        await monitor.start()

        # 保存监控器引用
        self.active_deployments[deployment_id]['monitor'] = monitor

        logging.info(f"Strategy {strategy_config.name} deployed successfully")

    def _validate_config(self, strategy_config: StrategyConfig, deployment_config: DeploymentConfig):
        """验证配置"""
        # 检查必需参数
        if not strategy_config.instruments:
            raise ValueError("Strategy must specify at least one instrument")

        if deployment_config.initial_capital <= 0:
            raise ValueError("Initial capital must be positive")

        # 检查风险限制
        if deployment_config.max_daily_loss <= 0:
            raise ValueError("Max daily loss must be positive")

    def _check_dependency(self, dependency: str) -> bool:
        """检查依赖是否满足"""
        try:
            __import__(dependency)
            return True
        except ImportError:
            return False

    def _load_strategy_class(self, strategy_name: str):
        """动态加载策略类"""
        module = importlib.import_module(f"strategies.{strategy_name}")
        # 假设策略类名为驼峰命名
        class_name = ''.join(word.capitalize() for word in strategy_name.split('_'))
        return getattr(module, class_name)

    async def _create_production_node(self, deployment_config: DeploymentConfig):
        """创建生产交易节点"""
        from nautilus_trader.live.node import TradingNode
        from nautilus_trader.config import LiveExecEngineConfig

        # 配置执行引擎
        exec_config = LiveExecEngineConfig(
            debug=False,
            barrier_init_timeout=30.0
        )

        # 创建交易节点
        node = TradingNode(config=exec_config)

        # 配置交易所适配器
        await self._setup_exchange_adapters(node, deployment_config)

        return node

    async def _setup_exchange_adapters(self, node, deployment_config: DeploymentConfig):
        """设置交易所适配器"""
        from nautilus_trader.adapters.binance.factories import BinanceLiveExecClientFactory
        from nautilus_trader.adapters.binance.config import BinanceExecClientConfig

        # 配置Binance适配器
        if deployment_config.exchange.lower() == 'binance':
            config = BinanceExecClientConfig(
                api_key=os.getenv('BINANCE_API_KEY'),
                api_secret=os.getenv('BINANCE_SECRET_KEY'),
                testnet=deployment_config.environment == 'staging'
            )

            factory = BinanceLiveExecClientFactory(config=config)
            await node.add_client_factory(factory)

    def _create_risk_engine(self, deployment_config: DeploymentConfig):
        """创建风险引擎"""
        from nautilus_trader.execution.risk import RiskEngine
        from nautilus_trader.model.enums import OrderSide

        risk_engine = RiskEngine(
            max_order_size=deployment_config.max_position_size,
            max_notional_per_order=deployment_config.max_position_size * 100,  # 假设价格
        )

        # 添加风险规则
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
        """回滚部署"""
        if deployment_id in self.active_deployments:
            deployment_info = self.active_deployments[deployment_id]
            monitor = deployment_info.get('monitor')

            if monitor:
                await monitor.stop()

            # 停止策略
            await self._stop_strategy(deployment_id)

            # 更新状态
            deployment_info['status'] = 'rolled_back'
            deployment_info['rollback_time'] = datetime.utcnow().isoformat()

            del self.active_deployments[deployment_id]

            logging.info(f"Deployment {deployment_id} rolled back successfully")

    async def _stop_strategy(self, deployment_id: str):
        """停止策略"""
        # 实现策略停止逻辑
        pass
```

### 15.1.2 配置管理

```python
class ConfigManager:
    """配置管理器"""

    def __init__(self, config_dir: str = "configs"):
        self.config_dir = Path(config_dir)
        self.configs = {}
        self.encryption_key = self._get_or_create_key()

    def _get_or_create_key(self) -> bytes:
        """获取或创建加密密钥"""
        key_file = self.config_dir / ".encryption_key"
        if key_file.exists():
            return key_file.read_bytes()
        else:
            from cryptography.fernet import Fernet
            key = Fernet.generate_key()
            key_file.write_bytes(key)
            return key

    def save_config(self, name: str, config: dict, encrypt: bool = False):
        """保存配置"""
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
        """加载配置"""
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
        """验证策略配置"""
        required_fields = ['name', 'version', 'instruments', 'parameters']
        for field in required_fields:
            if field not in config:
                raise ValueError(f"Missing required field: {field}")

        # 验证参数类型
        if not isinstance(config['instruments'], list):
            raise ValueError("Instruments must be a list")

        if not isinstance(config['parameters'], dict):
            raise ValueError("Parameters must be a dict")

        return True

    def merge_configs(self, base_config: dict, override_config: dict) -> dict:
        """合并配置"""
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

## 15.2 策略生命周期管理

### 15.2.1 生命周期状态

```python
from enum import Enum
import asyncio

class StrategyState(Enum):
    """策略状态枚举"""
    INITIALIZING = "initializing"      # 初始化中
    IDLE = "idle"                     # 空闲
    RUNNING = "running"               # 运行中
    PAUSED = "paused"                 # 暂停
    STOPPING = "stopping"            # 停止中
    STOPPED = "stopped"              # 已停止
    ERROR = "error"                  # 错误状态
    MAINTENANCE = "maintenance"       # 维护中

class StrategyLifecycleManager:
    """策略生命周期管理器"""

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
        """注册策略"""
        self.strategies[strategy_id] = {
            'strategy': strategy,
            'state': initial_state,
            'state_history': [(initial_state, datetime.utcnow())],
            'metrics': {},
            'last_heartbeat': datetime.utcnow()
        }

        # 触发状态变更回调
        await self._on_state_change(strategy_id, None, initial_state)

    async def transition_state(self, strategy_id: str, new_state: StrategyState):
        """转换策略状态"""
        if strategy_id not in self.strategies:
            raise ValueError(f"Strategy not found: {strategy_id}")

        current_state = self.strategies[strategy_id]['state']

        # 验证状态转换是否合法
        if new_state not in self.state_transitions.get(current_state, []):
            raise ValueError(f"Invalid state transition: {current_state} -> {new_state}")

        # 执行状态转换
        old_state = current_state
        self.strategies[strategy_id]['state'] = new_state
        self.strategies[strategy_id]['state_history'].append((new_state, datetime.utcnow()))

        # 执行状态进入/退出逻辑
        await self._exit_state(strategy_id, old_state)
        await self._enter_state(strategy_id, new_state)

        # 触发回调
        await self._on_state_change(strategy_id, old_state, new_state)

        logging.info(f"Strategy {strategy_id} transitioned: {old_state} -> {new_state}")

    async def _enter_state(self, strategy_id: str, state: StrategyState):
        """进入状态时的处理"""
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
        """退出状态时的处理"""
        # 状态退出时的清理工作
        pass

    async def _on_state_change(self, strategy_id: str, old_state: Optional[StrategyState], new_state: StrategyState):
        """状态变更回调"""
        # 通知监控系统
        # 更新UI
        # 发送告警等
        pass

    async def start_strategy(self, strategy_id: str):
        """启动策略"""
        await self.transition_state(strategy_id, StrategyState.RUNNING)

    async def stop_strategy(self, strategy_id: str):
        """停止策略"""
        await self.transition_state(strategy_id, StrategyState.STOPPING)
        # 等待策略完全停止
        while self.get_strategy_state(strategy_id) != StrategyState.STOPPED:
            await asyncio.sleep(0.1)

    async def pause_strategy(self, strategy_id: str):
        """暂停策略"""
        await self.transition_state(strategy_id, StrategyState.PAUSED)

    async def resume_strategy(self, strategy_id: str):
        """恢复策略"""
        await self.transition_state(strategy_id, StrategyState.RUNNING)

    def get_strategy_state(self, strategy_id: str) -> Optional[StrategyState]:
        """获取策略状态"""
        if strategy_id in self.strategies:
            return self.strategies[strategy_id]['state']
        return None

    def get_strategy_metrics(self, strategy_id: str) -> dict:
        """获取策略指标"""
        if strategy_id not in self.strategies:
            return {}

        return self.strategies[strategy_id]['metrics']

    def update_strategy_metrics(self, strategy_id: str, metrics: dict):
        """更新策略指标"""
        if strategy_id in self.strategies:
            self.strategies[strategy_id]['metrics'].update(metrics)

    async def heartbeat_check(self):
        """心跳检查"""
        current_time = datetime.utcnow()
        for strategy_id, info in self.strategies.items():
            last_heartbeat = info['last_heartbeat']
            state = info['state']

            # 如果策略在运行状态但超过5分钟没有心跳
            if state == StrategyState.RUNNING and (current_time - last_heartbeat).seconds > 300:
                logging.warning(f"Strategy {strategy_id} heartbeat timeout")
                await self.transition_state(strategy_id, StrategyState.ERROR)

    def update_heartbeat(self, strategy_id: str):
        """更新心跳"""
        if strategy_id in self.strategies:
            self.strategies[strategy_id]['last_heartbeat'] = datetime.utcnow()
```

### 15.2.2 健康检查

```python
import psutil
import GPUtil

class StrategyHealthChecker:
    """策略健康检查器"""

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
        """注册健康检查函数"""
        if strategy_id not in self.health_checks:
            self.health_checks[strategy_id] = []
        self.health_checks[strategy_id].append(check_func)

    async def run_health_checks(self, strategy_id: str) -> dict:
        """运行健康检查"""
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

# 预定义健康检查
class PredefinedHealthChecks:
    """预定义的健康检查"""

    @staticmethod
    def cpu_usage_check(strategy):
        """CPU使用率检查"""
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
        """内存使用率检查"""
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
        """订单延迟检查"""
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
        """持仓检查"""
        if hasattr(strategy, 'portfolio'):
            positions = strategy.portfolio.positions()
            # 检查是否有异常持仓
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
        """错误率检查"""
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

## 15.3 实时监控

### 15.3.1 监控系统

```python
import asyncio
from datetime import datetime, timedelta

class StrategyMonitor:
    """策略监控器"""

    def __init__(self, strategy_id: str, strategy, deployment_config):
        self.strategy_id = strategy_id
        self.strategy = strategy
        self.deployment_config = deployment_config
        self.metrics_collector = MetricsCollector()
        self.alert_manager = AlertManager()
        self.dashboard = Dashboard()
        self.running = False

    async def start(self):
        """启动监控"""
        self.running = True
        logging.info(f"Starting monitor for strategy {self.strategy_id}")

        # 启动监控任务
        asyncio.create_task(self._monitor_loop())
        asyncio.create_task(self._collect_metrics_loop())
        asyncio.create_task(self._check_alerts_loop())

    async def stop(self):
        """停止监控"""
        self.running = False
        logging.info(f"Stopping monitor for strategy {self.strategy_id}")

    async def _monitor_loop(self):
        """主监控循环"""
        while self.running:
            try:
                # 基本健康检查
                health = await self._check_strategy_health()
                if not health['healthy']:
                    await self._handle_unhealthy(health)

                # 性能检查
                performance = await self._check_performance()
                if not performance['acceptable']:
                    await self._handle_poor_performance(performance)

                # 风险检查
                risk = await self._check_risk_metrics()
                if risk['breached']:
                    await self._handle_risk_breach(risk)

                await asyncio.sleep(10)  # 10秒检查一次

            except Exception as e:
                logging.error(f"Monitor loop error: {e}")
                await asyncio.sleep(5)

    async def _collect_metrics_loop(self):
        """指标收集循环"""
        while self.running:
            try:
                metrics = await self._collect_metrics()
                await self.metrics_collector.store(self.strategy_id, metrics)
                await self.dashboard.update_metrics(self.strategy_id, metrics)
                await asyncio.sleep(5)  # 5秒收集一次
            except Exception as e:
                logging.error(f"Metrics collection error: {e}")
                await asyncio.sleep(5)

    async def _check_alerts_loop(self):
        """告警检查循环"""
        while self.running:
            try:
                alerts = await self._check_alert_conditions()
                for alert in alerts:
                    await self.alert_manager.send_alert(alert)
                await asyncio.sleep(30)  # 30秒检查一次
            except Exception as e:
                logging.error(f"Alert check error: {e}")
                await asyncio.sleep(30)

    async def _check_strategy_health(self) -> dict:
        """检查策略健康状态"""
        checks = []

        # CPU使用率
        cpu_percent = psutil.cpu_percent(interval=1)
        checks.append({
            'metric': 'cpu',
            'value': cpu_percent,
            'threshold': 80.0,
            'healthy': cpu_percent < 80.0
        })

        # 内存使用率
        memory = psutil.virtual_memory()
        checks.append({
            'metric': 'memory',
            'value': memory.percent,
            'threshold': 85.0,
            'healthy': memory.percent < 85.0
        })

        # 订单延迟
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
        """检查性能指标"""
        performance = {'acceptable': True, 'metrics': {}}

        # 检查PnL
        if hasattr(self.strategy, 'get_pnl'):
            pnl = self.strategy.get_pnl()
            performance['metrics']['pnl'] = pnl

            # 检查是否超过日损失限制
            daily_loss = -min(pnl, 0)
            if daily_loss > self.deployment_config.max_daily_loss:
                performance['acceptable'] = False
                performance['reason'] = 'Daily loss exceeded'

        # 检查夏普比率
        if hasattr(self.strategy, 'get_sharpe_ratio'):
            sharpe = self.strategy.get_sharpe_ratio()
            performance['metrics']['sharpe'] = sharpe

            if sharpe < 0.5:  # 最低夏普比率要求
                performance['acceptable'] = False
                performance['reason'] = 'Sharpe ratio too low'

        return performance

    async def _check_risk_metrics(self) -> dict:
        """检查风险指标"""
        risk = {'breached': False, 'metrics': {}}

        # 检查持仓大小
        if hasattr(self.strategy, 'get_total_position'):
            total_position = abs(self.strategy.get_total_position())
            risk['metrics']['position_size'] = total_position

            if total_position > self.deployment_config.max_position_size:
                risk['breached'] = True
                risk['reason'] = 'Position size exceeded'

        # 检查回撤
        if hasattr(self.strategy, 'get_max_drawdown'):
            drawdown = self.strategy.get_max_drawdown()
            risk['metrics']['drawdown'] = drawdown

            if drawdown > self.deployment_config.max_drawdown:
                risk['breached'] = True
                risk['reason'] = 'Max drawdown exceeded'

        return risk

    async def _collect_metrics(self) -> dict:
        """收集策略指标"""
        metrics = {
            'timestamp': datetime.utcnow().isoformat(),
            'strategy_id': self.strategy_id
        }

        # 基本指标
        if hasattr(self.strategy, 'portfolio'):
            portfolio = self.strategy.portfolio
            metrics.update({
                'total_pnl': portfolio.total_pnl(),
                'unrealized_pnl': portfolio.unrealized_pnl(),
                'realized_pnl': portfolio.realized_pnl(),
                'position_count': len(portfolio.positions())
            })

        # 订单统计
        if hasattr(self.strategy, 'order_stats'):
            metrics.update(self.strategy.order_stats)

        # 系统指标
        metrics.update({
            'cpu_percent': psutil.cpu_percent(),
            'memory_percent': psutil.virtual_memory().percent
        })

        return metrics

    async def _handle_unhealthy(self, health: dict):
        """处理不健康状态"""
        logging.warning(f"Strategy {self.strategy_id} unhealthy: {health}")

        # 发送告警
        alert = {
            'type': 'health',
            'severity': 'warning',
            'message': f"Strategy health check failed",
            'details': health
        }
        await self.alert_manager.send_alert(alert)

    async def _handle_poor_performance(self, performance: dict):
        """处理性能不佳"""
        logging.warning(f"Strategy {self.strategy_id} poor performance: {performance}")

        # 发送告警
        alert = {
            'type': 'performance',
            'severity': 'warning',
            'message': f"Strategy performance below threshold: {performance.get('reason')}",
            'details': performance['metrics']
        }
        await self.alert_manager.send_alert(alert)

    async def _handle_risk_breach(self, risk: dict):
        """处理风险突破"""
        logging.error(f"Strategy {self.strategy_id} risk breach: {risk}")

        # 发送紧急告警
        alert = {
            'type': 'risk',
            'severity': 'critical',
            'message': f"Risk limit breached: {risk['reason']}",
            'details': risk['metrics']
        }
        await self.alert_manager.send_alert(alert)

        # 考虑自动止损
        if risk['metrics'].get('drawdown', 0) > self.deployment_config.max_drawdown * 1.5:
            logging.critical(f"Emergency stop triggered for strategy {self.strategy_id}")
            await self._emergency_stop()

    async def _emergency_stop(self):
        """紧急停止"""
        # 停止新订单
        await self.strategy.stop_new_orders()

        # 尝试平仓
        await self.strategy.close_all_positions()

        # 完全停止策略
        await self.strategy.stop()

    async def _check_alert_conditions(self) -> List[dict]:
        """检查告警条件"""
        alerts = []

        # 检查长时间无交易
        if hasattr(self.strategy, 'last_trade_time'):
            if (datetime.utcnow() - self.strategy.last_trade_time).hours > 4:
                alerts.append({
                    'type': 'no_trades',
                    'severity': 'info',
                    'message': 'No trades in the last 4 hours'
                })

        # 检查错误率
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

### 15.3.2 告警系统

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
    """告警通道基类"""

    async def send(self, alert: dict):
        raise NotImplementedError

class EmailAlertChannel(AlertChannel):
    """邮件告警通道"""

    def __init__(self, smtp_config: dict, recipients: List[str]):
        self.smtp_config = smtp_config
        self.recipients = recipients

    async def send(self, alert: dict):
        """发送邮件告警"""
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
        """格式化告警信息"""
        return f"""
Alert Type: {alert['type']}
Severity: {alert['severity']}
Message: {alert['message']}
Time: {datetime.utcnow().strftime('%Y-%m-%d %H:%M:%S UTC')}
Details: {alert.get('details', 'N/A')}
        """

class SlackAlertChannel(AlertChannel):
    """Slack告警通道"""

    def __init__(self, webhook_url: str):
        self.webhook_url = webhook_url

    async def send(self, alert: dict):
        """发送Slack告警"""
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
    """告警管理器"""

    def __init__(self):
        self.channels = []
        self.alert_history = []
        self.rate_limits = {}  # 防止告警轰炸

    def add_channel(self, channel: AlertChannel):
        """添加告警通道"""
        self.channels.append(channel)

    async def send_alert(self, alert: dict):
        """发送告警"""
        # 检查限流
        if self._is_rate_limited(alert):
            return

        # 记录告警
        alert['timestamp'] = datetime.utcnow()
        self.alert_history.append(alert)

        # 发送到所有通道
        for channel in self.channels:
            try:
                await channel.send(alert)
            except Exception as e:
                logging.error(f"Failed to send alert via channel: {e}")

    def _is_rate_limited(self, alert: dict) -> bool:
        """检查是否被限流"""
        key = f"{alert['type']}_{alert.get('severity', 'info')}"
        now = datetime.utcnow()

        if key in self.rate_limits:
            last_sent = self.rate_limits[key]
            # 同类型告警5分钟内只发送一次
            if (now - last_sent).seconds < 300:
                return True

        self.rate_limits[key] = now
        return False
```

## 15.4 自动化运维

### 15.4.1 自动恢复

```python
class AutoRecoveryManager:
    """自动恢复管理器"""

    def __init__(self, deployment_manager: DeploymentManager):
        self.deployment_manager = deployment_manager
        self.recovery_strategies = {
            'strategy_crash': self._recover_from_crash,
            'exchange_disconnect': self._recover_from_disconnect,
            'high_latency': self._recover_from_high_latency,
            'memory_leak': self._recover_from_memory_leak
        }

    async def handle_failure(self, deployment_id: str, failure_type: str, details: dict):
        """处理故障"""
        logging.info(f"Handling failure {failure_type} for deployment {deployment_id}")

        if failure_type in self.recovery_strategies:
            await self.recovery_strategies[failure_type](deployment_id, details)
        else:
            logging.warning(f"No recovery strategy for failure type: {failure_type}")

    async def _recover_from_crash(self, deployment_id: str, details: dict):
        """从策略崩溃恢复"""
        # 停止崩溃的实例
        await self.deployment_manager.stop_strategy(deployment_id)

        # 等待一段时间
        await asyncio.sleep(5)

        # 重启策略
        deployment_info = self.deployment_manager.active_deployments.get(deployment_id)
        if deployment_info:
            await self._restart_deployment(deployment_info)

    async def _recover_from_disconnect(self, deployment_id: str, details: dict):
        """从交易所连接断开恢复"""
        logging.info(f"Attempting to reconnect to exchange for {deployment_id}")

        # 尝试重新连接
        max_attempts = 5
        for attempt in range(max_attempts):
            try:
                # 重置连接
                await self._reset_exchange_connection(deployment_id)
                logging.info(f"Successfully reconnected for {deployment_id}")
                return
            except Exception as e:
                logging.error(f"Reconnection attempt {attempt + 1} failed: {e}")
                if attempt < max_attempts - 1:
                    await asyncio.sleep(2 ** attempt)  # 指数退避

        # 如果重连失败，重启策略
        logging.error(f"Failed to reconnect, restarting deployment {deployment_id}")
        await self._restart_deployment(
            self.deployment_manager.active_deployments[deployment_id]
        )

    async def _recover_from_high_latency(self, deployment_id: str, details: dict):
        """从高延迟恢复"""
        # 尝试优化策略
        await self._optimize_strategy_performance(deployment_id)

        # 如果延迟仍然过高，考虑重启
        if await self._check_latency(deployment_id) > 2000:  # 2秒
            logging.warning(f"High latency persists, restarting {deployment_id}")
            await self._restart_deployment(
                self.deployment_manager.active_deployments[deployment_id]
            )

    async def _recover_from_memory_leak(self, deployment_id: str, details: dict):
        """从内存泄漏恢复"""
        # 记录内存使用情况
        memory_info = psutil.virtual_memory()
        logging.info(f"Memory usage before recovery: {memory_info.percent}%")

        # 尝试清理缓存
        await self._cleanup_strategy_cache(deployment_id)

        # 如果内存使用仍然过高，重启策略
        if memory_info.percent > 85:
            logging.warning(f"High memory usage persists, restarting {deployment_id}")
            await self._restart_deployment(
                self.deployment_manager.active_deployments[deployment_id]
            )

    async def _restart_deployment(self, deployment_info: dict):
        """重启部署"""
        deployment_id = deployment_info['id']
        logging.info(f"Restarting deployment {deployment_id}")

        try:
            # 停止当前部署
            await self.deployment_manager.stop_strategy(deployment_id)

            # 从配置重新部署
            strategy_config = StrategyConfig(**deployment_info['config'])
            deployment_config = DeploymentConfig(**deployment_info['config'])

            # 使用相同的配置重新部署
            new_deployment = await self.deployment_manager.deploy_strategy(
                strategy_config,
                deployment_config
            )

            # 迁移监控器
            if 'monitor' in deployment_info:
                old_monitor = deployment_info['monitor']
                new_monitor = self.deployment_manager.active_deployments[new_deployment['id']]['monitor']
                await new_monitor.copy_state(old_monitor)

            logging.info(f"Deployment {deployment_id} restarted successfully as {new_deployment['id']}")

        except Exception as e:
            logging.error(f"Failed to restart deployment {deployment_id}: {e}")
            raise
```

## 15.5 部署最佳实践

### 15.5.1 部署检查清单

```python
class DeploymentChecklist:
    """部署检查清单"""

    @staticmethod
    def pre_deployment_checklist():
        """部署前检查清单"""
        return {
            "代码质量": [
                "✓ 代码已通过代码审查",
                "✓ 单元测试覆盖率 > 90%",
                "✓ 集成测试通过",
                "✓ 性能测试通过"
            ],
            "配置检查": [
                "✓ 生产环境配置已准备",
                "✓ API密钥已配置",
                "✓ 风险参数已设定",
                "✓ 监控规则已配置"
            ],
            "环境准备": [
                "✓ 服务器资源充足",
                "✓ 网络连接稳定",
                "✓ 防火墙规则已配置",
                "✓ 备份策略已制定"
            ],
            "风险控制": [
                "✓ 最大持仓限制已设置",
                "✓ 日损失限制已设置",
                "✓ 紧急停止按钮已测试",
                "✓ 自动恢复机制已启用"
            ]
        }

    @staticmethod
    def post_deployment_checklist():
        """部署后检查清单"""
        return {
            "功能验证": [
                "□ 策略正常启动",
                "□ 接收市场数据",
                "□ 订单可以正常提交",
                "□ 风险控制生效"
            ],
            "监控验证": [
                "□ 监控系统正常",
                "□ 告警通道通畅",
                "□ 日志记录完整",
                "□ 性能指标正常"
            ],
            "业务验证": [
                "□ 交易逻辑正确",
                "□ 持仓计算准确",
                "□ PnL计算正确",
                "□ 风险指标正常"
            ]
        }

    @staticmethod
    def run_checklist(checklist: dict) -> dict:
        """运行检查清单"""
        results = {}
        for category, items in checklist.items():
            results[category] = []
            for item in items:
                status = "✓" if item.startswith("✓") else "□"
                results[category].append({
                    'item': item[2:],  # 移除前缀
                    'checked': status == "✓"
                })
        return results
```

## 小结

本章详细介绍了NautilusTrader策略实盘部署的完整流程，涵盖了：

1. **部署流程管理**
   - 多阶段部署流程
   - 配置管理
   - 环境隔离

2. **策略生命周期**
   - 状态管理
   - 生命周期控制
   - 健康检查

3. **实时监控系统**
   - 多维度监控
   - 告警管理
   - 性能分析

4. **自动化运维**
   - 故障自动恢复
   - 智能重启
   - 资源优化

5. **最佳实践**
   - 部署检查清单
   - 安全措施
   - 运维规范

通过本章的学习，读者可以安全、可靠地将策略部署到生产环境，并实现完善的监控和自动化运维。

## 参考资料

- [NautilusTrader Deployment Guide](https://docs.nautilustrader.io/)
- [Continuous Deployment Best Practices](https://docs.microsoft.com/en-us/azure/devops/learn/what-is-cicd)
- [Trading System Operations](https://www.oreilly.com/library/view/trading-system-operations/9781492048539/)
## 本章可运行代码入口（源码版）

- 入口脚本：`docs/tutorials_cn/code/chapter15/run_hf_trading.py`
- 关键示例：`docs/tutorials_cn/code/chapter15/market_making.py`、`docs/tutorials_cn/code/chapter15/latency_arbitrage.py`

依赖策略说明：优先展示 Nautilus 策略形态，同时保留订单簿微观结构的纯 Python 回退路径。

已验证命令：

```bash
uv run --python 3.13 python docs/tutorials_cn/code/chapter15/run_hf_trading.py
```
