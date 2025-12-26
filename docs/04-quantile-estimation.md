# 四分位数・パーセンタイル推定（Quantile Estimation）

## 概要

オンラインでの四分位数（Q1, Q2, Q3）やパーセンタイル計算は、**厳密解ではなく推定値**になります。これは、正確な計算にはすべてのデータのソートが必要だからです。

## なぜ推定値になるのか？

正確なパーセンタイル計算には：

$$\text{Percentile}_p = x_{(\lceil p \cdot n \rceil)}$$

- すべてのデータをソートする必要がある: $O(n \log n)$
- すべてのデータをメモリに保持する必要がある: $O(n)$

ストリーミング環境では、これは実現不可能です。そのため、限られたメモリで「十分に正確な」推定を行います。

## 主なアルゴリズム

### 1. P² アルゴリズム（Piecewise-Parabolic）

5つのマーカーを使用して、任意のパーセンタイルを推定します。

#### 考え方

5つのマーカー位置 $q_0, q_1, q_2, q_3, q_4$ を維持：
- $q_0$: 最小値
- $q_2$: 推定したいパーセンタイル
- $q_4$: 最大値

新しいデータが来るたびに、マーカーの理想位置と実際位置のずれを放物線補間で調整します。

#### 更新式

マーカー $q_i$ の理想位置 $n'_i$:

$$n'_i = n'_{i,prev} + \Delta_i$$

放物線補間による調整：

$$q'_i = q_i + \frac{d}{n_{i+1} - n_{i-1}} \left[ (n_i - n_{i-1} + d)\frac{q_{i+1} - q_i}{n_{i+1} - n_i} + (n_{i+1} - n_i - d)\frac{q_i - q_{i-1}}{n_i - n_{i-1}} \right]$$

### 2. t-digest

t-digestは、データ分布を「クラスタ」の集合として表現する手法です。

#### 基本概念

$$T = \{(c_1, m_1), (c_2, m_2), \ldots, (c_k, m_k)\}$$

ここで：
- $c_i$ はクラスタの重心（centroid）
- $m_i$ はクラスタの重み（データ点の数）

#### スケール関数

t-digestの鍵となるのは、スケール関数 $k(q)$ です：

$$k(q) = \frac{\delta}{2\pi} \cdot \arcsin(2q - 1)$$

この関数により、分布の両端（尾部）では小さなクラスタを維持し、中央部分では大きなクラスタを許容します。これにより、極端なパーセンタイル（1%, 99%など）の精度が向上します。

#### 実装例（Go）

```go
package quantile

import (
    "sort"
)

// Centroid はt-digestのクラスタを表す
type Centroid struct {
    Mean  float64
    Count int64
}

// TDigest はt-digestデータ構造
type TDigest struct {
    compression float64
    centroids   []Centroid
    count       int64
    buffer      []float64
    bufferSize  int
}

// NewTDigest は新しいt-digestを作成
func NewTDigest(compression float64) *TDigest {
    return &TDigest{
        compression: compression,
        centroids:   make([]Centroid, 0),
        buffer:      make([]float64, 0, 1000),
        bufferSize:  1000,
    }
}

// Add は値を追加
func (t *TDigest) Add(value float64) {
    t.buffer = append(t.buffer, value)
    t.count++
    
    if len(t.buffer) >= t.bufferSize {
        t.compress()
    }
}

// compress はバッファをクラスタにマージ
func (t *TDigest) compress() {
    sort.Float64s(t.buffer)
    
    for _, v := range t.buffer {
        t.addToCluster(v)
    }
    
    t.buffer = t.buffer[:0]
}

// Quantile はパーセンタイルを返す
func (t *TDigest) Quantile(q float64) float64 {
    if len(t.buffer) > 0 {
        t.compress()
    }
    
    if len(t.centroids) == 0 {
        return 0
    }
    
    // 累積カウントを計算してパーセンタイルを補間
    targetCount := q * float64(t.count)
    cumulative := int64(0)
    
    for i, c := range t.centroids {
        if cumulative+c.Count > int64(targetCount) {
            // このクラスタ内で補間
            if i == 0 {
                return c.Mean
            }
            prev := t.centroids[i-1]
            ratio := (targetCount - float64(cumulative)) / float64(c.Count)
            return prev.Mean + ratio*(c.Mean-prev.Mean)
        }
        cumulative += c.Count
    }
    
    return t.centroids[len(t.centroids)-1].Mean
}
```

### 3. GK アルゴリズム（Greenwald-Khanna）

$\varepsilon$-近似を保証するアルゴリズムです。

#### 保証

任意のクエリ $\phi$ に対して、返される値のランクは：

$$\phi \cdot n - \varepsilon \cdot n \leq \text{rank} \leq \phi \cdot n + \varepsilon \cdot n$$

#### 空間計算量

$$O\left(\frac{1}{\varepsilon} \log(\varepsilon n)\right)$$

## 比較

| アルゴリズム | 空間計算量 | 精度 | マージ可能 |
|-------------|-----------|------|-----------|
| P² | $O(1)$ | 中程度 | ❌ |
| t-digest | $O(\delta)$ | 高（特に尾部） | ✅ |
| GK | $O(\frac{1}{\varepsilon} \log n)$ | $\varepsilon$-近似保証 | ✅ |

## t-digestのマージ

t-digestはマージ可能で、分散システムに適しています：

$$T_{combined} = \text{Compress}(T_A \cup T_B)$$

```go
// Merge は2つのt-digestをマージ
func Merge(a, b *TDigest) *TDigest {
    result := NewTDigest(a.compression)
    
    // 両方のクラスタを結合
    allCentroids := append(a.centroids, b.centroids...)
    sort.Slice(allCentroids, func(i, j int) bool {
        return allCentroids[i].Mean < allCentroids[j].Mean
    })
    
    // 再圧縮
    for _, c := range allCentroids {
        result.addCentroid(c)
    }
    
    result.count = a.count + b.count
    return result
}
```

## 四分位数の計算

t-digestを使用した四分位数の取得：

```go
// Quartiles は四分位数を返す
func (t *TDigest) Quartiles() (q1, q2, q3 float64) {
    q1 = t.Quantile(0.25)  // 第1四分位数
    q2 = t.Quantile(0.50)  // 中央値
    q3 = t.Quantile(0.75)  // 第3四分位数
    return
}

// IQR は四分位範囲を返す
func (t *TDigest) IQR() float64 {
    _, _, q3 := t.Quartiles()
    q1, _, _ := t.Quartiles()
    return q3 - q1
}
```

## 参考資料

- [The P² Algorithm for Dynamic Calculation of Quantiles](https://www.cs.wustl.edu/~jain/papers/ftp/psqr.pdf)
- [Computing Extremely Accurate Quantiles Using t-Digests](https://github.com/tdunning/t-digest/blob/main/docs/t-digest-paper/histo.pdf)
- [Space-Efficient Online Computation of Quantile Summaries (GK)](https://dl.acm.org/doi/10.1145/375663.375670)
