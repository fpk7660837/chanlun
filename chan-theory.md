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
  
分型有效性检查（bi_fx_check="strict"）：
对于顶分型 -> 底分型：
- 顶分型的最低点（包含前、中、后三根K线）> 底分型的最高点（包含前、中、后三根K线）
- 顶分型的高点 > 底分型的高点
  
对于底分型 -> 顶分型：
- 底分型的最高点（包含前、中、后三根K线）< 顶分型的最低点（包含前、中、后三根K线）
- 底分型的低点 < 顶分型的低点
  
极值点要求（bi_end_is_peak=True）：
- 从起始分型到结束分型之间，结束分型必须是这段区间的极值点
- 向上笔：结束分型必须是区间内的最高点
- 向下笔：结束分型必须是区间内的最低点
  
2.3 笔的更新机制

确定笔（is_sure=True）：
- 已经完整成立的笔
  
虚笔（is_sure=False）：
- 为了实时计算而添加的未确认的笔
- 当新的K线出现时，如果满足条件会转为确定笔
  
笔的延伸：
- 如果新的分型与当前笔末端分型类型相同，且极值更优，则更新笔的终点
  
三、中枢（ZS）的识别逻辑

3.1 中枢的基本定义
中枢的构成： 至少由3笔组成，其中至少有连续的3笔（方向相反的笔）存在区间重叠

默认配置（zs_conf）：
need_combine = True          # 允许中枢合并
zs_combine_mode = "zs"       # 按中枢区间合并
one_bi_zs = False           # 不允许单笔中枢
zs_algo = "normal"          # 标准中枢算法

3.2 中枢形成的详细算法（zs_algo="normal"）

核心计算逻辑：

1. 线段内中枢识别：
  - 遍历线段内的笔
  - 只处理与线段方向相反的笔（即特征序列）
  - 对于连续的反向笔，检查是否满足中枢条件
    
2. 中枢成立条件：
设有反向笔列表 [bi1, bi2, ...]
对于最近的2笔（或更多）：

计算：
low = max(bi._low() for bi in 笔列表)   # 各笔低点的最大值
high = min(bi._high() for bi in 笔列表)  # 各笔高点的最小值

如果 low < high，则形成中枢
中枢区间 = [low, high]
  
3. 特征序列的提取：
  - 在一个线段内，与线段方向相反的笔构成特征序列
  - 例如：向上线段中，向下的笔是特征序列
  - 只有特征序列的笔才参与中枢计算
    
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

七、TradingView转换要点

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
    
