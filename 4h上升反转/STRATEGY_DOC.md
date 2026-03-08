# 4H 上升反转策略文档（空单专用）

## 1. 策略概述

**策略名称**: 4H Uptrend Reversal Short Strategy (Short Only)
**时间周期**: 4小时
**策略类型**: 上升反转做空策略
**交易方向**: 仅做空
**交易理念**: 在空头趋势和多头趋势之间的转换点捕捉反转做空机会，利用EMA12与EMA144的交叉作为开仓信号
**默认仓位**: 账户权益的20%

## 2. 技术指标

### 2.1 EMA指标
策略使用3条EMA均线：
- **EMA12**: 12周期指数移动平均线（短期均线）
- **EMA144**: 144周期指数移动平均线（中期均线，关键）
- **EMA565**: 565周期指数移动平均线（长期均线）

### 2.2 趋势判断

**空头趋势信号**:
```
EMA12 < EMA144 < EMA565
```
三条均线按照空头顺序排列。

**多头趋势信号**:
```
EMA12 > EMA144 > EMA565
```
三条均线按照多头顺序排列。

## 3. 交易逻辑

### 3.1 三标记位状态管理

策略使用三个标记位来管理状态转换，确保信号的精确性：

#### 标记位1：`was_downtrend`（曾经是空头趋势）
- **初始状态**: `false`
- **触发条件**: 检测到空头趋势（`EMA12 < EMA144 < EMA565`）
- **动作**:
  - 设置 `was_downtrend = true`
  - 清空所有记录数据和计数器

#### 标记位2：`was_uptrend`（曾经是多头趋势）
- **初始状态**: `false`
- **触发条件**: 检测到多头趋势（`EMA12 > EMA144 > EMA565`）且 `was_downtrend = true`
- **动作**:
  - 设置 `was_uptrend = true`
  - 设置 `strategy_active = true`
  - 初始化记录变量

#### 标记位3：`strategy_active`（策略激活）
- **初始状态**: `false`
- **设置为true**: 从空头趋势转换为多头趋势时
- **设置为false**: 当策略激活状态下再次检测到空头趋势时
- **作用**: 控制是否记录EMA交叉事件和记录最高点

### 3.2 空单逻辑（上升反转做空）

**策略思路**:
- 在空头和多头趋势之间的转换时启动策略
- 从空头转为多头时，开始记录关键最高点
- 记录EMA12下穿和上穿事件
- 根据具体开单逻辑在特定位置下单

**关键最高点记录**:

#### high1（多头趋势中EMA12的最高点）：
- 记录条件：`strategy_active = true` 且 `uptrend_signal = true`
- 记录对象：EMA12的最高值
- 用途：作为空单的止损位置

#### price_high_uptrend（多头趋势中的现价最高点）：
- 记录条件：`strategy_active = true` 且 `uptrend_signal = true`
- 记录对象：K线的最高价（`high`）
- 用途：作为止损参考

#### price_high_after_cu（上穿后到下穿之间的现价最高点）：
- 记录条件：`recording_price_high = true` 且 `strategy_active = true`
- 触发开始：检测到EMA12上穿EMA144时
- 触发停止：检测到第一次下穿时
- 记录对象：K线的最高价（`high`）
- 用途：与high1的关系用于开仓价格计算

**上穿和下穿事件计数**:

#### crossover_count（EMA12上穿EMA144的次数）：
- 首次上穿：启动price_high_after_cu的记录

#### crossunder_count（EMA12下穿EMA144的次数）：
- 首次下穿（crossunder_count = 1）：停止price_high_after_cu的更新
- 第二次下穿（crossunder_count = 2）：触发开仓逻辑

## 4. 可视化标记

### 4.1 状态标记（底部和顶部）
- **蓝色小方块 "1"**（底部）：空头趋势（`was_downtrend = true`）
- **黄色小方块 "2"**（顶部）：多头趋势且策略激活（`strategy_active = true`）

### 4.2 交叉标记（仅在策略激活时显示）
- **绿色向上三角 + "上穿"**（下方）：EMA12上穿EMA144（`strategy_active = true`）
- **红色向下三角 + "下穿"**（上方）：EMA12下穿EMA144（`strategy_active = true`）

### 4.3 关键价位线
- **红色圆点线**：high1（EMA12在多头趋势中的最高点）
- **橙色圆点线**：price_high_uptrend（多头趋势中的现价最高点）
- **紫色圆点线**：price_high_after_cu（上穿后到下穿间的现价最高点）
- **蓝色虚线**：entry_price（开仓价格，待定义）
- **棕色虚线**：stop_loss_price（止损价格，待定义）

### 4.4 开仓平仓标记
- **红色圆圈**：空单入场
- **红色X标记**：空单平仓

### 4.5 开仓设置标记
- **蓝色菱形 + "开仓"**：第二次下穿后，开仓信号已设置

## 5. 状态转换流程

### 5.1 完整状态转换

```
初始状态（所有标记位为false）
        ↓
[1] 检测到空头趋势（EMA12 < EMA144 < EMA565）
    → was_downtrend = true
    → 清空: crossover_count, crossunder_count, high1等所有数据
    → 状态显示: 蓝色方块 "1"
        ↓
[2] 检测到多头趋势（EMA12 > EMA144 > EMA565）且 was_downtrend = true
    → was_uptrend = true
    → strategy_active = true
    → 初始化: high1 = ema12, price_high_uptrend = close
    → 开始记录事件
    → 状态显示: 黄色方块 "2"
        ↓
[3] 在多头趋势中持续记录最高点和交叉事件
    → 记录 high1（EMA12最高点）
    → 记录 price_high_uptrend（现价最高点）
    → 检测 crossover 和 crossunder 事件
    → 记录 price_high_after_cu（上穿后的现价最高点）
        ↓
[4] 当 crossunder_count = 2（第二次下穿）
    → 根据开单逻辑计算开仓价格
    → 设置开仓信号（待开发）
    → 派出开仓订单
        ↓
[5] 开仓订单成交
    → 进入持仓状态
    → 执行止盈止损规则
        ↓
[6] 检测到空头趋势且 strategy_active = true
    → was_downtrend = true
    → was_uptrend = false
    → strategy_active = false
    → 清空计数器
    → 回到 [1] 状态
```

### 5.2 关键转换条件

**从空头到多头的转换必须满足**:
- 当前趋势：空头（`EMA12 < EMA144 < EMA565`）
- 状态标记：`was_downtrend = true`
- 新趋势：多头（`EMA12 > EMA144 > EMA565`）
- 结果：确认是从空头真正转变为多头，而不是中间的混乱状态

**事件只在策略激活时记录**:
- `strategy_active = true` 是所有事件记录的必要条件
- 防止在多空转换的过渡期产生误信号

## 6. 策略参数总结

| 参数类型 | 参数名称 | 数值 | 说明 |
|---------|---------|------|------|
| EMA周期 | EMA12 | 12 | 短期均线 |
| EMA周期 | EMA144 | 144 | 中期均线（关键） |
| EMA周期 | EMA565 | 565 | 长期均线 |
| 空头判断 | EMA12 < EMA144 < EMA565 | - | 三线空头排列 |
| 多头判断 | EMA12 > EMA144 > EMA565 | - | 三线多头排列 |
| 下穿触发 | crossunder_count >= 2 | 第二次 | 开仓触发条件 |
| 空单仓位 | Short1, Short2 | 各0.5 | 分批派单（待定） |
| 空单止损 | high1 或其他 | 参考值 | 多头趋势中的参考点 |
| 仓位大小 | 默认 | 20% | 账户权益的20% |

## 7. 策略特点

1. **精确的状态转换**:
   - 使用三个标记位确保状态清晰
   - 只有从空头转向多头时才激活策略
   - 防止在多空转换中产生错误信号

2. **关键最高点记录**:
   - 记录EMA12在多头趋势中的最高点
   - 记录现价在多头趋势中的最高点
   - 记录上穿后的现价最高点
   - 三个最高点相互配合确保精确开仓

3. **对称的逻辑设计**:
   - 与下跌反转策略完全对称
   - 多头趋势中记录最高点，而非最低点
   - 等待第二次下穿触发，而非第二次上穿
   - 记录的是上升过程中的最高值

4. **灵活的开仓逻辑**:
   - 开仓逻辑待完成
   - 可根据实际需求设计不同的开仓方式
   - 支持限价单、市价单等多种开仓方式

5. **分批派单**:
   - 将空单分为两笔各0.5单位
   - 更灵活的风险管理
   - 可独立平仓管理

6. **上行风险保护**:
   - 止损设置在多头趋势的参考点
   - 可以有效防止大幅上升

## 8. 注意事项

1. **状态转换必须完整**:
   - 必须先进入空头趋势（was_downtrend = true）
   - 然后检测到多头趋势时才启动策略
   - 不会在一开始就是多头趋势时启动

2. **事件记录条件严格**:
   - 所有交叉事件和最高点记录都需要 `strategy_active = true`
   - 确保不会在多空转换的过渡期产生噪音信号

3. **开仓逻辑待完成**:
   - 当前仅完成了标记位和最高点记录逻辑
   - 需要自行添加具体的开仓条件和价格计算方法
   - 可参考下跌反转策略的开仓逻辑进行反向设计

4. **二次下穿触发**:
   - 需要等待EMA12第二次下穿EMA144才会触发开仓逻辑
   - 这样可以确保反转信号足够强烈

5. **止损设置**:
   - 止损应该设置在high1或根据开仓逻辑计算
   - 用于保护空单的上行风险

6. **仓位管理**:
   - 默认使用账户权益的20%作为仓位大小
   - 可在策略设置中调整

7. **回测注意**:
   - 在TradingView上测试时，确保图表时间框架为4小时
   - 确保有足够的历史数据（至少1年以上）
   - 注意点差和滑点的影响

## 9. 与下跌反转策略的对比

| 方面 | 下跌反转（多单） | 上升反转（空单） |
|------|-----------------|-----------------|
| 初始趋势 | 多头趋势 | 空头趋势 |
| 记录起点 | 多头 → 空头 | 空头 → 多头 |
| 标记位1 | was_uptrend | was_downtrend |
| 标记位2 | was_downtrend | was_uptrend |
| 记录对象 | 最低点（low） | 最高点（high） |
| 关键点1 | ema12_low | high1 |
| 关键点2 | price_low_downtrend | price_high_uptrend |
| 关键点3 | price_low_after_cu | price_high_after_cu |
| 触发事件 | 第二次上穿 | 第二次下穿 |
| 开仓方向 | 做多 | 做空 |
| 止损参考 | 最低点 | 最高点 |

## 10. 待完成的开仓逻辑

您需要在以下条件满足时添加开仓逻辑：

```
当以下条件都满足时：
- strategy_active = true（策略已激活）
- crossunder_count >= 2（已检测到第二次下穿）
- high1 > 0（已记录EMA12最高点）
- price_high_after_cu > 0（已记录上穿后的现价最高点）

开仓逻辑（待开发）：
1. 计算开仓价格（可参考下跌反转的反向逻辑）
2. 设置开仓订单类型（限价单/市价单）
3. 定义分批派单方式
4. 设置止损价格（参考high1或price_high_uptrend）
5. 设置止盈规则（参考下跌反转的反向逻辑）
```

## 11. 代码框架提示

建议的标记位变量定义：
```
var bool was_downtrend = false       // 曾经是空头趋势标记
var bool was_uptrend = false         // 曾经是多头趋势标记
var bool strategy_active = false     // 策略是否激活标记
var int crossover_count = 0          // EMA12上穿EMA144的次数
var int crossunder_count = 0         // EMA12下穿EMA144的次数
var float high1 = 0.0                // 多头趋势中EMA12的最高点
var float price_high_uptrend = 0.0  // 多头趋势中的现价最高点
var float price_high_after_cu = 0.0 // 上穿后到下穿之间的现价最高点
var float entry_price = 0.0          // 开仓价格
var bool short_entered = false       // 空单是否已入场
var bool recording_price_high = false // 是否正在记录上穿后的现价最高点
var float stop_loss_price = 0.0      // 止损价格
var bool order_placed = false        // 是否已下单
```

