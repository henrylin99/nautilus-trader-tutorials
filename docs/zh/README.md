# NautilusTrader 中文教程

本目录包含 NautilusTrader 平台的中文教程，基于官方文档和示例编写，适合不同水平的用户学习。

## 教程结构

### 第一部分：入门基础（适合初学者）
- [第1章：NautilusTrader 简介](./01-introduction.md)
- [第2章：环境搭建与安装](./02-installation.md)
- [第3章：快速开始](./03-quickstart.md)

### 第二部分：回测基础（适合初学者到中级）
- [第4章：数据导入与处理](./04-data-import.md)
- [第5章：基础回测引擎使用](./05-backtest-engine.md)
- [第6章：时间管理和事件处理](./06-time-events.md)

### 第三部分：策略开发（适合中级用户）
- [第7章：技术指标系统](./07-indicators.md)
- [第8章：订单管理系统](./08-order-management.md)
- [第9章：投资组合与风险管理](./09-portfolio-risk.md)

### 第四部分：高级数据处理（适合中级到高级用户）
- [第10章：高级数据处理](./10-advanced-data.md)

### 第五部分：核心系统（适合中级到高级用户）
- [第11章：消息传递与Actor模型](./11-message-actor.md)
- [第12章：性能优化](./12-performance-optimization.md)
- [第13章：实盘交易架构](./13-live-trading-architecture.md)

### 第六部分：交易系统（适合高级用户）
- [第14章：交易所集成](./14-exchange-integration.md)
- [第15章：实盘策略部署](./15-live-strategy-deployment.md)

### 第七部分：策略开发（适合中级到高级用户）
- [第16章：算法交易策略](./16-algorithmic-trading-strategies.md)

### 第八部分：智能技术（适合高级用户）
- [第17章：机器学习和AI](./17-machine-learning-ai.md)

### 第九部分：系统扩展（适合高级用户）
- [第18章：自定义组件开发](./18-custom-component-development.md)

### 第十部分：项目实战（综合应用）
- [第19章：完整项目示例1：加密货币做市策略](./19-complete-project-example-1.md)
- [第20章：完整项目示例2：多策略量化基金](./20-complete-project-example-2.md)

## 学习路径建议

### 初学者路径（2-3个月）
1. 完成第一部分（第1-3章）
2. 学习第二部分（第4-6章）
3. 实践简单的策略开发

### 中级开发者路径（3-4个月）
1. 完成第三部分（第7-9章）
2. 深入第四部分（第10-12章）
3. 开发多策略组合

### 高级用户路径（4-6个月）
1. 掌握第五部分（第13-15章）
2. 研究第六部分（第16-18章）
3. 构建生产级系统

## 教程特色

- **循序渐进**：从基础概念到高级应用
- **实践导向**：每个章节都包含可运行的代码示例
- **详细注释**：代码都有中文注释说明
- **最佳实践**：包含实际项目经验总结
- **持续更新**：跟踪 NautilusTrader 最新版本

## 代码示例

### 配套代码目录
各章节的配套代码位于 `code/` 目录下：

- [第3章 快速开始](./code/chapter03/) - 基础回测示例
- [第6章 时间管理和事件处理](./code/chapter06/) - 事件驱动编程
- [第7章 技术指标系统](./code/chapter07/) - 自定义指标实现
- [第8章 订单管理系统](./code/chapter08/) - 订单生命周期管理
- [第9章 投资组合与风险管理](./code/chapter09/) - 多资产组合管理
- [第10章 高级数据处理](./code/chapter10/) - Parquet数据和外部数据源
- [第11章 执行算法](./code/chapter11/) - TWAP/VWAP等算法交易（入口：`run_execution_algorithms.py`）
- [第12章 机器学习](./code/chapter12/) - ML策略开发框架（入口：`run_ml_examples.py`）
- [第13章 实盘交易](./code/chapter13/) - 实盘系统配置和部署（入口：`run_live_trading.py`）
- [第14章 性能分析](./code/chapter14/) - 策略回测分析工具（入口：`run_performance_analysis.py`）
- [第15章 高频交易](./code/chapter15/) - 订单簿管理和做市策略（入口：`run_hf_trading.py`）
- [第16章 算法交易策略](./code/chapter16/) - 均值回归、动量和配对交易（入口：`run_algorithmic_trading.py`）
- [第17章 机器学习和AI](./code/chapter17/) - 强化学习、监督学习和无监督学习（入口：`run_ml_trading.py`）
- [第18章 自定义组件开发](./code/chapter18/) - 自定义指标、数据源和执行算法（入口：`run_custom_components.py`）
- [第19章 加密货币做市策略](./code/chapter19/) - 数字货币市场做市交易（入口：`run_crypto_mm.py`）
- [第20章 多策略量化基金](./code/chapter20/) - 量化基金组合管理系统（入口：`run_quant_fund.py`）

### 官方示例
- 回测示例：`/examples/backtest/`
- 实盘示例：`/examples/live/`
- 策略示例：`/examples/strategies/`

## 配套资源

- [NautilusTrader 官方文档](https://nautilustrader.io/docs/)
- [GitHub 仓库](https://github.com/nautechsystems/nautilus_trader)
- [Discord 社区](https://discord.gg/NautilusTrader)
- [NautilusTrader 功能和使用指南](../NautilusTrader功能和使用指南.md)
- [NautilusTrader 教程大纲](../NautilusTrader教程大纲.md)

## 反馈与贡献

如果您在学习过程中遇到问题或有改进建议，请：

1. 查看常见问题解答
2. 在 GitHub 提交 Issue
3. 加入 Discord 社区讨论
4. 贡献您的教程内容

## 注意事项

- 教程基于 NautilusTrader v1.220+ 版本编写
- 示例代码仅供参考，实盘交易需要谨慎
- 建议在测试环境充分验证后再进行实盘部署
- 交易有风险，投资需谨慎

---

祝您学习愉快！
