# Chan Theory: From Fractals to Buy/Sell Points - Complete Logic Summary

## Overview

Chan Theory (缠论) builds a hierarchical structure from raw price data to trading signals. The core philosophy is that market movements have fractal patterns that repeat across different time scales.

---

## Building Blocks (Bottom-Up)

### 1. **Merged K-Lines (合并K线)**

**Purpose**: Remove noise and establish the foundation for fractal identification.

**Process**:
- Two K-lines have "包含关系" (containment) if one's range is completely within the other's
- In uptrend: merge by taking `max(high1, high2)` and `max(low1, low2)`
- In downtrend: merge by taking `min(high1, high2)` and `min(low1, low2)`
- Result: A cleaned series of K-lines with no containment relationships

**Why**: Eliminates minor fluctuations that don't represent true market structure.

---

### 2. **Fractals (分型)**

**Purpose**: Identify local extremes (tops and bottoms).

**Definition**:
- **Top Fractal (顶分型)**: Middle K-line's high and low are both higher than adjacent K-lines
  - `K2.high > K1.high AND K2.high > K3.high`
  - `K2.low > K1.low AND K2.low > K3.low`

- **Bottom Fractal (底分型)**: Middle K-line's high and low are both lower than adjacent K-lines
  - `K2.high < K1.high AND K2.high < K3.high`
  - `K2.low < K1.low AND K2.low < K3.low`

**Confirmation**: A fractal is confirmed when the 4th K-line appears (preventing premature identification).

**Why**: Fractals mark potential turning points in price action.

---

### 3. **Strokes (笔, Bi)**

**Purpose**: Connect significant fractals to form directional price movements.

**Definition**:
- A stroke connects two consecutive fractals of opposite types
- **Upward Stroke (向上笔)**: From bottom fractal to top fractal
- **Downward Stroke (向下笔)**: From top fractal to bottom fractal

**Requirements**:
- At least 5 merged K-lines between fractals (configurable: `min_pen_bars`)
- Gap counting: Gaps between K-lines can count as additional K-lines
- End-point must be peak: Stroke endpoint must be the extreme value within the stroke

**Properties Calculated**:
- `high`, `low`: Price range
- `direction`: 1 (up) or -1 (down)
- `macd_area`: Sum of MACD histogram values (momentum measure)
- `amplitude`: Absolute price change
- `slope`: `amplitude / kx_count` (strength per bar)

**Why**: Strokes represent the basic directional movements of the market.

---

### 4. **Pivots (中枢, Zhongshu)**

**Purpose**: Identify consolidation zones where price oscillates.

**Definition** (3-Stroke Pivot):
- Formed by 3 consecutive strokes with alternating directions
- **Upward Pivot (上涨中枢)**: Down-Up-Down structure
- **Downward Pivot (下跌中枢)**: Up-Down-Up structure

**Key Boundaries**:
- **ZG (中枢上沿)**: `min(high1, high2)` - the lower of the first two highs
- **ZD (中枢下沿)**: `max(low1, low2)` - the higher of the first two lows
- **GG (最高点)**: Highest point across all strokes in pivot
- **DD (最低点)**: Lowest point across all strokes in pivot

**Example** (Upward Pivot):
```
Stroke1: Down (low=100, high=110)
Stroke2: Up   (low=105, high=120)
Stroke3: Down (low=108, high=115)

ZG = min(120, 115) = 115  ← Upper boundary
ZD = max(105, 108) = 108  ← Lower boundary
Pivot range: [108, 115]
```

**Extension**:
- If subsequent strokes continue to overlap with [ZD, ZG], they extend the pivot
- Pivot "closes" when a stroke breaks out of [ZD, ZG] range

**Why**: Pivots represent market equilibrium zones - areas where bulls and bears are balanced.

---

## Buy/Sell Points (买卖点) - The Core Trading Signals

### **Type 1 Buy/Sell Points (第一类买卖点)** - Trend Reversal

**Definition**:
> In a trending move, after leaving the last pivot, price creates a new extreme but with weakening momentum (divergence) → Reversal signal

**Type 1 Buy (1买) - Downtrend Reversal**:

**Conditions**:
1. **Context**: Downtrend with established pivot
2. **Structure**:
   - Enter Pivot: Down stroke breaks into pivot
   - Pivot Forms: 3+ strokes oscillate within range
   - Exit Pivot: Down stroke breaks below ZD (pivot's lower boundary)
3. **Divergence**: Current exit stroke vs entering stroke:
   - Price: Makes new low (`current.low < entering.low`)
   - BUT momentum weakens:
     - MACD area decreases, OR
     - Price amplitude decreases, OR
     - Slope (strength) decreases
4. **Signal**: Low point of the exit stroke = **Type 1 Buy**

**Visual**:
```
Price
  │
  │  Enter ↓     Pivot Zone [ZD, ZG]     Exit ↓
  │    ↓     ┌─────────────────┐          ↓
  │    ↓     │    ↗  ↘  ↗     │          ↓  ← New Low
  ├────┴─────┴─────────────────┴──────────┴─────
  │                             ZD              ↑
  │                                          1买 (Divergence)
```

**Type 1 Sell (1卖) - Uptrend Reversal**:

**Conditions**:
1. **Context**: Uptrend with established pivot
2. **Structure**: Up stroke breaks above ZG
3. **Divergence**: Price makes new high but momentum weakens
4. **Signal**: High point of exit stroke = **Type 1 Sell**

**Critical Insight**:
- **Location**: Type 1 points appear OUTSIDE the pivot (below ZD for buy, above ZG for sell)
- **Comparison**: Must compare with the stroke that ENTERED the pivot, not any random previous stroke
- **Essence**: "Price lies, momentum tells truth" - new extreme with weakening force signals exhaustion

---

### **Type 2 Buy/Sell Points (第二类买卖点)** - Confirmation/Second Chance

**Definition**:
> After Type 1 signal, price moves in the reversal direction, then pulls back → Safer entry with confirmation

**Type 2 Buy (2买)**:

**Conditions**:
1. **Prerequisite**: Type 1 Buy exists at `price_B1`
2. **Structure** (exact sequence):
   - Pen 1: Type 1 Buy stroke (down, ending at low)
   - Pen 2: Upward stroke (confirming reversal)
   - Pen 3: Downward pullback stroke ← **This is Type 2 Buy**
3. **Price**: Pullback low ≥ Type 1 Buy level (允许2%容差)
   - `pen3.low >= price_B1 - 2%`
4. **Signal**: Low point of pullback stroke = **Type 2 Buy**

**Visual**:
```
Price
  │                  ② Up
  │              ┌───────┐
  │              │       │  ③ Pullback ↓
  │              │       └───────┐  ← 2买 (Should hold above B1)
  ├──────────────┴───────────────┴─────
  │          ① Down to B1
  │              ↓ 1买
```

**Type 2 Sell (2卖)**:
- Mirror logic: After 1卖, price drops, then bounces
- Bounce high should not exceed Type 1 Sell level

**Critical Insight**:
- **Location**: Type 2 is AFTER the pivot, in the new trend direction
- **Timing**: Exactly the 2nd stroke after Type 1 (pen_idx = Type1_idx + 2)
- **Safety**: Provides confirmation that Type 1 wasn't a false signal
- **Risk/Reward**: Lower reward than Type 1 but higher probability

---

### **Type 3 Buy/Sell Points (第三类买卖点)** - Trend Continuation

**Definition**:
> After leaving a pivot, price pulls back to TEST the pivot boundary but doesn't re-enter → Continuation signal

**Type 3 Buy (3买)**:

**Conditions**:
1. **Context**: Upward pivot exists (Up-Down-Up structure, direction=1)
2. **Structure** (exact sequence):
   - Pivot closes
   - Exit stroke: Upward stroke breaks ABOVE ZG
   - Pullback stroke: Downward stroke pulls back
3. **Key Test**: Pullback low **DOES NOT** break ZG
   - `pullback.low >= ZG`  ← NOT `>`, must include `=`
4. **Signal**: Low point of pullback stroke = **Type 3 Buy**

**Visual**:
```
Price
  │           Exit ↑
  │              ↗────┐ New High
  │             ↗     │ Pullback ↓
  │            ↗      └────┐  ← 3买 (Tests but holds ZG)
  ├───────────┬───────────┬─────
  │    Pivot  │ ZG ←──────┘ (Doesn't break back through)
  │   [ZD,ZG] │
  │           │ ZD
  │           └────
```

**Type 3 Sell (3卖)**:

**Conditions**:
1. **Context**: Downward pivot (Down-Up-Down structure, direction=-1)
2. **Exit**: Downward stroke breaks BELOW ZD
3. **Bounce**: Upward stroke bounces but `bounce.high <= ZD`
4. **Signal**: High point of bounce stroke = **Type 3 Sell**

**Critical Insight**:
- **Location**: Type 3 appears just OUTSIDE the pivot, testing the boundary
- **Boundary Condition**: `>=` and `<=` (not strict `>` or `<`)
  - "不跌破ZG" (doesn't break ZG) means `>= ZG`, not `> ZG`
  - "不升破ZD" (doesn't break ZD) means `<= ZD`, not `< ZD`
- **Essence**: Market tests support/resistance (pivot boundary) and respects it → Trend continues
- **Psychology**: Weak hands exit, strong hands hold → Good entry for trend followers

---

## Buy/Sell Points Comparison

| Type | Context | Location | Risk | Reward | Probability | Best For |
|------|---------|----------|------|--------|-------------|----------|
| **Type 1** | Trend exhaustion | Outside pivot (furthest) | Highest | Highest | Medium | Aggressive reversal traders |
| **Type 2** | Post-Type 1 pullback | After Type 1 | Medium | Medium | High | Conservative traders, confirmation seekers |
| **Type 3** | Trend continuation | Pivot boundary | Low | Low-Medium | Highest | Trend followers, lower volatility |

---

## Complete Flow: From Data to Signal

```
Raw K-Lines
    ↓ [包含处理]
Merged K-Lines
    ↓ [3K线组合判断]
Fractals (顶分型/底分型)
    ↓ [连接对立分型]
Strokes/Pens (笔)
    ↓ [3笔重叠]
Pivots (中枢) [ZD, ZG]
    ↓ [离开中枢 + 背驰/回试]
Buy/Sell Points
    │
    ├─ Type 1: Divergence reversal (离开中枢后背驰)
    ├─ Type 2: Post-Type 1 pullback (Type 1后的次级回调)
    └─ Type 3: Boundary test (离开后回试边界但不入中枢)
```

---

## The Essence of Buy/Sell Points

### **Type 1 - The Divergence Principle**
- **What**: Price creates new extreme, momentum doesn't
- **Why it works**: Market exhaustion → reversal imminent
- **Chan's insight**: "背驰" (divergence) is THE signal of trend completion

### **Type 2 - The Confirmation Principle**
- **What**: First pullback after reversal signal
- **Why it works**: Tests if Type 1 was real; holds = trend confirmed
- **Chan's insight**: Second chance for those who missed Type 1

### **Type 3 - The Respect Principle**
- **What**: Pullback respects pivot boundary
- **Why it works**: Market acknowledges support/resistance → trend continues
- **Chan's insight**: "回试不入中枢" (tests but doesn't re-enter) = strength

---

## Key Relationships

### **Type 1 vs Pivot**:
- **Must break THROUGH** pivot boundary (ZD for buy, ZG for sell)
- Compares with stroke that ENTERED the pivot
- Signal appears OUTSIDE pivot

### **Type 2 vs Type 1**:
- **Must follow Type 1** exactly (2 strokes later)
- Tests Type 1 level
- Confirms reversal direction

### **Type 3 vs Pivot**:
- **Must respect** pivot boundary (doesn't re-enter)
- Appears at the boundary zone
- Signals trend continuation, not reversal

---

## Critical Implementation Details

### **For Type 1**:
1. Find relevant pivot (not just last pivot)
2. Current stroke must break ZD/ZG
3. Compare with ENTERING stroke (not any previous)
4. Must create new price extreme
5. Check divergence: MACD OR amplitude OR slope

### **For Type 2**:
1. Check exact position: `pen_idx == Type1_idx + 2`
2. Verify structure: Type1 → opposite direction → same direction
3. Allow 2% price tolerance (real markets aren't perfect)

### **For Type 3**:
1. Check exact sequence: pivot → exit stroke → pullback stroke
2. Exit stroke must be immediately after pivot (`pivot.end_pen_idx + 1`)
3. Use `>=` and `<=` for boundary (NOT strict `>` or `<`)
4. Pullback is exactly 1 stroke after exit

---

## Why This Works (Market Psychology)

### **Fractals**:
- Capture points where market participants change their minds
- High/low fractals = local consensus on value

### **Strokes**:
- Represent committed directional moves
- Filter out noise, keep significant moves

### **Pivots**:
- Zones of balance between buyers and sellers
- Market "rests" here before next move
- Historical reference points traders watch

### **Type 1 Points**:
- Catch THE moment when trend exhausts
- "Smart money" sees weakening momentum
- Beginning of regime change

### **Type 2 Points**:
- "Smart money" confirmation
- Last exit for trapped traders (sell side) or entry for late bulls (buy side)
- Lower risk than Type 1

### **Type 3 Points**:
- Strong trends respect their consolidation zones
- Failed retest = continuation signal
- Low-risk entry in established trend

---

## Practical Trading Wisdom

1. **Type 1 is the MOST IMPORTANT**
   - Catches major reversals
   - All other points derive from it
   - Requires courage (catching falling knife/fighting rally)

2. **Type 2 is the SAFEST**
   - Confirmation already in place
   - Better risk/reward than chasing
   - Miss some profit but avoid many false signals

3. **Type 3 is the MOST FREQUENT**
   - Trends have multiple Type 3 points
   - Lower reward per trade but high win rate
   - Good for systematic trading

4. **Levels Matter**
   - 5-min Type 1 < 15-min Type 1 < Daily Type 1
   - Higher timeframe signals override lower ones
   - Trade in direction of higher timeframe pivots

5. **Not All Signals Are Equal**
   - Type 1 in multi-stroke pivot > Type 1 in 3-stroke pivot
   - Type 3 after strong Type 1 > Type 3 in weak trend
   - Context is everything

---

## The Philosophy

Chan Theory doesn't predict the future. It identifies:
- **Structure**: Where we are in the market cycle
- **Exhaustion**: When current trend is ending (Type 1)
- **Confirmation**: When new trend is establishing (Type 2)
- **Continuation**: When trend is healthy and continuing (Type 3)

The beauty is in the **recursion**: These patterns repeat at every timeframe, creating a fractal structure where:
- Today's Type 1 Buy on 5-min chart might be part of a Type 3 Buy on 1-hour chart
- Understanding the multi-level structure gives edge over traders watching single timeframes

**"走势终完美"** (All trends eventually complete perfectly) - This is the core belief. By understanding structure, you can see completion before it happens.
