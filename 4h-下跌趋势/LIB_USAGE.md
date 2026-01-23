# Vegas Channel 策略库使用说明

## 概述

`vegas_channel_lib.pine` 是一个Pine Script函数库，包含Vegas Channel策略常用的工具函数。这些函数可以用于快速构建基于Vegas Channel的交易策略。

## 库文件位置

```
4h-下跌趋势/vegas_channel_lib.pine
```

## 函数列表

### 1. EMA计算函数

#### `f_calculate_emas(len12, len144, len156, len565, len676)`

**功能**: 计算5条EMA指标（Vegas Channel标准配置）

**参数**:
- `len12`: EMA12的周期（默认12）
- `len144`: EMA144的周期（默认144）
- `len156`: EMA156的周期（默认156）
- `len565`: EMA565的周期（默认565）
- `len676`: EMA676的周期（默认676）

**返回值**: `[ema12, ema144, ema156, ema565, ema676]` - 5条EMA值的数组

**使用示例**:
```pine
[ema12, ema144, ema156, ema565, ema676] = f_calculate_emas(12, 144, 156, 565, 676)
```

---

### 2. 趋势判断函数

#### `f_is_downtrend(ema12, ema144, ema156, ema565, ema676)`

**功能**: 检测空头趋势（Vegas Channel标准判断）

**判断条件**: `ema12 < min(ema144, ema156) < min(ema565, ema676)`

**返回值**: `bool` - true表示空头趋势，false表示非空头趋势

**使用示例**:
```pine
downtrend = f_is_downtrend(ema12, ema144, ema156, ema565, ema676)
if downtrend
    // 执行空头策略逻辑
```

#### `f_is_uptrend(ema12, ema144, ema156, ema565, ema676)`

**功能**: 检测多头趋势（Vegas Channel标准判断）

**判断条件**: `ema12 > max(ema144, ema156) > max(ema565, ema676)`

**返回值**: `bool` - true表示多头趋势，false表示非多头趋势

**使用示例**:
```pine
uptrend = f_is_uptrend(ema12, ema144, ema156, ema565, ema676)
if uptrend
    // 执行多头策略逻辑
```

---

### 3. 波动率计算函数

#### `f_calculate_volatility(period)`

**功能**: 计算真实波幅指标(ATR) - 反映市场波动性

**参数**:
- `period`: ATR计算周期（K线根数），默认14

**返回值**: `float` - ATR值，代表市场波动强度

**说明**:
- ATR值越大，市场波动越剧烈
- ATR值越小，市场波动越平缓
- 建议值范围：14-20（根据策略需求调整）

**使用示例**:
```pine
atr_value = f_calculate_volatility(14)
if atr_value > 100
    // 波动率高，适合空单
else
    // 波动率低，适合多单
```

---

### 4. 价格记录函数

#### `f_record_price(price, threshold, high_arr, low_arr, capacity)`

**功能**: 记录价格到数组（根据阈值分类）

**参数**:
- `price`: 要记录的价格（通常使用close）
- `threshold`: 阈值（通常使用EMA12）
- `high_arr`: 高于阈值的价格数组（引用传递，会被修改）
- `low_arr`: 低于阈值的价格数组（引用传递，会被修改）
- `capacity`: 数组最大容量，超过容量会删除最早的数据

**返回值**: `void` - 无返回值，直接修改传入的数组

**说明**:
- 如果 `price > threshold`，将price添加到high_arr
- 如果 `price <= threshold`，将price添加到low_arr
- 数组采用循环队列管理，超过容量自动删除最早数据
- 建议capacity值：48（约2天的4H数据）

**使用示例**:
```pine
var array<float> highprice = array.new<float>()
var array<float> lowprice = array.new<float>()

// 每根K线调用一次
if strategy.position_size > 0
    f_record_price(close, ema12, highprice, lowprice, 48)
```

---

### 5. 移动止损计算函数

#### `f_calculate_trailing_sl(is_long, entry_price, high_arr, low_arr, threshold)`

**功能**: 计算移动止损价格（基于价格数组的平均值）

**参数**:
- `is_long`: 是否为多单（true=多单, false=空单）
- `entry_price`: 入场价格
- `high_arr`: 高于阈值的价格数组
- `low_arr`: 低于阈值的价格数组
- `threshold`: 价格记录阈值（K线数），低于此值返回入场价

**返回值**: `float` - 计算出的止损价格

**说明**:
- **多单**: 使用low_arr的平均值作为止损（价格下跌时止损）
- **空单**: 使用high_arr的平均值作为止损（价格上涨时止损）
- 如果数组总数 <= threshold，返回入场价（不设置止损）
- 如果数组为空或不符合条件，返回入场价

**使用示例**:
```pine
// 多单止损
if strategy.position_size > 0
    trailing_sl = f_calculate_trailing_sl(true, entry_price, highprice, lowprice, 8)
    strategy.exit("SL", stop=trailing_sl)

// 空单止损
if strategy.position_size < 0
    trailing_sl = f_calculate_trailing_sl(false, entry_price, highprice, lowprice, 8)
    strategy.exit("SL", stop=trailing_sl)
```

---

### 6. 价格接近EMA检查函数

#### `f_check_price_near_ema(price, ema, offset_percent)`

**功能**: 检查价格是否接近指定EMA（容差范围）

**参数**:
- `price`: 要检查的价格（通常使用close）
- `ema`: EMA的值
- `offset_percent`: 容差百分比（例如2.0表示±2%）

**返回值**: `bool` - true表示价格在EMA的容差范围内

**判断条件**: `ema * (1 - offset_percent/100) <= price <= ema * (1 + offset_percent/100)`

**使用示例**:
```pine
if f_check_price_near_ema(close, ema12, 2.0)
    // 价格在EMA12的±2%范围内，可以开仓
    strategy.entry("Long", strategy.long)
```

#### `f_check_entry_price_short(price, ema144, ema150, ema156, ema565, ema600, offset_percent)`

**功能**: 检查价格是否在多个EMA范围内（用于空单开仓）

**参数**:
- `price`: 要检查的价格（通常使用close）
- `ema144`: EMA144的值
- `ema150`: EMA150的值
- `ema156`: EMA156的值
- `ema565`: EMA565的值
- `ema600`: EMA600的值
- `offset_percent`: 容差百分比

**返回值**: `bool` - true表示价格接近任意一个EMA

**说明**: 只要价格接近任意一个EMA（在容差范围内），就返回true

**使用示例**:
```pine
if f_check_entry_price_short(close, ema144, ema150, ema156, ema565, ema600, 2.0)
    // 价格接近任意一个EMA，可以开空单
    strategy.entry("Short", strategy.short)
```

---

## 完整使用示例

```pine
//@version=6
strategy("Vegas Channel Strategy Example", overlay=true)

// ============================================================================
// 导入库函数（复制vegas_channel_lib.pine中的函数到这里）
// ============================================================================

// ... 这里粘贴所有库函数 ...

// ============================================================================
// 参数定义
// ============================================================================

ENTRY_QTY = input(0.1, "Entry Position Size")
OFFSET_PERCENT = input(2.0, "Price Offset (%)")
QUEUE_CAPACITY = input(48, "Array Capacity")
PRICE_ARRAY_THRESHOLD = input(8, "Price Array Threshold")

// ============================================================================
// 指标计算
// ============================================================================

// 计算EMA
[ema12, ema144, ema156, ema565, ema676] = f_calculate_emas(12, 144, 156, 565, 676)

// 判断趋势
downtrend = f_is_downtrend(ema12, ema144, ema156, ema565, ema676)
uptrend = f_is_uptrend(ema12, ema144, ema156, ema565, ema676)

// 计算波动率
atr_value = f_calculate_volatility(14)

// ============================================================================
// 状态变量
// ============================================================================

var float entry_price = na
var array<float> highprice = array.new<float>()
var array<float> lowprice = array.new<float>()

// ============================================================================
// 交易逻辑
// ============================================================================

// 开仓条件：多头趋势 + 价格接近EMA12
if uptrend and strategy.position_size == 0
    if f_check_price_near_ema(close, ema12, OFFSET_PERCENT)
        entry_price := close
        strategy.entry("Long", strategy.long, qty=ENTRY_QTY)
        array.clear(highprice)
        array.clear(lowprice)

// 持仓管理：记录价格并设置移动止损
if strategy.position_size > 0 and not na(entry_price)
    // 记录价格
    f_record_price(close, ema12, highprice, lowprice, QUEUE_CAPACITY)
    
    // 计算移动止损
    trailing_sl = f_calculate_trailing_sl(true, entry_price, highprice, lowprice, PRICE_ARRAY_THRESHOLD)
    strategy.exit("SL", "Long", stop=trailing_sl)

// 如果持仓被平掉，重置变量
if strategy.position_size == 0 and not na(entry_price)
    entry_price := na
    array.clear(highprice)
    array.clear(lowprice)

// ============================================================================
// 可视化
// ============================================================================

plot(ema12, "EMA12", color.blue)
plot(ema144, "EMA144", color.red)
plot(ema565, "EMA565", color.purple)
```

---

## 参数建议值

### EMA周期
- **EMA12**: 12（快速线）
- **EMA144**: 144（中期线）
- **EMA156**: 156（中期线）
- **EMA565**: 565（长期线）
- **EMA676**: 676（长期线）

### ATR参数
- **ATR周期**: 14（标准值）
- **ATR阈值**: 80-150（根据币种调整）

### 价格记录参数
- **数组容量**: 48（约2天的4H数据）
- **记录阈值**: 8（至少8根K线才开始计算止损）

### 开仓参数
- **价格容差**: 1.0-3.0%（通常使用2.0%）

---

## 注意事项

1. **数组管理**: 使用`var`关键字声明数组，确保数组在K线之间保持状态
2. **容量限制**: 数组容量不要设置过大，避免内存问题
3. **阈值设置**: 价格记录阈值应根据策略需求调整，太小可能过早触发止损
4. **容差范围**: 价格容差百分比应根据市场波动性调整
5. **趋势判断**: 趋势判断函数返回的是布尔值，可以用于条件判断

---

## 常见问题

### Q: 如何在不同时间框架使用这些函数？
A: 可以使用`request.security()`函数获取不同时间框架的EMA值，然后传入趋势判断函数。

### Q: 移动止损不生效怎么办？
A: 检查价格数组是否有足够的数据（需要超过threshold值），以及是否正确调用了`f_record_price`函数。

### Q: 如何自定义EMA周期？
A: 修改`f_calculate_emas`函数的参数，或者直接使用`ta.ema()`函数计算单个EMA。

---

## 更新日志

### v2.0
- 添加了`f_is_uptrend`函数
- 完善了函数文档
- 优化了代码注释

### v1.0
- 初始版本
- 包含基本的EMA计算、趋势判断、价格记录等功能

