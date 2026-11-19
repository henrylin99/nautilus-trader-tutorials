---
title: "Chapter 19: Complete Project Example 1 - Cryptocurrency Market Making Strategy"
---

# Chapter 19: Complete Project Example 1 - Cryptocurrency Market Making Strategy

## Learning Objectives

- Integrate knowledge from previous chapters through a complete project practice
- Build a production-grade cryptocurrency market making trading system
- Learn to handle various challenges in real trading
- Understand the complete process from strategy development to deployment

## 19.1 Project Overview

### 19.1.1 Project Goals

This project will build a cryptocurrency market making trading system with the following features:

- **Multi-exchange support**: Simultaneously support mainstream exchanges like Binance, OKX, Bybit
- **Dynamic market making**: Dynamically adjust quotes based on market volatility and liquidity
- **Risk management**: Built-in multi-layer risk control mechanisms
- **Real-time monitoring**: Complete performance monitoring and alert system
- **High availability**: Support hot backup and automatic failover

### 19.1.2 System Architecture

```mermaid
graph TB
    subgraph "Data Layer"
        A[Market Data] --> B[Data Processor]
        B --> C[Feature Engineering]
    end

    subgraph "Strategy Layer"
        C --> D[Market Making Strategy]
        D --> E[Risk Management]
    end

    subgraph "Execution Layer"
        E --> F[Order Manager]
        F --> G[Execution Engine]
    end

    subgraph "Exchange Layer"
        G --> H[Binance]
        G --> I[OKX]
        G --> J[Bybit]
    end

    subgraph "Monitoring Layer"
        K[Monitoring System] --> L[Alert System]
        K --> M[Performance Analysis]
    end
```

## 19.2 Configuration System

### 19.2.1 Configuration File Structure

```yaml
# config/production.yaml
environment: production
logging:
  level: INFO
  file: logs/market_maker.log

# Data source configuration
data_sources:
  primary:
    exchange: binance
    symbols:
      - BTCUSDT
      - ETHUSDT
    channels:
      - depth
      - trades
      - ticker

  backup:
    exchange: okx
    symbols:
      - BTC-USDT
      - ETH-USDT

# Strategy configuration
strategy:
  name: DynamicMarketMaker
  parameters:
    base_spread_bps: 10          # Base spread (basis points)
    inventory_target: 0.0        # Target inventory ratio
    max_position: 50.0           # Maximum position (BTC)
    order_levels: 5              # Number of quote levels
    order_size_base: 0.1         # Base order size (BTC)
    skew_factor: 0.5             # Inventory skew factor
    volatility_window: 100       # Volatility calculation window
    quote_refresh_rate: 1.0      # Quote refresh rate (seconds)

# Risk management configuration
risk:
  max_daily_loss: 10000         # Maximum daily loss (USDT)
  max_drawdown: 0.05            # Maximum drawdown (5%)
  position_limit: {
    BTC: 50.0
    ETH: 1000.0
  }
  leverage_limit: 3.0           # Leverage limit

# Execution configuration
execution:
  order_type: limit
  time_in_force: GTC
  max_orders_per_second: 10
  retry_attempts: 3
  order_timeout: 30

# Monitoring configuration
monitoring:
  metrics_interval: 5           # Metrics collection interval (seconds)
  health_check_interval: 30     # Health check interval (seconds)
  alert_channels:
    - email
    - slack

  performance_metrics:
    - pnl
    - inventory
    - order_fill_rate
    - latency
    - error_rate

# Database configuration
database:
  type: postgresql
  host: localhost
  port: 5432
  database: market_maker
  username: nautilus
  password: ${DB_PASSWORD}

# Redis configuration (cache)
redis:
  host: localhost
  port: 6379
  db: 0

# API configuration
api:
  enabled: true
  host: 0.0.0.0
  port: 8080
  authentication: true
```

### 19.2.2 Configuration Manager

```python
# src/config/config_manager.py
import os
import yaml
from typing import Dict, Any
from pathlib import Path
from dotenv import load_dotenv

class ConfigManager:
    """Configuration Manager"""

    def __init__(self, config_path: str = "config/production.yaml"):
        # Load environment variables
        load_dotenv()

        # Load configuration file
        self.config_path = Path(config_path)
        self.config = self._load_config()

        # Handle environment variable substitution
        self.config = self._substitute_env_vars(self.config)

    def _load_config(self) -> Dict[str, Any]:
        """Load configuration file"""
        with open(self.config_path, 'r') as f:
            return yaml.safe_load(f)

    def _substitute_env_vars(self, config: Any) -> Any:
        """Substitute environment variables"""
        if isinstance(config, dict):
            return {k: self._substitute_env_vars(v) for k, v in config.items()}
        elif isinstance(config, list):
            return [self._substitute_env_vars(item) for item in config]
        elif isinstance(config, str) and config.startswith('${') and config.endswith('}'):
            env_var = config[2:-1]
            default_value = None
            if ':' in env_var:
                env_var, default_value = env_var.split(':', 1)
            return os.getenv(env_var, default_value)
        else:
            return config

    def get(self, key: str, default=None):
        """Get configuration value"""
        keys = key.split('.')
        value = self.config

        for k in keys:
            if isinstance(value, dict) and k in value:
                value = value[k]
            else:
                return default

        return value

    def update(self, key: str, value: Any):
        """Update configuration value"""
        keys = key.split('.')
        config = self.config

        for k in keys[:-1]:
            if k not in config:
                config[k] = {}
            config = config[k]

        config[keys[-1]] = value

    def save(self):
        """Save configuration to file"""
        with open(self.config_path, 'w') as f:
            yaml.dump(self.config, f, default_flow_style=False)
```

## 19.3 Data Management Module

### 19.3.1 Unified Data Interface

```python
# src/data/unified_data_client.py
from typing import Dict, List, Callable, Optional
from dataclasses import dataclass
import asyncio
import aiohttp
import websockets
from decimal import Decimal

@dataclass
class MarketData:
    """Market data structure"""
    symbol: str
    exchange: str
    timestamp: int
    bid: Decimal
    ask: Decimal
    bid_size: Decimal
    ask_size: Decimal
    last_price: Optional[Decimal] = None
    volume: Optional[Decimal] = None

class UnifiedDataClient:
    """Unified data client"""

    def __init__(self, config_manager: ConfigManager):
        self.config = config_manager
        self.clients = {}  # Exchange clients
        self.data_cache = {}
        self.subscribers = {
            'quotes': [],
            'trades': [],
            'bars': []
        }

    async def initialize(self):
        """Initialize all data clients"""
        # Primary data source
        primary_config = self.config.get('data_sources.primary')
        await self._add_exchange_client(
            primary_config['exchange'],
            primary_config,
            is_primary=True
        )

        # Backup data source
        backup_config = self.config.get('data_sources.backup')
        if backup_config:
            await self._add_exchange_client(
                backup_config['exchange'],
                backup_config,
                is_primary=False
            )

    async def _add_exchange_client(self, exchange: str, config: dict, is_primary: bool):
        """Add exchange client"""
        if exchange == 'binance':
            client = BinanceDataClient(config)
        elif exchange == 'okx':
            client = OKXDataClient(config)
        elif exchange == 'bybit':
            client = BybitDataClient(config)
        else:
            raise ValueError(f"Unsupported exchange: {exchange}")

        await client.initialize()
        self.clients[exchange] = {
            'client': client,
            'is_primary': is_primary,
            'config': config
        }

        # Subscribe to data streams
        for symbol in config['symbols']:
            await self._subscribe_symbol(exchange, symbol, config['channels'])

    async def _subscribe_symbol(self, exchange: str, symbol: str, channels: List[str]):
        """Subscribe to symbol data"""
        client_info = self.clients[exchange]
        client = client_info['client']

        for channel in channels:
            if channel == 'depth':
                await client.subscribe_depth(symbol, self._on_depth_update)
            elif channel == 'trades':
                await client.subscribe_trades(symbol, self._on_trade_update)
            elif channel == 'ticker':
                await client.subscribe_ticker(symbol, self._on_ticker_update)

    async def _on_depth_update(self, data: MarketData):
        """Handle depth updates"""
        # Cache latest data
        cache_key = f"{data.symbol}_{data.exchange}"
        self.data_cache[cache_key] = data

        # Notify subscribers
        for callback in self.subscribers['quotes']:
            await callback(data)

    async def _on_trade_update(self, data: MarketData):
        """Handle trade updates"""
        for callback in self.subscribers['trades']:
            await callback(data)

    async def _on_ticker_update(self, data: MarketData):
        """Handle ticker updates"""
        # Update price information
        cache_key = f"{data.symbol}_{data.exchange}"
        if cache_key in self.data_cache:
            self.data_cache[cache_key].last_price = data.last_price
            self.data_cache[cache_key].volume = data.volume

    def subscribe_quotes(self, callback: Callable[[MarketData], None]):
        """Subscribe to quote data"""
        self.subscribers['quotes'].append(callback)

    def subscribe_trades(self, callback: Callable[[MarketData], None]):
        """Subscribe to trade data"""
        self.subscribers['trades'].append(callback)

    def get_latest_quote(self, symbol: str) -> Optional[MarketData]:
        """Get latest quote"""
        # Prefer primary data source
        for exchange, client_info in self.clients.items():
            if client_info['is_primary']:
                cache_key = f"{symbol}_{exchange}"
                if cache_key in self.data_cache:
                    return self.data_cache[cache_key]

        # If primary unavailable, return backup data source
        for exchange, client_info in self.clients.items():
            if not client_info['is_primary']:
                cache_key = f"{symbol}_{exchange}"
                if cache_key in self.data_cache:
                    return self.data_cache[cache_key]

        return None

# Exchange-specific clients
class BinanceDataClient:
    """Binance data client"""

    def __init__(self, config: dict):
        self.config = config
        self.ws_url = "wss://stream.binance.com:9443/ws"
        self.rest_url = "https://api.binance.com"
        self.websocket = None
        self.subscriptions = {}

    async def initialize(self):
        """Initialize connection"""
        # WebSocket connection
        self.websocket = await websockets.connect(self.ws_url)

        # Start message processing
        asyncio.create_task(self._message_loop())

    async def subscribe_depth(self, symbol: str, callback: Callable):
        """Subscribe to depth data"""
        stream = f"{symbol.lower()}@depth"
        await self.websocket.send(json.dumps({
            "method": "SUBSCRIBE",
            "params": [stream],
            "id": len(self.subscriptions)
        }))
        self.subscriptions[stream] = callback

    async def subscribe_trades(self, symbol: str, callback: Callable):
        """Subscribe to trade data"""
        stream = f"{symbol.lower()}@trade"
        await self.websocket.send(json.dumps({
            "method": "SUBSCRIBE",
            "params": [stream],
            "id": len(self.subscriptions)
        }))
        self.subscriptions[stream] = callback

    async def _message_loop(self):
        """Message processing loop"""
        async for message in self.websocket:
            data = json.loads(message)

            if 'stream' in data:
                stream = data['stream']
                if stream in self.subscriptions:
                    await self._process_stream_data(stream, data['data'])

    async def _process_stream_data(self, stream: str, data: dict):
        """Process stream data"""
        if 'depth' in stream:
            market_data = self._parse_depth_data(data)
        elif 'trade' in stream:
            market_data = self._parse_trade_data(data)
        else:
            return

        await self.subscriptions[stream](market_data)

    def _parse_depth_data(self, data: dict) -> MarketData:
        """Parse depth data"""
        return MarketData(
            symbol=data['s'],
            exchange='binance',
            timestamp=data['E'],
            bid=Decimal(str(data['bids'][0][0])),
            ask=Decimal(str(data['asks'][0][0])),
            bid_size=Decimal(str(data['bids'][0][1])),
            ask_size=Decimal(str(data['asks'][0][1]))
        )

    def _parse_trade_data(self, data: dict) -> MarketData:
        """Parse trade data"""
        return MarketData(
            symbol=data['s'],
            exchange='binance',
            timestamp=data['T'],
            bid=Decimal('0'),
            ask=Decimal('0'),
            bid_size=Decimal('0'),
            ask_size=Decimal('0'),
            last_price=Decimal(str(data['p'])),
            volume=Decimal(str(data['q']))
        )
```

## 19.4 Market Making Strategy Implementation

### 19.4.1 Core Strategy Class

```python
# src/strategies/market_maker.py
import asyncio
import numpy as np
from decimal import Decimal
from typing import Dict, List, Optional
from nautilus_trader.strategy import Strategy
from nautilus_trader.model.data import QuoteTick
from nautilus_trader.model.events import OrderFilled
from nautilus_trader.model.enums import OrderSide, OrderType
from nautilus_trader.model.objects import Quantity, Price
from nautilus_trader.adapters.betfair.providers import BetfairInstrumentProvider
from nautilus_trader.model.identifiers import InstrumentId

class DynamicMarketMaker(Strategy):
    """Dynamic market making strategy"""

    def __init__(self, config: dict):
        super().__init__()

        # Strategy parameters
        self.symbols = config['symbols']
        self.base_spread = Decimal(str(config['base_spread_bps'])) / Decimal('10000')
        self.max_position = Decimal(str(config['max_position']))
        self.order_levels = config['order_levels']
        self.order_size_base = Decimal(str(config['order_size_base']))
        self.skew_factor = config['skew_factor']
        self.volatility_window = config['volatility_window']
        self.quote_refresh_rate = config['quote_refresh_rate']

        # State variables
        self.positions = {symbol: Decimal('0') for symbol in self.symbols}
        self.mid_prices = {symbol: Decimal('0') for symbol in self.symbols}
        self.volatilities = {symbol: Decimal('0.01') for symbol in self.symbols}
        self.price_history = {symbol: [] for symbol in self.symbols}
        self.active_orders = {symbol: {} for symbol in self.symbols}

        # Performance statistics
        self.trade_count = 0
        self.total_pnl = Decimal('0')
        self.quote_updates = 0

    def on_start(self):
        """Strategy startup"""
        # Subscribe to quote data for all symbols
        for symbol in self.symbols:
            self.subscribe_quote_ticks(InstrumentId.from_str(symbol))

        # Start quote update task
        asyncio.create_task(self._quote_update_loop())

        # Start risk monitoring task
        asyncio.create_task(self._risk_monitor_loop())

        self.log.info(f"DynamicMarketMaker started for symbols: {self.symbols}")

    async def _quote_update_loop(self):
        """Quote update loop"""
        while True:
            try:
                for symbol in self.symbols:
                    await self._update_quotes(symbol)
                await asyncio.sleep(self.quote_refresh_rate)
            except Exception as e:
                self.log.error(f"Quote update error: {e}")
                await asyncio.sleep(1)

    async def _update_quotes(self, symbol: str):
        """Update quotes for specific symbol"""
        # Get current quote
        quote = self.cache.quote_tick(InstrumentId.from_str(symbol))
        if not quote:
            return

        # Update mid price and price history
        mid_price = (quote.bid + quote.ask) / 2
        self.mid_prices[symbol] = mid_price
        self.price_history[symbol].append(float(mid_price))

        # Maintain fixed window size
        if len(self.price_history[symbol]) > self.volatility_window:
            self.price_history[symbol].pop(0)

        # Update volatility
        self._update_volatility(symbol)

        # Cancel existing orders
        await self._cancel_all_orders(symbol)

        # Calculate new orders
        orders = self._calculate_orders(symbol, quote)

        # Place new orders
        for order_config in orders:
            await self._place_order(symbol, order_config)

        self.quote_updates += 1

    def _update_volatility(self, symbol: str):
        """Update volatility"""
        if len(self.price_history[symbol]) < 20:
            return

        # Calculate log returns
        prices = np.array(self.price_history[symbol])
        log_returns = np.diff(np.log(prices))

        # Annualized volatility
        volatility = np.std(log_returns) * np.sqrt(365 * 24 * 60)  # Assuming minute data
        self.volatilities[symbol] = Decimal(str(volatility))

    def _calculate_orders(self, symbol: str, quote: QuoteTick) -> List[dict]:
        """Calculate orders"""
        orders = []
        mid_price = self.mid_prices[symbol]
        position = self.positions[symbol]
        volatility = self.volatilities[symbol]

        # Calculate dynamic spread
        # Base spread + volatility adjustment
        spread_adjustment = volatility * Decimal('2')  # 2 standard deviations
        total_spread = self.base_spread + spread_adjustment

        # Calculate inventory skew
        inventory_ratio = position / self.max_position
        skew_adjustment = inventory_ratio * self.skew_factor * total_spread

        # Generate multi-level quotes
        for level in range(self.order_levels):
            # Incremental spread
            level_multiplier = 1 + (level * 0.2)

            # Bid price
            bid_spread = total_spread * level_multiplier * (1 + skew_adjustment)
            bid_price = mid_price - bid_spread
            bid_size = self._calculate_order_size(
                symbol, OrderSide.BUY, position, level
            )

            # Ask price
            ask_spread = total_spread * level_multiplier * (1 - skew_adjustment)
            ask_price = mid_price + ask_spread
            ask_size = self._calculate_order_size(
                symbol, OrderSide.SELL, position, level
            )

            if bid_size > 0:
                orders.append({
                    'side': OrderSide.BUY,
                    'price': bid_price,
                    'size': bid_size,
                    'level': level
                })

            if ask_size > 0:
                orders.append({
                    'side': OrderSide.SELL,
                    'price': ask_price,
                    'size': ask_size,
                    'level': level
                })

        return orders

    def _calculate_order_size(self, symbol: str, side: OrderSide,
                             position: Decimal, level: int) -> Quantity:
        """Calculate order size"""
        base_size = self.order_size_base

        # Decrease by level
        size_multiplier = Decimal(str(1 - (level * 0.1)))

        # Adjust by inventory
        if side == OrderSide.BUY and position > 0:
            # Long position, reduce buy orders
            inventory_factor = max(Decimal('0'), 1 - abs(position / self.max_position))
        elif side == OrderSide.SELL and position < 0:
            # Short position, reduce sell orders
            inventory_factor = max(Decimal('0'), 1 - abs(position / self.max_position))
        else:
            inventory_factor = Decimal('1')

        # Calculate final size
        final_size = base_size * size_multiplier * inventory_factor

        # Ensure minimum order size
        min_size = Decimal('0.001')
        final_size = max(final_size, min_size)

        return Quantity.from_int(int(final_size))

    async def _place_order(self, symbol: str, order_config: dict):
        """Place order"""
        order = self.order_factory.limit(
            instrument_id=InstrumentId.from_str(symbol),
            side=order_config['side'],
            price=order_config['price'],
            quantity=order_config['size']
        )

        order_id = await self.submit_order(order)

        # Record active order
        self.active_orders[symbol][order_id] = {
            'config': order_config,
            'timestamp': self.clock.timestamp_ns()
        }

    async def _cancel_all_orders(self, symbol: str):
        """Cancel all orders"""
        for order_id in list(self.active_orders[symbol].keys()):
            await self.cancel_order(order_id)
            del self.active_orders[symbol][order_id]

    def on_order_filled(self, event: OrderFilled):
        """Handle order fill"""
        # Update position
        symbol = str(event.instrument_id)
        if event.side == OrderSide.BUY:
            self.positions[symbol] += event.filled_qty
        else:
            self.positions[symbol] -= event.filled_qty

        # Update statistics
        self.trade_count += 1
        # PnL calculation will be updated in position events

        self.log.info(f"Order filled: {event.instrument_id} {event.side} {event.filled_qty} @ {event.last_px}")

    async def _risk_monitor_loop(self):
        """Risk monitoring loop"""
        while True:
            try:
                await self._check_risk_limits()
                await asyncio.sleep(10)  # Check every 10 seconds
            except Exception as e:
                self.log.error(f"Risk monitor error: {e}")
                await asyncio.sleep(5)

    async def _check_risk_limits(self):
        """Check risk limits"""
        for symbol in self.symbols:
            position = self.positions[symbol]

            # Check position limits
            if abs(position) > self.max_position:
                self.log.warning(f"Position limit exceeded for {symbol}: {position}")
                # Force close position
                await self._emergency_close(symbol)

            # Check order count
            if len(self.active_orders[symbol]) > self.order_levels * 2 * len(self.symbols):
                self.log.warning(f"Too many orders for {symbol}: {len(self.active_orders[symbol])}")
                # Cancel some orders
                await self._cancel_excess_orders(symbol)

    async def _emergency_close(self, symbol: str):
        """Emergency close position"""
        position = self.positions[symbol]
        if position == 0:
            return

        # Cancel all orders
        await self._cancel_all_orders(symbol)

        # Place market order to close
        side = OrderSide.SELL if position > 0 else OrderSide.BUY
        order = self.order_factory.market(
            instrument_id=InstrumentId.from_str(symbol),
            side=side,
            quantity=Quantity.from_int(abs(int(position)))
        )

        await self.submit_order(order)
        self.log.critical(f"Emergency close position for {symbol}: {position}")

    async def _cancel_excess_orders(self, symbol: str):
        """Cancel excess orders"""
        orders = list(self.active_orders[symbol].items())
        # Keep best-priced orders
        orders.sort(key=lambda x: x[1]['config']['level'])

        max_orders = self.order_levels * 2  # Total orders on both sides
        for order_id, _ in orders[max_orders:]:
            await self.cancel_order(order_id)
            del self.active_orders[symbol][order_id]

    def get_strategy_metrics(self) -> Dict:
        """Get strategy metrics"""
        return {
            'positions': {symbol: float(pos) for symbol, pos in self.positions.items()},
            'mid_prices': {symbol: float(price) for symbol, price in self.mid_prices.items()},
            'volatilities': {symbol: float(vol) for symbol, vol in self.volatilities.items()},
            'active_orders': {symbol: len(orders) for symbol, orders in self.active_orders.items()},
            'total_trades': self.trade_count,
            'quote_updates': self.quote_updates,
            'total_pnl': float(self.total_pnl)
        }
```

## 19.5 Risk Management System

### 19.5.1 Risk Engine

```python
# src/risk/risk_engine.py
from decimal import Decimal
from typing import Dict, List, Optional
from dataclasses import dataclass
from datetime import datetime, timedelta
import asyncio

@dataclass
class RiskAlert:
    """Risk alert"""
    level: str  # INFO, WARNING, CRITICAL
    message: str
    timestamp: datetime
    details: dict

class RiskEngine:
    """Risk management engine"""

    def __init__(self, config: dict):
        self.config = config

        # Risk limits
        self.max_daily_loss = Decimal(str(config.get('max_daily_loss', 10000)))
        self.max_drawdown = Decimal(str(config.get('max_drawdown', 0.05)))
        self.position_limits = {
            symbol: Decimal(str(limit))
            for symbol, limit in config.get('position_limit', {}).items()
        }
        self.leverage_limit = Decimal(str(config.get('leverage_limit', 3.0)))

        # Monitoring metrics
        self.daily_pnl = Decimal('0')
        self.max_portfolio_value = Decimal('0')
        self.daily_high_water_mark = Decimal('0')
        self.start_of_day_value = Decimal('0')

        # Alert history
        self.alerts = []
        self.alert_handlers = []

        # Monitoring task
        self.monitoring_active = True

    async def start_monitoring(self):
        """Start risk monitoring"""
        # Reset daily metrics
        self._reset_daily_metrics()

        # Start monitoring tasks
        asyncio.create_task(self._monitoring_loop())
        asyncio.create_task(self._daily_reset_loop())

    async def stop_monitoring(self):
        """Stop risk monitoring"""
        self.monitoring_active = False

    async def _monitoring_loop(self):
        """Monitoring loop"""
        while self.monitoring_active:
            try:
                await self._check_all_risks()
                await asyncio.sleep(1)  # Check every second
            except Exception as e:
                self.log.error(f"Risk monitoring error: {e}")
                await asyncio.sleep(5)

    async def _daily_reset_loop(self):
        """Daily reset loop"""
        while self.monitoring_active:
            # Wait until midnight
            now = datetime.now()
            next_midnight = now.replace(hour=0, minute=0, second=0, microsecond=0) + timedelta(days=1)
            sleep_seconds = (next_midnight - now).total_seconds()

            await asyncio.sleep(sleep_seconds)
            self._reset_daily_metrics()

    def _reset_daily_metrics(self):
        """Reset daily metrics"""
        self.daily_pnl = Decimal('0')
        self.start_of_day_value = self._get_current_portfolio_value()
        self.daily_high_water_mark = self.start_of_day_value
        self.log.info("Daily risk metrics reset")

    async def _check_all_risks(self):
        """Check all risks"""
        # Check position limits
        await self._check_position_limits()

        # Check daily loss
        await self._check_daily_loss()

        # Check drawdown
        await self._check_drawdown()

        # Check leverage
        await self._check_leverage()

        # Check order risks
        await self._check_order_risks()

    async def _check_position_limits(self):
        """Check position limits"""
        positions = self._get_current_positions()

        for symbol, position in positions.items():
            if symbol in self.position_limits:
                limit = self.position_limits[symbol]
                if abs(position) > limit:
                    await self._trigger_alert(
                        level='CRITICAL',
                        message=f"Position limit exceeded for {symbol}",
                        details={
                            'symbol': symbol,
                            'position': float(position),
                            'limit': float(limit)
                        }
                    )

    async def _check_daily_loss(self):
        """Check daily loss"""
        current_value = self._get_current_portfolio_value()
        daily_pnl = current_value - self.start_of_day_value

        if -daily_pnl > self.max_daily_loss:
            await self._trigger_alert(
                level='CRITICAL',
                message="Daily loss limit exceeded",
                details={
                    'daily_loss': float(-daily_pnl),
                    'limit': float(self.max_daily_loss),
                    'daily_pnl': float(daily_pnl)
                }
            )
        elif -daily_pnl > self.max_daily_loss * Decimal('0.8'):
            await self._trigger_alert(
                level='WARNING',
                message="Approaching daily loss limit",
                details={
                    'daily_loss': float(-daily_pnl),
                    'limit': float(self.max_daily_loss),
                    'utilization': float(-daily_pnl / self.max_daily_loss)
                }
            )

    async def _check_drawdown(self):
        """Check drawdown"""
        current_value = self._get_current_portfolio_value()

        # Update high water mark
        self.max_portfolio_value = max(self.max_portfolio_value, current_value)

        # Calculate drawdown
        if self.max_portfolio_value > 0:
            drawdown = (self.max_portfolio_value - current_value) / self.max_portfolio_value

            if drawdown > self.max_drawdown:
                await self._trigger_alert(
                    level='CRITICAL',
                    message="Maximum drawdown exceeded",
                    details={
                        'drawdown': float(drawdown),
                        'limit': float(self.max_drawdown),
                        'current_value': float(current_value),
                        'peak_value': float(self.max_portfolio_value)
                    }
                )
            elif drawdown > self.max_drawdown * Decimal('0.8'):
                await self._trigger_alert(
                    level='WARNING',
                    message="Approaching maximum drawdown",
                    details={
                        'drawdown': float(drawdown),
                        'limit': float(self.max_drawdown)
                    }
                )

    async def _check_leverage(self):
        """Check leverage"""
        total_position_value = self._get_total_position_value()
        current_value = self._get_current_portfolio_value()

        if current_value > 0:
            leverage = total_position_value / current_value

            if leverage > self.leverage_limit:
                await self._trigger_alert(
                    level='WARNING',
                    message="Leverage limit exceeded",
                    details={
                        'leverage': float(leverage),
                        'limit': float(self.leverage_limit)
                    }
                )

    async def _check_order_risks(self):
        """Check order risks"""
        # Check order concentration
        # Check large orders
        # Check order frequency
        pass

    async def _trigger_alert(self, level: str, message: str, details: dict):
        """Trigger risk alert"""
        alert = RiskAlert(
            level=level,
            message=message,
            timestamp=datetime.now(),
            details=details
        )

        self.alerts.append(alert)

        # Notify all handlers
        for handler in self.alert_handlers:
            try:
                await handler(alert)
            except Exception as e:
                self.log.error(f"Alert handler error: {e}")

    def add_alert_handler(self, handler):
        """Add alert handler"""
        self.alert_handlers.append(handler)

    def get_risk_summary(self) -> dict:
        """Get risk summary"""
        return {
            'daily_pnl': float(self.daily_pnl),
            'max_drawdown': float(self._calculate_current_drawdown()),
            'leverage': float(self._get_current_leverage()),
            'alert_count': len(self.alerts),
            'critical_alerts': len([a for a in self.alerts if a.level == 'CRITICAL']),
            'position_utilization': self._get_position_utilization()
        }

    # Methods below need integration with actual system
    def _get_current_portfolio_value(self) -> Decimal:
        """Get current portfolio value"""
        # Implement portfolio value retrieval logic
        return Decimal('100000')  # Example value

    def _get_current_positions(self) -> Dict[str, Decimal]:
        """Get current positions"""
        # Implement position retrieval logic
        return {}  # Example value

    def _get_total_position_value(self) -> Decimal:
        """Get total position value"""
        # Implement total position value calculation logic
        return Decimal('50000')  # Example value

    def _calculate_current_drawdown(self) -> Decimal:
        """Calculate current drawdown"""
        current_value = self._get_current_portfolio_value()
        if self.max_portfolio_value > 0:
            return (self.max_portfolio_value - current_value) / self.max_portfolio_value
        return Decimal('0')

    def _get_current_leverage(self) -> Decimal:
        """Get current leverage"""
        total_position_value = self._get_total_position_value()
        current_value = self._get_current_portfolio_value()

        if current_value > 0:
            return total_position_value / current_value
        return Decimal('0')

    def _get_position_utilization(self) -> dict:
        """Get position utilization"""
        positions = self._get_current_positions()
        utilization = {}

        for symbol, position in positions.items():
            if symbol in self.position_limits:
                limit = self.position_limits[symbol]
                if limit > 0:
                    utilization[symbol] = float(abs(position) / limit)
                else:
                    utilization[symbol] = 0.0
            else:
                utilization[symbol] = 0.0

        return utilization
```

## 19.6 Monitoring and Alert System

### 19.6.1 Performance Monitoring

```python
# src/monitoring/performance_monitor.py
import time
import psutil
import asyncio
from datetime import datetime, timedelta
from typing import Dict, List
import pandas as pd
import plotly.graph_objects as go
import plotly.express as px
from prometheus_client import Gauge, Counter, Histogram, start_http_server

class PerformanceMonitor:
    """Performance monitor"""

    def __init__(self, config: dict):
        self.config = config
        self.metrics_interval = config.get('metrics_interval', 5)
        self.health_check_interval = config.get('health_check_interval', 30)

        # Prometheus metrics
        self.setup_prometheus_metrics()

        # Performance data storage
        self.metrics_history = {
            'timestamp': [],
            'cpu_usage': [],
            'memory_usage': [],
            'latency': [],
            'throughput': [],
            'error_rate': [],
            'pnl': []
        }

        # Monitoring status
        self.monitoring_active = False
        self.start_time = None

    def setup_prometheus_metrics(self):
        """Setup Prometheus metrics"""
        # System metrics
        self.cpu_gauge = Gauge('trading_system_cpu_usage', 'CPU usage percentage')
        self.memory_gauge = Gauge('trading_system_memory_usage', 'Memory usage percentage')

        # Business metrics
        self.pnl_gauge = Gauge('trading_system_pnl', 'Current PnL')
        self.position_gauge = Gauge('trading_system_position', 'Current position size')
        self.order_latency = Histogram('trading_order_latency_seconds', 'Order latency in seconds')
        self.order_counter = Counter('trading_orders_total', 'Total orders')
        self.trade_counter = Counter('trading_trades_total', 'Total trades')
        self.error_counter = Counter('trading_errors_total', 'Total errors')

    async def start_monitoring(self, port: int = 8000):
        """Start monitoring"""
        self.monitoring_active = True
        self.start_time = datetime.now()

        # Start Prometheus HTTP server
        start_http_server(port)

        # Start monitoring tasks
        asyncio.create_task(self._metrics_collection_loop())
        asyncio.create_task(self._health_check_loop())
        asyncio.create_task(self._performance_analysis_loop())

        self.log.info(f"Performance monitoring started on port {port}")

    async def stop_monitoring(self):
        """Stop monitoring"""
        self.monitoring_active = False
        self.log.info("Performance monitoring stopped")

    async def _metrics_collection_loop(self):
        """Metrics collection loop"""
        while self.monitoring_active:
            try:
                # Collect system metrics
                cpu_percent = psutil.cpu_percent()
                memory_percent = psutil.virtual_memory().percent

                # Update Prometheus metrics
                self.cpu_gauge.set(cpu_percent)
                self.memory_gauge.set(memory_percent)

                # Store historical data
                timestamp = datetime.now()
                self.metrics_history['timestamp'].append(timestamp)
                self.metrics_history['cpu_usage'].append(cpu_percent)
                self.metrics_history['memory_usage'].append(memory_percent)

                # Maintain fixed length
                max_history = 1000
                for key in self.metrics_history:
                    if len(self.metrics_history[key]) > max_history:
                        self.metrics_history[key].pop(0)

                await asyncio.sleep(self.metrics_interval)

            except Exception as e:
                self.log.error(f"Metrics collection error: {e}")
                await asyncio.sleep(5)

    async def _health_check_loop(self):
        """Health check loop"""
        while self.monitoring_active:
            try:
                await self._perform_health_check()
                await asyncio.sleep(self.health_check_interval)
            except Exception as e:
                self.log.error(f"Health check error: {e}")
                await asyncio.sleep(10)

    async def _perform_health_check(self):
        """Perform health check"""
        health_status = {
            'timestamp': datetime.now(),
            'status': 'healthy',
            'checks': {}
        }

        # CPU usage check
        cpu_usage = psutil.cpu_percent()
        health_status['checks']['cpu'] = {
            'status': 'ok' if cpu_usage < 80 else 'warning',
            'value': cpu_usage
        }

        # Memory usage check
        memory_usage = psutil.virtual_memory().percent
        health_status['checks']['memory'] = {
            'status': 'ok' if memory_usage < 85 else 'warning',
            'value': memory_usage
        }

        # Disk space check
        disk_usage = psutil.disk_usage('/').percent
        health_status['checks']['disk'] = {
            'status': 'ok' if disk_usage < 90 else 'warning',
            'value': disk_usage
        }

        # Check overall status
        if any(check['status'] != 'ok' for check in health_status['checks'].values()):
            health_status['status'] = 'unhealthy'

        # Store health status
        if not hasattr(self, 'health_history'):
            self.health_history = []
        self.health_history.append(health_status)

        # Keep last 100 checks
        if len(self.health_history) > 100:
            self.health_history.pop(0)

    async def _performance_analysis_loop(self):
        """Performance analysis loop"""
        while self.monitoring_active:
            try:
                # Perform performance analysis every minute
                await asyncio.sleep(60)
                await self._analyze_performance()
            except Exception as e:
                self.log.error(f"Performance analysis error: {e}")

    async def _analyze_performance(self):
        """Analyze performance"""
        if len(self.metrics_history['timestamp']) < 10:
            return

        # Create DataFrame
        df = pd.DataFrame({
            'timestamp': self.metrics_history['timestamp'][-60:],  # Last 60 data points
            'cpu_usage': self.metrics_history['cpu_usage'][-60:],
            'memory_usage': self.metrics_history['memory_usage'][-60:]
        })

        # Calculate statistical metrics
        performance_report = {
            'timestamp': datetime.now(),
            'cpu': {
                'avg': df['cpu_usage'].mean(),
                'max': df['cpu_usage'].max(),
                'min': df['cpu_usage'].min(),
                'std': df['cpu_usage'].std()
            },
            'memory': {
                'avg': df['memory_usage'].mean(),
                'max': df['memory_usage'].max(),
                'min': df['memory_usage'].min(),
                'std': df['memory_usage'].std()
            }
        }

        # Detect performance anomalies
        if performance_report['cpu']['avg'] > 70:
            await self._send_performance_alert(
                'High CPU usage detected',
                performance_report
            )

        if performance_report['memory']['avg'] > 80:
            await self._send_performance_alert(
                'High memory usage detected',
                performance_report
            )

    async def _send_performance_alert(self, message: str, details: dict):
        """Send performance alert"""
        alert = {
            'type': 'performance',
            'level': 'warning',
            'message': message,
            'details': details,
            'timestamp': datetime.now()
        }

        # Integrate with alert system here
        self.log.warning(f"Performance alert: {message}")

    def record_order_latency(self, latency_seconds: float):
        """Record order latency"""
        self.order_latency.observe(latency_seconds)

    def increment_order_counter(self):
        """Increment order counter"""
        self.order_counter.inc()

    def increment_trade_counter(self):
        """Increment trade counter"""
        self.trade_counter.inc()

    def increment_error_counter(self):
        """Increment error counter"""
        self.error_counter.inc()

    def update_pnl(self, pnl: float):
        """Update PnL"""
        self.pnl_gauge.set(pnl)
        self.metrics_history['pnl'].append(pnl)

        # Maintain fixed length
        if len(self.metrics_history['pnl']) > 1000:
            self.metrics_history['pnl'].pop(0)

    def update_position(self, position: float):
        """Update position"""
        self.position_gauge.set(abs(position))

    def generate_performance_report(self) -> dict:
        """Generate performance report"""
        uptime = (datetime.now() - self.start_time).total_seconds() if self.start_time else 0

        return {
            'uptime_seconds': uptime,
            'uptime_hours': uptime / 3600,
            'metrics_summary': {
                'avg_cpu': sum(self.metrics_history['cpu_usage']) / len(self.metrics_history['cpu_usage']) if self.metrics_history['cpu_usage'] else 0,
                'avg_memory': sum(self.metrics_history['memory_usage']) / len(self.metrics_history['memory_usage']) if self.metrics_history['memory_usage'] else 0,
                'peak_cpu': max(self.metrics_history['cpu_usage']) if self.metrics_history['cpu_usage'] else 0,
                'peak_memory': max(self.metrics_history['memory_usage']) if self.metrics_history['memory_usage'] else 0
            },
            'health_status': self.health_history[-1] if self.health_history else None,
            'prometheus_metrics_url': 'http://localhost:8000/metrics'
        }

    def create_performance_dashboard(self) -> str:
        """Create performance dashboard"""
        # CPU usage chart
        cpu_fig = go.Figure()
        cpu_fig.add_trace(go.Scatter(
            x=self.metrics_history['timestamp'],
            y=self.metrics_history['cpu_usage'],
            name='CPU Usage (%)',
            line=dict(color='blue')
        ))
        cpu_fig.update_layout(title='CPU Usage Over Time', yaxis_title='Percentage')

        # Memory usage chart
        memory_fig = go.Figure()
        memory_fig.add_trace(go.Scatter(
            x=self.metrics_history['timestamp'],
            y=self.metrics_history['memory_usage'],
            name='Memory Usage (%)',
            line=dict(color='red')
        ))
        memory_fig.update_layout(title='Memory Usage Over Time', yaxis_title='Percentage')

        # System resource overview
        current_cpu = self.metrics_history['cpu_usage'][-1] if self.metrics_history['cpu_usage'] else 0
        current_memory = self.metrics_history['memory_usage'][-1] if self.metrics_history['memory_usage'] else 0

        overview_fig = go.Figure()
        overview_fig.add_trace(go.Bar(
            name='Current Usage',
            x=['CPU', 'Memory'],
            y=[current_cpu, current_memory]
        ))
        overview_fig.add_trace(go.Bar(
            name='Threshold',
            x=['CPU', 'Memory'],
            y=[80, 85]
        ))
        overview_fig.update_layout(title='System Resource Overview', barmode='group')

        # Generate HTML report
        html_report = f"""
        <html>
            <head>
                <title>Performance Dashboard</title>
                <script src="https://cdn.plot.ly/plotly-latest.min.js"></script>
            </head>
            <body>
                <h1>Trading System Performance Dashboard</h1>
                <div id="cpu-chart"></div>
                <div id="memory-chart"></div>
                <div id="overview-chart"></div>
                <script>
                    {cpu_fig.to_html(include_plotlyjs=False, div_id="cpu-chart")}
                    {memory_fig.to_html(include_plotlyjs=False, div_id="memory-chart")}
                    {overview_fig.to_html(include_plotlyjs=False, div_id="overview-chart")}
                </script>
            </body>
        </html>
        """

        return html_report
```

## 19.7 Main Program Entry

### 19.7.1 Application Main Class

```python
# src/main.py
import asyncio
import signal
import sys
import logging
from pathlib import Path

from src.config.config_manager import ConfigManager
from src.data.unified_data_client import UnifiedDataClient
from src.strategies.market_maker import DynamicMarketMaker
from src.risk.risk_engine import RiskEngine
from src.monitoring.performance_monitor import PerformanceMonitor
from src.utils.logger import setup_logging

class TradingApplication:
    """Trading application main class"""

    def __init__(self, config_path: str):
        # Setup logging
        setup_logging()
        self.log = logging.getLogger(__name__)

        # Load configuration
        self.config_manager = ConfigManager(config_path)

        # Core components
        self.data_client = None
        self.strategy = None
        self.risk_engine = None
        self.performance_monitor = None

        # Running state
        self.running = False
        self.shutdown_event = asyncio.Event()

    async def initialize(self):
        """Initialize application"""
        self.log.info("Initializing trading application...")

        try:
            # Initialize data client
            self.data_client = UnifiedDataClient(self.config_manager)
            await self.data_client.initialize()
            self.log.info("Data client initialized")

            # Initialize risk engine
            risk_config = self.config_manager.get('risk', {})
            self.risk_engine = RiskEngine(risk_config)
            await self.risk_engine.start_monitoring()
            self.log.info("Risk engine initialized")

            # Initialize performance monitor
            monitor_config = self.config_manager.get('monitoring', {})
            self.performance_monitor = PerformanceMonitor(monitor_config)
            await self.performance_monitor.start_monitoring(
                port=self.config_manager.get('api.port', 8080)
            )
            self.log.info("Performance monitor initialized")

            # Initialize strategy
            strategy_config = self.config_manager.get('strategy', {})
            self.strategy = DynamicMarketMaker(strategy_config)
            self.log.info("Strategy initialized")

            # Register risk alert handlers
            self.risk_engine.add_alert_handler(self._handle_risk_alert)

            self.log.info("Trading application initialized successfully")

        except Exception as e:
            self.log.error(f"Failed to initialize application: {e}")
            raise

    async def start(self):
        """Start trading application"""
        self.log.info("Starting trading application...")

        try:
            self.running = True

            # Setup signal handlers
            self._setup_signal_handlers()

            # Start strategy
            self.data_client.subscribe_quotes(
                self.strategy.symbols,
                self.strategy.on_quote_tick
            )

            # Start main loop
            await self._main_loop()

        except Exception as e:
            self.log.error(f"Error in main loop: {e}")
            raise

    async def stop(self):
        """Stop trading application"""
        self.log.info("Stopping trading application...")

        self.running = False
        self.shutdown_event.set()

        try:
            # Stop strategy
            if self.strategy:
                # Cancel all orders
                for symbol in self.strategy.symbols:
                    await self.strategy._cancel_all_orders(symbol)
                self.log.info("All orders canceled")

            # Stop performance monitor
            if self.performance_monitor:
                await self.performance_monitor.stop_monitoring()
                self.log.info("Performance monitor stopped")

            # Stop risk engine
            if self.risk_engine:
                await self.risk_engine.stop_monitoring()
                self.log.info("Risk engine stopped")

            # Disconnect data client
            if self.data_client:
                await self.data_client.disconnect()
                self.log.info("Data client disconnected")

            self.log.info("Trading application stopped successfully")

        except Exception as e:
            self.log.error(f"Error during shutdown: {e}")

    def _setup_signal_handlers(self):
        """Setup signal handlers"""
        def signal_handler(signum, frame):
            self.log.info(f"Received signal {signum}, shutting down...")
            asyncio.create_task(self.stop())

        signal.signal(signal.SIGINT, signal_handler)
        signal.signal(signal.SIGTERM, signal_handler)

    async def _main_loop(self):
        """Main loop"""
        while self.running and not self.shutdown_event.is_set():
            try:
                # Regular tasks
                await self._perform_regular_tasks()

                # Wait a short time
                await asyncio.sleep(1)

            except Exception as e:
                self.log.error(f"Error in main loop: {e}")
                await asyncio.sleep(5)

    async def _perform_regular_tasks(self):
        """Perform regular tasks"""
        # Print status every 5 minutes
        if hasattr(self, '_last_status_time'):
            if time.time() - self._last_status_time > 300:
                await self._print_status()
                self._last_status_time = time.time()
        else:
            self._last_status_time = time.time()

    async def _print_status(self):
        """Print system status"""
        # Get strategy metrics
        strategy_metrics = self.strategy.get_strategy_metrics() if self.strategy else {}

        # Get risk summary
        risk_summary = self.risk_engine.get_risk_summary() if self.risk_engine else {}

        # Get performance report
        performance_report = self.performance_monitor.generate_performance_report() if self.performance_monitor else {}

        self.log.info("=== System Status ===")
        self.log.info(f"Strategy metrics: {strategy_metrics}")
        self.log.info(f"Risk summary: {risk_summary}")
        self.log.info(f"Performance: Uptime {performance_report.get('uptime_hours', 0):.1f}h")
        self.log.info("====================")

    async def _handle_risk_alert(self, alert):
        """Handle risk alert"""
        self.log.warning(f"Risk alert [{alert.level}]: {alert.message}")

        # Take action based on alert level
        if alert.level == 'CRITICAL':
            # Critical alert, may need to stop trading
            self.log.critical("Critical risk alert, considering emergency stop")
            # Implement emergency stop logic here

    def generate_report(self) -> str:
        """Generate trading report"""
        report_lines = [
            "=== Trading System Report ===",
            f"Timestamp: {datetime.now().isoformat()}",
            ""
        ]

        # Strategy report
        if self.strategy:
            strategy_metrics = self.strategy.get_strategy_metrics()
            report_lines.extend([
                "Strategy Metrics:",
                f"  Positions: {strategy_metrics.get('positions', {})}",
                f"  Total Trades: {strategy_metrics.get('total_trades', 0)}",
                f"  Quote Updates: {strategy_metrics.get('quote_updates', 0)}",
                ""
            ])

        # Risk report
        if self.risk_engine:
            risk_summary = self.risk_engine.get_risk_summary()
            report_lines.extend([
                "Risk Summary:",
                f"  Daily PnL: {risk_summary.get('daily_pnl', 0):.2f}",
                f"  Max Drawdown: {risk_summary.get('max_drawdown', 0):.2%}",
                f"  Leverage: {risk_summary.get('leverage', 0):.2f}",
                f"  Active Alerts: {risk_summary.get('alert_count', 0)}",
                ""
            ])

        # Performance report
        if self.performance_monitor:
            perf_report = self.performance_monitor.generate_performance_report()
            report_lines.extend([
                "Performance Metrics:",
                f"  Uptime: {perf_report.get('uptime_hours', 0):.1f} hours",
                f"  Avg CPU: {perf_report.get('metrics_summary', {}).get('avg_cpu', 0):.1f}%",
                f"  Avg Memory: {perf_report.get('metrics_summary', {}).get('avg_memory', 0):.1f}%",
                ""
            ])

        report_lines.append("=== End of Report ===")

        return "\n".join(report_lines)

async def main():
    """Main function"""
    # Check command line arguments
    if len(sys.argv) < 2:
        print("Usage: python main.py <config_path>")
        sys.exit(1)

    config_path = sys.argv[1]

    # Create application instance
    app = TradingApplication(config_path)

    try:
        # Initialize
        await app.initialize()

        # Start
        await app.start()

    except KeyboardInterrupt:
        print("\nReceived keyboard interrupt, shutting down...")
    except Exception as e:
        print(f"Fatal error: {e}")
        return 1
    finally:
        # Cleanup
        await app.stop()

    return 0

if __name__ == "__main__":
    # Run main function
    exit_code = asyncio.run(main())
    sys.exit(exit_code)
```

## 19.8 Deployment and Running

### 19.8.1 Docker Deployment

```dockerfile
# Dockerfile
FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy dependencies file
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY src/ ./src/
COPY config/ ./config/

# Create non-root user
RUN useradd --create-home --shell /bin/bash nautilus
USER nautilus

# Expose ports
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8080/health')" || exit 1

# Startup command
CMD ["python", "-m", "src.main", "config/production.yaml"]
```

### 19.8.2 Docker Compose Deployment

```yaml
# docker-compose.yml
version: '3.8'

services:
  trading-system:
    build: .
    environment:
      - DB_PASSWORD=${DB_PASSWORD}
      - REDIS_PASSWORD=${REDIS_PASSWORD}
    volumes:
      - ./logs:/app/logs
      - ./data:/app/data
      - ./config:/app/config:ro
    ports:
      - "8080:8080"
      - "8000:8000"  # Prometheus metrics
    depends_on:
      - postgres
      - redis
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '1'
          memory: 2G

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: market_maker
      POSTGRES_USER: nautilus
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    restart: unless-stopped

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
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - trading-system
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
  prometheus_data:
  grafana_data:
```

## Summary

This chapter demonstrates through a complete cryptocurrency market making trading system project how to:

1. **Integrate All Components**
   - Configuration management system
   - Unified data interface
   - Dynamic market making strategy
   - Risk management system
   - Performance monitoring system

2. **Implement Production-grade Features**
   - Multi-exchange support
   - Real-time risk control
   - Comprehensive monitoring and alerting
   - Automated deployment

3. **Apply Best Practices**
   - Modular design
   - Error handling
   - Logging
   - Performance optimization

This project showcases the entire process from concept to implementation, providing a complete reference for building actual trading systems.

## References

- [Algorithmic Trading with Python](https://www.oreilly.com/library/view/algorithmic-trading-with/9781492074215/)
- [High-Frequency Trading](https://www.oreilly.com/library/view/high-frequency-trading/9781492058434/)
- [Building Algorithmic Trading Systems](https://www.wiley.com/en-us/Building+Algorithmic+Trading+Systems)

## Runnable Code Entry (Source Build)

- Entry script: docs/tutorials_cn/code/chapter19/run_crypto_mm.py
- Key examples: docs/tutorials_cn/code/chapter19/crypto_market_making.py, docs/tutorials_cn/code/chapter19/mm_performance_analyzer.py

Dependency strategy: Implements a reproducible quote -> fill simulation -> PnL path -> performance analysis pipeline.

Verified command:

```bash
uv run --python 3.13 python docs/tutorials_cn/code/chapter19/run_crypto_mm.py
```
