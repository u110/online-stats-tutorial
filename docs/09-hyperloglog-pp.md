# HyperLogLog++（発展）

## 概要

HyperLogLog++（HLL++）は、Googleが2013年に発表したHyperLogLogの改良版です。オリジナルのHyperLogLogの問題点を解決し、より正確で効率的なカーディナリティ推定を提供します。

> **前提知識**: この記事は [08-hyperloglog.md](./08-hyperloglog.md) の内容を前提としています。

## HyperLogLogとの違い

| 機能 | HyperLogLog | HyperLogLog++ |
|------|-------------|---------------|
| ハッシュ長 | 32ビット | **64ビット** |
| 最大カーディナリティ | $\approx 10^9$ | **事実上無制限** |
| 小カーディナリティ精度 | 線形カウンティング | **経験的バイアス補正** |
| メモリ効率 | 固定 | **スパース表現** |

### 主な改良点

1. **64ビットハッシュ対応**: $2^{32}$ を超えるカーディナリティに対応
2. **バイアス補正の改善**: 経験的に求めたバイアス補正テーブル
3. **スパース表現**: 小カーディナリティ時のメモリ効率向上

## スパース表現（Sparse Representation）

### 考え方

初期状態では、ほとんどのレジスタが0です。すべてのレジスタを保持するのは無駄なので、非ゼロのレジスタのみを記録します。

```
Dense表現（通常のHLL）:
[0, 0, 3, 0, 0, 0, 5, 0, 0, 2, 0, 0, ...]  ← mバイト必要

Sparse表現（HLL++）:
[(2, 3), (6, 5), (9, 2)]  ← 非ゼロのみ記録
```

### 動作

1. **初期状態**: Sparse表現で開始
2. **要素追加**: 非ゼロエントリのみ記録
3. **閾値超過**: Dense表現に変換

### 変換の閾値

Sparse表現のサイズが Dense表現を超えたら変換：

$$\text{threshold} = \frac{6m}{8} = \frac{3m}{4}$$

ここで、各Sparseエントリは約6バイト（インデックス4バイト + 値2バイト）と仮定。

## バイアス補正

### オリジナルHLLの問題

オリジナルのHyperLogLogは、小〜中カーディナリティで系統的なバイアス（偏り）があります。

### 経験的バイアス補正

HLL++では、実験的に求めたバイアス値を使用して補正：

$$E_{corrected} = E_{raw} - \text{Bias}(E_{raw})$$

バイアス値は、各精度 $p$ に対してテーブルとして保持され、線形補間で求めます。

### 補正のフロー

```
         ┌─────────────────────┐
         │   生の推定値 E_raw   │
         └──────────┬──────────┘
                    │
         ┌──────────▼──────────┐
         │ E_raw < 5m ?        │
         └──────────┬──────────┘
          Yes       │       No
         ┌──────────▼──────────┐
         │ バイアス補正を適用   │──────────┐
         └──────────┬──────────┘          │
                    │                      │
         ┌──────────▼──────────┐          │
         │ 空レジスタ V > 0 ?  │          │
         └──────────┬──────────┘          │
          Yes       │       No            │
         ┌──────────▼──────────┐          │
         │ 線形カウンティング   │          │
         │ との調和平均        │          │
         └──────────┬──────────┘          │
                    │                      │
                    └──────────┬───────────┘
                               ▼
                    ┌──────────────────────┐
                    │     最終推定値       │
                    └──────────────────────┘
```

## 64ビットハッシュ

### なぜ64ビットが必要か？

32ビットハッシュでは、約 $2^{32} \approx 4.3 \times 10^9$ を超えるとハッシュ衝突が増加し、推定精度が低下します。

64ビットハッシュでは、事実上無制限のカーディナリティに対応できます。

### ハッシュ値の分割

```
64ビットハッシュ値:
┌─────────────┬────────────────────────────────────────────┐
│  上位pビット │              残り(64-p)ビット                │
│ (レジスタ選択) │           (先頭ゼロ数を計算)                 │
└─────────────┴────────────────────────────────────────────┘
```

## 実装例（Go）

```go
package hllpp

import (
    "hash/fnv"
    "math"
    "math/bits"
    "sort"
)

// SparseEntry はスパース表現のエントリ
type SparseEntry struct {
    Index uint32
    Rho   uint8
}

// HyperLogLogPP はHyperLogLog++データ構造
type HyperLogLogPP struct {
    p         uint8          // 精度 (4-18)
    m         uint32         // レジスタ数 = 2^p
    sparse    []SparseEntry  // スパース表現
    dense     []uint8        // 密表現
    isSparse  bool           // スパースモードか
    threshold int            // Sparse→Dense変換の閾値
    alpha     float64        // バイアス補正定数
}

// New は新しいHyperLogLog++を作成
func New(p uint8) *HyperLogLogPP {
    if p < 4 || p > 18 {
        panic("pは4から18の範囲で指定してください")
    }

    m := uint32(1) << p

    return &HyperLogLogPP{
        p:         p,
        m:         m,
        sparse:    make([]SparseEntry, 0),
        isSparse:  true,
        threshold: int(3 * m / 4), // 6m/8 = 3m/4 エントリ
        alpha:     getAlpha(m),
    }
}

// getAlpha はバイアス補正定数を返す
func getAlpha(m uint32) float64 {
    switch m {
    case 16:
        return 0.673
    case 32:
        return 0.697
    case 64:
        return 0.709
    default:
        return 0.7213 / (1.0 + 1.079/float64(m))
    }
}

// hash は要素のハッシュ値を計算（64ビット）
func (h *HyperLogLogPP) hash(item string) uint64 {
    hasher := fnv.New64a()
    hasher.Write([]byte(item))
    return hasher.Sum64()
}

// Add は要素を追加
func (h *HyperLogLogPP) Add(item string) {
    hashValue := h.hash(item)

    // 上位pビットでレジスタを選択
    j := uint32(hashValue >> (64 - h.p))

    // 残りのビットで先頭ゼロ数を計算
    w := hashValue << h.p
    rho := uint8(bits.LeadingZeros64(w)) + 1

    if h.isSparse {
        h.addSparse(j, rho)
    } else {
        h.addDense(j, rho)
    }
}

// addSparse はスパース表現に追加
func (h *HyperLogLogPP) addSparse(j uint32, rho uint8) {
    // 既存エントリを探す
    found := false
    for i := range h.sparse {
        if h.sparse[i].Index == j {
            if rho > h.sparse[i].Rho {
                h.sparse[i].Rho = rho
            }
            found = true
            break
        }
    }

    if !found {
        h.sparse = append(h.sparse, SparseEntry{Index: j, Rho: rho})
    }

    // 閾値を超えたらDenseに変換
    if len(h.sparse) > h.threshold {
        h.toDense()
    }
}

// addDense はDense表現に追加
func (h *HyperLogLogPP) addDense(j uint32, rho uint8) {
    if rho > h.dense[j] {
        h.dense[j] = rho
    }
}

// toDense はSparse表現からDense表現に変換
func (h *HyperLogLogPP) toDense() {
    h.dense = make([]uint8, h.m)

    for _, entry := range h.sparse {
        if entry.Rho > h.dense[entry.Index] {
            h.dense[entry.Index] = entry.Rho
        }
    }

    h.sparse = nil
    h.isSparse = false
}

// Estimate はカーディナリティを推定
func (h *HyperLogLogPP) Estimate() uint64 {
    if h.isSparse {
        return h.estimateSparse()
    }
    return h.estimateDense()
}

// estimateSparse はSparse表現からカーディナリティを推定
func (h *HyperLogLogPP) estimateSparse() uint64 {
    // 一時的にDense表現を構築して推定
    tempDense := make([]uint8, h.m)
    for _, entry := range h.sparse {
        tempDense[entry.Index] = entry.Rho
    }

    return h.estimateFromRegisters(tempDense)
}

// estimateDense はDense表現からカーディナリティを推定
func (h *HyperLogLogPP) estimateDense() uint64 {
    return h.estimateFromRegisters(h.dense)
}

// estimateFromRegisters はレジスタ配列から推定
func (h *HyperLogLogPP) estimateFromRegisters(registers []uint8) uint64 {
    // 調和平均の逆数を計算
    sum := 0.0
    zeroCount := uint32(0)

    for _, val := range registers {
        sum += math.Pow(2, -float64(val))
        if val == 0 {
            zeroCount++
        }
    }

    // 生の推定値
    rawEstimate := h.alpha * float64(h.m) * float64(h.m) / sum

    // バイアス補正と線形カウンティングの適用
    estimate := h.applyCorrections(rawEstimate, zeroCount)

    return uint64(estimate + 0.5)
}

// applyCorrections はバイアス補正と線形カウンティングを適用
func (h *HyperLogLogPP) applyCorrections(rawEstimate float64, zeroCount uint32) float64 {
    // 小カーディナリティの場合
    if rawEstimate < 5.0*float64(h.m) {
        // 経験的バイアス補正（簡略化版）
        rawEstimate = h.biasCorrection(rawEstimate)
    }

    // 空レジスタがある場合、線形カウンティングとの調和平均
    if zeroCount > 0 {
        linearEstimate := float64(h.m) * math.Log(float64(h.m)/float64(zeroCount))

        // 推定値が小さい場合は線形カウンティングを優先
        if linearEstimate < h.getThresholdForLinearCounting() {
            return linearEstimate
        }
    }

    return rawEstimate
}

// biasCorrection は経験的バイアス補正を適用（簡略化版）
func (h *HyperLogLogPP) biasCorrection(rawEstimate float64) float64 {
    // 実際の実装では、各精度pに対応するバイアステーブルを使用
    // ここでは簡略化した近似式を使用
    if rawEstimate < float64(h.m) {
        // 非常に小さなカーディナリティでのバイアス補正
        bias := 0.0
        switch {
        case rawEstimate < 0.5*float64(h.m):
            bias = rawEstimate * 0.1
        case rawEstimate < float64(h.m):
            bias = rawEstimate * 0.05
        default:
            bias = rawEstimate * 0.02
        }
        return rawEstimate - bias
    }
    return rawEstimate
}

// getThresholdForLinearCounting は線形カウンティングを使用する閾値を返す
func (h *HyperLogLogPP) getThresholdForLinearCounting() float64 {
    // 精度pに応じた閾値（経験的に決定された値）
    thresholds := map[uint8]float64{
        4:  10,
        5:  20,
        6:  40,
        7:  80,
        8:  220,
        9:  400,
        10: 900,
        11: 1800,
        12: 3100,
        13: 6500,
        14: 11500,
        15: 20000,
        16: 50000,
        17: 120000,
        18: 350000,
    }

    if threshold, ok := thresholds[h.p]; ok {
        return threshold
    }
    return 5.0 * float64(h.m)
}
```

## マージ（分散システム対応）

HyperLogLog++もマージ可能ですが、Sparse表現とDense表現の組み合わせを考慮する必要があります。

### マージのパターン

| パターン | 処理 |
|---------|------|
| Sparse + Sparse | エントリをマージ、閾値超過時はDenseに変換 |
| Dense + Dense | 各レジスタの max を取る |
| Sparse + Dense | SparseをDenseに変換してからマージ |

```go
// Merge は2つのHyperLogLog++をマージ
func Merge(a, b *HyperLogLogPP) *HyperLogLogPP {
    if a.p != b.p {
        panic("精度pが一致しません")
    }

    result := New(a.p)

    // 両方をDenseに変換してマージ（シンプルな実装）
    if a.isSparse {
        a.toDense()
    }
    if b.isSparse {
        b.toDense()
    }

    result.isSparse = false
    result.dense = make([]uint8, result.m)

    for i := uint32(0); i < result.m; i++ {
        if a.dense[i] > b.dense[i] {
            result.dense[i] = a.dense[i]
        } else {
            result.dense[i] = b.dense[i]
        }
    }

    return result
}

// MergeSparse は2つのSparse表現をマージ（メモリ効率版）
func MergeSparse(a, b *HyperLogLogPP) *HyperLogLogPP {
    if a.p != b.p {
        panic("精度pが一致しません")
    }

    result := New(a.p)

    // 両方のエントリをマージ
    entryMap := make(map[uint32]uint8)

    for _, entry := range a.sparse {
        entryMap[entry.Index] = entry.Rho
    }

    for _, entry := range b.sparse {
        if existing, ok := entryMap[entry.Index]; ok {
            if entry.Rho > existing {
                entryMap[entry.Index] = entry.Rho
            }
        } else {
            entryMap[entry.Index] = entry.Rho
        }
    }

    // マップからスライスに変換
    result.sparse = make([]SparseEntry, 0, len(entryMap))
    for idx, rho := range entryMap {
        result.sparse = append(result.sparse, SparseEntry{Index: idx, Rho: rho})
    }

    // ソート（オプション）
    sort.Slice(result.sparse, func(i, j int) bool {
        return result.sparse[i].Index < result.sparse[j].Index
    })

    // 閾値を超えたらDenseに変換
    if len(result.sparse) > result.threshold {
        result.toDense()
    }

    return result
}
```

### マージ時の実装注意点

> [!IMPORTANT]
> **マージを想定する場合、すべてのノードで `p`（精度）を統一する必要があります。**

オリジナルのHyperLogLogと同様に、精度が異なるとレジスタの対応関係が崩れます。

## 使用例

```go
func main() {
    // 精度14でHLL++を作成
    hll := New(14)

    // 大量のユーザーIDを追加
    for i := 0; i < 1000000; i++ {
        hll.Add(fmt.Sprintf("user%d", i))
    }

    fmt.Printf("推定ユニーク数: %d\n", hll.Estimate())
    fmt.Printf("スパースモード: %v\n", hll.isSparse)
}
```

## 精度比較

| アルゴリズム | 標準誤差 | 小カーディナリティ精度 | メモリ効率 |
|------------|---------|---------------------|-----------|
| HyperLogLog | $1.04/\sqrt{m}$ | 線形カウンティングで改善 | 固定 |
| HyperLogLog++ | $1.04/\sqrt{m}$ | **経験的補正で大幅改善** | **スパース表現** |

## 応用例

HyperLogLog++は多くの分散システムで採用されています：

| システム | コマンド/関数 | 備考 |
|---------|-------------|------|
| Redis | `PFADD`, `PFCOUNT`, `PFMERGE` | 標準機能として実装 |
| BigQuery | `APPROX_COUNT_DISTINCT()` | 大規模データ分析 |
| Presto/Trino | `approx_distinct()` | 分散クエリエンジン |
| Apache Spark | `approx_count_distinct()` | 大規模データ処理 |
| PostgreSQL | `hll` 拡張 | サードパーティ拡張 |

## 空間・時間計算量

| 操作 | 時間計算量 | 空間計算量 |
|------|-----------|-----------|
| Add（Sparse） | $O(\log n)$ | $O(n)$ ※$n$ は非ゼロレジスタ数 |
| Add（Dense） | $O(1)$ | $O(m)$ |
| Estimate | $O(m)$ | - |
| Sparse→Dense変換 | $O(n)$ | - |
| Merge（Dense） | $O(m)$ | $O(m)$ |
| Merge（Sparse） | $O(n \log n)$ | $O(n)$ |

## 参考資料

- [HyperLogLog in Practice: Algorithmic Engineering of a State of The Art Cardinality Estimation Algorithm](https://research.google/pubs/pub40671/) - Heule, Nunkesser, Hall (Google, 2013)
- [Redis HyperLogLog implementation](https://redis.io/docs/data-types/hyperloglogs/)
- [HyperLogLog - Wikipedia](https://en.wikipedia.org/wiki/HyperLogLog)
