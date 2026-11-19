# 第13章：实盘交易架构

## 学习目标

- 理解实盘交易系统的核心架构组件
- 掌握生产环境部署的最佳实践
- 学会构建高可用、低延迟的交易基础设施
- 实现完善的监控、告警和故障恢复机制

## 13.1 实盘交易系统架构

### 13.1.1 系统架构概览

```python
from dataclasses import dataclass
from typing import Dict, List, Optional, Protocol
from enum import Enum
import asyncio
import logging
from datetime import datetime

# 系统组件类型
class ComponentType(Enum):
    """系统组件类型枚举"""
    STRATEGY = "strategy"           # 策略组件
    EXECUTION = "execution"         # 执行组件
    RISK = "risk"                  # 风险管理
    DATA = "data"                  # 数据服务
    MONITOR = "monitor"            # 监控组件
    GATEWAY = "gateway"            # 交易网关
    PERSISTENCE = "persistence"    # 持久化层

# 系统状态
class SystemState(Enum):
    """系统状态枚举"""
    INITIALIZING = "initializing"   # 初始化中
    RUNNING = "running"            # 运行中
    DEGRADED = "degraded"          # 降级运行
    STOPPED = "stopped"            # 已停止
    ERROR = "error"                # 错误状态
    RECOVERING = "recovering"       # 恢复中

@dataclass
class ComponentHealth:
    """组件健康状态"""
    component_id: str
    component_type: ComponentType
    status: SystemState
    last_heartbeat: datetime
    cpu_usage: float = 0.0
    memory_usage: float = 0.0
    latency_ms: float = 0.0
    error_count: int = 0
    message: str = ""

# 系统架构管理器
class SystemArchitecture:
    """实盘交易系统架构管理器"""

    def __init__(self, config: dict):
        self.config = config
        self.components = {}
        self.health_monitor = HealthMonitor()
        self.recovery_manager = RecoveryManager()
        self.load_balancer = LoadBalancer()
        self.circuit_breaker = CircuitBreaker()
        self.logger = logging.getLogger(__name__)

        # 初始化核心组件
        self._initialize_components()

    def _initialize_components(self):
        """初始化系统组件"""
        # 策略组件集群
        strategy_cluster = ComponentCluster(
            component_type=ComponentType.STRATEGY,
            instances=self.config.get('strategy_instances', 3),
            config=self.config.get('strategy_config', {})
        )
        self.components['strategy'] = strategy_cluster

        # 执行引擎
        execution_engine = ExecutionEngine(
            config=self.config.get('execution_config', {})
        )
        self.components['execution'] = execution_engine

        # 风险管理
        risk_manager = RiskManager(
            config=self.config.get('risk_config', {})
        )
        self.components['risk'] = risk_manager

        # 数据服务
        data_service = DataService(
            config=self.config.get('data_config', {})
        )
        self.components['data'] = data_service

        # 交易网关
        trading_gateway = TradingGateway(
            exchanges=self.config.get('exchanges', []),
            config=self.config.get('gateway_config', {})
        )
        self.components['gateway'] = trading_gateway

        # 监控系统
        monitoring_system = MonitoringSystem(
            config=self.config.get('monitoring_config', {})
        )
        self.components['monitor'] = monitoring_system

    async def start_system(self):
        """启动系统"""
        self.logger.info("Starting live trading system...")

        try:
            # 按依赖顺序启动组件
            startup_order = [
                'persistence',  # 持久化层
                'data',         # 数据服务
                'gateway',      # 交易网关
                'risk',         # 风险管理
                'execution',    # 执行引擎
                'strategy',     # 策略组件
                'monitor'       # 监控系统
            ]

            for component_name in startup_order:
                if component_name in self.components:
                    await self._start_component(component_name)

            # 启动健康监控
            await self.health_monitor.start()

            # 启动故障恢复
            await self.recovery_manager.start()

            self.logger.info("System started successfully")

        except Exception as e:
            self.logger.error(f"Failed to start system: {e}")
            await self.shutdown_system()
            raise

    async def _start_component(self, component_name: str):
        """启动单个组件"""
        component = self.components[component_name]
        try:
            if hasattr(component, 'start'):
                await component.start()
            self.logger.info(f"Component {component_name} started")
        except Exception as e:
            self.logger.error(f"Failed to start {component_name}: {e}")
            raise

    async def shutdown_system(self):
        """关闭系统"""
        self.logger.info("Shutting down system...")

        # 按相反顺序关闭组件
        shutdown_order = [
            'strategy',     # 策略组件
            'execution',    # 执行引擎
            'risk',         # 风险管理
            'gateway',      # 交易网关
            'data',         # 数据服务
            'monitor',      # 监控系统
            'persistence'   # 持久化层
        ]

        for component_name in shutdown_order:
            if component_name in self.components:
                await self._stop_component(component_name)

        # 关闭健康监控和故障恢复
        await self.health_monitor.stop()
        await self.recovery_manager.stop()

        self.logger.info("System shutdown complete")

    async def _stop_component(self, component_name: str):
        """停止单个组件"""
        component = self.components[component_name]
        try:
            if hasattr(component, 'stop'):
                await component.stop()
            self.logger.info(f"Component {component_name} stopped")
        except Exception as e:
            self.logger.error(f"Failed to stop {component_name}: {e}")

    async def get_system_status(self) -> dict:
        """获取系统状态"""
        status = {
            'overall_status': SystemState.RUNNING,
            'components': {},
            'timestamp': datetime.utcnow().isoformat()
        }

        for name, component in self.components.items():
            health = await self._get_component_health(component)
            status['components'][name] = health

            # 更新整体状态
            if health['status'] == SystemState.ERROR:
                status['overall_status'] = SystemState.ERROR
            elif health['status'] == SystemState.DEGRADED:
                status['overall_status'] = SystemState.DEGRADED

        return status

    async def _get_component_health(self, component) -> dict:
        """获取组件健康状态"""
        if hasattr(component, 'get_health'):
            return await component.get_health()
        else:
            return {
                'status': SystemState.RUNNING,
                'message': 'Health check not implemented'
            }

# 组件集群管理
class ComponentCluster:
    """组件集群管理器"""

    def __init__(self, component_type: ComponentType, instances: int, config: dict):
        self.component_type = component_type
        self.instances = instances
        self.config = config
        self.components = []
        self.load_balancer = RoundRobinLoadBalancer()
        self.health_checker = HealthChecker()

    async def start(self):
        """启动集群"""
        for i in range(self.instances):
            component = await self._create_component(i)
            await component.start()
            self.components.append(component)
            self.health_checker.add_component(component)

        await self.health_checker.start()

    async def stop(self):
        """停止集群"""
        await self.health_checker.stop()
        for component in self.components:
            await component.stop()

    async def _create_component(self, instance_id: int):
        """创建组件实例"""
        if self.component_type == ComponentType.STRATEGY:
            return StrategyInstance(instance_id, self.config)
        # 可以扩展其他组件类型
        raise NotImplementedError(f"Component type {self.component_type} not implemented")

    async def get_instance(self):
        """获取一个可用实例"""
        healthy_instances = await self.health_checker.get_healthy_instances()
        if not healthy_instances:
            raise Exception("No healthy instances available")

        # 使用负载均衡器选择实例
        selected = self.load_balancer.select(healthy_instances)
        return selected
```

### 13.1.2 容错和故障转移

```python
import random
import time
from typing import Set, Callable

# 健康检查器
class HealthChecker:
    """组件健康检查器"""

    def __init__(self, check_interval: float = 1.0):
        self.components = {}
        self.healthy_instances: Set = set()
        self.unhealthy_instances: Set = set()
        self.check_interval = check_interval
        self.running = False

    def add_component(self, component):
        """添加要监控的组件"""
        component_id = id(component)
        self.components[component_id] = component
        self.healthy_instances.add(component_id)

    async def start(self):
        """启动健康检查"""
        self.running = True
        asyncio.create_task(self._health_check_loop())

    async def stop(self):
        """停止健康检查"""
        self.running = False

    async def _health_check_loop(self):
        """健康检查循环"""
        while self.running:
            await self._check_all_components()
            await asyncio.sleep(self.check_interval)

    async def _check_all_components(self):
        """检查所有组件"""
        for component_id, component in self.components.items():
            try:
                is_healthy = await self._check_component_health(component)
                if is_healthy:
                    self.healthy_instances.add(component_id)
                    self.unhealthy_instances.discard(component_id)
                else:
                    self.unhealthy_instances.add(component_id)
                    self.healthy_instances.discard(component_id)
            except Exception as e:
                logging.error(f"Health check failed for component {component_id}: {e}")
                self.unhealthy_instances.add(component_id)
                self.healthy_instances.discard(component_id)

    async def _check_component_health(self, component) -> bool:
        """检查单个组件健康状态"""
        if hasattr(component, 'is_healthy'):
            return await component.is_healthy()
        else:
            # 默认检查：ping-pong
            try:
                await component.ping()
                return True
            except:
                return False

    async def get_healthy_instances(self):
        """获取健康实例列表"""
        return [self.components[comp_id] for comp_id in self.healthy_instances]

# 故障转移管理器
class FailoverManager:
    """故障转移管理器"""

    def __init__(self, primary_components: List, backup_components: List):
        self.primary_components = primary_components
        self.backup_components = backup_components
        self.active_component = None
        self.failover_count = 0
        self.recovery_count = 0

    async def initialize(self):
        """初始化故障转移"""
        # 选择主组件
        self.active_component = await self._select_primary()
        await self.active_component.activate()

    async def _select_primary(self):
        """选择主组件"""
        # 优先选择可用的主组件
        for component in self.primary_components:
            if await component.is_available():
                return component

        # 如果主组件都不可用，选择备用组件
        for component in self.backup_components:
            if await component.is_available():
                return component

        raise Exception("No available components")

    async def handle_failure(self, failed_component):
        """处理组件故障"""
        logging.warning(f"Component failure detected: {failed_component}")

        if failed_component == self.active_component:
            await self._initiate_failover()

        self.failover_count += 1

    async def _initiate_failover(self):
        """启动故障转移"""
        logging.info("Initiating failover...")

        # 停用故障组件
        try:
            await self.active_component.deactivate()
        except:
            pass

        # 选择新组件
        try:
            new_component = await self._select_backup()
            await new_component.activate()
            self.active_component = new_component
            logging.info(f"Failover completed to {new_component}")
        except Exception as e:
            logging.error(f"Failover failed: {e}")
            raise

    async def _select_backup(self):
        """选择备用组件"""
        available_backups = [
            comp for comp in self.backup_components
            if await comp.is_available()
        ]

        if not available_backups:
            raise Exception("No backup components available")

        return random.choice(available_backups)

    async def attempt_recovery(self):
        """尝试恢复故障组件"""
        logging.info("Attempting component recovery...")

        for component in self.primary_components:
            if component != self.active_component:
                try:
                    if await component.recover():
                        self.recovery_count += 1
                        logging.info(f"Component {component} recovered")
                        return True
                except Exception as e:
                    logging.error(f"Recovery failed for {component}: {e}")

        return False

# 断路器模式
class CircuitBreaker:
    """断路器实现"""

    def __init__(self, failure_threshold: int = 5, timeout: float = 60.0):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN

    async def call(self, func: Callable, *args, **kwargs):
        """通过断路器调用函数"""
        if self.state == 'OPEN':
            if self._should_attempt_reset():
                self.state = 'HALF_OPEN'
            else:
                raise Exception("Circuit breaker is OPEN")

        try:
            result = await func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _should_attempt_reset(self) -> bool:
        """是否应该尝试重置"""
        return (self.last_failure_time and
                time.time() - self.last_failure_time >= self.timeout)

    def _on_success(self):
        """成功时的处理"""
        self.failure_count = 0
        self.state = 'CLOSED'

    def _on_failure(self):
        """失败时的处理"""
        self.failure_count += 1
        self.last_failure_time = time.time()

        if self.failure_count >= self.failure_threshold:
            self.state = 'OPEN'

# 自动恢复管理器
class RecoveryManager:
    """自动恢复管理器"""

    def __init__(self):
        self.recovery_strategies = {}
        self.running = False

    def register_strategy(self, component_type: ComponentType, strategy):
        """注册恢复策略"""
        self.recovery_strategies[component_type] = strategy

    async def start(self):
        """启动恢复管理器"""
        self.running = True
        asyncio.create_task(self._recovery_loop())

    async def stop(self):
        """停止恢复管理器"""
        self.running = False

    async def _recovery_loop(self):
        """恢复循环"""
        while self.running:
            # 检查需要恢复的组件
            await self._check_and_recover()
            await asyncio.sleep(5.0)  # 每5秒检查一次

    async def _check_and_recover(self):
        """检查并执行恢复"""
        # 这里需要与健康检查器集成
        # 获取不健康的组件并尝试恢复
        pass

    async def recover_component(self, component):
        """恢复单个组件"""
        component_type = getattr(component, 'type', None)
        if component_type in self.recovery_strategies:
            strategy = self.recovery_strategies[component_type]
            await strategy.recover(component)

# 恢复策略基类
class RecoveryStrategy(Protocol):
    """恢复策略协议"""

    async def recover(self, component):
        """执行恢复"""
        ...

# 具体恢复策略实现
class RestartRecoveryStrategy:
    """重启恢复策略"""

    def __init__(self, max_attempts: int = 3):
        self.max_attempts = max_attempts

    async def recover(self, component):
        """通过重启恢复组件"""
        for attempt in range(self.max_attempts):
            try:
                logging.info(f"Attempting recovery restart {attempt + 1}/{self.max_attempts}")
                await component.restart()
                return True
            except Exception as e:
                logging.error(f"Recovery restart attempt {attempt + 1} failed: {e}")
                await asyncio.sleep(2 ** attempt)  # 指数退避

        return False

class ReloadConfigRecoveryStrategy:
    """重新加载配置恢复策略"""

    async def recover(self, component):
        """通过重新加载配置恢复组件"""
        try:
            await component.reload_config()
            return True
        except Exception as e:
            logging.error(f"Config reload recovery failed: {e}")
            return False
```

## 13.2 生产环境部署

### 13.2.1 Docker容器化

```dockerfile
# Dockerfile
FROM python:3.11-slim

# 设置工作目录
WORKDIR /app

# 安装系统依赖
RUN apt-get update && apt-get install -y \
    build-essential \
    curl \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# 复制依赖文件
COPY requirements.txt .

# 安装Python依赖
RUN pip install --no-cache-dir -r requirements.txt

# 复制应用代码
COPY . .

# 创建非root用户
RUN useradd --create-home --shell /bin/bash nautilus
USER nautilus

# 暴露端口
EXPOSE 8000 8001 8080

# 健康检查
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python health_check.py

# 启动命令
CMD ["python", "run_production.py"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  # 核心交易服务
  trading-engine:
    build: .
    environment:
      - ENV=production
      - CONFIG_PATH=/app/config/production.yaml
    volumes:
      - ./config:/app/config:ro
      - ./data:/app/data
      - ./logs:/app/logs
    networks:
      - trading-network
    restart: unless-stopped
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '1'
          memory: 2G

  # 数据服务
  data-service:
    build: .
    command: python run_data_service.py
    environment:
      - ENV=production
    volumes:
      - ./data:/app/data
      - ./logs:/app/logs
    networks:
      - trading-network
    restart: unless-stopped

  # 监控服务
  monitoring:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    networks:
      - trading-network

  # 日志聚合
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - ./monitoring/loki.yml:/etc/loki/local-config.yaml:ro
      - loki_data:/loki
    networks:
      - trading-network

  # 可视化
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards:ro
      - ./monitoring/grafana/datasources:/etc/grafana/provisioning/datasources:ro
    networks:
      - trading-network

  # 消息队列
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - trading-network

  # 数据库
  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=nautilus
      - POSTGRES_USER=nautilus
      - POSTGRES_PASSWORD=secure_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init_db.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks:
      - trading-network

  # 负载均衡器
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - trading-engine
    networks:
      - trading-network

volumes:
  prometheus_data:
  loki_data:
  grafana_data:
  redis_data:
  postgres_data:

networks:
  trading-network:
    driver: bridge
```

### 13.2.2 Kubernetes部署

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nautilus-trading
  labels:
    name: nautilus-trading

---
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nautilus-config
  namespace: nautilus-trading
data:
  config.yaml: |
    env: production
    log_level: INFO
    trading:
      max_position: 1000000
      risk_limit: 0.02
    exchanges:
      - name: binance
        api_key: ${BINANCE_API_KEY}
        secret_key: ${BINANCE_SECRET_KEY}
        testnet: false
      - name: bybit
        api_key: ${BYBIT_API_KEY}
        secret_key: ${BYBIT_SECRET_KEY}
        testnet: false

---
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: nautilus-secrets
  namespace: nautilus-trading
type: Opaque
data:
  binance-api-key: <base64-encoded-key>
  binance-secret-key: <base64-encoded-secret>
  bybit-api-key: <base64-encoded-key>
  bybit-secret-key: <base64-encoded-secret>
  db-password: <base64-encoded-password>

---
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: trading-engine
  namespace: nautilus-trading
  labels:
    app: trading-engine
spec:
  replicas: 3
  selector:
    matchLabels:
      app: trading-engine
  template:
    metadata:
      labels:
        app: trading-engine
    spec:
      containers:
      - name: trading-engine
        image: nautilus/trading-engine:latest
        ports:
        - containerPort: 8000
          name: http
        - containerPort: 8001
          name: grpc
        env:
        - name: ENV
          value: "production"
        - name: CONFIG_PATH
          value: "/app/config/config.yaml"
        - name: BINANCE_API_KEY
          valueFrom:
            secretKeyRef:
              name: nautilus-secrets
              key: binance-api-key
        - name: BINANCE_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: nautilus-secrets
              key: binance-secret-key
        volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true
        - name: data
          mountPath: /app/data
        - name: logs
          mountPath: /app/logs
        resources:
          requests:
            cpu: 1000m
            memory: 2Gi
          limits:
            cpu: 2000m
            memory: 4Gi
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: config
        configMap:
          name: nautilus-config
      - name: data
        persistentVolumeClaim:
          claimName: data-pvc
      - name: logs
        persistentVolumeClaim:
          claimName: logs-pvc

---
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: trading-engine-service
  namespace: nautilus-trading
spec:
  selector:
    app: trading-engine
  ports:
  - name: http
    port: 80
    targetPort: 8000
  - name: grpc
    port: 81
    targetPort: 8001
  type: ClusterIP

---
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: trading-engine-ingress
  namespace: nautilus-trading
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  tls:
  - hosts:
    - api.nautilus-trading.com
    secretName: nautilus-tls
  rules:
  - host: api.nautilus-trading.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: trading-engine-service
            port:
              number: 80

---
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: trading-engine-hpa
  namespace: nautilus-trading
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: trading-engine
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
```

## 13.3 监控和告警

### 13.3.1 监控系统架构

```python
from typing import Dict, Any
from datetime import datetime, timedelta
import json
import asyncio

class MonitoringSystem:
    """监控系统"""

    def __init__(self, config: dict):
        self.config = config
        self.metrics_collector = MetricsCollector()
        self.alert_manager = AlertManager()
        self.dashboard = Dashboard()
        self.storage = MetricsStorage()

    async def start(self):
        """启动监控系统"""
        await self.metrics_collector.start()
        await self.alert_manager.start()
        await self.dashboard.start()

    async def stop(self):
        """停止监控系统"""
        await self.metrics_collector.stop()
        await self.alert_manager.stop()
        await self.dashboard.stop()

# 指标收集器
class MetricsCollector:
    """指标收集器"""

    def __init__(self):
        self.collectors = {}
        self.interval = 1.0  # 收集间隔（秒）
        self.running = False

    def register_collector(self, name: str, collector_func):
        """注册指标收集器"""
        self.collectors[name] = collector_func

    async def start(self):
        """启动收集"""
        self.running = True
        asyncio.create_task(self._collection_loop())

    async def stop(self):
        """停止收集"""
        self.running = False

    async def _collection_loop(self):
        """收集循环"""
        while self.running:
            metrics = {}
            for name, collector in self.collectors.items():
                try:
                    metrics[name] = await collector()
                except Exception as e:
                    logging.error(f"Failed to collect {name}: {e}")

            # 发送到存储
            await self._store_metrics(metrics)

            await asyncio.sleep(self.interval)

    async def _store_metrics(self, metrics: dict):
        """存储指标"""
        timestamp = datetime.utcnow()
        for name, value in metrics.items():
            await MetricsStorage.store(name, value, timestamp)

# 业务指标收集器
class BusinessMetricsCollector:
    """业务指标收集器"""

    def __init__(self, trading_engine):
        self.trading_engine = trading_engine

    async def collect_trading_metrics(self) -> dict:
        """收集交易指标"""
        return {
            'total_orders': self.trading_engine.total_orders,
            'filled_orders': self.trading_engine.filled_orders,
            'total_volume': self.trading_engine.total_volume,
            'total_pnl': self.trading_engine.total_pnl,
            'open_positions': len(self.trading_engine.open_positions),
            'active_strategies': len(self.trading_engine.active_strategies)
        }

    async def collect_performance_metrics(self) -> dict:
        """收集性能指标"""
        return {
            'order_latency_p50': self.trading_engine.get_order_latency_percentile(50),
            'order_latency_p90': self.trading_engine.get_order_latency_percentile(90),
            'order_latency_p99': self.trading_engine.get_order_latency_percentile(99),
            'throughput_orders_per_sec': self.trading_engine.get_order_throughput(),
            'cpu_usage': psutil.cpu_percent(),
            'memory_usage': psutil.virtual_memory().percent
        }

    async def collect_risk_metrics(self) -> dict:
        """收集风险指标"""
        return {
            'portfolio_var': self.trading_engine.calculate_var(),
            'max_drawdown': self.trading_engine.calculate_max_drawdown(),
            'sharpe_ratio': self.trading_engine.calculate_sharpe_ratio(),
            'risk_utilization': self.trading_engine.get_risk_utilization(),
            'margin_usage': self.trading_engine.get_margin_usage()
        }

# 告警管理器
class AlertManager:
    """告警管理器"""

    def __init__(self):
        self.rules = []
        self.notifiers = []
        self.alert_history = []

    def add_rule(self, rule):
        """添加告警规则"""
        self.rules.append(rule)

    def add_notifier(self, notifier):
        """添加通知器"""
        self.notifiers.append(notifier)

    async def start(self):
        """启动告警管理"""
        asyncio.create_task(self._alert_loop())

    async def _alert_loop(self):
        """告警检查循环"""
        while True:
            for rule in self.rules:
                try:
                    if await rule.check():
                        await self._handle_alert(rule)
                except Exception as e:
                    logging.error(f"Alert rule check failed: {e}")

            await asyncio.sleep(5)

    async def _handle_alert(self, rule):
        """处理告警"""
        alert = {
            'rule': rule.name,
            'message': rule.message,
            'severity': rule.severity,
            'timestamp': datetime.utcnow()
        }

        # 检查是否已经发送过
        if self._should_send_alert(alert):
            self.alert_history.append(alert)

            # 发送通知
            for notifier in self.notifiers:
                await notifier.send(alert)

    def _should_send_alert(self, alert: dict) -> bool:
        """判断是否应该发送告警"""
        # 检查冷却时间
        cooldown = timedelta(minutes=5)
        for historical_alert in reversed(self.alert_history[-10:]):
            if (historical_alert['rule'] == alert['rule'] and
                datetime.utcnow() - historical_alert['timestamp'] < cooldown):
                return False
        return True

# 告警规则
class AlertRule:
    """告警规则"""

    def __init__(self, name: str, check_func, message: str, severity: str):
        self.name = name
        self.check_func = check_func
        self.message = message
        self.severity = severity

    async def check(self) -> bool:
        """检查告警条件"""
        return await self.check_func()

# 预定义告警规则
class TradingAlertRules:
    """交易告警规则"""

    @staticmethod
    def high_latency_rule(threshold_ms: float = 100):
        """高延迟告警规则"""
        async def check():
            from nautilus_trader.core.clock import LiveClock
            clock = LiveClock()
            latency = clock.get_avg_latency_ms()
            return latency > threshold_ms

        return AlertRule(
            name="high_latency",
            check_func=check,
            message=f"Order latency exceeds {threshold_ms}ms",
            severity="warning"
        )

    @staticmethod
    def high_error_rate_rule(threshold: float = 0.05):
        """高错误率告警规则"""
        async def check():
            # 获取最近5分钟的错误率
            error_rate = get_recent_error_rate(minutes=5)
            return error_rate > threshold

        return AlertRule(
            name="high_error_rate",
            check_func=check,
            message=f"Error rate exceeds {threshold*100}%",
            severity="critical"
        )

    @staticmethod
    def risk_limit_rule():
        """风险限制告警规则"""
        async def check():
            risk_usage = get_risk_utilization()
            return risk_usage > 0.9  # 90%

        return AlertRule(
            name="risk_limit",
            check_func=check,
            message="Risk utilization exceeds 90%",
            severity="critical"
        )

# 通知器
class EmailNotifier:
    """邮件通知器"""

    def __init__(self, smtp_config: dict):
        self.smtp_config = smtp_config

    async def send(self, alert: dict):
        """发送邮件告警"""
        import smtplib
        from email.mime.text import MIMEText

        subject = f"[Nautilus Alert] {alert['rule'].upper()}"
        body = f"""
        Alert: {alert['rule']}
        Message: {alert['message']}
        Severity: {alert['severity']}
        Time: {alert['timestamp']}
        """

        msg = MIMEText(body)
        msg['Subject'] = subject
        msg['From'] = self.smtp_config['from']
        msg['To'] = ', '.join(self.smtp_config['to'])

        try:
            with smtplib.SMTP(self.smtp_config['host']) as server:
                server.send_message(msg)
        except Exception as e:
            logging.error(f"Failed to send email alert: {e}")

class SlackNotifier:
    """Slack通知器"""

    def __init__(self, webhook_url: str):
        self.webhook_url = webhook_url

    async def send(self, alert: dict):
        """发送Slack告警"""
        import aiohttp

        color = {
            'info': '#36a64f',
            'warning': '#ff9500',
            'critical': '#ff0000'
        }.get(alert['severity'], '#36a64f')

        payload = {
            "attachments": [{
                "color": color,
                "title": f"Nautilus Alert: {alert['rule']}",
                "text": alert['message'],
                "fields": [{
                    "title": "Severity",
                    "value": alert['severity'],
                    "short": True
                }, {
                    "title": "Time",
                    "value": alert['timestamp'].strftime('%Y-%m-%d %H:%M:%S UTC'),
                    "short": True
                }]
            }]
        }

        try:
            async with aiohttp.ClientSession() as session:
                async with session.post(self.webhook_url, json=payload) as response:
                    if response.status != 200:
                        logging.error(f"Failed to send Slack alert: {response.status}")
        except Exception as e:
            logging.error(f"Failed to send Slack alert: {e}")
```

## 13.4 日志管理

### 13.4.1 结构化日志

```python
import structlog
import json
from typing import Any, Dict

# 配置结构化日志
def configure_logging(log_level: str = "INFO"):
    """配置结构化日志"""

    structlog.configure(
        processors=[
            structlog.stdlib.filter_by_level,
            structlog.stdlib.add_logger_name,
            structlog.stdlib.add_log_level,
            structlog.stdlib.PositionalArgumentsFormatter(),
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            structlog.processors.UnicodeDecoder(),
            structlog.processors.JSONRenderer()  # JSON格式
        ],
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
        wrapper_class=structlog.stdlib.BoundLogger,
        cache_logger_on_first_use=True,
    )

# 交易日志记录器
class TradingLogger:
    """交易日志记录器"""

    def __init__(self, strategy_id: str):
        self.logger = structlog.get_logger()
        self.strategy_id = strategy_id

    def log_order_submitted(self, order):
        """记录订单提交"""
        self.logger.info(
            "order_submitted",
            strategy_id=self.strategy_id,
            order_id=order.client_order_id,
            symbol=order.symbol,
            side=order.side.value,
            type=order.order_type.value,
            quantity=order.quantity,
            price=order.price if hasattr(order, 'price') else None
        )

    def log_order_filled(self, order, fill):
        """记录订单成交"""
        self.logger.info(
            "order_filled",
            strategy_id=self.strategy_id,
            order_id=order.client_order_id,
            fill_id=fill.id,
            symbol=fill.symbol,
            side=fill.side.value,
            quantity=fill.last_qty,
            price=fill.last_px,
            commission=fill.commission,
            timestamp=fill.ts_event
        )

    def log_strategy_signal(self, signal_type: str, signal_data: Dict):
        """记录策略信号"""
        self.logger.info(
            "strategy_signal",
            strategy_id=self.strategy_id,
            signal_type=signal_type,
            **signal_data
        )

    def log_risk_event(self, event_type: str, details: Dict):
        """记录风险事件"""
        self.logger.warning(
            "risk_event",
            strategy_id=self.strategy_id,
            event_type=event_type,
            **details
        )

    def log_error(self, error: Exception, context: Dict = None):
        """记录错误"""
        self.logger.error(
            "error_occurred",
            strategy_id=self.strategy_id,
            error_type=type(error).__name__,
            error_message=str(error),
            **(context or {})
        )

# 日志聚合器
class LogAggregator:
    """日志聚合器"""

    def __init__(self, config: dict):
        self.config = config
        self.buffer = []
        self.batch_size = config.get('batch_size', 100)
        self.flush_interval = config.get('flush_interval', 5)

    async def start(self):
        """启动聚合器"""
        asyncio.create_task(self._flush_loop())

    async def add_log(self, log_entry: Dict):
        """添加日志条目"""
        self.buffer.append(log_entry)

        if len(self.buffer) >= self.batch_size:
            await self.flush()

    async def flush(self):
        """刷新缓冲区"""
        if not self.buffer:
            return

        batch = self.buffer.copy()
        self.buffer.clear()

        await self._send_batch(batch)

    async def _flush_loop(self):
        """定时刷新循环"""
        while True:
            await asyncio.sleep(self.flush_interval)
            await self.flush()

    async def _send_batch(self, batch: List[Dict]):
        """发送日志批次"""
        # 发送到日志存储系统（如Elasticsearch、Loki等）
        pass

# 日志查询和分析
class LogAnalyzer:
    """日志分析器"""

    def __init__(self, log_storage):
        self.log_storage = log_storage

    async def search_logs(self, query: Dict) -> List[Dict]:
        """搜索日志"""
        return await self.log_storage.search(query)

    async def get_error_summary(self, time_range: timedelta) -> Dict:
        """获取错误摘要"""
        end_time = datetime.utcnow()
        start_time = end_time - time_range

        query = {
            "timestamp": {"$gte": start_time, "$lte": end_time},
            "level": "error"
        }

        errors = await self.search_logs(query)

        # 按错误类型分组
        error_counts = {}
        for error in errors:
            error_type = error.get('error_type', 'Unknown')
            error_counts[error_type] = error_counts.get(error_type, 0) + 1

        return {
            "total_errors": len(errors),
            "error_types": error_counts,
            "time_range": f"{start_time} to {end_time}"
        }

    async def analyze_performance(self, time_range: timedelta) -> Dict:
        """分析性能"""
        # 查询订单相关日志
        end_time = datetime.utcnow()
        start_time = end_time - time_range

        # 分析订单延迟
        order_logs = await self.search_logs({
            "event": "order_submitted",
            "timestamp": {"$gte": start_time, "$lte": end_time}
        })

        fill_logs = await self.search_logs({
            "event": "order_filled",
            "timestamp": {"$gte": start_time, "$lte": end_time}
        })

        # 计算平均延迟
        latencies = []
        for order_log in order_logs:
            order_id = order_log.get('order_id')
            # 查找对应的成交日志
            fill_log = next((f for f in fill_logs if f.get('order_id') == order_id), None)
            if fill_log:
                submit_time = order_log.get('timestamp')
                fill_time = fill_log.get('timestamp')
                if submit_time and fill_time:
                    latency = (fill_time - submit_time).total_seconds()
                    latencies.append(latency)

        if latencies:
            return {
                "avg_latency_seconds": sum(latencies) / len(latencies),
                "p50_latency": sorted(latencies)[len(latencies)//2],
                "p95_latency": sorted(latencies)[int(len(latencies)*0.95)],
                "total_orders": len(order_logs),
                "filled_orders": len(fill_logs)
            }
        else:
            return {"message": "No performance data available"}
```

## 13.5 数据备份和恢复

### 13.5.1 备份策略

```python
import os
import shutil
import gzip
import boto3
from datetime import datetime, timedelta
from pathlib import Path

class BackupManager:
    """备份管理器"""

    def __init__(self, config: dict):
        self.config = config
        self.backup_dir = Path(config.get('backup_dir', './backups'))
        self.retention_days = config.get('retention_days', 30)
        self.compression = config.get('compression', True)

    async def create_backup(self, data_source: str) -> str:
        """创建备份"""
        timestamp = datetime.utcnow().strftime('%Y%m%d_%H%M%S')
        backup_name = f"{data_source}_{timestamp}"

        if self.compression:
            backup_path = await self._create_compressed_backup(data_source, backup_name)
        else:
            backup_path = await self._create_regular_backup(data_source, backup_name)

        # 上传到云存储
        if self.config.get('cloud_storage'):
            await self._upload_to_cloud(backup_path)

        return backup_path

    async def _create_compressed_backup(self, source: str, backup_name: str) -> str:
        """创建压缩备份"""
        source_path = Path(source)
        backup_path = self.backup_dir / f"{backup_name}.tar.gz"

        # 创建tar.gz文件
        shutil.make_archive(
            str(backup_path.with_suffix('')),
            'gztar',
            source_path.parent,
            source_path.name
        )

        return str(backup_path)

    async def _upload_to_cloud(self, backup_path: str):
        """上传到云存储"""
        cloud_config = self.config['cloud_storage']

        if cloud_config['provider'] == 'aws':
            await self._upload_to_s3(backup_path, cloud_config)
        elif cloud_config['provider'] == 'azure':
            await self._upload_to_azure(backup_path, cloud_config)
        elif cloud_config['provider'] == 'gcp':
            await self._upload_to_gcs(backup_path, cloud_config)

    async def _upload_to_s3(self, backup_path: str, config: dict):
        """上传到S3"""
        s3 = boto3.client('s3')
        bucket = config['bucket']
        key = f"backups/{os.path.basename(backup_path)}"

        s3.upload_file(backup_path, bucket, key)

    async def restore_backup(self, backup_name: str, target_path: str):
        """恢复备份"""
        # 从本地或云下载备份
        backup_path = await self._get_backup_file(backup_name)

        # 解压到目标路径
        if backup_path.endswith('.tar.gz'):
            shutil.unpack_archive(backup_path, target_path)
        else:
            shutil.copy2(backup_path, target_path)

    async def _get_backup_file(self, backup_name: str) -> str:
        """获取备份文件"""
        local_path = self.backup_dir / backup_name

        if local_path.exists():
            return str(local_path)
        else:
            # 从云存储下载
            return await self._download_from_cloud(backup_name)

    async def cleanup_old_backups(self):
        """清理旧备份"""
        cutoff_date = datetime.utcnow() - timedelta(days=self.retention_days)

        for backup_file in self.backup_dir.glob('*'):
            if backup_file.stat().st_mtime < cutoff_date.timestamp():
                backup_file.unlink()

        # 清理云存储中的旧备份
        if self.config.get('cloud_storage'):
            await self._cleanup_cloud_backups()

# 热备份（运行时备份）
class HotBackup:
    """热备份系统"""

    def __init__(self, trading_engine):
        self.trading_engine = trading_engine
        self.backup_interval = 300  # 5分钟
        self.running = False

    async def start(self):
        """启动热备份"""
        self.running = True
        asyncio.create_task(self._backup_loop())

    async def stop(self):
        """停止热备份"""
        self.running = False

    async def _backup_loop(self):
        """备份循环"""
        while self.running:
            await self._create_state_snapshot()
            await asyncio.sleep(self.backup_interval)

    async def _create_state_snapshot(self):
        """创建状态快照"""
        snapshot = {
            'timestamp': datetime.utcnow().isoformat(),
            'positions': self.trading_engine.get_positions_snapshot(),
            'orders': self.trading_engine.get_orders_snapshot(),
            'strategies': self.trading_engine.get_strategies_snapshot(),
            'risk_metrics': self.trading_engine.get_risk_metrics_snapshot()
        }

        # 保存快照
        snapshot_path = f"./backups/state_snapshot_{int(time.time())}.json"
        with open(snapshot_path, 'w') as f:
            json.dump(snapshot, f, indent=2)

# 灾难恢复
class DisasterRecovery:
    """灾难恢复系统"""

    def __init__(self, config: dict):
        self.config = config
        self.backup_regions = config.get('backup_regions', [])
        self.recovery_plan = RecoveryPlan()

    async def failover_to_backup(self):
        """故障转移到备份"""
        logging.critical("Initiating disaster recovery failover")

        try:
            # 1. 停止主系统
            await self._shutdown_primary_system()

            # 2. 启动备份系统
            await self._activate_backup_system()

            # 3. 恢复数据
            await self._restore_from_backup()

            # 4. 验证系统状态
            await self._verify_system_health()

            logging.info("Disaster recovery completed successfully")

        except Exception as e:
            logging.critical(f"Disaster recovery failed: {e}")
            raise

    async def _shutdown_primary_system(self):
        """关闭主系统"""
        # 优雅关闭所有连接
        # 停止订单提交
        # 保存当前状态
        pass

    async def _activate_backup_system(self):
        """激活备份系统"""
        # 启动备用实例
        # 切换流量
        pass

    async def _restore_from_backup(self):
        """从备份恢复"""
        # 从最近的备份恢复数据
        # 恢复订单状态
        # 恢复持仓信息
        pass

    async def _verify_system_health(self):
        """验证系统健康"""
        # 检查所有组件状态
        # 验证数据完整性
        # 确认风险控制正常
        pass

# 恢复计划
class RecoveryPlan:
    """恢复计划"""

    def __init__(self):
        self.steps = []
        self.checkpoints = {}

    def add_step(self, step_name: str, step_func, timeout: float = 60.0):
        """添加恢复步骤"""
        self.steps.append({
            'name': step_name,
            'func': step_func,
            'timeout': timeout
        })

    def add_checkpoint(self, checkpoint_name: str):
        """添加检查点"""
        self.checkpoints[checkpoint_name] = len(self.steps)

    async def execute(self):
        """执行恢复计划"""
        start_time = time.time()

        for i, step in enumerate(self.steps):
            try:
                logging.info(f"Executing recovery step: {step['name']}")

                # 执行步骤，带超时控制
                await asyncio.wait_for(
                    step['func'](),
                    timeout=step['timeout']
                )

                logging.info(f"Step {step['name']} completed")

            except asyncio.TimeoutError:
                logging.error(f"Step {step['name']} timed out")
                raise
            except Exception as e:
                logging.error(f"Step {step['name']} failed: {e}")
                raise

        duration = time.time() - start_time
        logging.info(f"Recovery plan completed in {duration:.2f} seconds")
```

## 13.6 安全管理

### 13.6.1 API密钥管理

```python
import keyring
from cryptography.fernet import Fernet
from typing import Dict, Optional

class SecureKeyManager:
    """安全的密钥管理器"""

    def __init__(self, service_name: str = "nautilus_trader"):
        self.service_name = service_name
        self.cipher_suite = Fernet(self._get_or_create_key())

    def _get_or_create_key(self) -> bytes:
        """获取或创建加密密钥"""
        key = keyring.get_password(self.service_name, "encryption_key")
        if not key:
            key = Fernet.generate_key().decode()
            keyring.set_password(self.service_name, "encryption_key", key)
        return key.encode()

    def store_api_keys(self, exchange: str, api_key: str, secret_key: str):
        """存储API密钥"""
        encrypted_api_key = self.cipher_suite.encrypt(api_key.encode())
        encrypted_secret = self.cipher_suite.encrypt(secret_key.encode())

        keyring.set_password(f"{self.service_name}_{exchange}", "api_key", encrypted_api_key.decode())
        keyring.set_password(f"{self.service_name}_{exchange}", "secret_key", encrypted_secret.decode())

    def get_api_keys(self, exchange: str) -> Optional[Dict[str, str]]:
        """获取API密钥"""
        try:
            encrypted_api_key = keyring.get_password(f"{self.service_name}_{exchange}", "api_key")
            encrypted_secret = keyring.get_password(f"{self.service_name}_{exchange}", "secret_key")

            if encrypted_api_key and encrypted_secret:
                api_key = self.cipher_suite.decrypt(encrypted_api_key.encode()).decode()
                secret_key = self.cipher_suite.decrypt(encrypted_secret.encode()).decode()
                return {'api_key': api_key, 'secret_key': secret_key}
        except Exception as e:
            logging.error(f"Failed to retrieve API keys for {exchange}: {e}")

        return None

    def rotate_keys(self, exchange: str, new_api_key: str, new_secret_key: str):
        """轮换密钥"""
        # 删除旧密钥
        keyring.delete_password(f"{self.service_name}_{exchange}", "api_key")
        keyring.delete_password(f"{self.service_name}_{exchange}", "secret_key")

        # 存储新密钥
        self.store_api_keys(exchange, new_api_key, new_secret_key)

# 访问控制
class AccessControl:
    """访问控制管理"""

    def __init__(self):
        self.roles = {}
        self.permissions = {}

    def define_role(self, role_name: str, permissions: List[str]):
        """定义角色"""
        self.roles[role_name] = permissions

    def assign_role(self, user: str, role: str):
        """分配角色"""
        if user not in self.permissions:
            self.permissions[user] = []
        self.permissions[user].extend(self.roles[role])

    def check_permission(self, user: str, permission: str) -> bool:
        """检查权限"""
        return permission in self.permissions.get(user, [])

    def add_permission(self, user: str, permission: str):
        """添加权限"""
        if user not in self.permissions:
            self.permissions[user] = []
        self.permissions[user].append(permission)

# 审计日志
class AuditLogger:
    """审计日志记录器"""

    def __init__(self, log_file: str):
        self.log_file = log_file

    def log_action(self, user: str, action: str, resource: str, result: str = "success"):
        """记录操作"""
        entry = {
            'timestamp': datetime.utcnow().isoformat(),
            'user': user,
            'action': action,
            'resource': resource,
            'result': result,
            'ip': self._get_client_ip()
        }

        with open(self.log_file, 'a') as f:
            json.dump(entry, f)
            f.write('\n')

    def _get_client_ip(self) -> str:
        """获取客户端IP"""
        # 实现IP获取逻辑
        return "unknown"
```

## 小结

本章详细介绍了NautilusTrader实盘交易系统的架构设计和部署实践，涵盖了：

1. **系统架构设计**
   - 组件化架构和微服务设计
   - 容错和故障转移机制
   - 高可用性设计

2. **生产环境部署**
   - Docker容器化
   - Kubernetes编排
   - CI/CD流水线

3. **监控和告警**
   - 全方位监控体系
   - 智能告警系统
   - 性能指标采集

4. **日志管理**
   - 结构化日志
   - 日志聚合和分析
   - 集中式日志存储

5. **数据备份和恢复**
   - 多层备份策略
   - 热备份机制
   - 灾难恢复方案

6. **安全管理**
   - API密钥安全存储
   - 访问控制
   - 审计日志

通过本章的学习，读者能够构建稳定、安全、高性能的实盘交易系统，确保交易业务的安全可靠运行。

## 参考资料

- [Designing Data-Intensive Applications](https://dataintensive.net/)
- [Building Microservices](https://www.oreilly.com/library/view/building-microservices/9781491950340/)
- [Kubernetes: Up and Running](https://www.oreilly.com/library/view/kubernetes-up-and/9781492046530/)
- [Prometheus: Up & Running](https://www.oreilly.com/library/view/prometheus-up-running/9781492034131/)
## 本章可运行代码入口（源码版）

- 入口脚本：`docs/tutorials_cn/code/chapter13/run_live_trading.py`
- 关键示例：`docs/tutorials_cn/code/chapter13/live_gateway.py`、`docs/tutorials_cn/code/chapter13/live_monitor.py`

依赖策略说明：本章为可重放的实盘架构模拟，不依赖外部交易所连接即可完整演示 config → gateway → monitor 流程。

已验证命令：

```bash
uv run --python 3.13 python docs/tutorials_cn/code/chapter13/run_live_trading.py
```
