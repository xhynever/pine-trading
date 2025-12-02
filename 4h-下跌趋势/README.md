# 4小时维加斯通道下跌趋势策略

## 项目概述

基于4小时维加斯通道的自动化交易策略，使用Pine Script v5编写，适用于TradingView。该策略包含完整的多单和空单逻辑，支持自适应移动止损和分层止盈。

## 文件说明

### 1. `strategy.pine` - 基础策略
基础版本的完整策略实现，包含所有核心逻辑。

下面主要操作是基于4h的走势图。除非特殊强调，才使用额外说明的。
记录日线的EMA指标（EMA12, EMA144, EMA156, EMA565, EMA676） 1d
**功能：**

- 计算5条EMA指标（EMA12, EMA144, EMA156, EMA565, EMA676） 4h
- 检测空头趋势信号。
- 多单开单逻辑（Duo1信号）
- 空单开单逻辑（Kong1信号）
- 自适应移动止损
- 分层止盈管理

### 2. `strategy_enhanced.pine` - 增强版策略
集成了库函数的增强版本，提供更多参数定制和信息面板。

**改进：**
- 导入和使用vegas_channel_lib库
- 可调整的策略参数
- 实时信息面板显示策略状态
- 更好的代码组织和可维护性

### 3. `vegas_channel_lib.pine` - 维加斯通道库
可重用的函数库，提供所有核心计算功能。

**包含的导出函数：**
- `f_calculate_emas()` - 计算所有EMA指标
- `f_is_downtrend()` - 检测空头趋势
- `f_calculate_slope()` - 计算斜率（线性回归）：
这个斜率，变更。在计算时，采用循环队列。容量是48根。
向队列插入时，记录最大值，最小值。 
计算斜率时，记录最小值和最大值的波动。
- `f_slope_below_threshold()` - 判断斜率是否小于阈值：
- `f_slope_above_threshold()` - 判断斜率是否大于阈值：
- `f_record_price()` - 记录收盘价到数组
- `f_calculate_trailing_stop()` - 计算移动止损
- `f_get_take_profit_levels()` - 获取止盈价位
- `f_get_stop_loss()` - 获取止损价位

### 4. `backtest_lib.pine` - 回测函数库
提供策略性能分析和回测统计的工具函数。

**包含的导出函数：**
- `f_init_backtest_stats()` - 初始化统计对象
- `f_record_trade()` - 记录单笔交易
- `f_calculate_win_rate()` - 计算胜率
- `f_calculate_total_profit()` - 计算总收益
- `f_calculate_avg_profit()` - 计算平均利润
- `f_calculate_max_drawdown()` - 计算最大回撤
- `f_calculate_sharpe_ratio()` - 计算夏普比率
- `f_calculate_profit_factor()` - 计算盈亏比
- `f_generate_backtest_report()` - 生成回测报告


## 核心策略逻辑

### 1. 空头趋势检测
```
条件: ema12 < min(ema144, ema156) < min(ema565, ema676)
```
辅助参数
EMA12上穿EMA144。 第几次上穿，需要记录。  参数为 scema
现价上穿ema144
现价上穿ema565
4h ema150,4h ema600
注：这里的现价应该包括，4h的最高点，最低点。

### 2. 多单逻辑（Duo1）
1. 检测：空头趋势 + EMA144斜率 小于 15%
2. 标记为Duo1
3.当现价上穿ema144，第一次上传无效。 即  scema>1
如下条件，设置开单价：
 高于ema12 2%
 等于ema12
开单数量0.1
4. 止损：每单止损固定5%或基于lowprice数组平均值的自适应止损
5. 止盈：
   - 上涨10%平仓50%
   - 上涨15%平仓25%
   - 上涨25%平仓25%

### 3. 空单逻辑（Kong1）
1. 检测：空头趋势 + EMA144斜率 > 15%
2. 标记为Kong1
3. 满足下面的条件时，开仓0.1.
条件：
低于ema144 2%的价格，
等于ema144
等于ema150
等于ema156
低于mea565 2%的价格，
ema565
ema565
ema600
4. 止损：固定5%或基于highprice数组平均值的自适应止损
5. 止盈：
   - 下跌10%平仓50%
   - 下跌15%平仓25%
   - 下跌25%平仓25%

### 4. 自适应移动止损
- 当收集的价格总数 > 8根时启动
- 多单：高于EMA12则加入highprice，低于则加入lowprice
  - 若highprice数 < lowprice数：移动止损为成本价
  - 否则：移动止损为lowprice平均值
- 空单：逻辑相反

## 使用方法

### 在TradingView上使用

1. **基础版本：**
   - 将 `strategy.pine` 的代码复制到TradingView Pine编辑器
   - 选择合适的交易对和4小时时间框架
   - 运行回测

2. **增强版本：**
   - 先在编辑器中创建 `vegas_channel_lib` 库
   - 然后创建 `strategy_enhanced.pine` 策略脚本
   - 调整参数以适应您的交易品种

### 参数说明（增强版本）

| 参数 | 默认值 | 说明 |
|------|--------|------|
| Slope Threshold | 25 | 斜率判断阈值（度数） |
| Slope Period | 28 | 计算斜率的周期 |
| Price Array Threshold | 8 | 触发移动止损的价格记录数 |
| Stop Loss % | 5 | 固定止损百分比 |
| Take Profit 1 % | 10 | 第一个止盈点 |
| Take Profit 2 % | 15 | 第二个止盈点 |
| Take Profit 3 % | 25 | 第三个止盈点 |

## 回测和性能分析

### 使用回测库进行分析

```pine
import backtest_lib as btlib

// 在您的策略中集成：
var array<TradeRecord> trades = array.new<TradeRecord>()

// 记录交易
btlib.f_record_trade(trades, trade_id, entry, exit, trade_type)

// 生成报告
stats = btlib.f_generate_backtest_report(trades)
```

### 性能指标

- **胜率** - 盈利交易占比
- **总收益** - 所有交易的总盈亏
- **平均利润** - 单笔交易平均收益
- **最大回撤** - 最大期间回撤
- **夏普比率** - 风险调整后收益
- **盈亏比** - 总盈利/总亏损

## 注意事项

1. **回测周期** - 建议在4小时时间框架上使用
2. **市场条件** - 策略专为下跌趋势优化，需考虑市场环境
3. **风险管理** - 建议结合位置大小和风险管理规则使用
4. **参数优化** - 应根据具体交易品种进行回测和优化
5. **滑点和费用** - 回测时应考虑实际交易成本

## 扩展和自定义

### 添加新的指标
在 `vegas_channel_lib` 中添加导出函数：

```pine
export f_custom_indicator() =>
    // 您的指标逻辑
```

### 修改止盈止损逻辑
编辑策略中的止盈止损部分，或使用 `backtest_lib` 中的计算函数。

### 集成其他策略模块
利用库函数接口，轻松集成其他交易模块。

## 免责声明

本策略仅供参考和教育用途。过去的表现不代表未来结果。在实盘交易前，请充分理解策略逻辑并进行充分回测。
