# 快速开始指南

## 文件清单

已为您创建以下文件：

1. **strategy.pine** (5.0K) - 基础策略脚本
2. **strategy_enhanced.pine** (7.0K) - 增强版策略（推荐使用）
3. **vegas_channel_lib.pine** (3.0K) - 可重用函数库
4. **backtest_lib.pine** (5.0K) - 回测分析工具库
5. **README.md** - 完整文档
6. **QUICKSTART.md** - 本文件

## 5分钟快速上手

### 第一步：选择策略版本

**推荐使用增强版本** - 功能更完善，参数可调

### 第二步：在TradingView上运行

1. 登录 [TradingView](https://www.tradingview.com)
2. 打开任意交易对的4小时图表
3. 点击 `Pine Editor`
4. 选择 `New indicator` 或 `New strategy`
5. 复制 `strategy_enhanced.pine` 的全部代码到编辑器
6. 点击 `Add to Chart`

### 第三步：调整参数

在策略设置中调整以下参数：

```
Slope Threshold: 25 (斜率阈值，单位度数)
Slope Period: 28 (斜率计算周期)
Price Array Threshold: 8 (触发移动止损的记录数)
Stop Loss %: 5 (固定止损百分比)
Take Profit 1-3 %: 10/15/25 (分层止盈)
```

### 第四步：回测

1. 点击 `Strategy Tester`
2. 选择回测时间段和初始资金
3. 点击 `Run Backtest`
4. 查看性能指标

## 核心逻辑速览

### 多单条件 (Duo1)
```
空头趋势 + 斜率 < 25°
→ EMA12上穿EMA144
→ 开多单
```

### 空单条件 (Kong1)
```
空头趋势 + 斜率 > 25°
→ 价格接近EMA144/EMA156
→ 开空单
```

### 止损止盈
- 多单: 止损 -5%, 止盈 +10%/+15%/+25%
- 空单: 止损 +5%, 止盈 -10%/-15%/-25%
- 自适应移动止损: 基于价格数组计算

## 常见问题

**Q: 应该在哪个时间框架上使用？**
A: 4小时（4H）时间框架，这是策略的设计基础

**Q: 可以改变参数吗？**
A: 可以，但建议先在回测中验证参数的有效性

**Q: 支持哪些交易品种？**
A: 适用于所有主要交易品种（外汇、加密货币、股票期货等）

**Q: 如何集成到其他策略？**
A: 使用 `vegas_channel_lib.pine` 中的函数进行集成

## 文件用途对应表

| 用途 | 文件 |
|------|------|
| 学习/演示 | strategy.pine |
| 实际交易 | strategy_enhanced.pine |
| 开发新策略 | vegas_channel_lib.pine |
| 性能分析 | backtest_lib.pine |
| 文档参考 | README.md |

## 下一步

1. 在TradingView上测试策略
2. 运行至少3个月的历史回测
3. 查看性能指标和交易列表
4. 根据需要微调参数
5. 在小仓位上进行实盘测试

## 技术支持

- 对于Pine Script语法问题，参考 [Pine Script文档](https://www.tradingview.com/pine-script-docs/)
- 对于策略逻辑问题，参考 README.md 中的详细说明
- 对于函数库用法，参考各库文件中的导出函数注释

---

祝您交易顺利！
