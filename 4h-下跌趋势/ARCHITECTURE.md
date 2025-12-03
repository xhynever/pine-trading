# 项目架构文档

## 1. 项目结构

```
4h-下跌趋势/
├── strategy_enhanced.pine          # 主策略文件（已优化，内联库）
├── strategy.pine                   # 基础策略版本（参考）
├── strategy_v1.pine                # 历史版本v1
├── vegas_channel_lib.pine          # 库文件（已集成到main）
├── backtest_lib.pine               # 回测库（可选）
│
├── README.md                       # 项目说明书
├── IMPLEMENTATION_SUMMARY.md       # 实现总结
├── QUICKSTART.md                   # 快速开始指南
├── FLOWCHART.md                    # 流程图（新）
├── ARCHITECTURE.md                 # 本文件
└── VERSIONS.md                     # 版本历史
```

## 2. 核心模块说明

### 2.1 strategy_enhanced.pine - 主策略文件

**职责**: 完整的交易策略实现，包含所有交易逻辑

**结构**:
```
策略头部 (4H Vegas Channel Strategy Enhanced v2.0)
    ↓
[第一部分] 内联库函数 (第7-36行)
├─ f_calculate_emas()       - EMA计算
├─ f_is_downtrend()         - 趋势检测
└─ f_calculate_slope()      - 斜率计算

[第二部分] 参数定义 (第38-47行)
├─ SLOPE_THRESHOLD          - 斜率阈值
├─ SLOPE_PERIOD             - 斜率周期
├─ PRICE_ARRAY_THRESHOLD    - 数组触发阈值
├─ QUEUE_CAPACITY           - 数组容量
├─ SL_PERCENT               - 止损百分比
├─ TP1/2/3_PERCENT          - 止盈百分比
└─ OFFSET_PERCENT           - 价格偏移

[第三部分] 指标计算 (第49-65行)
├─ 日线EMA (参考用)
├─ 4H主EMA (5条)
├─ 额外EMA (EMA150, EMA600)
├─ 趋势判断
└─ 斜率计算

[第四部分] 状态变量 (第67-77行)
├─ duo1_marked, kong1_marked
├─ entry_price (多/空)
├─ highprice, lowprice 数组
└─ 其他计数和标志

[第五部分] 交易逻辑 (第79-214行)
├─ EMA12上穿计数 (第79-83行)
├─ Duo1多单逻辑 (第85-131行)
├─ Kong1空单逻辑 (第133-189行)
└─ 止盈止损执行 (第191-214行)

[第六部分] 可视化 (第216-227行)
├─ EMA绘制 (5条)
└─ 信号标记 (Duo1/Kong1)

[第七部分] 信息面板 (第229-273行)
└─ f_update_panel() - 实时显示策略状态
```

**关键数据流**:
```
新K线
  ↓
[计算所有指标]
  ↓
[更新计数器] (EMA12上穿)
  ↓
[信号标记] (Duo1/Kong1)
  ↓
[开仓检查]
  ↓
[持仓管理] (价格记录、止损、止盈)
  ↓
[更新面板显示]
```

### 2.2 vegas_channel_lib.pine - 库文件（参考，已内联）

**已集成的函数**:

| 函数名 | 参数 | 返回值 | 用途 |
|--------|------|--------|------|
| f_calculate_emas | len12, len144, len156, len565, len676 | array[5] | 批量计算5条EMA |
| f_is_downtrend | ema12, ema144, ema156, ema565, ema676 | bool | 判断空头趋势 |
| f_calculate_slope | source, period | float | 计算线性回归斜率 |

**内联位置**: strategy_enhanced.pine 第7-36行

## 3. 关键算法

### 3.1 趋势检测算法

```
空头趋势判断 (f_is_downtrend)
└─ 3层过滤器:
   ├─ 层1: EMA12 < min(EMA144, EMA156)
   │       └─ 快线低于两条中线 = 看空
   │
   ├─ 层2: min(EMA144, EMA156) < min(EMA565, EMA676)
   │       └─ 两条中线都低于两条慢线 = 强看空
   │
   └─ 结果: 三个条件均为TRUE = 空头趋势
```

**复杂度**: O(1) - 仅6次比较

### 3.2 斜率计算算法 (线性回归)

```
输入: 最近N根K线的EMA144值
输出: 斜率角度 (度数)

步骤:
1. 校验条件
   if bar_index < period: return 0.0

2. 初始化累加变量
   x_sum = 0, y_sum = 0, xy_sum = 0, x2_sum = 0

3. 循环计算求和
   for i=0 to period-1:
       x = i
       y = source[i]
       累加 x, y, xy, x²

4. 应用线性回归公式
   slope = (n·Σxy - Σx·Σy) / (n·Σx² - (Σx)²)

5. 转换为角度
   angle = arctan(slope) × 180 / π
```

**复杂度**: O(N) - N=28根K线

**为什么用斜率**:
- EMA144斜率 < 15° → 图形平缓，适合多头 (Duo1)
- EMA144斜率 > 15° → 图形陡峭，适合空头 (Kong1)

### 3.3 自适应移动止损算法

```
价格分类机制:
├─ 收盘 > EMA12 → highprice 数组
└─ 收盘 < EMA12 → lowprice 数组

触发条件: 总价格数 > 8根

多单移动止损:
├─ if highprice.size < lowprice.size
│   └─ 止损 = 入场价 (防守)
└─ else
    └─ 止损 = lowprice.average (动态)

空单移动止损:
├─ if highprice.size > lowprice.size
│   └─ 止损 = 入场价 (防守)
└─ else
    └─ 止损 = highprice.average (动态)
```

**设计原理**:
- 当反向价格较多 → 说明行情不利 → 用固定止损防守
- 当同向价格较多 → 说明行情顺利 → 用动态止损获利更多

### 3.4 分层止盈算法

```
多单分层止盈:
├─ 入场价 × 110% → 平仓 50%
├─ 入场价 × 115% → 平仓 25%
└─ 入场价 × 125% → 平仓 100%

空单分层止盈 (反向):
├─ 入场价 × 90% → 平仓 50%
├─ 入场价 × 85% → 平仓 25%
└─ 入场价 × 75% → 平仓 100%
```

**优点**: 锁定部分利润，同时保留继续获利的机会

## 4. 数据结构

### 4.1 状态变量

```
信号相关:
  duo1_marked         : bool     # Duo1是否标记
  kong1_marked        : bool     # Kong1是否标记
  ema12_cross_count   : int      # EMA12上穿计数

头寸相关:
  long_entry_price    : float    # 多单入场价
  short_entry_price   : float    # 空单入场价
  strategy.position_size : float # 当前持仓量

价格记录:
  highprice           : array    # 高于EMA12的价格
  lowprice            : array    # 低于EMA12的价格

时间追踪:
  bars_since_entry    : int      # 入场后经过的K线数
  prev_crossover/under: bool     # 前一根的交叉状态
```

### 4.2 数组管理 (循环队列)

```
highprice/lowprice 数组:
├─ 最大容量: QUEUE_CAPACITY (默认48)
├─ 当size > 容量时: 自动删除最早的数据
└─ 用途: 计算价格统计值

循环机制:
  新数据 → array.push()
  达到容量 → array.shift() 删除旧数据
  → 保持数据最新性和计算效率
```

## 5. 信号流程

### 5.1 信号标记阶段

```
每根K线检查:
1. 计算 downtrend = f_is_downtrend(...)
2. 计算 slope144 = f_calculate_slope(...)

Duo1标记:
├─ 检查: downtrend AND slope_low AND not duo1_marked
├─ 动作: 如果全部为真 → duo1_marked = true
└─ 持续: 直到满足开仓条件

Kong1标记:
├─ 检查: downtrend AND slope_high AND not kong1_marked
├─ 动作: 如果全部为真 → kong1_marked = true
└─ 取消: 如果 not downtrend → kong1_marked = false
```

### 5.2 开仓阶段

```
Duo1开仓检查 (每根K线):
1. duo1_marked == true?
2. close > ema144 (上穿)?
3. ema12_cross_count > 1?
4. strategy.position_size == 0?
5. close 在 ema12 ± 2% 范围?

全部满足 → 执行开多仓

Kong1开仓检查 (每根K线):
1. kong1_marked == true?
2. close 在 EMA144/150/156/565/600 ± 2%?
3. close < ema144 (下穿)?
4. strategy.position_size == 0?

全部满足 → 执行开空仓
```

### 5.3 持仓管理阶段

```
每根持仓K线:
1. 价格分类 → highprice/lowprice 数组
2. 数组大小管理 → 循环队列
3. 移动止损计算 → 基于数组统计
4. 止盈止损执行 → 根据收益率

退出条件:
├─ 止盈触发 (10%, 15%, 25%)
├─ 止损触发 (5% 或 移动止损)
└─ 强制平仓 (其他逻辑)
```

## 6. 参数设计

### 6.1 参数分类

**趋势参数**:
- SLOPE_THRESHOLD (15.0) - 区分多空
- SLOPE_PERIOD (28) - 斜率计算周期

**风险参数**:
- SL_PERCENT (5.0) - 基础止损
- TP1/2/3_PERCENT - 三层止盈

**数组参数**:
- PRICE_ARRAY_THRESHOLD (8) - 移动止损触发
- QUEUE_CAPACITY (48) - 数组最大长度

**价格参数**:
- OFFSET_PERCENT (2.0) - 开仓价格容差

### 6.2 参数调整指南

```
想要更多交易 → 降低 SLOPE_THRESHOLD
想要更少交易 → 提高 SLOPE_THRESHOLD

想要更激进 → 降低 SL_PERCENT, TP_PERCENT
想要更保守 → 提高 SL_PERCENT, 降低 TP_PERCENT

想要更灵敏 → 降低 PRICE_ARRAY_THRESHOLD
想要更稳定 → 提高 PRICE_ARRAY_THRESHOLD
```

## 7. 扩展点

### 7.1 代码扩展
```
添加新的开仓条件:
  → 在 if 语句中添加条件
  → 不影响现有逻辑

修改止盈止损:
  → 编辑 TP_PERCENT 或 SL_PERCENT 计算
  → 或修改 strategy.close/exit 调用

集成新的指标:
  → 添加新的计算函数
  → 在趋势检测中使用
```

### 7.2 参数扩展
```
添加新参数:
  1. 在第38-47行添加 input() 定义
  2. 在相应逻辑中使用该参数
  3. 注意参数命名规范

参数样板:
  PARAM_NAME = input(默认值, "显示名称")
```

## 8. 维护指南

### 8.1 代码修改清单

修改前检查:
- [ ] 理解现有逻辑
- [ ] 查看 FLOWCHART.md
- [ ] 考虑副作用

修改后检查:
- [ ] 代码编译通过
- [ ] 回测验证正常
- [ ] 更新相关文档

### 8.2 文档更新清单

若修改代码:
- [ ] 更新 FLOWCHART.md (如有流程变化)
- [ ] 更新 ARCHITECTURE.md (本文件)
- [ ] 更新 README.md (如有说明变化)
- [ ] 更新 VERSIONS.md (记录变更)

### 8.3 版本管理

```
版本格式: v主.次.修

v2.0 (当前)
  ├─ 内联库函数
  ├─ 完整的开仓逻辑
  └─ 自适应移动止损

v1.x (历史)
  └─ 导入库版本
```

## 9. 常见问题 FAQ

**Q1: 如何添加新的止盈点?**
```
A: 复制现有的 TP 逻辑:
   if close > long_entry_price * (1 + TP_NEW_PERCENT / 100)
       strategy.close("Long", comment="TP_NEW")
```

**Q2: 如何改变开仓数量?**
```
A: 修改 strategy.entry 中的 qty:
   strategy.entry("Long", strategy.long, qty=你的数量)
```

**Q3: 如何关闭Duo1或Kong1?**
```
A: 注释掉相应的 if 块:
   // if not duo1_marked and downtrend and slope_low
```

**Q4: 如何调整移动止损敏感度?**
```
A: 修改 PRICE_ARRAY_THRESHOLD (降低=更敏感)
   或修改 QUEUE_CAPACITY (降低=更快响应)
```

## 10. 性能指标

### 计算复杂度
```
每根K线:
  - EMA计算: O(1) (Pine Script缓存)
  - 趋势检测: O(1)
  - 斜率计算: O(SLOPE_PERIOD) = O(28)
  - 数组操作: O(1) 摊销

总计: O(N) ≈ O(1) 实际
```

### 内存使用
```
状态变量: ~500 字节
数组缓冲: QUEUE_CAPACITY × 2 × 8 字节 = ~768 字节 (48×2)

总计: ~1KB 内存占用
```

## 11. 测试清单

新版本发布前:
- [ ] 不同品种回测
- [ ] 不同时间段测试
- [ ] 极端市场条件测试
- [ ] 参数敏感度分析
- [ ] 绘图验证正确性
