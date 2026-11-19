---
title: "Chapter 13: Live Trading Architecture"
---

# Chapter 13: Live Trading Architecture

## Learning Objectives

- Understand the core architectural components of a live trading system
- Master best practices for production environment deployment
- Learn to build high-availability, low-latency trading infrastructure
- Implement comprehensive monitoring, alerting, and failure recovery mechanisms

## 13.1 Live Trading System Architecture

### 13.1.1 System Architecture Overview

```python
from dataclasses import dataclass
from typing import Dict, List, Optional, Protocol
from enum import Enum
import asyncio
import logging
from datetime import datetime

# System component types
class ComponentType(Enum):
    """System component type enumeration"""
    STRATEGY = "strategy"           # Strategy component
    EXECUTION = "execution"         # Execution component
    RISK = "risk"                  # Risk management
    DATA = "data"                  # Data service
    MONITOR = "monitor"            # Monitoring component
    GATEWAY = "gateway"            # Trading gateway
    PERSISTENCE = "persistence"    # Persistence layer

# System states
class SystemState(Enum):
    """System state enumeration"""
    INITIALIZING = "initializing"   # Initializing
    RUNNING = "running"            # Running
    DEGRADED = "degraded"          # Degraded operation
    STOPPED = "stopped"            # Stopped
    ERROR = "error"                # Error state
    RECOVERING = "recovering"       # Recovering

@dataclass
class ComponentHealth:
    """Component health status"""
    component_id: str
    component_type: ComponentType
    status: SystemState
    last_heartbeat: datetime
    cpu_usage: float = 0.0
    memory_usage: float = 0.0
    latency_ms: float = 0.0
    error_count: int = 0
    message: str = ""

# System architecture manager
class SystemArchitecture:
    """Live trading system architecture manager"""

    def __init__(self, config: dict):
        self.config = config
        self.components = {}
        self.health_monitor = HealthMonitor()
        self.recovery_manager = RecoveryManager()
        self.load_balancer = LoadBalancer()
        self.circuit_breaker = CircuitBreaker()
        self.logger = logging.getLogger(__name__)

        # Initialize core components
        self._initialize_components()

    def _initialize_components(self):
        """Initialize system components"""
        # Strategy component cluster
        strategy_cluster = ComponentCluster(
            component_type=ComponentType.STRATEGY,
            instances=self.config.get('strategy_instances', 3),
            config=self.config.get('strategy_config', {})
        )
        self.components['strategy'] = strategy_cluster

        # Execution engine
        execution_engine = ExecutionEngine(
            config=self.config.get('execution_config', {})
        )
        self.components['execution'] = execution_engine

        # Risk management
        risk_manager = RiskManager(
            config=self.config.get('risk_config', {})
        )
        self.components['risk'] = risk_manager

        # Data service
        data_service = DataService(
            config=self.config.get('data_config', {})
        )
        self.components['data'] = data_service

        # Trading gateway
        trading_gateway = TradingGateway(
            exchanges=self.config.get('exchanges', []),
            config=self.config.get('gateway_config', {})
        )
        self.components['gateway'] = trading_gateway

        # Monitoring system
        monitoring_system = MonitoringSystem(
            config=self.config.get('monitoring_config', {})
        )
        self.components['monitor'] = monitoring_system

    async def start_system(self):
        """Start the system"""
        self.logger.info("Starting live trading system...")

        try:
            # Start components in dependency order
            startup_order = [
                'persistence',  # Persistence layer
                'data',         # Data service
                'gateway',      # Trading gateway
                'risk',         # Risk management
                'execution',    # Execution engine
                'strategy',     # Strategy components
                'monitor'       # Monitoring system
            ]

            for component_name in startup_order:
                if component_name in self.components:
                    await self._start_component(component_name)

            # Start health monitoring
            await self.health_monitor.start()

            # Start failure recovery
            await self.recovery_manager.start()

            self.logger.info("System started successfully")

        except Exception as e:
            self.logger.error(f"Failed to start system: {e}")
            await self.shutdown_system()
            raise

    async def _start_component(self, component_name: str):
        """Start a single component"""
        component = self.components[component_name]
        try:
            if hasattr(component, 'start'):
                await component.start()
            self.logger.info(f"Component {component_name} started")
        except Exception as e:
            self.logger.error(f"Failed to start {component_name}: {e}")
            raise

    async def shutdown_system(self):
        """Shutdown the system"""
        self.logger.info("Shutting down system...")

        # Shutdown components in reverse order
        shutdown_order = [
            'strategy',     # Strategy components
            'execution',    # Execution engine
            'risk',         # Risk management
            'gateway',      # Trading gateway
            'data',         # Data service
            'monitor',      # Monitoring system
            'persistence'   # Persistence layer
        ]

        for component_name in shutdown_order:
            if component_name in self.components:
                await self._stop_component(component_name)

        # Shutdown health monitoring and failure recovery
        await self.health_monitor.stop()
        await self.recovery_manager.stop()

        self.logger.info("System shutdown complete")

    async def _stop_component(self, component_name: str):
        """Stop a single component"""
        component = self.components[component_name]
        try:
            if hasattr(component, 'stop'):
                await component.stop()
            self.logger.info(f"Component {component_name} stopped")
        except Exception as e:
            self.logger.error(f"Failed to stop {component_name}: {e}")

    async def get_system_status(self) -> dict:
        """Get system status"""
        status = {
            'overall_status': SystemState.RUNNING,
            'components': {},
            'timestamp': datetime.utcnow().isoformat()
        }

        for name, component in self.components.items():
            health = await self._get_component_health(component)
            status['components'][name] = health

            # Update overall status
            if health['status'] == SystemState.ERROR:
                status['overall_status'] = SystemState.ERROR
            elif health['status'] == SystemState.DEGRADED:
                status['overall_status'] = SystemState.DEGRADED

        return status

    async def _get_component_health(self, component) -> dict:
        """Get component health status"""
        if hasattr(component, 'get_health'):
            return await component.get_health()
        else:
            return {
                'status': SystemState.RUNNING,
                'message': 'Health check not implemented'
            }

# Component cluster management
class ComponentCluster:
    """Component cluster manager"""

    def __init__(self, component_type: ComponentType, instances: int, config: dict):
        self.component_type = component_type
        self.instances = instances
        self.config = config
        self.components = []
        self.load_balancer = RoundRobinLoadBalancer()
        self.health_checker = HealthChecker()

    async def start(self):
        """Start the cluster"""
        for i in range(self.instances):
            component = await self._create_component(i)
            await component.start()
            self.components.append(component)
            self.health_checker.add_component(component)

        await self.health_checker.start()

    async def stop(self):
        """Stop the cluster"""
        await self.health_checker.stop()
        for component in self.components:
            await component.stop()

    async def _create_component(self, instance_id: int):
        """Create component instance"""
        if self.component_type == ComponentType.STRATEGY:
            return StrategyInstance(instance_id, self.config)
        # Can extend to other component types
        raise NotImplementedError(f"Component type {self.component_type} not implemented")

    async def get_instance(self):
        """Get an available instance"""
        healthy_instances = await self.health_checker.get_healthy_instances()
        if not healthy_instances:
            raise Exception("No healthy instances available")

        # Use load balancer to select instance
        selected = self.load_balancer.select(healthy_instances)
        return selected
```

### 13.1.2 Fault Tolerance and Failover

```python
import random
import time
from typing import Set, Callable

# Health checker
class HealthChecker:
    """Component health checker"""

    def __init__(self, check_interval: float = 1.0):
        self.components = {}
        self.healthy_instances: Set = set()
        self.unhealthy_instances: Set = set()
        self.check_interval = check_interval
        self.running = False

    def add_component(self, component):
        """Add component to monitor"""
        component_id = id(component)
        self.components[component_id] = component
        self.healthy_instances.add(component_id)

    async def start(self):
        """Start health checking"""
        self.running = True
        asyncio.create_task(self._health_check_loop())

    async def stop(self):
        """Stop health checking"""
        self.running = False

    async def _health_check_loop(self):
        """Health check loop"""
        while self.running:
            await self._check_all_components()
            await asyncio.sleep(self.check_interval)

    async def _check_all_components(self):
        """Check all components"""
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
        """Check individual component health"""
        if hasattr(component, 'is_healthy'):
            return await component.is_healthy()
        else:
            # Default check: ping-pong
            try:
                await component.ping()
                return True
            except:
                return False

    async def get_healthy_instances(self):
        """Get list of healthy instances"""
        return [self.components[comp_id] for comp_id in self.healthy_instances]

# Failover manager
class FailoverManager:
    """Failover manager"""

    def __init__(self, primary_components: List, backup_components: List):
        self.primary_components = primary_components
        self.backup_components = backup_components
        self.active_component = None
        self.failover_count = 0
        self.recovery_count = 0

    async def initialize(self):
        """Initialize failover"""
        # Select primary component
        self.active_component = await self._select_primary()
        await self.active_component.activate()

    async def _select_primary(self):
        """Select primary component"""
        # Prefer available primary components
        for component in self.primary_components:
            if await component.is_available():
                return component

        # If primary components unavailable, select backup component
        for component in self.backup_components:
            if await component.is_available():
                return component

        raise Exception("No available components")

    async def handle_failure(self, failed_component):
        """Handle component failure"""
        logging.warning(f"Component failure detected: {failed_component}")

        if failed_component == self.active_component:
            await self._initiate_failover()

        self.failover_count += 1

    async def _initiate_failover(self):
        """Initiate failover"""
        logging.info("Initiating failover...")

        # Deactivate failed component
        try:
            await self.active_component.deactivate()
        except:
            pass

        # Select new component
        try:
            new_component = await self._select_backup()
            await new_component.activate()
            self.active_component = new_component
            logging.info(f"Failover completed to {new_component}")
        except Exception as e:
            logging.error(f"Failover failed: {e}")
            raise

    async def _select_backup(self):
        """Select backup component"""
        available_backups = [
            comp for comp in self.backup_components
            if await comp.is_available()
        ]

        if not available_backups:
            raise Exception("No backup components available")

        return random.choice(available_backups)

    async def attempt_recovery(self):
        """Attempt to recover failed components"""
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

# Circuit breaker pattern
class CircuitBreaker:
    """Circuit breaker implementation"""

    def __init__(self, failure_threshold: int = 5, timeout: float = 60.0):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN

    async def call(self, func: Callable, *args, **kwargs):
        """Call function through circuit breaker"""
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
        """Whether to attempt reset"""
        return (self.last_failure_time and
                time.time() - self.last_failure_time >= self.timeout)

    def _on_success(self):
        """Handle success"""
        self.failure_count = 0
        self.state = 'CLOSED'

    def _on_failure(self):
        """Handle failure"""
        self.failure_count += 1
        self.last_failure_time = time.time()

        if self.failure_count >= self.failure_threshold:
            self.state = 'OPEN'

# Auto-recovery manager
class RecoveryManager:
    """Auto-recovery manager"""

    def __init__(self):
        self.recovery_strategies = {}
        self.running = False

    def register_strategy(self, component_type: ComponentType, strategy):
        """Register recovery strategy"""
        self.recovery_strategies[component_type] = strategy

    async def start(self):
        """Start recovery manager"""
        self.running = True
        asyncio.create_task(self._recovery_loop())

    async def stop(self):
        """Stop recovery manager"""
        self.running = False

    async def _recovery_loop(self):
        """Recovery loop"""
        while self.running:
            # Check components needing recovery
            await self._check_and_recover()
            await asyncio.sleep(5.0)  # Check every 5 seconds

    async def _check_and_recover(self):
        """Check and execute recovery"""
        # Need to integrate with health checker
        # Get unhealthy components and attempt recovery
        pass

    async def recover_component(self, component):
        """Recover single component"""
        component_type = getattr(component, 'type', None)
        if component_type in self.recovery_strategies:
            strategy = self.recovery_strategies[component_type]
            await strategy.recover(component)

# Recovery strategy base class
class RecoveryStrategy(Protocol):
    """Recovery strategy protocol"""

    async def recover(self, component):
        """Execute recovery"""
        ...

# Concrete recovery strategy implementations
class RestartRecoveryStrategy:
    """Restart recovery strategy"""

    def __init__(self, max_attempts: int = 3):
        self.max_attempts = max_attempts

    async def recover(self, component):
        """Recover component via restart"""
        for attempt in range(self.max_attempts):
            try:
                logging.info(f"Attempting recovery restart {attempt + 1}/{self.max_attempts}")
                await component.restart()
                return True
            except Exception as e:
                logging.error(f"Recovery restart attempt {attempt + 1} failed: {e}")
                await asyncio.sleep(2 ** attempt)  # Exponential backoff

        return False

class ReloadConfigRecoveryStrategy:
    """Reload config recovery strategy"""

    async def recover(self, component):
        """Recover component by reloading config"""
        try:
            await component.reload_config()
            return True
        except Exception as e:
            logging.error(f"Config reload recovery failed: {e}")
            return False
```

## 13.2 Production Environment Deployment

### 13.2.1 Docker Containerization

```dockerfile
# Dockerfile
FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    curl \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Copy dependency files
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create non-root user
RUN useradd --create-home --shell /bin/bash nautilus
USER nautilus

# Expose ports
EXPOSE 8000 8001 8080

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python health_check.py

# Startup command
CMD ["python", "run_production.py"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Core trading service
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

  # Data service
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

  # Monitoring service
  monitoring:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    networks:
      - trading-network

  # Log aggregation
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - ./monitoring/loki.yml:/etc/loki/local-config.yaml:ro
      - loki_data:/loki
    networks:
      - trading-network

  # Visualization
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

  # Message queue
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - trading-network

  # Database
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

  # Load balancer
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

### 13.2.2 Kubernetes Deployment

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

## 13.3 Monitoring and Alerting

### 13.3.1 Monitoring System Architecture

```python
from typing import Dict, Any
from datetime import datetime, timedelta
import json
import asyncio

class MonitoringSystem:
    """Monitoring system"""

    def __init__(self, config: dict):
        self.config = config
        self.metrics_collector = MetricsCollector()
        self.alert_manager = AlertManager()
        self.dashboard = Dashboard()
        self.storage = MetricsStorage()

    async def start(self):
        """Start monitoring system"""
        await self.metrics_collector.start()
        await self.alert_manager.start()
        await self.dashboard.start()

    async def stop(self):
        """Stop monitoring system"""
        await self.metrics_collector.stop()
        await self.alert_manager.stop()
        await self.dashboard.stop()

# Metrics collector
class MetricsCollector:
    """Metrics collector"""

    def __init__(self):
        self.collectors = {}
        self.interval = 1.0  # Collection interval (seconds)
        self.running = False

    def register_collector(self, name: str, collector_func):
        """Register metrics collector"""
        self.collectors[name] = collector_func

    async def start(self):
        """Start collection"""
        self.running = True
        asyncio.create_task(self._collection_loop())

    async def stop(self):
        """Stop collection"""
        self.running = False

    async def _collection_loop(self):
        """Collection loop"""
        while self.running:
            metrics = {}
            for name, collector in self.collectors.items():
                try:
                    metrics[name] = await collector()
                except Exception as e:
                    logging.error(f"Failed to collect {name}: {e}")

            # Send to storage
            await self._store_metrics(metrics)

            await asyncio.sleep(self.interval)

    async def _store_metrics(self, metrics: dict):
        """Store metrics"""
        timestamp = datetime.utcnow()
        for name, value in metrics.items():
            await MetricsStorage.store(name, value, timestamp)

# Business metrics collector
class BusinessMetricsCollector:
    """Business metrics collector"""

    def __init__(self, trading_engine):
        self.trading_engine = trading_engine

    async def collect_trading_metrics(self) -> dict:
        """Collect trading metrics"""
        return {
            'total_orders': self.trading_engine.total_orders,
            'filled_orders': self.trading_engine.filled_orders,
            'total_volume': self.trading_engine.total_volume,
            'total_pnl': self.trading_engine.total_pnl,
            'open_positions': len(self.trading_engine.open_positions),
            'active_strategies': len(self.trading_engine.active_strategies)
        }

    async def collect_performance_metrics(self) -> dict:
        """Collect performance metrics"""
        return {
            'order_latency_p50': self.trading_engine.get_order_latency_percentile(50),
            'order_latency_p90': self.trading_engine.get_order_latency_percentile(90),
            'order_latency_p99': self.trading_engine.get_order_latency_percentile(99),
            'throughput_orders_per_sec': self.trading_engine.get_order_throughput(),
            'cpu_usage': psutil.cpu_percent(),
            'memory_usage': psutil.virtual_memory().percent
        }

    async def collect_risk_metrics(self) -> dict:
        """Collect risk metrics"""
        return {
            'portfolio_var': self.trading_engine.calculate_var(),
            'max_drawdown': self.trading_engine.calculate_max_drawdown(),
            'sharpe_ratio': self.trading_engine.calculate_sharpe_ratio(),
            'risk_utilization': self.trading_engine.get_risk_utilization(),
            'margin_usage': self.trading_engine.get_margin_usage()
        }

# Alert manager
class AlertManager:
    """Alert manager"""

    def __init__(self):
        self.rules = []
        self.notifiers = []
        self.alert_history = []

    def add_rule(self, rule):
        """Add alert rule"""
        self.rules.append(rule)

    def add_notifier(self, notifier):
        """Add notifier"""
        self.notifiers.append(notifier)

    async def start(self):
        """Start alert management"""
        asyncio.create_task(self._alert_loop())

    async def _alert_loop(self):
        """Alert check loop"""
        while True:
            for rule in self.rules:
                try:
                    if await rule.check():
                        await self._handle_alert(rule)
                except Exception as e:
                    logging.error(f"Alert rule check failed: {e}")

            await asyncio.sleep(5)

    async def _handle_alert(self, rule):
        """Handle alert"""
        alert = {
            'rule': rule.name,
            'message': rule.message,
            'severity': rule.severity,
            'timestamp': datetime.utcnow()
        }

        # Check if already sent
        if self._should_send_alert(alert):
            self.alert_history.append(alert)

            # Send notifications
            for notifier in self.notifiers:
                await notifier.send(alert)

    def _should_send_alert(self, alert: dict) -> bool:
        """Determine whether to send alert"""
        # Check cooldown period
        cooldown = timedelta(minutes=5)
        for historical_alert in reversed(self.alert_history[-10:]):
            if (historical_alert['rule'] == alert['rule'] and
                datetime.utcnow() - historical_alert['timestamp'] < cooldown):
                return False
        return True

# Alert rule
class AlertRule:
    """Alert rule"""

    def __init__(self, name: str, check_func, message: str, severity: str):
        self.name = name
        self.check_func = check_func
        self.message = message
        self.severity = severity

    async def check(self) -> bool:
        """Check alert condition"""
        return await self.check_func()

# Predefined alert rules
class TradingAlertRules:
    """Trading alert rules"""

    @staticmethod
    def high_latency_rule(threshold_ms: float = 100):
        """High latency alert rule"""
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
        """High error rate alert rule"""
        async def check():
            # Get error rate for last 5 minutes
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
        """Risk limit alert rule"""
        async def check():
            risk_usage = get_risk_utilization()
            return risk_usage > 0.9  # 90%

        return AlertRule(
            name="risk_limit",
            check_func=check,
            message="Risk utilization exceeds 90%",
            severity="critical"
        )

# Notifiers
class EmailNotifier:
    """Email notifier"""

    def __init__(self, smtp_config: dict):
        self.smtp_config = smtp_config

    async def send(self, alert: dict):
        """Send email alert"""
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
    """Slack notifier"""

    def __init__(self, webhook_url: str):
        self.webhook_url = webhook_url

    async def send(self, alert: dict):
        """Send Slack alert"""
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

## 13.4 Log Management

### 13.4.1 Structured Logging

```python
import structlog
import json
from typing import Any, Dict

# Configure structured logging
def configure_logging(log_level: str = "INFO"):
    """Configure structured logging"""

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
            structlog.processors.JSONRenderer()  # JSON format
        ],
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
        wrapper_class=structlog.stdlib.BoundLogger,
        cache_logger_on_first_use=True,
    )

# Trading logger
class TradingLogger:
    """Trading logger"""

    def __init__(self, strategy_id: str):
        self.logger = structlog.get_logger()
        self.strategy_id = strategy_id

    def log_order_submitted(self, order):
        """Log order submission"""
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
        """Log order fill"""
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
        """Log strategy signal"""
        self.logger.info(
            "strategy_signal",
            strategy_id=self.strategy_id,
            signal_type=signal_type,
            **signal_data
        )

    def log_risk_event(self, event_type: str, details: Dict):
        """Log risk event"""
        self.logger.warning(
            "risk_event",
            strategy_id=self.strategy_id,
            event_type=event_type,
            **details
        )

    def log_error(self, error: Exception, context: Dict = None):
        """Log error"""
        self.logger.error(
            "error_occurred",
            strategy_id=self.strategy_id,
            error_type=type(error).__name__,
            error_message=str(error),
            **(context or {})
        )

# Log aggregator
class LogAggregator:
    """Log aggregator"""

    def __init__(self, config: dict):
        self.config = config
        self.buffer = []
        self.batch_size = config.get('batch_size', 100)
        self.flush_interval = config.get('flush_interval', 5)

    async def start(self):
        """Start aggregator"""
        asyncio.create_task(self._flush_loop())

    async def add_log(self, log_entry: Dict):
        """Add log entry"""
        self.buffer.append(log_entry)

        if len(self.buffer) >= self.batch_size:
            await self.flush()

    async def flush(self):
        """Flush buffer"""
        if not self.buffer:
            return

        batch = self.buffer.copy()
        self.buffer.clear()

        await self._send_batch(batch)

    async def _flush_loop(self):
        """Timed flush loop"""
        while True:
            await asyncio.sleep(self.flush_interval)
            await self.flush()

    async def _send_batch(self, batch: List[Dict]):
        """Send log batch"""
        # Send to log storage system (e.g., Elasticsearch, Loki, etc.)
        pass

# Log query and analysis
class LogAnalyzer:
    """Log analyzer"""

    def __init__(self, log_storage):
        self.log_storage = log_storage

    async def search_logs(self, query: Dict) -> List[Dict]:
        """Search logs"""
        return await self.log_storage.search(query)

    async def get_error_summary(self, time_range: timedelta) -> Dict:
        """Get error summary"""
        end_time = datetime.utcnow()
        start_time = end_time - time_range

        query = {
            "timestamp": {"$gte": start_time, "$lte": end_time},
            "level": "error"
        }

        errors = await self.search_logs(query)

        # Group by error type
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
        """Analyze performance"""
        # Query order-related logs
        end_time = datetime.utcnow()
        start_time = end_time - time_range

        # Analyze order latency
        order_logs = await self.search_logs({
            "event": "order_submitted",
            "timestamp": {"$gte": start_time, "$lte": end_time}
        })

        fill_logs = await self.search_logs({
            "event": "order_filled",
            "timestamp": {"$gte": start_time, "$lte": end_time}
        })

        # Calculate average latency
        latencies = []
        for order_log in order_logs:
            order_id = order_log.get('order_id')
            # Find corresponding fill log
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

## 13.5 Data Backup and Recovery

### 13.5.1 Backup Strategy

```python
import os
import shutil
import gzip
import boto3
from datetime import datetime, timedelta
from pathlib import Path

class BackupManager:
    """Backup manager"""

    def __init__(self, config: dict):
        self.config = config
        self.backup_dir = Path(config.get('backup_dir', './backups'))
        self.retention_days = config.get('retention_days', 30)
        self.compression = config.get('compression', True)

    async def create_backup(self, data_source: str) -> str:
        """Create backup"""
        timestamp = datetime.utcnow().strftime('%Y%m%d_%H%M%S')
        backup_name = f"{data_source}_{timestamp}"

        if self.compression:
            backup_path = await self._create_compressed_backup(data_source, backup_name)
        else:
            backup_path = await self._create_regular_backup(data_source, backup_name)

        # Upload to cloud storage
        if self.config.get('cloud_storage'):
            await self._upload_to_cloud(backup_path)

        return backup_path

    async def _create_compressed_backup(self, source: str, backup_name: str) -> str:
        """Create compressed backup"""
        source_path = Path(source)
        backup_path = self.backup_dir / f"{backup_name}.tar.gz"

        # Create tar.gz file
        shutil.make_archive(
            str(backup_path.with_suffix('')),
            'gztar',
            source_path.parent,
            source_path.name
        )

        return str(backup_path)

    async def _upload_to_cloud(self, backup_path: str):
        """Upload to cloud storage"""
        cloud_config = self.config['cloud_storage']

        if cloud_config['provider'] == 'aws':
            await self._upload_to_s3(backup_path, cloud_config)
        elif cloud_config['provider'] == 'azure':
            await self._upload_to_azure(backup_path, cloud_config)
        elif cloud_config['provider'] == 'gcp':
            await self._upload_to_gcs(backup_path, cloud_config)

    async def _upload_to_s3(self, backup_path: str, config: dict):
        """Upload to S3"""
        s3 = boto3.client('s3')
        bucket = config['bucket']
        key = f"backups/{os.path.basename(backup_path)}"

        s3.upload_file(backup_path, bucket, key)

    async def restore_backup(self, backup_name: str, target_path: str):
        """Restore backup"""
        # Download backup from local or cloud
        backup_path = await self._get_backup_file(backup_name)

        # Extract to target path
        if backup_path.endswith('.tar.gz'):
            shutil.unpack_archive(backup_path, target_path)
        else:
            shutil.copy2(backup_path, target_path)

    async def _get_backup_file(self, backup_name: str) -> str:
        """Get backup file"""
        local_path = self.backup_dir / backup_name

        if local_path.exists():
            return str(local_path)
        else:
            # Download from cloud storage
            return await self._download_from_cloud(backup_name)

    async def cleanup_old_backups(self):
        """Clean up old backups"""
        cutoff_date = datetime.utcnow() - timedelta(days=self.retention_days)

        for backup_file in self.backup_dir.glob('*'):
            if backup_file.stat().st_mtime < cutoff_date.timestamp():
                backup_file.unlink()

        # Clean up old backups in cloud storage
        if self.config.get('cloud_storage'):
            await self._cleanup_cloud_backups()

# Hot backup (runtime backup)
class HotBackup:
    """Hot backup system"""

    def __init__(self, trading_engine):
        self.trading_engine = trading_engine
        self.backup_interval = 300  # 5 minutes
        self.running = False

    async def start(self):
        """Start hot backup"""
        self.running = True
        asyncio.create_task(self._backup_loop())

    async def stop(self):
        """Stop hot backup"""
        self.running = False

    async def _backup_loop(self):
        """Backup loop"""
        while self.running:
            await self._create_state_snapshot()
            await asyncio.sleep(self.backup_interval)

    async def _create_state_snapshot(self):
        """Create state snapshot"""
        snapshot = {
            'timestamp': datetime.utcnow().isoformat(),
            'positions': self.trading_engine.get_positions_snapshot(),
            'orders': self.trading_engine.get_orders_snapshot(),
            'strategies': self.trading_engine.get_strategies_snapshot(),
            'risk_metrics': self.trading_engine.get_risk_metrics_snapshot()
        }

        # Save snapshot
        snapshot_path = f"./backups/state_snapshot_{int(time.time())}.json"
        with open(snapshot_path, 'w') as f:
            json.dump(snapshot, f, indent=2)

# Disaster recovery
class DisasterRecovery:
    """Disaster recovery system"""

    def __init__(self, config: dict):
        self.config = config
        self.backup_regions = config.get('backup_regions', [])
        self.recovery_plan = RecoveryPlan()

    async def failover_to_backup(self):
        """Failover to backup"""
        logging.critical("Initiating disaster recovery failover")

        try:
            # 1. Shutdown primary system
            await self._shutdown_primary_system()

            # 2. Activate backup system
            await self._activate_backup_system()

            # 3. Restore data
            await self._restore_from_backup()

            # 4. Verify system health
            await self._verify_system_health()

            logging.info("Disaster recovery completed successfully")

        except Exception as e:
            logging.critical(f"Disaster recovery failed: {e}")
            raise

    async def _shutdown_primary_system(self):
        """Shutdown primary system"""
        # Gracefully close all connections
        # Stop order submission
        # Save current state
        pass

    async def _activate_backup_system(self):
        """Activate backup system"""
        # Start backup instances
        # Switch traffic
        pass

    async def _restore_from_backup(self):
        """Restore from backup"""
        # Restore data from latest backup
        # Restore order status
        # Restore position information
        pass

    async def _verify_system_health(self):
        """Verify system health"""
        # Check all component statuses
        # Verify data integrity
        # Confirm risk controls are normal
        pass

# Recovery plan
class RecoveryPlan:
    """Recovery plan"""

    def __init__(self):
        self.steps = []
        self.checkpoints = {}

    def add_step(self, step_name: str, step_func, timeout: float = 60.0):
        """Add recovery step"""
        self.steps.append({
            'name': step_name,
            'func': step_func,
            'timeout': timeout
        })

    def add_checkpoint(self, checkpoint_name: str):
        """Add checkpoint"""
        self.checkpoints[checkpoint_name] = len(self.steps)

    async def execute(self):
        """Execute recovery plan"""
        start_time = time.time()

        for i, step in enumerate(self.steps):
            try:
                logging.info(f"Executing recovery step: {step['name']}")

                # Execute step with timeout control
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

## 13.6 Security Management

### 13.6.1 API Key Management

```python
import keyring
from cryptography.fernet import Fernet
from typing import Dict, Optional

class SecureKeyManager:
    """Secure key manager"""

    def __init__(self, service_name: str = "nautilus_trader"):
        self.service_name = service_name
        self.cipher_suite = Fernet(self._get_or_create_key())

    def _get_or_create_key(self) -> bytes:
        """Get or create encryption key"""
        key = keyring.get_password(self.service_name, "encryption_key")
        if not key:
            key = Fernet.generate_key().decode()
            keyring.set_password(self.service_name, "encryption_key", key)
        return key.encode()

    def store_api_keys(self, exchange: str, api_key: str, secret_key: str):
        """Store API keys"""
        encrypted_api_key = self.cipher_suite.encrypt(api_key.encode())
        encrypted_secret = self.cipher_suite.encrypt(secret_key.encode())

        keyring.set_password(f"{self.service_name}_{exchange}", "api_key", encrypted_api_key.decode())
        keyring.set_password(f"{self.service_name}_{exchange}", "secret_key", encrypted_secret.decode())

    def get_api_keys(self, exchange: str) -> Optional[Dict[str, str]]:
        """Get API keys"""
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
        """Rotate keys"""
        # Delete old keys
        keyring.delete_password(f"{self.service_name}_{exchange}", "api_key")
        keyring.delete_password(f"{self.service_name}_{exchange}", "secret_key")

        # Store new keys
        self.store_api_keys(exchange, new_api_key, new_secret_key)

# Access control
class AccessControl:
    """Access control management"""

    def __init__(self):
        self.roles = {}
        self.permissions = {}

    def define_role(self, role_name: str, permissions: List[str]):
        """Define role"""
        self.roles[role_name] = permissions

    def assign_role(self, user: str, role: str):
        """Assign role"""
        if user not in self.permissions:
            self.permissions[user] = []
        self.permissions[user].extend(self.roles[role])

    def check_permission(self, user: str, permission: str) -> bool:
        """Check permission"""
        return permission in self.permissions.get(user, [])

    def add_permission(self, user: str, permission: str):
        """Add permission"""
        if user not in self.permissions:
            self.permissions[user] = []
        self.permissions[user].append(permission)

# Audit log
class AuditLogger:
    """Audit logger"""

    def __init__(self, log_file: str):
        self.log_file = log_file

    def log_action(self, user: str, action: str, resource: str, result: str = "success"):
        """Log action"""
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
        """Get client IP"""
        # Implement IP retrieval logic
        return "unknown"
```

## Summary

This chapter provides a detailed introduction to the architectural design and deployment practices of the NautilusTrader live trading system, covering:

1. **System Architecture Design**
   - Component-based architecture and microservices design
   - Fault tolerance and failover mechanisms
   - High availability design

2. **Production Environment Deployment**
   - Docker containerization
   - Kubernetes orchestration
   - CI/CD pipelines

3. **Monitoring and Alerting**
   - Comprehensive monitoring system
   - Intelligent alerting system
   - Performance metrics collection

4. **Log Management**
   - Structured logging
   - Log aggregation and analysis
   - Centralized log storage

5. **Data Backup and Recovery**
   - Multi-layer backup strategy
   - Hot backup mechanism
   - Disaster recovery solution

6. **Security Management**
   - Secure API key storage
   - Access control
   - Audit logging

Through this chapter, readers can build stable, secure, high-performance live trading systems, ensuring the safe and reliable operation of trading business.

## References

- [Designing Data-Intensive Applications](https://dataintensive.net/)
- [Building Microservices](https://www.oreilly.com/library/view/building-microservices/9781491950340/)
- [Kubernetes: Up and Running](https://www.oreilly.com/library/view/kubernetes-up-and/9781492046530/)
- [Prometheus: Up & Running](https://www.oreilly.com/library/view/prometheus-up-running/9781492034131/)

## Runnable Code Entry (Source Build)

- Entry script: docs/tutorials_cn/code/chapter13/run_live_trading.py
- Key examples: docs/tutorials_cn/code/chapter13/live_gateway.py, docs/tutorials_cn/code/chapter13/live_monitor.py

Dependency strategy: This chapter is a replayable live-architecture simulation and runs without external exchange connectivity.

Verified command:

```bash
uv run --python 3.13 python docs/tutorials_cn/code/chapter13/run_live_trading.py
```
