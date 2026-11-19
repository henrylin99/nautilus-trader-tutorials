---
title: "Chapter 10: Advanced Data Processing"
---

# Chapter 10: Advanced Data Processing

## Learning Objectives

After completing this chapter, you will:
- Master the use of Parquet data catalogs
- Learn to build efficient data pipelines
- Understand data indexing and query optimization
- Implement data version management
- Integrate external data sources

## 10.1 Parquet Data Catalogs

### 10.1.1 Data Catalog Structure

```python
"""
Parquet Data Catalog Management
"""

import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq
from pathlib import Path
from typing import Dict, List, Optional, Union
from datetime import datetime, timedelta
import numpy as np
from decimal import Decimal

from nautilus_trader.persistence.catalog import DataCatalog
from nautilus_trader.persistence.parquet import ParquetDataCatalog
from nautilus_trader.model.identifiers import Venue, InstrumentId
from nautilus_trader.model.data import Bar, QuoteTick, TradeTick
from nautilus_trader.persistence.wranglers import (
    BarDataWrangler,
    QuoteTickDataWrangler,
    TradeTickDataWrangler,
)


class DataDirectoryManager:
    """Data Directory Manager"""

    def __init__(self, base_path: Path):
        """
        Initialize Data Directory Manager

        Parameters
        ----------
        base_path : Path
            Base path for data directory
        """
        self.base_path = base_path
        self.catalog_path = base_path / "catalog"
        self.raw_path = base_path / "raw"
        self.processed_path = base_path / "processed"
        self.cache_path = base_path / "cache"

        # Create directory structure
        self._create_directory_structure()

        # Create data catalog
        self.catalog = self._create_data_catalog()

    def _create_directory_structure(self):
        """Create directory structure"""
        directories = [
            self.base_path,
            self.catalog_path,
            self.raw_path,
            self.processed_path,
            self.cache_path,
            self.raw_path / "bars",
            self.raw_path / "quotes",
            self.raw_path / "trades",
            self.processed_path / "daily",
            self.processed_path / "hourly",
            self.processed_path / "minute",
        ]

        for directory in directories:
            directory.mkdir(parents=True, exist_ok=True)

    def _create_data_catalog(self) -> DataCatalog:
        """Create data catalog"""
        config_path = self.base_path / "catalog_config.json"

        catalog = ParquetDataCatalog(
            path=str(self.catalog_path),
            fs_protocol="file",
        )

        # Save configuration
        config = {
            "path": str(self.catalog_path),
            "fs_protocol": "file",
            "created_at": datetime.utcnow().isoformat(),
        }

        import json
        with open(config_path, 'w') as f:
            json.dump(config, f, indent=2)

        return catalog

    def save_bars(
        self,
        bars: List[Bar],
        instrument_id: InstrumentId,
        bar_type,
        venue: Optional[Venue] = None,
    ):
        """
        Save bar data

        Parameters
        ----------
        bars : List[Bar]
            List of bar data
        instrument_id : InstrumentId
            Instrument ID
        bar_type
            Bar type
        venue : Venue, optional
            Venue
        """
        if not bars:
            return

        # Create data wrangler
        wrangler = BarDataWrangler(bar_type=bar_type, instrument=instrument_id)
        df = wrangler.process_bars_to_dataframe(bars)

        # Add metadata
        df['instrument_id'] = str(instrument_id)
        df['venue'] = venue.value if venue else None

        # Save to Parquet
        timestamp = bars[0].ts_event
        date_str = datetime.fromtimestamp(timestamp / 1e9).strftime('%Y-%m-%d')

        file_path = (
            self.processed_path / "daily" /
            venue.value if venue else "UNKNOWN" /
            str(instrument_id).replace(".", "_") /
            f"bars_{date_str}.parquet"
        )

        file_path.parent.mkdir(parents=True, exist_ok=True)

        # Write Parquet file
        df.to_parquet(file_path, engine='pyarrow')

        # Update catalog
        self._update_catalog_entry(file_path, 'bars', df)

        self.log.info(f"Saved {len(bars)} bars to: {file_path}")

    def load_bars(
        self,
        instrument_id: InstrumentId,
        start_date: datetime,
        end_date: datetime,
        venue: Optional[Venue] = None,
    ) -> pd.DataFrame:
        """
        Load bar data

        Parameters
        ----------
        instrument_id : InstrumentId
            Instrument ID
        start_date : datetime
            Start date
        end_date : datetime
            End date
        venue : Venue, optional
            Venue

        Returns
        -------
        pd.DataFrame
            Bar data
        """
        dfs = []
        current_date = start_date

        while current_date <= end_date:
            date_str = current_date.strftime('%Y-%m-%d')
            venue_str = venue.value if venue else "UNKNOWN"
            instrument_str = str(instrument_id).replace(".", "_")

            file_path = (
                self.processed_path / "daily" /
                venue_str /
                instrument_str /
                f"bars_{date_str}.parquet"
            )

            if file_path.exists():
                df = pd.read_parquet(file_path, engine='pyarrow')
                df['ts_event'] = pd.to_datetime(df['ts_event'])
                dfs.append(df)

            current_date += timedelta(days=1)

        if dfs:
            combined_df = pd.concat(dfs, ignore_index=True)
            combined_df = combined_df.sort_values('ts_event')
            return combined_df
        else:
            return pd.DataFrame()

    def _update_catalog_entry(self, file_path: Path, data_type: str, df: pd.DataFrame):
        """Update catalog entry"""
        # Use NautilusTrader's catalog update feature
        self.catalog.write_data(data_type, df)


# Usage example
class DataDirectoryExample(Strategy):
    """Data Directory Usage Example"""

    def __init__(self):
        """Initialize strategy"""
        super().__init__()

        # Create data directory manager
        data_path = Path("data/nautilus")
        self.data_manager = DataDirectoryManager(data_path)

    def process_and_save_data(self):
        """Process and save data"""
        # Generate sample data
        self._generate_sample_data()

        # Load from cache and save
        self._load_from_cache_and_save()

    def _generate_sample_data(self):
        """Generate sample data"""
        self.log.info("Generating sample data...")

        # Generate 1 month of 1-minute bar data
        bars = []
        start_time = datetime(2024, 1, 1)
        base_price = 50000.0

        for i in range(43 * 24 * 60):  # 43 days, every minute
            timestamp = start_time + timedelta(minutes=i)

            # Simple random walk
            price_change = np.random.normal(0, 50)
            current_price = base_price + price_change
            high_price = current_price + abs(np.random.normal(0, 20))
            low_price = current_price - abs(np.random.normal(0, 20))
            volume = np.random.uniform(100, 500)

            bar = Bar(
                bar_type=BarType(
                    instrument_id=InstrumentId.from_str("BTCUSDT.BINANCE"),
                    bar_spec=BarSpecification(
                        step=1,
                        aggregation=BarAggregation.MINUTE,
                        price_type=PriceType.LAST,
                    ),
                    aggregation_source="BINANCE",
                ),
                open=Price(str(current_price - 10)),
                high=Price(str(high_price)),
                low=Price(str(low_price)),
                close=Price(str(current_price)),
                volume=Quantity(str(volume)),
                ts_event=int(timestamp.timestamp() * 1e9),
                ts_init=int(timestamp.timestamp() * 1e9),
            )

            bars.append(bar)
            base_price = current_price

        # Save data
        self.data_manager.save_bars(
            bars,
            InstrumentId.from_str("BTCUSDT.BINANCE"),
            bar_type,
            Venue("BINANCE"),
        )

        self.log.info(f"Generated {len(bars)} sample bars")

    def _load_from_cache_and_save(self):
        """Load from cache and save"""
        # This demonstrates how to load data from NautilusTrader cache
        try:
            # Assume cache has data
            from nautilus_trader.core.fsm import FSMScope
            from nautilus_trader.test_kit.caches import TestCacheProvider

            cache_provider = TestCacheProvider()

            # Get data from cache (example)
            # In actual application, you need to adjust according to cache API

            # Load and save data for multiple assets
            instruments = [
                "BTCUSDT.BINANCE",
                "ETHUSDT.BINANCE",
                "ADAUSDT.BINANCE",
            ]

            for instrument_str in instruments:
                # This simulates data retrieval
                bars = self._load_from_cache(instrument_str)
                if bars:
                    instrument_id = InstrumentId.from_str(instrument_str)
                    self.data_manager.save_bars(bars, instrument_id, None, Venue("BINANCE"))
                    self.log.info(f"Saved data for {instrument_str}")

        except Exception as e:
            self.log.error(f"Failed to load data from cache: {e}")

    def _load_from_cache(self, instrument_str: str) -> List[Bar]:
        """Load bar data from cache (simulation)"""
        # This should be implemented according to actual cache API
        return []
```

### 10.1.2 Data Compression and Partitioning

```python
"""
Data Compression and Partitioning Strategy
"""

import pyarrow as pa
import pyarrow.parquet as pq
from typing import Dict, Any
import hashlib
import json


class DataCompression:
    """Data Compression Configuration"""

    @staticmethod
    def get_compression_config(data_type: str) -> Dict[str, Any]:
        """
        Get compression configuration

        Parameters
        ----------
        data_type : str
            Data type

        Returns
        -------
        Dict[str, Any]
            Compression configuration
        """
        base_config = {
            'compression': 'snappy',  # Fast compression, suitable for real-time scenarios
            'use_dictionary': True,
            'statistics': True,
            'index_columns': ['ts_event'],
        }

        # Adjust configuration based on data type
        if data_type == 'quotes':
            base_config.update({
                'compression': 'zstd',  # Higher compression ratio
                'compression_level': 3,
                'dictionary_pagesize': 32768,
            })
        elif data_type == 'bars':
            base_config.update({
                'compression': 'zstd',
                'compression_level': 5,
                'dictionary_pagesize': 65536,
            })
        elif data_type == 'trades':
            base_config.update({
                'compression': 'lz4',  # Fastest compression
                'compression_level': 1,
            })

        return base_config

    @staticmethod
    def write_optimized_parquet(
        df: pd.DataFrame,
        file_path: Path,
        data_type: str = 'default',
    ):
        """
        Write optimized Parquet file

        Parameters
        ----------
        df : pd.DataFrame
            Data frame
        file_path : Path
            File path
        data_type : str
            Data type
        """
        config = DataCompression.get_compression_config(data_type)

        # Convert data types to optimize storage
        table = pa.Table.from_pandas(df)

        # Apply configuration
        writer = pq.ParquetWriter(
            file_path,
            table.schema,
            **config
        )

        try:
            writer.write_table(table)
        except Exception as e:
            raise RuntimeError(f"Failed to write Parquet file: {e}")
        finally:
            writer.close()

        return file_path.stat().st_size


class DataPartitioning:
    """Data Partitioning Strategy"""

    @staticmethod
    def get_partition_path(
        base_path: Path,
        data_type: str,
        instrument_id: str,
        venue: str,
        timestamp: int,
        partition_type: str = 'daily',
    ) -> Path:
        """
        Generate partition path

        Parameters
        ----------
        base_path : Path
            Base path
        data_type : str
            Data type
        instrument_id : str
            Instrument ID
        venue : str
            Venue
        timestamp : int
            Timestamp (nanoseconds)
        partition_type : str
            Partition type

        Returns
        -------
        Path
            Partition path
        """
        dt = datetime.fromtimestamp(timestamp / 1e9)

        if partition_type == 'daily':
            year = dt.strftime('%Y')
            month = dt.strftime('%m')
            day = dt.strftime('%d')
            path = (
                base_path /
                data_type /
                venue /
                instrument_id.replace('.', '_') /
                f"year={year}" /
                f"month={month}" /
                f"day={day}"
            )
        elif partition_type == 'hourly':
            year = dt.strftime('%Y')
            month = dt.strftime('%m')
            day = dt.strftime('%d')
            hour = dt.strftime('%H')
            path = (
                base_path /
                data_type /
                venue /
                instrument_id.replace('.', '_') /
                f"year={year}" /
                f"month={month}" /
                f"day={day}" /
                f"hour={hour}"
            )
        elif partition_type == 'monthly':
            year = dt.strftime('%Y')
            month = dt.strftime('%m')
            path = (
                base_path /
                data_type /
                venue /
                instrument_id.replace('.', '_') /
                f"year={year}" /
                f"month={month}"
            )
        else:
            # No partitioning
            path = (
                base_path /
                data_type /
                venue /
                instrument_id.replace('.', '_')
            )

        return path

    @staticmethod
    def get_metadata(
        data: pd.DataFrame,
        source: str,
        version: str = "1.0",
    ) -> Dict[str, Any]:
        """
        Generate metadata

        Parameters
        ----------
        data : pd.DataFrame
            Data
        source : str
            Data source
        version : str
            Version

        Returns
        -------
        Dict[str, Any]
            Metadata
        """
        metadata = {
            'source': source,
            'version': version,
            'created_at': datetime.utcnow().isoformat(),
            'data_shape': data.shape,
            'columns': list(data.columns),
            'dtypes': {col: str(dtype) for col, dtype in data.dtypes.items()},
            'null_counts': data.isnull().sum().to_dict(),
        }

        # Calculate data fingerprint
        data_bytes = data.to_parquet_engine(engine='pyarrow')
        metadata['data_fingerprint'] = hashlib.md5(data_bytes).hexdigest()

        return metadata

    @staticmethod
    def save_with_metadata(
        df: pd.DataFrame,
        file_path: Path,
        metadata: Dict[str, Any],
    ):
        """
        Save data with metadata

        Parameters
        ----------
        df : pd.DataFrame
            Data
        file_path : Path
            File path
        metadata : Dict[str, Any]
            Metadata
        """
        # Save data
        DataCompression.write_optimized_parquet(df, file_path)

        # Save metadata
        metadata_path = file_path.with_suffix('.metadata.json')
        with open(metadata_path, 'w') as f:
            json.dump(metadata, f, indent=2, default=str)


class PartitionedDataStore:
    """Partitioned Data Storage"""

    def __init__(self, base_path: Path, partition_type: str = 'daily'):
        """
        Initialize partitioned data storage

        Parameters
        ----------
        base_path : Path
            Base path
        partition_type : str
            Partition type
        """
        self.base_path = base_path
        self.partition_type = partition_type
        self.metadata_store = {}

        # Create base directories
        (base_path / 'bars').mkdir(parents=True, exist_ok=True)
        (base_path / 'quotes').mkdir(parents=True, exist_ok=True)
        (base_path / 'trades').mkdir(parents=True, exist_ok=True)

    def save_partitioned_bars(
        self,
        bars: List[Bar],
        instrument_id: InstrumentId,
        venue: Venue,
        source: str = 'unknown',
    ):
        """
        Save partitioned bar data

        Parameters
        ----------
        bars : List[Bar]
            Bar data
        instrument_id : InstrumentId
            Instrument ID
        venue : Venue
            Venue
        source : str
            Data source
        """
        if not bars:
            return

        # Group by date
        date_groups = {}
        for bar in bars:
            dt = datetime.fromtimestamp(bar.ts_event / 1e9)
            date_key = dt.date()
            if date_key not in date_groups:
                date_groups[date_key] = []
            date_groups[date_key].append(bar)

        # Save each partition
        for date, date_bars in date_groups.items():
            # Get timestamp of first bar for generating partition path
            partition_path = DataPartitioning.get_partition_path(
                self.base_path,
                'bars',
                str(instrument_id),
                venue.value,
                date_bars[0].ts_event,
                self.partition_type
            )

            partition_path.mkdir(parents=True, exist_ok=True)

            # Convert to DataFrame
            wrangler = BarDataWrangler(
                bar_type=date_bars[0].bar_type,
                instrument=instrument_id
            )
            df = wrangler.process_bars_to_dataframe(date_bars)

            # Add additional metadata
            df['date'] = str(date)
            df['partition_path'] = str(partition_path)

            # Generate filename
            file_name = f"{date.strftime('%Y%m%d')}_{len(date_bars)}_bars.parquet"
            file_path = partition_path / file_name

            # Generate metadata
            metadata = DataPartitioning.get_metadata(df, source)
            metadata['instrument_id'] = str(instrument_id)
            metadata['venue'] = venue.value
            metadata['date'] = str(date)
            metadata['bar_count'] = len(date_bars)

            # Save data
            DataPartitioning.save_with_metadata(df, file_path, metadata)

            self.log.info(
                f"Saved partitioned bar data: {file_path.name}, "
                f"date: {date}, count: {len(date_bars)}"
            )

    def load_partitioned_bars(
        self,
        instrument_id: InstrumentId,
        start_date: datetime,
        end_date: datetime,
        venue: Venue,
    ) -> pd.DataFrame:
        """
        Load partitioned bar data

        Parameters
        ----------
        instrument_id : InstrumentId
            Instrument ID
        start_date : datetime
            Start date
        end_date : datetime
            End date
        venue : Venue
            Venue

        Returns
        -------
        pd.DataFrame
            Bar data
        """
        dfs = []
        current_date = start_date

        while current_date <= end_date:
            # Generate partition path
            partition_path = DataPartitioning.get_partition_path(
                self.base_path,
                'bars',
                str(instrument_id),
                venue.value,
                int(current_date.timestamp() * 1e9),
                self.partition_type
            )

            # Find all files in this partition
            if partition_path.exists():
                for file_path in partition_path.glob("*.parquet"):
                    try:
                        df = pd.read_parquet(file_path, engine='pyarrow')
                        dfs.append(df)
                    except Exception as e:
                        self.log.warning(f"Failed to read file {file_path}: {e}")

            current_date += timedelta(days=1)

        if dfs:
            combined_df = pd.concat(dfs, ignore_index=True)
            combined_df = combined_df.sort_values('ts_event')
            return combined_df
        else:
            return pd.DataFrame()

    def get_partition_info(self) -> Dict:
        """
        Get partition information

        Returns
        -------
        Dict
            Partition information statistics
        """
        info = {
            'total_files': 0,
            'total_size_bytes': 0,
            'data_types': {},
            'venues': set(),
            'instruments': set(),
            'partition_type': self.partition_type,
        }

        for data_type in ['bars', 'quotes', 'trades']:
            data_type_path = self.base_path / data_type

            if data_type_path.exists():
                for file_path in data_type_path.rglob("*.parquet"):
                    info['total_files'] += 1
                    info['total_size_bytes'] += file_path.stat().st_size

                    # Extract information from file path
                    path_parts = str(file_path.relative(data_type_path)).split('/')
                    if len(path_parts) >= 3:
                        info['venues'].add(path_parts[1])
                        info['instruments'].add(path_parts[2])

        # Convert sets to lists
        info['venues'] = list(info['venues'])
        info['instruments'] = list(info['instruments'])

        return info


# Example of using partitioned storage
class PartitionedDataExample(Strategy):
    """Partitioned Storage Example"""

    def __init__(self):
        """Initialize strategy"""
        super().__init__()

        # Create partitioned data storage
        data_path = Path("data/partitioned")
        self.data_store = PartitionedDataStore(data_path, partition_type='daily')

    def save_historical_data(self):
        """Save historical data"""
        self.log.info("Starting to save historical data...")

        # Get data from NautilusTrader cache
        try:
            # Get BTC/USDT data
            from nautilus_trader.test_kit.providers import TestInstrumentProvider

            # Generate test data
            self._generate_and_save_bars(
                TestInstrumentProvider.btcusdt_binance(),
                "TEST_SOURCE"
            )

            self._generate_and_save_bars(
                TestInstrumentProvider.ethusdt_binance(),
                "TEST_SOURCE"
            )

        except Exception as e:
            self.log.error(f"Failed to save historical data: {e}")

    def _generate_and_save_bars(self, instrument, source: str):
        """Generate and save bar data"""
        self.log.info(f"Generating test data for {instrument}...")

        bars = []
        start_time = datetime(2024, 1, 1)

        for i in range(24 * 60 * 7):  # One week of 1-minute data
            timestamp = start_time + timedelta(minutes=i)

            # Generate simulated bars
            base_price = 50000.0 if 'BTC' in str(instrument) else 3000.0
            price_change = np.random.normal(0, base_price * 0.001)

            bar = Bar(
                bar_type=BarType(
                    instrument_id=instrument.id,
                    bar_spec=BarSpecification(
                        step=1,
                        aggregation=BarAggregation.MINUTE,
                        price_type=PriceType.LAST,
                    ),
                    aggregation_source="BINANCE",
                ),
                open=Price(str(base_price + price_change)),
                high=Price(str(base_price + price_change + abs(np.random.normal(0, base_price * 0.002)))),
                low=Price(str(base_price + price_change - abs(np.random.normal(0, base_price * 0.002)))),
                close=Price(str(base_price + price_change + np.random.normal(0, base_price * 0.001))),
                volume=Quantity(str(np.random.uniform(100, 1000))),
                ts_event=int(timestamp.timestamp() * 1e9),
                ts_init=int(timestamp.timestamp() * 1e9),
            )

            bars.append(bar)

        # Save partitioned data
        self.data_store.save_partitioned_bars(
            bars,
            instrument.id,
            Venue("BINANCE"),
            source
        )

    def on_start(self):
        """Strategy startup"""
        self.log.info("Partitioned storage example started")
        self.save_historical_data()

    def on_bar(self, bar: Bar):
        """Process bar data"""
        # Can save new data to current day's partition
        current_date = datetime.fromtimestamp(bar.ts_event / 1e9)

        # Check if need to create new partition file
        if self.count_bars() % 1440 == 0:  # Save every 24 hours
            self.log.info(f"Preparing to save data for {current_date.date()}")

    def on_stop(self):
        """Strategy stop"""
        # Output partition information
        partition_info = self.data_store.get_partition_info()

        self.log.info("Partitioned storage statistics:")
        self.log.info(f"  Total files: {partition_info['total_files']}")
        self.log.info(f"  Total size: {partition_info['total_size_bytes'] / 1024 / 1024:.2f} MB")
        self.log.info(f"  Supported venues: {partition_info['venues']}")
        self.log.info(f"  Supported instruments: {len(partition_info['instruments'])}")
        self.log.info(f"  Partition type: {partition_info['partition_type']}")
```

## 10.2 Data Pipeline Construction

### 10.2.1 ETL Process

```python
"""
Data Pipeline Construction
"""

from abc import ABC, abstractmethod
from typing import Any, Dict, List, Optional
import asyncio
import aiofiles
from datetime import datetime, timedelta


class DataExtractor(ABC):
    """Data extractor base class"""

    @abstractmethod
    async def extract(self, config: Dict) -> Any:
        """Extract data"""
        pass

class DataTransformer(ABC):
    """Data transformer base class"""

    @abstractmethod
    async def transform(self, raw_data: Any) -> pd.DataFrame:
        """Transform data"""
        pass

class DataLoader(ABC):
    """Data loader base class"""

    @abstractmethod
    async def load(self, data: pd.DataFrame, config: Dict) -> Any:
        """Load data"""
        pass


class CryptoExchangeExtractor(DataExtractor):
    """Cryptocurrency exchange data extractor"""

    def __init__(self, exchange_client, instrument_id: InstrumentId):
        """
        Initialize

        Parameters
        ----------
        exchange_client
            Exchange client
        instrument_id : InstrumentId
            Instrument ID
        """
        self.client = exchange_client
        self.instrument_id = instrument_id

    async def extract(self, config: Dict) -> pd.DataFrame:
        """Extract historical bar data"""
        start_date = config['start_date']
        end_date = config['end_date']
        bar_spec = config['bar_spec']

        # Get data from exchange API
        # This needs to be implemented according to specific exchange API
        bars = []

        # Batch fetch data
        current_date = start_date
        while current_date <= end_date:
            batch_end = min(current_date + timedelta(days=30), end_date)

            # Get 30 days of data
            batch_bars = await self._fetch_bars_batch(
                current_date,
                batch_end,
                bar_spec
            )

            bars.extend(batch_bars)
            current_date = batch_end + timedelta(days=1)

        # Convert to DataFrame
        if bars:
            wrangler = BarDataWrangler(
                bar_type=BarType(
                    instrument_id=self.instrument_id,
                    bar_spec=bar_spec,
                    price_type=PriceType.LAST,
                    aggregation_source="EXCHANGE",
                ),
                instrument=self.instrument_id,
            )

            return wrangler.process_bars_to_dataframe(bars)
        else:
            return pd.DataFrame()

    async def _fetch_bars_batch(self, start: datetime, end: datetime, bar_spec) -> List[Bar]:
        """Fetch batch data"""
        # Implement specific data retrieval logic here
        # Needs to be adjusted according to exchange API
        return []


class DataValidator:
    """Data validator"""

    def __init__(self):
        """Initialize"""
        self.validation_rules = []

    def add_rule(self, rule: callable):
        """
        Add validation rule

        Parameters
        ----------
        rule : callable
            Validation rule function
        """
        self.validation_rules.append(rule)

    def validate(self, data: pd.DataFrame) -> Dict[str, Any]:
        """
        Validate data

        results = {
            'is_valid': True,
            'errors': [],
            'warnings': [],
            'stats': {},
        }

        # Apply all validation rules
        for rule in self.validation_rules:
            rule_result = rule(data)
            if not rule_result['is_valid']:
                results['is_valid'] = False
                results['errors'].append(rule_result['error'])
            else:
                if 'warning' in rule_result:
                    results['warnings'].append(rule_result['warning'])
                if 'stats' in rule_result:
                    results['stats'].update(rule_result['stats'])

        return results

    def create_standard_rules(self):
        """Create standard validation rules"""
        def check_empty_data(data):
            if data.empty:
                return {'is_valid': False, 'error': "Data is empty"}
            return {'is_valid': True}

        def check_timestamps(data):
            if 'ts_event' in data.columns:
                # Check if timestamps are in order
                if not data['ts_event'].is_monotonic_increasing:
                    return {
                        'is_valid': False,
                        'error': "Timestamps are not in increasing order",
                        'stats': {'out_of_order_count': data['ts_event'].diff().lt(timedelta(0)).sum()}
                    }
            return {'is_valid': True}

        def check_price_validity(data):
            price_cols = ['open', 'high', 'low', 'close']
            for col in price_cols:
                if col in data.columns:
                    if (data[col] <= 0).any():
                        return {
                            'is_valid': False,
                            'error': f"Column {col} contains non-positive values",
                            'stats': {'invalid_prices_count': (data[col] <= 0).sum()}
                        }

                    # Check price relationships
                    if 'high' in data.columns and 'low' in data.columns:
                        invalid_high = data['high'] < data['low']
                        if invalid_high.any():
                            return {
                                'is_valid': False,
                                'error': "High price is less than low price",
                                'stats': {'invalid_price_relations_count': invalid_high.sum()}
                            }

            return {'is_valid': True}

        def check_volume_validity(data):
            if 'volume' in data.columns:
                if (data['volume'] < 0).any():
                    return {
                        'is_valid': False,
                        'error': "Volume is negative",
                        'stats': {'negative_volume_count': (data['volume'] < 0).sum()}
                    }

                # Check abnormal volume
                q3 = data['volume'].quantile(0.75)
                outlier_threshold = q3 * 10

                outliers = data['volume'] > outlier_threshold
                if outliers.any():
                    return {
                        'is_valid': True,  # Still valid, but with warning
                        'warning': f"Detected abnormal volume (>10x Q3): {outliers.sum()} records",
                        'stats': {'volume_outliers': outliers.sum()}
                    }

            return {'is_valid': True}

        self.add_rule(check_empty_data)
        self.add_rule(check_timestamps)
        self.add_rule(check_price_validity)
        self.add_rule(check_volume_validity)


class DataPipeline:
    """Data pipeline"""

    def __init__(self):
        """Initialize data pipeline"""
        self.extractors = {}
        self.transformers = []
        self.validators = [DataValidator()]

    def add_extractor(self, name: str, extractor: DataExtractor):
        """Add data extractor"""
        self.extractors[name] = extractor

    def add_transformer(self, transformer: DataTransformer):
        """Add data transformer"""
        self.transformers.append(transformer)

    def add_validator(self, validator: DataValidator):
        """Add data validator"""
        self.validators.append(validator)

    async def run_pipeline(
        self,
        extraction_config: Dict,
        transformation_config: Optional[Dict] = None,
        validation_config: Optional[Dict] = None,
    ) -> Dict[str, Any]:
        """
        Run data pipeline

        Parameters
        ----------
        extraction_config : Dict
            Extraction configuration
        transformation_config : Dict
            Transformation configuration
        validation_config : Dict
            Validation configuration

        Returns
        -------
        Dict[str, Any]
            Pipeline results
        """
        results = {
            'extraction': {},
            'transformation': {},
            'validation': {},
            'total_records': 0,
            'valid_records': 0,
            'invalid_records': 0,
            'errors': [],
        }

        try:
            # 1. Extraction phase
            self.log.info("Starting data extraction...")
            for name, extractor in self.extractors.items():
                try:
                    raw_data = await extractor.extract(extraction_config.get(name, {}))
                    results['extraction'][name] = {
                        'status': 'success',
                        'record_count': len(raw_data) if hasattr(raw_data, '__len__') else 0
                    }
                except Exception as e:
                    results['extraction'][name] = {
                        'status': 'error',
                        'error': str(e),
                        'record_count': 0
                    }
                    results['errors'].append(f"Extractor {name} error: {e}")

            # 2. Transformation phase
            if self.transformers:
                self.log.info("Starting data transformation...")
                for i, transformer in enumerate(self.transformers):
                    try:
                        if isinstance(raw_data, pd.DataFrame) and not raw_data.empty:
                            transformed_data = await transformer.transform(raw_data)
                            results['transformation'][f'transformer_{i}'] = {
                                'status': 'success',
                                'input_records': len(raw_data),
                                'output_records': len(transformed_data)
                            }
                            raw_data = transformed_data
                    except Exception as e:
                        results['transformation'][f'transformer_{i}'] = {
                            'status': 'error',
                            'error': str(e)
                        }
                        results['errors'].append(f"Transformer {i} error: {e}")

            # 3. Validation phase
            self.log.info("Starting data validation...")
            results['validation'] = {}

            if hasattr(raw_data, '__len__'):
                total_records = len(raw_data)
                results['total_records'] = total_records

            if total_records > 0:
                for validator in self.validators:
                    try:
                        validator_result = validator.validate(raw_data)
                        results['validation'][f'validator_{len(results["validation"])}'] = {
                            'is_valid': validator_result['is_valid'],
                            'error_count': len(validator_result['errors']),
                            'warning_count': len(validator_result['warnings']),
                            'stats': validator_result['stats']
                        }

                        if validator_result['is_valid']:
                            results['valid_records'] += total_records
                        else:
                            results['invalid_records'] += total_records
                            results['errors'].extend(validator_result['errors'])
                    except Exception as e:
                        results['errors'].append(f"Validator error: {e}")

        except Exception as e:
            results['errors'].append(f"Pipeline execution error: {e}")
            self.log.error(f"Data pipeline failed: {e}")

        # Output summary
        self.log.info("Data pipeline execution completed:")
        self.log.info(f"  Extractor count: {len(self.extractors)}")
        self.log.info(f"  Transformer count: {len(self.transformers)}")
        self.log.info(f"  Validator count: {len(self.validators)}")
        self.log.info(f"  Total records: {results['total_records']}")
        self.log.info(f"  Valid records: {results['valid_records']}")
        self.log.info(f"  Invalid records: {results['invalid_records']}")
        if results['errors']:
            self.log.error(f"  Error count: {len(results['errors'])}")

        return results


# Data pipeline usage example
class DataPipelineExample:
    """Data pipeline example"""

    def __init__(self):
        """Initialize"""
        # Create data pipeline
        self.pipeline = DataPipeline()

        # Create validator
        validator = DataValidator()
        validator.create_standard_rules()
        self.pipeline.add_validator(validator)

    async def run_etl_pipeline(self):
        """Run ETL pipeline"""
        self.log.info("Starting to run ETL data pipeline...")

        # Configure extraction
        extraction_config = {
            'start_date': datetime(2024, 1, 1),
            'end_date': datetime(2024, 12, 31),
            'bar_spec': BarSpecification(
                step=1,
                adjustment=0,
                aggregation=BarAggregation.MINUTE,
                price_type=PriceType.LAST,
            ),
        }

        # Run pipeline
        results = await self.pipeline.run_pipeline(
            extraction_config=extraction_config
        )

        return results

    async def on_bar(self, bar: Bar):
        """Process bar data"""
        pass

    async def run_periodic_pipeline(self):
        """Run pipeline periodically"""
        # Can set timer to periodically execute data updates
        while True:
            self.log.info("Running data pipeline periodically...")
            await self.run_etl_pipeline()
            # Run once a day
            await asyncio.sleep(24 * 60 * 60)


# Using data pipeline in strategy
class PipelineIntegratedStrategy(Strategy):
    """Strategy with integrated data pipeline"""

    def __init__(self):
        """Initialize strategy"""
        super().__init__()
        self.data_pipeline = DataPipelineExample()
        self.pipeline_results = None

    def on_start(self):
        """Strategy startup"""
        self.log.info("Data pipeline integrated strategy started")
        # Run pipeline in background
        asyncio.create_task(self._run_pipeline_background())

    async def _run_pipeline_background(self):
        """Run pipeline in background"""
        self.pipeline_results = await self.data_pipeline.run_etl_pipeline()
        self._log_pipeline_results()

    def _log_pipeline_results(self):
        """Log pipeline results"""
        if self.pipeline_results:
            self.log.info("Data pipeline background task completed")
            self.log.info(f"Processed records: {self.pipeline_results['total_records']}")
            self.log.info(f"Valid records: {self.pipeline_results['valid_records']}")
        else:
            self.log.warning("Pipeline results are empty")

    def on_bar(self, bar: Bar):
        """Process bar data"""
        # Use data processed through the pipeline
        pass
```

## 10.3 Data Indexing and Query Optimization

### 10.3.1 Index Construction

```python
"""
Data Indexing and Query Optimization
"""

import polars as pl
import pandas as pd
import sqlite3
from typing import List, Dict, Tuple, Optional
from pathlib import Path
import bisect
import time


class DataIndex:
    """Data Index Management"""

    def __init__(self, index_path: Path):
        """
        Initialize data index

        Parameters
        ----------
        index_path : Path
            Index file path
        """
        self.index_path = index_path
        self.index_path.mkdir(parents=True, exist_ok=True)

        # Index files
        self.bars_index_path = index_path / "bars_index.pkl"
        self.quotes_index_path = index_path / "quotes_index.pkl"
        self.trades_index_path = index_path / "trades_index.pkl"

        # Memory indices
        self.bars_index = {}
        self.quotes_index = {}
        self.trades_index = {}

        # Timestamp indices (for fast lookup)
        self.bars_time_index = []
        self.quotes_time_index = []
        self.trades_time_index = []

        # Load existing indices
        self._load_indices()

    def _load_indices(self):
        """Load existing indices"""
        try:
            import pickle

            # Load bars index
            if self.bars_index_path.exists():
                with open(self.bars_index_path, 'rb') as f:
                    self.bars_index = pickle.load(f)
                    self._rebuild_time_index(self.bars_index, 'bars')

            # Load quotes index
            if self.quotes_index_path.exists():
                with open(self.quotes_index_path, 'rb') as f:
                    self.quotes_index = pickle.load(f)
                    self._rebuild_time_index(self.quotes_index, 'quotes')

            # Load trades index
            if self.trades_index_path.exists():
                with open(self.trades_index_path, 'rb') as f:
                    self.trades_index = pickle.load(f)
                    self._rebuild_time_index(self.trades_index, 'trades')

        except Exception as e:
            self.log.warning(f"Failed to load indices: {e}")

    def _rebuild_time_index(self, index_dict: dict, data_type: str):
        """Rebuild timestamp index"""
        time_list = []
        for key, info in index_dict.items():
            time_list.append((info['start_time'], key))

        time_list.sort()

        if data_type == 'bars':
            self.bars_time_index = time_list
        elif data_type == 'quotes':
            self.quotes_time_index = time_list
        elif data_type == 'trades':
            self.trades_time_index = time_list

    def save_indices(self):
        """Save indices"""
        try:
            import pickle

            # Save bars index
            with open(self.bars_index_path, 'wb') as f:
                pickle.dump(self.bars_index, f)

            # Save quotes index
            with open(self.quotes_index_path, 'wb') as f:
                pickle.dump(self.quotes_index, f)

            # Save trades index
            with open(self.trades_index_path, 'wb') as f:
                pickle.dump(self.trades_index, f)

        except Exception as e:
            self.log.error(f"Failed to save indices: {e}")

    def add_bars_index(
        self,
        file_path: str,
        start_time: int,
        end_time: int,
        bar_count: int,
        instrument_id: str,
    ):
        """Add bars index"""
        self.bars_index[file_path] = {
            'file_path': file_path,
            'start_time': start_time,
            'end_time': end_time,
            'bar_count': bar_count,
            'instrument_id': instrument_id,
            'created_at': time.time(),
        }

        # Update time index
        bisect.insort(self.bars_time_index, (start_time, file_path))

    def add_quotes_index(
        self,
        file_path: str,
        start_time: int,
        end_time: int,
        quote_count: int,
        instrument_id: str,
    ):
        """Add quotes index"""
        self.quotes_index[file_path] = {
            'file_path': file_path,
            'start_time': start_time,
            'end_time': end_time,
            'quote_count': quote_count,
            'instrument_id': instrument_id,
            'created_at': time.time(),
        }

        bisect.insert(self.quotes_time_index, (start_time, file_path))

    def add_trades_index(
        self,
        file_path: str,
        start_time: int,
        end_time: int,
        trade_count: int,
        instrument_id: str,
    ):
        """Add trades index"""
        self.trades_index[file_path] = {
            'file_path': file_path,
            'start_time': start_time,
            'end_time': end_time,
            'trade_count': trade_count,
            'instrument_id': instrument_id,
            'created_at': time.time(),
        }

        bisect.insort(self.trades_time_index, (start_time, file_path))

    def query_bars(
        self,
        instrument_id: Optional[str] = None,
        start_time: Optional[int] = None,
        end_time: Optional[int] = None,
        limit: Optional[int] = None,
    ) -> List[str]:
        """
        Query bar data

        Parameters
        ----------
        instrument_id : str, optional
            Instrument ID
        start_time : int, optional
            Start timestamp
        end_time : int, optional
            End timestamp
        limit : int, optional
            Limit count

        Returns
        -------
        List[str]
            List of matching file paths
        """
        results = []

        for file_path, info in self.bars_index.items():
            # Check instrument
            if instrument_id and info['instrument_id'] != instrument_id:
                continue

            # Check time range
            if start_time and info['end_time'] < start_time:
                continue
            if end_time and info['start_time'] > end_time:
                continue

            results.append(file_path)

        # Apply limit
        if limit:
            results = results[:limit]

        return results

    def query_quotes(
        self,
        instrument_id: Optional[str] = None,
        start_time: Optional[int] = None,
        end_time: Optional[int] = None,
        limit: Optional[int] = None,
    ) -> List[str]:
        """Query quote data"""
        results = []

        for file_path, info in self.quotes_index.items():
            if instrument_id and info['instrument_id'] != instrument_id:
                continue
            if start_time and info['end_time'] < start_time:
                continue
            if end_time and info['start_time'] > end_time:
                continue

            results.append(file_path)

        if limit:
            results = results[:limit]

        return results

    def query_trades(
        self,
        instrument_id: Optional[str] = None,
        start_time: Optional[int] = None,
        end_time: Optional[int] = None,
        limit: Optional[int] = None,
    ) -> List[str]:
        """Query trade data"""
        results = []

        for file_path, info in self.trades_index.items():
            if instrument_id and info['instrument_id'] != instrument_id:
                continue
            if start_time and info['end_time'] < start_time:
                continue
            if end_time and info['start_time'] > end_time:
                continue

            results.append(file_path)

        if limit:
            results = results[:limit]

        return results

    def get_index_stats(self) -> Dict:
        """Get index statistics"""
        return {
            'bars_count': len(self.bars_index),
            'quotes_count': len(self.quotes_index),
            'trades_count': len(self.trades_index),
            'bars_files': len(self.bars_time_index),
            'quotes_files': len(self.quotes_time_index),
            'trades_files': len(self.trades_time_index),
        }


class QueryOptimizer:
    """Query Optimizer"""

    def __init__(self, data_index: DataIndex):
        """
        Initialize query optimizer

        Parameters
        ----------
        data_index : DataIndex
            Data index
        """
        self.data_index = data_index
        self.query_cache = {}

    def query_with_cache(
        self,
        query_type: str,
        **kwargs
    ) -> List[str]:
        """
        Query with cache

        parameters
        ----------
        query_type : str
            Query type
        **kwargs
            Query parameters

        Returns
        -------
        List[str]
            Result list
        """
        # Create cache key
        cache_key = self._create_cache_key(query_type, **kwargs)

        # Check cache
        if cache_key in self.query_cache:
            cache_time, cached_result = self.query_cache[cache_key]
            # Cache validity period: 1 minute
            if time.time() - cache_time < 60:
                return cached_result

        # Execute query
        if query_type == 'bars':
            result = self.data_index.query_bars(**kwargs)
        elif query_type == 'quotes':
            result = self.data_index.query_quotes(**kwargs)
        elif query_type == 'trades':
            result = self.data_index.query_trades(**kwargs)
        else:
            result = []

        # Update cache
        self.query_cache[cache_key] = (time.time(), result)

        return result

    def _create_cache_key(self, query_type: str, **kwargs) -> str:
        """Create cache key"""
        key_parts = [query_type]

        # Sort parameters alphabetically
        for key in sorted(kwargs.keys()):
            value = kwargs[key]
            if value is not None:
                key_parts.append(f"{key}={value}")

        return "|".join(key_parts)


class FastDataLoader:
    """Fast Data Loader"""

    def __init__(self, data_index: DataIndex):
        """
        Initialize fast loader

        Parameters
        ----------
        data_index : DataIndex
            Data index
        """
        self.data_index = data_index
        self.query_optimizer = QueryOptimizer(data_index)
        self.file_cache = {}

    def load_bars_fast(
        self,
        instrument_id: str,
        start_date: datetime,
        end_date: datetime,
        use_polars: bool = True,
    ) -> pd.DataFrame:
        """
        Fast load bar data

        parameters
        ----------
        instrument_id : str
            Instrument ID
        start_date : datetime
            Start date
        end_date : datetime
            End date
        use_polars : bool
            Whether to use Polars

        Returns
        -------
        pd.DataFrame
            Bar data
        """
        start_time_ns = int(start_date.timestamp() * 1e9)
        end_time_ns = int(end_date.timestamp() * 1e9)

        # Use optimized query
        file_paths = self.query_optimizer.query_with_cache(
            'bars',
            instrument_id=instrument_id,
            start_time=start_time_ns,
            end_time=end_time_ns,
        )

        if not file_paths:
            return pd.DataFrame()

        # Load multiple files in parallel
        all_dataframes = []

        for file_path in file_paths:
            if file_path in self.file_cache:
                # Use cached data
                df = self.file_cache[file_path]
            else:
                # Load and cache data
                try:
                    if use_polars:
                        df = pl.read_parquet(file_path).to_pandas()
                    else:
                        df = pd.read_parquet(file_path, engine='pyarrow')
                    self.file_cache[file_path] = df
                except Exception as e:
                    self.log.warning(f"Failed to load file {file_path}: {e}")
                    continue

                # Ensure cache doesn't get too large
                if len(self.file_cache) > 100:
                    # Remove oldest cache item
                    oldest_key = min(self.file_cache.keys())
                    del self.file_cache[oldest_key]

            if not df.empty:
                # Filter data range
                df['ts_event'] = pd.to_datetime(df['ts_event'])
                filtered_df = df[
                    (df['ts_event'] >= start_date) &
                    (df['ts_event'] <= end_date)
                ]
                if not filtered_df.empty:
                    all_dataframes.append(filtered_df)

        # Merge all data
        if all_dataframes:
            if use_polars and len(all_dataframes) > 1:
                # Use Polars for efficient merging
                combined = pl.concat([
                    pl.from_pandas(df) for df in all_dataframes
                ]).to_pandas()
                # Sort by timestamp
                combined = combined.sort('ts_event')
            else:
                # Use Pandas merge
                combined = pd.concat(all_dataframes, ignore_index=True)
                combined = combined.sort_values('ts_event')

            return combined

        return pd.DataFrame()


# Example of using indices
class IndexedDataExample(Strategy):
    """Indexed Data Example"""

    def __init__(self):
        """Initialize"""
        super().__init__()

        # Create data index
        index_path = Path("data/index")
        self.data_index = DataIndex(index_path)

        # Create fast loader
        self.fast_loader = FastDataLoader(self.data_index)

    def load_historical_data(self):
        """Load historical data"""
        self.log.info("Loading historical data...")

        # Scan directory and build index
        data_path = Path("data/processed")
        if data_path.exists():
            self._scan_and_index_directory(data_path)

        # Example: Load BTC/USDT data
        instrument_id = "BTCUSDT.BINANCE"
        start_date = datetime(2024, 1, 1)
        end_date = datetime(2024, 3, 31)

        start_time = time.time()
        bars_df = self.fast_loader.load_bars_fast(
            instrument_id=instrument_id,
            start_date=start_date,
            end_date=end_date,
        )
        load_time = time.time() - start_time

        if not bars_df.empty:
            self.log.info(f"Successfully loaded {len(bars_df)} bars")
            self.log.info(f"Load time: {load_time:.2f} seconds")
            self.log.info(f"Time range: {bars_df['ts_event'].min()} to {bars_df['ts_event'].max()}")
        else:
            self.log.warning("No data found")

    def _scan_and_index_directory(self, data_path: Path):
        """Scan directory and build index"""
        for data_type in ['bars', 'quotes', 'trades']:
            type_path = data_path / data_type

            if not type_path.exists():
                continue

            for file_path in type_path.rglob("*.parquet"):
                try:
                    # Read metadata
                    metadata_path = file_path.with_suffix('.metadata.json')
                    if metadata_path.exists():
                        import json
                        with open(metadata_path, 'r') as f:
                            metadata = json.load(f)

                        # Add to index
                        timestamp = metadata.get('created_at', '2024-01-01')
                        created_dt = datetime.fromisoformat(timestamp)

                        if data_type == 'bars':
                            self.data_index.add_bars_index(
                                str(file_path),
                                created_dt.timestamp(),
                                created_dt.timestamp(),
                                metadata.get('bar_count', 0),
                                metadata.get('instrument_id', 'UNKNOWN'),
                            )
                        elif data_type == 'quotes':
                            self.data_index.add_quotes_index(
                                str(file_path),
                                created_dt.timestamp(),
                                created_dt.timestamp(),
                                metadata.get('quote_count', 0),
                                metadata.get('instrument_id', 'UNKNOWN'),
                            )
                        elif data_type == 'trades':
                            self.data_index.add_trades_index(
                                str(file_path),
                                created_dt.timestamp(),
                                created_dt.timestamp(),
                                metadata.get('trade_count', 0),
                                metadata.get('instrument_id', 'UNKNOWN'),
                            )

                except Exception as e:
                    self.log.warning(f"Failed to process file {file_path}: {e}")

        # Save index
        self.data_index.save_indices()

        # Output statistics
        stats = self.data_index.get_index_stats()
        self.log.info("Index statistics:")
        self.log.info(f"  Bar data: {stats['bars_count']} files")
        self.log.info(f"  Quote data: {stats['quotes_count']} files")
        self.log.info(f"  Trade data: {stats['trades_count']} files")

    def on_start(self):
        """Strategy startup"""
        self.log.info("Indexed data example strategy started")
        self.load_historical_data()

    def on_bar(self, bar: Bar):
        """Process bar data"""
        pass

    def on_stop(self):
        """Strategy stop"""
        # Save index
        self.data_index.save_indices()

        # Output statistics
        stats = self.data_index.get_index_stats()
        self.log.info("Final index statistics:")
        self.log.info(f"  Total files: {stats['bars_count'] + stats['quotes_count'] + stats['trades_count']}")
        self.log.info(f"  Cache size: {len(self.fast_loader.file_cache)} files")
```

## 10.4 Next Steps

In this chapter, we learned:

1. Using Parquet data catalogs
2. Data compression and partitioning strategies
3. Building ETL data pipelines
4. Data indexing and query optimization
5. High-performance data loading

In the next chapter, we will learn:

1. Message bus and Actor model
2. Inter-component communication
3. Concurrent processing patterns
4. Error handling and recovery
5. System architecture best practices

## 10.5 Summary

### Key Points
1. Parquet format provides efficient storage and compression
2. Partitioning strategies improve data query performance
3. Data indexing significantly speeds up data retrieval
4. ETL pipelines automate data processing
5. Caching mechanisms reduce repeated loading time

### Best Practices
1. Choose appropriate compression algorithms based on data type
2. Regularly clean and optimize indices
3. Implement caching strategies to avoid repeated loading
4. Use monitoring to track data quality
5. Establish data backup mechanisms

## 10.6 References

- [Apache Arrow Documentation](https://arrow.apache.org/docs/)
- [Parquet Format Specification](https://parquet.apache.org/)
- [PyArrow Usage Guide](https://pandas.pydata.org/docs/reference/api/pandas.io.parquet.html)
- [Polars High-Performance Data Analysis](https://pola-rs.github.io/docs/)