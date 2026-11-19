---
title: "Chapter 14: Exchange Integration"
---

# Chapter 14: Exchange Integration

## Learning Objectives

- Understand the core concepts and architecture of exchange integration
- Master the integration methods for different types of exchanges
- Learn to handle exchange API rate limiting and errors
- Implement stable trading data streams and order management

## 14.1 Exchange Integration Architecture

### 14.1.1 Adapter Pattern Design

```python
from abc import ABC, abstractmethod
from typing import Dict, List, Optional, Union
from dataclasses import dataclass
from enum import Enum
import asyncio
import aiohttp
import websockets
from datetime import datetime

# Exchange types
class ExchangeType(Enum):
    """Exchange type enumeration"""
    CENTRALIZED = "centralized"     # Centralized exchange
    DECENTRALIZED = "decentralized" # Decentralized exchange
    DERIVATIVES = "derivatives"     # Derivatives exchange
    SPOT = "spot"                  # Spot exchange
    FX = "fx"                      # Foreign exchange

# Trading pair information
@dataclass
class TradingPair:
    """Trading pair information"""
    symbol: str
    base_asset: str
    quote_asset: str
    min_quantity: float
    max_quantity: float
    quantity_precision: int
    price_precision: int
    min_notional: float
    is_active: bool = True

# Base exchange adapter class
class ExchangeAdapter(ABC):
    """Base exchange adapter class"""

    def __init__(self, config: dict):
        self.config = config
        self.name = config.get('name')
        self.api_key = config.get('api_key')
        self.secret_key = config.get('secret_key')
        self.passphrase = config.get('passphrase')  # Required by some exchanges
        self.sandbox = config.get('sandbox', False)
        self.rate_limiter = RateLimiter(config.get('rate_limits', {}))
        self.session = None
        self.ws_connection = None

    async def initialize(self):
        """Initialize adapter"""
        self.session = aiohttp.ClientSession()
        await self._setup_websocket()

    async def close(self):
        """Close adapter"""
        if self.session:
            await self.session.close()
        if self.ws_connection:
            await self.ws_connection.close()

    @abstractmethod
    async def get_trading_pairs(self) -> List[TradingPair]:
        """Get trading pairs list"""
        pass

    @abstractmethod
    async def get_ticker(self, symbol: str) -> dict:
        """Get ticker data"""
        pass

    @abstractmethod
    async def get_orderbook(self, symbol: str, limit: int = 100) -> dict:
        """Get order book"""
        pass

    @abstractmethod
    async def get_recent_trades(self, symbol: str, limit: int = 100) -> List[dict]:
        """Get recent trades"""
        pass

    @abstractmethod
    async def place_order(self, order: dict) -> dict:
        """Place order"""
        pass

    @abstractmethod
    async def cancel_order(self, order_id: str, symbol: str) -> dict:
        """Cancel order"""
        pass

    @abstractmethod
    async def get_order_status(self, order_id: str, symbol: str) -> dict:
        """Query order status"""
        pass

    @abstractmethod
    async def get_balance(self) -> dict:
        """Get account balance"""
        pass

    @abstractmethod
    async def subscribe_to_market_data(self, symbols: List[str], callbacks: dict):
        """Subscribe to market data"""
        pass

    async def _make_request(self, method: str, endpoint: str, params: dict = None,
                           auth_required: bool = False) -> dict:
        """Send HTTP request"""
        await self.rate_limiter.wait_if_needed()

        url = self._build_url(endpoint)

        if auth_required:
            params = self._sign_request(params or {})

        async with self.session.request(method, url, params=params) as response:
            if response.status == 200:
                return await response.json()
            else:
                raise ExchangeError(f"Request failed: {response.status}")

    def _build_url(self, endpoint: str) -> str:
        """Build complete URL"""
        base_url = self._get_base_url()
        return f"{base_url}{endpoint}"

    @abstractmethod
    def _get_base_url(self) -> str:
        """Get base URL"""
        pass

    @abstractmethod
    def _sign_request(self, params: dict) -> dict:
        """Sign request"""
        pass

    async def _setup_websocket(self):
        """Setup WebSocket connection"""
        pass

# Rate limiter
class RateLimiter:
    """API rate limiter"""

    def __init__(self, limits: dict):
        self.limits = limits
        self.requests = {}  # {endpoint: [timestamp1, timestamp2, ...]}
        self.lock = asyncio.Lock()

    async def wait_if_needed(self, endpoint: str = "default"):
        """Wait if needed to comply with rate limiting"""
        async with self.lock:
            if endpoint not in self.requests:
                self.requests[endpoint] = []

            # Get limit for this endpoint
            limit = self.limits.get(endpoint, self.limits.get("default", 10))
            window = self.limits.get(f"{endpoint}_window", 60)  # Default 60-second window

            # Clean expired request records
            now = time.time()
            self.requests[endpoint] = [
                ts for ts in self.requests[endpoint]
                if now - ts < window
            ]

            # Check if limit is exceeded
            if len(self.requests[endpoint]) >= limit:
                # Calculate required wait time
                oldest_request = min(self.requests[endpoint])
                wait_time = window - (now - oldest_request)
                if wait_time > 0:
                    await asyncio.sleep(wait_time)

            # Record current request
            self.requests[endpoint].append(now)

# Exception classes
class ExchangeError(Exception):
    """Base exchange error class"""
    pass

class RateLimitError(ExchangeError):
    """Rate limit error"""
    pass

class InvalidOrderError(ExchangeError):
    """Invalid order error"""
    pass

class InsufficientFundsError(ExchangeError):
    """Insufficient funds error"""
    pass
```

### 14.1.2 Binance Integration Example

```python
import hashlib
import hmac
import time
from urllib.parse import urlencode

class BinanceAdapter(ExchangeAdapter):
    """Binance exchange adapter"""

    def __init__(self, config: dict):
        super().__init__(config)
        self.base_url = "https://api.binance.com"
        if self.sandbox:
            self.base_url = "https://testnet.binance.vision"
        self.ws_base_url = "wss://stream.binance.com:9443"
        if self.sandbox:
            self.ws_base_url = "wss://testnet.binance.vision"

    def _get_base_url(self) -> str:
        return self.base_url

    def _sign_request(self, params: dict) -> dict:
        """Sign Binance request"""
        if not params:
            params = {}

        # Add timestamp
        params['timestamp'] = int(time.time() * 1000)

        # Create query string
        query_string = urlencode(params)

        # Sign
        signature = hmac.new(
            self.secret_key.encode('utf-8'),
            query_string.encode('utf-8'),
            hashlib.sha256
        ).hexdigest()

        params['signature'] = signature
        return params

    async def get_trading_pairs(self) -> List[TradingPair]:
        """Get Binance trading pairs"""
        endpoint = "/api/v3/exchangeInfo"
        response = await self._make_request("GET", endpoint)

        pairs = []
        for symbol_info in response['symbols']:
            if symbol_info['status'] == 'TRADING':
                # Find minimum and maximum quantity limits
                min_qty = 0.0
                max_qty = float('inf')
                qty_precision = 8
                price_precision = 8
                min_notional = 0.0

                for filter_item in symbol_info['filters']:
                    if filter_item['filterType'] == 'LOT_SIZE':
                        min_qty = float(filter_item['minQty'])
                        max_qty = float(filter_item['maxQty'])
                        qty_precision = self._get_precision(filter_item['stepSize'])
                    elif filter_item['filterType'] == 'PRICE_FILTER':
                        price_precision = self._get_precision(filter_item['tickSize'])
                    elif filter_item['filterType'] == 'MIN_NOTIONAL':
                        min_notional = float(filter_item['minNotional'])

                pair = TradingPair(
                    symbol=symbol_info['symbol'],
                    base_asset=symbol_info['baseAsset'],
                    quote_asset=symbol_info['quoteAsset'],
                    min_quantity=min_qty,
                    max_quantity=max_qty,
                    quantity_precision=qty_precision,
                    price_precision=price_precision,
                    min_notional=min_notional,
                    is_active=True
                )
                pairs.append(pair)

        return pairs

    def _get_precision(self, value_str: str) -> int:
        """Get precision from string"""
        if '.' in value_str:
            return len(value_str.split('.')[1].rstrip('0'))
        return 0

    async def get_ticker(self, symbol: str) -> dict:
        """Get ticker data"""
        endpoint = "/api/v3/ticker/24hr"
        params = {'symbol': symbol}
        response = await self._make_request("GET", endpoint, params)

        return {
            'symbol': response['symbol'],
            'price': float(response['lastPrice']),
            'change': float(response['priceChange']),
            'change_percent': float(response['priceChangePercent']),
            'high': float(response['highPrice']),
            'low': float(response['lowPrice']),
            'volume': float(response['volume']),
            'quote_volume': float(response['quoteVolume']),
            'open_time': response['openTime'],
            'close_time': response['closeTime']
        }

    async def get_orderbook(self, symbol: str, limit: int = 100) -> dict:
        """Get order book"""
        endpoint = "/api/v3/depth"
        params = {'symbol': symbol, 'limit': limit}
        response = await self._make_request("GET", endpoint, params)

        return {
            'symbol': symbol,
            'bids': [[float(price), float(qty)] for price, qty in response['bids']],
            'asks': [[float(price), float(qty)] for price, qty in response['asks']],
            'last_update': response['lastUpdateId']
        }

    async def get_recent_trades(self, symbol: str, limit: int = 100) -> List[dict]:
        """Get recent trades"""
        endpoint = "/api/v3/trades"
        params = {'symbol': symbol, 'limit': limit}
        response = await self._make_request("GET", endpoint, params)

        trades = []
        for trade in response:
            trades.append({
                'id': trade['id'],
                'price': float(trade['price']),
                'quantity': float(trade['qty']),
                'time': trade['time'],
                'is_buyer_maker': trade['isBuyerMaker']
            })

        return trades

    async def place_order(self, order: dict) -> dict:
        """Place order"""
        endpoint = "/api/v3/order"

        params = {
            'symbol': order['symbol'],
            'side': order['side'].upper(),
            'type': order['type'].upper(),
            'quantity': order['quantity']
        }

        # Add optional parameters
        if 'price' in order:
            params['price'] = order['price']
        if 'time_in_force' in order:
            params['timeInForce'] = order['time_in_force']
        if 'client_order_id' in order:
            params['newClientOrderId'] = order['client_order_id']

        response = await self._make_request("POST", endpoint, params, auth_required=True)

        return {
            'order_id': str(response['orderId']),
            'client_order_id': response['clientOrderId'],
            'symbol': response['symbol'],
            'status': response['status'],
            'side': response['side'],
            'type': response['type'],
            'quantity': float(response['origQty']),
            'executed_quantity': float(response['executedQty']),
            'price': float(response['price']) if response['price'] else None,
            'time': response['transactTime']
        }

    async def cancel_order(self, order_id: str, symbol: str) -> dict:
        """Cancel order"""
        endpoint = "/api/v3/order"
        params = {
            'symbol': symbol,
            'orderId': int(order_id)
        }

        response = await self._make_request("DELETE", endpoint, params, auth_required=True)

        return {
            'order_id': str(response['orderId']),
            'client_order_id': response['clientOrderId'],
            'symbol': response['symbol'],
            'status': response['status'],
            'side': response['side'],
            'type': response['type']
        }

    async def get_order_status(self, order_id: str, symbol: str) -> dict:
        """Query order status"""
        endpoint = "/api/v3/order"
        params = {
            'symbol': symbol,
            'orderId': int(order_id)
        }

        response = await self._make_request("GET", endpoint, params, auth_required=True)

        return {
            'order_id': str(response['orderId']),
            'client_order_id': response['clientOrderId'],
            'symbol': response['symbol'],
            'status': response['status'],
            'side': response['side'],
            'type': response['type'],
            'quantity': float(response['origQty']),
            'executed_quantity': float(response['executedQty']),
            'cummulative_quote_qty': float(response['cummulativeQuoteQty']),
            'price': float(response['price']) if response['price'] else None,
            'time': response['time'],
            'update_time': response['updateTime']
        }

    async def get_balance(self) -> dict:
        """Get account balance"""
        endpoint = "/api/v3/account"
        response = await self._make_request("GET", endpoint, auth_required=True)

        balances = {}
        for balance in response['balances']:
            free = float(balance['free'])
            locked = float(balance['locked'])
            if free > 0 or locked > 0:
                balances[balance['asset']] = {
                    'free': free,
                    'locked': locked,
                    'total': free + locked
                }

        return {
            'maker_commission': response['makerCommission'],
            'taker_commission': response['takerCommission'],
            'buyer_commission': response['buyerCommission'],
            'seller_commission': response['sellerCommission'],
            'can_trade': response['canTrade'],
            'can_withdraw': response['canWithdraw'],
            'can_deposit': response['canDeposit'],
            'balances': balances
        }

    async def subscribe_to_market_data(self, symbols: List[str], callbacks: dict):
        """Subscribe to market data"""
        # Build WebSocket stream names
        streams = []
        for symbol in symbols:
            if 'ticker' in callbacks:
                streams.append(f"{symbol.lower()}@ticker")
            if 'depth' in callbacks:
                streams.append(f"{symbol.lower()}@depth")
            if 'trade' in callbacks:
                streams.append(f"{symbol.lower()}@trade")
            if 'kline' in callbacks:
                streams.append(f"{symbol.lower()}@kline_1m")

        # Connect WebSocket
        url = f"{self.ws_base_url}/ws/{'/'.join(streams)}"
        self.ws_connection = await websockets.connect(url)

        # Handle messages
        async for message in self.ws_connection:
            data = json.loads(message)
            await self._handle_ws_message(data, callbacks)

    async def _handle_ws_message(self, data: dict, callbacks: dict):
        """Handle WebSocket message"""
        if 'e' in data:  # Event type
            event_type = data['e']
            symbol = data.get('s')

            if event_type == '24hrTicker' and 'ticker' in callbacks:
                ticker_data = {
                    'symbol': data['s'],
                    'price': float(data['c']),
                    'change': float(data['x']),
                    'change_percent': float(data['P']),
                    'high': float(data['h']),
                    'low': float(data['l']),
                    'volume': float(data['v']),
                    'quote_volume': float(data['q'])
                }
                await callbacks['ticker'](ticker_data)

            elif event_type == 'depthUpdate' and 'depth' in callbacks:
                depth_data = {
                    'symbol': data['s'],
                    'bids': [[float(price), float(qty)] for price, qty in data['b']],
                    'asks': [[float(price), float(qty)] for price, qty in data['a']],
                    'first_update': data['U'],
                    'final_update': data['u']
                }
                await callbacks['depth'](depth_data)

            elif event_type == 'trade' and 'trade' in callbacks:
                trade_data = {
                    'symbol': data['s'],
                    'id': data['t'],
                    'price': float(data['p']),
                    'quantity': float(data['q']),
                    'time': data['T'],
                    'is_buyer_maker': data['m']
                }
                await callbacks['trade'](trade_data)
```

## 14.2 Decentralized Exchange Integration

### 14.2.1 Uniswap Integration

```python
from web3 import Web3
from web3.middleware import geth_poa_middleware
import json

class UniswapAdapter(ExchangeAdapter):
    """Uniswap DEX adapter"""

    def __init__(self, config: dict):
        super().__init__(config)
        self.rpc_url = config.get('rpc_url', 'https://mainnet.infura.io/v3/YOUR_PROJECT_ID')
        self.w3 = Web3(Web3.HTTPProvider(self.rpc_url))

        # Uniswap contract addresses
        self.factory_address = config.get('factory_address', '0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f')
        self.router_address = config.get('router_address', '0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D')

        # Load ABIs
        self.factory_abi = self._load_abi('uniswap_factory.json')
        self.router_abi = self._load_abi('uniswap_router.json')
        self.erc20_abi = self._load_abi('erc20.json')

        # Create contract instances
        self.factory_contract = self.w3.eth.contract(
            address=self.factory_address,
            abi=self.factory_abi
        )
        self.router_contract = self.w3.eth.contract(
            address=self.router_address,
            abi=self.router_abi
        )

        # Setup account
        self.account = config.get('account_address')
        self.private_key = config.get('private_key')

    def _load_abi(self, filename: str) -> list:
        """Load ABI file"""
        with open(f'abi/{filename}', 'r') as f:
            return json.load(f)

    def _get_base_url(self) -> str:
        return ""

    def _sign_request(self, params: dict) -> dict:
        return params

    async def get_trading_pairs(self) -> List[TradingPair]:
        """Get trading pairs list (requires indexer support)"""
        # This needs to use The Graph or other indexer services
        # Simplified example, return some major pairs
        common_pairs = [
            ('WETH', 'USDC'),
            ('WETH', 'USDT'),
            ('WETH', 'DAI'),
            ('USDC', 'USDT'),
            ('WETH', 'WBTC')
        ]

        pairs = []
        for base, quote in common_pairs:
            pair = TradingPair(
                symbol=f"{base}/{quote}",
                base_asset=base,
                quote_asset=quote,
                min_quantity=0.000001,
                max_quantity=float('inf'),
                quantity_precision=6,
                price_precision=18,
                min_notional=10.0,
                is_active=True
            )
            pairs.append(pair)

        return pairs

    async def get_price(self, token_in: str, token_out: str, amount_in: float) -> dict:
        """Get exchange price"""
        try:
            # Get exchange path
            path = [token_in, token_out]

            # Get output amount
            amounts_out = self.router_contract.functions.getAmountsOut(
                int(amount_in * 10**18),  # Convert to wei
                path
            ).call()

            return {
                'token_in': token_in,
                'token_out': token_out,
                'amount_in': amount_in,
                'amount_out': amounts_out[-1] / 10**18,
                'price': amounts_out[-1] / amounts_out[0]
            }
        except Exception as e:
            raise ExchangeError(f"Failed to get price: {e}")

    async def get_ticker(self, symbol: str) -> dict:
        """Get ticker data (requires external price source)"""
        base, quote = symbol.split('/')

        # Use Chainlink or other price oracles
        # Simplified handling here
        try:
            # Get price (actual implementation needed)
            price = await self._get_oracle_price(base, quote)

            return {
                'symbol': symbol,
                'price': price,
                'change': 0.0,  # DEX usually doesn't provide historical changes
                'change_percent': 0.0,
                'high': price * 1.01,
                'low': price * 0.99,
                'volume': 0.0  # Requires indexer support
            }
        except Exception as e:
            raise ExchangeError(f"Failed to get ticker: {e}")

    async def get_orderbook(self, symbol: str, limit: int = 100) -> dict:
        """DEX usually doesn't provide traditional order books"""
        # Can estimate liquidity through simulated orders
        base, quote = symbol.split('/')

        # Get liquidity at different price points
        bids = []
        asks = []

        for i in range(10):
            # Simulate buy orders
            price_factor = 1 - (i * 0.001)
            price = await self._get_oracle_price(base, quote) * price_factor
            amount = await self._estimate_liquidity(base, quote, price, 'buy')
            if amount > 0:
                bids.append([price, amount])

            # Simulate sell orders
            price_factor = 1 + (i * 0.001)
            price = await self._get_oracle_price(base, quote) * price_factor
            amount = await self._estimate_liquidity(base, quote, price, 'sell')
            if amount > 0:
                asks.append([price, amount])

        return {
            'symbol': symbol,
            'bids': bids,
            'asks': asks
        }

    async def place_order(self, order: dict) -> dict:
        """Execute trade"""
        try:
            # Parse order
            token_in = order['symbol'].split('/')[0] if order['side'] == 'sell' else order['symbol'].split('/')[1]
            token_out = order['symbol'].split('/')[1] if order['side'] == 'sell' else order['symbol'].split('/')[0]
            amount_in = order['quantity']

            # Get trade path
            path = [token_in, token_out]

            # Calculate minimum output amount (slippage protection)
            amounts_out = self.router_contract.functions.getAmountsOut(
                int(amount_in * 10**18),
                path
            ).call()

            min_amount_out = int(amounts_out[-1] * 0.995 * 10**18)  # 0.5% slippage

            # Build transaction
            deadline = int(time.time()) + 300  # 5 minutes expiration
            function_call = self.router_contract.functions.swapExactTokensForTokens(
                int(amount_in * 10**18),
                min_amount_out,
                path,
                self.account,
                deadline
            )

            # Estimate gas
            gas_estimate = function_call.estimate_gas({'from': self.account})
            gas_price = self.w3.eth.gas_price

            # Build transaction
            transaction = function_call.build_transaction({
                'from': self.account,
                'gas': gas_estimate,
                'gasPrice': gas_price,
                'nonce': self.w3.eth.get_transaction_count(self.account)
            })

            # Sign transaction
            signed_txn = self.w3.eth.account.sign_transaction(transaction, self.private_key)

            # Send transaction
            tx_hash = self.w3.eth.send_raw_transaction(signed_txn.rawTransaction)

            # Wait for transaction confirmation
            receipt = self.w3.eth.wait_for_transaction_receipt(tx_hash)

            if receipt.status == 1:
                return {
                    'order_id': tx_hash.hex(),
                    'symbol': order['symbol'],
                    'status': 'FILLED',
                    'side': order['side'],
                    'quantity': amount_in,
                    'executed_quantity': amount_in,
                    'transaction_hash': tx_hash.hex(),
                    'gas_used': receipt.gasUsed,
                    'time': time.time()
                }
            else:
                raise ExchangeError("Transaction failed")

        except Exception as e:
            raise ExchangeError(f"Failed to place order: {e}")

    async def cancel_order(self, order_id: str, symbol: str) -> dict:
        """DEX transactions usually cannot be cancelled once submitted"""
        raise NotImplementedError("DEX orders cannot be cancelled once submitted")

    async def get_balance(self) -> dict:
        """Get account balance"""
        balances = {}

        # Get ETH balance
        eth_balance = self.w3.eth.get_balance(self.account)
        balances['ETH'] = {
            'free': eth_balance / 10**18,
            'locked': 0.0,
            'total': eth_balance / 10**18
        }

        # Need to iterate through held tokens here
        # Simplified handling

        return {'balances': balances}

    async def subscribe_to_market_data(self, symbols: List[str], callbacks: dict):
        """Subscribe to blockchain events"""
        # Listen to swap events
        swap_filter = self.router_contract.events.Swap.create_filter(
            fromBlock='latest'
        )

        while True:
            for event in swap_filter.get_new_entries():
                await self._handle_swap_event(event, callbacks)
            await asyncio.sleep(1)

    async def _handle_swap_event(self, event, callbacks: dict):
        """Handle swap event"""
        if 'trade' in callbacks:
            trade_data = {
                'symbol': f"{event.args.token0}/{event.args.token1}",
                'id': event.transactionHash.hex(),
                'price': event.args.amount1 / event.args.amount0,
                'quantity': event.args.amount0,
                'time': event.args.timestamp if hasattr(event.args, 'timestamp') else time.time()
            }
            await callbacks['trade'](trade_data)

    async def _get_oracle_price(self, base: str, quote: str) -> float:
        """Get price from price oracle"""
        # Actual implementation needs to integrate Chainlink or other oracles
        # Returns simulated price here
        return 1.0

    async def _estimate_liquidity(self, token_in: str, token_out: str, price: float, side: str) -> float:
        """Estimate liquidity"""
        # Actual liquidity calculation logic needed here
        return 1000.0
```

## 14.3 Connector Management

### 14.3.1 Connection Pool Management

```python
from typing import Dict, List
import random
from datetime import datetime, timedelta

class ExchangeConnectorPool:
    """Exchange connector pool"""

    def __init__(self):
        self.connectors = {}  # {exchange_name: [adapters]}
        self.primary_connections = {}  # {exchange_name: primary_adapter}
        self.health_checker = HealthChecker()
        self.rebalancer = LoadRebalancer()

    def add_connector(self, exchange: str, adapter: ExchangeAdapter, is_primary: bool = False):
        """Add connector"""
        if exchange not in self.connectors:
            self.connectors[exchange] = []

        self.connectors[exchange].append(adapter)

        if is_primary or exchange not in self.primary_connections:
            self.primary_connections[exchange] = adapter

    async def get_connector(self, exchange: str, use_primary: bool = True) -> ExchangeAdapter:
        """Get connector"""
        if exchange not in self.connectors:
            raise ValueError(f"No connectors configured for {exchange}")

        # Get healthy connectors
        healthy_connectors = [
            c for c in self.connectors[exchange]
            if await self.health_checker.is_healthy(c)
        ]

        if not healthy_connectors:
            raise ConnectionError(f"No healthy connectors available for {exchange}")

        # Prioritize primary connection
        if use_primary and self.primary_connections[exchange] in healthy_connectors:
            return self.primary_connections[exchange]

        # Load balancing
        return self.rebalancer.select_connector(healthy_connectors)

    async def initialize_all(self):
        """Initialize all connectors"""
        for exchange, adapters in self.connectors.items():
            for adapter in adapters:
                try:
                    await adapter.initialize()
                    logging.info(f"Initialized connector for {exchange}")
                except Exception as e:
                    logging.error(f"Failed to initialize {exchange} connector: {e}")

    async def close_all(self):
        """Close all connectors"""
        for exchange, adapters in self.connectors.items():
            for adapter in adapters:
                try:
                    await adapter.close()
                except Exception as e:
                    logging.error(f"Error closing {exchange} connector: {e}")

# Health checker
class HealthChecker:
    """Connector health checker"""

    def __init__(self, check_interval: float = 30.0):
        self.check_interval = check_interval
        self.health_status = {}  # {adapter_id: (status, last_check)}
        self.running = False

    async def start(self):
        """Start health check"""
        self.running = True
        asyncio.create_task(self._check_loop())

    async def stop(self):
        """Stop health check"""
        self.running = False

    async def is_healthy(self, adapter: ExchangeAdapter) -> bool:
        """Check if connector is healthy"""
        adapter_id = id(adapter)
        if adapter_id not in self.health_status:
            # First check
            await self._check_adapter_health(adapter)

        status, last_check = self.health_status[adapter_id]

        # Recheck if too long since last check
        if datetime.utcnow() - last_check > timedelta(seconds=self.check_interval):
            await self._check_adapter_health(adapter)
            status, _ = self.health_status[adapter_id]

        return status

    async def _check_loop(self):
        """Periodic check loop"""
        while self.running:
            await asyncio.sleep(self.check_interval)
            # Need to maintain connector list and check periodically here

    async def _check_adapter_health(self, adapter: ExchangeAdapter) -> bool:
        """Check single adapter health status"""
        try:
            # Try to get server time
            await adapter.get_server_time()
            self.health_status[id(adapter)] = (True, datetime.utcnow())
            return True
        except:
            self.health_status[id(adapter)] = (False, datetime.utcnow())
            return False

# Load balancer
class LoadRebalancer:
    """Load rebalancer"""

    def __init__(self, strategy: str = 'round_robin'):
        self.strategy = strategy
        self.counters = {}  # Used for round-robin counting

    def select_connector(self, connectors: List[ExchangeAdapter]) -> ExchangeAdapter:
        """Select connector"""
        if self.strategy == 'round_robin':
            return self._round_robin_select(connectors)
        elif self.strategy == 'random':
            return random.choice(connectors)
        elif self.strategy == 'least_latency':
            return self._least_latency_select(connectors)
        else:
            return connectors[0]

    def _round_robin_select(self, connectors: List[ExchangeAdapter]) -> ExchangeAdapter:
        """Round-robin selection"""
        exchange_id = id(connectors[0])  # Use first connector as key
        if exchange_id not in self.counters:
            self.counters[exchange_id] = 0

        index = self.counters[exchange_id] % len(connectors)
        self.counters[exchange_id] += 1
        return connectors[index]

    def _least_latency_select(self, connectors: List[ExchangeAdapter]) -> ExchangeAdapter:
        """Least latency selection"""
        # Actual latency measurement needed here
        # Simplified implementation
        return connectors[0]
```

### 14.3.2 Data Normalization

```python
from decimal import Decimal
from dataclasses import asdict

class DataNormalizer:
    """Data normalizer"""

    @staticmethod
    def normalize_ticker(raw_data: dict, exchange: str) -> dict:
        """Normalize ticker data"""
        normalized = {
            'exchange': exchange,
            'symbol': raw_data['symbol'],
            'timestamp': datetime.utcnow().isoformat(),
            'price': Decimal(str(raw_data['price'])),
            'bid': Decimal(str(raw_data.get('bid', 0))),
            'ask': Decimal(str(raw_data.get('ask', 0))),
            'bid_size': Decimal(str(raw_data.get('bid_size', 0))),
            'ask_size': Decimal(str(raw_data.get('ask_size', 0))),
            'volume_24h': Decimal(str(raw_data.get('volume', 0))),
            'high_24h': Decimal(str(raw_data.get('high', 0))),
            'low_24h': Decimal(str(raw_data.get('low', 0))),
            'change_24h': Decimal(str(raw_data.get('change', 0))),
            'change_pct_24h': Decimal(str(raw_data.get('change_percent', 0)))
        }

        return normalized

    @staticmethod
    def normalize_orderbook(raw_data: dict, exchange: str) -> dict:
        """Normalize order book data"""
        normalized = {
            'exchange': exchange,
            'symbol': raw_data['symbol'],
            'timestamp': datetime.utcnow().isoformat(),
            'bids': [
                {
                    'price': Decimal(str(price)),
                    'size': Decimal(str(size))
                }
                for price, size in raw_data['bids']
            ],
            'asks': [
                {
                    'price': Decimal(str(price)),
                    'size': Decimal(str(size))
                }
                for price, size in raw_data['asks']
            ]
        }

        return normalized

    @staticmethod
    def normalize_trade(raw_data: dict, exchange: str) -> dict:
        """Normalize trade data"""
        normalized = {
            'exchange': exchange,
            'symbol': raw_data['symbol'],
            'trade_id': str(raw_data.get('id', '')),
            'timestamp': datetime.fromtimestamp(
                raw_data['time'] / 1000 if isinstance(raw_data['time'], int)
                else raw_data['time']
            ).isoformat(),
            'price': Decimal(str(raw_data['price'])),
            'size': Decimal(str(raw_data['quantity'])),
            'side': 'buy' if raw_data.get('is_buyer_maker') == False else 'sell'
        }

        return normalized

    @staticmethod
    def normalize_order(raw_data: dict, exchange: str) -> dict:
        """Normalize order data"""
        normalized = {
            'exchange': exchange,
            'order_id': str(raw_data['order_id']),
            'client_order_id': raw_data.get('client_order_id', ''),
            'symbol': raw_data['symbol'],
            'status': raw_data['status'],
            'side': raw_data['side'],
            'type': raw_data['type'],
            'quantity': Decimal(str(raw_data['quantity'])),
            'executed_quantity': Decimal(str(raw_data['executed_quantity'])),
            'price': Decimal(str(raw_data['price'])) if raw_data['price'] else None,
            'avg_price': Decimal(str(raw_data.get('avg_price', 0))) if raw_data.get('avg_price') else None,
            'timestamp': datetime.fromtimestamp(
                raw_data['time'] / 1000 if isinstance(raw_data['time'], int)
                else raw_data['time']
            ).isoformat()
        }

        return normalized

# Unified interface
class UnifiedExchangeInterface:
    """Unified exchange interface"""

    def __init__(self, connector_pool: ExchangeConnectorPool):
        self.connector_pool = connector_pool
        self.normalizer = DataNormalizer()
        self.cache = DataCache()

    async def get_ticker(self, exchange: str, symbol: str) -> dict:
        """Get normalized ticker data"""
        connector = await self.connector_pool.get_connector(exchange)
        raw_data = await connector.get_ticker(symbol)
        return self.normalizer.normalize_ticker(raw_data, exchange)

    async def get_orderbook(self, exchange: str, symbol: str, limit: int = 100) -> dict:
        """Get normalized order book"""
        connector = await self.connector_pool.get_connector(exchange)
        raw_data = await connector.get_orderbook(symbol, limit)
        return self.normalizer.normalize_orderbook(raw_data, exchange)

    async def get_recent_trades(self, exchange: str, symbol: str, limit: int = 100) -> List[dict]:
        """Get normalized trade data"""
        connector = await self.connector_pool.get_connector(exchange)
        raw_trades = await connector.get_recent_trades(symbol, limit)
        return [self.normalizer.normalize_trade(trade, exchange) for trade in raw_trades]

    async def place_order(self, exchange: str, order: dict) -> dict:
        """Place order"""
        connector = await self.connector_pool.get_connector(exchange)
        raw_result = await connector.place_order(order)
        return self.normalizer.normalize_order(raw_result, exchange)

    async def cancel_order(self, exchange: str, order_id: str, symbol: str) -> dict:
        """Cancel order"""
        connector = await self.connector_pool.get_connector(exchange)
        return await connector.cancel_order(order_id, symbol)

    async def get_order_status(self, exchange: str, order_id: str, symbol: str) -> dict:
        """Query order status"""
        connector = await self.connector_pool.get_connector(exchange)
        raw_data = await connector.get_order_status(order_id, symbol)
        return self.normalizer.normalize_order(raw_data, exchange)

    async def get_balance(self, exchange: str) -> dict:
        """Get account balance"""
        connector = await self.connector_pool.get_connector(exchange)
        return await connector.get_balance()

    async def get_best_price(self, symbol: str, exchanges: List[str] = None) -> dict:
        """Get best price"""
        if not exchanges:
            exchanges = list(self.connector_pool.connectors.keys())

        best_bid = None
        best_ask = None
        best_bid_exchange = None
        best_ask_exchange = None

        for exchange in exchanges:
            try:
                ticker = await self.get_ticker(exchange, symbol)

                if ticker['bid'] > 0 and (best_bid is None or ticker['bid'] > best_bid):
                    best_bid = ticker['bid']
                    best_bid_exchange = exchange

                if ticker['ask'] > 0 and (best_ask is None or ticker['ask'] < best_ask):
                    best_ask = ticker['ask']
                    best_ask_exchange = exchange
            except Exception as e:
                logging.error(f"Failed to get ticker from {exchange}: {e}")

        return {
            'symbol': symbol,
            'best_bid': best_bid,
            'best_bid_exchange': best_bid_exchange,
            'best_ask': best_ask,
            'best_ask_exchange': best_ask_exchange,
            'spread': (best_ask - best_bid) if best_bid and best_ask else None
        }

# Data cache
class DataCache:
    """Data cache"""

    def __init__(self):
        self.cache = {}
        self.ttl = {}  # time to live

    def get(self, key: str) -> Optional[dict]:
        """Get cached data"""
        if key in self.cache:
            if datetime.utcnow() < self.ttl[key]:
                return self.cache[key]
            else:
                del self.cache[key]
                del self.ttl[key]
        return None

    def set(self, key: str, data: dict, ttl_seconds: int = 60):
        """Set cached data"""
        self.cache[key] = data
        self.ttl[key] = datetime.utcnow() + timedelta(seconds=ttl_seconds)

    def clear(self):
        """Clear cache"""
        self.cache.clear()
        self.ttl.clear()
```

## 14.4 Advanced Features

### 14.4.1 Smart Routing

```python
from typing import Tuple, List

class SmartRouter:
    """Smart order router"""

    def __init__(self, unified_interface: UnifiedExchangeInterface):
        self.interface = unified_interface
        self.liquidity_tracker = LiquidityTracker()

    async def route_order(self, order: dict) -> List[dict]:
        """Smart route order"""
        symbol = order['symbol']
        total_quantity = Decimal(str(order['quantity']))
        side = order['side']

        # Get liquidity from all exchanges
        exchanges = await self.liquidity_tracker.get_available_liquidity(symbol, side)

        if not exchanges:
            raise ValueError("No available exchanges for routing")

        # Determine optimal routing strategy
        if side == 'buy':
            # Buy order: find lowest ask price
            sorted_exchanges = sorted(exchanges, key=lambda x: x['ask_price'])
        else:
            # Sell order: find highest bid price
            sorted_exchanges = sorted(exchanges, key=lambda x: x['bid_price'], reverse=True)

        # Split order across different exchanges
        routed_orders = []
        remaining_quantity = total_quantity

        for exchange_data in sorted_exchanges:
            if remaining_quantity <= 0:
                break

            exchange = exchange_data['exchange']
            available_liquidity = exchange_data['available_quantity']

            order_size = min(remaining_quantity, available_liquidity)

            routed_order = {
                'exchange': exchange,
                'symbol': symbol,
                'side': side,
                'type': order.get('type', 'market'),
                'quantity': float(order_size)
            }

            # Limit orders need to specify price
            if order.get('type') == 'limit':
                if side == 'buy':
                    routed_order['price'] = float(exchange_data['ask_price'])
                else:
                    routed_order['price'] = float(exchange_data['bid_price'])

            routed_orders.append(routed_order)
            remaining_quantity -= order_size

        if remaining_quantity > 0:
            logging.warning(f"Could not route full order. Remaining: {remaining_quantity}")

        return routed_orders

    async def get_arbitrage_opportunities(self, symbol: str, min_profit: float = 0.001) -> List[dict]:
        """Find arbitrage opportunities"""
        opportunities = []
        exchanges = list(self.interface.connector_pool.connectors.keys())

        # Get prices from all exchanges
        prices = {}
        for exchange in exchanges:
            try:
                ticker = await self.interface.get_ticker(exchange, symbol)
                prices[exchange] = {
                    'bid': ticker['bid'],
                    'ask': ticker['ask']
                }
            except Exception as e:
                logging.error(f"Failed to get price from {exchange}: {e}")

        # Find arbitrage opportunities
        for buy_exchange in prices:
            for sell_exchange in prices:
                if buy_exchange == sell_exchange:
                    continue

                buy_price = prices[buy_exchange]['ask']
                sell_price = prices[sell_exchange]['bid']

                if buy_price > 0 and sell_price > buy_price:
                    profit = (sell_price - buy_price) / buy_price
                    if profit >= min_profit:
                        opportunities.append({
                            'symbol': symbol,
                            'buy_exchange': buy_exchange,
                            'sell_exchange': sell_exchange,
                            'buy_price': buy_price,
                            'sell_price': sell_price,
                            'profit_pct': profit * 100
                        })

        # Sort by profit
        opportunities.sort(key=lambda x: x['profit_pct'], reverse=True)
        return opportunities

class LiquidityTracker:
    """Liquidity tracker"""

    def __init__(self):
        self.depth_cache = {}

    async def get_available_liquidity(self, symbol: str, side: str) -> List[dict]:
        """Get available liquidity"""
        liquidity = []
        exchanges = ['binance', 'okx', 'huobi']  # Example exchanges

        for exchange in exchanges:
            try:
                # Get depth data
                depth = await self._get_depth_with_cache(exchange, symbol)

                if side == 'buy':
                    # Calculate ask side liquidity
                    total_qty = 0
                    weighted_price = 0
                    for price, qty in depth['asks'][:10]:  # Take top 10 levels
                        total_qty += qty
                        weighted_price += price * qty

                    if total_qty > 0:
                        avg_price = weighted_price / total_qty
                        liquidity.append({
                            'exchange': exchange,
                            'ask_price': avg_price,
                            'available_quantity': total_qty
                        })

                else:  # sell
                    # Calculate bid side liquidity
                    total_qty = 0
                    weighted_price = 0
                    for price, qty in depth['bids'][:10]:  # Take top 10 levels
                        total_qty += qty
                        weighted_price += price * qty

                    if total_qty > 0:
                        avg_price = weighted_price / total_qty
                        liquidity.append({
                            'exchange': exchange,
                            'bid_price': avg_price,
                            'available_quantity': total_qty
                        })

            except Exception as e:
                logging.error(f"Failed to get liquidity from {exchange}: {e}")

        return liquidity

    async def _get_depth_with_cache(self, exchange: str, symbol: str, cache_seconds: int = 1) -> dict:
        """Get cached depth data"""
        cache_key = f"{exchange}_{symbol}"
        now = time.time()

        if cache_key in self.depth_cache:
            data, timestamp = self.depth_cache[cache_key]
            if now - timestamp < cache_seconds:
                return data

        # Get new data from exchange
        connector = await self.interface.connector_pool.get_connector(exchange)
        depth = await connector.get_orderbook(symbol)
        self.depth_cache[cache_key] = (depth, now)
        return depth
```

## 14.5 Best Practices

### 14.5.1 Error Handling and Retry Mechanisms

```python
import random
from enum import Enum

class ErrorType(Enum):
    RATE_LIMIT = "rate_limit"
    NETWORK_ERROR = "network_error"
    INVALID_ORDER = "invalid_order"
    INSUFFICIENT_FUNDS = "insufficient_funds"
    EXCHANGE_ERROR = "exchange_error"
    UNKNOWN = "unknown"

class RetryStrategy:
    """Retry strategy"""

    def __init__(self, max_retries: int = 3, base_delay: float = 1.0, max_delay: float = 60.0):
        self.max_retries = max_retries
        self.base_delay = base_delay
        self.max_delay = max_delay

    async def execute_with_retry(self, func, *args, **kwargs):
        """Execute function with retry"""
        last_error = None

        for attempt in range(self.max_retries + 1):
            try:
                return await func(*args, **kwargs)
            except Exception as e:
                last_error = e
                error_type = self._classify_error(e)

                if attempt == self.max_retries:
                    break

                if error_type == ErrorType.INVALID_ORDER or error_type == ErrorType.INSUFFICIENT_FUNDS:
                    # These errors won't succeed with retry
                    break

                delay = self._calculate_delay(attempt)
                logging.warning(f"Attempt {attempt + 1} failed: {e}. Retrying in {delay}s...")
                await asyncio.sleep(delay)

        raise last_error

    def _classify_error(self, error: Exception) -> ErrorType:
        """Classify error type"""
        error_msg = str(error).lower()

        if 'rate limit' in error_msg or 'too many requests' in error_msg:
            return ErrorType.RATE_LIMIT
        elif 'network' in error_msg or 'connection' in error_msg or 'timeout' in error_msg:
            return ErrorType.NETWORK_ERROR
        elif 'invalid order' in error_msg or 'invalid symbol' in error_msg:
            return ErrorType.INVALID_ORDER
        elif 'insufficient' in error_msg or 'balance' in error_msg:
            return ErrorType.INSUFFICIENT_FUNDS
        else:
            return ErrorType.EXCHANGE_ERROR

    def _calculate_delay(self, attempt: int) -> float:
        """Calculate delay time (exponential backoff)"""
        delay = self.base_delay * (2 ** attempt) + random.uniform(0, 1)
        return min(delay, self.max_delay)
```

## Summary

This chapter深入探讨了交易所集成的各个方面，涵盖了：

1. **Adapter Pattern Design**
   - Unified interface abstraction
   - Rate limiting and error handling
   - Health check mechanisms

2. **Centralized Exchange Integration**
   - Binance and other mainstream exchange APIs
   - Authentication and signature mechanisms
   - WebSocket real-time data streams

3. **Decentralized Exchange Integration**
   - Uniswap and other DEX integration
   - Web3 interaction
   - Smart contract calls

4. **Connector Management**
   - Connection pool management
   - Load balancing
   - Failover

5. **Data Normalization**
   - Unified data formats
   - Caching mechanisms
   - Real-time data processing

6. **Advanced Features**
   - Smart routing
   - Arbitrage opportunity identification
   - Liquidity aggregation

通过本章的学习，读者可以构建灵活、可靠的交易所集成系统，支持多个交易所的交易和数据处理需求。

## References

- [Binance API Documentation](https://binance-docs.github.io/apidocs/)
- [Uniswap Documentation](https://docs.uniswap.org/)
- [Web3.py Documentation](https://web3py.readthedocs.io/)
- [CCXT Library](https://github.com/ccxt/ccxt)

## Runnable Code Entry (Source Build)

- Entry script: docs/tutorials_cn/code/chapter14/run_performance_analysis.py
- Key examples: docs/tutorials_cn/code/chapter14/performance_metrics.py, docs/tutorials_cn/code/chapter14/performance_charts.py

Dependency strategy: matplotlib is optional; when unavailable, chart output is rendered as text summaries.

Verified command:

```bash
uv run --python 3.13 python docs/tutorials_cn/code/chapter14/run_performance_analysis.py
```
