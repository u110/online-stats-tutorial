# オンラインヒストグラム（Online Histogram）

## 概要

オンラインヒストグラムは、ストリーミングデータの分布を推定するための手法です。固定されたメモリ量で、データの到着順序に依存せずに分布を近似します。

## 基本概念

ヒストグラムは、データを複数の「ビン」（区間）に分割し、各ビンに含まれるデータの数を記録します。

$$H = \{(b_1, c_1), (b_2, c_2), \ldots, (b_k, c_k)\}$$

ここで：
- $b_i$ はビンの境界または中心
- $c_i$ はビンに含まれるデータの数
- $k$ はビンの総数

## 固定幅ヒストグラム

最もシンプルな形式です。

### パラメータ

$$
\begin{aligned}
\text{ビン幅}: w &= \frac{max - min}{k} \\
\text{ビンのインデックス}: i &= \left\lfloor \frac{x - min}{w} \right\rfloor
\end{aligned}
$$

ここで：
- $w$: 各ビンの幅
- $min$: ヒストグラムがカバーするデータ範囲の下限
- $max$: ヒストグラムがカバーするデータ範囲の上限
- $k$: ビンの総数
- $x$: 新しく追加するデータ
- $i$: データが属するビンのインデックス（0始まり）
- $\lfloor \cdot \rfloor$: 床関数（小数点以下を切り捨て）

### 実装例（Go）

```go
package histogram

// FixedWidthHistogram は固定幅ヒストグラム
type FixedWidthHistogram struct {
    min      float64
    max      float64
    binWidth float64
    bins     []int64
    count    int64
}

// NewFixedWidth は固定幅ヒストグラムを作成
func NewFixedWidth(min, max float64, numBins int) *FixedWidthHistogram {
    binWidth := (max - min) / float64(numBins)
    return &FixedWidthHistogram{
        min:      min,
        max:      max,
        binWidth: binWidth,
        bins:     make([]int64, numBins),
    }
}

// Add は値を追加
func (h *FixedWidthHistogram) Add(value float64) {
    if value < h.min || value >= h.max {
        return // 範囲外は無視
    }
    
    idx := int((value - h.min) / h.binWidth)
    if idx >= len(h.bins) {
        idx = len(h.bins) - 1
    }
    h.bins[idx]++
    h.count++
}

// Bins はビンのカウントを返す
func (h *FixedWidthHistogram) Bins() []int64 {
    return h.bins
}
```

## 適応型ヒストグラム

データの範囲が事前にわからない場合に使用します。

### アルゴリズム

1. ビンが溢れたら、隣接する2つのビンをマージ
2. 新しいデータ点で分割が必要なら分割

### 実装例（Go）

```go
// AdaptiveHistogram は適応型ヒストグラム
type AdaptiveHistogram struct {
    maxBins int
    bins    []Bin
    count   int64
}

type Bin struct {
    Center float64
    Count  int64
}

// NewAdaptive は適応型ヒストグラムを作成
func NewAdaptive(maxBins int) *AdaptiveHistogram {
    return &AdaptiveHistogram{
        maxBins: maxBins,
        bins:    make([]Bin, 0, maxBins),
    }
}

// Add は値を追加
func (h *AdaptiveHistogram) Add(value float64) {
    h.count++
    
    // 挿入位置を二分探索
    idx := h.findInsertPosition(value)
    
    // 新しいビンを挿入
    newBin := Bin{Center: value, Count: 1}
    h.bins = append(h.bins[:idx], append([]Bin{newBin}, h.bins[idx:]...)...)
    
    // ビン数が上限を超えたらマージ
    for len(h.bins) > h.maxBins {
        h.mergeClosestBins()
    }
}

// mergeClosestBins は最も近い2つのビンをマージ
func (h *AdaptiveHistogram) mergeClosestBins() {
    minDist := math.MaxFloat64
    mergeIdx := 0
    
    for i := 0; i < len(h.bins)-1; i++ {
        dist := h.bins[i+1].Center - h.bins[i].Center
        if dist < minDist {
            minDist = dist
            mergeIdx = i
        }
    }
    
    // 加重平均で新しい中心を計算
    b1, b2 := h.bins[mergeIdx], h.bins[mergeIdx+1]
    totalCount := b1.Count + b2.Count
    newCenter := (b1.Center*float64(b1.Count) + b2.Center*float64(b2.Count)) / float64(totalCount)
    
    h.bins[mergeIdx] = Bin{Center: newCenter, Count: totalCount}
    h.bins = append(h.bins[:mergeIdx+1], h.bins[mergeIdx+2:]...)
}
```

## マージ（分散システム対応）

2つのヒストグラムをマージ：

$$H_{combined} = H_A \cup H_B \text{、必要に応じてビンをマージ}$$

```go
// Merge は2つのヒストグラムをマージ
func Merge(a, b *AdaptiveHistogram) *AdaptiveHistogram {
    result := NewAdaptive(a.maxBins)
    
    // 両方のビンを結合
    allBins := append(a.bins, b.bins...)
    sort.Slice(allBins, func(i, j int) bool {
        return allBins[i].Center < allBins[j].Center
    })
    
    result.bins = allBins
    result.count = a.count + b.count
    
    // ビン数を制限
    for len(result.bins) > result.maxBins {
        result.mergeClosestBins()
    }
    
    return result
}
```

### マージ時の実装注意点

> [!IMPORTANT]
> **固定幅ヒストグラムをマージする場合、すべてのノードで以下のパラメータを統一する必要があります：**
> - **min（データ範囲の下限）**: ヒストグラムが受け付ける最小値
> - **max（データ範囲の上限）**: ヒストグラムが受け付ける最大値
> - **numBins（ビン数）**: 同じ値

#### なぜパラメータを統一する必要があるのか？

ビンの境界が異なると、同じ値が異なるビンに振り分けられてしまいます：

```
❌ パラメータが異なる場合:
Node A (min=0, max=100, bins=10):  値50 → ビン5
Node B (min=0, max=200, bins=10):  値50 → ビン2  ← ビンがずれる！

✅ パラメータが共通の場合:
Node A (min=0, max=100, bins=10):  値50 → ビン5
Node B (min=0, max=100, bins=10):  値50 → ビン5   ← 同じビン！
```

#### 範囲外の値の扱い

分散システムでは、事前にデータの範囲がわからないことがあります。対策：

1. **十分に広い範囲を設定**: 想定される最大・最小値を余裕を持って設定
2. **範囲外カウンタを追加**: 範囲外の値を別途カウント
3. **適応型ヒストグラムを使用**: maxBinsを統一すればマージ可能

## パーセンタイルの推定

ヒストグラムからパーセンタイルを推定：

$$
\text{Percentile}(p) \approx b_i + w \cdot \frac{p \cdot n - \sum_{j=1}^{i-1} c_j}{c_i}
$$

ここで：
- $p$ は求めるパーセンタイル（0〜1）
- $n$ は総データ数
- $b_i$ はパーセンタイルを含むビンの左境界

## 計算量

| 操作 | 時間計算量 | 空間計算量 |
|------|-----------|-----------|
| Add（固定幅） | $O(1)$ | $O(k)$ |
| Add（適応型） | $O(k)$ | $O(k)$ |
| Merge | $O(k \log k)$ | $O(k)$ |

## 参考資料

- [Streaming Algorithms for Computing Histograms](https://dl.acm.org/doi/10.1145/1142473.1142501)
- [A Streaming Parallel Decision Tree Algorithm](https://www.jmlr.org/papers/v11/ben-haim10a.html)
