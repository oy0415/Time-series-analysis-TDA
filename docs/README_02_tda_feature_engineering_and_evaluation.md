# TDA Feature Engineering and Evaluation for Time Series Forecasting

* * *

## 要約

目的 : 家庭用電力使用量の時系列データに対して、Sliding Window / Takens Embedding と Persistent Homology を用いて TDA特徴量を作成し、通常特徴量モデルに追加したときの予測性能への影響を検証する  
手法 : 時系列の点群化、Persistent Homology の計算、Persistence Diagram / Barcode の可視化、Rolling Window TDA特徴量の作成、通常特徴量との結合、機械学習モデルによる比較  
評価方法 : MAE、RMSE、MAPE、MASE、R²、TDA追加による改善率、TDA特徴量重要度  
結果 : 今回の設定では、通常特徴量のみの `GradientBoosting` が最良であり、TDA特徴量の追加による明確な予測性能改善は確認できなかった  
使用技術 : Python / pandas / numpy / matplotlib / scikit-learn / giotto-tda / ripser

* * *

## プロジェクト概要

この notebook では、UCI Individual Household Electric Power Consumption データを用いて、日次平均の `Global_active_power` を対象に、TDA特徴量の作成と有効性検証を行いました。

前段の notebook では、通常の時系列解析として、ラグ特徴量・移動平均特徴量・曜日・月などを用いた予測モデルを構築しました。本 notebook では、その通常特徴量モデルに対して、時系列の形状情報を表す TDA特徴量を追加し、予測性能が改善するかを検証しています。

分析の流れは次の通りです。

1. 目的設定
2. データの読み込み・前処理
3. TDAに用いる系列の選択
4. Sliding Window / Takens Embedding による点群化
5. Persistent Homology の計算
6. Persistence Diagram / Barcode の可視化
7. TDA特徴量の作成
8. TDA特徴量の考察
9. Rolling Window によるTDA特徴量の作成
10. 通常特徴量とTDA特徴量の結合
11. モデル学習：通常特徴量のみ vs 通常特徴量 + TDA特徴量
12. 評価：TDA特徴量の有効性確認
13. 結果の解釈

* * *

## 使用データ

- データ名: UCI Individual Household Electric Power Consumption
- 使用ファイル: `data/household_power_consumption.txt`
- 元データの頻度: 1分間隔
- 分析対象: `Global_active_power`
- TDA対象系列: `Global_active_power` の日次平均系列

### 主な変数

| 変数名 | 内容 |
|---|---|
| `Date` | 日付 |
| `Time` | 時刻 |
| `Global_active_power` | 家庭全体の有効電力消費量 |
| `Global_reactive_power` | 家庭全体の無効電力消費量 |
| `Voltage` | 電圧 |
| `Global_intensity` | 電流 |
| `Sub_metering_1` | サブメータリング1 |
| `Sub_metering_2` | サブメータリング2 |
| `Sub_metering_3` | サブメータリング3 |

本分析では、1変量時系列に対するTDA特徴量作成を目的として、`Global_active_power` の日次平均系列を使用しました。

* * *

## 分析のポイント

### 1. データ読み込み・前処理

元データの `Date` と `Time` を結合し、`datetime` 型の時系列インデックスを作成しました。その後、数値列を `float` 型に変換し、欠損値処理を行ったうえで、日次平均系列を作成しました。

#### 処理内容

- `Date` と `Time` を結合して `datetime` を作成
- 数値列を `float` 型に変換
- 欠損値の確認と処理
- 時系列順に並べ替え
- 1分間隔データから日次平均系列を作成
- TDA対象系列として `Global_active_power` を選択
- `StandardScaler` により標準化

* * *

### 2. TDAに用いる系列の選択

TDAに用いる系列として、家庭全体の有効電力消費量を表す `Global_active_power` の日次平均系列を選択しました。

理由は、以下の通りです。

- 電力使用量の中心的な変数である
- 通常の時系列予測 notebook でも予測対象として使用している
- 日次の反復パターンや変動構造を、Sliding Window によって点群化しやすい
- 通常特徴量モデルとの比較対象を揃えられる

* * *

### 3. Sliding Window / Takens Embedding による点群化

1変量時系列を、一定期間の値を並べたベクトルとして表現し、点群に変換しました。

今回の主な設定は次の通りです。

| 項目 | 値 |
|---|---:|
| `embedding_dim` | 30 |
| `delay` | 1 |
| `stride` | 1 |
| PH計算用点群数 | 500 |

この設定では、1点が「30日分の標準化された電力使用量パターン」を表します。

点群は30次元であるため、そのまま可視化することはできません。そのため、確認用に以下の可視化を行いました。

- 最初の3つの遅延座標を使った3次元プロット
- PCAによる2次元可視化
- PCAによる3次元可視化

* * *

### 4. Persistent Homology の計算

点群 `X_takens_ph` に対して、Vietoris-Rips 複体に基づく Persistent Homology を計算しました。

計算対象は以下です。

| ホモロジー次元 | 意味 |
|---|---|
| H0 | 点群の連結成分の統合過程 |
| H1 | ループ構造の候補 |

実装では、基本的に `giotto-tda` の `VietorisRipsPersistence` を使用し、環境によって利用できない場合に `ripser` を使うフォールバックを用意しています。

#### Persistent Homology の要約

| dimension | n_points | max_lifetime | mean_lifetime | median_lifetime | total_lifetime |
|---:|---:|---:|---:|---:|---:|
| H0 | 499 | 7.9137 | 3.4830 | 3.2088 | 1738.0052 |
| H1 | 426 | 0.6816 | 0.1436 | 0.1151 | 61.1528 |

H1特徴が検出されているため、今回の点群化設定では、ループ構造の候補が存在することが確認されました。

* * *

### 5. Persistence Diagram / Barcode の可視化

Persistent Homology の計算結果を以下の形式で可視化しました。

- Persistence Diagram
- H0 / H1 別 Persistence Diagram
- Barcode
- H0 / H1 別 Barcode
- H1 の長寿命特徴の確認

Persistence Diagram では、対角線から離れた点ほど lifetime が長く、相対的に強いトポロジカル特徴と解釈できます。

Barcode では、横棒が長いほど、その特徴が広いスケール範囲で持続していることを表します。

* * *

## TDA特徴量の作成

Persistent Homology の結果から、機械学習モデルに投入できる数値特徴量を作成しました。

### 作成した主なTDA特徴量

| 特徴量 | 内容 |
|---|---|
| `H0_n_points` | H0の persistence point 数 |
| `H0_lifetime_max` | H0の最大 lifetime |
| `H0_lifetime_sum` | H0の lifetime 合計 |
| `H0_lifetime_mean` | H0の lifetime 平均 |
| `H0_persistence_entropy` | H0の persistence entropy |
| `H1_n_points` | H1の persistence point 数 |
| `H1_lifetime_max` | H1の最大 lifetime |
| `H1_lifetime_sum` | H1の lifetime 合計 |
| `H1_lifetime_mean` | H1の lifetime 平均 |
| `H1_persistence_entropy` | H1の persistence entropy |
| `H1_lifetime_top1`〜`top5` | H1 lifetime 上位値 |
| `H1_max_to_sum_ratio` | H1最大 lifetime が合計 lifetime に占める割合 |

### H1特徴量の確認結果

| 特徴量 | 値 |
|---|---:|
| `H1_n_points` | 426 |
| `H1_lifetime_max` | 0.6816 |
| `H1_lifetime_sum` | 61.1528 |
| `H1_lifetime_mean` | 0.1436 |
| `H1_lifetime_median` | 0.1151 |
| `H1_persistence_entropy` | 5.7125 |
| `H1_lifetime_top1` | 0.6816 |
| `H1_lifetime_top2` | 0.5662 |
| `H1_lifetime_top3` | 0.5318 |
| `H1_max_to_sum_ratio` | 0.0111 |

H1の lifetime は複数存在しており、特定の1本だけが圧倒的に支配的というより、複数のループ候補が分散して存在している可能性があります。

ただし、この段階では「TDA特徴量が作成できた」「H1特徴が検出された」と言えるのみであり、予測性能への有効性はまだ判断できません。

* * *

## Rolling Window TDA特徴量

系列全体に対する1行のTDA特徴量だけでは、各時点の予測モデルに投入できません。そのため、Rolling Window によって、時点ごとのTDA特徴量を作成しました。

### Rolling Window の設定

| 項目 | 内容 |
|---|---:|
| rolling window size | 180日 |
| rolling step | 7日 |
| embedding_dim | 30 |
| delay | 1 |
| stride | 1 |
| PH計算用点群数 | 150 |

作成された Rolling TDA特徴量は以下の構造です。

| 項目 | 値 |
|---|---:|
| 行数 | 181 |
| 列数 | 34 |

各行は、ある rolling window の終了時点までの時系列パターンから作成されたTDA特徴量を表します。

* * *

## 通常特徴量とTDA特徴量の結合

通常特徴量には、前段の時系列予測 notebook と同様に、ラグ特徴量・移動平均特徴量・曜日・月・週末フラグを使用しました。

### 通常特徴量

| 特徴量 | 内容 |
|---|---|
| `lag_1` | 1日前の値 |
| `lag_2` | 2日前の値 |
| `lag_3` | 3日前の値 |
| `lag_7` | 7日前の値 |
| `lag_14` | 14日前の値 |
| `rolling_mean_7` | 過去7日の移動平均 |
| `rolling_mean_14` | 過去14日の移動平均 |
| `rolling_std_7` | 過去7日の移動標準偏差 |
| `rolling_std_14` | 過去14日の移動標準偏差 |
| `dayofweek` | 曜日 |
| `month` | 月 |
| `is_weekend` | 週末フラグ |

TDA特徴量は rolling window の終了日そのものではなく、翌日以降に利用可能な特徴量として扱いました。これにより、同じ日の目的変数を作る際に未来情報が混入しないようにしています。

### 結合後データ

| 項目 | 値 |
|---|---:|
| 結合前の通常特徴量データ | 1428行 × 14列 |
| TDA結合直後 | 1428行 × 49列 |
| TDA利用可能行のみ | 1262行 × 49列 |
| 使用期間 | 2007-06-14 〜 2010-11-26 |

### train / test 分割

時系列順に train / test を分割しました。

| データ | shape |
|---|---:|
| `X_normal_train` | 1009 × 12 |
| `X_normal_test` | 253 × 12 |
| `X_tda_train` | 1009 × 45 |
| `X_tda_test` | 253 × 45 |

| 区分 | 期間 |
|---|---|
| train | 2007-06-14 〜 2010-03-18 |
| test | 2010-03-19 〜 2010-11-26 |

* * *

## モデル構築

通常特徴量のみの場合と、通常特徴量にTDA特徴量を追加した場合を、同じ train/test 期間で比較しました。

### 比較したモデル

| モデル | 役割 |
|---|---|
| Ridge | 線形モデルの比較対象 |
| RandomForest | 非線形モデル |
| GradientBoosting | 非線形モデル |

* * *

## 評価結果

### 全モデルの評価結果

| feature_set | model | MAE | RMSE | MAPE | MASE | R² |
|---|---|---:|---:|---:|---:|---:|
| normal_only | GradientBoosting | 0.1583 | 0.2205 | 17.6333 | 0.6038 | 0.4462 |
| normal_plus_tda | GradientBoosting | 0.1592 | 0.2208 | 18.4358 | 0.6075 | 0.4450 |
| normal_only | RandomForest | 0.1632 | 0.2223 | 18.6036 | 0.6227 | 0.4372 |
| normal_plus_tda | RandomForest | 0.1662 | 0.2226 | 19.3889 | 0.6341 | 0.4356 |
| normal_only | Ridge | 0.1690 | 0.2285 | 19.4943 | 0.6449 | 0.4056 |
| normal_plus_tda | Ridge | 0.1799 | 0.2370 | 21.3660 | 0.6864 | 0.3604 |

RMSE基準の最良モデルは、通常特徴量のみの `GradientBoosting` でした。

### 最良モデル

| 項目 | 値 |
|---|---:|
| feature_set | `normal_only` |
| model | `GradientBoosting` |
| MAE | 0.1583 |
| RMSE | 0.2205 |
| MAPE | 17.6333 |
| MASE | 0.6038 |
| R² | 0.4462 |

MASEが1未満であるため、今回の評価期間では、ナイーブ予測よりも誤差を小さくできていると判断できます。

* * *

## TDA特徴量の有効性確認

TDA特徴量の追加効果を、同じモデル同士で比較しました。

| モデル | RMSE改善率 | MAE改善率 | RMSE改善 | MAE改善 |
|---|---:|---:|---|---|
| GradientBoosting | -0.10% | -0.60% | False | False |
| RandomForest | -0.14% | -1.84% | False | False |
| Ridge | -3.74% | -6.44% | False | False |

今回の設定では、3モデルすべてでTDA特徴量追加後にRMSE・MAEの改善は確認されませんでした。

そのため、今回の結果からは、TDA特徴量の追加によって予測性能が改善したとは言えません。

### TDA特徴量重要度

木ベースモデルでは、TDA特徴量にも一定の特徴量重要度が確認されました。

| モデル | TDA特徴量重要度割合 |
|---|---:|
| RandomForest | 約9.30% |
| GradientBoosting | 約6.78% |

ただし、特徴量重要度があることと、予測性能が改善することは同じではありません。今回の結果では、TDA特徴量はモデル内部で一部利用されていたものの、最終的な予測精度の改善にはつながりませんでした。

### 日次絶対誤差の変化

最もRMSE改善率が大きかった `GradientBoosting` について、日ごとの絶対誤差を比較しました。

| 項目 | 値 |
|---|---:|
| TDA追加で日次絶対誤差が改善した割合 | 45.85% |

一部の日ではTDA特徴量の追加により絶対誤差が小さくなりましたが、test期間全体の平均的な評価指標では改善しませんでした。

* * *

## 結論

本分析では、時系列データに対して Sliding Window / Takens Embedding を行い、Persistent Homology によってTDA特徴量を作成しました。

さらに、Rolling Window によって時変的なTDA特徴量を作成し、通常特徴量と結合して予測モデルに投入しました。

その結果、今回の設定では、通常特徴量のみの `GradientBoosting` が最も良い性能を示しました。

一方で、TDA特徴量を追加したモデルでは、RMSE、MAE、MASE、R²の改善は確認されませんでした。

したがって、今回の実験からは、TDA特徴量の追加による明確な予測性能の改善は確認できませんでした。

ただし、TDA特徴量は木ベースモデルにおいて一定の特徴量重要度を持っていたため、時系列の形状情報をある程度表現していた可能性はあります。

今回言えるのは、あくまで以下の範囲です。

> 今回のデータ、前処理、embedding_dim、delay、rolling window size、TDA特徴量設計、モデル設定のもとでは、TDA特徴量の追加による明確な予測性能の改善は確認できなかった。

* * *

## 今後の課題

TDA特徴量の有効性をより詳しく検証するためには、以下の追加検証が必要です。

1. Sliding Window / Takens Embedding のパラメータ変更
   - `embedding_dim`
   - `delay`
   - `stride`

2. Rolling Window の幅の変更
   - 短期的な変動を捉える window
   - 中期的な変動を捉える window
   - 長期的な変動を捉える window

3. TDA特徴量の種類の追加
   - lifetime の最大値
   - lifetime の平均
   - lifetime の合計
   - persistence entropy
   - Betti curve
   - persistence image

4. モデルの変更
   - Ridge
   - RandomForest
   - GradientBoosting
   - XGBoost
   - LightGBM

5. タスク設定の変更
   - 1日先予測
   - 数日先予測
   - 異常検知
   - 変動局面の分類

* * *

## 使用ライブラリ

主な使用ライブラリは以下の通りです。

```text
pandas
numpy
matplotlib
scikit-learn
giotto-tda
ripser
tqdm
```

`giotto-tda` が使用できない環境では、Persistent Homology の計算に `ripser` を使うフォールバックを用意しています。

* * *

## 出力ファイル

notebook実行時に、主に以下のCSVファイルが生成されます。

| ファイル名 | 内容 |
|---|---|
| `tda_features_global.csv` | 系列全体に対するTDA特徴量 |
| `rolling_tda_features.csv` | Rolling Window によるTDA特徴量 |
| `model_features_with_tda.csv` | 通常特徴量とTDA特徴量を結合したモデル入力用データ |
| `model_comparison_normal_vs_tda.csv` | 通常特徴量のみ vs TDA追加モデルの比較結果 |
| `model_improvement_tda_effect.csv` | TDA追加による改善率の表 |
| `best_model_predictions.csv` | 最良モデルの予測結果 |
| `tda_effect_summary_section12.csv` | TDA有効性評価の要約 |
| `tda_importance_summary_section12.csv` | TDA特徴量重要度の要約 |
| `tda_error_compare_section12.csv` | TDA追加前後の日次誤差比較 |
| `tda_error_correlation_section12.csv` | TDA特徴量と予測誤差の関係確認結果 |

* * *

## ファイル構成

想定するファイル構成は以下です。

```text
Time-series-analysis-TDA/
│
├── README.md
│
├── data/
│   └── household_power_consumption.txt
│
├── docs/
│   ├── README_01_time_series_forecasting.md
│   └── README_02_tda_feature_engineering_and_evaluation.md
│
├── notebooks/
│   ├── 01_time_series_forecasting.ipynb
│   └── 02_tda_feature_engineering_and_evaluation.ipynb
│
└── results/
    ├── figures/
    └── tables/
```

現状の notebook では、CSVファイルは実行時のカレントディレクトリに保存されます。GitHub上で整理する場合は、出力CSVを `results/tables/` に移動する構成にすると見通しが良くなります。

* * *

## このnotebookの位置づけ

この notebook は、TDA特徴量の導入によって必ず予測性能が上がることを示すものではありません。

むしろ、以下を示すための検証 notebook です。

- 時系列データを Sliding Window / Takens Embedding によって点群化できる
- Persistent Homology を計算し、Persistence Diagram / Barcode を確認できる
- Persistence Diagram から数値特徴量を作成できる
- Rolling Window によって時点ごとのTDA特徴量を作成できる
- 通常特徴量とTDA特徴量を結合して、同一条件でモデル比較できる
- TDA特徴量が予測性能に寄与したかを、事実ベースで評価できる

今回の結果では明確な性能改善は確認されませんでしたが、TDAを時系列予測モデルに組み込む一連の実装手順は整理できています。
