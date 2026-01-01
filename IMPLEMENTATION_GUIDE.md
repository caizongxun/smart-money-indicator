# Smart Money Concepts 指標 - 完整實施指南

## 核心檢測邏輯

### 1. 鶴嘴點 (Pivot Points) 檢測

#### 檢測原理

鶴嘴點的本質是通過腿部變化來識別的：

```
腿部狀態機制：

┌─────────────────────────────────────────────────────┐
│              向上腿 (Bullish Leg)                    │
│  ─ Low[n] < Ta.Lowest(Low, n) ✓                    │
│  ─ 代表價格向下穿破前 n 根柱子的最低點             │
│  ─ 這標誌著向下動能的開始，準備形成 Swing High    │
└─────────────────────────────────────────────────────┘
                        ↓
            腿部變化被檢測 (ta.change(leg) != 0)
                        ↓
┌─────────────────────────────────────────────────────┐
│              向下腿 (Bearish Leg)                    │
│  ─ High[n] > Ta.Highest(High, n) ✓                 │
│  ─ 代表價格向上穿破前 n 根柱子的最高點             │
│  ─ 這標誌著向上動能的開始，準備形成 Swing Low     │
└─────────────────────────────────────────────────────┘
```

#### 鶴嘴點儲存

```pine
f_updatePivots(int length) =>
    currentLeg = f_getLeg(length)
    newLeg = ta.change(currentLeg) != 0
    
    if newLeg
        if ta.change(currentLeg) == +1  // 從下降腿變為上升腿
            swingLow.lastLevel = swingLow.level
            swingLow.level = low[length]          // 形成新的搖擺低點
            swingLow.barTime = time[length]
            swingLow.barIndex = bar_index[length]
            swingLow.crossed = false              // 重置穿破狀態
            swingTrend := BULLISH                 // 趨勢改為向上
        else                                      // 從上升腿變為下降腿
            swingHigh.lastLevel = swingHigh.level
            swingHigh.level = high[length]        // 形成新的搖擺高點
            swingHigh.barTime = time[length]
            swingHigh.barIndex = bar_index[length]
            swingHigh.crossed = false
            swingTrend := BEARISH
```

### 2. 結構突破 (Structure Breakout) 檢測

#### BOS (Breakout of Structure) vs CHoCH (Change of Character)

```
┌──────────────────────────────────────────────────────────┐
│  前趨勢: Bullish (swingTrend == BULLISH)                │
│  當前鶴嘴點: Swing Low                                 │
└──────────────────────────────────────────────────────────┘
                         ↓
         Close > Swing Low (向上突破)
                         ↓
         ┌─────────────────────────────┐
         │ 是否低於前一個 Swing Low?   │
         └─────────────────────────────┘
              ↙         ↖
          YES             NO
           ↓               ↓
         BOS            CHoCH
      (簡單突破)      (性質改變)
```

**BOS (Breakout of Structure)**
- 定義：價格突破鶴嘴點，但未改變趨勢方向的強度
- 觸發條件：
  ```
  swingTrend == BULLISH
  ta.crossover(close, swingLow.level)
  swingLow.lastLevel >= swingLow.level  (本次低點未低於上次)
  ```
- 含義：較弱的上升結構

**CHoCH (Change of Character)**
- 定義：價格突破鶴嘴點，並形成更強的新低/新高
- 觸發條件：
  ```
  swingTrend == BULLISH
  ta.crossover(close, swingLow.level)
  swingLow.lastLevel < swingLow.level   (本次低點更低)
  ```
- 含義：強勢的上升結構

#### 實施代碼

```pine
f_detectStructures() =>
    // 向上突破 (Bullish Breakout)
    if showSwingInput and swingTrend == BEARISH
        if ta.crossover(close, swingHigh.level) and not swingHigh.crossed
            swingHigh.crossed := true
            
            // 判斷 BOS 還是 CHoCH
            tag = (swingHigh.lastLevel < swingHigh.level) ? 'CHoCH' : 'BOS'
            
            // 根據過濾設置決定是否繪製
            if modeInput == MODE_ALL or 
               (modeInput == MODE_BOS and tag == 'BOS') or 
               (modeInput == MODE_CHOCH and tag == 'CHoCH')
                f_drawStructure(swingHigh, tag, COLOR_BULLISH, false)
            
            // 存儲訂單塊
            if showOBInput
                f_storeOrderBlock(swingHigh, BULLISH, false)
```

### 3. 訂單塊 (Order Blocks) 計算

#### OB 的定義

**Bullish Order Block (看漲訂單塊)**
- 位置：在向上腿中
- 識別方式：找出向上腿中的最低點 (Lowest Low)
- 含義：機構或大戶在此處購入，可能成為支撐
- 高度：
  - 上邊界：最低點的高線
  - 下邊界：最低點的低線

**Bearish Order Block (看跌訂單塊)**
- 位置：在向下腿中
- 識別方式：找出向下腿中的最高點 (Highest High)
- 含義：機構或大戶在此處賣出，可能成為阻力
- 高度：
  - 上邊界：最高點的高線
  - 下邊界：最高點的低線

#### 波動性過濾 (Volatility Filter)

```
判定標準：
if (High - Low) >= 2 × VolatilityMeasure
    這是高波動率柱子，需要過濾
    
過濾方法：
    正常柱子：使用 High 和 Low
    高波動柱子：交換 High 和 Low
    (避免極端波動扭曲 OB 計算)

波動率計算方式：
    ATR Method：ta.atr(200)          # 200 週期真實波幅平均
    Range Method：ta.cum(tr)/bar_index # 累積平均真實波幅
```

#### OB 儲存邏輯

```pine
f_storeOrderBlock(Pivot pvt, int bias) =>
    if bias == BEARISH
        // 看跌 OB：找最高點
        upperRange = parsedHighs.slice(pvt.barIndex, bar_index)
        maxIdx = pvt.barIndex + upperRange.indexof(upperRange.max())
        obHigh = parsedHighs.get(maxIdx)
        obLow = parsedLows.get(maxIdx)
    else
        // 看漲 OB：找最低點
        lowerRange = parsedLows.slice(pvt.barIndex, bar_index)
        minIdx = pvt.barIndex + lowerRange.indexof(lowerRange.min())
        obHigh = parsedHighs.get(minIdx)
        obLow = parsedLows.get(minIdx)
        maxIdx = minIdx
    
    ob = OrderBlock.new(obHigh, obLow, barTimes.get(maxIdx), bias, na)
    obArray.unshift(ob)  // 插入到陣列開頭
    
    if obArray.size() > 100
        obArray.pop()  // 保持最多 100 個 OB
```

#### OB 消除機制 (Mitigation)

```pine
f_deleteOrderBlocks() =>
    for i = obArray.size() - 1 to 0 by -1
        ob = obArray.get(i)
        
        // 選擇消除源
        source = obMitigationInput == SOURCE_CLOSE ? close : 
                 (ob.bias == BEARISH ? high : low)
        
        // 消除條件
        if (ob.bias == BEARISH and source > ob.high) or    // 看跌 OB 被穿破
           (ob.bias == BULLISH and source < ob.low)        // 看漲 OB 被穿破
            
            obArray.remove(i)  // 刪除該 OB
```

**消除源選擇**
- Close：使用收盤價判定
- High/Low：使用當前柱的高/低判定

### 4. 內部結構 (Internal Structure) 檢測

內部結構是指在 Swing 結構內部的更小規模結構變化。

```
時間軸對比：

Swing Structure (大尺度)
├─ Internal Structure (小尺度)
│  ├─ 內部 BOS/CHoCH
│  ├─ 內部 OB
│  └─ 更小的鶴嘴點
└─ [下一個 Swing Structure]

實施方式：
- 使用不同的鶴嘴點偵測長度（如 internal length = 5-20）
- 虛線 (dashed) 表示內部結構
- 淺色顯示內部 OB
- 僅在 Swing 結構內部顯示
```

### 5. 公平價值缺口 (Fair Value Gaps - FVG)

#### FVG 的定義

```
向上 FVG (Bullish FVG)：
  條件1：Previous Close > Previous High
  條件2：Current Low > Previous High
  含義：價格跳空向上，出現未填補的空隙
  交易機制：未來可能回落填補該缺口

向下 FVG (Bearish FVG)：
  條件1：Previous Close < Previous Low
  條件2：Current High < Previous Low
  含義：價格跳空向下，出現未填補的空隙
  交易機制：未來可能反彈填補該缺口
```

#### FVG 繪製

```pine
// FVG 由三個部分組成
┌─────────────────────────┐
│   Current High/Low      │ ← 上方盒子
├─────────────────────────┤
│   中點 (平均值)         │ ← 中間盒子
├─────────────────────────┤
│   Previous High/Low     │ ← 下方盒子
└─────────────────────────┘

透明度：70%
顏色：向上 FVG = 淺藍色，向下 FVG = 淺紅色
```

### 6. 溢價/折扣區間 (Premium/Discount Zones)

#### 定義

```
Premium Zone (溢價區)：
  ─ 上方交易區間（通常在 Daily 高點上方）
  ─ 強勢區域，資金流出
  ─ 機構傾向於在此賣出

Discount Zone (折扣區)：
  ─ 下方交易區間（通常在 Daily 低點下方）
  ─ 虛弱區域，資金流入
  ─ 機構傾向於在此買入

Equilibrium Zone (均衡區)：
  ─ 中間交易區間（Daily 高低點之間）
  ─ 平衡區域，方向性不明確
```

#### 計算方式

```
基於 Daily 框架：

高點: Daily_High[1]      # 前一個交易日的高點
低點: Daily_Low[1]       # 前一個交易日的低點

Discount Zone:
  上界 = Daily_Low[1]
  下界 = Daily_Low[1] - (Daily_High[1] - Daily_Low[1])
  
Premium Zone:
  上界 = Daily_High[1] + (Daily_High[1] - Daily_Low[1])
  下界 = Daily_High[1]
  
Equilibrium Zone:
  上界 = Daily_High[1]
  下界 = Daily_Low[1]
```

---

## 視覺化標準

### 線條樣式與寬度

| 元素 | 樣式 | 寬度 | 顏色 |
|------|------|------|------|
| Swing Bullish | 實線 | 3 | 綠色 #089981 |
| Swing Bearish | 實線 | 3 | 紅色 #f23645 |
| Internal Bullish | 虛線 | 2 | 綠色 #089981 |
| Internal Bearish | 虛線 | 2 | 紅色 #f23645 |
| Pivot Reference | 點線 | 2 | 灰色 #878b94 |

### 訂單塊顏色

| 類型 | 填充色 | 透明度 | 邊框色 | 寬度 |
|------|--------|--------|--------|------|
| Swing Bull OB | 藍色 #1848cc | 75% | 藍色 | 2 |
| Swing Bear OB | 紅色 #b22833 | 75% | 紅色 | 2 |
| Internal Bull OB | 綠色 #089981 | 75% | 綠色 | 2 |
| Internal Bear OB | 紅色 #f23645 | 75% | 紅色 | 2 |

### 標籤放置

```
BOS/CHoCH 標籤：
  ─ 水平位置：在結構線的中點
  ─ 垂直位置：鶴嘴點價格位置
  ─ 對齐：居中
  ─ 大小：small
  ─ 顏色：與結構線相同
```

---

## 測試清單

### 功能測試

- [ ] Swing 鶴嘴點檢測
  - [ ] Swing High 正確識別
  - [ ] Swing Low 正確識別
  - [ ] 腿部變化被正確檢測

- [ ] 結構檢測
  - [ ] BOS 正確標記
  - [ ] CHoCH 正確標記
  - [ ] 標籤位置準確

- [ ] 訂單塊
  - [ ] 計算位置正確
  - [ ] 消除機制有效
  - [ ] 最多 8-10 個顯示
  - [ ] 邊框清晰可見

- [ ] 波動率過濾
  - [ ] 高波動率柱被正確過濾
  - [ ] OB 計算不受影響

### 視覺化測試

- [ ] 線條顏色正確
- [ ] 線條寬度明顯
- [ ] OB 透明度合適
- [ ] 標籤不重疊
- [ ] 不同時間框架清晰可見

### 性能測試

- [ ] 標籤數量 < 500
- [ ] 線條數量 < 500
- [ ] 盒子數量 < 500
- 指標加載時間 < 2 秒
- 圖表滾動順暢

---

## 部署步驟

1. 複製完整的 Pine Script 代碼
2. 在 TradingView 上新建指標
3. 粘貼代碼
4. 調整輸入參數以符合交易風格
5. 在歷史數據上進行回測
6. 對比 TV 官方版本的效果
7. 微調顏色和線條寬度
8. 標記為收藏

---

## 常見問題

**Q：為什麼有時候看不到 Internal Structure？**
A：Internal Structure 只在 Swing Structure 內部顯示，且需要啟用"Show Internal Structure"選項。

**Q：Order Block 為什麼消失？**
A：這是正常的"Mitigation"機制。當價格穿過 OB 的邊界時，該 OB 會被刪除。

**Q：如何區分 BOS 和 CHoCH？**
A：在設定中使用"Structure Filter"選項。或根據新鶴嘴點是否創造新高/新低判斷。

**Q：能支持多時間框架嗎？**
A：當前版本僅支持單一時間框架。可通過 request.security() 擴展。
