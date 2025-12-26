# 統計値のマージ（Merging Statistics）

## 概要

分散システムでは、各ノードで独立に計算した統計を**マージ**して全体の統計を得る必要があります。これは MapReduce、Apache Spark、分散データベースなどで頻繁に使用されるパターンです。

## なぜマージが必要か？

```
┌─────────┐     ┌─────────┐     ┌─────────┐
│ Node A  │     │ Node B  │     │ Node C  │
│ データA  │     │ データB  │     │ データC  │
│ 統計A    │     │ 統計B    │     │ 統計C    │
└────┬────┘     └────┬────┘     └────┬────┘
     │               │               │
     └───────────────┼───────────────┘
                     ▼
            ┌──────────────┐
            │ マージノード  │
            │ 全体の統計    │
            └──────────────┘
```

## 統計量のマージ可能性

| 統計量 | マージ可能性 | 複雑さ | 備考 |
|--------|------------|--------|------|
| カウント | ✅ | 簡単 | 単純に足し合わせ |
| 合計 | ✅ | 簡単 | 単純に足し合わせ |
| 平均 | ✅ | 簡単 | 加重平均 |
| 分散 | ✅ | 中程度 | 並列Welford |
| 最小/最大 | ✅ | 簡単 | min/max演算 |
| 中央値・四分位数 | ⚠️ | 複雑 | 近似が必要 |
| ヒストグラム | ✅ | 中程度 | ビンごとにマージ |
| Count-Min Sketch | ✅ | 簡単 | セルごとにmax |

## 各統計量のマージ方法

### 1. カウントと合計

最もシンプルなケース：

$$
\begin{aligned}
n_{combined} &= n_A + n_B \\
\sum_{combined} &= \sum_A + \sum_B
\end{aligned}
$$

```go
// CountSum はカウントと合計を保持
type CountSum struct {
    Count int64
    Sum   float64
}

// Merge は2つのCountSumをマージ
func (a *CountSum) Merge(b *CountSum) *CountSum {
    return &CountSum{
        Count: a.Count + b.Count,
        Sum:   a.Sum + b.Sum,
    }
}
```

### 2. 平均のマージ

加重平均として計算：

$$\mu_{combined} = \frac{n_A \cdot \mu_A + n_B \cdot \mu_B}{n_A + n_B}$$

```go
// MergeMean は2つの平均をマージ
func MergeMean(countA int64, meanA float64, countB int64, meanB float64) (int64, float64) {
    totalCount := countA + countB
    if totalCount == 0 {
        return 0, 0
    }
    
    combinedMean := (float64(countA)*meanA + float64(countB)*meanB) / float64(totalCount)
    return totalCount, combinedMean
}
```

### 3. 分散のマージ（並列Welfordアルゴリズム）

Chan et al. (1979) による並列アルゴリズム：

$$
\begin{aligned}
\delta &= \mu_B - \mu_A \\
n_{combined} &= n_A + n_B \\
\mu_{combined} &= \frac{n_A \cdot \mu_A + n_B \cdot \mu_B}{n_{combined}} \\
M_{2,combined} &= M_{2,A} + M_{2,B} + \delta^2 \cdot \frac{n_A \cdot n_B}{n_{combined}}
\end{aligned}
$$

ここで：
- $\delta$: 2つのノード間の平均の差
- $n_A, n_B$: 各ノードのデータ数
- $\mu_A, \mu_B$: 各ノードの平均
- $M_{2,A}, M_{2,B}$: 各ノードの二乗偏差の合計
- $M_2 = \sum_{i=1}^{n}(x_i - \mu)^2$（二乗偏差の合計）
- 分散は $\sigma^2 = \frac{M_2}{n}$ で得られます

```go
// Stats は統計情報を保持
type Stats struct {
    Count int64
    Mean  float64
    M2    float64  // 二乗偏差の合計
}

// Variance は分散を返す
func (s *Stats) Variance() float64 {
    if s.Count < 2 {
        return 0
    }
    return s.M2 / float64(s.Count)
}

// Merge は2つのStatsをマージ
func (a *Stats) Merge(b *Stats) *Stats {
    if a.Count == 0 {
        return b
    }
    if b.Count == 0 {
        return a
    }
    
    totalCount := a.Count + b.Count
    delta := b.Mean - a.Mean
    
    combinedMean := (float64(a.Count)*a.Mean + float64(b.Count)*b.Mean) / float64(totalCount)
    
    // 並列Welfordアルゴリズム
    combinedM2 := a.M2 + b.M2 + delta*delta*float64(a.Count)*float64(b.Count)/float64(totalCount)
    
    return &Stats{
        Count: totalCount,
        Mean:  combinedMean,
        M2:    combinedM2,
    }
}
```

### 4. 最小/最大のマージ

$$
\begin{aligned}
\min_{combined} &= \min(\min_A, \min_B) \\
\max_{combined} &= \max(\max_A, \max_B)
\end{aligned}
$$

```go
// MinMax は最小値と最大値を保持
type MinMax struct {
    Min float64
    Max float64
}

// Merge は2つのMinMaxをマージ
func (a *MinMax) Merge(b *MinMax) *MinMax {
    return &MinMax{
        Min: math.Min(a.Min, b.Min),
        Max: math.Max(a.Max, b.Max),
    }
}
```

### 5. ヒストグラムのマージ

固定幅ヒストグラムの場合、各ビンのカウントを足し合わせます：

$$H_{combined}[i] = H_A[i] + H_B[i]$$

```go
// MergeHistograms は2つのヒストグラムをマージ
func MergeHistograms(a, b *FixedWidthHistogram) *FixedWidthHistogram {
    // 同じビン構成であることを確認
    if a.min != b.min || a.max != b.max || len(a.bins) != len(b.bins) {
        panic("ヒストグラムのビン構成が一致しません")
    }
    
    result := NewFixedWidth(a.min, a.max, len(a.bins))
    for i := range a.bins {
        result.bins[i] = a.bins[i] + b.bins[i]
    }
    result.count = a.count + b.count
    
    return result
}
```

## 完全な統計サマリー構造体

すべての統計を一度にマージできる構造体：

```go
// Summary は完全な統計サマリー
type Summary struct {
    Count int64
    Sum   float64
    Mean  float64
    M2    float64  // 分散計算用
    Min   float64
    Max   float64
}

// NewSummary は新しいSummaryを作成
func NewSummary() *Summary {
    return &Summary{
        Min: math.MaxFloat64,
        Max: -math.MaxFloat64,
    }
}

// Add は値を追加
func (s *Summary) Add(value float64) {
    s.Count++
    s.Sum += value
    
    // Welfordアルゴリズム
    delta := value - s.Mean
    s.Mean += delta / float64(s.Count)
    delta2 := value - s.Mean
    s.M2 += delta * delta2
    
    // Min/Max
    if value < s.Min {
        s.Min = value
    }
    if value > s.Max {
        s.Max = value
    }
}

// Merge は2つのSummaryをマージ
func (a *Summary) Merge(b *Summary) *Summary {
    if a.Count == 0 {
        return b
    }
    if b.Count == 0 {
        return a
    }
    
    totalCount := a.Count + b.Count
    delta := b.Mean - a.Mean
    
    return &Summary{
        Count: totalCount,
        Sum:   a.Sum + b.Sum,
        Mean:  (float64(a.Count)*a.Mean + float64(b.Count)*b.Mean) / float64(totalCount),
        M2:    a.M2 + b.M2 + delta*delta*float64(a.Count)*float64(b.Count)/float64(totalCount),
        Min:   math.Min(a.Min, b.Min),
        Max:   math.Max(a.Max, b.Max),
    }
}

// Variance は分散を返す
func (s *Summary) Variance() float64 {
    if s.Count < 2 {
        return 0
    }
    return s.M2 / float64(s.Count)
}
```

## MapReduceパターン

```go
// Map: 各パーティションでSummaryを計算
func Map(data []float64) *Summary {
    s := NewSummary()
    for _, v := range data {
        s.Add(v)
    }
    return s
}

// Reduce: すべてのSummaryをマージ
func Reduce(summaries []*Summary) *Summary {
    result := NewSummary()
    for _, s := range summaries {
        result = result.Merge(s)
    }
    return result
}
```

## マージの結合法則

マージ演算は結合法則を満たすべきです：

$$(A \oplus B) \oplus C = A \oplus (B \oplus C)$$

これにより：
- 任意の順序でマージ可能
- 並列処理が可能
- 部分結果をキャッシュ可能

## 参考資料

- Chan, Tony F.; Golub, Gene H.; LeVeque, Randall J. (1979), "Updating Formulae and a Pairwise Algorithm for Computing Sample Variances"
- [MapReduce: Simplified Data Processing on Large Clusters](https://research.google/pubs/pub62/)
