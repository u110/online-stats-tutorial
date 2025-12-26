# 指数移動平均（Exponential Moving Average: EMA）

## 概要

指数移動平均（EMA）は、最新のデータにより大きな重みを与える移動平均です。時系列データの傾向分析、ノイズ除去、異常検知などに広く使用されます。

## なぜEMAが必要か？

### 単純移動平均（SMA）の問題

単純移動平均は過去 $k$ 個のデータの平均：

$$SMA = \frac{1}{k}\sum_{i=0}^{k-1} x_{n-i}$$

問題点：
- 過去 $k$ 個のデータをメモリに保持する必要がある
- 古いデータと新しいデータが同じ重みを持つ
- ウィンドウの開始/終了時に急激な変化が起きる

### EMAの利点

- 固定メモリ量（状態は1つの値のみ）
- 最新のデータにより大きな重みを与える
- 滑らかな更新

## 数式

### 更新式

$$EMA_n = \alpha \cdot x_n + (1 - \alpha) \cdot EMA_{n-1}$$

ここで：
- $EMA_n$: $n$ 番目のデータを追加した後のEMA値
- $EMA_{n-1}$: 前回のEMA値
- $\alpha$: 平滑化係数（$0 < \alpha \leq 1$）。大きいほど新しいデータを重視
- $x_n$: 新しく追加するデータ

### 別の表記

$$EMA_n = EMA_{n-1} + \alpha \cdot (x_n - EMA_{n-1})$$

> **ポイント**: この形式はオンライン平均の漸化式と同じ構造です。違いは $\frac{1}{n}$ の代わりに固定の $\alpha$ を使う点です。

### 重みの減衰

$k$ 時点前のデータの重み：

$$w_k = \alpha \cdot (1 - \alpha)^k$$

ここで：
- $w_k$: $k$ 時点前のデータに与えられる重み
- $k = 0$ は最新データ、$k = 1$ は1つ前のデータ

重みの合計は1に収束：

$$\sum_{k=0}^{\infty} \alpha \cdot (1 - \alpha)^k = 1$$

## αの選び方

### スパンからの変換

「過去N個のデータを考慮」という意味でのスパン $N$：

$$\alpha = \frac{2}{N + 1}$$

例：
- $N = 10$ → $\alpha = 0.182$
- $N = 20$ → $\alpha = 0.095$
- $N = 50$ → $\alpha = 0.039$

### 半減期からの変換

重みが半分になるまでの時間 $t_{1/2}$：

$$\alpha = 1 - 2^{-1/t_{1/2}}$$

## 実装例（Go）

```go
package ema

// EMA は指数移動平均を計算
type EMA struct {
    alpha   float64
    value   float64
    count   int64
    initialized bool
}

// NewFromSpan はスパンからEMAを作成
func NewFromSpan(span int) *EMA {
    alpha := 2.0 / (float64(span) + 1.0)
    return &EMA{alpha: alpha}
}

// NewFromAlpha はalphaから直接EMAを作成
func NewFromAlpha(alpha float64) *EMA {
    return &EMA{alpha: alpha}
}

// NewFromHalfLife は半減期からEMAを作成
func NewFromHalfLife(halfLife float64) *EMA {
    alpha := 1.0 - math.Pow(2.0, -1.0/halfLife)
    return &EMA{alpha: alpha}
}

// Add は値を追加
func (e *EMA) Add(value float64) {
    e.count++
    if !e.initialized {
        e.value = value
        e.initialized = true
        return
    }
    
    e.value = e.alpha*value + (1-e.alpha)*e.value
}

// Value は現在のEMAを返す
func (e *EMA) Value() float64 {
    return e.value
}

// Count はデータ数を返す
func (e *EMA) Count() int64 {
    return e.count
}
```

## バイアス補正

初期化直後のEMAは、初期値に偏っています（バイアス）。これを補正する方法：

$$EMA_{corrected} = \frac{EMA_n}{1 - (1 - \alpha)^n}$$

```go
// ValueCorrected はバイアス補正済みのEMAを返す
func (e *EMA) ValueCorrected() float64 {
    if e.count == 0 {
        return 0
    }
    correction := 1.0 - math.Pow(1-e.alpha, float64(e.count))
    return e.value / correction
}
```

## 変種

### 二重指数移動平均（Double EMA, DEMA）

トレンドをより早く追跡：

$$DEMA = 2 \cdot EMA_1 - EMA_2$$

ここで $EMA_2$ は $EMA_1$ のEMA。

```go
// DEMA は二重指数移動平均
type DEMA struct {
    ema1 *EMA
    ema2 *EMA
}

// NewDEMA は新しいDEMAを作成
func NewDEMA(span int) *DEMA {
    return &DEMA{
        ema1: NewFromSpan(span),
        ema2: NewFromSpan(span),
    }
}

// Add は値を追加
func (d *DEMA) Add(value float64) {
    d.ema1.Add(value)
    d.ema2.Add(d.ema1.Value())
}

// Value はDEMAを返す
func (d *DEMA) Value() float64 {
    return 2*d.ema1.Value() - d.ema2.Value()
}
```

### 指数加重移動分散（EWMV）

分散のEMA版：

$$EWMV_n = (1 - \alpha) \cdot (EWMV_{n-1} + \alpha \cdot (x_n - EMA_{n-1})^2)$$

```go
// EWMV は指数加重移動分散
type EWMV struct {
    alpha    float64
    mean     float64
    variance float64
    initialized bool
}

// Add は値を追加
func (e *EWMV) Add(value float64) {
    if !e.initialized {
        e.mean = value
        e.variance = 0
        e.initialized = true
        return
    }
    
    diff := value - e.mean
    incr := e.alpha * diff
    e.mean += incr
    e.variance = (1 - e.alpha) * (e.variance + diff*incr)
}
```

## 実装時の注意点

> [!CAUTION]
> **EMAは分散システムでマージできません。**

EMAは時系列の順序に依存するため、2つの独立したEMAをマージして「全体のEMA」を得ることは**数学的に不可能**です。

### なぜマージできないのか？

EMAは各時点での値が過去のすべての値に依存しています。2つのノードで異なる順序でデータを処理した場合、マージしても意味のある結果は得られません：

```
Node A: x1 → x2 → x3  →  EMA_A
Node B: x4 → x5 → x6  →  EMA_B

EMA_A + EMA_B ≠ 全体のEMA（x1 → x2 → x3 → x4 → x5 → x6）
```

### 分散システムでの代替策

| 方法 | 説明 | 適用場面 |
|------|------|---------|
| 中央ノードで処理 | すべてのデータを1箇所に集約してEMA計算 | データ量が少ない場合 |
| ウィンドウベースの統計 | 固定ウィンドウの平均など、マージ可能な統計を使用 | 厳密なEMAが不要な場合 |
| 各ノードで独立したEMA | 各ノードのEMAを個別に監視 | ノードごとの傾向が重要な場合 |
| Kafka Streams等を使用 | パーティションキーで時系列順序を保証 | ストリーム処理基盤がある場合 |

## 用途

| 用途 | 説明 |
|------|------|
| トレンド分析 | 時系列データの傾向を可視化 |
| ノイズフィルタリング | センサーデータなどのノイズ除去 |
| 異常検知 | EMAからの逸脱を検出 |
| レート制限 | リクエストレートの推定 |
| 負荷平均 | システム負荷の平滑化（uptime） |

## 計算量

| 操作 | 時間計算量 | 空間計算量 |
|------|-----------|-----------|
| Add | $O(1)$ | $O(1)$ |
| Value | $O(1)$ | $O(1)$ |

## 参考資料

- [Exponential smoothing - Wikipedia](https://en.wikipedia.org/wiki/Exponential_smoothing)
- [Moving average - Wikipedia](https://en.wikipedia.org/wiki/Moving_average#Exponential_moving_average)
