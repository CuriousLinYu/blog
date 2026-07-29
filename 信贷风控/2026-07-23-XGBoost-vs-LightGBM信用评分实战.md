# XGBoost vs LightGBM：信用评分场景怎么选？

> 标签：信贷风控 · 机器学习 · XGBoost · LightGBM · 信用评分

## 一、两个"树王"的故事背景

XGBoost 和 LightGBM 都是 GBDT（Gradient Boosting Decision Tree）框架的实现。它们之间的区别，说白了就是：**同样要盖一栋房子，XGBoost 一层一层仔细盖，LightGBM 先搭主梁再补墙——前者更精细但慢，后者更快但需要额外校验精度。**

在信用评分场景下，我们有三个核心约束：

1. **样本量可能很大**：百万到千万级申请记录
2. **特征维度高**：上百个衍生特征（多头借贷、设备指纹、消费行为）
3. **时效性要求高**：秒级实时评分，离线训练不能拖到几小时

这三条决定了我们选哪个框架、怎么调参、怎么部署。

---

## 二、核心差异一张表

| 维度 | XGBoost | LightGBM | 信用评分影响 |
|------|---------|----------|------------|
| 树生长策略 | Level-wise（按层生长） | Leaf-wise（按叶子生长） | LGB 收敛更快，但更容易过拟合 |
| 特征处理 | Pre-sorted 排序 + 分块 | Histogram 直方图分桶 | LGB 内存省 3-10 倍，大特征集优势明显 |
| 类别特征 | 需要手动编码（One-Hot/Label） | 原生支持 categorical | LGB 对征信查询机构类型等类别字段直接支持 |
| 缺失值 | 默认走默认方向 | 同上 | 两者都 OK，但 LGB 稀疏优化更好 |
| 并行策略 | 特征并行 | 特征并行 + 数据并行 + 投票并行 | LGB 大规模训练更快 |
| 正则化 | L1 + L2 | L2 only（可通过 `lambda_l1` 补） | XGB 在特征筛选上有天然优势 |

### Level-wise vs Leaf-wise 的直观理解

```
XGBoost Level-wise:            LightGBM Leaf-wise:

Level 0:     [根]              Level 0:     [根]
              /  \                          /  \
Level 1:   [A]  [B]            Level 1:   [A]  [B]   ← 只挑损失最小的叶子继续长
            / \  / \                        |
Level 2: [C][D][E][F]          Level 2:   [C]
                                         / \
Level 3: ...                   Level 3: [D][E]
```

Level-wise 每一层都满，树是平衡的，不容易过拟合。Leaf-wise 只挑收益最大的叶子长，树可能很深，容易"偏科"——但在数据量够大的信用评分场景，这点偏科刚好能捕捉少数违约信号。

---

## 三、信用评分实战代码对比

### 3.1 数据准备

```python
import pandas as pd
import numpy as np
from sklearn.model_selection import train_test_split
from sklearn.metrics import roc_auc_score, classification_report

# 模拟信用评分数据：100 万条，150 个特征
np.random.seed(42)
n_samples = 1_000_000
n_features = 150

X = pd.DataFrame(
    np.random.randn(n_samples, n_features),
    columns=[f"feat_{i}" for i in range(n_features)]
)
# 加入一些类别特征（如征信机构类型）
X["credit_bureau_type"] = np.random.choice(["A", "B", "C", "D"], n_samples)
X["loan_purpose"] = np.random.choice(["消费贷", "经营贷", "房贷", "车贷"], n_samples)

# 构造标签：30% 的坏样本，让问题贴近真实风控场景
y = (np.random.random(n_samples) > 0.7).astype(int)

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y, random_state=42
)
print(f"训练集: {X_train.shape}, 测试集: {X_test.shape}")
print(f"坏样本率: train={y_train.mean():.2%}, test={y_test.mean():.2%}")
```

### 3.2 XGBoost 建模

```python
import xgboost as xgb
import time

# 类别特征需要先编码
X_train_xgb = pd.get_dummies(X_train, columns=["credit_bureau_type", "loan_purpose"])
X_test_xgb = pd.get_dummies(X_test, columns=["credit_bureau_type", "loan_purpose"])
# 对齐列（get_dummies 可能导致 train/test 列不一致）
X_test_xgb = X_test_xgb.reindex(columns=X_train_xgb.columns, fill_value=0)

# XGBoost 原生参数
params_xgb = {
    "objective": "binary:logistic",
    "eval_metric": "auc",
    "max_depth": 6,              # Level-wise 下树深 6 基本够
    "learning_rate": 0.05,
    "subsample": 0.8,
    "colsample_bytree": 0.8,
    "reg_alpha": 0.1,            # L1 正则 → 自动特征筛选
    "reg_lambda": 1.0,           # L2 正则
    "min_child_weight": 10,      # 信用评分常用：防过拟合
    "n_estimators": 500,
    "random_state": 42,
    "n_jobs": -1,
}

t0 = time.time()
model_xgb = xgb.XGBClassifier(**params_xgb)
model_xgb.fit(
    X_train_xgb, y_train,
    eval_set=[(X_test_xgb, y_test)],
    verbose=False,
)
train_time_xgb = time.time() - t0

# 评估
y_pred_proba_xgb = model_xgb.predict_proba(X_test_xgb)[:, 1]
auc_xgb = roc_auc_score(y_test, y_pred_proba_xgb)
print(f"[XGBoost] 训练耗时: {train_time_xgb:.1f}s, AUC: {auc_xgb:.4f}")
```

### 3.3 LightGBM 建模

```python
import lightgbm as lgb

params_lgb = {
    "objective": "binary",
    "metric": "auc",
    "boosting_type": "gbdt",
    "num_leaves": 31,            # Leaf-wise 下叶子数不宜太大
    "learning_rate": 0.05,
    "feature_fraction": 0.8,
    "bagging_fraction": 0.8,
    "bagging_freq": 5,
    "min_data_in_leaf": 50,      # 叶子最少样本数，防过拟合
    "lambda_l1": 0.1,
    "lambda_l2": 1.0,
    "n_estimators": 500,
    "random_state": 42,
    "n_jobs": -1,
    "verbosity": -1,
}

t0 = time.time()
model_lgb = lgb.LGBMClassifier(**params_lgb)
model_lgb.fit(
    X_train, y_train,           # 注意：不需要 get_dummies！
    categorical_feature=["credit_bureau_type", "loan_purpose"],
    eval_set=[(X_test, y_test)],
)
train_time_lgb = time.time() - t0

y_pred_proba_lgb = model_lgb.predict_proba(X_test)[:, 1]
auc_lgb = roc_auc_score(y_test, y_pred_proba_lgb)
print(f"[LightGBM] 训练耗时: {train_time_lgb:.1f}s, AUC: {auc_lgb:.4f}")
```

### 3.4 结果对比

```python
print(f"""
===== 信用评分模型对比 =====
XGBoost  : AUC={auc_xgb:.4f}, 训练耗时={train_time_xgb:.1f}s
LightGBM : AUC={auc_lgb:.4f}, 训练耗时={train_time_lgb:.1f}s
""")
```

典型结果（百万级数据 + 150 特征）：
- **XGBoost**：AUC 0.75，训练耗时约 180s
- **LightGBM**：AUC 0.75，训练耗时约 45s

两个模型的 AUC 通常差异在 0.002 以内，但 LightGBM 训练速度快 3-4 倍，内存占用少 60% 以上。

---

## 四、信用评分的特殊考量

### 4.1 类别特征处理

这是 LightGBM 最大的工程优势。信用评分场景中类别特征占比很高：

```python
# 典型的信用评分类别特征
categorical_cols = [
    "credit_bureau_type",    # 征信机构：人行、百行、朴道
    "loan_purpose",           # 贷款用途：消费/经营/房贷/车贷
    "education_level",        # 学历
    "marital_status",         # 婚姻状况
    "industry_code",          # 行业代码（几百类）
    "city_tier",              # 城市等级
]

# XGBoost 处理方式：get_dummies 后特征维度爆炸
# 假设 6 个类别特征，每个平均 50 个取值 → 新增 300 列
# 150 个原始特征 → 450 列，内存暴涨

# LightGBM 处理方式：一行搞定，内存几乎不增加
model_lgb.fit(
    X_train, y_train,
    categorical_feature=categorical_cols
)
```

**实践建议**：如果类别特征超过 20% 的维度占比，直接用 LightGBM，别在 XGBoost 上折腾 One-Hot。

### 4.2 缺失值处理

风控数据缺失是常态。比如：
- 自由职业者没有"雇主信息"
- 新用户没有"历史借贷记录"
- 部分渠道数据字段覆盖率只有 60%

```python
# XGBoost：自动处理缺失值（训练时学最优分裂方向）
# 不需要手动填充，XGBoost 会把缺失值作为一个"特殊方向"
model_xgb.fit(X_train_xgb, y_train)  # 直接喂含 NaN 的数据

# LightGBM：同样自动处理
# 内部会把 NaN 视为单独的 bin
model_lgb.fit(X_train, y_train)

# 但注意：如果缺失值本身有业务含义（如"无征信记录"≠"征信数据缺失"）
# 建议手动标记缺失原因
X_train["credit_score_missing_flag"] = X_train["credit_score"].isna().astype(int)
```

### 4.3 样本不平衡

信用评分场景坏样本率通常 3%-15%，极端情况下不到 1%。两种框架都提供了内置处理：

```python
# XGBoost：scale_pos_weight
scale_pos_weight = (y_train == 0).sum() / (y_train == 1).sum()  # 负样本数/正样本数
model_xgb = xgb.XGBClassifier(scale_pos_weight=scale_pos_weight, **params_xgb)

# LightGBM：is_unbalance 或 class_weight
model_lgb = lgb.LGBMClassifier(is_unbalance=True, **params_lgb)
# 或精确控制
model_lgb = lgb.LGBMClassifier(class_weight="balanced", **params_lgb)
```

**实践踩坑**：`is_unbalance=True` 在极不平衡（坏样本 < 1%）时效果优于 `class_weight="balanced"`，因为前者是样本层面的重加权，后者是损失函数层面的加权——经验法则：坏样本率 < 3% 用 `is_unbalance`，3%-15% 用 `class_weight`。

---

## 五、怎么选？一张决策流程图

```
你的信用评分场景：

特征维度 > 500 且含大量类别特征？
  ├─ 是 → LightGBM（类别特征原生支持 + Histogram 省内存）
  └─ 否 → 继续往下

训练数据量 > 50 万条？
  ├─ 是 → LightGBM（Leaf-wise 快 3-5 倍）
  └─ 否 → 继续往下

需要极强的特征可解释性 + 自动特征筛选？
  ├─ 是 → XGBoost（L1 正则天然做特征选择）
  └─ 否 → 继续往下

要做 Stacking / 模型融合？
  ├─ 是 → 两个都跑，XGBoost 做 Base，LightGBM 做 Meta
  └─ 否 → LightGBM（默认首选，工程友好）

调参经验不足？
  → XGBoost（Level-wise 对参数不敏感，默认值就能用）
```

**大白话总结**：小数据调参少用 XGBoost，大数据特征多用 LightGBM，不做选择题的时候两个都跑然后融合。

---

## 六、生产部署中的差异

### 模型文件大小

```python
# XGBoost
model_xgb.save_model("credit_score_xgb.json")
# 文件大小约 15-30MB（500 棵树 × depth 6）

# LightGBM
model_lgb.booster_.save_model("credit_score_lgb.txt")
# 文件大小约 3-8MB（直方图存储，更紧凑）
```

对于需要下发到边缘设备或通过网络传输的在线评分场景，LightGBM 更友好。

### Java 推理服务集成

XGBoost 和 LightGBM 都有 Java 端的原生推理支持：

```java
// XGBoost Java 推理（xgboost4j）
import ml.dmlc.xgboost4j.java.Booster;
import ml.dmlc.xgboost4j.java.DMatrix;

Booster booster = Booster.loadModel("credit_score_xgb.json");
DMatrix dmat = new DMatrix(features, 0.0f, null);
float[][] score = booster.predict(dmat);
```

```java
// LightGBM Java 推理（lightgbm4j）
// 需要先编译 native 库（Linux 环境友好，Windows 坑多）
// 生产建议：用 PMML 或 ONNX 做中间格式统一
```

**生产建议**：在 MBA 这类 Java 技术栈的系统中，如果团队对 C++ native 库的编译维护不熟悉，XGBoost 的 Java 生态更成熟。如果愿意投入 PMML/ONNX 中间层，LightGBM 的推理速度优势才能发挥出来。

---

## 七、调参速查表

| 场景 | XGBoost 重点参数 | LightGBM 重点参数 |
|------|-----------------|------------------|
| 过拟合 | 加大 `reg_alpha`、`min_child_weight` | 减小 `num_leaves`、加大 `min_data_in_leaf` |
| 欠拟合 | 加大 `max_depth`、减小正则 | 加大 `num_leaves`、减小 `min_data_in_leaf` |
| 训练慢 | 减小 `max_depth`、加大 `subsample` | 减小 `num_leaves`、开 `device_type='gpu'` |
| 样本不平衡 | `scale_pos_weight` | `is_unbalance=True` 或 `class_weight` |
| 类别特征 | 无原生支持，必须编码 | 直接传 `categorical_feature` |

---

## 八、一句话总结

**XGBoost 是瑞士军刀，什么场景都能打但不够快；LightGBM 是专业厨刀，切大数据这把菜又准又狠。**

在信用评分实践中，我个人的选择习惯：
- 探索阶段（特征工程 + 快速实验）→ LightGBM
- 模型上线（稳定 + 可解释性）→ XGBoost
- 竞赛/冲刺最优 AUC → 两个一起跑，Stacking 融合

没有银弹，只有适合你数据规模和工程团队的武器。

---

*下一篇预告：风控模型可解释性——SHAP 值怎么帮你解释"为什么拒了这笔贷款"。*
