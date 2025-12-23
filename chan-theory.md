缠论系统核心逻辑详细说明

一、基础K线处理层

1.1 K线合并（包含关系处理）
目标： 将原始K线（KLine_Unit）合并为处理后的K线（KLine）

规则：
- 向上趋势（前一根K线方向为UP）：
  - 当前K线高点 = max(当前高, 前一根高)
  - 当前K线低点 = max(当前低, 前一根低)
- 向下趋势（前一根K线方向为DOWN）：
  - 当前K线高点 = min(当前高, 前一根高)
  - 当前K线低点 = min(当前低, 前一根低)
- 如果不满足包含关系，则形成新的独立K线
  
1.2 分型识别（FX）
定义： 识别顶分型和底分型

顶分型条件：
- 中间K线的高点 >= 左右两边K线的高点
- 中间K线的低点 >= 左右两边K线的低点

底分型条件：
- 中间K线的高点 <= 左右两边K线的高点
- 中间K线的低点 <= 左右两边K线的低点

1.3 分型识别详细规则

分型的基本结构：
- 分型由连续的3根处理后的K线（KLine，已处理包含关系）组成
- 结构：[左K线, 中间K线, 右K线]
- 中间K线称为分型的"核心K线"

分型的标注位置：
- 分型标注在中间K线上
- 顶分型：标记中间K线的高点
- 底分型：标记中间K线的低点

分型的更新规则：
1. 连续顶分型处理：
   - 如果出现连续的顶分型（多个满足顶分型条件的三K组合）
   - 只保留最后一个（最右边）的顶分型
   - 等价于：取这些连续顶分型中，高点最高的那个

2. 连续底分型处理：
   - 如果出现连续的底分型
   - 只保留最后一个（最右边）的底分型
   - 等价于：取这些连续底分型中，低点最低的那个

分型的确认：
- 分型需要在右侧K线出现后才能确认
- 最新的分型可能是"虚分型"，随着新K线出现可能被更新或取消

边界情况：
- 前两根K线无法形成分型（需要等第3根K线）
- 最后一根K线无法作为分型的右侧K线（分型需要完整的3根K线结构）

二、笔（Bi）的识别逻辑

2.1 笔的基本定义
笔的构成： 从一个分型到另一个相反的分型

默认配置（bi_conf）：
bi_algo = "normal"           # 标准笔算法
is_strict = True             # 严格模式
bi_fx_check = "strict"       # 严格的分型校验
gap_as_kl = False           # gap不算作一根K线
bi_end_is_peak = True       # 笔的终点必须是极值点
bi_allow_sub_peak = True    # 允许次高点

2.2 笔的成立条件

跨度要求（is_strict=True）：
- 从起始分型到结束分型之间至少需要4根合并后的K线（KLine）
- 说明：这4根K线不包括构成分型的K线本身
- 计算方式：分型1的右K线 到 分型2的左K线 之间的K线数量 >= 4
- 示例：分型1在第3根K线，分型2在第8根K线，则中间有4根K线（第4,5,6,7根）

具体计算逻辑：
```
假设：
- 顶分型位于索引i处（核心K线）
- 底分型位于索引j处（核心K线）
- 则笔成立需要：j - i >= 5
  （因为分型需要左中右三根，所以：i+1是顶分型右侧，j-1是底分型左侧，(j-1)-(i+1)+1 = j-i-1 >= 4，即j-i >= 5）
```

分型有效性检查（bi_fx_check="strict"）：
目标：确保两个分型之间有明确的高低分离，避免"假笔"

对于顶分型 -> 底分型（向下笔）：
- 条件1：顶分型的最低点 > 底分型的最高点
  * 顶分型最低点 = min(左K线低点, 中K线低点, 右K线低点)
  * 底分型最高点 = max(左K线高点, 中K线高点, 右K线高点)
  * 这确保了顶分型整体在底分型上方，有明确的分离
- 条件2：顶分型的高点 > 底分型的高点
  * 额外的方向性检查

对于底分型 -> 顶分型（向上笔）：
- 条件1：底分型的最高点 < 顶分型的最低点
  * 底分型最高点 = max(左K线高点, 中K线高点, 右K线高点)
  * 顶分型最低点 = min(左K线低点, 中K线低点, 右K线低点)
  * 这确保了底分型整体在顶分型下方，有明确的分离
- 条件2：底分型的低点 < 顶分型的低点
  * 额外的方向性检查

图示说明：
```
向下笔的严格检查：
     顶分型区域
    /  \
   /____\  <- 顶分型最低点

  （必须有分离）
   ______  <- 底分型最高点
  /      \
 /  底分型 \
```

极值点要求（bi_end_is_peak=True）：
- 从起始分型到结束分型之间，结束分型必须是这段区间的极值点
- 向上笔：结束顶分型的高点 >= 这段区间内所有K线的高点
- 向下笔：结束底分型的低点 <= 这段区间内所有K线的低点
- 检查范围：从起始分型的核心K线 到 结束分型的核心K线
- 目的：确保笔的终点是真正的拐点，不是中途的次级波动
  
2.3 笔的更新机制

确定笔（is_sure=True）：
- 已经完整成立的笔
  
虚笔（is_sure=False）：
- 为了实时计算而添加的未确认的笔
- 当新的K线出现时，如果满足条件会转为确定笔
  
笔的延伸：
- 如果新的分型与当前笔末端分型类型相同，且极值更优，则更新笔的终点

2.4 MACD计算与背驰判断

MACD指标计算：
- MACD是Moving Average Convergence Divergence（指数平滑异同移动平均线）
- 用于判断笔的力度，是背驰判断的核心指标

默认参数：
- 快线周期（fast）：12
- 慢线周期（slow）：26
- 信号线周期（signal）：9

计算公式：
```
1. EMA_fast = EMA(close, 12)    # 12周期指数移动平均
2. EMA_slow = EMA(close, 26)    # 26周期指数移动平均
3. DIF = EMA_fast - EMA_slow    # 差离值（快慢线之差）
4. DEA = EMA(DIF, 9)            # 9周期DIF的指数移动平均（信号线）
5. MACD = (DIF - DEA) * 2       # MACD柱状图（红绿柱）
```

注意：
- MACD柱 > 0 为红柱（多头力量）
- MACD柱 < 0 为绿柱（空头力量）
- 每根K线单元（KLine_Unit）都需要计算并存储MACD值

笔的MACD指标提取（macd_algo="peak"）：
目标：量化笔的力度，用于背驰判断

方法1：峰值法（peak，默认推荐）
- 遍历笔内所有K线单元（从begin_klc到end_klc）
- 向上笔：取MACD > 0 的最大值（红柱最高峰）
- 向下笔：取|MACD < 0|的最大值（绿柱最低谷的绝对值）
- 返回：abs(峰值)

方法2：面积法（area）
- 计算笔内所有MACD柱的面积和
- 向上笔：sum(MACD) for MACD > 0
- 向下笔：sum(abs(MACD)) for MACD < 0
- 返回：abs(总面积)

方法3：平均法（avg）
- 计算笔内MACD的平均值
- 返回：abs(mean(MACD_list))

背驰判断原理：
```python
# 以一类买卖点为例
in_metric = zs.bi_in.cal_macd_metric("peak")    # 进入中枢笔的力度
out_metric = seg.end_bi.cal_macd_metric("peak")  # 离开中枢笔的力度
divergence_rate = out_metric / in_metric         # 背驰比例

# 判断：如果离开笔的力度 < 进入笔的力度，则发生背驰
if divergence_rate < 1.0:  # 实际可配置阈值
    # 发生背驰，形成一类买卖点
    pass
```

is_reverse参数说明：
- is_reverse=False：正常计算（从笔的起点到终点）
- is_reverse=True：反向计算（只考虑笔的后半段或反向查找峰值）
- 用于更精确地判断笔末端的力度衰竭

二之一、线段（Seg）的识别逻辑

重要性说明：
- 线段是笔的更高层级结构
- 中枢必须在线段内计算
- 买卖点判断依赖线段位置
- 特征序列是相对线段方向提取的

2A.1 线段的基本定义

默认配置（seg_conf）：
seg_algo = "chan"              # 标准缠论线段算法
left_method = "all"            # 左侧笔选择方法

线段的构成：
- 线段由连续的笔组成
- 线段有明确的方向（向上或向下）
- 线段的起点和终点都是笔的端点

2A.2 线段识别算法（seg_algo="chan"）

核心规则：特征序列的破坏

什么是特征序列：
- 在一组同方向的笔中，每相邻两笔之间的间隔笔称为"特征序列元素"
- 示例：笔1(↑) - 笔2(↓) - 笔3(↑) - 笔4(↓) - 笔5(↑)
  * 同向笔：笔1、笔3、笔5（都向上）
  * 特征序列：笔2、笔4（间隔的向下笔）

线段的形成条件：
1. 起始：从第一笔开始，或从上一个线段结束后的下一笔开始
2. 方向：线段的方向由起始笔的方向决定
3. 延伸：只要特征序列不被破坏，线段就持续延伸
4. 终止：当特征序列被破坏时，线段结束

特征序列破坏的判断（核心算法）：

定义：
- 设线段方向为D（UP或DOWN）
- 提取线段内所有方向为D的笔，记为[bi_0, bi_2, bi_4, ...]（同向笔）
- 提取它们之间的反向笔，记为[bi_1, bi_3, bi_5, ...]（特征序列）

破坏条件（以向上线段为例）：
```
向上线段的特征序列是向下笔序列[bi_1, bi_3, bi_5, ...]

对于最新的特征序列笔bi_n（向下笔）：
1. 找到前面的所有特征序列笔
2. 检查是否存在某个特征序列笔bi_k，满足：
   - bi_n的低点 < bi_k的低点  （创新低）
   - 且bi_n的高点 < bi_k的低点 （完全破坏，不仅仅是底分型重叠）

如果满足，则特征序列被破坏，线段在bi_n的前一笔结束
```

向下线段的破坏条件（相反）：
```
向下线段的特征序列是向上笔序列

对于最新的特征序列笔bi_n（向上笔）：
检查是否存在某个前面的特征序列笔bi_k，满足：
   - bi_n的高点 > bi_k的高点  （创新高）
   - 且bi_n的低点 > bi_k的高点 （完全破坏）

如果满足，则特征序列被破坏，线段在bi_n的前一笔结束
```

2A.3 线段识别的详细步骤

算法伪代码：
```python
def identify_segments(bi_list):
    seg_list = []
    current_seg_start_idx = 0

    while current_seg_start_idx < len(bi_list):
        # 1. 确定线段起始笔
        start_bi = bi_list[current_seg_start_idx]
        seg_direction = start_bi.dir

        # 2. 提取同向笔和特征序列
        same_dir_bi_list = [start_bi]  # 同向笔列表
        feature_bi_list = []             # 特征序列笔列表

        # 3. 逐笔扫描，构建特征序列
        for i in range(current_seg_start_idx + 1, len(bi_list)):
            bi = bi_list[i]

            if bi.dir == seg_direction:
                # 同向笔
                same_dir_bi_list.append(bi)
                if i > current_seg_start_idx + 1:
                    # 添加前一笔到特征序列
                    feature_bi_list.append(bi_list[i-1])

            # 4. 检查特征序列是否被破坏
            if len(feature_bi_list) >= 1:
                latest_feature_bi = feature_bi_list[-1]

                if is_feature_sequence_broken(
                    seg_direction,
                    feature_bi_list,
                    latest_feature_bi
                ):
                    # 特征序列被破坏，线段结束
                    seg_end_idx = i - 2  # 倒数第二笔
                    seg = Segment(
                        start_bi=bi_list[current_seg_start_idx],
                        end_bi=bi_list[seg_end_idx],
                        direction=seg_direction
                    )
                    seg_list.append(seg)

                    # 下一个线段从破坏点开始
                    current_seg_start_idx = seg_end_idx + 1
                    break
        else:
            # 没有破坏，线段延续到最后
            seg = Segment(
                start_bi=bi_list[current_seg_start_idx],
                end_bi=bi_list[-1],
                direction=seg_direction,
                is_sure=False  # 最后一个线段是虚线段
            )
            seg_list.append(seg)
            break

    return seg_list

def is_feature_sequence_broken(seg_direction, feature_bi_list, latest_bi):
    """检查特征序列是否被破坏"""
    if len(feature_bi_list) < 2:
        return False

    # 检查前面的特征序列笔（不包括最新的）
    for old_feature_bi in feature_bi_list[:-1]:
        if seg_direction == Direction.UP:
            # 向上线段，特征序列是向下笔
            # 破坏条件：新笔创新低且完全脱离旧笔
            if (latest_bi._low() < old_feature_bi._low() and
                latest_bi._high() < old_feature_bi._low()):
                return True
        else:
            # 向下线段，特征序列是向上笔
            # 破坏条件：新笔创新高且完全脱离旧笔
            if (latest_bi._high() > old_feature_bi._high() and
                latest_bi._low() > old_feature_bi._high()):
                return True

    return False
```

2A.4 线段的关键属性

线段结构（CSeg）：
```python
class CSeg:
    idx: int                    # 线段索引
    start_bi: CBi              # 起始笔
    end_bi: CBi                # 结束笔
    dir: SEG_DIR               # 方向（UP/DOWN）
    is_sure: bool              # 是否确定（最后一个线段为虚线段）

    # 中枢相关
    zs_lst: List[CZS]          # 线段内的中枢列表

    # 方法
    get_begin_val()            # 起始值
    get_end_val()              # 结束值
    amp()                      # 振幅
    get_bi_list()              # 获取线段内所有笔
```

线段的确认：
- 确定线段（is_sure=True）：已经完整形成，有明确的终点
- 虚线段（is_sure=False）：最后一个线段，可能随新笔出现而延伸或改变

2A.5 线段识别的实例说明

示例1：向上线段的形成与破坏
```
笔序列：
笔1(↑): [100, 110]  - 向上笔，起始线段
笔2(↓): [110, 105]  - 向下笔，特征序列第1个
笔3(↑): [105, 115]  - 向上笔，线段延续
笔4(↓): [115, 108]  - 向下笔，特征序列第2个
笔5(↑): [108, 120]  - 向上笔，线段延续
笔6(↓): [120, 100]  - 向下笔，特征序列第3个

分析：
- 线段方向：向上（由笔1决定）
- 特征序列：笔2(低点105), 笔4(低点108), 笔6(低点100)
- 笔6的低点100 < 笔2的低点105 （创新低）
- 笔6的高点120 > 笔2的低点105 （未完全破坏）
- 继续检查笔4：笔6低点100 < 笔4低点108，笔6高点120 > 笔4低点108
- 特征序列未完全破坏，线段继续

如果笔6变为：笔6(↓): [120, 95]，高点120 > 笔2低点105但接近
需要笔6(↓): [120, 90]，高点120 >> 笔4低点108，不满足完全脱离
继续...

完全破坏示例：
笔7(↓): [120, 104] - 向下笔
- 笔7低点104 < 笔2低点105（创新低）
- 笔7高点104 < 笔2低点105（完全破坏！）
则线段在笔5结束，新线段从笔6开始
```

2A.6 特征序列的提取（用于中枢计算）

在线段确定后，需要提取特征序列用于中枢识别：

```python
def extract_feature_sequence(seg, bi_list):
    """提取线段内的特征序列笔"""
    feature_bi_list = []

    # 遍历线段内的所有笔
    for bi in bi_list[seg.start_bi.idx : seg.end_bi.idx + 1]:
        # 与线段方向相反的笔是特征序列
        if bi.dir != seg.dir:
            bi.seg_idx = seg.idx  # 标记笔所属线段
            feature_bi_list.append(bi)

    return feature_bi_list
```

特征序列的作用：
- 中枢必须由至少连续的3笔特征序列形成区间重叠
- 买卖点的背驰判断基于特征序列的力度对比
- 特征序列代表了线段运行过程中的"回调"或"反弹"


三、中枢（ZS）的识别逻辑

3.1 中枢的基本定义

定义澄清（重要）：
中枢的构成：
- 由线段内的特征序列笔组成（至少3笔特征序列）
- 这些特征序列笔必须有连续的区间重叠
- 注意：这里说的"3笔"是指3笔特征序列（与线段方向相反的笔）

示例理解：
```
向上线段包含：笔1(↑), 笔2(↓), 笔3(↑), 笔4(↓), 笔5(↑), 笔6(↓), 笔7(↑)
- 同向笔：笔1, 笔3, 笔5, 笔7（向上）
- 特征序列：笔2, 笔4, 笔6（向下，共3笔）
- 如果笔2、笔4、笔6的区间有重叠，则形成中枢
```

中枢形成的最小结构：
- 至少需要5笔：同向-反向-同向-反向-同向
- 其中3个反向笔（特征序列）形成重叠区间
- 所以完整结构至少需要5笔，但计算中枢区间只用特征序列的3笔

默认配置（zs_conf）：
need_combine = True          # 允许中枢合并
zs_combine_mode = "zs"       # 按中枢区间合并
one_bi_zs = False           # 不允许单笔中枢
zs_algo = "normal"          # 标准中枢算法

3.2 中枢形成的详细算法（zs_algo="normal"）

核心思路：
1. 在每个线段内，提取特征序列笔（与线段方向相反的笔）
2. 对特征序列笔进行滑动窗口检查，寻找区间重叠
3. 当至少3笔特征序列存在区间重叠时，形成中枢

详细计算逻辑：

步骤1：提取特征序列
```python
# 在线段seg内
feature_bi_list = []
for bi in seg.get_bi_list():
    if bi.dir != seg.dir:  # 与线段方向相反
        feature_bi_list.append(bi)
```

步骤2：检查区间重叠（滑动窗口）
```python
# 从前往后扫描特征序列笔
i = 0
while i < len(feature_bi_list):
    # 尝试以feature_bi_list[i]和feature_bi_list[i+1]为起点构建中枢
    if i + 1 >= len(feature_bi_list):
        break

    bi1 = feature_bi_list[i]
    bi2 = feature_bi_list[i + 1]

    # 计算前两笔的重叠区间
    low = max(bi1._low(), bi2._low())   # 两笔低点的最大值
    high = min(bi1._high(), bi2._high())  # 两笔高点的最小值

    if low >= high:
        # 没有重叠，继续下一组
        i += 1
        continue

    # 有重叠，尝试扩展中枢
    zs_bi_list = [bi1, bi2]
    j = i + 2

    # 向后扩展，检查后续笔是否仍在重叠区间内
    while j < len(feature_bi_list):
        bi_next = feature_bi_list[j]

        # 更新重叠区间
        new_low = max(low, bi_next._low())
        new_high = min(high, bi_next._high())

        if new_low >= new_high:
            # 不再重叠，中枢结束
            break

        # 仍然重叠，加入中枢
        zs_bi_list.append(bi_next)
        low = new_low
        high = new_high
        j += 1

    # 至少需要3笔特征序列才能形成中枢
    if len(zs_bi_list) >= 3:
        zs = CZS(
            begin_bi=zs_bi_list[0],
            end_bi=zs_bi_list[-1],
            low=low,
            high=high
        )
        # 记录中枢...
        # 下一次从中枢结束后继续
        i = i + len(zs_bi_list)
    else:
        i += 1
```

步骤3：计算中枢属性
```python
# 对于形成的中枢zs
zs.low = low                           # 中枢下沿
zs.high = high                         # 中枢上沿
zs.mid = (low + high) / 2              # 中枢中点
zs.peak_low = min(bi._low() for bi in zs_bi_list)    # 最低点
zs.peak_high = max(bi._high() for bi in zs_bi_list)  # 最高点
zs.begin_bi = zs_bi_list[0]           # 第一笔特征序列
zs.end_bi = zs_bi_list[-1]            # 最后一笔特征序列
```

步骤4：设置bi_in和bi_out
```python
# bi_in：进入中枢的笔（中枢第一笔特征序列的前一笔）
if zs.begin_bi.idx > 0:
    zs.bi_in = bi_list[zs.begin_bi.idx - 1]

# bi_out：离开中枢的笔（中枢最后一笔特征序列的后一笔）
if zs.end_bi.idx < len(bi_list) - 1:
    zs.bi_out = bi_list[zs.end_bi.idx + 1]
```

关键概念解释：

区间重叠计算：
```
笔1: [low1, high1]
笔2: [low2, high2]

重叠区间 = [max(low1, low2), min(high1, high2)]

判断重叠：
if max(low1, low2) < min(high1, high2):
    有重叠
else:
    无重叠
```

中枢区间 vs 中枢边界：
- 中枢区间[low, high]：所有特征序列笔的共同重叠部分（较窄）
- 中枢边界[peak_low, peak_high]：所有特征序列笔的极值范围（较宽）
- low >= peak_low, high <= peak_high

图示：
```
  peak_high ---|  /\
               | /  \
  high     ----|/____\---- 中枢上沿（重叠区间上界）
               |\    /|
               | \  / |
  low      ----|\/__\/|--- 中枢下沿（重叠区间下界）
               | \  /
  peak_low ----|  \/

               笔1 笔2 笔3（特征序列）
```
    
3.3 中枢的关键属性

中枢范围：
- low: 中枢下沿 = 构成中枢的笔的低点的最大值
- high: 中枢上沿 = 构成中枢的笔的高点的最小值
- mid: 中枢中点 = (low + high) / 2
  
中枢边界：
- peak_low: 中枢内所有笔的最低点
- peak_high: 中枢内所有笔的最高点
  
中枢笔信息：
- begin_bi: 中枢开始的笔（特征序列的第一笔）
- end_bi: 中枢结束的笔（特征序列的最后一笔）
- bi_in: 进入中枢的笔（begin_bi的前一笔）
- bi_out: 离开中枢的笔（end_bi的后一笔）
- bi_lst: 中枢内包含的所有笔列表
  
3.4 中枢合并逻辑（need_combine=True, zs_combine_mode="zs"）

合并条件：
对于相邻的两个中枢 zs1 和 zs2：
1. 必须在同一个线段内（seg_idx相同）
2. zs2不是单笔中枢
3. zs1和zs2的区间有重叠：
   has_overlap(zs1.low, zs1.high, zs2.low, zs2.high, equal=True)

合并后的中枢：
- low = min(zs1.low, zs2.low)
- high = max(zs1.high, zs2.high)
- peak_low = min(zs1.peak_low, zs2.peak_low)
- peak_high = max(zs1.peak_high, zs2.peak_high)
- 保留原始中枢在 sub_zs_lst 中
  
3.5 中枢延伸

延伸条件：
- 新的笔（特征序列）与当前中枢区间有重叠
- 更新中枢的 end_bi 为新笔
- 更新 peak_high 和 peak_low
  
四、买卖点（BSP）识别逻辑

4.1 买卖点默认配置（bs_point_conf）

divergence_rate = float("inf")        # 背驰比例（默认不限制）
min_zs_cnt = 1                       # 最少中枢数量
bsp1_only_multibi_zs = True          # 一类买卖点只考虑多笔中枢
max_bs2_rate = 0.9999                # 二类买卖点最大回撤比例
macd_algo = "peak"                   # MACD背驰算法（峰值）
bs1_peak = True                      # 一类买卖点必须是极值
bs_type = "1,1p,2,2s,3a,3b"         # 开启的买卖点类型
bsp2_follow_1 = True                 # 二类必须跟随一类
bsp3_follow_1 = True                 # 三类必须跟随一类
bsp3_peak = False                    # 三类不要求极值
bsp2s_follow_2 = False              # 类二不要求跟随二类
max_bsp2s_lv = None                 # 类二最大级别不限制
strict_bsp3 = False                 # 三类不严格模式
bsp3a_max_zs_cnt = 1                # 三类a最多1个中枢

4.2 一类买卖点（T1）

识别条件：

1. 基本条件：
  - 线段必须包含至少一个中枢
  - 如果 bsp1_only_multibi_zs=True，则只考虑多笔中枢（至少3笔的中枢）
  - 中枢数量 >= min_zs_cnt
    
2. 位置要求：
  - 线段的最后一个（多笔）中枢的 bi_out 存在
  - bi_out 的位置 >= 线段的 end_bi
  - end_bi 与中枢 bi_in 之间至少间隔2笔
    
3. 背驰判断（divergence_rate=inf时自动满足）：
in_metric = zs.bi_in.cal_macd_metric("peak", is_reverse=False)
out_metric = seg.end_bi.cal_macd_metric("peak", is_reverse=True)

背驰条件：out_metric <= divergence_rate * in_metric
  
4. MACD计算方法（macd_algo="peak"）：
  - 遍历笔内所有K线单元（klu）
  - 找出MACD柱子（红绿柱）的峰值（绝对值最大）
  - 向上笔：取MACD > 0 的最大值
  - 向下笔：取MACD < 0 的最大绝对值
    
5. 极值点要求（bs1_peak=True）：
  - 检查 bi_out 是否是中枢之后的极值点
  - 向下笔：bi_out 的低点必须低于中枢内所有笔的低点
  - 向上笔：bi_out 的高点必须高于中枢内所有笔的高点
    
买卖方向：
- 向下线段的末端 -> 买点
- 向上线段的末端 -> 卖点
  
特征值：
- divergence_rate: 背驰比例 = out_metric / in_metric
  
4.3 一类盘整买卖点（T1P）

识别条件：

1. 触发场景：
  - 线段没有有效的中枢可以形成T1买卖点
  - 或者中枢不满足T1的其他条件
    
2. 盘整判断：
last_bi = seg.end_bi
pre_bi = bi_list[last_bi.idx - 2]  # 倒数第3笔

条件：
- last_bi 和 pre_bi 在同一线段内（seg_idx相同）
- last_bi 的方向 = seg 的方向
- 没有创新高/新低：
  * 向下笔：last_bi._low() <= pre_bi._low()（盘整不创新低）
  * 向上笔：last_bi._high() >= pre_bi._high()（盘整不创新高）
  
3. 背驰判断：
in_metric = pre_bi.cal_macd_metric("peak", is_reverse=False)
out_metric = last_bi.cal_macd_metric("peak", is_reverse=True)

背驰条件：out_metric <= divergence_rate * in_metric
  
4.4 二类买卖点（T2）

识别条件：

1. 基本结构：
一类买卖点笔（bsp1_bi） -> 突破笔（break_bi） -> 回拉笔（bsp2_bi）
即：bsp2_bi = bi_list[bsp1_bi.idx + 2]
  
2. 前置条件（bsp2_follow_1=True）：
  - 必须先存在一类买卖点（bsp1_bi在买卖点字典中）
    
3. 回撤要求：
retrace_rate = bsp2_bi.amp() / break_bi.amp()

条件：retrace_rate <= max_bs2_rate (默认0.9999，即几乎任何回撤都可以)
  
计算说明：
- bi.amp() = abs(bi.get_end_val() - bi.get_begin_val())
- 回撤比例表示回拉笔相对突破笔的幅度比
  
4.5 类二买卖点（T2S）

识别条件：

1. 触发场景：
  - 二类买卖点不成立时（retrace_rate > max_bs2_rate）
  - 或者作为二类买卖点的延伸
    
2. 递归查找：
从 bsp2_bi 开始，每次跳过2笔（bias += 2）
bsp2s_bi = bi_list[bsp2_bi.idx + bias]

停止条件：
- bias/2 > max_bsp2s_lv（如果设置了级别限制）
- bsp2s_bi 跨线段了
- bsp2s_bi 与前一个笔不再有区间重叠
- bsp2s_bi 突破了 break_bi 的极值
  
3. 区间重叠检查：
首次（bias=2）：
检查 bsp2_bi 和 bsp2s_bi 是否有重叠
如果有，计算重叠区间 [low, high]

后续（bias>2）：
检查 bsp2s_bi 是否与之前的重叠区间 [low, high] 有重叠
  
4. 回撤检查：
retrace_rate = abs(bsp2s_bi.get_end_val() - break_bi.get_end_val()) / break_bi.amp()

条件：retrace_rate <= max_bs2_rate
  
4.6 三类买卖点（T3A和T3B）

前置条件（bsp3_follow_1=True）：
- 必须存在一类买卖点
  
T3A：中枢在一类买卖点之后

识别条件：

1. 结构要求：
  - 一类买卖点所在线段的下一个线段（next_seg）
  - next_seg 中有多笔中枢
    
2. 中枢要求：
  - 取 next_seg 的第一个多笔中枢（first_zs）
  - 如果 strict_bsp3=True，则 first_zs.bi_in 必须紧跟 bsp1_bi
    
3. 三类点位置：
bsp3_bi = bi_list[zs.bi_out.idx + 1]  # 中枢出来后的第一笔

约束：
- bsp3_bi 的方向与 next_seg 方向相反
- bsp3_bi 仍在 next_seg 内（或最后一个未确认线段内）
  
4. 不回中枢检查：
def bsp3_back2zs(bsp3_bi, zs):
    向下笔：bsp3_bi._low() >= zs.high
    向上笔：bsp3_bi._high() <= zs.low

必须不回中枢才是有效的三类买卖点
  
5. 极值检查（bsp3_peak=False时不要求）：
def bsp3_break_zspeak(bsp3_bi, zs):
    向下笔：bsp3_bi._high() >= zs.peak_high
    向上笔：bsp3_bi._low() <= zs.peak_low
  
6. 多中枢处理（bsp3a_max_zs_cnt=1）：
  - 最多检查next_seg中的前1个中枢
  - 可以配置为更多
    
T3B：中枢在一类买卖点之前

识别条件：

1. 结构要求：
  - 使用一类买卖点所在线段（seg）的最后一个多笔中枢（cmp_zs）
  - 如果 strict_bsp3=True，则 cmp_zs.bi_out 必须是 bsp1_bi
    
2. 三类点查找：
从 bsp1_bi.idx + 2 开始，每次跳2笔（同方向的笔）
在下一个线段（next_seg）或未确认区域查找

bsp3_bi 必须满足：
- 在合适的线段范围内
- 不回到中枢区间（不满足bsp3_back2zs）
  
3. 搜索终止：
  - 找到第一个不回中枢的笔就是T3B
  - 或者到达线段边界
    
五、实际计算流程

5.1 完整计算链

原始K线数据
    ↓
K线合并（处理包含关系）
    ↓
识别分型（顶分型/底分型）
    ↓
形成笔（Bi）
    ↓
在线段（Seg）内提取特征序列
    ↓
根据特征序列计算中枢（ZS）
    ↓
根据中枢和笔计算买卖点（BSP）

5.2 中枢计算的关键步骤

# 伪代码
for seg in seg_list:
    # 1. 清空该线段的中枢列表
    seg.clear_zs_lst()
    
    # 2. 提取线段内的笔
    seg_bi_list = bi_list[seg.start_bi.idx : seg.end_bi.idx+1]
    
    # 3. 筛选出特征序列（与线段方向相反的笔）
    free_bi_list = []
    for bi in seg_bi_list:
        if bi.dir != seg.dir:  # 特征序列
            # 4. 尝试加入现有中枢或形成新中枢
            if len(zs_list) > 0 and zs_list[-1].try_add_to_end(bi):
                # 延伸现有中枢
                continue
            else:
                # 加入待处理列表
                free_bi_list.append(bi)
                
            # 5. 尝试构造中枢
            if len(free_bi_list) >= 2:
                zs = try_construct_zs(free_bi_list[-2:])
                if zs is not None:
                    zs_list.append(zs)
                    free_bi_list = []
                    
            # 6. 中枢合并
            while len(zs_list) >= 2:
                if zs_list[-2].combine(zs_list[-1], "zs"):
                    zs_list.pop()
                else:
                    break
    
    # 7. 更新中枢的bi_in, bi_out, bi_lst
    for zs in zs_list:
        if zs.is_inside(seg):
            zs.set_bi_in(bi_list[zs.begin_bi.idx - 1])
            zs.set_bi_out(bi_list[zs.end_bi.idx + 1])
            zs.set_bi_lst(bi_list[zs.begin_bi.idx : zs.end_bi.idx+1])
            seg.add_zs(zs)

5.3 买卖点计算顺序

# 伪代码
for seg in seg_list:
    # 1. 计算一类买卖点（T1和T1P）
    cal_single_bs1point(seg)
    
    # 2. 计算二类买卖点（T2和T2S）
    treat_bsp2(seg)
    
    # 3. 计算三类买卖点（T3A和T3B）
    treat_bsp3_after(seg)   # T3A
    treat_bsp3_before(seg)  # T3B

六、关键数据结构

6.1 笔（CBi）

class CBi:
    idx: int                    # 笔的索引
    begin_klc: CKLine          # 起始合并K线
    end_klc: CKLine            # 结束合并K线
    dir: BI_DIR                # 方向（UP/DOWN）
    is_sure: bool              # 是否确定
    seg_idx: int               # 所属线段索引
    bsp: CBS_Point             # 关联的买卖点
    
    # 方法
    get_begin_val()            # 起始值（低点或高点）
    get_end_val()              # 结束值（高点或低点）
    amp()                      # 振幅
    cal_macd_metric()          # 计算MACD指标

6.2 中枢（CZS）

class CZS:
    begin_bi: CBi              # 开始笔
    end_bi: CBi                # 结束笔
    low: float                 # 中枢下沿
    high: float                # 中枢上沿
    mid: float                 # 中枢中点
    peak_low: float            # 最低点
    peak_high: float           # 最高点
    bi_in: CBi                 # 进入笔
    bi_out: CBi                # 离开笔
    bi_lst: List[CBi]          # 中枢内笔列表
    sub_zs_lst: List[CZS]      # 合并前的子中枢
    
    # 方法
    is_one_bi_zs()             # 是否单笔中枢
    in_range(bi)               # 笔是否在中枢区间内
    try_add_to_end(bi)         # 尝试延伸中枢
    combine(zs2)               # 合并中枢
    is_divergence()            # 判断背驰
    end_bi_break()             # 离开笔是否突破中枢

6.3 买卖点（CBS_Point）

class CBS_Point:
    bi: CBi                    # 关联的笔
    klu: CKLine_Unit           # 关联的K线单元
    is_buy: bool               # 是否买点
    type: List[BSP_TYPE]       # 买卖点类型（可能多个）
    relate_bsp1: CBS_Point     # 关联的一类买卖点
    features: CFeatures        # 特征值字典

七、边界情况与实时更新处理

7.1 数据不足时的处理

K线数据初始化：
- 至少需要3根原始K线才能开始处理包含关系
- 至少需要3根处理后的K线才能识别第一个分型
- 至少需要2个相反的分型才能形成第一笔
- 至少需要5笔才能形成第一个线段
- 至少需要5笔才能形成第一个中枢（1个线段+3笔特征序列）

建议：
- 在数据不足时，返回空列表或标记为"数据不足"
- 不要强行输出不完整的结构
- 至少等待足够的K线数据再开始计算

MACD计算的冷启动：
- EMA需要历史数据才能稳定
- 建议至少需要 2 * slow（即52根K线）才开始计算MACD
- 在MACD值不稳定期间，可以跳过背驰判断或使用默认值

7.2 实时更新机制

虚笔与确定笔的转换：
```python
# 虚笔的状态转换
虚笔 (is_sure=False) → 确定笔 (is_sure=True)

触发条件：
1. 当新的相反分型出现时，虚笔转为确定笔
2. 虚笔的终点分型被更新时，仍保持虚笔状态
3. 当虚笔之后出现新笔时，虚笔转为确定笔
```

实时更新流程：
```
新K线到来
    ↓
1. 更新K线合并
    - 检查是否与前一根K线包含
    - 更新或创建新的合并K线
    ↓
2. 更新分型
    - 检查最后3根合并K线是否形成分型
    - 更新或创建新分型
    - 处理连续分型（保留最优）
    ↓
3. 更新笔
    - 检查是否能形成新笔
    - 更新虚笔或转为确定笔
    - 检查笔的延伸（同类型分型极值更优）
    ↓
4. 更新线段
    - 检查特征序列是否被破坏
    - 更新虚线段或创建新线段
    ↓
5. 更新中枢
    - 重新计算受影响线段的中枢
    - 检查中枢延伸和合并
    ↓
6. 更新买卖点
    - 重新计算受影响线段的买卖点
    - 更新T1/T2/T3类型
```

7.3 回溯计算范围

最小化回溯原则：
- 当确定笔不变时，不需要重新计算
- 只有虚笔或最后几笔改变时才回溯

回溯范围建议：
```python
# 当新K线到来
if 最后一根合并K线被更新:
    回溯范围 = [最后一个分型开始的位置]

if 新分型形成:
    回溯范围 = [倒数第2个分型位置]

if 新笔形成:
    回溯范围 = [倒数第2笔所在的线段]
    重新计算该线段及之后的：
    - 线段
    - 中枢
    - 买卖点

if 新线段形成:
    回溯范围 = [倒数第2个线段]
    重新计算该线段及之后的：
    - 中枢
    - 买卖点
```

7.4 边界情况处理

情况1：分型更新
```python
问题：连续出现多个顶分型时如何处理？
处理：
1. 实时维护一个"候选分型"列表
2. 当新的同类型分型出现时，比较极值
3. 只保留最后一个（最优）分型
4. 已经形成笔的分型不再更新
```

情况2：笔的延伸
```python
问题：当前笔的末端出现更极端的同类型分型？
处理：
1. 检查新分型类型是否与当前笔末端分型相同
2. 如果相同且极值更优：
   - 向上笔：新分型高点 > 当前笔末端高点
   - 向下笔：新分型低点 < 当前笔末端低点
3. 更新笔的end_klc和end_bi
4. 保持笔的is_sure状态（仍为虚笔）
```

情况3：线段的延伸
```python
问题：虚线段如何延伸？
处理：
1. 虚线段是seg_list中的最后一个线段
2. 每次新笔形成时，检查特征序列是否被破坏
3. 如果未破坏，更新虚线段的end_bi
4. 如果被破坏，虚线段转为确定线段，创建新线段
```

情况4：中枢延伸
```python
问题：新的特征序列笔出现时，如何判断是延伸还是新中枢？
处理：
1. 计算新笔与当前中枢区间[low, high]的重叠
2. 如果新笔的区间与[low, high]有重叠：
   - 更新中枢的end_bi为新笔
   - 更新中枢的peak_low和peak_high
   - 重新计算中枢的bi_out
3. 如果无重叠：
   - 当前中枢结束
   - 新笔加入待处理列表，准备构建新中枢
```

情况5：最后一笔/线段/中枢的处理
```python
问题：最后一个结构（笔/线段/中枢）如何标记？
处理：
1. 使用is_sure标记：
   - is_sure=True：确定的，不会再改变
   - is_sure=False：虚的，可能随新数据改变

2. 在实时计算中：
   - 最后一笔：is_sure=False（虚笔）
   - 最后一个线段：is_sure=False（虚线段）
   - 最后一个中枢：可能延伸

3. 在历史数据回测中：
   - 所有结构都是确定的（is_sure=True）
   - 不需要考虑实时更新
```

7.5 性能优化建议

增量计算：
```python
# 不要每次都从头计算
# 维护已确定的结构列表

确定笔列表 = []    # is_sure=True的笔
虚笔列表 = []      # is_sure=False的笔（最多1个）

当新K线到来：
1. 只更新虚笔或最后几个确定笔
2. 不要重新计算所有确定笔
3. 中枢和买卖点同理
```

缓存机制：
```python
# 缓存计算结果
class CachedCalculator:
    def __init__(self):
        self.klc_cache = []          # 合并K线缓存
        self.fx_cache = []           # 分型缓存
        self.bi_cache = []           # 确定笔缓存
        self.seg_cache = []          # 确定线段缓存
        self.zs_cache = []           # 中枢缓存

        self.last_update_kl_idx = -1  # 最后更新的K线索引

    def update(self, new_kl):
        # 只计算受影响的部分
        affected_range = self.get_affected_range(new_kl)
        self.recalculate(affected_range)
```

数据结构优化：
- 使用索引代替对象引用（节省内存）
- 使用数组代替列表（提高访问速度）
- 预分配空间（减少动态扩容开销）

7.6 调试建议

日志记录：
```python
# 关键节点输出日志
def log_structure_change(event_type, old_value, new_value):
    """
    记录结构变化
    event_type: "fx_update", "bi_new", "seg_break", "zs_extend"等
    """
    logger.info(f"{event_type}: {old_value} -> {new_value}")

# 示例
log_structure_change("bi_new", None, f"笔{bi.idx}: {bi.dir}")
log_structure_change("zs_extend", f"[{old_low}, {old_high}]",
                    f"[{new_low}, {new_high}]")
```

可视化验证：
- 在图表上标注分型位置
- 画出笔的连接线
- 标记线段的起止点
- 用矩形框表示中枢区间
- 用不同颜色标记买卖点

单元测试：
- 准备典型的K线序列
- 验证每个步骤的输出
- 特别测试边界情况（连续分型、笔延伸、线段破坏等）

八、TradingView转换要点

1. 状态管理：
  - 需要维护笔列表、中枢列表、买卖点列表
  - 需要实现确定笔和虚笔的切换机制
    
2. 特征序列提取：
  - 在每个线段内，只有与线段方向相反的笔才参与中枢计算
  - 这是缠论中"特征序列"的概念
    
3. 实时更新：
  - 最后一笔可能是虚笔，需要动态更新
  - 中枢和买卖点都需要支持回溯计算
    
4. 性能优化：
  - 只需要重新计算受影响的部分
  - 可以缓存确定的笔、中枢和买卖点
    
5. 配置简化：
  - 按默认配置实现
  - 关键参数都已在上文标注
    
