# オンライン平均（Online Mean）

## 概要

オンライン平均は、すべてのデータを保持せずに、新しいデータが到着するたびに平均を更新する手法です。

## なぜ必要か？

**従来の方法（バッチ処理）**:

$$\bar{x} = \frac{1}{n}\sum_{i=1}^{n} x_i$$

この方法には問題があります：
- すべてのデータをメモリに保持する必要がある
- 新しいデータが来るたびに全データを再計算する必要がある
- 大量のデータでオーバーフローのリスク

## アルゴリズム

### 漸化式

新しいデータ $x_n$ が到着したとき：

$$\mu_n = \mu_{n-1} + \frac{x_n - \mu_{n-1}}{n}$$

ここで：
- $\mu_n$ は $n$ 個目までのデータの平均
- $\mu_{n-1}$ は $n-1$ 個目までのデータの平均
- $x_n$ は新しいデータ
- $n$ は現在のデータ数

### なぜこの式が成り立つのか？

$$
\begin{aligned}
\mu_n &= \frac{1}{n} \sum_{i=1}^{n} x_i \\
      &= \frac{1}{n} \left( \sum_{i=1}^{n-1} x_i + x_n \right) \\
      &= \frac{1}{n} \left( (n-1)\mu_{n-1} + x_n \right) \\
      &= \mu_{n-1} \cdot \frac{n-1}{n} + \frac{x_n}{n} \\
      &= \mu_{n-1} + \frac{x_n - \mu_{n-1}}{n}
\end{aligned}
$$

## 実装例（Go）

```go
package mean

// OnlineMean はオンライン平均を計算する構造体
type OnlineMean struct {
    count int64
    mean  float64
}

// New は新しいOnlineMeanを作成
func New() *OnlineMean {
    return &OnlineMean{}
}

// Add は新しい値を追加して平均を更新
func (m *OnlineMean) Add(value float64) {
    m.count++
    delta := value - m.mean
    m.mean += delta / float64(m.count)
}

// Mean は現在の平均を返す
func (m *OnlineMean) Mean() float64 {
    return m.mean
}

// Count は現在のデータ数を返す
func (m *OnlineMean) Count() int64 {
    return m.count
}
```

## マージ（分散システム対応）

2つの OnlineMean をマージする場合：

$$\mu_{combined} = \frac{n_A \cdot \mu_A + n_B \cdot \mu_B}{n_A + n_B}$$

ここで：
- $\mu_{combined}$: マージ後の平均
- $n_A, n_B$: 各ノードのデータ数
- $\mu_A, \mu_B$: 各ノードの平均

```go
// Merge は2つのOnlineMeanをマージ
func Merge(a, b *OnlineMean) *OnlineMean {
    if a.count == 0 {
        return b
    }
    if b.count == 0 {
        return a
    }
    
    totalCount := a.count + b.count
    combinedMean := (float64(a.count)*a.mean + float64(b.count)*b.mean) / float64(totalCount)
    
    return &OnlineMean{
        count: totalCount,
        mean:  combinedMean,
    }
}
```

## 数値的安定性

この漸化式は数値的に安定しています：
- 大きな数値の足し算によるオーバーフローを防ぐ
- 差分 $(x_n - \mu_{n-1})$ は通常小さい値になる

## 計算量

| 操作 | 時間計算量 | 空間計算量 |
|------|-----------|-----------|
| Add | $O(1)$ | $O(1)$ |
| Mean | $O(1)$ | $O(1)$ |
| Merge | $O(1)$ | $O(1)$ |

## 参考資料

- [Incremental calculation of weighted mean and variance](https://en.wikipedia.org/wiki/Algorithms_for_calculating_variance#Online_algorithm)
