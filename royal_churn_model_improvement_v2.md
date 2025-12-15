# ロイヤル顧客離脱モデル改善提案（詳細版）

## 1. 現状の整理

### 1.1 データの特性
- 本番データの離脱率: 66%（flag_1m=1）
- ロイヤル顧客: 約6万人（1年間に5回以上購入）
- 全購入顧客: 70万人/年
- データ粒度: 1顧客1レコード

### 1.2 特徴量の期間定義
- R01: 1ヶ月前
- R02〜R06: 2〜6ヶ月前
- A2: 1年前

### 1.3 運用制約
- Precisionの最低ライン: 70%
- 配布人数上限: なし
- コスト上限: なし

### 1.4 現状の問題点
離脱率が66%と高いため、何も考えず全員を「離脱」と予測してもPrecision=66%になります。現状のモデル（Precision 70-72%）は、ベースラインからわずか4-6ポイントの改善に留まっています。

---

## 2. 改善戦略の全体像

**目標**: Precision 70%以上を維持しながら、Recallを最大化する

**アプローチ**:
1. 閾値最適化（Precision≥70%制約下でRecall最大化）
2. class_weightの調整（False Positive削減）
3. 特徴量エンジニアリング（変化・傾向の強調）
4. モデルチューニング（PR-AUC最適化）
5. 確率校正（閾値の安定化）

---

## 3. 具体的な実装コード

### 3.1 【最優先】Precision≥70%制約下での閾値最適化

```python
import numpy as np
import pandas as pd
from sklearn.metrics import precision_score, recall_score, f1_score

def find_optimal_threshold_with_precision_constraint(y_true, y_prob, min_precision=0.70):
    """
    Precision≥min_precision を満たす範囲で、Recallが最大となる閾値を探索
    """
    thresholds = np.arange(0.30, 0.96, 0.01)
    results = []
    
    for threshold in thresholds:
        y_pred = (y_prob >= threshold).astype(int)
        
        # 予測が全て0の場合はスキップ
        if y_pred.sum() == 0:
            continue
            
        precision = precision_score(y_true, y_pred)
        recall = recall_score(y_true, y_pred)
        f1 = f1_score(y_true, y_pred)
        predicted_positive = y_pred.sum()
        
        results.append({
            'threshold': threshold,
            'precision': precision,
            'recall': recall,
            'f1': f1,
            'predicted_positive': predicted_positive,
            'predicted_positive_ratio': predicted_positive / len(y_pred)
        })
    
    results_df = pd.DataFrame(results)
    
    # Precision≥70%を満たす候補をフィルタ
    valid_results = results_df[results_df['precision'] >= min_precision]
    
    if len(valid_results) == 0:
        print(f"警告: Precision≥{min_precision}を満たす閾値が見つかりません")
        return results_df, None
    
    # Recallが最大の閾値を選択
    best_idx = valid_results['recall'].idxmax()
    best_threshold = valid_results.loc[best_idx, 'threshold']
    
    print(f"=== Precision≥{min_precision} 制約下での最適閾値 ===")
    print(f"閾値: {best_threshold:.2f}")
    print(f"Precision: {valid_results.loc[best_idx, 'precision']:.4f}")
    print(f"Recall: {valid_results.loc[best_idx, 'recall']:.4f}")
    print(f"F1: {valid_results.loc[best_idx, 'f1']:.4f}")
    print(f"施策対象人数: {valid_results.loc[best_idx, 'predicted_positive']:.0f}")
    print(f"施策対象比率: {valid_results.loc[best_idx, 'predicted_positive_ratio']:.2%}")
    
    return results_df, best_threshold

# 使用例
results_df, optimal_threshold = find_optimal_threshold_with_precision_constraint(
    y_test, y_pred_2_prob, min_precision=0.70
)

# 結果の可視化
print("\n=== 閾値別の性能一覧 ===")
print(results_df[results_df['precision'] >= 0.68].to_string(index=False))
```

### 3.2 上位N人運用（可変N）の実装

```python
def find_optimal_n_with_precision_constraint(y_true, y_prob, min_precision=0.70):
    """
    スコア上位から順に対象を増やし、累積Precisionが70%を割る直前まで配布
    """
    # スコアの高い順にソート
    sorted_indices = np.argsort(y_prob)[::-1]
    y_true_sorted = y_true.iloc[sorted_indices].values if hasattr(y_true, 'iloc') else y_true[sorted_indices]
    
    results = []
    cumsum_positive = 0
    
    for i, is_positive in enumerate(y_true_sorted, 1):
        cumsum_positive += is_positive
        precision_at_n = cumsum_positive / i
        
        # 100人刻みで記録（計算効率のため）
        if i % 100 == 0 or i == len(y_true_sorted):
            results.append({
                'top_n': i,
                'precision_at_n': precision_at_n,
                'recall_at_n': cumsum_positive / y_true.sum(),
                'cumsum_positive': cumsum_positive
            })
    
    results_df = pd.DataFrame(results)
    
    # Precision≥70%を満たす最大のN
    valid_results = results_df[results_df['precision_at_n'] >= min_precision]
    
    if len(valid_results) == 0:
        print(f"警告: Precision≥{min_precision}を満たすNが見つかりません")
        return results_df, None
    
    optimal_n = valid_results['top_n'].max()
    optimal_row = valid_results[valid_results['top_n'] == optimal_n].iloc[0]
    
    print(f"=== Precision≥{min_precision} 制約下での最適配布人数 ===")
    print(f"配布人数: {optimal_n:,}人")
    print(f"Precision@N: {optimal_row['precision_at_n']:.4f}")
    print(f"Recall@N: {optimal_row['recall_at_n']:.4f}")
    print(f"捕捉できる離脱者数: {optimal_row['cumsum_positive']:.0f}人")
    
    return results_df, optimal_n

# 使用例
n_results_df, optimal_n = find_optimal_n_with_precision_constraint(
    y_test, y_pred_2_prob, min_precision=0.70
)
```

### 3.3 Lift（改善度）の計算

```python
def calculate_lift_chart(y_true, y_prob, n_deciles=10):
    """
    Lift曲線を計算（上位何%の離脱率が全体の何倍か）
    """
    df = pd.DataFrame({'y_true': y_true, 'y_prob': y_prob})
    df = df.sort_values('y_prob', ascending=False).reset_index(drop=True)
    
    # 10分位に分割
    df['decile'] = pd.qcut(df.index, n_deciles, labels=False) + 1
    
    baseline_rate = y_true.mean()
    
    lift_table = df.groupby('decile').agg({
        'y_true': ['sum', 'count', 'mean']
    }).reset_index()
    lift_table.columns = ['decile', 'churners', 'total', 'churn_rate']
    lift_table['lift'] = lift_table['churn_rate'] / baseline_rate
    lift_table['cumulative_churners'] = lift_table['churners'].cumsum()
    lift_table['cumulative_total'] = lift_table['total'].cumsum()
    lift_table['cumulative_precision'] = lift_table['cumulative_churners'] / lift_table['cumulative_total']
    lift_table['cumulative_recall'] = lift_table['cumulative_churners'] / y_true.sum()
    
    print(f"=== Lift表（ベースライン離脱率: {baseline_rate:.2%}）===")
    print(lift_table.to_string(index=False))
    
    return lift_table

# 使用例
lift_table = calculate_lift_chart(y_test.values, y_pred_2_prob)
```

---

## 4. class_weightの調整（False Positive削減）

離脱率66%で「1が多数派」のため、0クラス（非離脱）の重みを上げてFalse Positiveを減らします。

```python
from sklearn.linear_model import LogisticRegression
from sklearn.ensemble import RandomForestClassifier
import lightgbm as lgb

# ロジスティック回帰（0クラスを重くする）
def train_with_class_weight(x_train, y_train, x_test, y_test, weight_0=2.0):
    """
    class_weightを調整してモデルを学習
    """
    model = LogisticRegression(
        random_state=0,
        class_weight={0: weight_0, 1: 1.0},
        max_iter=1000
    )
    model.fit(x_train, y_train)
    
    y_prob = model.predict_proba(x_test)[:, 1]
    
    # Precision≥70%制約下での最適閾値を探索
    results_df, optimal_threshold = find_optimal_threshold_with_precision_constraint(
        y_test, y_prob, min_precision=0.70
    )
    
    return model, y_prob, results_df, optimal_threshold

# 重みを探索
for weight_0 in [1.5, 2.0, 2.5, 3.0, 4.0]:
    print(f"\n=== class_weight={{0: {weight_0}, 1: 1.0}} ===")
    model, y_prob, results_df, optimal_threshold = train_with_class_weight(
        x_train_scaled, y_train, x_test_scaled, y_test, weight_0=weight_0
    )
```

### LightGBMでのscale_pos_weight調整

```python
# LightGBM（scale_pos_weightで調整）
# 離脱が多数派(66%)なので、scale_pos_weight < 1.0 にする
n_negative = (y_train == 0).sum()
n_positive = (y_train == 1).sum()

# scale_pos_weightを探索
for spw in [0.3, 0.4, 0.5, 0.6, 0.7]:
    print(f"\n=== scale_pos_weight={spw} ===")
    
    model_lgb = lgb.LGBMClassifier(
        objective='binary',
        n_estimators=500,
        learning_rate=0.05,
        num_leaves=31,
        max_depth=7,
        min_data_in_leaf=50,
        scale_pos_weight=spw,
        random_state=0,
        verbose=-1
    )
    model_lgb.fit(x_train_scaled, y_train)
    
    y_prob_lgb = model_lgb.predict_proba(x_test_scaled)[:, 1]
    
    results_df, optimal_threshold = find_optimal_threshold_with_precision_constraint(
        y_test, y_prob_lgb, min_precision=0.70
    )
```

---

## 5. 特徴量エンジニアリング（変化・傾向の強調）

ロイヤル顧客は「購入頻度が元々高い」ので、絶対値より「変化」が重要です。

```python
def create_trend_features(df):
    """
    変化・傾向を強調した特徴量を作成
    """
    df = df.copy()
    
    # === 1. 購買頻度の変化 ===
    # 直近3ヶ月 vs 過去3ヶ月
    df['order_cnt_recent'] = df['order_cnt_r01'] + df['order_cnt_r02'] + df['order_cnt_r03']
    df['order_cnt_past'] = df['order_cnt_r04'] + df['order_cnt_r05'] + df['order_cnt_r06']
    df['order_cnt_change'] = df['order_cnt_recent'] - df['order_cnt_past']
    df['order_cnt_change_ratio'] = df['order_cnt_recent'] / (df['order_cnt_past'] + 1)
    
    # 直近1ヶ月の寄与率（急激な落ち込み検出）
    df['order_cnt_r01_ratio'] = df['order_cnt_r01'] / (df['order_cnt_a2'] + 1)
    
    # === 2. 購買金額の変化 ===
    df['order_amt_recent'] = df['order_price_amt_r01'] + df['order_price_amt_r02'] + df['order_price_amt_r03']
    df['order_amt_past'] = df['order_price_amt_r04'] + df['order_price_amt_r05'] + df['order_price_amt_r06']
    df['order_amt_change'] = df['order_amt_recent'] - df['order_amt_past']
    df['order_amt_change_ratio'] = df['order_amt_recent'] / (df['order_amt_past'] + 1)
    
    # === 3. 購入間隔の異常検出 ===
    # last_diff（最終購入からの日数）が平均購入間隔（days_diff）の何倍か
    df['purchase_delay_ratio'] = df['last_diff'] / (df['days_diff'] + 1)
    
    # === 4. Web行動の変化 ===
    # セッション数の変化
    df['session_cnt_recent'] = df['session_cnt_r01'] + df['session_cnt_r02'] + df['session_cnt_r03']
    df['session_cnt_past'] = df['session_cnt_r04'] + df['session_cnt_r05'] + df['session_cnt_r06']
    df['session_cnt_change_ratio'] = df['session_cnt_recent'] / (df['session_cnt_past'] + 1)
    
    # PV数の変化
    df['pv_cnt_recent'] = df['pv_cnt_r01'] + df['pv_cnt_r02'] + df['pv_cnt_r03']
    df['pv_cnt_past'] = df['pv_cnt_r04'] + df['pv_cnt_r05'] + df['pv_cnt_r06']
    df['pv_cnt_change_ratio'] = df['pv_cnt_recent'] / (df['pv_cnt_past'] + 1)
    
    # === 5. 閲覧と購買の乖離（見てるが買わない）===
    df['pv_per_session'] = df['pv_cnt_a2'] / (df['session_cnt_a2'] + 1)
    df['order_per_session'] = df['order_cnt_a2'] / (df['session_cnt_a2'] + 1)
    df['browse_buy_gap'] = df['pv_cnt_a2'] / (df['order_cnt_a2'] + 1)
    
    # 直近の乖離
    df['browse_buy_gap_recent'] = df['pv_cnt_recent'] / (df['order_cnt_recent'] + 1)
    
    # === 6. 値引き依存度 ===
    df['discount_rate'] = df['pricedown_amt_a2'] / (df['order_price_amt_a2'] + 1)
    df['coupon_dependency'] = df['coupon_cnt'] / (df['order_cnt_a2'] + 1)
    df['sale_ratio'] = df['sale_cnt'] / (df['order_cnt_a2'] + 1)
    
    # 直近の値引き依存度の変化
    df['discount_amt_recent'] = df['pricedown_amt_r01'] + df['pricedown_amt_r02'] + df['pricedown_amt_r03']
    df['discount_amt_past'] = df['pricedown_amt_r04'] + df['pricedown_amt_r05'] + df['pricedown_amt_r06']
    df['discount_dependency_change'] = (df['discount_amt_recent'] / (df['order_amt_recent'] + 1)) - \
                                        (df['discount_amt_past'] / (df['order_amt_past'] + 1))
    
    # === 7. 顧客ライフサイクル ===
    # 顧客年齢（初回購入からの日数）のカテゴリ化
    df['customer_tenure_months'] = df['ftime_diff'] / 30
    
    # === 8. 単価の変化 ===
    df['avg_price_recent'] = df['order_amt_recent'] / (df['order_cnt_recent'] + 1)
    df['avg_price_past'] = df['order_amt_past'] / (df['order_cnt_past'] + 1)
    df['avg_price_change_ratio'] = df['avg_price_recent'] / (df['avg_price_past'] + 1)
    
    return df

# 使用例
df_with_features = create_trend_features(df_target_flag_1m)
```

---

## 6. LightGBMのPR-AUC最適化チューニング

```python
import lightgbm as lgb
from sklearn.model_selection import train_test_split, StratifiedKFold
from sklearn.metrics import average_precision_score

# 検証データを分離
x_tr, x_val, y_tr, y_val = train_test_split(
    x_train_scaled, y_train, test_size=0.2, random_state=0, stratify=y_train
)

# PR-AUC最適化でLightGBMを学習
model_lgb_tuned = lgb.LGBMClassifier(
    objective='binary',
    n_estimators=1000,
    learning_rate=0.03,
    num_leaves=31,
    max_depth=7,
    min_data_in_leaf=50,
    reg_alpha=0.1,
    reg_lambda=0.1,
    feature_fraction=0.8,
    bagging_fraction=0.8,
    bagging_freq=5,
    scale_pos_weight=0.5,  # 離脱が多数派なので < 1.0
    random_state=0,
    verbose=-1
)

# Early Stoppingで学習
model_lgb_tuned.fit(
    x_tr, y_tr,
    eval_set=[(x_val, y_val)],
    eval_metric='average_precision',
    callbacks=[lgb.early_stopping(stopping_rounds=50, verbose=True)]
)

# 評価
y_prob_lgb_tuned = model_lgb_tuned.predict_proba(x_test_scaled)[:, 1]
ap_score = average_precision_score(y_test, y_prob_lgb_tuned)
print(f"PR-AUC (Average Precision): {ap_score:.4f}")

# Precision≥70%制約下での最適閾値
results_df, optimal_threshold = find_optimal_threshold_with_precision_constraint(
    y_test, y_prob_lgb_tuned, min_precision=0.70
)
```

### ハイパーパラメータ探索（Optuna使用）

```python
# pip install optuna
import optuna
from sklearn.model_selection import cross_val_score
from sklearn.metrics import make_scorer, average_precision_score

def objective(trial):
    params = {
        'objective': 'binary',
        'n_estimators': 500,
        'learning_rate': trial.suggest_float('learning_rate', 0.01, 0.1),
        'num_leaves': trial.suggest_int('num_leaves', 15, 63),
        'max_depth': trial.suggest_int('max_depth', 4, 10),
        'min_data_in_leaf': trial.suggest_int('min_data_in_leaf', 20, 100),
        'reg_alpha': trial.suggest_float('reg_alpha', 0, 1.0),
        'reg_lambda': trial.suggest_float('reg_lambda', 0, 1.0),
        'feature_fraction': trial.suggest_float('feature_fraction', 0.6, 1.0),
        'bagging_fraction': trial.suggest_float('bagging_fraction', 0.6, 1.0),
        'scale_pos_weight': trial.suggest_float('scale_pos_weight', 0.3, 0.7),
        'random_state': 0,
        'verbose': -1
    }
    
    model = lgb.LGBMClassifier(**params)
    
    # PR-AUCで評価
    ap_scorer = make_scorer(average_precision_score, needs_proba=True)
    cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=0)
    scores = cross_val_score(model, x_train_scaled, y_train, cv=cv, scoring=ap_scorer)
    
    return scores.mean()

# 最適化実行
study = optuna.create_study(direction='maximize')
study.optimize(objective, n_trials=100, show_progress_bar=True)

print(f"Best PR-AUC: {study.best_value:.4f}")
print(f"Best params: {study.best_params}")
```

---

## 7. 確率校正（閾値の安定化）

```python
from sklearn.calibration import CalibratedClassifierCV

# 確率校正（Platt Scaling）
model_calibrated = CalibratedClassifierCV(
    model_lgb_tuned,
    method='sigmoid',  # Platt Scaling
    cv=5
)
model_calibrated.fit(x_train_scaled, y_train)

y_prob_calibrated = model_calibrated.predict_proba(x_test_scaled)[:, 1]

# 校正前後の比較
print("=== 校正前 ===")
results_before, _ = find_optimal_threshold_with_precision_constraint(
    y_test, y_prob_lgb_tuned, min_precision=0.70
)

print("\n=== 校正後 ===")
results_after, _ = find_optimal_threshold_with_precision_constraint(
    y_test, y_prob_calibrated, min_precision=0.70
)
```

---

## 8. 評価指標の可視化

```python
import matplotlib.pyplot as plt
from sklearn.metrics import precision_recall_curve, average_precision_score

def plot_pr_curve_with_threshold(y_true, y_prob, min_precision=0.70):
    """
    PR曲線とPrecision≥70%ラインを可視化
    """
    precision_vals, recall_vals, thresholds = precision_recall_curve(y_true, y_prob)
    ap_score = average_precision_score(y_true, y_prob)
    
    fig, axes = plt.subplots(1, 2, figsize=(14, 5))
    
    # PR曲線
    axes[0].plot(recall_vals, precision_vals, 'b-', linewidth=2, label=f'PR curve (AP = {ap_score:.4f})')
    axes[0].axhline(y=min_precision, color='r', linestyle='--', label=f'Precision = {min_precision}')
    axes[0].axhline(y=0.66, color='gray', linestyle=':', label='Baseline (66%)')
    axes[0].set_xlabel('Recall', fontsize=12)
    axes[0].set_ylabel('Precision', fontsize=12)
    axes[0].set_title('Precision-Recall Curve', fontsize=14)
    axes[0].legend(loc='lower left')
    axes[0].grid(True, alpha=0.3)
    axes[0].set_xlim([0, 1])
    axes[0].set_ylim([0.5, 1])
    
    # 閾値別のPrecision/Recall
    axes[1].plot(thresholds, precision_vals[:-1], 'b-', label='Precision')
    axes[1].plot(thresholds, recall_vals[:-1], 'g-', label='Recall')
    axes[1].axhline(y=min_precision, color='r', linestyle='--', label=f'Precision = {min_precision}')
    axes[1].set_xlabel('Threshold', fontsize=12)
    axes[1].set_ylabel('Score', fontsize=12)
    axes[1].set_title('Precision/Recall vs Threshold', fontsize=14)
    axes[1].legend(loc='center right')
    axes[1].grid(True, alpha=0.3)
    
    plt.tight_layout()
    plt.savefig('pr_analysis.png', dpi=150)
    plt.show()
    
    return ap_score

ap_score = plot_pr_curve_with_threshold(y_test, y_prob_lgb_tuned, min_precision=0.70)
```

---

## 9. 推奨する実装順序

1. **閾値最適化の実装**（即効性あり）
   - `find_optimal_threshold_with_precision_constraint()` を実装
   - Precision≥70%制約下でRecallが最大となる閾値を採用

2. **特徴量エンジニアリング**（中期的改善）
   - `create_trend_features()` で変化・傾向特徴量を追加
   - 特に `order_cnt_change_ratio`, `purchase_delay_ratio`, `browse_buy_gap` が効きやすい

3. **class_weight/scale_pos_weightの調整**（FP削減）
   - ロジスティック回帰: `class_weight={0: 2.0, 1: 1.0}`
   - LightGBM: `scale_pos_weight=0.5` 程度から探索

4. **LightGBMのチューニング**（PR-AUC最適化）
   - Early Stopping + 正則化パラメータ調整
   - Optunaでの自動探索

5. **確率校正**（閾値の安定化）
   - 月次運用で閾値がブレにくくなる

---

## 10. 期待される改善効果

現状（Precision 70-72%、Recall 89-95%）から、以下の改善が期待できます：

- **閾値最適化のみ**: Precision 70%維持でRecall 80-85%程度に調整可能
- **特徴量追加 + チューニング**: PR-AUC改善により、同一Precision条件でRecall +3-5%
- **総合的な改善**: Precision 70%維持でRecall 85-90%、または Precision 75%でRecall 75-80%

最終的には、「Precision 70%を維持しながら、どれだけ多くの離脱者を捕捉できるか」という運用最適化の問題として、PRカーブ上で最適な運用点を選択することが重要です。
