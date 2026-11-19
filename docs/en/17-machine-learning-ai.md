---
title: "Chapter 17: Machine Learning and AI"
---

# Chapter 17: Machine Learning and AI

## Learning Objectives

- Understand the application scenarios of machine learning in quantitative trading
- Master the methods of integrating ML models with NautilusTrader
- Learn to build end-to-end machine learning trading systems
- Implement model training, deployment, and online learning

## 17.1 Feature Engineering

### 17.1.1 Price Feature Extraction

```python
import numpy as np
import pandas as pd
from typing import Dict, List, Tuple
from scipy import stats
from sklearn.preprocessing import StandardScaler, RobustScaler

class FeatureExtractor:
    """Trading Feature Extractor"""

    def __init__(self, lookback_periods: List[int] = [5, 10, 20, 50]):
        self.lookback_periods = lookback_periods
        self.scalers = {}

    def extract_price_features(self, prices: pd.Series) -> pd.DataFrame:
        """Extract price-related features"""
        features = pd.DataFrame(index=prices.index)

        # Basic price features
        features['returns'] = prices.pct_change()
        features['log_returns'] = np.log(prices / prices.shift(1))

        # Returns for different periods
        for period in self.lookback_periods:
            features[f'return_{period}d'] = prices.pct_change(period)
            features[f'log_return_{period}d'] = np.log(prices / prices.shift(period))

        # Rolling statistical features
        for period in [10, 20, 50]:
            features[f'ma_{period}'] = prices.rolling(period).mean()
            features[f'std_{period}'] = prices.rolling(period).std()
            features[f'min_{period}'] = prices.rolling(period).min()
            features[f'max_{period}'] = prices.rolling(period).max()

            # Price relative position
            features[f'price_position_{period}'] = (
                (prices - features[f'min_{period}']) /
                (features[f'max_{period}'] - features[f'min_{period}'])
            )

        # Technical indicators
        features = self._add_technical_indicators(features, prices)

        return features

    def _add_technical_indicators(self, features: pd.DataFrame, prices: pd.Series) -> pd.DataFrame:
        """Add technical indicators"""
        # RSI
        delta = prices.diff()
        gain = (delta.where(delta > 0, 0)).rolling(window=14).mean()
        loss = (-delta.where(delta < 0, 0)).rolling(window=14).mean()
        rs = gain / loss
        features['rsi'] = 100 - (100 / (1 + rs))

        # MACD
        exp1 = prices.ewm(span=12).mean()
        exp2 = prices.ewm(span=26).mean()
        features['macd'] = exp1 - exp2
        features['macd_signal'] = features['macd'].ewm(span=9).mean()
        features['macd_histogram'] = features['macd'] - features['macd_signal']

        # Bollinger Bands
        ma20 = prices.rolling(20).mean()
        std20 = prices.rolling(20).std()
        features['bb_upper'] = ma20 + 2 * std20
        features['bb_lower'] = ma20 - 2 * std20
        features['bb_position'] = (prices - features['bb_lower']) / (features['bb_upper'] - features['bb_lower'])

        # Momentum indicators
        for period in [5, 10, 20]:
            features[f'momentum_{period}'] = prices / prices.shift(period) - 1

        # Volatility
        features['volatility_10'] = features['log_returns'].rolling(10).std() * np.sqrt(252)
        features['volatility_20'] = features['log_returns'].rolling(20).std() * np.sqrt(252)

        # Price acceleration
        features['acceleration'] = features['log_returns'].diff()

        # Z-score
        for period in [20, 50]:
            features[f'zscore_{period}'] = (
                (prices - prices.rolling(period).mean()) /
                prices.rolling(period).std()
            )

        return features

    def extract_volume_features(self, volumes: pd.Series) -> pd.DataFrame:
        """Extract volume features"""
        features = pd.DataFrame(index=volumes.index)

        # Basic volume features
        features['volume_ma_10'] = volumes.rolling(10).mean()
        features['volume_ma_20'] = volumes.rolling(20).mean()
        features['volume_ratio_10'] = volumes / features['volume_ma_10']
        features['volume_ratio_20'] = volumes / features['volume_ma_20']

        # Volume standard deviation
        features['volume_std_10'] = volumes.rolling(10).std()

        # Volume change rate
        features['volume_change'] = volumes.pct_change()

        # VWAP (simplified version)
        # Need to pass price data simultaneously
        if hasattr(self, 'prices'):
            vwap = (self.prices * volumes).rolling(20).sum() / volumes.rolling(20).sum()
            features['vwap'] = vwap
            features['price_vwap_ratio'] = self.prices / vwap

        # On-Balance Volume (OBV)
        price_change = self.prices.diff() if hasattr(self, 'prices') else pd.Series(0, index=volumes.index)
        obv = (price_change > 0).astype(int) * volumes - (price_change < 0).astype(int) * volumes
        features['obv'] = obv.cumsum()
        features['obv_ma_10'] = features['obv'].rolling(10).mean()

        return features

    def extract_microstructure_features(self, orderbook: Dict) -> pd.DataFrame:
        """Extract market microstructure features"""
        features = pd.DataFrame()

        if not orderbook:
            return features

        # Bid-ask spread
        if 'bids' in orderbook and 'asks' in orderbook:
            best_bid = orderbook['bids'][0][0] if orderbook['bids'] else 0
            best_ask = orderbook['asks'][0][0] if orderbook['asks'] else 0
            features['spread'] = best_ask - best_bid
            features['spread_pct'] = features['spread'] / ((best_bid + best_ask) / 2)

            # Order book imbalance
            bid_volume = sum(level[1] for level in orderbook['bids'][:5])
            ask_volume = sum(level[1] for level in orderbook['asks'][:5])
            total_volume = bid_volume + ask_volume
            features['order_imbalance'] = (bid_volume - ask_volume) / total_volume if total_volume > 0 else 0

            # Mid price
            features['mid_price'] = (best_bid + best_ask) / 2

            # Weighted mid price
            features['weighted_mid'] = (best_bid * ask_volume + best_ask * bid_volume) / total_volume

        return features

    def extract_time_features(self, timestamps: pd.DatetimeIndex) -> pd.DataFrame:
        """Extract time-related features"""
        features = pd.DataFrame(index=timestamps)

        # Time features
        features['hour'] = timestamps.hour
        features['day_of_week'] = timestamps.dayofweek
        features['day_of_month'] = timestamps.day
        features['month'] = timestamps.month

        # Cyclical encoding (sine cosine transformation)
        features['hour_sin'] = np.sin(2 * np.pi * features['hour'] / 24)
        features['hour_cos'] = np.cos(2 * np.pi * features['hour'] / 24)
        features['dow_sin'] = np.sin(2 * np.pi * features['day_of_week'] / 7)
        features['dow_cos'] = np.cos(2 * np.pi * features['day_of_week'] / 7)

        # Market sessions (for 24-hour trading)
        features['is_asian_session'] = ((features['hour'] >= 0) & (features['hour'] < 8)).astype(int)
        features['is_london_session'] = ((features['hour'] >= 8) & (features['hour'] < 16)).astype(int)
        features['is_ny_session'] = ((features['hour'] >= 13) & (features['hour'] < 22)).astype(int)

        return features

    def create_lag_features(self, features: pd.DataFrame, lags: List[int] = [1, 2, 3, 5, 10]) -> pd.DataFrame:
        """Create lag features"""
        lagged_features = pd.DataFrame(index=features.index)

        for column in features.columns:
            for lag in lags:
                lagged_features[f'{column}_lag_{lag}'] = features[column].shift(lag)

        return lagged_features

    def create_rolling_features(self, features: pd.DataFrame, windows: List[int] = [5, 10, 20]) -> pd.DataFrame:
        """Create rolling window features"""
        rolling_features = pd.DataFrame(index=features.index)

        for column in features.select_dtypes(include=[np.number]).columns:
            for window in windows:
                # Rolling mean
                rolling_features[f'{column}_rolling_mean_{window}'] = features[column].rolling(window).mean()
                # Rolling standard deviation
                rolling_features[f'{column}_rolling_std_{window}'] = features[column].rolling(window).std()
                # Rolling maximum
                rolling_features[f'{column}_rolling_max_{window}'] = features[column].rolling(window).max()
                # Rolling minimum
                rolling_features[f'{column}_rolling_min_{window}'] = features[column].rolling(window).min()

        return rolling_features

    def scale_features(self, features: pd.DataFrame, method: str = 'standard') -> pd.DataFrame:
        """Feature scaling"""
        if method == 'standard':
            scaler = StandardScaler()
        elif method == 'robust':
            scaler = RobustScaler()
        else:
            return features

        # Fit and transform numeric features
        numeric_features = features.select_dtypes(include=[np.number])
        scaled_features = scaler.fit_transform(numeric_features)

        # Create new DataFrame
        scaled_df = pd.DataFrame(
            scaled_features,
            index=features.index,
            columns=numeric_features.columns
        )

        # Preserve non-numeric features
        non_numeric_features = features.select_dtypes(exclude=[np.number])
        final_features = pd.concat([scaled_df, non_numeric_features], axis=1)

        # Save scaler for subsequent use
        self.scalers[method] = scaler

        return final_features

    def select_features(self, features: pd.DataFrame, target: pd.Series, method: str = 'correlation') -> pd.DataFrame:
        """Feature selection"""
        if method == 'correlation':
            # Correlation-based feature selection
            correlations = features.corrwith(target).abs()
            selected_features = correlations[correlations > 0.1].index
            return features[selected_features]

        elif method == 'mutual_info':
            # Mutual information feature selection
            from sklearn.feature_selection import mutual_info_regression
            mi_scores = mutual_info_regression(features.fillna(0), target.fillna(0))
            mi_scores = pd.Series(mi_scores, index=features.columns)
            selected_features = mi_scores[mi_scores > 0.01].index
            return features[selected_features]

        return features

class RealTimeFeatureExtractor:
    """Real-time Feature Extractor"""

    def __init__(self, feature_extractor: FeatureExtractor):
        self.feature_extractor = feature_extractor
        self.window_data = {
            'prices': pd.Series(),
            'volumes': pd.Series(),
            'timestamps': pd.DatetimeIndex([])
        }
        self.window_size = 200  # Keep the most recent 200 data points

    def update(self, price: float, volume: float, timestamp: pd.Timestamp) -> Dict:
        """Update real-time data and extract features"""
        # Add new data
        self.window_data['prices'] = self.window_data['prices'].append(
            pd.Series([price], index=[timestamp])
        )
        self.window_data['volumes'] = self.window_data['volumes'].append(
            pd.Series([volume], index=[timestamp])
        )

        # Maintain window size
        if len(self.window_data['prices']) > self.window_size:
            self.window_data['prices'] = self.window_data['prices'].iloc[-self.window_size:]
            self.window_data['volumes'] = self.window_data['volumes'].iloc[-self.window_size:]

        # Extract features
        features = {}
        if len(self.window_data['prices']) >= 20:
            # Price features
            price_features = self.feature_extractor.extract_price_features(self.window_data['prices'])
            if not price_features.empty:
                latest_features = price_features.iloc[-1].to_dict()
                features.update(latest_features)

            # Volume features
            volume_features = self.feature_extractor.extract_volume_features(self.window_data['volumes'])
            if not volume_features.empty:
                latest_volume_features = volume_features.iloc[-1].to_dict()
                features.update(latest_volume_features)

        return features
```

### 17.1.2 Label Engineering

```python
class LabelEngine:
    """Label Generation Engine"""

    def __init__(self):
        self.label_methods = {}

    def create_fixed_horizon_labels(self, prices: pd.Series, horizon: int = 5,
                                   threshold: float = 0.001) -> pd.Series:
        """Create fixed time window labels"""
        future_returns = prices.pct_change(horizon).shift(-horizon)

        labels = pd.Series(0, index=prices.index)  # 0: Hold

        # Set labels
        labels[future_returns > threshold] = 1   # 1: Buy
        labels[future_returns < -threshold] = -1  # -1: Sell

        return labels

    def create_triple_barrier_labels(self, prices: pd.Series, upper_barrier: float = 0.02,
                                    lower_barrier: float = -0.02, max_holding: int = 20) -> pd.Series:
        """Create triple barrier labels"""
        labels = pd.Series(0, index=prices.index)  # Default hold
        returns = np.log(prices / prices.shift(1))

        for i in range(len(prices) - max_holding):
            # Calculate cumulative returns
            cum_returns = returns.iloc[i:i+max_holding+1].cumsum()

            # Check if upper barrier is touched
            upper_touch = cum_returns >= upper_barrier
            if upper_touch.any():
                touch_index = cum_returns[upper_touch].index[0]
                labels.loc[touch_index] = 1
                continue

            # Check if lower barrier is touched
            lower_touch = cum_returns <= lower_barrier
            if lower_touch.any():
                touch_index = cum_returns[lower_touch].index[0]
                labels.loc[touch_index] = -1
                continue

            # Neither barrier touched, mark as hold
            labels.iloc[i + max_holding] = 0

        return labels

    def create_dynamic_threshold_labels(self, prices: pd.Series, atr_period: int = 14,
                                       multiplier: float = 2.0) -> pd.Series:
        """Create dynamic threshold labels (ATR-based)"""
        # Calculate ATR
        high_low = pd.DataFrame({
            'high': prices,
            'low': prices,
            'prev_close': prices.shift(1)
        })
        tr = pd.concat([
            high_low['high'] - high_low['low'],
            abs(high_low['high'] - high_low['prev_close']),
            abs(high_low['low'] - high_low['prev_close'])
        ], axis=1).max(axis=1)
        atr = tr.rolling(atr_period).mean()

        # Dynamic thresholds
        upper_threshold = multiplier * atr
        lower_threshold = -multiplier * atr

        labels = pd.Series(0, index=prices.index)

        for i in range(atr_period, len(prices) - 5):
            future_returns = np.log(prices.iloc[i+1:i+6] / prices.iloc[i])

            # Check if exceeds upper barrier
            if (future_returns > upper_threshold.iloc[i]).any():
                labels.iloc[i] = 1
            # Check if exceeds lower barrier
            elif (future_returns < lower_threshold.iloc[i]).any():
                labels.iloc[i] = -1
            else:
                labels.iloc[i] = 0

        return labels

    def create_regression_labels(self, prices: pd.Series, horizon: int = 5) -> pd.Series:
        """Create regression labels (predict returns)"""
        future_returns = prices.pct_change(horizon).shift(-horizon)
        return future_returns

    def create_volatility_labels(self, returns: pd.Series, window: int = 20) -> pd.Series:
        """Create volatility prediction labels"""
        future_vol = returns.rolling(window).std().shift(-window)
        current_vol = returns.rolling(window).std()

        # Classification: high volatility, normal volatility, low volatility
        labels = pd.Series(1, index=returns.index)  # Default normal
        labels[future_vol > current_vol * 1.2] = 2   # High volatility
        labels[future_vol < current_vol * 0.8] = 0   # Low volatility

        return labels

    def create_trend_labels(self, prices: pd.Series, trend_period: int = 20) -> pd.Series:
        """Create trend labels"""
        # Calculate trend slope
        def calculate_trend_slope(prices_window):
            if len(prices_window) < 2:
                return 0
            x = np.arange(len(prices_window))
            slope, _ = np.polyfit(x, prices_window, 1)
            return slope

        slopes = prices.rolling(trend_period).apply(calculate_trend_slope)

        # Normalize slopes
        normalized_slopes = slopes / prices

        labels = pd.Series(0, index=prices.index)
        labels[normalized_slopes > 0.001] = 1      # Uptrend
        labels[normalized_slopes < -0.001] = -1     # Downtrend

        return labels

    def combine_labels(self, *label_series, weights: List[float] = None) -> pd.Series:
        """Combine multiple labels"""
        if weights is None:
            weights = [1.0] * len(label_series)

        combined = pd.Series(0.0, index=label_series[0].index)

        for i, label_series_i in enumerate(label_series):
            combined += label_series_i * weights[i]

        # Normalize to -1, 0, 1
        combined[combined > 0.5] = 1
        combined[combined < -0.5] = -1
        combined[(combined >= -0.5) & (combined <= 0.5)] = 0

        return combined.astype(int)

    def create_multi_class_labels(self, prices: pd.Series, bins: int = 5) -> pd.Series:
        """Create multi-class labels"""
        returns = prices.pct_change()

        # Use quantiles to create labels
        labels = pd.qcut(returns, bins, labels=False, duplicates='drop')
        labels = labels - bins // 2  # Center at 0

        return labels.fillna(0)

    def get_label_distribution(self, labels: pd.Series) -> Dict:
        """Get label distribution statistics"""
        distribution = labels.value_counts(normalize=True).to_dict()
        return {
            'distribution': distribution,
            'total_samples': len(labels),
            'class_balance': max(distribution.values()) / min(distribution.values()) if len(distribution) > 1 else 1.0
        }
```

## 17.2 Prediction Models

### 17.2.1 Regression Models

```python
from sklearn.ensemble import RandomForestRegressor, GradientBoostingRegressor
from sklearn.linear_model import LinearRegression, Ridge, Lasso
from sklearn.svm import SVR
import xgboost as xgb
import lightgbm as lgb

class PricePredictionModel:
    """Price Prediction Model"""

    def __init__(self, model_type: str = 'xgboost', **model_params):
        self.model_type = model_type
        self.model_params = model_params
        self.model = None
        self.feature_importance = None
        self.prediction_history = []

    def create_model(self):
        """Create model"""
        if self.model_type == 'linear':
            self.model = LinearRegression(**self.model_params)
        elif self.model_type == 'ridge':
            self.model = Ridge(**self.model_params)
        elif self.model_type == 'lasso':
            self.model = Lasso(**self.model_params)
        elif self.model_type == 'random_forest':
            self.model = RandomForestRegressor(**self.model_params)
        elif self.model_type == 'gradient_boosting':
            self.model = GradientBoostingRegressor(**self.model_params)
        elif self.model_type == 'xgboost':
            self.model = xgb.XGBRegressor(**self.model_params)
        elif self.model_type == 'lightgbm':
            self.model = lgb.LGBMRegressor(**self.model_params)
        elif self.model_type == 'svr':
            self.model = SVR(**self.model_params)
        else:
            raise ValueError(f"Unsupported model type: {self.model_type}")

    def train(self, X: pd.DataFrame, y: pd.Series, validation_split: float = 0.2):
        """Train model"""
        # Split training and validation sets
        split_index = int(len(X) * (1 - validation_split))
        X_train, X_val = X[:split_index], X[split_index:]
        y_train, y_val = y[:split_index], y[split_index:]

        # Create and train model
        self.create_model()
        self.model.fit(X_train, y_train)

        # Validate model
        train_score = self.model.score(X_train, y_train)
        val_score = self.model.score(X_val, y_val)

        # Get feature importance
        if hasattr(self.model, 'feature_importances_'):
            self.feature_importance = pd.Series(
                self.model.feature_importances_,
                index=X.columns
            ).sort_values(ascending=False)

        return {
            'train_score': train_score,
            'validation_score': val_score,
            'feature_importance': self.feature_importance
        }

    def predict(self, X: pd.DataFrame) -> np.ndarray:
        """Predict"""
        if self.model is None:
            raise ValueError("Model not trained yet")

        predictions = self.model.predict(X)
        self.prediction_history.extend(predictions)
        return predictions

    def predict_with_confidence(self, X: pd.DataFrame) -> Tuple[np.ndarray, np.ndarray]:
        """Predict with confidence"""
        predictions = self.predict(X)

        # For tree-based models, use standard deviation as uncertainty estimate
        if self.model_type in ['random_forest', 'gradient_boosting', 'xgboost', 'lightgbm']:
            # Get predictions from each tree
            if hasattr(self.model, 'estimators_'):
                tree_predictions = np.array([
                    tree.predict(X) for tree in self.model.estimators_
                ])
                confidence = np.std(tree_predictions, axis=0)
            else:
                # XGBoost/LightGBM use other methods
                confidence = np.ones_like(predictions) * 0.1  # Default confidence
        else:
            confidence = np.ones_like(predictions) * 0.1

        return predictions, confidence

    def update_model(self, X_new: pd.DataFrame, y_new: pd.Series):
        """Incrementally update model"""
        if self.model is None:
            self.train(X_new, y_new)
        elif hasattr(self.model, 'partial_fit'):
            # Models that support incremental learning
            self.model.partial_fit(X_new, y_new)
        else:
            # Models that don't support incremental learning, need retraining
            # Need to save historical data
            pass

    def save_model(self, filepath: str):
        """Save model"""
        import joblib
        joblib.dump({
            'model': self.model,
            'model_type': self.model_type,
            'model_params': self.model_params,
            'feature_importance': self.feature_importance
        }, filepath)

    def load_model(self, filepath: str):
        """Load model"""
        import joblib
        data = joblib.load(filepath)
        self.model = data['model']
        self.model_type = data['model_type']
        self.model_params = data['model_params']
        self.feature_importance = data.get('feature_importance')
```

### 17.2.2 Classification Models

```python
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.svm import SVC
from sklearn.naive_bayes import GaussianNB
from sklearn.metrics import classification_report, confusion_matrix

class TradingSignalClassifier:
    """Trading Signal Classifier"""

    def __init__(self, model_type: str = 'xgboost', **model_params):
        self.model_type = model_type
        self.model_params = model_params
        self.model = None
        self.classes_ = None
        self.probability_history = []

    def create_model(self):
        """Create classification model"""
        if self.model_type == 'logistic':
            self.model = LogisticRegression(**self.model_params)
        elif self.model_type == 'random_forest':
            self.model = RandomForestClassifier(**self.model_params)
        elif self.model_type == 'gradient_boosting':
            self.model = GradientBoostingClassifier(**self.model_params)
        elif self.model_type == 'xgboost':
            self.model = xgb.XGBClassifier(**self.model_params)
        elif self.model_type == 'lightgbm':
            self.model = lgb.LGBMClassifier(**self.model_params)
        elif self.model_type == 'svm':
            self.model = SVC(probability=True, **self.model_params)
        elif self.model_type == 'naive_bayes':
            self.model = GaussianNB(**self.model_params)
        else:
            raise ValueError(f"Unsupported model type: {self.model_type}")

    def train(self, X: pd.DataFrame, y: pd.Series, validation_split: float = 0.2):
        """Train classification model"""
        # Split data
        split_index = int(len(X) * (1 - validation_split))
        X_train, X_val = X[:split_index], X[split_index:]
        y_train, y_val = y[:split_index], y[split_index:]

        # Create and train model
        self.create_model()
        self.model.fit(X_train, y_train)
        self.classes_ = self.model.classes_

        # Evaluate model
        train_pred = self.model.predict(X_train)
        val_pred = self.model.predict(X_val)

        train_report = classification_report(y_train, train_pred, output_dict=True)
        val_report = classification_report(y_val, val_pred, output_dict=True)

        # Get feature importance
        feature_importance = None
        if hasattr(self.model, 'feature_importances_'):
            feature_importance = pd.Series(
                self.model.feature_importances_,
                index=X.columns
            ).sort_values(ascending=False)

        return {
            'train_report': train_report,
            'validation_report': val_report,
            'confusion_matrix': confusion_matrix(y_val, val_pred),
            'feature_importance': feature_importance
        }

    def predict_signal(self, X: pd.DataFrame) -> np.ndarray:
        """Predict trading signals"""
        if self.model is None:
            raise ValueError("Model not trained yet")

        return self.model.predict(X)

    def predict_proba(self, X: pd.DataFrame) -> np.ndarray:
        """Predict probability distribution"""
        if self.model is None:
            raise ValueError("Model not trained yet")

        probabilities = self.model.predict_proba(X)
        self.probability_history.extend(probabilities)
        return probabilities

    def predict_with_threshold(self, X: pd.DataFrame, threshold: float = 0.6) -> np.ndarray:
        """Predict using threshold (only high-confidence predictions)"""
        probabilities = self.predict_proba(X)

        # Find maximum probability and its corresponding class
        max_probs = np.max(probabilities, axis=1)
        predictions = np.argmax(probabilities, axis=1)

        # Apply threshold
        confident_predictions = np.where(
            max_probs >= threshold,
            predictions,
            0  # Default to hold
        )

        return confident_predictions

    def get_signal_strength(self, X: pd.DataFrame) -> Dict:
        """Get signal strength"""
        probabilities = self.predict_proba(X)
        predictions = self.predict_signal(X)
        max_probs = np.max(probabilities, axis=1)

        signal_strength = {}
        for i, (pred, prob) in enumerate(zip(predictions, max_probs)):
            signal_strength[i] = {
                'signal': int(pred),
                'confidence': float(prob),
                'strength': self._calculate_strength(prob)
            }

        return signal_strength

    def _calculate_strength(self, probability: float) -> str:
        """Calculate signal strength"""
        if probability > 0.8:
            return 'very_strong'
        elif probability > 0.7:
            return 'strong'
        elif probability > 0.6:
            return 'moderate'
        elif probability > 0.5:
            return 'weak'
        else:
            return 'very_weak'
```

## 17.3 Ensemble Learning

### 17.3.1 Model Fusion

```python
from sklearn.ensemble import VotingClassifier, StackingClassifier
from sklearn.model_selection import cross_val_score
import numpy as np

class EnsembleModel:
    """Ensemble Learning Model"""

    def __init__(self, models: Dict, ensemble_method: str = 'voting'):
        self.models = models
        self.ensemble_method = ensemble_method
        self.ensemble_model = None
        self.model_weights = None

    def create_voting_ensemble(self, voting: str = 'soft', weights: List[float] = None):
        """Create voting ensemble model"""
        estimators = [(name, model) for name, model in self.models.items()]

        self.ensemble_model = VotingClassifier(
            estimators=estimators,
            voting=voting,
            weights=weights
        )

        self.model_weights = weights

    def create_stacking_ensemble(self, meta_model=None, cv: int = 5):
        """Create stacking ensemble model"""
        estimators = [(name, model) for name, model in self.models.items()]

        if meta_model is None:
            from sklearn.linear_model import LogisticRegression
            meta_model = LogisticRegression()

        self.ensemble_model = StackingClassifier(
            estimators=estimators,
            final_estimator=meta_model,
            cv=cv
        )

    def train(self, X: pd.DataFrame, y: pd.Series):
        """Train ensemble model"""
        if self.ensemble_method == 'voting':
            self.create_voting_ensemble()
        elif self.ensemble_method == 'stacking':
            self.create_stacking_ensemble()

        self.ensemble_model.fit(X, y)

        # Evaluate individual base models
        individual_scores = {}
        for name, model in self.models.items():
            scores = cross_val_score(model, X, y, cv=5)
            individual_scores[name] = {
                'mean_score': scores.mean(),
                'std_score': scores.std()
            }

        # Evaluate ensemble model
        ensemble_scores = cross_val_score(self.ensemble_model, X, y, cv=5)

        return {
            'individual_scores': individual_scores,
            'ensemble_score': {
                'mean_score': ensemble_scores.mean(),
                'std_score': ensemble_scores.std()
            }
        }

    def predict(self, X: pd.DataFrame) -> np.ndarray:
        """Predict"""
        if self.ensemble_model is None:
            raise ValueError("Ensemble model not trained yet")

        return self.ensemble_model.predict(X)

    def predict_proba(self, X: pd.DataFrame) -> np.ndarray:
        """Predict probabilities"""
        if self.ensemble_model is None:
            raise ValueError("Ensemble model not trained yet")

        return self.ensemble_model.predict_proba(X)

class AdaptiveEnsemble:
    """Adaptive Ensemble Model"""

    def __init__(self, window_size: int = 100):
        self.window_size = window_size
        self.models = {}
        self.model_performance = {}
        self.current_weights = {}
        self.performance_history = []

    def add_model(self, name: str, model):
        """Add model"""
        self.models[name] = model
        self.model_performance[name] = []

    def update_weights(self):
        """Update model weights"""
        if not self.performance_history:
            return

        # Get recent performance
        recent_performance = self.performance_history[-self.window_size:]

        # Calculate average performance for each model
        avg_performance = {}
        for name in self.models.keys():
            performances = [p[name] for p in recent_performance if name in p]
            if performances:
                avg_performance[name] = np.mean(performances)
            else:
                avg_performance[name] = 0

        # Convert to weights
        total_performance = sum(avg_performance.values())
        if total_performance > 0:
            self.current_weights = {
                name: perf / total_performance
                for name, perf in avg_performance.items()
            }
        else:
            # Equal weights
            n_models = len(self.models)
            self.current_weights = {
                name: 1.0 / n_models
                for name in self.models.keys()
            }

    def predict(self, X: pd.DataFrame) -> np.ndarray:
        """Predict using adaptive weights"""
        predictions = {}
        weighted_predictions = np.zeros(len(X))

        # Get predictions from each model
        for name, model in self.models.items():
            if hasattr(model, 'predict_proba'):
                pred = model.predict_proba(X)
                # Use probability of positive class
                if pred.shape[1] == 2:
                    pred = pred[:, 1]
            else:
                pred = model.predict(X)
            predictions[name] = pred

        # Weighted average
        for name, pred in predictions.items():
            weight = self.current_weights.get(name, 0)
            weighted_predictions += pred * weight

        return weighted_predictions

    def evaluate_models(self, X: pd.DataFrame, y: pd.Series):
        """Evaluate model performance"""
        performance = {}

        for name, model in self.models.items():
            if hasattr(model, 'predict_proba'):
                pred = model.predict_proba(X)
                if pred.shape[1] == 2:
                    pred = pred[:, 1]
            else:
                pred = model.predict(X)

            # Calculate accuracy or other metrics
            score = np.mean((pred > 0.5) == y)
            performance[name] = score

        self.performance_history.append(performance)
        self.update_weights()

        return performance
```

## 17.4 Deep Learning

### 17.4.1 LSTM Price Prediction

```python
import torch
import torch.nn as nn
from torch.utils.data import Dataset, DataLoader
from sklearn.preprocessing import MinMaxScaler

class PriceDataset(Dataset):
    """Price Dataset"""

    def __init__(self, data: np.ndarray, sequence_length: int, horizon: int = 1):
        self.data = data
        self.sequence_length = sequence_length
        self.horizon = horizon

    def __len__(self):
        return len(self.data) - self.sequence_length - self.horizon + 1

    def __getitem__(self, idx):
        # Input sequence
        x = self.data[idx:idx + self.sequence_length]

        # Target value (future price)
        y = self.data[idx + self.sequence_length + self.horizon - 1]

        return torch.FloatTensor(x), torch.FloatTensor([y])

class LSTMPricePredictor(nn.Module):
    """LSTM Price Prediction Model"""

    def __init__(self, input_size: int, hidden_size: int = 64, num_layers: int = 2,
                 dropout: float = 0.2, bidirectional: bool = False):
        super().__init__()

        self.hidden_size = hidden_size
        self.num_layers = num_layers
        self.bidirectional = bidirectional

        # LSTM layer
        self.lstm = nn.LSTM(
            input_size=input_size,
            hidden_size=hidden_size,
            num_layers=num_layers,
            dropout=dropout if num_layers > 1 else 0,
            batch_first=True,
            bidirectional=bidirectional
        )

        # Fully connected layers
        lstm_output_size = hidden_size * 2 if bidirectional else hidden_size
        self.fc1 = nn.Linear(lstm_output_size, hidden_size)
        self.fc2 = nn.Linear(hidden_size, hidden_size // 2)
        self.fc3 = nn.Linear(hidden_size // 2, 1)

        # Dropout and activation function
        self.dropout = nn.Dropout(dropout)
        self.relu = nn.ReLU()

    def forward(self, x):
        # LSTM forward pass
        lstm_out, (h_n, c_n) = self.lstm(x)

        # Take the output from the last time step
        if self.bidirectional:
            # Bidirectional LSTM, concatenate forward and backward last outputs
            last_output = torch.cat([
                lstm_out[:, -1, :self.hidden_size],
                lstm_out[:, 0, self.hidden_size:]
            ], dim=1)
        else:
            last_output = lstm_out[:, -1, :]

        # Fully connected layers
        out = self.relu(self.fc1(last_output))
        out = self.dropout(out)
        out = self.relu(self.fc2(out))
        out = self.fc3(out)

        return out

class DeepLearningTrainer:
    """Deep Learning Model Trainer"""

    def __init__(self, model: nn.Module, device: str = 'cuda'):
        self.model = model.to(device)
        self.device = device
        self.optimizer = None
        self.criterion = None
        self.scheduler = None
        self.training_history = []

    def setup_training(self, learning_rate: float = 0.001, weight_decay: float = 1e-5):
        """Setup training parameters"""
        self.optimizer = torch.optim.Adam(
            self.model.parameters(),
            lr=learning_rate,
            weight_decay=weight_decay
        )
        self.criterion = nn.MSELoss()
        self.scheduler = torch.optim.lr_scheduler.ReduceLROnPlateau(
            self.optimizer,
            mode='min',
            factor=0.5,
            patience=10,
            verbose=True
        )

    def train_epoch(self, dataloader: DataLoader) -> float:
        """Train one epoch"""
        self.model.train()
        total_loss = 0

        for batch_idx, (x, y) in enumerate(dataloader):
            x, y = x.to(self.device), y.to(self.device)

            # Forward pass
            self.optimizer.zero_grad()
            output = self.model(x)
            loss = self.criterion(output, y)

            # Backward pass
            loss.backward()

            # Gradient clipping
            torch.nn.utils.clip_grad_norm_(self.model.parameters(), max_norm=1.0)

            self.optimizer.step()

            total_loss += loss.item()

            # Print progress
            if batch_idx % 100 == 0:
                print(f'Batch {batch_idx}/{len(dataloader)}, Loss: {loss.item():.6f}')

        return total_loss / len(dataloader)

    def validate(self, dataloader: DataLoader) -> float:
        """Validate model"""
        self.model.eval()
        total_loss = 0

        with torch.no_grad():
            for x, y in dataloader:
                x, y = x.to(self.device), y.to(self.device)
                output = self.model(x)
                loss = self.criterion(output, y)
                total_loss += loss.item()

        return total_loss / len(dataloader)

    def train(self, train_loader: DataLoader, val_loader: DataLoader,
              epochs: int = 100, patience: int = 20):
        """Train model"""
        best_val_loss = float('inf')
        patience_counter = 0

        for epoch in range(epochs):
            # Train
            train_loss = self.train_epoch(train_loader)

            # Validate
            val_loss = self.validate(val_loader)

            # Update learning rate
            self.scheduler.step(val_loss)

            # Record history
            self.training_history.append({
                'epoch': epoch,
                'train_loss': train_loss,
                'val_loss': val_loss,
                'learning_rate': self.optimizer.param_groups[0]['lr']
            })

            # Early stopping
            if val_loss < best_val_loss:
                best_val_loss = val_loss
                patience_counter = 0
                # Save best model
                torch.save(self.model.state_dict(), 'best_model.pth')
            else:
                patience_counter += 1

            print(f'Epoch {epoch}: Train Loss = {train_loss:.6f}, Val Loss = {val_loss:.6f}')

            if patience_counter >= patience:
                print(f'Early stopping at epoch {epoch}')
                break

        # Load best model
        self.model.load_state_dict(torch.load('best_model.pth'))

    def predict(self, dataloader: DataLoader) -> np.ndarray:
        """Predict"""
        self.model.eval()
        predictions = []

        with torch.no_grad():
            for x, _ in dataloader:
                x = x.to(self.device)
                output = self.model(x)
                predictions.extend(output.cpu().numpy())

        return np.array(predictions)

    def save_model(self, filepath: str):
        """Save model"""
        torch.save({
            'model_state_dict': self.model.state_dict(),
            'optimizer_state_dict': self.optimizer.state_dict(),
            'training_history': self.training_history
        }, filepath)

    def load_model(self, filepath: str):
        """Load model"""
        checkpoint = torch.load(filepath, map_location=self.device)
        self.model.load_state_dict(checkpoint['model_state_dict'])
        self.optimizer.load_state_dict(checkpoint['optimizer_state_dict'])
        self.training_history = checkpoint.get('training_history', [])
```

## 17.5 Online Learning

### 17.5.1 Incremental Learning

```python
from sklearn.base import BaseEstimator
import joblib
import pickle

class OnlineLearningModel:
    """Online Learning Model"""

    def __init__(self, model, update_frequency: int = 100, decay_factor: float = 0.95):
        self.model = model
        self.update_frequency = update_frequency
        self.decay_factor = decay_factor
        self.training_samples = 0
        self.performance_window = []
        self.last_update_time = None

    def partial_fit(self, X: pd.DataFrame, y: pd.Series):
        """Incremental fitting"""
        # Models that support incremental learning
        if hasattr(self.model, 'partial_fit'):
            self.model.partial_fit(X, y)
        else:
            # For models that don't support incremental learning, special handling is needed
            self._batch_update(X, y)

        self.training_samples += len(X)
        self.last_update_time = pd.Timestamp.now()

    def _batch_update(self, X: pd.DataFrame, y: pd.Series):
        """Batch update (for models that don't support incremental learning)"""
        # Save partial historical data
        if not hasattr(self, 'historical_X'):
            self.historical_X = X
            self.historical_y = y
        else:
            # Limit historical data size
            max_samples = 10000
            if len(self.historical_X) > max_samples:
                self.historical_X = self.historical_X.iloc[-max_samples:]
                self.historical_y = self.historical_y.iloc[-max_samples:]

            # Add new data
            self.historical_X = pd.concat([self.historical_X, X], ignore_index=True)
            self.historical_y = pd.concat([self.historical_y, y], ignore_index=True)

            # Retrain
            self.model.fit(self.historical_X, self.historical_y)

    def predict(self, X: pd.DataFrame) -> np.ndarray:
        """Predict"""
        return self.model.predict(X)

    def evaluate_and_update(self, X: pd.DataFrame, y: pd.Series, threshold: float = 0.1):
        """Evaluate and decide whether to update"""
        predictions = self.predict(X)

        # Calculate performance metrics
        if hasattr(self.model, 'score'):
            score = self.model.score(X, y)
        else:
            # Custom scoring
            score = np.mean((predictions - y) ** 2)

        self.performance_window.append(score)

        # Maintain fixed window size
        if len(self.performance_window) > 10:
            self.performance_window.pop(0)

        # If performance degrades, update model
        if len(self.performance_window) >= 2:
            if self.performance_window[-1] > self.performance_window[-2] * (1 + threshold):
                self.partial_fit(X, y)
                return True

        # Periodic updates
        if self.training_samples % self.update_frequency == 0:
            self.partial_fit(X, y)
            return True

        return False

    def save_state(self, filepath: str):
        """Save model state"""
        state = {
            'model': self.model,
            'training_samples': self.training_samples,
            'performance_window': self.performance_window,
            'last_update_time': self.last_update_time
        }
        with open(filepath, 'wb') as f:
            pickle.dump(state, f)

    def load_state(self, filepath: str):
        """Load model state"""
        with open(filepath, 'rb') as f:
            state = pickle.load(f)

        self.model = state['model']
        self.training_samples = state['training_samples']
        self.performance_window = state['performance_window']
        self.last_update_time = state['last_update_time']

class ConceptDriftDetector:
    """Concept Drift Detector"""

    def __init__(self, window_size: int = 1000, threshold: float = 2.0):
        self.window_size = window_size
        self.threshold = threshold
        self.reference_distribution = None
        self.current_window = []
        self.drift_detected = False

    def add_sample(self, sample):
        """Add new sample"""
        self.current_window.append(sample)

        if len(self.current_window) > self.window_size:
            self.current_window.pop(0)

    def detect_drift(self) -> bool:
        """Detect concept drift"""
        if len(self.current_window) < self.window_size:
            return False

        if self.reference_distribution is None:
            # Set reference distribution
            self.reference_distribution = np.array(self.current_window)
            return False

        # Use Kolmogorov-Smirnov test to detect distribution changes
        from scipy import stats
        statistic, p_value = stats.ks_2samp(
            self.reference_distribution,
            self.current_window
        )

        # If p-value is very small, consider drift occurred
        if p_value < 0.05:
            self.drift_detected = True
            return True

        return False

    def reset_reference(self):
        """Reset reference distribution"""
        if self.current_window:
            self.reference_distribution = np.array(self.current_window)
            self.drift_detected = False
```

## Summary

This chapter provided an in-depth exploration of machine learning and AI applications in quantitative trading, covering:

1. **Feature Engineering**
   - Price feature extraction
   - Technical indicator calculations
   - Time series features
   - Real-time feature processing

2. **Prediction Models**
   - Regression models
   - Classification models
   - Model evaluation and selection

3. **Ensemble Learning**
   - Voting ensembles
   - Stacking ensembles
   - Adaptive weights

4. **Deep Learning**
   - LSTM models
   - Training techniques
   - Model deployment

5. **Online Learning**
   - Incremental learning
   - Concept drift detection
   - Model update strategies

Through studying this chapter, readers can build complete machine learning trading systems to achieve data-driven intelligent trading decisions.

## References

- [Advances in Financial Machine Learning](https://www.wiley.com/en-us/Advances+in+Financial+Machine+Learning)
- [Machine Learning for Algorithmic Trading](https://www.packtpub.com/product/machine-learning-for-algorithmic-trading/9781789138999)
- [Deep Learning for Time Series Forecasting](https://www.oreilly.com/library/view/deep-learning-for/9781492048316/)

## Runnable Code Entry (Source Build)

- Entry script: docs/tutorials_cn/code/chapter17/run_ml_trading.py
- Key examples: docs/tutorials_cn/code/chapter17/supervised_learning.py, docs/tutorials_cn/code/chapter17/unsupervised_learning.py, docs/tutorials_cn/code/chapter17/reinforcement_learning.py

Dependency strategy: Default path is heavy-dependency-free; optional advanced paths activate automatically when dependencies exist.

Verified command:

```bash
uv run --python 3.13 python docs/tutorials_cn/code/chapter17/run_ml_trading.py
```
