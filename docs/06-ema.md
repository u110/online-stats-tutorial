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
- $\alpha$ は平滑化係数（$0 < \alpha \leq 1$）
- $x_n$ は新しいデータ
- $EMA_{n-1}$ は前回のEMA

### 別の表記

$$EMA_n = EMA_{n-1} + \alpha \cdot (x_n - EMA_{n-1})$$

### 重みの減衰

$k$ 時点前のデータの重み：

$$w_k = \alpha \cdot (1 - \alpha)^k$$

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

## マージについて

> **注意**: EMAは一般的にマージできません。

EMAは時系列の順序に依存するため、2つの独立したEMAをマージして「全体のEMA」を得ることはできません。

分散システムでEMAを使用する場合：
1. **中央ノードで時系列順に処理する**
2. **近似としてウィンドウベースの統計を使用する**
3. **各ノードで独立したEMAを維持し、個別に監視する**

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
