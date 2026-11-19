---
title: "Chapter 4: Data Import and Processing (Part 1)"
---

# Chapter 4: Data Import and Processing (Part 1)

## Learning Objectives

By studying this chapter, you will:
- Understand the various data types supported by NautilusTrader
- Master methods for importing data from CSV files
- Learn to process and clean historical data
- Understand the importance of time synchronization
- Create and validate high-quality datasets

## 4.1 Data Types Overview

NautilusTrader supports multiple market data types, each with its specific purpose:

### 4.1.1 Tick Data

#### QuoteTick
Contains bid and ask price information, the highest frequency data type:
```python
from nautilus_trader.model.data import QuoteTick
from nautilus_trader.model.objects import Price, Quantity

# Create a quote tick
quote = QuoteTick(
    instrument_id=BTCUSDT_BINANCE,
    bid_price=Price.from_str("50000.00"),
    ask_price=Price.from_str("50001.00"),
    bid_size=Quantity.from_int(10),
    ask_size=Quantity.from_int(15),
    ts_event=1640995200000000000,  # Nanosecond timestamp
    ts_init=1640995200000000000,
)

# Use cases
# - High-frequency trading
# - Order book analysis
# - Latency analysis
# - Slippage calculation
```

#### TradeTick
Records actual traded prices and quantities:
```python
from nautilus_trader.model.data import TradeTick
from nautilus_trader.model.enums import AggressorSide

# Create a trade tick
trade = TradeTick(
    instrument_id=BTCUSDT_BINANCE,
    price=Price.from_str("50000.50"),
    size=Quantity.from_int(5),
    aggressor_side=AggressorSide.BUYER,  # Buyer takes the order
    trade_id="123456789",
    ts_event=1640995200000000000,
    ts_init=1640995200000000000,
)

# Use cases
# - Volume analysis
# - Trade flow analysis
# - Microstructure research
```

### 4.1.2 Bar Data (K-line)

Bar data is aggregated from tick data, containing OHLCV information:

```python
from nautilus_trader.model.data import Bar, BarType
from nautilus_trader.model.enums import BarAggregation, PriceType

# Define bar type
bar_type = BarType(
    instrument_id=InstrumentId.from_str("BTCUSDT.BINANCE"),
    bar_spec=BarSpecification(
        step=1,
        aggregation=BarAggregation.MINUTE,  # 1-minute bars
        price_type=PriceType.LAST,         # Last price
    ),
    aggregation_source="BINANCE",
)

# Create a bar
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

# Use cases
# - Technical analysis
# - Backtesting validation
# - Strategy signals
# - Risk management
```

### 4.1.3 Order Book Data

#### OrderBookSnapshot
```python
from nautilus_trader.model.data import OrderBookSnapshot
from nautilus_trader.model.orderbook import OrderBook
from nautilus_trader.model.enums import OrderSide

# Create order book
book = OrderBook(
    instrument_id=InstrumentId.from_str("BTCUSDT.BINANCE"),
    price_precision=2,
    size_precision=6,
)

# Add orders
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

# Create snapshot
snapshot = OrderBookSnapshot(
    instrument_id=book.instrument_id,
    bids=book.bids(),
    asks=book.asks(),
    ts_event=1640995200000000000,
    ts_init=1640995200000000000,
)

# Use cases
# - Market making strategies
# - Liquidity analysis
# - Large order detection
# - Slippage prediction
```

## 4.2 Importing Data from CSV

CSV is the most common historical data format. Let's create a complete data import workflow.

### 4.2.1 Preparing CSV Data

Create sample CSV file `btcusdt_1m.csv`:
```csv
timestamp,open,high,low,close,volume
2024-01-01 00:00:00,50000.00,50100.00,49900.00,50050.00,125.5
2024-01-01 00:01:00,50050.00,50200.00,49950.00,50150.00,98.3
2024-01-01 00:02:00,50150.00,50300.00,50000.00,50250.00,156.7
2024-01-01 00:03:00,50250.00,50400.00,50150.00,50350.00,203.4
2024-01-01 00:04:00,50350.00,50500.00,50250.00,50450.00,187.2
```

### 4.2.2 Creating a Data Importer

Create `data_importer.py`:
```python
"""
CSV Data Importer
Supports multiple CSV formats and configuration options
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
    CSV Data Importer

    Supported features:
    - Automatic column name detection
    - Time format parsing
    - Data validation
    - Missing value handling
    """

    def __init__(self):
        """Initialize the importer"""
        # Common column name mappings
        self.column_mappings = {
            'timestamp': ['timestamp', 'datetime', 'time', 'date'],
            'open': ['open', 'o', 'Open'],
            'high': ['high', 'h', 'High'],
            'low': ['low', 'l', 'Low'],
            'close': ['close', 'c', 'Close'],
            'volume': ['volume', 'vol', 'v', 'Volume'],
        }

        # Time format list
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
        Load CSV file

        Parameters
        ----------
        file_path : Path
            CSV file path

        Returns
        -------
        pd.DataFrame
            Loaded data
        """
        print(f"Loading CSV file: {file_path}")

        # Try different encodings
        encodings = ['utf-8', 'gbk', 'gb2312', 'latin-1']

        for encoding in encodings:
            try:
                # Auto-detect separator
                df = pd.read_csv(
                    file_path,
                    encoding=encoding,
                    **kwargs
                )
                print(f"Successfully loaded using encoding: {encoding}")
                break
            except UnicodeDecodeError:
                continue
        else:
            raise ValueError(f"Unable to decode file: {file_path}")

        # Display basic information
        print(f"Data shape: {df.shape}")
        print(f"Column names: {list(df.columns)}")

        return df

    def normalize_columns(self, df: pd.DataFrame) -> pd.DataFrame:
        """
        Normalize column names

        Parameters
        ----------
        df : pd.DataFrame
            Original data

        Returns
        -------
        pd.DataFrame
            Normalized data
        """
        print("Normalizing column names...")

        # Convert to lowercase
        df.columns = df.columns.str.lower()

        # Find and rename columns
        column_map = {}
        for standard_name, possible_names in self.column_mappings.items():
            for col in df.columns:
                if col in possible_names:
                    column_map[col] = standard_name
                    break

        if column_map:
            df = df.rename(columns=column_map)
            print(f"Renamed columns: {column_map}")

        # Check required columns
        required_columns = ['timestamp', 'open', 'high', 'low', 'close']
        missing_columns = [col for col in required_columns if col not in df.columns]

        if missing_columns:
            raise ValueError(f"Missing required columns: {missing_columns}")

        # If no volume column, add default value
        if 'volume' not in df.columns:
            df['volume'] = 1.0
            print("Added default volume column")

        return df

    def parse_timestamp(self, df: pd.DataFrame, timestamp_col: str = 'timestamp') -> pd.DataFrame:
        """
        Parse timestamps

        Parameters
        ----------
        df : pd.DataFrame
            Data containing timestamps
        timestamp_col : str
            Timestamp column name

        Returns
        -------
        pd.DataFrame
            Parsed data
        """
        print("Parsing timestamps...")

        # Try different time formats
        for time_format in self.time_formats:
            try:
                df['timestamp'] = pd.to_datetime(
                    df[timestamp_col],
                    format=time_format
                )
                print(f"Successfully parsed time format: {time_format}")
                break
            except (ValueError, TypeError):
                continue
        else:
            # If all fail, try auto-parsing
            try:
                df['timestamp'] = pd.to_datetime(df[timestamp_col])
                print("Auto-parsed timestamps successfully")
            except Exception as e:
                raise ValueError(f"Unable to parse timestamps: {e}")

        # Check timestamp range
        print(f"Time range: {df['timestamp'].min()} to {df['timestamp'].max()}")

        # Set as index (optional)
        df = df.set_index('timestamp')

        return df

    def validate_data(self, df: pd.DataFrame) -> pd.DataFrame:
        """
        Validate and clean data

        Parameters
        ----------
        df : pd.DataFrame
            Data to validate

        Returns
        -------
        pd.DataFrame
            Validated data
        """
        print("Validating data...")

        # Check numeric columns
        numeric_columns = ['open', 'high', 'low', 'close', 'volume']

        # Check for null values
        null_counts = df[numeric_columns].isnull().sum()
        if null_counts.any():
            print(f"Found null values: {null_counts[null_counts > 0]}")

            # Remove rows containing null values
            df = df.dropna(subset=numeric_columns)
            print("Removed rows containing null values")

        # Convert to numeric types
        for col in numeric_columns:
            df[col] = pd.to_numeric(df[col], errors='coerce')

        # Check price relationships
        invalid_prices = df[
            (df['high'] < df['low']) |
            (df['high'] < df['open']) |
            (df['high'] < df['close']) |
            (df['low'] > df['open']) |
            (df['low'] > df['close'])
        ]

        if len(invalid_prices) > 0:
            print(f"Found {len(invalid_prices)} rows with invalid price relationships")
            # Fix price relationships
            df['high'] = df[['high', 'open', 'close']].max(axis=1)
            df['low'] = df[['low', 'open', 'close']].min(axis=1)
            print("Fixed price relationships")

        # Check negative values
        negative_values = df[
            (df['open'] <= 0) |
            (df['high'] <= 0) |
            (df['low'] <= 0) |
            (df['close'] <= 0)
        ]

        if len(negative_values) > 0:
            print(f"Found {len(negative_values)} rows with negative prices")
            df = df[df['open'] > 0]
            df = df[df['high'] > 0]
            df = df[df['low'] > 0]
            df = df[df['close'] > 0]

        # Check duplicate timestamps
        duplicates = df.index.duplicated().sum()
        if duplicates > 0:
            print(f"Found {duplicates} duplicate timestamps")
            df = df[~df.index.duplicated(keep='first')]

        print(f"Validation complete, remaining data: {len(df)} rows")
        return df

    def convert_to_bars(
        self,
        df: pd.DataFrame,
        instrument_id: InstrumentId,
        bar_type: BarType
    ) -> List[Bar]:
        """
        Convert to Bar objects

        Parameters
        ----------
        df : pd.DataFrame
            Validated data
        instrument_id : InstrumentId
            Instrument ID
        bar_type : BarType
            Bar type

        Returns
        -------
        List[Bar]
            List of Bar objects
        """
        print("Converting to Bar objects...")

        # Create instrument (using test instrument as example)
        instrument = TestInstrumentProvider.btcusdt_binance()

        # Create data wrangler
        wrangler = BarDataWrangler(
            bar_type=bar_type,
            instrument=instrument
        )

        # Reset index to access timestamps
        df = df.reset_index()

        # Convert data
        bars = wrangler.process_df(df)

        print(f"Successfully converted {len(bars)} bars")
        return bars

    def import_csv_to_bars(
        self,
        file_path: Path,
        instrument_id: InstrumentId,
        bar_type: BarType,
        **kwargs
    ) -> List[Bar]:
        """
        Complete workflow for importing from CSV and converting to Bar objects

        Parameters
        ----------
        file_path : Path
            CSV file path
        instrument_id : InstrumentId
            Instrument ID
        bar_type : BarType
            Bar type
        **kwargs
            Parameters to pass to pd.read_csv

        Returns
        -------
        List[Bar]
            List of Bar objects
        """
        # 1. Load CSV
        df = self.load_csv(file_path, **kwargs)

        # 2. Normalize column names
        df = self.normalize_columns(df)

        # 3. Parse timestamps
        df = self.parse_timestamp(df)

        # 4. Validate data
        df = self.validate_data(df)

        # 5. Convert to Bar objects
        bars = self.convert_to_bars(df, instrument_id, bar_type)

        return bars


def main():
    """Main function - Example usage"""

    # Create importer
    importer = CSVDataImporter()

    # Define instrument and bar type
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

    # Import data
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

            # Display results
            print(f"\nSuccessfully imported {len(bars)} bars")
            if bars:
                print(f"First bar: {bars[0]}")
                print(f"Last bar: {bars[-1]}")

        except Exception as e:
            print(f"Import failed: {e}")
    else:
        print(f"File does not exist: {csv_path}")
        print("Please create test CSV file before running")


if __name__ == "__main__":
    main()
```

### 4.2.3 Creating a CSV File Generator

For testing, let's create a tool to generate simulated CSV data:

Create `generate_csv_data.py`:
```python
"""
Generate simulated CSV historical data
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
    Generate simulated 1-minute bar data

    Parameters
    ----------
    days : int
        Number of days to generate
    start_price : float
        Starting price
    volatility : float
        Volatility
    trend : float
        Trend (positive for up, negative for down)
    output_path : Path
        Output file path
    """
    print(f"Generating {days} days of 1-minute bar data...")

    # Calculate number of minutes
    minutes = days * 24 * 60
    timestamps = pd.date_range(
        start=datetime(2024, 1, 1),
        periods=minutes,
        freq='1min'
    )

    # Generate price data
    prices = [start_price]

    for i in range(1, minutes):
        # Random walk + trend
        change = np.random.normal(trend, volatility / np.sqrt(1440))
        new_price = prices[-1] * (1 + change)
        prices.append(max(new_price, 1.0))  # Ensure price is not negative

    prices = np.array(prices)

    # Generate OHLC
    opens = prices[:-1]
    closes = prices[1:]

    # Generate high and low prices (between open and close)
    highs = np.maximum(opens, closes) * (1 + np.random.uniform(0, 0.005, minutes-1))
    lows = np.minimum(opens, closes) * (1 - np.random.uniform(0, 0.005, minutes-1))

    # Generate volume (correlated with price changes)
    volume_base = 100
    volume = volume_base + np.abs(np.random.normal(0, 50, minutes-1))
    volume = np.maximum(volume, 1)

    # Create DataFrame
    df = pd.DataFrame({
        'timestamp': timestamps[:-1],
        'open': opens,
        'high': highs,
        'low': lows,
        'close': closes,
        'volume': volume,
    })

    # Format timestamps
    df['timestamp'] = df['timestamp'].dt.strftime('%Y-%m-%d %H:%M:%S')

    # Save to CSV
    df.to_csv(output_path, index=False)
    print(f"Data saved to: {output_path}")
    print(f"Data range: {df['timestamp'].iloc[0]} to {df['timestamp'].iloc[-1]}")
    print(f"Price range: {df['open'].min():.2f} to {df['high'].max():.2f}")


def main():
    """Main function"""
    # Generate 7 days of data
    generate_price_data(
        days=7,
        start_price=50000.0,
        volatility=0.02,  # 2% daily volatility
        trend=0.0001,     # Slight upward trend
    )


if __name__ == "__main__":
    main()
```

### 4.2.4 Running the Example

```bash
# 1. Generate test data
python generate_csv_data.py

# 2. Import and process data
python data_importer.py
```

## 4.3 Data Quality Control

High-quality data is key to successful backtesting. Here are some important quality control measures:

### 4.3.1 Data Validation Checklist

Create `data_validator.py`:
```python
"""
Data validation tool
Ensure data quality
"""

import pandas as pd
import numpy as np
from typing import Dict, List, Tuple
from datetime import datetime


class DataValidator:
    """Data validator"""

    def __init__(self):
        """Initialize validator"""
        self.errors = []
        self.warnings = []

    def validate_completeness(self, df: pd.DataFrame) -> bool:
        """
        Validate data completeness

        Parameters
        ----------
        df : pd.DataFrame
            Data to validate

        Returns
        -------
        bool
            Whether validation passed
        """
        print("Validating data completeness...")

        # Check for missing values
        null_counts = df.isnull().sum()
        if null_counts.any():
            self.errors.append(f"Found missing values: {null_counts[null_counts > 0].to_dict()}")
            return False

        # Check empty dataset
        if len(df) == 0:
            self.errors.append("Dataset is empty")
            return False

        print(f"Data completeness validation passed, total {len(df)} rows")
        return True

    def validate_time_sequence(self, df: pd.DataFrame, timestamp_col: str = 'timestamp') -> bool:
        """
        Validate time sequence

        Parameters
        ----------
        df : pd.DataFrame
            Data to validate
        timestamp_col : str
            Timestamp column name

        Returns
        -------
        bool
            Whether validation passed
        """
        print("Validating time sequence...")

        if timestamp_col not in df.columns:
            self.errors.append(f"Missing timestamp column: {timestamp_col}")
            return False

        # Ensure timestamp is datetime type
        if not pd.api.types.is_datetime64_any_dtype(df[timestamp_col]):
            df[timestamp_col] = pd.to_datetime(df[timestamp_col])

        # Check if timestamps are increasing
        time_diff = df[timestamp_col].diff()
        if (time_diff <= 0).any():
            invalid_count = (time_diff <= 0).sum()
            self.errors.append(f"Found {invalid_count} non-increasing timestamps")
            return False

        # Check time intervals
        min_interval = time_diff.min()
        max_interval = time_diff.max()

        print(f"Time interval range: {min_interval} to {max_interval}")

        # Check for abnormal time intervals
        expected_interval = pd.Timedelta(minutes=1)  # Assuming 1-minute data
        tolerance = pd.Timedelta(seconds=30)

        irregular_intervals = df[time_diff > expected_interval + tolerance]
        if len(irregular_intervals) > 0:
            self.warnings.append(f"Found {len(irregular_intervals)} abnormal time intervals")

        print("Time sequence validation passed")
        return True

    def validate_price_relationships(self, df: pd.DataFrame) -> bool:
        """
        Validate price relationships

        Parameters
        ----------
        df : pd.DataFrame
            Data to validate

        Returns
        -------
        bool
            Whether validation passed
        """
        print("Validating price relationships...")

        required_columns = ['open', 'high', 'low', 'close']
        missing_columns = [col for col in required_columns if col not in df.columns]

        if missing_columns:
            self.errors.append(f"Missing price columns: {missing_columns}")
            return False

        # Check price relationships
        invalid_high = df['high'] < df[['open', 'close']].max(axis=1)
        invalid_low = df['low'] > df[['open', 'close']].min(axis=1)

        total_invalid = (invalid_high | invalid_low).sum()

        if total_invalid > 0:
            self.errors.append(f"Found {total_invalid} rows with invalid price relationships")
            return False

        # Check if prices are positive
        negative_prices = (
            (df['open'] <= 0) |
            (df['high'] <= 0) |
            (df['low'] <= 0) |
            (df['close'] <= 0)
        ).sum()

        if negative_prices > 0:
            self.errors.append(f"Found {negative_prices} rows with negative or zero prices")
            return False

        print("Price relationships validation passed")
        return True

    def validate_volume(self, df: pd.DataFrame) -> bool:
        """
        Validate volume

        Parameters
        ----------
        df : pd.DataFrame
            Data to validate

        Returns
        -------
        bool
            Whether validation passed
        """
        print("Validating volume...")

        if 'volume' not in df.columns:
            self.warnings.append("Missing volume column")
            return True

        # Check for negative volume
        negative_volume = (df['volume'] < 0).sum()
        if negative_volume > 0:
            self.errors.append(f"Found {negative_volume} rows with negative volume")
            return False

        # Check for zero volume
        zero_volume = (df['volume'] == 0).sum()
        if zero_volume > 0:
            self.warnings.append(f"Found {zero_volume} rows with zero volume")

        # Check for abnormal volume (3 times the mean)
        volume_mean = df['volume'].mean()
        volume_std = df['volume'].std()
        threshold = volume_mean + 3 * volume_std

        outliers = df['volume'] > threshold
        outlier_count = outliers.sum()

        if outlier_count > 0:
            self.warnings.append(
                f"Found {outlier_count} rows with abnormal volume "
                f"(>{threshold:.2f}, mean: {volume_mean:.2f})"
            )

        print("Volume validation passed")
        return True

    def validate_duplicates(self, df: pd.DataFrame) -> bool:
        """
        Validate duplicate data

        Parameters
        ----------
        df : pd.DataFrame
            Data to validate

        Returns
        -------
        bool
            Whether validation passed
        """
        print("Validating duplicate data...")

        # Check completely duplicate rows
        duplicate_rows = df.duplicated().sum()
        if duplicate_rows > 0:
            self.warnings.append(f"Found {duplicate_rows} completely duplicate rows")

        # Check duplicate timestamps if timestamp column exists
        if 'timestamp' in df.columns:
            duplicate_timestamps = df['timestamp'].duplicated().sum()
            if duplicate_timestamps > 0:
                self.errors.append(f"Found {duplicate_timestamps} duplicate timestamps")
                return False

        print("Duplicate data validation passed")
        return True

    def generate_report(self) -> Dict:
        """
        Generate validation report

        Returns
        -------
        Dict
            Validation report
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
        Execute all validations

        Parameters
        ----------
        df : pd.DataFrame
            Data to validate

        Returns
        -------
        Dict
            Validation report
        """
        print("\nStarting data validation...")
        print("=" * 50)

        # Execute all validations
        validations = [
            self.validate_completeness,
            self.validate_time_sequence,
            self.validate_price_relationships,
            self.validate_volume,
            self.validate_duplicates,
        ]

        for validation in validations:
            validation(df)

        # Generate report
        report = self.generate_report()

        # Print report
        print("\n" + "=" * 50)
        print("Validation Report")
        print("=" * 50)
        print(f"Status: {report['status']}")
        print(f"Error count: {report['total_errors']}")
        print(f"Warning count: {report['total_warnings']}")

        if report['errors']:
            print("\nErrors:")
            for error in report['errors']:
                print(f"  - {error}")

        if report['warnings']:
            print("\nWarnings:")
            for warning in report['warnings']:
                print(f"  - {warning}")

        return report


def main():
    """Main function - Example usage"""

    # Create validator
    validator = DataValidator()

    # Load data
    csv_path = Path("btcusdt_1m.csv")
    if csv_path.exists():
        df = pd.read_csv(csv_path)
        df['timestamp'] = pd.to_datetime(df['timestamp'])

        # Validate data
        report = validator.validate_all(df)

        # Save report
        report_path = Path("validation_report.json")
        import json
        with open(report_path, 'w') as f:
            # Convert datetime object to string
            report['validation_time'] = str(report['validation_time'])
            json.dump(report, f, indent=2)

        print(f"\nValidation report saved to: {report_path}")
    else:
        print(f"File does not exist: {csv_path}")


if __name__ == "__main__":
    main()
```

### 4.3.2 Data Cleaning Strategies

For identified issues, we need corresponding cleaning strategies:

```python
"""
Data cleaning tool
"""
import pandas as pd
import numpy as np
from typing import Optional


def clean_data(df: pd.DataFrame) -> pd.DataFrame:
    """
    General function for cleaning data

    Parameters
    ----------
    df : pd.DataFrame
        Original data

    Returns
    -------
    pd.DataFrame
        Cleaned data
    """
    print("Starting data cleaning...")
    original_rows = len(df)

    # 1. Remove completely duplicate rows
    df = df.drop_duplicates()

    # 2. Handle duplicate timestamps (keep first)
    if 'timestamp' in df.columns:
        df = df.drop_duplicates(subset=['timestamp'], keep='first')

    # 3. Handle missing values
    numeric_columns = df.select_dtypes(include=[np.number]).columns
    df[numeric_columns] = df[numeric_columns].fillna(method='ffill').fillna(method='bfill')

    # 4. Fix price relationships
    if all(col in df.columns for col in ['open', 'high', 'low', 'close']):
        # Ensure high is the maximum
        df['high'] = df[['high', 'open', 'close']].max(axis=1)
        # Ensure low is the minimum
        df['low'] = df[['low', 'open', 'close']].min(axis=1)

    # 5. Handle outliers
    for col in ['open', 'high', 'low', 'close']:
        if col in df.columns:
            # Use IQR method to detect outliers
            Q1 = df[col].quantile(0.25)
            Q3 = df[col].quantile(0.75)
            IQR = Q3 - Q1

            # Define outlier range
            lower_bound = Q1 - 1.5 * IQR
            upper_bound = Q3 + 1.5 * IQR

            # Clip outliers
            df[col] = df[col].clip(lower=lower_bound, upper_bound=upper_bound)

    # 6. Ensure prices are positive
    price_columns = ['open', 'high', 'low', 'close']
    for col in price_columns:
        if col in df.columns:
            df[col] = df[col].abs()

    cleaned_rows = len(df)
    print(f"Data cleaning complete: {original_rows} -> {cleaned_rows} rows")

    return df
```

## 4.4 Time Synchronization Processing

In real trading, data comes from multiple sources, making time synchronization crucial.

### 4.4.1 Time Zone Handling

```python
"""
Time synchronization tool
Handles different time zones and timestamp formats
"""

import pandas as pd
from datetime import datetime, timezone
import pytz


class TimeSynchronizer:
    """Time synchronizer"""

    def __init__(self, target_timezone: str = 'UTC'):
        """
        Initialize time synchronizer

        Parameters
        ----------
        target_timezone : str
            Target timezone
        """
        self.target_tz = pytz.timezone(target_timezone)

    def convert_timezone(
        self,
        df: pd.DataFrame,
        timestamp_col: str,
        source_timezone: str
    ) -> pd.DataFrame:
        """
        Convert time zone

        Parameters
        ----------
        df : pd.DataFrame
            Data
        timestamp_col : str
            Timestamp column name
        source_timezone : str
            Source timezone

        Returns
        -------
        pd.DataFrame
            Converted data
        """
        print(f"Converting timezone: {source_timezone} -> {self.target_tz}")

        source_tz = pytz.timezone(source_timezone)

        # Localize timestamps
        df[timestamp_col] = df[timestamp_col].dt.tz_localize(source_tz)

        # Convert to target timezone
        df[timestamp_col] = df[timestamp_col].dt.tz_convert(self.target_tz)

        return df

    def align_time_series(
        self,
        df_list: list[pd.DataFrame],
        timestamp_col: str = 'timestamp'
    ) -> list[pd.DataFrame]:
        """
        Align multiple time series

        Parameters
        ----------
        df_list : list[pd.DataFrame]
            List of dataframes
        timestamp_col : str
            Timestamp column name

        Returns
        -------
        list[pd.DataFrame]
            List of aligned dataframes
        """
        print("Aligning time series...")

        # Find intersection of all timestamps
        all_timestamps = None

        for df in df_list:
            timestamps = set(df[timestamp_col])
            if all_timestamps is None:
                all_timestamps = timestamps
            else:
                all_timestamps &= timestamps

        print(f"Common timestamp count: {len(all_timestamps)}")

        # Filter each dataframe
        aligned_dfs = []
        for df in df_list:
            aligned_df = df[df[timestamp_col].isin(all_timestamps)]
            aligned_dfs.append(aligned_df.sort_values(timestamp_col))

        return aligned_dfs
```

## 4.5 Next Steps

In the first part of this chapter, we learned:

1. Data types supported by NautilusTrader
2. Complete workflow for importing data from CSV
3. Data quality control methods
4. Time synchronization processing

In the next part, we will learn:

1. Using Parquet data directories
2. Efficient data storage and retrieval
3. Data vendor integration
4. Real-time data processing

## 4.6 Summary

### Key Points
1. NautilusTrader supports multiple market data types
2. CSV import requires careful data validation and cleaning
3. Time synchronization is crucial for multi-source data
4. High-quality data is the foundation for successful backtesting

### Practical Recommendations
1. Always validate imported data
2. Save data validation reports
3. Use appropriate data cleaning strategies
4. Consider using Parquet format for improved performance

## 4.7 References

- [pandas documentation](https://pandas.pydata.org/docs/)
- [NautilusTrader data processing guide](https://nautilustrader.io/docs/latest/concepts/data/)
- [Time series processing best practices](https://nautilustrader.io/docs/latest/tutorials/)