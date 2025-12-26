# Count-Min Sketch（発展）

## 概要

Count-Min Sketch（CMS）は、ストリーミングデータにおける要素の頻度を推定するための確率的データ構造です。2003年にCormode と Muthukrishnanによって提案されました。

## 特徴

- **固定メモリ**: データ量に関係なく一定のメモリ使用量
- **高速**: 追加・クエリともに $O(d)$（$d$ はハッシュ関数の数）
- **過大推定のみ**: 頻度を過小推定することはない
- **マージ可能**: 分散システムで使用可能

## データ構造

Count-Min Sketchは、2次元配列とハッシュ関数の集合で構成されます：

$$CMS = \begin{pmatrix} c_{1,1} & c_{1,2} & \cdots & c_{1,w} \\ c_{2,1} & c_{2,2} & \cdots & c_{2,w} \\ \vdots & \vdots & \ddots & \vdots \\ c_{d,1} & c_{d,2} & \cdots & c_{d,w} \end{pmatrix}$$

ここで：
- $w$ は配列の幅（カラム数）
- $d$ はハッシュ関数の数（行数）
- $h_1, h_2, \ldots, h_d$ は独立なハッシュ関数

## アルゴリズム

### 追加（Add）

要素 $x$ をカウント $c$ だけ追加：

$$\forall i \in [1, d]: \quad count[i][h_i(x)] \mathrel{+}= c$$

> **記号の補足**: $\forall$ は「すべての〜について」を意味する全称記号です。
> この式は「$i = 1$ から $d$ までの**すべての** $i$ について、`count[i][h_i(x)]` に `c` を加える」という意味です。
> つまり、要素を追加するときは、$d$ 個すべてのハッシュ関数に対応するカウンタを更新します。

### クエリ（Query）

要素 $x$ の頻度を推定：

$$\hat{f}(x) = \min_{i \in [1, d]} count[i][h_i(x)]$$

**重要**: 最小値を取ることで、ハッシュ衝突による過大推定を抑制します。

## パラメータの選び方

誤差 $\varepsilon$ と信頼度 $\delta$ を指定：

$$
\begin{aligned}
w &= \left\lceil \frac{e}{\varepsilon} \right\rceil \\
d &= \left\lceil \ln\left(\frac{1}{\delta}\right) \right\rceil
\end{aligned}
$$

ここで $e \approx 2.718$ は自然対数の底です。

### 誤差の保証

$$P\left[\hat{f}(x) \leq f(x) + \varepsilon \cdot N\right] \geq 1 - \delta$$

- $\hat{f}(x)$: 推定頻度
- $f(x)$: 真の頻度
- $N$: 全要素の合計カウント

## 実装例（Go）

```go
package cms

import (
    "hash/fnv"
    "math"
)

// CountMinSketch はCount-Min Sketchデータ構造
type CountMinSketch struct {
    width  int
    depth  int
    count  [][]int64
    seeds  []uint64
    total  int64
}

// New は新しいCount-Min Sketchを作成
// epsilon: 誤差率 (例: 0.01 = 1%)
// delta: 失敗確率 (例: 0.01 = 1%)
func New(epsilon, delta float64) *CountMinSketch {
    width := int(math.Ceil(math.E / epsilon))
    depth := int(math.Ceil(math.Log(1.0 / delta)))
    
    count := make([][]int64, depth)
    seeds := make([]uint64, depth)
    
    for i := 0; i < depth; i++ {
        count[i] = make([]int64, width)
        seeds[i] = uint64(i * 12345) // シンプルなシード生成
    }
    
    return &CountMinSketch{
        width: width,
        depth: depth,
        count: count,
        seeds: seeds,
    }
}

// hash は要素のハッシュ値を計算
func (c *CountMinSketch) hash(item string, seed uint64) int {
    h := fnv.New64a()
    h.Write([]byte(item))
    hash := h.Sum64() ^ seed
    return int(hash % uint64(c.width))
}

// Add は要素を追加
func (c *CountMinSketch) Add(item string, count int64) {
    for i := 0; i < c.depth; i++ {
        idx := c.hash(item, c.seeds[i])
        c.count[i][idx] += count
    }
    c.total += count
}

// AddOne は要素を1つ追加
func (c *CountMinSketch) AddOne(item string) {
    c.Add(item, 1)
}

// Estimate は要素の頻度を推定
func (c *CountMinSketch) Estimate(item string) int64 {
    minCount := int64(math.MaxInt64)
    
    for i := 0; i < c.depth; i++ {
        idx := c.hash(item, c.seeds[i])
        if c.count[i][idx] < minCount {
            minCount = c.count[i][idx]
        }
    }
    
    return minCount
}

// Total は全要素の合計カウントを返す
func (c *CountMinSketch) Total() int64 {
    return c.total
}
```

## マージ（分散システム対応）

Count-Min Sketchはマージ可能です：

$$CMS_{combined}[i][j] = CMS_A[i][j] + CMS_B[i][j]$$

```go
// Merge は2つのCMSをマージ
func Merge(a, b *CountMinSketch) *CountMinSketch {
    if a.width != b.width || a.depth != b.depth {
        panic("CMSの次元が一致しません")
    }
    
    result := &CountMinSketch{
        width: a.width,
        depth: a.depth,
        count: make([][]int64, a.depth),
        seeds: a.seeds, // 同じシードを使用
        total: a.total + b.total,
    }
    
    for i := 0; i < a.depth; i++ {
        result.count[i] = make([]int64, a.width)
        for j := 0; j < a.width; j++ {
            result.count[i][j] = a.count[i][j] + b.count[i][j]
        }
    }
    
    return result
}
```

### マージ時の実装注意点

> [!IMPORTANT]
> **マージを想定する場合、すべてのノードで以下のパラメータを統一する必要があります：**
> - **seed（シード）**: 必ず共通のものを使用
> - **width（幅）**: 同じ値
> - **depth（深さ）**: 同じ値

#### なぜシードを共通にする必要があるのか？

マージは「同じ位置のカウンタを足し合わせる」ことで動作します。シードが異なると、同じ要素が異なる位置にハッシュされ、マージ結果が無意味になります：

```
❌ シードが異なる場合:
Node A (seed=123):  "apple" → 位置 5
Node B (seed=456):  "apple" → 位置 12  ← 位置がずれる！

✅ シードが共通の場合:
Node A (seed=123):  "apple" → 位置 5
Node B (seed=123):  "apple" → 位置 5   ← 同じ位置！
```

#### 推奨される実装パターン

```go
// グローバルまたは設定ファイルで共通のシードを定義
var GlobalCMSConfig = struct {
    Epsilon float64
    Delta   float64
    Seeds   []uint64
}{
    Epsilon: 0.01,
    Delta:   0.01,
    Seeds:   []uint64{12345, 67890, 11111, 22222, 33333},
}

// 各ノードで同じ設定を使用
func NewWithConfig(config CMSConfig) *CountMinSketch {
    cms := &CountMinSketch{
        width: int(math.Ceil(math.E / config.Epsilon)),
        depth: len(config.Seeds),
        seeds: config.Seeds,  // 共通シードを使用
    }
    // ...
    return cms
}
```

## 使用例

```go
func main() {
    // 1%の誤差、1%の失敗確率
    cms := New(0.01, 0.01)
    
    // ストリーミングデータを追加
    words := []string{"apple", "banana", "apple", "cherry", "apple", "banana"}
    for _, word := range words {
        cms.AddOne(word)
    }
    
    // 頻度を推定
    fmt.Printf("apple: %d (真の値: 3)\n", cms.Estimate("apple"))
    fmt.Printf("banana: %d (真の値: 2)\n", cms.Estimate("banana"))
    fmt.Printf("cherry: %d (真の値: 1)\n", cms.Estimate("cherry"))
    fmt.Printf("grape: %d (真の値: 0)\n", cms.Estimate("grape"))
}
```

## 変種

### Conservative Update

過大推定をさらに抑制する手法：

```go
// AddConservative は保守的な更新
func (c *CountMinSketch) AddConservative(item string, count int64) {
    // 現在の推定値を取得
    currentMin := c.Estimate(item)
    newValue := currentMin + count
    
    // 各カウンタを必要な分だけ更新
    for i := 0; i < c.depth; i++ {
        idx := c.hash(item, c.seeds[i])
        if c.count[i][idx] < newValue {
            c.count[i][idx] = newValue
        }
    }
    c.total += count
}
```

### Count-Mean-Min Sketch

ノイズ推定を行い、より正確な値を返す：

$$\hat{f}_{CMM}(x) = \text{median}_{i} \left( count[i][h_i(x)] - \frac{N - count[i][h_i(x)]}{w - 1} \right)$$

## 比較：他の確率的データ構造

| データ構造 | 用途 | 過小推定 | 過大推定 | マージ |
|-----------|------|---------|---------|--------|
| Count-Min Sketch | 頻度推定 | ❌ なし | ✅ あり | ✅ |
| Bloom Filter | 存在確認 | ❌ なし | ✅ あり | ✅ |
| HyperLogLog | カーディナリティ | ⚠️ 近似 | ⚠️ 近似 | ✅ |

## 応用例

| 応用 | 説明 |
|------|------|
| ネットワークトラフィック分析 | IPアドレスごとのパケット数 |
| Heavy Hitters検出 | 高頻度アイテムの検出 |
| NLP | 単語頻度のカウント |
| データベース | クエリ最適化のための統計 |
| 広告システム | インプレッション/クリックのカウント |

## 空間・時間計算量

| 操作 | 時間計算量 | 空間計算量 |
|------|-----------|-----------|
| Add | $O(d)$ | - |
| Estimate | $O(d)$ | - |
| 構造体全体 | - | $O(w \cdot d) = O\left(\frac{1}{\varepsilon} \log \frac{1}{\delta}\right)$ |

## 参考資料

- [An Improved Data Stream Summary: The Count-Min Sketch and its Applications](https://www.cs.princeton.edu/courses/archive/spring04/cos598B/bib/CormijdeM-JCSS.pdf) - Cormode & Muthukrishnan (2005)
- [Count-Min Sketch - Wikipedia](https://en.wikipedia.org/wiki/Count%E2%80%93min_sketch)
