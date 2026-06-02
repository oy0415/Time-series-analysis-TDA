# Time-series-analysis-TDA
※本リポジトリは初期版です。現在は改良版として以下のリポジトリを主に公開しています。
https://github.com/oy0415/Time-series-analysis-TDA_v2

## 概要

このリポジトリは、UCI Individual Household Electric Power Consumption データを用いて、家庭用電力使用量の時系列予測を行い、さらに Topological Data Analysis（TDA）による特徴量を追加した場合の予測性能への影響を検証したものです。

分析は大きく2段階で構成されています。

1. 通常の時系列解析・特徴量エンジニアリングによる予測モデルの構築
2. Sliding Window / Takens Embedding と Persistent Homology によるTDA特徴量の作成・評価

通常特徴量のみのモデルを比較基準として作成したうえで、TDA特徴量を追加したモデルと同一条件で比較し、TDA特徴量が予測性能改善に寄与するかを検証しています。

---

## 分析の目的

本プロジェクトの目的は、単に予測精度の高いモデルを作ることではなく、時系列データに対してTDA特徴量を実装し、通常特徴量との比較によりその有効性を事実ベースで評価することです。

具体的には、以下を確認します。

- 家庭用電力使用量データを時系列データとして前処理できるか
- ラグ特徴量・移動平均特徴量・カレンダー特徴量を用いた通常の予測モデルを構築できるか
- 1変量時系列を Sliding Window / Takens Embedding により点群化できるか
- 点群に対して Persistent Homology を計算できるか
- Persistence Diagram / Barcode からTDA特徴量を作成できるか
- Rolling Window によって時点ごとのTDA特徴量を作成できるか
- 通常特徴量とTDA特徴量を結合し、モデル性能を比較できるか
- TDA特徴量の追加が予測性能に有効だったかを評価できるか

---

## 使用データ

- データ名: UCI Individual Household Electric Power Consumption
- 使用ファイル: `household_power_consumption.txt`
- 元データの頻度: 1分間隔
- 主な分析対象: `Global_active_power`
- 予測対象: `Global_active_power` の日次平均

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

本分析では、通常の時系列予測および1変量TDA特徴量作成の対象として、`Global_active_power` の日次平均系列を使用しています。

> 元データはサイズが大きいため、GitHubには直接配置せず、`data/` フォルダに各自で配置する前提です。

---


### 各notebookの位置づけ

| notebook | 役割 |
|---|---|
| `01_time_series_forecasting.ipynb` | 通常の時系列解析と通常特徴量モデルの構築 |
| `02_tda_feature_engineering_and_evaluation.ipynb` | TDA特徴量の作成、通常特徴量との結合、TDA特徴量の有効性評価 |

### 詳細README

各notebookの詳細は以下に分けて整理しています。

- `docs/README_01_time_series_forecasting.md`
- `docs/README_02_tda_feature_engineering_and_evaluation.md`

---

## 分析1: 通常の時系列予測

対象notebook:

```text
notebooks/01_time_series_forecasting.ipynb
```

### 実施内容

通常の時系列解析として、以下を実装しています。

1. 目的設定
2. データの読み込み・確認
3. `Date` と `Time` の結合による datetime index の作成
4. 数値列の型変換
5. 欠損値・重複時刻・時系列間隔の確認
6. `Global_active_power` の日次平均系列の作成
7. 可視化による傾向把握
8. トレンド・季節性の分解
9. 時系列順の train / test 分割
10. ベースラインモデルの作成
11. 指数平滑化系モデルの作成
12. 通常特徴量を用いた機械学習モデルの作成
13. 評価指標によるモデル比較
14. 残差分析
15. 結果の解釈

### 作成した通常特徴量

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

### 比較したモデル

| 種類 | モデル |
|---|---|
| ベースライン | ナイーブ予測、季節ナイーブ予測、平均値予測、移動平均予測、ドリフト予測 |
| 指数平滑化系 | Simple Exponential Smoothing、Holt、Holt-Winters |
| 機械学習 | LinearRegression、RandomForest、GradientBoosting |

### 通常特徴量モデルの結果

全モデルを比較した結果、通常特徴量を用いた `GradientBoosting` が、MAE、RMSE、MAPE、MASE のすべてで最良となりました。

| 順位 | model | type | MAE | RMSE | MAPE | MASE |
|---:|---|---|---:|---:|---:|---:|
| 1 | GradientBoosting | ML | 0.173594 | 0.241535 | 0.187738 | 0.613682 |
| 2 | RandomForest | ML | 0.185017 | 0.248096 | 0.211194 | 0.654064 |
| 3 | LinearRegression | ML | 0.185454 | 0.253395 | 0.212178 | 0.655610 |
| 4 | 移動平均予測 | baseline | 0.249951 | 0.320676 | 0.346069 | 0.883617 |
| 5 | 平均値予測 | baseline | 0.257438 | 0.330300 | 0.363286 | 0.910085 |
| 6 | 季節ナイーブ予測 | baseline | 0.267271 | 0.347744 | 0.291348 | 0.944847 |
| 7 | ドリフト予測 | baseline | 0.280566 | 0.352588 | 0.385887 | 0.991846 |
| 8 | ナイーブ予測 | baseline | 0.390970 | 0.469802 | 0.557871 | 1.382141 |
| 9 | HoltWinters(week) | smoothing | 0.403454 | 0.488563 | 0.572144 | 1.426273 |
| 10 | Holt | smoothing | 0.438865 | 0.514862 | 0.616438 | 1.551458 |
| 11 | SimpleSmoothing | smoothing | 0.469151 | 0.544725 | 0.654711 | 1.658522 |

`GradientBoosting` の MASE は `0.613682` であり、1未満でした。  
このため、少なくとも今回の評価期間では、ナイーブ予測よりも有効な予測ができていたと判断できます。

---

## 分析2: TDA特徴量の作成と有効性評価

対象notebook:

```text
notebooks/02_tda_feature_engineering_and_evaluation.ipynb
```

### 実施内容

TDA特徴量の作成と評価として、以下を実装しています。

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
11. モデル学習: 通常特徴量のみ vs 通常特徴量 + TDA特徴量
12. 評価: TDA特徴量の有効性確認
13. 結果の解釈

### TDAに用いる系列

TDA対象系列として、`Global_active_power` の日次平均系列を使用しました。

理由は以下です。

- 電力使用量の中心的な変数である
- 通常の時系列予測でも予測対象として使用している
- 1変量時系列として Sliding Window / Takens Embedding に適用しやすい
- 通常特徴量モデルとの比較対象を揃えられる

### Sliding Window / Takens Embedding の設定

| 項目 | 値 |
|---|---:|
| `embedding_dim` | 30 |
| `delay` | 1 |
| `stride` | 1 |
| PH計算用点群数 | 500 |

この設定では、1つの点が「30日分の標準化された電力使用量パターン」を表します。

### Persistent Homology の計算

点群に対して Vietoris-Rips 複体に基づく Persistent Homology を計算しました。

| ホモロジー次元 | 意味 |
|---|---|
| H0 | 点群の連結成分の統合過程 |
| H1 | ループ構造の候補 |

実装では `giotto-tda` の `VietorisRipsPersistence` を基本とし、環境によって利用できない場合には `ripser` を使用するフォールバックを用意しています。

### Persistent Homology の要約

| dimension | n_points | max_lifetime | mean_lifetime | median_lifetime | total_lifetime |
|---:|---:|---:|---:|---:|---:|
| H0 | 499 | 7.9137 | 3.4830 | 3.2088 | 1738.0052 |
| H1 | 426 | 0.6816 | 0.1436 | 0.1151 | 61.1528 |

H1特徴が検出されているため、今回の点群化設定では、ループ構造の候補が存在することが確認されました。

ただし、H1の最大 lifetime が合計 lifetime に占める割合は大きくなく、特定の1本のループだけが圧倒的に支配的というより、複数のループ候補が分散して存在している可能性があります。

---

## 作成したTDA特徴量

Persistent Diagram / Barcode から、以下のような数値特徴量を作成しました。

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
| `H1_lifetime_top1`〜`H1_lifetime_top5` | H1 lifetime 上位値 |
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

---

## Rolling Window TDA特徴量

系列全体に対する1行のTDA特徴量だけでは、日次予測モデルにそのまま投入できません。  
そのため、Rolling Window によって、各時点で利用可能なTDA特徴量を作成しました。

### Rolling Window の設定

| 項目 | 値 |
|---|---:|
| rolling window size | 180日 |
| rolling step | 7日 |
| `embedding_dim` | 30 |
| `delay` | 1 |
| `stride` | 1 |
| PH計算用点群数 | 150 |

### 作成されたRolling TDA特徴量

| 項目 | 値 |
|---|---:|
| 行数 | 181 |
| 列数 | 34 |

各行は、ある rolling window の終了時点までの時系列パターンから作成されたTDA特徴量を表します。

---

## 通常特徴量とTDA特徴量の結合

通常特徴量には、分析1と同様に、ラグ特徴量・移動平均特徴量・曜日・月・週末フラグを使用しました。

TDA特徴量は、rolling window の終了日そのものではなく、翌日以降に利用可能な特徴量として扱いました。  
これにより、目的変数を予測する際に未来情報が混入しないようにしています。

### 結合後データ

| 項目 | 値 |
|---|---:|
| 結合前の通常特徴量データ | 1428行 × 14列 |
| TDA結合直後 | 1428行 × 49列 |
| TDA利用可能行のみ | 1262行 × 49列 |
| 使用期間 | 2007-06-14 〜 2010-11-26 |

### train / test 分割

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

> TDA特徴量との比較では、Rolling Window TDA特徴量が利用可能な期間に揃えて評価しています。  
> そのため、分析1の全体モデル比較とは評価期間が完全には一致しません。

---

## 通常特徴量のみ vs 通常特徴量 + TDA特徴量

同じ train / test 期間で、通常特徴量のみのモデルと、通常特徴量にTDA特徴量を追加したモデルを比較しました。

### 比較したモデル

| モデル | 役割 |
|---|---|
| Ridge | 線形モデルの比較対象 |
| RandomForest | 非線形モデル |
| GradientBoosting | 非線形モデル |

### 評価結果

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

---

## TDA特徴量の有効性確認

TDA特徴量の追加効果を、同じモデル同士で比較しました。

| モデル | RMSE改善率 | MAE改善率 | RMSE改善 | MAE改善 |
|---|---:|---:|---|---|
| GradientBoosting | -0.10% | -0.60% | False | False |
| RandomForest | -0.14% | -1.84% | False | False |
| Ridge | -3.74% | -6.44% | False | False |

今回の設定では、3モデルすべてでTDA特徴量追加後にRMSE・MAEの改善は確認されませんでした。

したがって、今回のデータ・前処理・TDAパラメータ・特徴量設計・モデル設定の範囲では、TDA特徴量の追加によって明確に予測性能が改善したとは言えません。

### TDA特徴量重要度

木ベースモデルでは、TDA特徴量にも一定の特徴量重要度が確認されました。

| モデル | TDA特徴量重要度割合 |
|---|---:|
| RandomForest | 約9.30% |
| GradientBoosting | 約6.78% |

ただし、特徴量重要度があることと、予測性能が改善することは同じではありません。  
今回の結果では、TDA特徴量はモデル内部で一部利用されていたものの、最終的な予測精度の改善にはつながりませんでした。

### 日次絶対誤差の変化

`GradientBoosting` では、TDA追加によって日次絶対誤差が改善した日もありました。

| 項目 | 値 |
|---|---:|
| TDA追加で日次絶対誤差が改善した割合 | 45.85% |

一部の日ではTDA特徴量の追加により絶対誤差が小さくなりましたが、test期間全体の平均的な評価指標では改善しませんでした。

---

## 結論

本プロジェクトでは、UCI Individual Household Electric Power Consumption データを用いて、日次平均の `Global_active_power` を予測しました。

まず、通常の時系列解析として、ラグ特徴量・移動平均特徴量・曜日・月・週末フラグを用いた予測モデルを構築しました。  
その結果、通常特徴量のみの `GradientBoosting` が最良の性能を示しました。

次に、`Global_active_power` の日次平均系列に対して Sliding Window / Takens Embedding を適用し、点群を作成しました。  
その点群に対して Persistent Homology を計算し、Persistence Diagram / Barcode を確認したうえで、TDA特徴量を作成しました。

さらに、Rolling Window によって時点ごとのTDA特徴量を作成し、通常特徴量と結合して予測モデルに投入しました。

評価の結果、今回の設定では、通常特徴量のみの `GradientBoosting` が最良であり、TDA特徴量を追加したモデルでは、RMSE、MAE、MASE、R²の明確な改善は確認されませんでした。

一方で、TDA特徴量は木ベースモデルにおいて一定の特徴量重要度を持っていたため、時系列の形状情報をある程度表現していた可能性はあります。

本プロジェクトの結論は以下です。

> 今回のデータ、前処理、Sliding Window / Takens Embedding の設定、Rolling Window の設定、TDA特徴量設計、モデル設定のもとでは、TDA特徴量の追加による明確な予測性能改善は確認できなかった。  
> ただし、時系列データを点群化し、Persistent Homology から特徴量を作成し、通常特徴量モデルに結合して評価する一連の実装手順は構築できた。

---

## 使用ライブラリ

主な使用ライブラリは以下です。

```text
numpy
pandas
matplotlib
seaborn
scikit-learn
statsmodels
sktime
giotto-tda
ripser
tqdm
```

`giotto-tda` が使用できない環境では、Persistent Homology の計算に `ripser` を使うフォールバックを用意しています。

---

## 実行時に生成される主なファイル

TDA notebook 実行時には、主に以下のCSVファイルが生成されます。

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

GitHub上で整理する場合は、これらのCSVを `results/tables/` に保存する構成にすると見通しが良くなります。  
図表ファイルを保存する場合は、`results/figures/` に配置する想定です。

---

## 今後の課題

TDA特徴量の有効性をより詳しく検証するためには、以下の追加実験が必要です。

### 1. Sliding Window / Takens Embedding のパラメータ変更

- `embedding_dim`
- `delay`
- `stride`
- PH計算に使う点群数

### 2. Rolling Window の幅の変更

- 短期的な変動を捉える window
- 中期的な変動を捉える window
- 長期的な変動を捉える window

### 3. TDA特徴量の種類の追加

- lifetime の最大値・平均・合計
- persistence entropy
- Betti curve
- persistence image
- persistence landscape
- Wasserstein / bottleneck 系の距離特徴量

### 4. モデルの変更

- Ridge
- RandomForest
- GradientBoosting
- XGBoost
- LightGBM

### 5. タスク設定の変更

- 1日先予測
- 数日先予測
- 異常検知
- 変動局面の分類
- 予測性能ではなく、局面検出にTDA特徴量を使う設定

---

## このプロジェクトの位置づけ

このプロジェクトは、TDA特徴量を追加すれば必ず予測性能が向上することを示すものではありません。

むしろ、以下を実装・検証したポートフォリオ用プロジェクトです。

- 通常の時系列解析を一通り実装する
- 通常特徴量モデルを比較基準として構築する
- 時系列データを Sliding Window / Takens Embedding によって点群化する
- Persistent Homology を計算する
- Persistence Diagram / Barcode を可視化する
- Persistence Diagram / Barcode からTDA特徴量を作成する
- Rolling Window によって時点ごとのTDA特徴量を作成する
- 通常特徴量とTDA特徴量を結合してモデルに投入する
- TDA特徴量の有効性を事実ベースで評価する

今回の結果ではTDA特徴量による明確な性能改善は確認されませんでしたが、TDAを時系列予測モデルに組み込む一連の実装手順は整理できています。

---

## References / 参考文献

本プロジェクトでは、時系列データに対するSliding Window / Takens Embeddingと、
Persistent Homologyによる特徴量抽出の考え方を理解するため、以下の文献を参考にした。

1. Jose A. Perea and John Harer,  
   *Sliding Windows and Persistence: An Application of Topological Methods to Signal Analysis*,  
   Foundations of Computational Mathematics, 15, 799–838, 2015.  
   https://arxiv.org/abs/1307.6188

2. Jose A. Perea,  
   *Topological Time Series Analysis*,  
   Notices of the American Mathematical Society, 66(5), 686–694, 2019.  
   https://arxiv.org/abs/1812.05143

3. Guillaume Tauzin, Umberto Lupo, Lewis Tunstall, Julian Burella Pérez, Matteo Caorsi, Wojciech Reise, Anibal M. Medina-Mardones, Alberto Dassatti, Kathryn Hess,  
   *giotto-tda: A Topological Data Analysis Toolkit for Machine Learning and Data Exploration*,  
   NeurIPS 2020 Workshop on Topological Data Analysis and Beyond, 2021.  
   https://arxiv.org/abs/2004.02551

4. Robert Ghrist,  
   *Barcodes: The Persistent Topology of Data*,  
   Bulletin of the American Mathematical Society, 45(1), 61–75, 2008.  
   https://www.ams.org/journals/bull/2008-45-01/S0273-0979-07-01191-3/

### 補足
TDA部分では、時系列データをSliding Window / Takens Embeddingによって点群化し、
その点群に対してPersistent Homologyを計算する流れを採用した。
この考え方は、Perea and Harer (2015) および Perea (2019) の
Topological Time Series Analysis の枠組みを参考にしている。
Persistent Homologyの計算およびPersistence Diagram / Barcodeの可視化には、
PythonでTDAを機械学習パイプラインに組み込みやすい giotto-tda を利用した。
