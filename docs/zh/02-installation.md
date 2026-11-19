# 第2章：环境搭建与安装

## 学习目标

通过本章学习，您将：
- 了解 NautilusTrader 的系统要求和依赖
- 掌握 Python 环境的配置方法
- 学会从不同渠道安装 NautilusTrader
- 解决常见的安装问题
- 配置合适的开发环境

## 2.1 系统要求

### 操作系统支持
NautilusTrader 支持以下操作系统：

| 操作系统 | 架构 | 状态 | 说明 |
|----------|------|------|------|
| Linux | x86_64 | ✓ 完全支持 | 推荐，性能最佳 |
| Linux | ARM64 | ✓ 完全支持 | 适用于 ARM 服务器 |
| macOS | ARM64 (Apple Silicon) | ✓ 完全支持 | 推荐 M1/M2/M3 芯片 |
| macOS | x86_64 | ✓ 支持 | 适用于 Intel 芯片 |
| Windows | x86_64 | ✓ 支持 | 64位 Windows 10/11 |

### Python 版本要求
- **最低要求**：Python 3.12
- **推荐版本**：Python 3.12 - 3.14
- **不支持**：Python < 3.12

### 硬件要求

#### 最低配置
- CPU：双核 2GHz
- 内存：4GB RAM
- 存储：10GB 可用空间
- 网络：稳定的互联网连接

#### 推荐配置
- CPU：四核 3GHz+
- 内存：16GB+ RAM
- 存储：50GB+ SSD
- 网络：低延迟连接（< 50ms）

#### 生产环境
- CPU：八核 3GHz+
- 内存：32GB+ RAM
- 存储：500GB+ NVMe SSD
- 网络：专用线路，延迟 < 10ms

### 可选依赖

#### Redis（可选）
用于缓存和消息总线持久化：
```bash
# Ubuntu/Debian
sudo apt-get install redis-server

# macOS (使用 Homebrew)
brew install redis

# Windows
# 下载并安装 Redis for Windows
```

#### Docker（可选）
用于容器化部署：
```bash
# 安装 Docker Desktop
# https://www.docker.com/products/docker-desktop
```

## 2.2 Python 环境配置

### 2.2.1 使用 pyenv 管理 Python 版本（推荐）

pyenv 是一个 Python 版本管理工具，可以轻松切换不同版本的 Python。

#### 安装 pyenv

**macOS：**
```bash
# 使用 Homebrew 安装
brew install pyenv

# 添加到 shell 配置文件
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.zshrc
echo 'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.zshrc
echo 'eval "$(pyenv init -)"' >> ~/.zshrc

# 重新加载配置
source ~/.zshrc
```

**Linux：**
```bash
# 安装依赖
sudo apt-get update
sudo apt-get install -y make build-essential libssl-dev zlib1g-dev \
    libbz2-dev libreadline-dev libsqlite3-dev wget curl llvm \
    libncurses5-dev libncursesw5-dev xz-utils tk-dev

# 克隆 pyenv 仓库
git clone https://github.com/pyenv/pyenv.git ~/.pyenv

# 添加到 shell 配置
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc

# 重新加载配置
source ~/.bashrc
```

#### 安装 Python 3.12
```bash
# 列出可安装的版本
pyenv install --list | grep "3.12"

# 安装 Python 3.12
pyenv install 3.12.7

# 设置全局默认版本
pyenv global 3.12.7

# 验证安装
python --version
# Python 3.12.7
```

### 2.2.2 使用 uv 包管理器（强烈推荐）

uv 是一个极快的 Python 包管理器，比 pip 快 10-100 倍。

#### 安装 uv

**macOS 和 Linux：**
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

**Windows (PowerShell)：**
```bash
irm https://astral.sh/uv/install.ps1 | iex
```

#### 创建虚拟环境
```bash
# 创建项目目录
mkdir nautilus_trading
cd nautilus_trading

# 创建虚拟环境
uv venv

# 激活虚拟环境
source .venv/bin/activate  # Linux/macOS
# 或
.venv\Scripts\activate     # Windows
```

## 2.3 安装 NautilusTrader

### 2.3.1 方法一：从 PyPI 安装（最简单）

这是最推荐的安装方式，适合大多数用户。

```bash
# 升级 uv 到最新版本
uv self update

# 安装 nautilus_trader
uv pip install nautilus_trader
```

#### 安装特定集成（可选）

如果您需要使用特定的交易所集成，可以安装额外的依赖：

```bash
# Interactive Brokers 支持
uv pip install "nautilus_trader[ib]"

# Docker 支持
uv pip install "nautilus_trader[docker]"

# 可视化工具
uv pip install "nautilus_trader[visualization]"

# Betfair 支持
uv pip install "nautilus_trader[betfair]"

# DydX 支持
uv pip install "nautilus_trader[dydx]"

# 安装所有额外功能
uv pip install "nautilus_trader[all]"
```

#### 验证安装
```python
# 创建测试脚本 test_install.py
import nautilus_trader as nt
from nautilus_trader.model import InstrumentId

print(f"NautilusTrader 版本: {nt.__version__}")
print("安装成功！")

# 测试基本功能
try:
    # 尝试创建一个简单的对象
    instrument_id = InstrumentId.from_str("BTCUSDT.BINANCE")
    print(f"测试通过: {instrument_id}")
except Exception as e:
    print(f"测试失败: {e}")
```

运行测试：
```bash
python test_install.py
```

### 2.3.2 方法二：从 Nautech Systems 包索引安装

这个方法提供了更多的版本选择，包括开发版本。

#### 稳定版本
```bash
# 从官方包索引安装最新稳定版
uv pip install nautilus_trader \
    --index-url https://packages.nautechsystems.io/simple
```

#### 开发版本
```bash
# 安装最新的开发版本（包含最新功能）
uv pip install --pre nautilus_trader \
    --index-url https://packages.nautechsystems.io/simple
```

#### 指定版本
```bash
# 安装特定版本
uv pip install nautilus_trader==1.220.0 \
    --index-url https://packages.nautechsystems.io/simple
```

### 2.3.3 方法三：从源码构建（开发者）

如果您需要修改源码或使用最新的未发布功能。

#### 安装构建依赖

**安装 Rust：**
```bash
# 使用 rustup 安装（推荐）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"

# 验证安装
rustc --version
cargo --version
```

**安装 Clang（Linux）：**
```bash
sudo apt-get install clang
```

**安装 Clang（Windows）：**
1. 安装 Visual Studio 2022
2. 在安装器中选择 "C++ Clang tools for Windows"

#### 克隆并构建
```bash
# 克隆仓库
git clone --branch develop https://github.com/nautechsystems/nautilus_trader.git
cd nautilus_trader

# 安装所有依赖
uv sync --all-extras

# 设置环境变量（Linux/macOS）
export PYO3_PYTHON=$(pwd)/.venv/bin/python
export PYTHONHOME=$(python -c "import sys; print(sys.base_prefix)")
```

#### 构建和安装
```bash
# 构建项目
make build

# 或使用 make 构建调试版本
make build-debug

# 安装到当前环境
uv pip install -e .
```

### 2.3.4 方法四：使用 Docker（容器化）

如果您希望使用容器化的环境。

#### 拉取镜像
```bash
# 拉取最新版本
docker pull ghcr.io/nautechsystems/nautilus_trader:latest

# 或拉取夜间版本
docker pull ghcr.io/nautechsystems/nautilus_trader:nightly
```

#### 运行容器
```bash
# 运行交互式容器
docker run -it --rm \
    -v $(pwd):/workspace \
    ghcr.io/nautechsystems/nautilus_trader:latest \
    bash

# 或使用 JupyterLab
docker run -p 8888:8888 \
    ghcr.io/nautechsystems/jupyterlab:latest
```

#### Dockerfile 示例
```dockerfile
FROM ghcr.io/nautechsystems/nautilus_trader:latest

# 安装额外的依赖
RUN pip install "nautilus_trader[all]"

# 复制您的策略代码
COPY ./strategies /app/strategies
COPY ./data /app/data

WORKDIR /app

# 运行您的策略
CMD ["python", "-m", "nautilus_trader", "run", "--config", "config.yml"]
```

## 2.4 开发环境配置

### 2.4.1 IDE 配置

#### VS Code 配置

**安装扩展：**
1. Python
2. Python Docstring Generator
3. Pylance
4. GitLens
5. Docker（如果使用 Docker）

**配置 settings.json：**
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

**配置调试器 (.vscode/launch.json)：**
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

#### PyCharm 配置

1. 打开项目
2. 设置 Python 解释器指向虚拟环境
3. 配置代码风格（Black）
4. 设置类型检查（mypy）

### 2.4.2 代码格式化和质量检查

**创建 pyproject.toml：**
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

### 2.4.3 Git 配置

创建 `.gitignore`：
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

## 2.5 常见问题解决

### 问题1：Python 版本不兼容
**错误信息：**
```
ERROR: Package requires a different Python
```

**解决方案：**
```bash
# 确认 Python 版本
python --version

# 使用 pyenv 切换到 3.12+
pyenv global 3.12.7

# 或使用 conda 创建新环境
conda create -n nautilus python=3.12
conda activate nautilus
```

### 问题2：编译错误（Linux）
**错误信息：**
```
error: invalid value for 'target-feature': 'crt-static'
```

**解决方案：**
```bash
# 安装系统依赖
sudo apt-get install build-essential

# 或使用静态链接
export RUSTFLAGS="-C target-feature=+crt-static"
uv pip install nautilus_trader
```

### 问题3：内存不足
**错误信息：**
```
MemoryError during build
```

**解决方案：**
```bash
# 增加交换空间
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# 限制并行编译
export CARGO_BUILD_JOBS=2
uv pip install nautilus_trader
```

### 问题4：网络问题（中国用户）
**错误信息：**
```
Could not fetch URL https://pypi.org/simple/...
```

**解决方案：**
```bash
# 使用国内镜像源
uv pip install nautilus_trader \
    --index-url https://pypi.tuna.tsinghua.edu.cn/simple

# 或配置 pip
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

### 问题5：权限错误
**错误信息：**
```
PermissionError: [Errno 13] Permission denied
```

**解决方案：**
```bash
# 使用用户安装
uv pip install --user nautilus_trader

# 或使用虚拟环境（推荐）
uv venv
source .venv/bin/activate
uv pip install nautilus_trader
```

## 2.6 验证安装

### 创建测试脚本

创建 `verify_installation.py`：
```python
#!/usr/bin/env python3
"""
验证 NautilusTrader 安装
"""

import sys
from pathlib import Path

def test_imports():
    """测试基础导入"""
    print("测试导入...")

    try:
        import nautilus_trader as nt
        print(f"✓ NautilusTrader {nt.__version__}")
    except ImportError as e:
        print(f"✗ 导入失败: {e}")
        return False

    try:
        from nautilus_trader.model import InstrumentId, Price, Quantity
        print("✓ 核心模型")
    except ImportError as e:
        print(f"✗ 核心模型导入失败: {e}")
        return False

    try:
        from nautilus_trader.backtest.engine import BacktestEngine
        print("✓ 回测引擎")
    except ImportError as e:
        print(f"✗ 回测引擎导入失败: {e}")
        return False

    try:
        from nautilus_trader.adapters.binance.common.enums import BinanceAccountType
        print("✓ Binance 适配器")
    except ImportError as e:
        print("⚠ Binance 适配器未安装（可选）")

    return True

def test_basic_functionality():
    """测试基本功能"""
    print("\n测试基本功能...")

    try:
        from nautilus_trader.model import InstrumentId
        from decimal import Decimal

        # 创建交易工具ID
        instrument_id = InstrumentId.from_str("BTCUSDT.BINANCE")
        print(f"✓ 创建工具ID: {instrument_id}")

        # 创建价格对象
        from nautilus_trader.model.objects import Price
        price = Price.from_str("50000.00")
        print(f"✓ 创建价格对象: {price}")

        return True
    except Exception as e:
        print(f"✗ 基本功能测试失败: {e}")
        return False

def test_performance():
    """测试性能"""
    print("\n测试性能...")

    try:
        import time
        from nautilus_trader.model.objects import Price
        from decimal import Decimal

        # 测试对象创建速度
        start = time.time()
        for _ in range(10000):
            Price.from_str("50000.00")
        elapsed = time.time() - start

        print(f"✓ 创建 10,000 个价格对象耗时: {elapsed:.3f} 秒")
        return True
    except Exception as e:
        print(f"✗ 性能测试失败: {e}")
        return False

def main():
    print("NautilusTrader 安装验证")
    print("=" * 40)

    # 检查 Python 版本
    if sys.version_info < (3, 12):
        print(f"⚠ Python 版本过低: {sys.version}")
        print("需要 Python 3.12 或更高版本")
        return

    print(f"✓ Python 版本: {sys.version.split()[0]}")

    # 运行测试
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
        print("✓ 所有测试通过！安装成功。")
    else:
        print("⚠ 部分测试失败。请检查安装。")
        sys.exit(1)

if __name__ == "__main__":
    main()
```

### 运行验证
```bash
python verify_installation.py
```

### 预期输出
```
NautilusTrader 安装验证
========================================
✓ Python 版本: 3.12.7
测试导入...
✓ NautilusTrader 1.220.0
✓ 核心模型
✓ 回测引擎
✓ Binance 适配器
测试基本功能...
✓ 创建工具ID: BTCUSDT.BINANCE
✓ 创建价格对象: 50000.00
测试性能...
✓ 创建 10,000 个价格对象耗时: 0.045 秒

========================================
✓ 所有测试通过！安装成功。
```

## 2.7 下一步

恭喜！您已经成功安装了 NautilusTrader。在下一章中，我们将：

1. 学习 NautilusTrader 的核心概念
2. 创建第一个简单的策略
3. 运行第一个回测
4. 理解基本的工作流程

## 2.8 总结

本章详细介绍了 NautilusTrader 的安装过程：

### 关键点
1. 系统要求：Python 3.12+，支持主流操作系统
2. 推荐使用 uv 包管理器，安装速度快
3. 多种安装方式：PyPI、源码构建、Docker
4. 虚拟环境隔离，避免依赖冲突
5. 验证安装确保功能正常

### 最佳实践
- 使用 pyenv 管理 Python 版本
- 使用 uv 管理包依赖
- 创建专用虚拟环境
- 配置好开发工具和环境
- 运行验证脚本确认安装

## 2.9 参考资料

- [官方安装指南](https://nautilustrader.io/docs/latest/getting_started/installation/)
- [uv 文档](https://docs.astral.sh/uv/)
- [pyenv 文档](https://github.com/pyenv/pyenv)
- [Rust 安装指南](https://www.rust-lang.org/tools/install)