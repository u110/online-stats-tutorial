# オンライン分散（Welford's Algorithm）

## 概要

Welfordのアルゴリズムは、オンラインで分散を計算するための数値的に安定したアルゴリズムです。1962年にB.P. Welfordによって提案されました。

## なぜ Welford が必要か？

### ナイーブな方法の問題

従来の2パス法：

$$\sigma^2 = \frac{\sum x_i^2}{n} - \left(\frac{\sum x_i}{n}\right)^2$$

この方法の問題点：
1. **桁落ち誤差**: 大きな値同士の引き算で精度が落ちる
2. **2パス必要**: 平均を計算してから分散を計算
3. **オンライン処理不可**: すべてのデータが必要

### 例：ナイーブ法の失敗

データ: $[10000000.0, 10000001.0, 10000002.0]$

$$
\begin{aligned}
\sum x_i^2 &= 300000030000000005.0 \\
\frac{(\sum x_i)^2}{n} &= 300000030000000003.0 \\
\text{差} &= 2.0 \rightarrow \text{実際の分散} = \frac{2}{3} \approx 0.667
\end{aligned}
$$

浮動小数点誤差により、計算結果が不正確になる可能性があります。

## Welford のアルゴリズム

### 漸化式

$$
\begin{aligned}
\mu_n &= \mu_{n-1} + \frac{x_n - \mu_{n-1}}{n} \\
M_{2,n} &= M_{2,n-1} + (x_n - \mu_{n-1})(x_n - \mu_n)
\end{aligned}
$$

分散の計算：

$$
\begin{aligned}
\sigma^2 &= \frac{M_2}{n} \quad \text{（母分散）} \\
s^2 &= \frac{M_2}{n-1} \quad \text{（標本分散、不偏分散）}
\end{aligned}
$$

ここで：
- $\mu_n$: $n$ 個目までのデータの平均
- $\mu_{n-1}$: $n-1$ 個目までのデータの平均
- $x_n$: 新しく追加するデータ
- $M_2$: 二乗偏差の合計（sum of squared differences）= $\sum_{i=1}^{n}(x_i - \mu_n)^2$
- $\sigma^2$: 母分散（population variance）
- $s^2$: 標本分散（sample variance、不偏分散）

> **ポイント**: $(x_n - \mu_{n-1})$ と $(x_n - \mu_n)$ の両方を使うことで数値的安定性を確保しています。

## 実装例（Go）

```go
package variance

import "math"

// WelfordVariance はWelfordのアルゴリズムで分散を計算
type WelfordVariance struct {
    count int64
    mean  float64
    m2    float64  // 二乗偏差の合計
}

// New は新しいWelfordVarianceを作成
func New() *WelfordVariance {
    return &WelfordVariance{}
}

// Add は新しい値を追加
func (w *WelfordVariance) Add(value float64) {
    w.count++
    delta := value - w.mean
    w.mean += delta / float64(w.count)
    delta2 := value - w.mean
    w.m2 += delta * delta2
}

// Mean は現在の平均を返す
func (w *WelfordVariance) Mean() float64 {
    return w.mean
}

// Variance は母分散を返す
func (w *WelfordVariance) Variance() float64 {
    if w.count < 2 {
        return 0
    }
    return w.m2 / float64(w.count)
}

// SampleVariance は標本分散を返す（不偏分散）
func (w *WelfordVariance) SampleVariance() float64 {
    if w.count < 2 {
        return 0
    }
    return w.m2 / float64(w.count-1)
}

// StdDev は標準偏差を返す
func (w *WelfordVariance) StdDev() float64 {
    return math.Sqrt(w.Variance())
}

// Count は現在のデータ数を返す
func (w *WelfordVariance) Count() int64 {
    return w.count
}
```

## 並列マージ（Chan et al., 1979）

分散システムで2つの WelfordVariance をマージ：

$$
\begin{aligned}
\delta &= \mu_B - \mu_A \\
\mu_{combined} &= \frac{n_A \cdot \mu_A + n_B \cdot \mu_B}{n_A + n_B} \\
M_{2,combined} &= M_{2,A} + M_{2,B} + \delta^2 \cdot \frac{n_A \cdot n_B}{n_A + n_B}
\end{aligned}
$$

ここで：
- $\delta$: 2つのノード間の平均の差
- $n_A, n_B$: 各ノードのデータ数
- $\mu_A, \mu_B$: 各ノードの平均
- $M_{2,A}, M_{2,B}$: 各ノードの二乗偏差の合計
- $M_{2,combined}$: マージ後の二乗偏差の合計

```go
// Merge は2つのWelfordVarianceをマージ
func Merge(a, b *WelfordVariance) *WelfordVariance {
    if a.count == 0 {
        return b
    }
    if b.count == 0 {
        return a
    }
    
    totalCount := a.count + b.count
    delta := b.mean - a.mean
    
    combinedMean := (float64(a.count)*a.mean + float64(b.count)*b.mean) / float64(totalCount)
    
    // 並列アルゴリズム: M2を結合
    combinedM2 := a.m2 + b.m2 + delta*delta*float64(a.count)*float64(b.count)/float64(totalCount)
    
    return &WelfordVariance{
        count: totalCount,
        mean:  combinedMean,
        m2:    combinedM2,
    }
}
```

## なぜこの式が安定しているのか？

1. **差分ベース**: 大きな数値の足し算を避ける
2. **順序独立**: データの到着順序に依存しない
3. **累積的**: 中間状態をマージ可能

## 計算量

| 操作 | 時間計算量 | 空間計算量 |
|------|-----------|-----------|
| Add | $O(1)$ | $O(1)$ |
| Variance | $O(1)$ | $O(1)$ |
| Merge | $O(1)$ | $O(1)$ |

## 参考資料

- [Welford's online algorithm - Wikipedia](https://en.wikipedia.org/wiki/Algorithms_for_calculating_variance#Welford's_online_algorithm)
- [Parallel algorithm - Wikipedia](https://en.wikipedia.org/wiki/Algorithms_for_calculating_variance#Parallel_algorithm)
- Chan, Tony F.; Golub, Gene H.; LeVeque, Randall J. (1979), "Updating Formulae and a Pairwise Algorithm for Computing Sample Variances"
