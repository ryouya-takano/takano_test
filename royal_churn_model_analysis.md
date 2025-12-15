# ロイヤル顧客離脱モデル分析レポート

## 1. 現状の分析結果

### 1.1 データ構造
- 64カラム（cust_no + 62特徴量 + flag_1m）
- 本番データ: 約30万件予定
- 目的変数: flag_1m（1=離脱、0=非離脱）

### 1.2 現在のモデル性能（クラス1=離脱）

| モデル | Precision | Recall | F1-score |
|--------|-----------|--------|----------|
| 決定木 | 0.70 | 0.68 | 0.69 |
| ロジスティック回帰 | 0.71 | 0.95 | 0.81 |
| ランダムフォレスト | 0.71 | 0.91 | 0.80 |
| XGBoost | 0.72 | 0.89 | 0.79 |
| LightGBM | 0.72 | 0.92 | 0.80 |
| ニューラルネットワーク | 0.71 | 0.93 | 0.81 |

### 1.3 問題点の特定

現在のロジスティック回帰モデルでは、テストデータ19,422件中17,355件（約89%）を「離脱」と予測しています。これは「ほぼ全員を離脱と予測している」状態であり、Precisionが70%程度で頭打ちになっている主要因です。

---

## 2. コードレベルの修正提案（優先度順）

### 2.1 【最優先】閾値最適化の実装

現在のコードでは、コメントに「閾値を0.7に上げる」と書いてありますが、実際は`new_threshold = 0.5`のままです。

**修正前:**
```python
# 例: 適合率を重視するため、閾値を 0.7 に上げる
new_threshold = 0.5  # ← コメントと不一致
```

**修正後（閾値スイープの実装）:**
```python
import numpy as np
from sklearn.metrics import precision_score, recall_score, f1_score

# 閾値をスイープしてPrecision/Recall/対象人数を確認
thresholds = np.arange(0.3, 0.96, 0.05)
results = []

for threshold in thresholds:
    y_pred_new = (y_pred_2_prob >= threshold).astype(int)
    precision = precision_score(y_test, y_pred_new)
    recall = recall_score(y_test, y_pred_new)
    f1 = f1_score(y_test, y_pred_new)
    predicted_positive = y_pred_new.sum()  # 施策対象人数
    
    results.append({
        'threshold': threshold,
        'precision': precision,
        'recall': recall,
        'f1': f1,
        'predicted_positive': predicted_positive,
        'predicted_positive_ratio': predicted_positive / len(y_pred_new)
    })

results_df = pd.DataFrame(results)
print(results_df.to_string(index=False))
```

### 2.2 【優先度高】上位N人での評価（Precision@N）

運用が「上位N人にクーポン配布」なら、閾値よりも「スコア上位N人でのPrecision」が直結します。

```python
def precision_at_k(y_true, y_prob, k):
    """上位k人でのPrecisionを計算"""
    # スコアの高い順にソート
    sorted_indices = np.argsort(y_prob)[::-1]
    top_k_indices = sorted_indices[:k]
    top_k_true = y_true.iloc[top_k_indices] if hasattr(y_true, 'iloc') else y_true[top_k_indices]
    return top_k_true.sum() / k

# 例: 上位1万人、2万人、3万人でのPrecisionを確認
for k in [10000, 20000, 30000, 50000]:
    prec_k = precision_at_k(y_test, y_pred_2_prob, k)
    print(f"Precision@{k}: {prec_k:.4f}")
```

### 2.3 【優先度高】train_test_splitにstratifyを追加

```python
# 修正前
x_train, x_test, y_train, y_test = train_test_split(x, y, test_size=0.3, random_state=0)

# 修正後（層化抽出で目的変数の比率を維持）
x_train, x_test, y_train, y_test = train_test_split(
    x, y, test_size=0.3, random_state=0, stratify=y
)
```

### 2.4 【優先度高】XGBoostの引数修正

```python
# 修正前（引数名が間違っている可能性）
model_4 = xgb.XGBClassifier(use_label_encoder=False, eval_metrics='logloss')

# 修正後
model_4 = xgb.XGBClassifier(
    use_label_encoder=False,
    eval_metric='logloss',  # eval_metrics → eval_metric
    random_state=0
)
```

### 2.5 【優先度中】class_weightの調整

False Positive（実際は非離脱なのに離脱と予測）を減らすために、クラス0の重みを上げます。

```python
# ロジスティック回帰
from sklearn.linear_model import LogisticRegression

# 方法1: balanced（自動調整）
model_2 = LogisticRegression(random_state=0, class_weight='balanced', max_iter=1000)

# 方法2: 手動で0クラスを重くする（Precision重視）
model_2 = LogisticRegression(random_state=0, class_weight={0: 2.0, 1: 1.0}, max_iter=1000)

# ランダムフォレスト
model_3 = RandomForestClassifier(random_state=0, class_weight='balanced')

# LightGBM（scale_pos_weightで調整）
# 離脱が多数派の場合、scale_pos_weight < 1.0 にする
n_negative = (y_train == 0).sum()
n_positive = (y_train == 1).sum()
model_5 = lgb.LGBMClassifier(
    importance_type='gain',
    random_state=0,
    scale_pos_weight=n_negative / n_positive  # 例: 0.5程度
)
```

### 2.6 【優先度中】交差検証の実装

```python
from sklearn.model_selection import StratifiedKFold, cross_val_score
from sklearn.metrics import make_scorer, precision_score, average_precision_score

# 5分割の層化交差検証
cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=0)

# Precisionでの評価
precision_scorer = make_scorer(precision_score)
scores = cross_val_score(model_2, x_train_scaled, y_train, cv=cv, scoring=precision_scorer)
print(f"Precision (CV): {scores.mean():.4f} (+/- {scores.std():.4f})")

# PR-AUC（Average Precision）での評価（推奨）
ap_scorer = make_scorer(average_precision_score, needs_proba=True)
ap_scores = cross_val_score(model_2, x_train_scaled, y_train, cv=cv, scoring=ap_scorer)
print(f"PR-AUC (CV): {ap_scores.mean():.4f} (+/- {ap_scores.std():.4f})")
```

---

## 3. 特徴量エンジニアリングの提案

### 3.1 RFM系の比率・傾向特徴量

```python
# 直近/過去の比率（トレンド）
df['order_cnt_trend'] = df['order_cnt_r01'] / (df['order_cnt_a2'] + 1)
df['order_amt_trend'] = df['order_price_amt_r01'] / (df['order_price_amt_a2'] + 1)

# 直近の寄与率
df['recent_order_ratio'] = (df['order_cnt_r01'] + df['order_cnt_r02']) / (df['order_cnt_a2'] + 1)

# 購買頻度の変化（直近3ヶ月 vs 過去3ヶ月）
df['order_cnt_recent'] = df['order_cnt_r01'] + df['order_cnt_r02'] + df['order_cnt_r03']
df['order_cnt_past'] = df['order_cnt_r04'] + df['order_cnt_r05'] + df['order_cnt_r06']
df['order_cnt_change'] = df['order_cnt_recent'] - df['order_cnt_past']
```

### 3.2 Web行動の特徴量

```python
# 1セッションあたりPV（エンゲージメント指標）
df['pv_per_session'] = df['pv_cnt_a2'] / (df['session_cnt_a2'] + 1)

# 疑似CVR（購入/セッション）
df['pseudo_cvr'] = df['order_cnt_a2'] / (df['session_cnt_a2'] + 1)

# 閲覧と購買の乖離（見てるが買わない）
df['browse_buy_gap'] = df['pv_cnt_a2'] / (df['order_cnt_a2'] + 1)
```

### 3.3 値引き依存度

```python
# 値引率
df['discount_rate'] = df['pricedown_amt_a2'] / (df['order_price_amt_a2'] + 1)

# クーポン依存度
df['coupon_dependency'] = df['coupon_cnt'] / (df['order_cnt_a2'] + 1)

# セール購入比率
df['sale_ratio'] = df['sale_cnt'] / (df['order_cnt_a2'] + 1)
```

### 3.4 顧客ライフサイクル

```python
# 顧客年齢（初回購入からの日数）
# ftime_diff は既にあるが、カテゴリ化も有効
df['customer_tenure_category'] = pd.cut(
    df['ftime_diff'],
    bins=[0, 90, 180, 365, 730, float('inf')],
    labels=['new', 'developing', 'established', 'mature', 'veteran']
)

# 購買間隔の安定性
df['purchase_interval_avg'] = df['days_diff']
```

---

## 4. LightGBM/XGBoostのハイパーパラメータチューニング

### 4.1 LightGBMの推奨設定

```python
import lightgbm as lgb
from sklearn.model_selection import GridSearchCV

# パラメータグリッド
param_grid = {
    'num_leaves': [15, 31, 63],
    'max_depth': [5, 7, 10],
    'min_data_in_leaf': [20, 50, 100],
    'reg_alpha': [0, 0.1, 1.0],
    'reg_lambda': [0, 0.1, 1.0],
    'feature_fraction': [0.7, 0.8, 0.9],
    'bagging_fraction': [0.7, 0.8, 0.9],
}

# Precision重視の場合
model_lgb = lgb.LGBMClassifier(
    objective='binary',
    importance_type='gain',
    random_state=0,
    n_estimators=500,
    learning_rate=0.05,
)

# GridSearchCV（簡易版）
from sklearn.model_selection import RandomizedSearchCV

search = RandomizedSearchCV(
    model_lgb,
    param_grid,
    n_iter=50,
    cv=5,
    scoring='precision',  # または 'average_precision'
    random_state=0,
    n_jobs=-1
)
search.fit(x_train_scaled, y_train)
print(f"Best params: {search.best_params_}")
print(f"Best precision: {search.best_score_:.4f}")
```

### 4.2 Early Stoppingの実装

```python
# 検証データを分離
from sklearn.model_selection import train_test_split

x_tr, x_val, y_tr, y_val = train_test_split(
    x_train_scaled, y_train, test_size=0.2, random_state=0, stratify=y_train
)

# Early Stoppingで学習
model_lgb = lgb.LGBMClassifier(
    objective='binary',
    n_estimators=1000,
    learning_rate=0.05,
    num_leaves=31,
    max_depth=7,
    min_data_in_leaf=50,
    reg_alpha=0.1,
    reg_lambda=0.1,
    random_state=0,
)

model_lgb.fit(
    x_tr, y_tr,
    eval_set=[(x_val, y_val)],
    eval_metric='average_precision',
    callbacks=[lgb.early_stopping(stopping_rounds=50, verbose=True)]
)
```

---

## 5. 評価方法の改善

### 5.1 PR曲線とPR-AUCの可視化

```python
from sklearn.metrics import precision_recall_curve, average_precision_score
import matplotlib.pyplot as plt

# PR曲線
precision_vals, recall_vals, thresholds = precision_recall_curve(y_test, y_pred_2_prob)
ap_score = average_precision_score(y_test, y_pred_2_prob)

plt.figure(figsize=(10, 6))
plt.plot(recall_vals, precision_vals, label=f'PR curve (AP = {ap_score:.4f})')
plt.xlabel('Recall')
plt.ylabel('Precision')
plt.title('Precision-Recall Curve')
plt.legend()
plt.grid(True)
plt.savefig('pr_curve.png')
plt.show()
```

### 5.2 Lift曲線

```python
def calculate_lift(y_true, y_prob, n_bins=10):
    """Lift曲線を計算"""
    df_lift = pd.DataFrame({'y_true': y_true, 'y_prob': y_prob})
    df_lift = df_lift.sort_values('y_prob', ascending=False).reset_index(drop=True)
    
    # 10分位に分割
    df_lift['decile'] = pd.qcut(df_lift.index, n_bins, labels=False) + 1
    
    # 各分位の離脱率
    baseline_rate = y_true.mean()
    lift_table = df_lift.groupby('decile').agg({
        'y_true': ['sum', 'count', 'mean']
    }).reset_index()
    lift_table.columns = ['decile', 'churners', 'total', 'churn_rate']
    lift_table['lift'] = lift_table['churn_rate'] / baseline_rate
    
    return lift_table

lift_table = calculate_lift(y_test.values, y_pred_2_prob)
print(lift_table)
```

---

## 6. 確認事項（重要）

以下の点を確認いただくと、より精度の高いアドバイスが可能です：

1. **本番データの離脱率**: flag_1m=1 の比率は何%ですか？（サンプルデータでは77.8%でしたが、本番と異なる可能性があります）

2. **特徴量の集計期間**: r01〜r06、a2 はそれぞれ何ヶ月前のデータですか？目的変数（flag_1m）の定義日より未来の情報が混ざっていないか確認が必要です。

3. **データの粒度**: 1顧客1レコードですか？それとも月次スナップショットで同一cust_noが複数行ありますか？

4. **施策の運用制約**: 
   - 毎月何人に配布可能ですか？
   - Precisionの最低ラインはありますか？
   - コスト（媒体費）の上限はありますか？

5. **追加データの有無**: 以下のようなデータがあれば、特徴量として追加することでPrecision改善が期待できます：
   - 返品・キャンセル履歴
   - 問い合わせ履歴
   - メール/LINE開封・クリック履歴
   - 過去のクーポン配布・利用履歴

---

## 7. まとめ

Precision改善のための優先度順アクション：

1. **閾値最適化**: 0.5固定ではなく、運用に合わせた閾値を選定（または上位N人運用に切り替え）
2. **class_weightの調整**: False Positiveを減らす方向に重み付け
3. **交差検証の導入**: 単発splitではなくStratifiedKFoldで安定した評価
4. **ハイパーパラメータチューニング**: LightGBM/XGBoostの正則化パラメータを調整
5. **特徴量エンジニアリング**: RFMの比率・傾向、Web行動、値引き依存度などを追加

これらの施策により、Precision 75-80%程度への改善が期待できます。さらなる改善には、追加データの活用やUpliftモデリングへの発展が有効です。
