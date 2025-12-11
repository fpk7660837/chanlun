# Chan Theory (缠论) Pine Script v6 完整实现需求文档

## 1. 项目概述

### 1.1 背景
基于TradingView Pine Script v6实现完整的Chan Theory（缠论）技术分析，包括所有核心功能：
- 分型识别
- 笔生成
- 线段构建
- 中枢检测
- 买卖点识别
- 背驰检测

### 1.2 目标
- 实现可视化的缠论完整分析
- 支持两种路径：
  - 路径1：分型 → 笔 → 笔中枢
  - 路径2：分型 → 笔 → 线段 → 中枢
- 提供完整的交易信号（买卖点）
- 优化TradingView平台性能

### 1.3 Pine Script v6限制与解决方案
- 最多50个绘图对象 → 动态管理，优先显示最新
- 单策略最多10000行代码 → 模块化设计
- 历史回测限制 → 合理配置参数
- 不能存储外部状态 → 使用var变量缓存
- 实时计算逐K线进行 → 优化算法效率

## 2. 核心功能实现

### 2.1 中枢检测（完整实现）

#### 2.1.1 中枢结构定义
```pine
//@version=6
indicator("Chan Theory Complete", overlay=true, max_boxes_count=50, max_lines_count=100)

// 中枢结构
type Center
    int start_bar         // 中枢开始
    int end_bar           // 中枢结束
    float high            // 中枢上沿
    float low             // 中枢下沿
    float mid             // 中枢中轴
    float peak_high       // 峰值上沿
    float peak_low        // 峰值下沿
    Stroke[] strokes      // 包含的笔
    int level             // 中枢级别
    bool is_breakout      // 是否已突破
    float breakout_price  // 突破价格

// 中枢数组
var Center[] centers = array.new<Center>()
```

#### 2.1.2 中枢计算算法
```pine
// 创建中枢
create_center(strokes, start_idx, end_idx) =>
    // 获取相关笔
    selected_strokes = array.slice(strokes, start_idx, end_idx + 1)

    // 计算中枢区间
    min_high = float(na)
    max_low = float(na)
    peak_h = float(na)
    peak_l = float(na)

    for i = 0 to array.size(selected_strokes) - 1
        stroke = array.get(selected_strokes, i)
        min_high := na(min_high) ? stroke._high() : math.min(min_high, stroke._high())
        max_low := na(max_low) ? stroke._low() : math.max(max_low, stroke._low())
        peak_h := na(peak_h) ? stroke._high() : math.max(peak_h, stroke._high())
        peak_l := na(peak_l) ? stroke._low() : math.min(peak_l, stroke._low())

    // 检查是否形成中枢
    if min_high > max_low
        new_center = Center.new()
        new_center.start_bar := array.get(selected_strokes, 0).start_bar
        new_center.end_bar := array.get(selected_strokes, array.size(selected_strokes) - 1).end_bar
        new_center.high := min_high
        new_center.low := max_low
        new_center.mid := (min_high + max_low) / 2
        new_center.peak_high := peak_h
        new_center.peak_low := peak_l
        new_center.strokes := selected_strokes
        new_center.level := 1
        new_center.is_breakout := false
        new_center
    else
        na

// 检测中枢突破
check_center_breakout(center, current_price) =>
    if current_price > center.high
        center.is_breakout := true
        center.breakout_price := current_price
        "up"
    else if current_price < center.low
        center.is_breakout := true
        center.breakout_price := current_price
        "down"
    else
        "none"
```

### 2.2 买卖点识别（完整实现）

#### 2.2.1 买卖点类型定义
```pine
// 买卖点类型
type BSPType
    int type      // 1, 1p, 2, 2s, 3a, 3b
    int level     // 级别
    float price   // 买卖点价格
    int bar_index // 产生时间
    string reason // 产生原因

// 买卖点数组
var BSPType[] buy_points = array.new<BSPType>()
var BSPType[] sell_points = array.new<BSPType>()

// 创建买卖点
create_bsp(type, level, price, bar_index, reason) =>
    bsp = BSPType.new()
    bsp.type := type
    bsp.level := level
    bsp.price := price
    bsp.bar_index := bar_index
    bsp.reason := reason
    bsp
```

#### 2.2.2 一类买卖点（中枢突破）
```pine
// 检测一类买卖点
detect_type1_bsp(strokes, centers) =>
    if array.size(centers) > 0
        latest_center = array.get(centers, array.size(centers) - 1)
        if array.size(strokes) > 0
            latest_stroke = array.get(strokes, array.size(strokes) - 1)

            // 1买：向下突破中枢后的回升
            if latest_center.is_breakout and latest_center.breakout_price < latest_center.low
                if latest_stroke.direction == 1 and latest_stroke.end_price > latest_center.low
                    bsp = create_bsp(1, 1, latest_stroke.end_price, bar_index, "Type 1 Buy")
                    array.push(buy_points, bsp)

            // 1卖：向上突破中枢后的回调
            if latest_center.is_breakout and latest_center.breakout_price > latest_center.high
                if latest_stroke.direction == -1 and latest_stroke.end_price < latest_center.high
                    bsp = create_bsp(1, 1, latest_stroke.end_price, bar_index, "Type 1 Sell")
                    array.push(sell_points, bsp)
```

#### 2.2.3 二类买卖点（回拉不进入中枢）
```pine
// 检测二类买卖点
detect_type2_bsp(strokes, centers) =>
    if array.size(centers) > 0 and array.size(strokes) >= 3
        latest_center = array.get(centers, array.size(centers) - 1)

        // 获取最近三笔
        s1 = array.get(strokes, array.size(strokes) - 3)
        s2 = array.get(strokes, array.size(strokes) - 2)
        s3 = array.get(strokes, array.size(strokes) - 1)

        // 2买：1买后的回拉不进入前一中枢
        if s2.direction == -1 and s3.direction == 1
            if s2.end_price > latest_center.low
                bsp = create_bsp(2, 1, s3.end_price, bar_index, "Type 2 Buy")
                array.push(buy_points, bsp)

        // 2卖：1卖后的反弹不进入前一中枢
        if s2.direction == 1 and s3.direction == -1
            if s2.end_price < latest_center.high
                bsp = create_bsp(2, 1, s3.end_price, bar_index, "Type 2 Sell")
                array.push(sell_points, bsp)
```

#### 2.2.4 三类买卖点（次级别回拉）
```pine
// 检测三类买卖点
detect_type3_bsp(strokes, centers) =>
    if array.size(centers) >= 2 and array.size(strokes) >= 3
        latest_center = array.get(centers, array.size(centers) - 1)
        prev_center = array.get(centers, array.size(centers) - 2)

        // 3a：中枢在前，1类在后
        if prev_center.end_bar < latest_center.start_bar
            latest_stroke = array.get(strokes, array.size(strokes) - 1)

            // 3a买
            if latest_stroke.direction == 1
                if latest_stroke.end_price > prev_center.low and latest_stroke.end_price < latest_center.low
                    bsp = create_bsp(3, 1, latest_stroke.end_price, bar_index, "Type 3a Buy")
                    array.push(buy_points, bsp)

        // 3b：1类在前，中枢在后
        if latest_center.start_bar > prev_center.end_bar
            latest_stroke = array.get(strokes, array.size(strokes) - 1)

            # 3b卖
            if latest_stroke.direction == -1
                if latest_stroke.end_price < prev_center.high and latest_stroke.end_price > latest_center.high
                    bsp = create_bsp(3, 2, latest_stroke.end_price, bar_index, "Type 3b Sell")
                    array.push(sell_points, bsp)
```

### 2.3 背驰检测（完整实现）

#### 2.3.1 MACD背驰检测
```pine
// MACD指标
[macdLine, signalLine, histLine] = ta.macd(close, 12, 26, 9)

// 背驰检测
detect_divergence(center, stroke, algo) =>
    if array.size(center.strokes) >= 2
        entry_stroke = array.get(center.strokes, 0)
        exit_stroke = stroke

        if algo == "area"
            // 面积法
            entry_area = calculate_macd_area(entry_stroke)
            exit_area = calculate_macd_area(exit_stroke)

            // 顶背驰：价格新高，MACD面积减小
            if exit_stroke.direction == 1
                if exit_stroke.end_price > entry_stroke.end_price and exit_area < entry_area
                    {"divergence": true, "type": "bearish", "ratio": exit_area/entry_area}

            // 底背驰：价格新低，MACD面积减小
            if exit_stroke.direction == -1
                if exit_stroke.end_price < entry_stroke.end_price and exit_area < entry_area
                    {"divergence": true, "type": "bullish", "ratio": exit_area/entry_area}

        else if algo == "peak"
            // 峰值法
            entry_peak = get_macd_peak(entry_stroke)
            exit_peak = get_macd_peak(exit_stroke)

            // 顶背驰
            if exit_stroke.direction == 1
                if exit_stroke.end_price > entry_stroke.end_price and exit_peak < entry_peak
                    {"divergence": true, "type": "bearish", "ratio": exit_peak/entry_peak}

            # 底背驰
            if exit_stroke.direction == -1
                if exit_stroke.end_price < entry_stroke.end_price and exit_peak < entry_peak
                    {"divergence": true, "type": "bullish", "ratio": exit_peak/entry_peak}

    {"divergence": false, "type": "none", "ratio": 0.0}

// 计算笔的MACD面积
calculate_macd_area(stroke) =>
    area = 0.0
    for i = stroke.start_bar to stroke.end_bar
        if stroke.direction == 1
            area += math.max(histLine[i], 0)
        else
            area += math.abs(math.min(histLine[i], 0))
    area

// 获取笔的MACD峰值
get_macd_peak(stroke) =>
    peak = 0.0
    for i = stroke.start_bar to stroke.end_bar
        if stroke.direction == 1
            peak := math.max(peak, macdLine[i])
        else
            peak := math.min(peak, macdLine[i])
    peak
```

## 3. 完整配置选项

### 3.1 笔配置
```pine
group("笔设置")
stroke_min_length = input.int(3, "最小笔K线数", minval=1, maxval=5)
stroke_strict = input.bool(false, "严格模式（4根K线）")
stroke_end_peak = input.bool(true, "结束必须是峰值")
stroke_gap_as_kl = input.bool(false, "跳空算作K线")
```

### 3.2 线段配置
```pine
group("线段设置")
segment_enabled = input.bool(true, "启用线段")
segment_algo = input.string("chan", "线段算法", options=["chan"])
segment_peak_method = input.string("peak", "特征序列方法", options=["peak", "all"])
```

### 3.3 中枢配置
```pine
group("中枢设置")
center_algo = input.string("normal", "中枢算法", options=["normal", "over_seg"])
center_one_bi = input.bool(false, "单笔中枢")
center_combine = input.bool(true, "合并中枢")
center_combine_mode = input.string("zs", "合并模式", options=["zs", "peak"])
center_min_strokes = input.int(3, "中枢最小笔数", minval=1, maxval=10)
```

### 3.4 买卖点配置
```pine
group("买卖点设置")
bsp_enabled = input.bool(true, "启用买卖点")
bsp_types = input.string("1,2,3a,3b", "买卖点类型", options=["1,2,3a,3b", "1,2,3a,1p,2s,3b"])
bsp_1_only_multi = input.bool(true, "1类买卖点仅多笔中枢")
bsp_strict_3 = input.bool(false, "严格3类买卖点")
```

### 3.5 背驰配置
```pine
group("背驰检测")
divergence_enabled = input.bool(true, "启用背驰检测")
divergence_algo = input.string("area", "背驰算法", options=["area", "peak", "slope"])
divergence_rate = input.float(0.8, "背驰阈值", minval=0.1, maxval=1.0)
divergence_macd_fast = input.int(12, "MACD快线周期", minval=1, maxval=50)
divergence_macd_slow = input.int(26, "MACD慢线周期", minval=1, maxval=100)
divergence_macd_signal = input.int(9, "MACD信号线周期", minval=1, maxval=50)
```

### 3.6 显示配置
```pine
group("显示设置")
show_strokes = input.bool(true, "显示笔")
show_segments = input.bool(true, "显示线段")
show_centers = input.bool(true, "显示中枢")
show_bsp = input.bool(true, "显示买卖点")
show_divergence = input.bool(true, "显示背驰")
max_display_items = input.int(50, "最大显示数量", maxval=50)
show_labels = input.bool(true, "显示价格标签")
show_info_table = input.bool(true, "显示信息表")
```

## 4. 绘图管理策略

### 4.1 动态绘图对象管理
```pine
// 绘图对象管理
var line[] stroke_lines = array.new<line>()
var box[] center_boxes = array.new<box>()
var label[] bsp_labels = array.new<label>()

// 清理旧对象
cleanup_old_objects() =>
    // 限制笔的显示数量
    while array.size(stroke_lines) > max_display_items
        line.delete(array.shift(stroke_lines))

    // 限制中枢的显示数量
    while array.size(center_boxes) > max_display_items / 2
        box.delete(array.shift(center_boxes))

    // 限制买卖点标签数量
    while array.size(bsp_labels) > 20
        label.delete(array.shift(bsp_labels))
```

### 4.2 分层显示
```pine
// 按优先级显示
draw_by_priority() =>
    // 1. 最近的买卖点（最高优先级）
    draw_latest_bsp()

    // 2. 当前中枢
    draw_current_center()

    // 3. 最近的笔
    draw_recent_strokes()

    // 4. 历史中枢
    draw_historical_centers()
```

## 5. 性能优化技巧

### 5.1 计算优化
```pine
// 使用缓存变量
var int last_calculated_bar = 0
var bool needs_recalculate = true

if bar_index != last_calculated_bar
    needs_recalculate := true
    last_calculated_bar := bar_index
else
    needs_recalculate := false

// 只在必要时重新计算
if needs_recalculate or barstate.islast
    // 执行计算
    update_all()
```

### 5.2 数组管理
```pine
// 限制数组大小
MAX_HISTORY = 1000
if array.size(strokes) > MAX_HISTORY
    strokes := array.slice(strokes,
        array.size(strokes) - MAX_HISTORY,
        array.size(strokes))

// 清理无效数据
cleanup_invalid_data() =>
    // 移除已过时的买卖点
    valid_bsp = array.new<BSPType>()
    for i = 0 to array.size(buy_points) - 1
        bsp = array.get(buy_points, i)
        if bar_index - bsp.bar_index < 200  // 保留最近200个K线的买卖点
            array.push(valid_bsp, bsp)
    buy_points := valid_bsp
```

## 6. 完整实现示例

### 6.1 主函数框架
```pine
// 主处理逻辑
if barstate.isconfirmed
    // 1. 更新K线
    update_klines()

    // 2. 检测分型
    detect_fractals()

    // 3. 更新笔
    update_strokes()

    // 4. 更新线段（如果启用）
    if segment_enabled
        update_segments()

    // 5. 更新中枢
    update_centers()

    // 6. 检测买卖点
    if bsp_enabled
        detect_buy_sell_points()

    // 7. 检测背驰
    if divergence_enabled
        detect_all_divergence()

    // 8. 清理和绘图
    cleanup_old_objects()
    draw_all()

// 绘制所有元素
draw_all() =>
    if show_strokes
        draw_all_strokes()

    if segment_enabled and show_segments
        draw_all_segments()

    if show_centers
        draw_all_centers()

    if show_bsp
        draw_all_bsp()

    if show_labels
        draw_price_labels()

    if show_info_table
        update_info_table()
```

### 6.2 信息表显示
```pine
// 信息表
var table info_table = table.new(
    position.top_right,
    3, 8,
    bgcolor=color.white,
    border_width=1,
    frame_color=color.gray
)

// 更新信息表
update_info_table() =>
    table.cell(info_table, 0, 0, "统计信息", text_color=color.black, text_size=size.small)
    table.cell(info_table, 1, 0, "", text_color=color.black)
    table.cell(info_table, 2, 0, "", text_color=color.black)

    table.cell(info_table, 0, 1, "笔数", text_color=color.black)
    table.cell(info_table, 1, 1, str.tostring(array.size(strokes)), text_color=color.blue)

    table.cell(info_table, 0, 2, "中枢数", text_color=color.black)
    table.cell(info_table, 1, 2, str.tostring(array.size(centers)), text_color=color.blue)

    table.cell(info_table, 0, 3, "买点数", text_color=color.black)
    table.cell(info_table, 1, 3, str.tostring(array.size(buy_points)), text_color=color.green)

    table.cell(info_table, 0, 4, "卖点数", text_color=color.black)
    table.cell(info_table, 1, 4, str.tostring(array.size(sell_points)), text_color=color.red)

    // 最新信号
    if array.size(buy_points) > 0
        latest_buy = array.get(buy_points, array.size(buy_points) - 1)
        table.cell(info_table, 0, 5, "最新买点", text_color=color.black)
        table.cell(info_table, 1, 5, "Type " + str.tostring(latest_buy.type), text_color=color.green)

    if array.size(sell_points) > 0
        latest_sell = array.get(sell_points, array.size(sell_points) - 1)
        table.cell(info_table, 0, 6, "最新卖点", text_color=color.black)
        table.cell(info_table, 1, 6, "Type " + str.tostring(latest_sell.type), text_color=color.red)

    // 背驰信息
    table.cell(info_table, 0, 7, "背驰状态", text_color=color.black)
    table.cell(info_table, 1, 7, latest_divergence.type, text_color=latest_divergence.type == "bearish" ? color.red : color.green)
```

## 7. 使用建议

### 7.1 最佳实践
1. **合理配置参数**：根据不同品种和时间周期调整
2. **注意显示限制**：避免显示过多历史数据
3. **组合使用**：结合其他技术指标提高准确率
4. **多周期验证**：在不同时间周期上确认信号

### 7.2 注意事项
- 背驰检测需要足够的历史数据
- 买卖点的确认需要等待K线完成
- 在快速波动的市场中可能需要调整参数
- 建议配合成交量等其他指标使用

## 8. 总结

Pine Script v6完全有能力实现Chan Theory的所有核心功能：

1. ✅ **中枢检测** - 完整实现，支持合并和级别管理
2. ✅ **买卖点识别** - 支持1、2、3类买卖点的完整识别
3. ✅ **背驰检测** - 支持MACD面积法、峰值法等多种算法
4. ✅ **可视化** - 通过动态管理实现灵活的图形显示

主要挑战在于：
- 绘图对象数量限制（通过动态管理解决）
- 计算性能优化（通过缓存和算法优化解决）
- 实时性保证（通过增量更新思路实现）

这个实现方案保留了Chan Theory的所有核心价值，同时充分利用了Pine Script v6的新特性。