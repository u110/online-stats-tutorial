# Online Statistics Tutorial

分散システムにおけるデータの傾向を調べるための基礎知識と実装をまとめたリポジトリです。

## 概要

オンライン統計（Online Statistics）とは、すべてのデータをメモリに保持せずに、データが到着するたびに統計量を逐次更新する手法です。これは以下のような場面で特に有用です：

- **ストリーミングデータ処理**: リアルタイムでデータが流れてくる場合
- **メモリ制約**: 大量のデータを一度に保持できない場合
- **分散システム**: 複数のノードで計算した統計を集約する場合

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
    ├── README.md               # 学習トピック一覧
    ├── 01-online-mean.md
    ├── 02-online-variance.md
    ├── 03-online-histogram.md
    ├── 04-quantile-estimation.md
    ├── 05-merging-statistics.md
    ├── 06-ema.md
    ├── 07-count-min-sketch.md
    ├── 08-hyperloglog.md
    └── 09-hyperloglog-pp.md
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
