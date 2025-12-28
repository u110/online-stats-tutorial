# HyperLogLog（発展）

## 概要

HyperLogLog（HLL）は、ストリーミングデータにおける**カーディナリティ**（ユニーク要素数）を推定するための確率的データ構造です。2007年にFlajolet et al.によって提案されました。

## 特徴

- **固定メモリ**: データ量に関係なく一定のメモリ使用量
- **高速**: 追加は $O(1)$
- **近似値**: 標準誤差 $\approx 1.04/\sqrt{m}$（$m$ はレジスタ数）
- **マージ可能**: 分散システムで使用可能

## 直感的理解

### コイン投げのアナロジー

コインを繰り返し投げて、連続で表が出た最大回数を記録することを考えます。

- 1回連続で表 → 約50%の確率
- 2回連続で表 → 約25%の確率
- 3回連続で表 → 約12.5%の確率
- $k$ 回連続で表 → 約 $1/2^k$ の確率

もし最大で5回連続の表を観測したなら、おそらく $2^5 = 32$ 回程度コインを投げたと推測できます。

### ハッシュ値と先頭ゼロビット

HyperLogLogは、この考え方をハッシュ値に適用します：

```
ハッシュ値の例:
  0001 0110 1010 ...  → 先頭ゼロ数 = 3
  0000 0001 1101 ...  → 先頭ゼロ数 = 6
  1010 0011 0101 ...  → 先頭ゼロ数 = 0
```

先頭ゼロが $k$ 個あるハッシュ値を観測する確率は $1/2^{k+1}$ です。したがって、観測した最大先頭ゼロ数から、ユニーク要素数を推定できます。

### 精度向上のための工夫

単一のカウンタでは分散が大きすぎるため、HyperLogLogでは：

1. $m$ 個のレジスタ（バケット）を用意
2. ハッシュ値の上位ビットでレジスタを選択
3. 残りのビットで先頭ゼロ数を計算
4. **調和平均**を使って推定値を統合

```
ハッシュ値（64ビット）:
┌─────────────┬──────────────────────────────────────┐
│  上位pビット │           残りのビット                 │
│  (レジスタ選択) │       (先頭ゼロ数を計算)              │
└─────────────┴──────────────────────────────────────┘
```

## データ構造

HyperLogLogは $m$ 個のレジスタで構成されます：

$$M = [M[0], M[1], \ldots, M[m-1]]$$

ここで：
- $m = 2^p$（$p$ は精度パラメータ）
- 各 $M[j]$ は観測された最大の「先頭ゼロ数 + 1」を保持
- 各レジスタは通常6ビット（最大値64まで記録可能）

## アルゴリズム

### 追加（Add）

要素 $x$ を追加：

1. ハッシュ値を計算: $h = hash(x)$
2. 上位 $p$ ビットでレジスタを選択: $j = \langle h_1, h_2, \ldots, h_p \rangle$
3. 残りのビットから先頭ゼロ数を計算: $w = \langle h_{p+1}, h_{p+2}, \ldots \rangle$

$$M[j] = \max(M[j], \rho(w))$$

ここで $\rho(w)$ は $w$ の先頭ゼロ数 + 1 です。

> **記号の補足**: $\langle h_1, h_2, \ldots, h_p \rangle$ はビット列を表し、$h_1$ から $h_p$ までの $p$ 個のビットからなる値を意味します。

### クエリ（Estimate）

カーディナリティの推定：

$$E = \alpha_m \cdot m^2 \cdot \left(\sum_{j=0}^{m-1} 2^{-M[j]}\right)^{-1}$$

ここで：
- $E$: 推定カーディナリティ
- $\alpha_m$: バイアス補正定数
- $m$: レジスタ数
- $M[j]$: 各レジスタの値

**重要**: 調和平均を使うことで、外れ値の影響を抑制し、精度を向上させています。

### バイアス補正定数 $\alpha_m$

$$\alpha_m = \begin{cases}
0.673 & (m = 16) \\
0.697 & (m = 32) \\
0.709 & (m = 64) \\
\frac{0.7213}{1 + 1.079/m} & (m \geq 128)
\end{cases}$$

## パラメータの選び方

精度 $p$ を選ぶと、レジスタ数 $m = 2^p$ が決まります。

### 標準誤差

$$\sigma \approx \frac{1.04}{\sqrt{m}}$$

| 精度 $p$ | レジスタ数 $m$ | 標準誤差 | メモリ使用量 |
|---------|---------------|---------|------------|
| 4 | 16 | 26% | 12 bytes |
| 8 | 256 | 6.5% | 192 bytes |
| 10 | 1,024 | 3.25% | 768 bytes |
| 12 | 4,096 | 1.625% | 3 KB |
| 14 | 16,384 | 0.81% | 12 KB |
| 16 | 65,536 | 0.41% | 48 KB |

### 推奨値

- 一般的な用途: $p = 14$（誤差約0.81%、12KB）
- 高精度が必要: $p = 16$（誤差約0.41%、48KB）
- メモリ制約がある: $p = 10$（誤差約3.25%、768バイト）

## 誤差補正

### 小カーディナリティ補正（線形カウンティング）

推定値が小さい場合、空レジスタが多く存在します。この場合、線形カウンティングを使用：

$$E_{linear} = m \cdot \ln\left(\frac{m}{V}\right)$$

ここで $V$ は値が0のレジスタ数です。

**適用条件**: $E < \frac{5}{2} m$ かつ $V > 0$

### 大カーディナリティ補正

32ビットハッシュを使用する場合、$E > 2^{32}/30$ のとき：

$$E_{corrected} = -2^{32} \cdot \ln\left(1 - \frac{E}{2^{32}}\right)$$

> **注意**: 64ビットハッシュを使用する場合、この補正は実質的に不要です。

## 実装例（Go）

```go
package hll

import (
    "hash/fnv"
    "math"
    "math/bits"
)

// HyperLogLog はHyperLogLogデータ構造
type HyperLogLog struct {
    p         uint8    // 精度 (4-16)
    m         uint32   // レジスタ数 = 2^p
    registers []uint8  // 各レジスタ
    alpha     float64  // バイアス補正定数
}

// New は新しいHyperLogLogを作成
// p: 精度パラメータ (4-16を推奨)
func New(p uint8) *HyperLogLog {
    if p < 4 || p > 16 {
        panic("pは4から16の範囲で指定してください")
    }

    m := uint32(1) << p

    return &HyperLogLog{
        p:         p,
        m:         m,
        registers: make([]uint8, m),
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

// hash は要素のハッシュ値を計算
func (h *HyperLogLog) hash(item string) uint64 {
    hasher := fnv.New64a()
    hasher.Write([]byte(item))
    return hasher.Sum64()
}

// Add は要素を追加
func (h *HyperLogLog) Add(item string) {
    hashValue := h.hash(item)

    // 上位pビットでレジスタを選択
    j := hashValue >> (64 - h.p)

    // 残りのビットで先頭ゼロ数を計算
    w := hashValue << h.p
    rho := uint8(bits.LeadingZeros64(w)) + 1

    // 最大値を更新
    if rho > h.registers[j] {
        h.registers[j] = rho
    }
}

// Estimate はカーディナリティを推定（補正込み）
func (h *HyperLogLog) Estimate() uint64 {
    // 調和平均の逆数を計算
    sum := 0.0
    zeroCount := uint32(0)

    for _, val := range h.registers {
        sum += math.Pow(2, -float64(val))
        if val == 0 {
            zeroCount++
        }
    }

    // 生の推定値
    estimate := h.alpha * float64(h.m) * float64(h.m) / sum

    // 小カーディナリティ補正
    if estimate < 2.5*float64(h.m) && zeroCount > 0 {
        // 線形カウンティング
        estimate = float64(h.m) * math.Log(float64(h.m)/float64(zeroCount))
    }

    // 大カーディナリティ補正（32ビットハッシュの場合）
    // 64ビットハッシュを使用しているため、ここでは省略

    return uint64(estimate + 0.5) // 四捨五入
}
```

## マージ（分散システム対応）

HyperLogLogはマージ可能です：

$$M_{combined}[j] = \max(M_A[j], M_B[j])$$

```go
// Merge は2つのHyperLogLogをマージ
func Merge(a, b *HyperLogLog) *HyperLogLog {
    if a.p != b.p {
        panic("精度pが一致しません")
    }

    result := New(a.p)

    for i := uint32(0); i < a.m; i++ {
        if a.registers[i] > b.registers[i] {
            result.registers[i] = a.registers[i]
        } else {
            result.registers[i] = b.registers[i]
        }
    }

    return result
}
```

### マージ時の実装注意点

> [!IMPORTANT]
> **マージを想定する場合、すべてのノードで `p`（精度）を統一する必要があります。**

#### なぜ精度を統一する必要があるのか？

精度 $p$ が異なると、レジスタ数 $m$ が異なり、同じ要素が異なるレジスタに割り当てられます：

```
❌ 精度が異なる場合:
Node A (p=10):  "user123" → レジスタ 512
Node B (p=12):  "user123" → レジスタ 2048  ← レジスタがずれる！

✅ 精度が共通の場合:
Node A (p=10):  "user123" → レジスタ 512
Node B (p=10):  "user123" → レジスタ 512   ← 同じレジスタ！
```

## 使用例

```go
func main() {
    // 精度14（誤差約0.81%）でHLLを作成
    hll := New(14)

    // ユーザーIDを追加
    userIDs := []string{"user1", "user2", "user1", "user3", "user2", "user4"}
    for _, id := range userIDs {
        hll.Add(id)
    }

    // ユニークユーザー数を推定
    fmt.Printf("推定ユニーク数: %d (真の値: 4)\n", hll.Estimate())
}
```

## 関連アルゴリズム

### LogLog

HyperLogLogの前身（2003年、Durand & Flajolet）：

- **幾何平均**を使用（HyperLogLogは調和平均）
- 標準誤差 $\approx 1.30/\sqrt{m}$（HyperLogLogより約25%劣る）

### HyperLogLog++

Googleによる2013年の改良版：

- 64ビットハッシュ対応
- バイアス補正の改善
- スパース表現による省メモリ化

→ 詳細は [09-hyperloglog-pp.md](./09-hyperloglog-pp.md) を参照

## 比較：他の確率的データ構造

| データ構造 | 用途 | 過小推定 | 過大推定 | マージ |
|-----------|------|---------|---------|--------|
| HyperLogLog | カーディナリティ | ⚠️ 近似 | ⚠️ 近似 | ✅ |
| Count-Min Sketch | 頻度推定 | ❌ なし | ✅ あり | ✅ |
| Bloom Filter | 存在確認 | ❌ なし | ✅ あり | ✅ |

## 応用例

| 応用 | 説明 |
|------|------|
| ユニークユーザー数 | Webアクセスログのユニークビジター数 |
| データベース最適化 | クエリプランナーでのカーディナリティ推定 |
| ネットワーク監視 | ユニークIP数のカウント |
| 広告システム | リーチ（到達ユーザー数）の測定 |
| 重複検出 | 大規模データセットでの重複率推定 |

## 空間・時間計算量

| 操作 | 時間計算量 | 空間計算量 |
|------|-----------|-----------|
| Add | $O(1)$ | - |
| Estimate | $O(m)$ | - |
| Merge | $O(m)$ | - |
| 構造体全体 | - | $O(m) = O(2^p)$ |

## 参考資料

- [HyperLogLog: the analysis of a near-optimal cardinality estimation algorithm](http://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf) - Flajolet et al. (2007)
- [HyperLogLog - Wikipedia](https://en.wikipedia.org/wiki/HyperLogLog)
- [Redis HyperLogLog implementation](https://redis.io/docs/data-types/hyperloglogs/)
