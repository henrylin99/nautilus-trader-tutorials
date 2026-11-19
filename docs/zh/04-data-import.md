# 第4章：数据导入与处理

## 学习目标

通过本章学习，您将：
- 了解 NautilusTrader 支持的各种数据类型
- 掌握从 CSV 文件导入数据的方法
- 学会处理和清洗历史数据
- 理解时间同步的重要性
- 创建和验证高质量的数据集

## 4.1 数据类型概览

NautilusTrader 支持多种市场数据类型，每种数据都有其特定的用途：

### 4.1.1 Tick 数据

#### QuoteTick（报价数据）
包含买卖价格信息，是最高频的数据类型：
```python
from nautilus_trader.model.data import QuoteTick
from nautilus_trader.model.objects import Price, Quantity

# 创建报价tick
quote = QuoteTick(
    instrument_id=BTCUSDT_BINANCE,
    bid_price=Price.from_str("50000.00"),
    ask_price=Price.from_str("50001.00"),
    bid_size=Quantity.from_int(10),
    ask_size=Quantity.from_int(15),
    ts_event=1640995200000000000,  # 纳秒时间戳
    ts_init=1640995200000000000,
)

# 使用场景
# - 高频交易
# - 订单簿分析
# - 延迟分析
# - 滑点计算
```

#### TradeTick（成交数据）
记录实际成交的价格和数量：
```python
from nautilus_trader.model.data import TradeTick
from nautilus_trader.model.enums import AggressorSide

# 创建成交tick
trade = TradeTick(
    instrument_id=BTCUSDT_BINANCE,
    price=Price.from_str("50000.50"),
    size=Quantity.from_int(5),
    aggressor_side=AggressorSide.BUYER,  # 买方吃单
    trade_id="123456789",
    ts_event=1640995200000000000,
    ts_init=1640995200000000000,
)

# 使用场景
# - 成交量分析
# - 交易流向分析
# - 微观结构研究
```

### 4.1.2 K线数据（Bar）

K线数据是tick数据的聚合，包含OHLCV信息：

```python
from nautilus_trader.model.data import Bar, BarType
from nautilus_trader.model.enums import BarAggregation, PriceType

# 定义K线类型
bar_type = BarType(
    instrument_id=InstrumentId.from_str("BTCUSDT.BINANCE"),
    bar_spec=BarSpecification(
        step=1,
        aggregation=BarAggregation.MINUTE,  # 1分钟K线
        price_type=PriceType.LAST,         # 成交价
    ),
    aggregation_source="BINANCE",
)

# 创建K线
bar = Bar(
    bar_type=bar_type,
    open=Price.from_str("50000.00"),
    high=Price.from_str("50100.00"),
    low=Price.from_str("49900.00"),
    close=Price.from_str("50050.00"),
    volume=Quantity.from_int(100),
    ts_event=1640995200000000000,
    ts_init=1640995200000000000,
)

# 使用场景
# - 技术分析
# - 回测验证
# - 策略信号
# - 风险管理
```

### 4.1.3 订单簿数据

#### OrderBookSnapshot（订单簿快照）
```python
from nautilus_trader.model.data import OrderBookSnapshot
from nautilus_trader.model.orderbook import OrderBook
from nautilus_trader.model.enums import OrderSide

# 创建订单簿
book = OrderBook(
    instrument_id=InstrumentId.from_str("BTCUSDT.BINANCE"),
    price_precision=2,
    size_precision=6,
)

# 添加订单
book.update(
    OrderSide.BUY,
    Price.from_str("50000.00"),
    Quantity.from_int(10),
    1640995200000000000,
)

book.update(
    OrderSide.SELL,
    Price.from_str("50001.00"),
    Quantity.from_int(15),
    1640995200000000000,
)

# 创建快照
snapshot = OrderBookSnapshot(
    instrument_id=book.instrument_id,
    bids=book.bids(),
    asks=book.asks(),
    ts_event=1640995200000000000,
    ts_init=1640995200000000000,
)

# 使用场景
# - 做市策略
# - 流动性分析
# - 大单检测
# - 滑点预测
```

## 4.2 从 CSV 导入数据

CSV 是最常见的历史数据格式。让我们创建一个完整的数据导入流程。

### 4.2.1 准备 CSV 数据

创建示例 CSV 文件 `btcusdt_1m.csv`：
```csv
timestamp,open,high,low,close,volume
2024-01-01 00:00:00,50000.00,50100.00,49900.00,50050.00,125.5
2024-01-01 00:01:00,50050.00,50200.00,49950.00,50150.00,98.3
2024-01-01 00:02:00,50150.00,50300.00,50000.00,50250.00,156.7
2024-01-01 00:03:00,50250.00,50400.00,50150.00,50350.00,203.4
2024-01-01 00:04:00,50350.00,50500.00,50250.00,50450.00,187.2
```

### 4.2.2 创建数据导入器

创建 `data_importer.py`：
```python
"""
CSV 数据导入器
支持多种 CSV 格式和配置选项
"""

import pandas as pd
from datetime import datetime
from decimal import Decimal
from pathlib import Path
from typing import List, Optional, Dict, Any

from nautilus_trader.model.data import Bar
from nautilus_trader.model.data import BarType
from nautilus_trader.model.data import BarSpecification
from nautilus_trader.model.enums import BarAggregation
from nautilus_trader.model.enums import PriceType
from nautilus_trader.model.identifiers import InstrumentId
from nautilus_trader.persistence.wranglers import BarDataWrangler
from nautilus_trader.test_kit.providers import TestInstrumentProvider


class CSVDataImporter:
    """
    CSV 数据导入器

    支持的功能：
    - 自动检测列名
    - 时间格式解析
    - 数据验证
    - 缺失值处理
    """

    def __init__(self):
        """初始化导入器"""
        # 常见的列名映射
        self.column_mappings = {
            'timestamp': ['timestamp', 'datetime', 'time', 'date'],
            'open': ['open', 'o', 'Open'],
            'high': ['high', 'h', 'High'],
            'low': ['low', 'l', 'Low'],
            'close': ['close', 'c', 'Close'],
            'volume': ['volume', 'vol', 'v', 'Volume'],
        }

        # 时间格式列表
        self.time_formats = [
            '%Y-%m-%d %H:%M:%S',
            '%Y-%m-%d %H:%M:%S.%f',
            '%Y-%m-%dT%H:%M:%S',
            '%Y-%m-%dT%H:%M:%S.%f',
            '%Y-%m-%d',
            '%d/%m/%Y %H:%M:%S',
            '%m/%d/%Y %H:%M:%S',
        ]

    def load_csv(self, file_path: Path, **kwargs) -> pd.DataFrame:
        """
        加载 CSV 文件

        Parameters
        ----------
        file_path : Path
            CSV 文件路径

        Returns
        -------
        pd.DataFrame
            加载的数据
        """
        print(f"加载 CSV 文件: {file_path}")

        # 尝试不同的编码
        encodings = ['utf-8', 'gbk', 'gb2312', 'latin-1']

        for encoding in encodings:
            try:
                # 自动检测分隔符
                df = pd.read_csv(
                    file_path,
                    encoding=encoding,
                    **kwargs
                )
                print(f"成功加载，使用编码: {encoding}")
                break
            except UnicodeDecodeError:
                continue
        else:
            raise ValueError(f"无法解码文件: {file_path}")

        # 显示基本信息
        print(f"数据形状: {df.shape}")
        print(f"列名: {list(df.columns)}")

        return df

    def normalize_columns(self, df: pd.DataFrame) -> pd.DataFrame:
        """
        标准化列名

        Parameters
        ----------
        df : pd.DataFrame
            原始数据

        Returns
        -------
        pd.DataFrame
            标准化后的数据
        """
        print("标准化列名...")

        # 转换为小写
        df.columns = df.columns.str.lower()

        # 查找并重命名列
        column_map = {}
        for standard_name, possible_names in self.column_mappings.items():
            for col in df.columns:
                if col in possible_names:
                    column_map[col] = standard_name
                    break

        if column_map:
            df = df.rename(columns=column_map)
            print(f"重命名列: {column_map}")

        # 检查必需的列
        required_columns = ['timestamp', 'open', 'high', 'low', 'close']
        missing_columns = [col for col in required_columns if col not in df.columns]

        if missing_columns:
            raise ValueError(f"缺少必需的列: {missing_columns}")

        # 如果没有volume列，添加默认值
        if 'volume' not in df.columns:
            df['volume'] = 1.0
            print("添加默认volume列")

        return df

    def parse_timestamp(self, df: pd.DataFrame, timestamp_col: str = 'timestamp') -> pd.DataFrame:
        """
        解析时间戳

        Parameters
        ----------
        df : pd.DataFrame
            包含时间戳的数据
        timestamp_col : str
            时间戳列名

        Returns
        -------
        pd.DataFrame
            解析后的数据
        """
        print("解析时间戳...")

        # 尝试不同的时间格式
        for time_format in self.time_formats:
            try:
                df['timestamp'] = pd.to_datetime(
                    df[timestamp_col],
                    format=time_format
                )
                print(f"成功解析时间格式: {time_format}")
                break
            except (ValueError, TypeError):
                continue
        else:
            # 如果都失败了，尝试自动解析
            try:
                df['timestamp'] = pd.to_datetime(df[timestamp_col])
                print("自动解析时间戳成功")
            except Exception as e:
                raise ValueError(f"无法解析时间戳: {e}")

        # 检查时间戳范围
        print(f"时间范围: {df['timestamp'].min()} 到 {df['timestamp'].max()}")

        # 设置为索引（可选）
        df = df.set_index('timestamp')

        return df

    def validate_data(self, df: pd.DataFrame) -> pd.DataFrame:
        """
        验证和清洗数据

        Parameters
        ----------
        df : pd.DataFrame
            待验证的数据

        Returns
        -------
        pd.DataFrame
            验证后的数据
        """
        print("验证数据...")

        # 检查数值列
        numeric_columns = ['open', 'high', 'low', 'close', 'volume']

        # 检查是否有空值
        null_counts = df[numeric_columns].isnull().sum()
        if null_counts.any():
            print(f"发现空值: {null_counts[null_counts > 0]}")

            # 删除包含空值的行
            df = df.dropna(subset=numeric_columns)
            print("删除包含空值的行")

        # 转换为数值类型
        for col in numeric_columns:
            df[col] = pd.to_numeric(df[col], errors='coerce')

        # 检查价格关系
        invalid_prices = df[
            (df['high'] < df['low']) |
            (df['high'] < df['open']) |
            (df['high'] < df['close']) |
            (df['low'] > df['open']) |
            (df['low'] > df['close'])
        ]

        if len(invalid_prices) > 0:
            print(f"发现 {len(invalid_prices)} 行无效的价格关系")
            # 修正价格关系
            df['high'] = df[['high', 'open', 'close']].max(axis=1)
            df['low'] = df[['low', 'open', 'close']].min(axis=1)
            print("修正价格关系")

        # 检查负值
        negative_values = df[
            (df['open'] <= 0) |
            (df['high'] <= 0) |
            (df['low'] <= 0) |
            (df['close'] <= 0)
        ]

        if len(negative_values) > 0:
            print(f"发现 {len(negative_values)} 行负价格")
            df = df[df['open'] > 0]
            df = df[df['high'] > 0]
            df = df[df['low'] > 0]
            df = df[df['close'] > 0]

        # 检查重复时间戳
        duplicates = df.index.duplicated().sum()
        if duplicates > 0:
            print(f"发现 {duplicates} 个重复时间戳")
            df = df[~df.index.duplicated(keep='first')]

        print(f"验证完成，剩余数据: {len(df)} 行")
        return df

    def convert_to_bars(
        self,
        df: pd.DataFrame,
        instrument_id: InstrumentId,
        bar_type: BarType
    ) -> List[Bar]:
        """
        转换为 Bar 对象

        Parameters
        ----------
        df : pd.DataFrame
            验证后的数据
        instrument_id : InstrumentId
            交易工具ID
        bar_type : BarType
            K线类型

        Returns
        -------
        List[Bar]
            Bar对象列表
        """
        print("转换为 Bar 对象...")

        # 创建交易工具（使用测试工具作为示例）
        instrument = TestInstrumentProvider.btcusdt_binance()

        # 创建数据整理器
        wrangler = BarDataWrangler(
            bar_type=bar_type,
            instrument=instrument
        )

        # 重置索引以便访问时间戳
        df = df.reset_index()

        # 转换数据
        bars = wrangler.process_df(df)

        print(f"成功转换 {len(bars)} 根K线")
        return bars

    def import_csv_to_bars(
        self,
        file_path: Path,
        instrument_id: InstrumentId,
        bar_type: BarType,
        **kwargs
    ) -> List[Bar]:
        """
        从CSV导入并转换为Bar对象的完整流程

        Parameters
        ----------
        file_path : Path
            CSV文件路径
        instrument_id : InstrumentId
            交易工具ID
        bar_type : BarType
            K线类型
        **kwargs
            传递给pd.read_csv的参数

        Returns
        -------
        List[Bar]
            Bar对象列表
        """
        # 1. 加载CSV
        df = self.load_csv(file_path, **kwargs)

        # 2. 标准化列名
        df = self.normalize_columns(df)

        # 3. 解析时间戳
        df = self.parse_timestamp(df)

        # 4. 验证数据
        df = self.validate_data(df)

        # 5. 转换为Bar对象
        bars = self.convert_to_bars(df, instrument_id, bar_type)

        return bars


def main():
    """主函数 - 示例用法"""

    # 创建导入器
    importer = CSVDataImporter()

    # 定义交易工具和K线类型
    instrument_id = InstrumentId.from_str("BTCUSDT.BINANCE")
    bar_type = BarType(
        instrument_id=instrument_id,
        bar_spec=BarSpecification(
            step=1,
            aggregation=BarAggregation.MINUTE,
            price_type=PriceType.LAST,
        ),
        aggregation_source="BINANCE",
    )

    # 导入数据
    csv_path = Path("btcusdt_1m.csv")

    if csv_path.exists():
        try:
            bars = importer.import_csv_to_bars(
                file_path=csv_path,
                instrument_id=instrument_id,
                bar_type=bar_type,
                sep=',',
                skipinitialspace=True,
            )

            # 显示结果
            print(f"\n成功导入 {len(bars)} 根K线")
            if bars:
                print(f"第一根K线: {bars[0]}")
                print(f"最后一根K线: {bars[-1]}")

        except Exception as e:
            print(f"导入失败: {e}")
    else:
        print(f"文件不存在: {csv_path}")
        print("请创建测试CSV文件后再运行")


if __name__ == "__main__":
    main()
```

### 4.2.3 创建CSV文件生成器

为了测试，我们创建一个生成模拟CSV数据的工具：

创建 `generate_csv_data.py`：
```python
"""
生成模拟的CSV历史数据
"""

import pandas as pd
import numpy as np
from datetime import datetime, timedelta
from decimal import Decimal
from pathlib import Path


def generate_price_data(
    days: int = 7,
    start_price: float = 50000.0,
    volatility: float = 0.02,
    trend: float = 0.0001,
    output_path: Path = Path("btcusdt_1m.csv")
):
    """
    生成模拟的1分钟K线数据

    Parameters
    ----------
    days : int
        生成天数
    start_price : float
        起始价格
    volatility : float
        波动率
    trend : float
        趋势（正值上涨，负值下跌）
    output_path : Path
        输出文件路径
    """
    print(f"生成 {days} 天的1分钟K线数据...")

    # 计算分钟数
    minutes = days * 24 * 60
    timestamps = pd.date_range(
        start=datetime(2024, 1, 1),
        periods=minutes,
        freq='1min'
    )

    # 生成价格数据
    prices = [start_price]

    for i in range(1, minutes):
        # 随机游走 + 趋势
        change = np.random.normal(trend, volatility / np.sqrt(1440))
        new_price = prices[-1] * (1 + change)
        prices.append(max(new_price, 1.0))  # 确保价格不为负

    prices = np.array(prices)

    # 生成OHLC
    opens = prices[:-1]
    closes = prices[1:]

    # 生成高低价（在开盘和收盘之间）
    highs = np.maximum(opens, closes) * (1 + np.random.uniform(0, 0.005, minutes-1))
    lows = np.minimum(opens, closes) * (1 - np.random.uniform(0, 0.005, minutes-1))

    # 生成成交量（与价格变动相关）
    volume_base = 100
    volume = volume_base + np.abs(np.random.normal(0, 50, minutes-1))
    volume = np.maximum(volume, 1)

    # 创建DataFrame
    df = pd.DataFrame({
        'timestamp': timestamps[:-1],
        'open': opens,
        'high': highs,
        'low': lows,
        'close': closes,
        'volume': volume,
    })

    # 格式化时间戳
    df['timestamp'] = df['timestamp'].dt.strftime('%Y-%m-%d %H:%M:%S')

    # 保存到CSV
    df.to_csv(output_path, index=False)
    print(f"数据已保存到: {output_path}")
    print(f"数据范围: {df['timestamp'].iloc[0]} 到 {df['timestamp'].iloc[-1]}")
    print(f"价格范围: {df['open'].min():.2f} 到 {df['high'].max():.2f}")


def main():
    """主函数"""
    # 生成7天的数据
    generate_price_data(
        days=7,
        start_price=50000.0,
        volatility=0.02,  # 2% 日波动率
        trend=0.0001,     # 轻微上涨趋势
    )


if __name__ == "__main__":
    main()
```

### 4.2.4 运行示例

```bash
# 1. 生成测试数据
python generate_csv_data.py

# 2. 导入并处理数据
python data_importer.py
```

## 4.3 数据质量控制

高质量的数据是回测成功的关键。以下是一些重要的质量控制措施：

### 4.3.1 数据验证清单

创建 `data_validator.py`：
```python
"""
数据验证工具
确保数据质量
"""

import pandas as pd
import numpy as np
from typing import Dict, List, Tuple
from datetime import datetime


class DataValidator:
    """数据验证器"""

    def __init__(self):
        """初始化验证器"""
        self.errors = []
        self.warnings = []

    def validate_completeness(self, df: pd.DataFrame) -> bool:
        """
        验证数据完整性

        Parameters
        ----------
        df : pd.DataFrame
            待验证的数据

        Returns
        -------
        bool
            验证是否通过
        """
        print("验证数据完整性...")

        # 检查缺失值
        null_counts = df.isnull().sum()
        if null_counts.any():
            self.errors.append(f"发现缺失值: {null_counts[null_counts > 0].to_dict()}")
            return False

        # 检查空数据集
        if len(df) == 0:
            self.errors.append("数据集为空")
            return False

        print(f"数据完整性验证通过，共 {len(df)} 行")
        return True

    def validate_time_sequence(self, df: pd.DataFrame, timestamp_col: str = 'timestamp') -> bool:
        """
        验证时间序列

        Parameters
        ----------
        df : pd.DataFrame
            待验证的数据
        timestamp_col : str
            时间戳列名

        Returns
        -------
        bool
            验证是否通过
        """
        print("验证时间序列...")

        if timestamp_col not in df.columns:
            self.errors.append(f"缺少时间戳列: {timestamp_col}")
            return False

        # 确保时间戳是datetime类型
        if not pd.api.types.is_datetime64_any_dtype(df[timestamp_col]):
            df[timestamp_col] = pd.to_datetime(df[timestamp_col])

        # 检查时间戳是否递增
        time_diff = df[timestamp_col].diff()
        if (time_diff <= 0).any():
            invalid_count = (time_diff <= 0).sum()
            self.errors.append(f"发现 {invalid_count} 个非递增的时间戳")
            return False

        # 检查时间间隔
        min_interval = time_diff.min()
        max_interval = time_diff.max()

        print(f"时间间隔范围: {min_interval} 到 {max_interval}")

        # 检查是否有异常的时间间隔
        expected_interval = pd.Timedelta(minutes=1)  # 假设是1分钟数据
        tolerance = pd.Timedelta(seconds=30)

        irregular_intervals = df[time_diff > expected_interval + tolerance]
        if len(irregular_intervals) > 0:
            self.warnings.append(f"发现 {len(irregular_intervals)} 个异常时间间隔")

        print("时间序列验证通过")
        return True

    def validate_price_relationships(self, df: pd.DataFrame) -> bool:
        """
        验证价格关系

        Parameters
        ----------
        df : pd.DataFrame
            待验证的数据

        Returns
        -------
        bool
            验证是否通过
        """
        print("验证价格关系...")

        required_columns = ['open', 'high', 'low', 'close']
        missing_columns = [col for col in required_columns if col not in df.columns]

        if missing_columns:
            self.errors.append(f"缺少价格列: {missing_columns}")
            return False

        # 检查价格关系
        invalid_high = df['high'] < df[['open', 'close']].max(axis=1)
        invalid_low = df['low'] > df[['open', 'close']].min(axis=1)

        total_invalid = (invalid_high | invalid_low).sum()

        if total_invalid > 0:
            self.errors.append(f"发现 {total_invalid} 行无效的价格关系")
            return False

        # 检查价格是否为正数
        negative_prices = (
            (df['open'] <= 0) |
            (df['high'] <= 0) |
            (df['low'] <= 0) |
            (df['close'] <= 0)
        ).sum()

        if negative_prices > 0:
            self.errors.append(f"发现 {negative_prices} 行负价格或零价格")
            return False

        print("价格关系验证通过")
        return True

    def validate_volume(self, df: pd.DataFrame) -> bool:
        """
        验证成交量

        Parameters
        ----------
        df : pd.DataFrame
            待验证的数据

        Returns
        -------
        bool
            验证是否通过
        """
        print("验证成交量...")

        if 'volume' not in df.columns:
            self.warnings.append("缺少volume列")
            return True

        # 检查负成交量
        negative_volume = (df['volume'] < 0).sum()
        if negative_volume > 0:
            self.errors.append(f"发现 {negative_volume} 行负成交量")
            return False

        # 检查零成交量
        zero_volume = (df['volume'] == 0).sum()
        if zero_volume > 0:
            self.warnings.append(f"发现 {zero_volume} 行零成交量")

        # 检查异常成交量（均值的三倍）
        volume_mean = df['volume'].mean()
        volume_std = df['volume'].std()
        threshold = volume_mean + 3 * volume_std

        outliers = df['volume'] > threshold
        outlier_count = outliers.sum()

        if outlier_count > 0:
            self.warnings.append(
                f"发现 {outlier_count} 行异常成交量 "
                f"(>{threshold:.2f}, 均值: {volume_mean:.2f})"
            )

        print("成交量验证通过")
        return True

    def validate_duplicates(self, df: pd.DataFrame) -> bool:
        """
        验证重复数据

        Parameters
        ----------
        df : pd.DataFrame
            待验证的数据

        Returns
        -------
        bool
            验证是否通过
        """
        print("验证重复数据...")

        # 检查完全重复的行
        duplicate_rows = df.duplicated().sum()
        if duplicate_rows > 0:
            self.warnings.append(f"发现 {duplicate_rows} 行完全重复的数据")

        # 如果有时间戳列，检查重复的时间戳
        if 'timestamp' in df.columns:
            duplicate_timestamps = df['timestamp'].duplicated().sum()
            if duplicate_timestamps > 0:
                self.errors.append(f"发现 {duplicate_timestamps} 个重复的时间戳")
                return False

        print("重复数据验证通过")
        return True

    def generate_report(self) -> Dict:
        """
        生成验证报告

        Returns
        -------
        Dict
            验证报告
        """
        report = {
            'validation_time': datetime.now(),
            'total_errors': len(self.errors),
            'total_warnings': len(self.warnings),
            'errors': self.errors,
            'warnings': self.warnings,
            'status': 'PASSED' if len(self.errors) == 0 else 'FAILED'
        }

        return report

    def validate_all(self, df: pd.DataFrame) -> Dict:
        """
        执行所有验证

        Parameters
        ----------
        df : pd.DataFrame
            待验证的数据

        Returns
        -------
        Dict
            验证报告
        """
        print("\n开始数据验证...")
        print("=" * 50)

        # 执行所有验证
        validations = [
            self.validate_completeness,
            self.validate_time_sequence,
            self.validate_price_relationships,
            self.validate_volume,
            self.validate_duplicates,
        ]

        for validation in validations:
            validation(df)

        # 生成报告
        report = self.generate_report()

        # 打印报告
        print("\n" + "=" * 50)
        print("验证报告")
        print("=" * 50)
        print(f"状态: {report['status']}")
        print(f"错误数: {report['total_errors']}")
        print(f"警告数: {report['total_warnings']}")

        if report['errors']:
            print("\n错误:")
            for error in report['errors']:
                print(f"  - {error}")

        if report['warnings']:
            print("\n警告:")
            for warning in report['warnings']:
                print(f"  - {warning}")

        return report


def main():
    """主函数 - 示例用法"""

    # 创建验证器
    validator = DataValidator()

    # 加载数据
    csv_path = Path("btcusdt_1m.csv")
    if csv_path.exists():
        df = pd.read_csv(csv_path)
        df['timestamp'] = pd.to_datetime(df['timestamp'])

        # 验证数据
        report = validator.validate_all(df)

        # 保存报告
        report_path = Path("validation_report.json")
        import json
        with open(report_path, 'w') as f:
            # 转换datetime对象为字符串
            report['validation_time'] = str(report['validation_time'])
            json.dump(report, f, indent=2)

        print(f"\n验证报告已保存到: {report_path}")
    else:
        print(f"文件不存在: {csv_path}")


if __name__ == "__main__":
    main()
```

### 4.3.2 数据清洗策略

对于发现的问题，我们需要相应的清洗策略：

```python
"""
数据清洗工具
"""
import pandas as pd
import numpy as np
from typing import Optional


def clean_data(df: pd.DataFrame) -> pd.DataFrame:
    """
    清洗数据的通用函数

    Parameters
    ----------
    df : pd.DataFrame
        原始数据

    Returns
    -------
    pd.DataFrame
        清洗后的数据
    """
    print("开始数据清洗...")
    original_rows = len(df)

    # 1. 删除完全重复的行
    df = df.drop_duplicates()

    # 2. 处理时间戳重复（保留第一条）
    if 'timestamp' in df.columns:
        df = df.drop_duplicates(subset=['timestamp'], keep='first')

    # 3. 处理空值
    numeric_columns = df.select_dtypes(include=[np.number]).columns
    df[numeric_columns] = df[numeric_columns].fillna(method='ffill').fillna(method='bfill')

    # 4. 修正价格关系
    if all(col in df.columns for col in ['open', 'high', 'low', 'close']):
        # 确保high是最大的
        df['high'] = df[['high', 'open', 'close']].max(axis=1)
        # 确保low是最小的
        df['low'] = df[['low', 'open', 'close']].min(axis=1)

    # 5. 处理异常值
    for col in ['open', 'high', 'low', 'close']:
        if col in df.columns:
            # 使用IQR方法检测异常值
            Q1 = df[col].quantile(0.25)
            Q3 = df[col].quantile(0.75)
            IQR = Q3 - Q1

            # 定义异常值范围
            lower_bound = Q1 - 1.5 * IQR
            upper_bound = Q3 + 1.5 * IQR

            # 限制异常值
            df[col] = df[col].clip(lower=lower_bound, upper_bound=upper_bound)

    # 6. 确保价格为正
    price_columns = ['open', 'high', 'low', 'close']
    for col in price_columns:
        if col in df.columns:
            df[col] = df[col].abs()

    cleaned_rows = len(df)
    print(f"数据清洗完成: {original_rows} -> {cleaned_rows} 行")

    return df
```

## 4.4 时间同步处理

在真实交易中，数据来自多个源，时间同步至关重要。

### 4.4.1 时区处理

```python
"""
时间同步工具
处理不同时区和时间戳格式
"""

import pandas as pd
from datetime import datetime, timezone
import pytz


class TimeSynchronizer:
    """时间同步器"""

    def __init__(self, target_timezone: str = 'UTC'):
        """
        初始化时间同步器

        Parameters
        ----------
        target_timezone : str
            目标时区
        """
        self.target_tz = pytz.timezone(target_timezone)

    def convert_timezone(
        self,
        df: pd.DataFrame,
        timestamp_col: str,
        source_timezone: str
    ) -> pd.DataFrame:
        """
        转换时区

        Parameters
        ----------
        df : pd.DataFrame
            数据
        timestamp_col : str
            时间戳列名
        source_timezone : str
            源时区

        Returns
        -------
        pd.DataFrame
            转换后的数据
        """
        print(f"转换时区: {source_timezone} -> {self.target_tz}")

        source_tz = pytz.timezone(source_timezone)

        # 本地化时间戳
        df[timestamp_col] = df[timestamp_col].dt.tz_localize(source_tz)

        # 转换到目标时区
        df[timestamp_col] = df[timestamp_col].dt.tz_convert(self.target_tz)

        return df

    def align_time_series(
        self,
        df_list: list[pd.DataFrame],
        timestamp_col: str = 'timestamp'
    ) -> list[pd.DataFrame]:
        """
        对齐多个时间序列

        Parameters
        ----------
        df_list : list[pd.DataFrame]
            数据框列表
        timestamp_col : str
            时间戳列名

        Returns
        -------
        list[pd.DataFrame]
            对齐后的数据框列表
        """
        print("对齐时间序列...")

        # 找到所有时间戳的交集
        all_timestamps = None

        for df in df_list:
            timestamps = set(df[timestamp_col])
            if all_timestamps is None:
                all_timestamps = timestamps
            else:
                all_timestamps &= timestamps

        print(f"共同时间戳数量: {len(all_timestamps)}")

        # 过滤每个数据框
        aligned_dfs = []
        for df in df_list:
            aligned_df = df[df[timestamp_col].isin(all_timestamps)]
            aligned_dfs.append(aligned_df.sort_values(timestamp_col))

        return aligned_dfs
```

## 4.5 下一步

在本章的第一部分，我们学习了：

1. NautilusTrader 支持的数据类型
2. 从 CSV 导入数据的完整流程
3. 数据质量控制方法
4. 时间同步处理

在下一部分，我们将学习：

1. Parquet 数据目录的使用
2. 高效的数据存储和检索
3. 数据供应商集成
4. 实时数据处理

## 4.6 总结

### 关键要点
1. NautilusTrader 支持多种市场数据类型
2. CSV 导入需要仔细的数据验证和清洗
3. 时间同步对于多源数据至关重要
4. 高质量的数据是成功回测的基础

### 实践建议
1. 始终验证导入的数据
2. 保存数据验证报告
3. 使用适当的数据清洗策略
4. 考虑使用 Parquet 格式提高性能

## 4.7 参考资料

- [pandas 文档](https://pandas.pydata.org/docs/)
- [NautilusTrader 数据处理指南](https://nautilustrader.io/docs/latest/concepts/data/)
- [时间序列处理最佳实践](https://nautilustrader.io/docs/latest/tutorials/)