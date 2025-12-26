# Online Statistics Tutorial

分散システムにおけるデータの傾向を調べるための基礎知識と実装をまとめたリポジトリです。

## 概要

オンライン統計（Online Statistics）とは、すべてのデータをメモリに保持せずに、データが到着するたびに統計量を逐次更新する手法です。これは以下のような場面で特に有用です：

- **ストリーミングデータ処理**: リアルタイムでデータが流れてくる場合
- **メモリ制約**: 大量のデータを一度に保持できない場合
- **分散システム**: 複数のノードで計算した統計を集約する場合

## 学習トピック

### 1. オンライン平均（Online Mean）
データを1つずつ処理しながら平均を更新する手法。

```
新しい平均 = 古い平均 + (新しい値 - 古い平均) / カウント
```

### 2. オンライン分散（Welford's Algorithm）
数値的に安定した方法で分散・標準偏差を計算する手法。

### 3. オンラインヒストグラム
固定幅または適応型のビンを使用して、データ分布を近似する手法。

### 4. 四分位数・パーセンタイル推定 ⭐
オンラインでの四分位数計算は**厳密解ではなく推定値**になります。主なアルゴリズム：

- **P² アルゴリズム**: 5つのマーカーを使用してパーセンタイルを推定
- **t-digest**: 圧縮されたヒストグラムを使用した高精度な推定
- **GK アルゴリズム（Greenwald-Khanna）**: ε-近似を保証する手法

> **なぜ推定値になるのか？**
> 
> 正確な四分位数を計算するには、すべてのデータをソートする必要があります。
> しかし、ストリーミング環境ではすべてのデータを保持できないため、
> 限られたメモリで「十分に正確な」値を推定する必要があります。

### 5. 統計値のマージ（Mergeable Statistics）⭐
分散システムでは、各ノードで計算した統計を**マージ**して全体の統計を得る必要があります。

| 統計量 | マージ可能性 | 備考 |
|--------|------------|------|
| カウント | ✅ 容易 | 単純に足し合わせ |
| 合計 | ✅ 容易 | 単純に足し合わせ |
| 平均 | ✅ 可能 | 加重平均として計算 |
| 分散 | ✅ 可能 | 並列アルゴリズムが必要 |
| 最小/最大 | ✅ 容易 | min/maxを取る |
| 中央値 | ⚠️ 近似 | t-digestなどが必要 |
| ヒストグラム | ✅ 可能 | ビンごとにマージ |

#### マージの例：平均のマージ

```
全体の平均 = (n₁ × mean₁ + n₂ × mean₂) / (n₁ + n₂)
```

#### マージの例：分散のマージ（並列Welfordアルゴリズム）

```
δ = mean₂ - mean₁
combined_mean = (n₁ × mean₁ + n₂ × mean₂) / (n₁ + n₂)
combined_M2 = M2₁ + M2₂ + δ² × n₁ × n₂ / (n₁ + n₂)
```

### 6. 指数移動平均（Exponential Moving Average）
最新のデータにより大きな重みを与える移動平均。

### 7. Count-Min Sketch（発展）
確率的データ構造を使用した頻度推定。

## ディレクトリ構造

```
.
├── README.md
├── go.mod
├── cmd/
│   └── examples/          # 実行可能なサンプル
├── pkg/
│   ├── mean/              # オンライン平均（マージ対応）
│   ├── variance/          # オンライン分散（Welford、並列マージ対応）
│   ├── histogram/         # オンラインヒストグラム
│   ├── quantile/          # 四分位数・パーセンタイル推定（t-digest）
│   ├── ema/               # 指数移動平均
│   └── cms/               # Count-Min Sketch
└── docs/
    ├── 01-online-mean.md
    ├── 02-online-variance.md
    ├── 03-online-histogram.md
    ├── 04-quantile-estimation.md
    ├── 05-merging-statistics.md
    ├── 06-ema.md
    └── 07-count-min-sketch.md
```

## 環境

- Go 1.21+

## クイックスタート

```bash
# リポジトリのクローン
git clone https://github.com/your-username/online-stats-tutorial.git
cd online-stats-tutorial

# 依存関係のインストール
go mod tidy

# サンプルの実行
go run cmd/examples/mean/main.go
```

## 使用例

### オンライン平均

```go
package main

import (
    "fmt"
    "online-stats-tutorial/pkg/mean"
)

func main() {
    m := mean.New()
    
    // データを1つずつ追加
    for _, v := range []float64{1.0, 2.0, 3.0, 4.0, 5.0} {
        m.Add(v)
        fmt.Printf("現在の平均: %.2f (n=%d)\n", m.Mean(), m.Count())
    }
}
```

## 参考資料

- [Welford's online algorithm](https://en.wikipedia.org/wiki/Algorithms_for_calculating_variance#Welford's_online_algorithm)
- [Exponential smoothing](https://en.wikipedia.org/wiki/Exponential_smoothing)
- [Count-Min Sketch](https://en.wikipedia.org/wiki/Count%E2%80%93min_sketch)

## ライセンス

MIT License
