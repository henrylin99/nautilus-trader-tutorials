---
title: "Chapter 2: Environment Setup and Installation"
---

# Chapter 2: Environment Setup and Installation

## Learning Objectives

After completing this chapter, you will:
- Understand NautilusTrader's system requirements and dependencies
- Master Python environment configuration methods
- Learn to install NautilusTrader from different sources
- Solve common installation issues
- Configure an appropriate development environment

## 2.1 System Requirements

### Operating System Support
NautilusTrader supports the following operating systems:

| Operating System | Architecture | Status | Description |
|------------------|--------------|--------|-------------|
| Linux | x86_64 | ✓ Fully supported | Recommended, best performance |
| Linux | ARM64 | ✓ Fully supported | For ARM servers |
| macOS | ARM64 (Apple Silicon) | ✓ Fully supported | Recommended for M1/M2/M3 chips |
| macOS | x86_64 | ✓ Supported | For Intel chips |
| Windows | x86_64 | ✓ Supported | 64-bit Windows 10/11 |

### Python Version Requirements
- **Minimum requirement**: Python 3.12
- **Recommended version**: Python 3.12 - 3.14
- **Not supported**: Python < 3.12

### Hardware Requirements

#### Minimum Configuration
- CPU: Dual-core 2GHz
- Memory: 4GB RAM
- Storage: 10GB available space
- Network: Stable internet connection

#### Recommended Configuration
- CPU: Quad-core 3GHz+
- Memory: 16GB+ RAM
- Storage: 50GB+ SSD
- Network: Low-latency connection (< 50ms)

#### Production Environment
- CPU: Octa-core 3GHz+
- Memory: 32GB+ RAM
- Storage: 500GB+ NVMe SSD
- Network: Dedicated line, latency < 10ms

### Optional Dependencies

#### Redis (Optional)
For caching and message bus persistence:
```bash
# Ubuntu/Debian
sudo apt-get install redis-server

# macOS (using Homebrew)
brew install redis

# Windows
# Download and install Redis for Windows
```

#### Docker (Optional)
For containerized deployment:
```bash
# Install Docker Desktop
# https://www.docker.com/products/docker-desktop
```

## 2.2 Python Environment Configuration

### 2.2.1 Using pyenv to Manage Python Versions (Recommended)

pyenv is a Python version management tool that allows easy switching between different Python versions.

#### Installing pyenv

**macOS:**
```bash
# Install using Homebrew
brew install pyenv

# Add to shell configuration file
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.zshrc
echo 'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.zshrc
echo 'eval "$(pyenv init -)"' >> ~/.zshrc

# Reload configuration
source ~/.zshrc
```

**Linux:**
```bash
# Install dependencies
sudo apt-get update
sudo apt-get install -y make build-essential libssl-dev zlib1g-dev \
    libbz2-dev libreadline-dev libsqlite3-dev wget curl llvm \
    libncurses5-dev libncursesw5-dev xz-utils tk-dev

# Clone pyenv repository
git clone https://github.com/pyenv/pyenv.git ~/.pyenv

# Add to shell configuration
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc

# Reload configuration
source ~/.bashrc
```

#### Installing Python 3.12
```bash
# List available versions
pyenv install --list | grep "3.12"

# Install Python 3.12
pyenv install 3.12.7

# Set global default version
pyenv global 3.12.7

# Verify installation
python --version
# Python 3.12.7
```

### 2.2.2 Using uv Package Manager (Highly Recommended)

uv is an extremely fast Python package manager, 10-100x faster than pip.

#### Installing uv

**macOS and Linux:**
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

**Windows (PowerShell):**
```bash
irm https://astral.sh/uv/install.ps1 | iex
```

#### Creating Virtual Environment
```bash
# Create project directory
mkdir nautilus_trading
cd nautilus_trading

# Create virtual environment
uv venv

# Activate virtual environment
source .venv/bin/activate  # Linux/macOS
# or
.venv\Scripts\activate     # Windows
```

## 2.3 Installing NautilusTrader

### 2.3.1 Method 1: Install from PyPI (Easiest)

This is the most recommended installation method for most users.

```bash
# Upgrade uv to latest version
uv self update

# Install nautilus_trader
uv pip install nautilus_trader
```

#### Installing Specific Integrations (Optional)

If you need to use specific exchange integrations, you can install additional dependencies:

```bash
# Interactive Brokers support
uv pip install "nautilus_trader[ib]"

# Docker support
uv pip install "nautilus_trader[docker]"

# Visualization tools
uv pip install "nautilus_trader[visualization]"

# Betfair support
uv pip install "nautilus_trader[betfair]"

# DydX support
uv pip install "nautilus_trader[dydx]"

# Install all extra features
uv pip install "nautilus_trader[all]"
```

#### Verifying Installation
```python
# Create test script test_install.py
import nautilus_trader as nt
from nautilus_trader.model import InstrumentId

print(f"NautilusTrader version: {nt.__version__}")
print("Installation successful!")

# Test basic functionality
try:
    # Try to create a simple object
    instrument_id = InstrumentId.from_str("BTCUSDT.BINANCE")
    print(f"Test passed: {instrument_id}")
except Exception as e:
    print(f"Test failed: {e}")
```

Run the test:
```bash
python test_install.py
```

### 2.3.2 Method 2: Install from Nautech Systems Package Index

This method provides more version options, including development versions.

#### Stable Version
```bash
# Install latest stable version from official package index
uv pip install nautilus_trader \
    --index-url https://packages.nautechsystems.io/simple
```

#### Development Version
```bash
# Install latest development version (includes latest features)
uv pip install --pre nautilus_trader \
    --index-url https://packages.nautechsystems.io/simple
```

#### Specific Version
```bash
# Install specific version
uv pip install nautilus_trader==1.220.0 \
    --index-url https://packages.nautechsystems.io/simple
```

### 2.3.3 Method 3: Build from Source (Developers)

If you need to modify source code or use the latest unreleased features.

#### Installing Build Dependencies

**Installing Rust:**
```bash
# Install using rustup (recommended)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"

# Verify installation
rustc --version
cargo --version
```

**Installing Clang (Linux):**
```bash
sudo apt-get install clang
```

**Installing Clang (Windows):**
1. Install Visual Studio 2022
2. Select "C++ Clang tools for Windows" in the installer

#### Clone and Build
```bash
# Clone repository
git clone --branch develop https://github.com/nautechsystems/nautilus_trader.git
cd nautilus_trader

# Install all dependencies
uv sync --all-extras

# Set environment variables (Linux/macOS)
export PYO3_PYTHON=$(pwd)/.venv/bin/python
export PYTHONHOME=$(python -c "import sys; print(sys.base_prefix)")
```

#### Build and Install
```bash
# Build project
make build

# Or use make to build debug version
make build-debug

# Install to current environment
uv pip install -e .
```

### 2.3.4 Method 4: Using Docker (Containerization)

If you want to use a containerized environment.

#### Pull Image
```bash
# Pull latest version
docker pull ghcr.io/nautechsystems/nautilus_trader:latest

# Or pull nightly version
docker pull ghcr.io/nautechsystems/nautilus_trader:nightly
```

#### Run Container
```bash
# Run interactive container
docker run -it --rm \
    -v $(pwd):/workspace \
    ghcr.io/nautechsystems/nautilus_trader:latest \
    bash

# Or use JupyterLab
docker run -p 8888:8888 \
    ghcr.io/nautechsystems/jupyterlab:latest
```

#### Dockerfile Example
```dockerfile
FROM ghcr.io/nautechsystems/nautilus_trader:latest

# Install additional dependencies
RUN pip install "nautilus_trader[all]"

# Copy your strategy code
COPY ./strategies /app/strategies
COPY ./data /app/data

WORKDIR /app

# Run your strategy
CMD ["python", "-m", "nautilus_trader", "run", "--config", "config.yml"]
```

## 2.4 Development Environment Configuration

### 2.4.1 IDE Configuration

#### VS Code Configuration

**Install extensions:**
1. Python
2. Python Docstring Generator
3. Pylance
4. GitLens
5. Docker (if using Docker)

**Configure settings.json:**
```json
{
    "python.defaultInterpreterPath": "./.venv/bin/python",
    "python.linting.enabled": true,
    "python.linting.pylintEnabled": true,
    "python.formatting.provider": "black",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
        "source.organizeImports": true
    }
}
```

**Configure debugger (.vscode/launch.json):**
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug Strategy",
            "type": "python",
            "request": "launch",
            "program": "${workspaceFolder}/run_strategy.py",
            "console": "integratedTerminal",
            "cwd": "${workspaceFolder}",
            "env": {
                "PYTHONPATH": "${workspaceFolder}"
            }
        }
    ]
}
```

#### PyCharm Configuration

1. Open project
2. Set Python interpreter to point to virtual environment
3. Configure code style (Black)
4. Set up type checking (mypy)

### 2.4.2 Code Formatting and Quality Checking

**Create pyproject.toml:**
```toml
[tool.black]
line-length = 100
target-version = ['py312']
include = '\.pyi?$'
extend-exclude = '''
/(
  # directories
  \.eggs
  | \.git
  | \.hg
  | \.mypy_cache
  | \.tox
  | \.venv
  | build
  | dist
)/
'''

[tool.isort]
profile = "black"
multi_line_output = 3
line_length = 100
include_trailing_comma = true

[tool.mypy]
python_version = "3.12"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true

[tool.ruff]
line-length = 100
select = ["E", "F", "W", "C90"]
ignore = ["E501"]
```

### 2.4.3 Git Configuration

Create `.gitignore`:
```gitignore
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
env/
venv/
ENV/
env.bak/
venv.bak/

# IDEs
.vscode/
.idea/
*.swp
*.swo
*~

# Jupyter Notebook
.ipynb_checkpoints

# OS
.DS_Store
Thumbs.db

# NautilusTrader specific
logs/
data/
*.log
config_live.yml
```

## 2.5 Common Issues and Solutions

### Issue 1: Python Version Incompatibility
**Error message:**
```
ERROR: Package requires a different Python
```

**Solution:**
```bash
# Check Python version
python --version

# Switch to 3.12+ using pyenv
pyenv global 3.12.7

# Or create new environment using conda
conda create -n nautilus python=3.12
conda activate nautilus
```

### Issue 2: Compilation Error (Linux)
**Error message:**
```
error: invalid value for 'target-feature': 'crt-static'
```

**Solution:**
```bash
# Install system dependencies
sudo apt-get install build-essential

# Or use static linking
export RUSTFLAGS="-C target-feature=+crt-static"
uv pip install nautilus_trader
```

### Issue 3: Insufficient Memory
**Error message:**
```
MemoryError during build
```

**Solution:**
```bash
# Increase swap space
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Limit parallel compilation
export CARGO_BUILD_JOBS=2
uv pip install nautilus_trader
```

### Issue 4: Network Issues (Chinese Users)
**Error message:**
```
Could not fetch URL https://pypi.org/simple/...
```

**Solution:**
```bash
# Use domestic mirror
uv pip install nautilus_trader \
    --index-url https://pypi.tuna.tsinghua.edu.cn/simple

# Or configure pip
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

### Issue 5: Permission Error
**Error message:**
```
PermissionError: [Errno 13] Permission denied
```

**Solution:**
```bash
# Use user installation
uv pip install --user nautilus_trader

# Or use virtual environment (recommended)
uv venv
source .venv/bin/activate
uv pip install nautilus_trader
```

## 2.6 Verifying Installation

### Create Test Script

Create `verify_installation.py`:
```python
#!/usr/bin/env python3
"""
Verify NautilusTrader Installation
"""

import sys
from pathlib import Path

def test_imports():
    """Test basic imports"""
    print("Testing imports...")

    try:
        import nautilus_trader as nt
        print(f"✓ NautilusTrader {nt.__version__}")
    except ImportError as e:
        print(f"✗ Import failed: {e}")
        return False

    try:
        from nautilus_trader.model import InstrumentId, Price, Quantity
        print("✓ Core models")
    except ImportError as e:
        print(f"✗ Core models import failed: {e}")
        return False

    try:
        from nautilus_trader.backtest.engine import BacktestEngine
        print("✓ Backtest engine")
    except ImportError as e:
        print(f"✗ Backtest engine import failed: {e}")
        return False

    try:
        from nautilus_trader.adapters.binance.common.enums import BinanceAccountType
        print("✓ Binance adapter")
    except ImportError as e:
        print("⚠ Binance adapter not installed (optional)")

    return True

def test_basic_functionality():
    """Test basic functionality"""
    print("\nTesting basic functionality...")

    try:
        from nautilus_trader.model import InstrumentId
        from decimal import Decimal

        # Create instrument ID
        instrument_id = InstrumentId.from_str("BTCUSDT.BINANCE")
        print(f"✓ Created instrument ID: {instrument_id}")

        # Create price object
        from nautilus_trader.model.objects import Price
        price = Price.from_str("50000.00")
        print(f"✓ Created price object: {price}")

        return True
    except Exception as e:
        print(f"✗ Basic functionality test failed: {e}")
        return False

def test_performance():
    """Test performance"""
    print("\nTesting performance...")

    try:
        import time
        from nautilus_trader.model.objects import Price
        from decimal import Decimal

        # Test object creation speed
        start = time.time()
        for _ in range(10000):
            Price.from_str("50000.00")
        elapsed = time.time() - start

        print(f"✓ Created 10,000 price objects in {elapsed:.3f} seconds")
        return True
    except Exception as e:
        print(f"✗ Performance test failed: {e}")
        return False

def main():
    print("NautilusTrader Installation Verification")
    print("=" * 40)

    # Check Python version
    if sys.version_info < (3, 12):
        print(f"⚠ Python version too low: {sys.version}")
        print("Python 3.12 or higher is required")
        return

    print(f"✓ Python version: {sys.version.split()[0]}")

    # Run tests
    tests = [
        test_imports,
        test_basic_functionality,
        test_performance,
    ]

    all_passed = True
    for test in tests:
        if not test():
            all_passed = False

    print("\n" + "=" * 40)
    if all_passed:
        print("✓ All tests passed! Installation successful.")
    else:
        print("⚠ Some tests failed. Please check installation.")
        sys.exit(1)

if __name__ == "__main__":
    main()
```

### Run Verification
```bash
python verify_installation.py
```

### Expected Output
```
NautilusTrader Installation Verification
========================================
✓ Python version: 3.12.7
Testing imports...
✓ NautilusTrader 1.220.0
✓ Core models
✓ Backtest engine
✓ Binance adapter
Testing basic functionality...
✓ Created instrument ID: BTCUSDT.BINANCE
✓ Created price object: 50000.00
Testing performance...
✓ Created 10,000 price objects in 0.045 seconds

========================================
✓ All tests passed! Installation successful.
```

## 2.7 Next Steps

Congratulations! You have successfully installed NautilusTrader. In the next chapter, we will:

1. Learn NautilusTrader's core concepts
2. Create your first simple strategy
3. Run your first backtest
4. Understand the basic workflow

## 2.8 Summary

This chapter detailed the NautilusTrader installation process:

### Key Points
1. System requirements: Python 3.12+, supports mainstream operating systems
2. Recommend using uv package manager for fast installation
3. Multiple installation methods: PyPI, source build, Docker
4. Virtual environment isolation to avoid dependency conflicts
5. Verify installation to ensure functionality works

### Best Practices
- Use pyenv to manage Python versions
- Use uv to manage package dependencies
- Create dedicated virtual environments
- Configure development tools and environment properly
- Run verification scripts to confirm installation

## 2.9 References

- [Official Installation Guide](https://nautilustrader.io/docs/latest/getting_started/installation/)
- [uv Documentation](https://docs.astral.sh/uv/)
- [pyenv Documentation](https://github.com/pyenv/pyenv)
- [Rust Installation Guide](https://www.rust-lang.org/tools/install)