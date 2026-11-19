# 第10章：高级数据处理

## 学习目标

通过本章学习，您将：
- 掌握 Parquet 数据目录的使用
- 学会构建高效的数据流水线
- 理解数据索引和查询优化
- 实现数据版本管理
- 集成外部数据源

## 10.1 Parquet 数据目录

### 10.1.1 数据目录结构

```python
"""
Parquet 数据目录管理
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
    """数据目录管理器"""

    def __init__(self, base_path: Path):
        """
        初始化数据目录管理器

        Parameters
        ----------
        base_path : Path
            数据目录基础路径
        """
        self.base_path = base_path
        self.catalog_path = base_path / "catalog"
        self.raw_path = base_path / "raw"
        self.processed_path = base_path / "processed"
        self.cache_path = base_path / "cache"

        # 创建目录结构
        self._create_directory_structure()

        # 创建数据目录
        self.catalog = self._create_data_catalog()

    def _create_directory_structure(self):
        """创建目录结构"""
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
        """创建数据目录"""
        config_path = self.base_path / "catalog_config.json"

        catalog = ParquetDataCatalog(
            path=str(self.catalog_path),
            fs_protocol="file",
        )

        # 保存配置
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
        保存K线数据

        Parameters
        ----------
        bars : List[Bar]
            K线数据列表
        instrument_id : InstrumentId
            交易工具ID
        bar_type
            K线类型
        venue : Venue, optional
            交易所
        """
        if not bars:
            return

        # 创建数据整理器
        wrangler = BarDataWrangler(bar_type=bar_type, instrument=instrument_id)
        df = wrangler.process_bars_to_dataframe(bars)

        # 添加元数据
        df['instrument_id'] = str(instrument_id)
        df['venue'] = venue.value if venue else None

        # 保存到Parquet
        timestamp = bars[0].ts_event
        date_str = datetime.fromtimestamp(timestamp / 1e9).strftime('%Y-%m-%d')

        file_path = (
            self.processed_path / "daily" /
            venue.value if venue else "UNKNOWN" /
            str(instrument_id).replace(".", "_") /
            f"bars_{date_str}.parquet"
        )

        file_path.parent.mkdir(parents=True, exist_ok=True)

        # 写入Parquet文件
        df.to_parquet(file_path, engine='pyarrow')

        # 更新目录
        self._update_catalog_entry(file_path, 'bars', df)

        self.log.info(f"保存 {len(bars)} 根K线到: {file_path}")

    def load_bars(
        self,
        instrument_id: InstrumentId,
        start_date: datetime,
        end_date: datetime,
        venue: Optional[Venue] = None,
    ) -> pd.DataFrame:
        """
        加载K线数据

        Parameters
        ----------
        instrument_id : InstrumentId
            交易工具ID
        start_date : datetime
            开始日期
        end_date : datetime
            结束日期
        venue : Venue, optional
            交易所

        Returns
        -------
        pd.DataFrame
            K线数据
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
        """更新目录条目"""
        # 使用NautilusTrader的目录更新功能
        self.catalog.write_data(data_type, df)


# 使用示例
class DataDirectoryExample(Strategy):
    """数据目录使用示例"""

    def __init__(self):
        """初始化策略"""
        super().__init__()

        # 创建数据目录管理器
        data_path = Path("data/nautilus")
        self.data_manager = DataDirectoryManager(data_path)

    def process_and_save_data(self):
        """处理并保存数据"""
        # 生成示例数据
        self._generate_sample_data()

        # 从缓存加载并保存
        self._load_from_cache_and_save()

    def _generate_sample_data(self):
        """生成示例数据"""
        self.log.info("生成示例数据...")

        # 生成1个月的1分钟K线数据
        bars = []
        start_time = datetime(2024, 1, 1)
        base_price = 50000.0

        for i in range(43 * 24 * 60):  # 43天，每分钟
            timestamp = start_time + timedelta(minutes=i)

            # 简单的随机游走
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

        # 保存数据
        self.data_manager.save_bars(
            bars,
            InstrumentId.from_str("BTCUSDT.BINANCE"),
            bar_type,
            Venue("BINANCE"),
        )

        self.log.info(f"生成了 {len(bars)} 根示例K线")

    def _load_from_cache_and_save(self):
        """从缓存加载并保存"""
        # 这里演示如何从NautilusTrader缓存加载数据
        try:
            # 假设缓存中有数据
            from nautilus_trader.core.fsm import FSMScope
            from nautilus_trader.test_kit.caches import TestCacheProvider

            cache_provider = TestCacheProvider()

            # 从缓存获取数据（示例）
            # 实际应用中，您需要根据缓存API进行调整

            # 加载并保存多个资产的数据
            instruments = [
                "BTCUSDT.BINANCE",
                "ETHUSDT.BINANCE",
                "ADAUSDT.BINANCE",
            ]

            for instrument_str in instruments:
                # 这里模拟数据获取
                bars = self._load_from_cache(instrument_str)
                if bars:
                    instrument_id = InstrumentId.from_str(instrument_str)
                    self.data_manager.save_bars(bars, instrument_id, None, Venue("BINANCE"))
                    self.log.info(f"保存 {instrument_str} 的数据")

        except Exception as e:
            self.log.error(f"从缓存加载数据失败: {e}")

    def _load_from_cache(self, instrument_str: str) -> List[Bar]:
        """从缓存加载K线数据（模拟）"""
        # 这里应该根据实际的缓存API实现
        return []
```

### 10.1.2 数据压缩和分区

```python
"""
数据压缩和分区策略
"""

import pyarrow as pa
import pyarrow.parquet as pq
from typing import Dict, Any
import hashlib
import json


class DataCompression:
    """数据压缩配置"""

    @staticmethod
    def get_compression_config(data_type: str) -> Dict[str, Any]:
        """
        获取压缩配置

        Parameters
        ----------
        data_type : str
            数据类型

        Returns
        -------
        Dict[str, Any]
            压缩配置
        """
        base_config = {
            'compression': 'snappy',  # 快速压缩，适合实时场景
            'use_dictionary': True,
            'statistics': True,
            'index_columns': ['ts_event'],
        }

        # 根据数据类型调整配置
        if data_type == 'quotes':
            base_config.update({
                'compression': 'zstd',  # 更高的压缩率
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
                'compression': 'lz4',  # 最快的压缩
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
        写入优化的Parquet文件

        Parameters
        ----------
        df : pd.DataFrame
            数据框
        file_path : Path
            文件路径
        data_type : str
            数据类型
        """
        config = DataCompression.get_compression_config(data_type)

        # 转换数据类型以优化存储
        table = pa.Table.from_pandas(df)

        # 应用配置
        writer = pq.ParquetWriter(
            file_path,
            table.schema,
            **config
        )

        try:
            writer.write_table(table)
        except Exception as e:
            raise RuntimeError(f"写入Parquet文件失败: {e}")
        finally:
            writer.close()

        return file_path.stat().st_size


class DataPartitioning:
    """数据分区策略"""

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
        生成分区路径

        Parameters
        ----------
        base_path : Path
            基础路径
        data_type : str
            数据类型
        instrument_id : str
            交易工具ID
        venue : str
            交易所
        timestamp : int
            时间戳（纳秒）
        partition_type : str
            分区类型

        Returns
        -------
        Path
            分区路径
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
            # 不分区
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
        生成元数据

        Parameters
        ----------
        data : pd.DataFrame
            数据
        source : str
            数据源
        version : str
            版本

        Returns
        -------
        Dict[str, Any]
            元数据
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

        # 计算数据指纹
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
        保存数据与元数据

        Parameters
        ----------
        df : pd.DataFrame
            数据
        file_path : Path
            文件路径
        metadata : Dict[str, Any]
            元数据
        """
        # 保存数据
        DataCompression.write_optimized_parquet(df, file_path)

        # 保存元数据
        metadata_path = file_path.with_suffix('.metadata.json')
        with open(metadata_path, 'w') as f:
            json.dump(metadata, f, indent=2, default=str)


class PartitionedDataStore:
    """分区数据存储"""

    def __init__(self, base_path: Path, partition_type: str = 'daily'):
        """
        初始化分区数据存储

        Parameters
        ----------
        base_path : Path
            基础路径
        partition_type : str
            分区类型
        """
        self.base_path = base_path
        self.partition_type = partition_type
        self.metadata_store = {}

        # 创建基础目录
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
        保存分区K线数据

        Parameters
        ----------
        bars : List[Bar]
            K线数据
        instrument_id : InstrumentId
            交易工具ID
        venue : Venue
            交易所
        source : str
            数据源
        """
        if not bars:
            return

        # 按日期分组
        date_groups = {}
        for bar in bars:
            dt = datetime.fromtimestamp(bar.ts_event / 1e9)
            date_key = dt.date()
            if date_key not in date_groups:
                date_groups[date_key] = []
            date_groups[date_key].append(bar)

        # 保存每个分区
        for date, date_bars in date_groups.items():
            # 获取第一个bar的时间戳用于生成分区路径
            partition_path = DataPartitioning.get_partition_path(
                self.base_path,
                'bars',
                str(instrument_id),
                venue.value,
                date_bars[0].ts_event,
                self.partition_type
            )

            partition_path.mkdir(parents=True, exist_ok=True)

            # 转换为DataFrame
            wrangler = BarDataWrangler(
                bar_type=date_bars[0].bar_type,
                instrument=instrument_id
            )
            df = wrangler.process_bars_to_dataframe(date_bars)

            # 添加额外元数据
            df['date'] = str(date)
            df['partition_path'] = str(partition_path)

            # 生成文件名
            file_name = f"{date.strftime('%Y%m%d')}_{len(date_bars)}_bars.parquet"
            file_path = partition_path / file_name

            # 生成元数据
            metadata = DataPartitioning.get_metadata(df, source)
            metadata['instrument_id'] = str(instrument_id)
            metadata['venue'] = venue.value
            metadata['date'] = str(date)
            metadata['bar_count'] = len(date_bars)

            # 保存数据
            DataPartitioning.save_with_metadata(df, file_path, metadata)

            self.log.info(
                f"保存分区K线数据: {file_path.name}, "
                f"日期: {date}, 数量: {len(date_bars)}"
            )

    def load_partitioned_bars(
        self,
        instrument_id: InstrumentId,
        start_date: datetime,
        end_date: datetime,
        venue: Venue,
    ) -> pd.DataFrame:
        """
        加载分区K线数据

        Parameters
 ----------
        instrument_id : InstrumentId
            交易工具ID
        start_date : datetime
            开始日期
        end_date : datetime
            结束日期
        venue : Venue
            交易所

        Returns
        -------
        pd.DataFrame
            K线数据
        """
        dfs = []
        current_date = start_date

        while current_date <= end_date:
            # 生成分区路径
            partition_path = DataPartitioning.get_partition_path(
                self.base_path,
                'bars',
                str(instrument_id),
                venue.value,
                int(current_date.timestamp() * 1e9),
                self.partition_type
            )

            # 查找该分区下的所有文件
            if partition_path.exists():
                for file_path in partition_path.glob("*.parquet"):
                    try:
                        df = pd.read_parquet(file_path, engine='pyarrow')
                        dfs.append(df)
                    except Exception as e:
                        self.log.warning(f"读取文件失败 {file_path}: {e}")

            current_date += timedelta(days=1)

        if dfs:
            combined_df = pd.concat(dfs, ignore_index=True)
            combined_df = combined_df.sort_values('ts_event')
            return combined_df
        else:
            return pd.DataFrame()

    def get_partition_info(self) -> Dict:
        """
        获取分区信息

        Returns
        -------
        Dict
            分区信息统计
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

                    # 从文件路径提取信息
                    path_parts = str(file_path.relative(data_type_path)).split('/')
                    if len(path_parts) >= 3:
                        info['venues'].add(path_parts[1])
                        info['instruments'].add(path_parts[2])

        # 转换set为list
        info['venues'] = list(info['venues'])
        info['instruments'] = list(info['instruments'])

        return info


# 使用分区存储的示例
class PartitionedDataExample(Strategy):
    """分区存储示例"""

    def __init__(self):
        """初始化策略"""
        super().__init__()

        # 创建分区数据存储
        data_path = Path("data/partitioned")
        self.data_store = PartitionedDataStore(data_path, partition_type='daily')

    def save_historical_data(self):
        """保存历史数据"""
        self.log.info("开始保存历史数据...")

        # 从NautilusTrader缓存获取数据
        try:
            # 获取BTC/USDT数据
            from nautilus_trader.test_kit.providers import TestInstrumentProvider

            # 生成测试数据
            self._generate_and_save_bars(
                TestInstrumentProvider.btcusdt_binance(),
                "TEST_SOURCE"
            )

            self._generate_and_save_bars(
                TestInstrumentProvider.ethusdt_binance(),
                "TEST_SOURCE"
            )

        except Exception as e:
            self.log.error(f"保存历史数据失败: {e}")

    def _generate_and_save_bars(self, instrument, source: str):
        """生成并保存K线数据"""
        self.log.info(f"生成 {instrument} 的测试数据...")

        bars = []
        start_time = datetime(2024, 1, 1)

        for i in range(24 * 60 * 7):  # 一周的1分钟数据
            timestamp = start_time + timedelta(minutes=i)

            # 生成模拟K线
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

        # 保存分区数据
        self.data_store.save_partitioned_bars(
            bars,
            instrument.id,
            Venue("BINANCE"),
            source
        )

    def on_start(self):
        """策略启动"""
        self.log.info("分区存储示例启动")
        self.save_historical_data()

    def on_bar(self, bar: Bar):
        """处理K线数据"""
        # 可以将新数据保存到当日分区
        current_date = datetime.fromtimestamp(bar.ts_event / 1e9)

        # 检查是否需要创建新的分区文件
        if self.count_bars() % 1440 == 0:  # 每24小时保存一次
            self.log.info(f"准备保存 {current_date.date()} 的数据")

    def on_stop(self):
        """策略停止"""
        # 输出分区信息
        partition_info = self.data_store.get_partition_info()

        self.log.info("分区存储统计:")
        self.log.info(f"  总文件数: {partition_info['total_files']}")
        self.log.info(f"  总大小: {partition_info['total_size_bytes'] / 1024 / 1024:.2f} MB")
        self.log.info(f"  支持的交易所: {partition_info['venues']}")
        self.log.info(f"  支持的工具: {len(partition_info['instruments'])} 个")
        self.log.info(f"  分区类型: {partition_info['partition_type']}")
```

## 10.2 数据流水线构建

### 10.2.1 ETL 流程

```python
"""
数据流水线构建
"""

from abc import ABC, abstractmethod
from typing import Any, Dict, List, Optional
import asyncio
import aiofiles
from datetime import datetime, timedelta


class DataExtractor(ABC):
    """数据提取器基类"""

    @abstractmethod
    async def extract(self, config: Dict) -> Any:
        """提取数据"""
        pass

class DataTransformer(ABC):
    """数据转换器基类"""

    @abstractmethod
    async def transform(self, raw_data: Any) -> pd.DataFrame:
        """转换数据"""
        pass

class DataLoader(ABC):
    """数据加载器基类"""

    @abstractmethod
    async def load(self, data: pd.DataFrame, config: Dict) -> Any:
        """加载数据"""
        pass


class CryptoExchangeExtractor(DataExtractor):
    """加密货币交易所数据提取器"""

    def __init__(self, exchange_client, instrument_id: InstrumentId):
        """
        初始化

        Parameters
        ----------
        exchange_client
            交易所客户端
        instrument_id : InstrumentId
            交易工具ID
        """
        self.client = exchange_client
        self.instrument_id = instrument_id

    async def extract(self, config: Dict) -> pd.DataFrame:
        """提取历史K线数据"""
        start_date = config['start_date']
        end_date = config['end_date']
        bar_spec = config['bar_spec']

        # 从交易所API获取数据
        # 这里需要根据具体交易所的API实现
        bars = []

        # 分批获取数据
        current_date = start_date
        while current_date <= end_date:
            batch_end = min(current_date + timedelta(days=30), end_date)

            # 获取30天的数据
            batch_bars = await self._fetch_bars_batch(
                current_date,
                batch_end,
                bar_spec
            )

            bars.extend(batch_bars)
            current_date = batch_end + timedelta(days=1)

        # 转换为DataFrame
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
        """获取批量数据"""
        # 这里实现具体的数据获取逻辑
        # 需要根据交易所API调整
        return []


class DataValidator:
    """数据验证器"""

    def __init__(self):
        """初始化"""
        self.validation_rules = []

    def add_rule(self, rule: callable):
        """
        添加验证规则

        Parameters
        ----------
        rule : callable
            验证规则函数
        """
        self.validation_rules.append(rule)

    def validate(self, data: pd.DataFrame) -> Dict[str, Any]:
        """
        验证数据

        results = {
            'is_valid': True,
            'errors': [],
            'warnings': [],
            'stats': {},
        }

        # 应用所有验证规则
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
        """创建标准验证规则"""
        def check_empty_data(data):
            if data.empty:
                return {'is_valid': False, 'error': "数据为空"}
            return {'is_valid': True}

        def check_timestamps(data):
            if 'ts_event' in data.columns:
                # 检查时间戳是否有序
                if not data['ts_event'].is_monotonic_increasing:
                    return {
                        'is_valid': False,
                        'error': "时间戳不是递增顺序",
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
                            'error': f"列 {col} 包含非正值",
                            'stats': {'invalid_prices_count': (data[col] <= 0).sum()}
                        }

                    # 检查价格关系
                    if 'high' in data.columns and 'low' in data.columns:
                        invalid_high = data['high'] < data['low']
                        if invalid_high.any():
                            return {
                                'is_valid': False,
                                'error': "最高价小于最低价",
                                'stats': {'invalid_price_relations_count': invalid_high.sum()}
                            }

            return {'is_valid': True}

        def check_volume_validity(data):
            if 'volume' in data.columns:
                if (data['volume'] < 0).any():
                    return {
                        'is_valid': False,
                        'error': "成交量为负",
                        'stats': {'negative_volume_count': (data['volume'] < 0).sum()}
                    }

                # 检查异常成交量
                q3 = data['volume'].quantile(0.75)
                outlier_threshold = q3 * 10

                outliers = data['volume'] > outlier_threshold
                if outliers.any():
                    return {
                        'is_valid': True,  # 仍然有效，但有警告
                        'warning': f"检测到异常成交量 (>10倍Q3): {outliers.sum()}条",
                        'stats': {'volume_outliers': outliers.sum()}
                    }

            return {'is_valid': True}

        self.add_rule(check_empty_data)
        self.add_rule(check_timestamps)
        self.add_rule(check_price_validity)
        self.add_rule(check_volume_validity)


class DataPipeline:
    """数据流水线"""

    def __init__(self):
        """初始化数据流水线"""
        self.extractors = {}
        self.transformers = []
        self.validators = [DataValidator()]

    def add_extractor(self, name: str, extractor: DataExtractor):
        """添加数据提取器"""
        self.extractors[name] = extractor

    def add_transformer(self, transformer: DataTransformer):
        """添加数据转换器"""
        self.transformers.append(transformer)

    def add_validator(self, validator: DataValidator):
        """添加数据验证器"""
        self.validators.append(validator)

    async def run_pipeline(
        self,
        extraction_config: Dict,
        transformation_config: Optional[Dict] = None,
        validation_config: Optional[Dict] = None,
    ) -> Dict[str, Any]:
        """
        运行数据流水线

        Parameters
        ----------
        extraction_config : Dict
            提取配置
        transformation_config : Dict
            转换配置
        validation_config : Dict
            验证配置

        Returns
        -------
        Dict[str, Any]
            流水线结果
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
            # 1. 提取阶段
            self.log.info("开始数据提取...")
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
                    results['errors'].append(f"提取器 {name} 错误: {e}")

            # 2. 转换阶段
            if self.transformers:
                self.log.info("开始数据转换...")
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
                        results['errors'].append(f"转换器 {i} 错误: {e}")

            # 3. 验证阶段
            self.log.info("开始数据验证...")
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
                        results['errors'].append(f"验证器错误: {e}")

        except Exception as e:
            results['errors'].append(f"流水线执行错误: {e}")
            self.log.error(f"数据流水线失败: {e}")

        # 输出总结
        self.log.info("数据流水线执行完成:")
        self.log.info(f"  提取器数量: {len(self.extractors)}")
        self.log.info(f"  转换器数量: {len(self.transformers)}")
        self.log.info(f"  验证器数量: {len(self.validators)}")
        self.log.info(f"  总记录数: {results['total_records']}")
        self.log.info(f"  有效记录数: {results['valid_records']}")
        self.log.info(f"  无效记录数: {results['invalid_records']}")
        if results['errors']:
            self.log.error(f"  错误数量: {len(results['errors'])}")

        return results


# 数据流水线使用示例
class DataPipelineExample:
    """数据流水线示例"""

    def __init__(self):
        """初始化"""
        # 创建数据流水线
        self.pipeline = DataPipeline()

        # 创建验证器
        validator = DataValidator()
        validator.create_standard_rules()
        self.pipeline.add_validator(validator)

    async def run_etl_pipeline(self):
        """运行ETL流水线"""
        self.log.info("开始运行ETL数据流水线...")

        # 配置提取
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

        # 运行流水线
        results = await self.pipeline.run_pipeline(
            extraction_config=extraction_config
        )

        return results

    async def on_bar(self, bar: Bar):
        """处理K线数据"""
        pass

    async def run_periodic_pipeline(self):
        """定期运行流水线"""
        # 可以设置定时器，定期执行数据更新
        while True:
            self.log.info("定期运行数据流水线...")
            await self.run_etl_pipeline()
            # 每天运行一次
            await asyncio.sleep(24 * 60 * 60)


# 策略中使用数据流水线
class PipelineIntegratedStrategy(Strategy):
    """集成数据流水线的策略"""

    def __init__(self):
        """初始化策略"""
        super().__init__()
        self.data_pipeline = DataPipelineExample()
        self.pipeline_results = None

    def on_start(self):
        """策略启动"""
        self.log.info("数据流水线集成策略启动")
        # 在后台运行流水线
        asyncio.create_task(self._run_pipeline_background())

    async def _run_pipeline_background(self):
        """在后台运行流水线"""
        self.pipeline_results = await self.data_pipeline.run_etl_pipeline()
        self._log_pipeline_results()

    def _log_pipeline_results(self):
        """记录流水线结果"""
        if self.pipeline_results:
            self.log.info("数据流水线后台任务完成")
            self.log.info(f"处理记录数: {self.pipeline_results['total_records']}")
            self.log.info(f"有效记录数: {self.pipeline_results['valid_records']}")
        else:
            self.log.warning("流水线结果为空")

    def on_bar(self, bar: Bar):
        """处理K线数据"""
        # 使用经过流水线处理的数据
        pass
```

## 10.3 数据索引和查询优化

### 10.3.1 索引构建

```python
"""
数据索引和查询优化
"""

import polars as pl
import pandas as pd
import sqlite3
from typing import List, Dict, Tuple, Optional
from pathlib import Path
import bisect
import time


class DataIndex:
    """数据索引管理"""

    def __init__(self, index_path: Path):
        """
        初始化数据索引

        Parameters
        ----------
        index_path : Path
            索引文件路径
        """
        self.index_path = index_path
        self.index_path.mkdir(parents=True, exist_ok=True)

        # 索引文件
        self.bars_index_path = index_path / "bars_index.pkl"
        self.quotes_index_path = index_path / "quotes_index.pkl"
        self.trades_index_path = index_path / "trades_index.pkl"

        # 内存索引
        self.bars_index = {}
        self.quotes_index = {}
        self.trades_index = {}

        # 时间戳索引（用于快速查找）
        self.bars_time_index = []
        self.quotes_time_index = []
        self.trades_time_index = []

        # 加载现有索引
        self._load_indices()

    def _load_indices(self):
        """加载现有索引"""
        try:
            import pickle

            # 加载K线索引
            if self.bars_index_path.exists():
                with open(self.bars_index_path, 'rb') as f:
                    self.bars_index = pickle.load(f)
                    self._rebuild_time_index(self.bars_index, 'bars')

            # 加载报价索引
            if self.quotes_index_path.exists():
                with open(self.quotes_index_path, 'rb') as f:
                    self.quotes_index = pickle.load(f)
                    self._rebuild_time_index(self.quotes_index, 'quotes')

            # 加载成交索引
            if self.trades_index_path.exists():
                with open(self.trades_index_path, 'rb') as f:
                    self.trades_index = pickle.load(f)
                    self._rebuild_time_index(self.trades_index, 'trades')

        except Exception as e:
            self.log.warning(f"加载索引失败: {e}")

    def _rebuild_time_index(self, index_dict: dict, data_type: str):
        """重建时间戳索引"""
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
        """保存索引"""
        try:
            import pickle

            # 保存K线索引
            with open(self.bars_index_path, 'wb') as f:
                pickle.dump(self.bars_index, f)

            # 保存报价索引
            with open(self.quotes_index_path, 'wb') as f:
                pickle.dump(self.quotes_index, f)

            # 保存成交索引
            with open(self.trades_index_path, 'wb') as f:
                pickle.dump(self.trades_index, f)

        except Exception as e:
            self.log.error(f"保存索引失败: {e}")

    def add_bars_index(
        self,
        file_path: str,
        start_time: int,
        end_time: int,
        bar_count: int,
        instrument_id: str,
    ):
        """添加K线索引"""
        self.bars_index[file_path] = {
            'file_path': file_path,
            'start_time': start_time,
            'end_time': end_time,
            'bar_count': bar_count,
            'instrument_id': instrument_id,
            'created_at': time.time(),
        }

        # 更新时间索引
        bisect.insort(self.bars_time_index, (start_time, file_path))

    def add_quotes_index(
        self,
        file_path: str,
        start_time: int,
        end_time: int,
        quote_count: int,
        instrument_id: str,
    ):
        """添加报价索引"""
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
        """添加成交索引"""
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
        查询K线数据

        Parameters
        ----------
        instrument_id : str, optional
            交易工具ID
        start_time : int, optional
            开始时间戳
        end_time : int, optional
            结束时间戳
        limit : int, optional
            限制数量

        Returns
        -------
        List[str]
            匹配的文件路径列表
        """
        results = []

        for file_path, info in self.bars_index.items():
            # 检查交易工具
            if instrument_id and info['instrument_id'] != instrument_id:
                continue

            # 检查时间范围
            if start_time and info['end_time'] < start_time:
                continue
            if end_time and info['start_time'] > end_time:
                continue

            results.append(file_path)

        # 应用限制
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
        """查询报价数据"""
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
        """查询成交数据"""
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
        """获取索引统计"""
        return {
            'bars_count': len(self.bars_index),
            'quotes_count': len(self.quotes_index),
            'trades_count': len(self.trades_index),
            'bars_files': len(self.bars_time_index),
            'quotes_files': len(self.quotes_time_index),
            'trades_files': len(self.trades_time_index),
        }


class QueryOptimizer:
    """查询优化器"""

    def __init__(self, data_index: DataIndex):
        """
        初始化查询优化器

        Parameters
        ----------
        data_index : DataIndex
            数据索引
        """
        self.data_index = data_index
        self.query_cache = {}

    def query_with_cache(
        self,
        query_type: str,
        **kwargs
    ) -> List[str]:
        """
        带缓存的查询

        parameters
        ----------
        query_type : str
            查询类型
        **kwargs
            查询参数

        Returns
        -------
        List[str]
            结果列表
        """
        # 创建缓存键
        cache_key = self._create_cache_key(query_type, **kwargs)

        # 检查缓存
        if cache_key in self.query_cache:
            cache_time, cached_result = self.query_cache[cache_key]
            # 缓存有效期：1分钟
            if time.time() - cache_time < 60:
                return cached_result

        # 执行查询
        if query_type == 'bars':
            result = self.data_index.query_bars(**kwargs)
        elif query_type == 'quotes':
            result = self.data_index.query_quotes(**kwargs)
        elif query_type == 'trades':
            result = self.data_index.query_trades(**kwargs)
        else:
            result = []

        # 更新缓存
        self.query_cache[cache_key] = (time.time(), result)

        return result

    def _create_cache_key(self, query_type: str, **kwargs) -> str:
        """创建缓存键"""
        key_parts = [query_type]

        # 按字母顺序排序参数
        for key in sorted(kwargs.keys()):
            value = kwargs[key]
            if value is not None:
                key_parts.append(f"{key}={value}")

        return "|".join(key_parts)


class FastDataLoader:
    """快速数据加载器"""

    def __init__(self, data_index: DataIndex):
        """
        初始化快速加载器

        Parameters
        ----------
        data_index : DataIndex
            数据索引
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
        快速加载K线数据

        parameters
        ----------
        instrument_id : str
            交易工具ID
        start_date : datetime
            开始日期
        end_date : datetime
            结束日期
        use_polars : bool
            是否使用Polars

        Returns
        -------
        pd.DataFrame
            K线数据
        """
        start_time_ns = int(start_date.timestamp() * 1e9)
        end_time_ns = int(end_date.timestamp() * 1e9)

        # 使用优化的查询
        file_paths = self.query_optimizer.query_with_cache(
            'bars',
            instrument_id=instrument_id,
            start_time=start_time_ns,
            end_time=end_time_ns,
        )

        if not file_paths:
            return pd.DataFrame()

        # 并行加载多个文件
        all_dataframes = []

        for file_path in file_paths:
            if file_path in self.file_cache:
                # 使用缓存的数据
                df = self.file_cache[file_path]
            else:
                # 加载并缓存数据
                try:
                    if use_polars:
                        df = pl.read_parquet(file_path).to_pandas()
                    else:
                        df = pd.read_parquet(file_path, engine='pyarrow')
                    self.file_cache[file_path] = df
                except Exception as e:
                    self.log.warning(f"加载文件失败 {file_path}: {e}")
                    continue

                # 确保缓存不会太大
                if len(self.file_cache) > 100:
                    # 清除最老的缓存项
                    oldest_key = min(self.file_cache.keys())
                    del self.file_cache[oldest_key]

            if not df.empty:
                # 过滤数据范围
                df['ts_event'] = pd.to_datetime(df['ts_event'])
                filtered_df = df[
                    (df['ts_event'] >= start_date) &
                    (df['ts_event'] <= end_date)
                ]
                if not filtered_df.empty:
                    all_dataframes.append(filtered_df)

        # 合并所有数据
        if all_dataframes:
            if use_polars and len(all_dataframes) > 1:
                # 使用Polars进行高效合并
                combined = pl.concat([
                    pl.from_pandas(df) for df in all_dataframes
                ]).to_pandas()
                # 按时间戳排序
                combined = combined.sort('ts_event')
            else:
                # 使用Pandas合并
                combined = pd.concat(all_dataframes, ignore_index=True)
                combined = combined.sort_values('ts_event')

            return combined

        return pd.DataFrame()


# 使用索引的示例
class IndexedDataExample(Strategy):
    """索引数据示例"""

    def __init__(self):
        """初始化"""
        super().__init__()

        # 创建数据索引
        index_path = Path("data/index")
        self.data_index = DataIndex(index_path)

        # 创建快速加载器
        self.fast_loader = FastDataLoader(self.data_index)

    def load_historical_data(self):
        """加载历史数据"""
        self.log.info("加载历史数据...")

        # 从目录扫描并建立索引
        data_path = Path("data/processed")
        if data_path.exists():
            self._scan_and_index_directory(data_path)

        # 示例：加载BTC/USDT数据
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
            self.log.info(f"成功加载 {len(bars_df)} 根K线")
            self.log.info(f"加载时间: {load_time:.2f} 秒")
            self.log.info(f"时间范围: {bars_df['ts_event'].min()} 到 {bars_df['ts_event'].max()}")
        else:
            self.log.warning("未找到数据")

    def _scan_and_index_directory(self, data_path: Path):
        """扫描目录并建立索引"""
        for data_type in ['bars', 'quotes', 'trades']:
            type_path = data_path / data_type

            if not type_path.exists():
                continue

            for file_path in type_path.rglob("*.parquet"):
                try:
                    # 读取元数据
                    metadata_path = file_path.with_suffix('.metadata.json')
                    if metadata_path.exists():
                        import json
                        with open(metadata_path, 'r') as f:
                            metadata = json.load(f)

                        # 添加到索引
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
                    self.log.warning(f"处理文件失败 {file_path}: {e}")

        # 保存索引
        self.data_index.save_indices()

        # 输出统计
        stats = self.data_index.get_index_stats()
        self.log.info("索引统计:")
        self.log.info(f"  K线数据: {stats['bars_count']} 个文件")
        self.log.info(f"  报价数据: {stats['quotes_count']} 个文件")
        self.log.info(f"  成交数据: {stats['trades_count']} 个文件")

    def on_start(self):
        """策略启动"""
        self.log.info("索引数据示例策略启动")
        self.load_historical_data()

    def on_bar(self, bar: Bar):
        """处理K线数据"""
        pass

    def on_stop(self):
        """策略停止"""
        # 保存索引
        self.data_index.save_indices()

        # 输出统计
        stats = self.data_index.get_index_stats()
        self.log.info("最终索引统计:")
        self.log.info(f"  总文件数: {stats['bars_count'] + stats['quotes_count'] + stats['trades_count']}")
        self.log.info(f"  缓存大小: {len(self.fast_loader.file_cache)} 个文件")
```

## 10.4 下一步

在本章中，我们学习了：

1. Parquet 数据目录的使用
2. 数据压缩和分区策略
3. ETL 数据流水线构建
4. 数据索引和查询优化
5. 高性能数据加载

在下一章中，我们将学习：

1. 消息总线和 Actor 模型
2. 组件间通信
3. 并发处理模式
4. 错误处理和恢复
5. 系统架构最佳实践

## 10.5 总结

### 关键要点
1. Parquet 格式提供了高效的存储和压缩
2. 分区策略提高了数据查询性能
3. 数据索引显著加速数据检索
4. ETL 流水线实现了数据处理自动化
5. 缓存机制减少了重复加载时间

### 最佳实践
1. 根据数据类型选择合适的压缩算法
2. 定期清理和优化索引
3. 实施缓存策略避免重复加载
4. 使用监控跟踪数据质量
5. 建立数据备份机制

## 10.6 参考资料

- [Apache Arrow 文档](https://arrow.apache.org/docs/)
- [Parquet 格式规范](https://parquet.apache.org/)
- [PyArrow 使用指南](https://pandas.pydata.org/docs/reference/api/pandas.io.parquet.html)
- [Polars 高性能数据分析](https://pola-rs.github.io/docs/)