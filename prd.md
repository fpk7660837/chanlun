好的！以下是一份基于缠论笔中枢的买卖点识别系统 PRD（产品需求文档），专为程序化交易设计。内容覆盖从原始K线到三类买点的完整逻辑、判定规则、数据结构、边界条件和算法细节，可直接用于开发。

📄 缠论笔中枢买卖点识别系统 —— PRD 文档

版本：1.0
目标平台：量化交易平台（Python / C++ / Rust 等）
适用级别：单周期（如5分钟、30分钟），支持扩展多级别联立
核心目标：在不使用未来函数的前提下，实时识别第一、第二、第三类买点（对称逻辑适用于卖点）

一、输入与输出定义
1.1 输入数据
字段 类型 说明
------ ------ ------
timestamp int64 Unix 时间戳（秒或毫秒）
open float 开盘价
high float 最高价
low float 最低价
close float 收盘价
volume float 成交量（可选）
✅ 要求：按时间升序排列，无缺失，无重复。

二、处理流程总览

mermaid
graph TD
A[原始K线] --> B[包含关系处理]
B --> C[分型识别]
C --> D[笔构建]
D --> E[笔中枢构建]
E --> F[背驰检测]
F --> G[1/2/3买点判定]
G --> H[信号输出]

三、模块详细规范

模块 1：包含关系处理（K线合并）
目的
消除“包含K线”（即一根K线完全包含下一根），确保分型有效。
规则（标准缠论）
从左到右遍历；
若当前K线与前一根存在包含关系：
上升趋势（前高 < 当前高）→ 合并为 高取高，低取高前低；
下降趋势（前低 > 当前低）→ 合并为 低取低，高取前高；
趋势方向由最近两个非包含K线决定；
合并后生成新K线序列 merged_klines。
输出
merged_klines: List[KLine]
原始索引映射表 orig_idx_map: List[int]（用于回溯）
⚠️ 必须在分型前执行！

模块 2：分型识别（Fractal Detection）
定义
顶分型：三根连续K线，中间高点最高，左右高点均低于它，且三者不共用K线；
底分型：三根连续K线，中间低点最低，左右低点均高于它。
判定条件
分型间隔 ≥ 1 根K线（即不能相邻）；
分型必须出现在 merged_klines 上；
分型确认需等待第4根K线收盘（避免未来函数）。
数据结构
python
class Fractal:
idx: int # 在 merged_klines 中的索引
orig_idx: int # 对应原始K线索引
type: str # 'top' or 'bottom'
price: float # 顶分型=high, 底分型=low
输出
fractals: List[Fractal]，按时间排序

模块 3：笔构建（Bi Construction）
笔定义
由一个底分型 + 一个顶分型（或反之）构成；
方向交替：up → down → up …；
新笔必须突破前一笔的极值（确认有效性）。
构建规则
1. 初始化：取第一个分型作为起点；
2. 遍历后续分型：
若方向交替（如上一笔是 up，当前为 bottom→top 则为 down）；
且当前分型价格 突破前一笔的端点（例如：向下笔的 low < 前向上笔的 low）；
则形成新笔；
3. 笔确认：需等待下一分型出现才能最终确认当前笔结束（避免漂移）。
数据结构
python
class Bi:
start_f_idx: int # 起始分型索引（在 fractals 中）
end_f_idx: int # 结束分型索引
direction: str # 'up' or 'down'
high: float # 笔的最高价
low: float # 笔的最低价
start_k_idx: int # 起始K线索引（原始）
end_k_idx: int # 结束K线索引（原始）
confirmed: bool # 是否已确认（防止未来函数）
输出
bi_list: List[Bi]
💡 提示：未确认的笔（最后一笔）可用于实时监控，但不可用于买卖点判定。

模块 4：笔中枢构建（ZhongShu from Bi）
中枢定义
至少由连续3笔构成（down-up-down 或 up-down-up）；
三笔的价格区间存在重叠部分；
重叠区间 = [ZD, ZG]，其中：
ZG = min(第1笔高, 第3笔高)
ZD = max(第1笔低, 第3笔低)
若 ZG > ZD → 有效中枢
扩展规则
若第4、5…笔仍在 [ZD, ZG] 内震荡，则扩展中枢（更新 end_bi_idx）；
一旦新笔完全离开中枢（如向上突破 ZG + 一定幅度），则中枢封闭。
数据结构
python
class ZhongShu:
start_bi_idx: int # 起始笔索引（bi_list）
end_bi_idx: int # 结束笔索引
zg: float # 中枢高
zd: float # 中枢低
direction: str # 'down' (下跌中枢) or 'up' (上涨中枢)
closed: bool # 是否已结束
输出
zhongshu_list: List[ZhongShu]

模块 5：背驰检测（Divergence Detection）
方法：MACD 柱状图面积法（推荐）

1. 计算 MACD（默认参数：12, 26, 9）；
2. 对每一段同向笔（如下跌笔），计算其覆盖K线的 MACD 柱状图绝对值之和（或仅负值面积）；
3. 比较最近两段同向笔的面积：
若当前笔价格创新低，但面积 < 前一笔 → 底背驰成立
数据结构
python
divergence_map = {
bi_idx: {
'has_divergence': bool,
'area_current': float,
'area_prev': float
}
}
✅ 仅对已确认的笔进行背驰判断。

模块 6：买卖点判定逻辑
通用原则
所有买点必须在反向分型确认后标记（即下一K线收盘后）；
不使用未来数据；
买点类型互斥（同一位置只标一种）。

6.1 第一类买点（Buy_1）

触发条件（AND）：
1. 存在一个已确认的向下笔（bi.direction == 'down'）；
2. 该笔 price.low < 前一个向下笔的 low（创新低）；
3. 该笔存在 底背驰（divergence_map[bi_idx]['has_divergence'] == True）；
4. 该笔结束后，出现一个底分型（即反弹起点）；
5. 该底分型已被下一K线确认（即当前K线是分型后第1根）；

标记位置：
K线索引 = 底分型所在K线的下一根K线收盘时；
价格 = 该K线收盘价（或分型 low）；

数据结构：
python
Signal(type='buy_1', k_idx=int, price=float, ref_bi=bi_idx)

6.2 第二类买点（Buy_2）

前提：已存在一个 Buy_1 信号（记为 buy1）

触发条件（AND）：
1. Buy_1 之后出现向上笔（反弹）；
2. 随后出现向下回调笔；
3. 该回调笔的 low >= buy1.price（不破前低）；
4. 回调结束后出现底分型，且被确认；
5. 该底分型位置在 Buy_1 之后，且未出现新中枢破坏；

标记位置：确认底分型的下一根K线

注意：允许多个2买（如二次回踩）

6.3 第三类买点（Buy_3）

前提：存在一个已封闭的下跌中枢（ZhongShu.direction == 'down'）

触发条件（AND）：
1. 中枢结束后，出现向上突破笔（high > ZG）；
2. 随后出现回调笔；
3. 回调笔的 low > ZG（未进入中枢内部）；
4. 回调结束后出现底分型，且被确认；
5. 该底分型发生在中枢结束之后；

标记位置：确认底分型的下一根K线

模块 7：信号输出与去重
输出格式
json
{
"signals": [
{
"type": "buy_1", // "buy_2", "buy_3"
"k_index": 1250, // 原始K线索引（触发信号的K线）
"price": 10.25,
"timestamp": 1712345678,
"ref_objects": {
"bi_idx": 24,
"zhongshu_idx": null
}
}
]
}
去重规则
同一K线只允许一个买点类型（优先级：1买 > 2买 > 3买）；
已标记信号不再重复触发；

四、边界与异常处理

场景 处理方式
------ --------
K线不足 返回空信号
无分型 无法构建笔，跳过
笔未确认 不参与中枢/买卖点计算
中枢未封闭 不用于3买判定
背驰计算失败（MACD未就绪） 跳过该笔

