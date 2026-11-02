# 模型卡：学业风险预测与综合分回归（阶段 4）

> 数据版本：`3600258c1cd4`（原始 CSV sha256 前缀）；输入：data/processed/features.parquet + students.parquet；
> 生成时间：2026-10-29T07:45:01+00:00；随机种子：42。
> 本卡片数字全部由 `../.venv/bin/python scripts/run_pipeline.py --stage model` 自动生成，可重跑复现。

## 1. 任务与目标

- **分类任务**：预测学业风险 risk = (avg_score < 60)，
  用于识别需重点干预的学生（285 人 / 28.5%）。
- **回归任务**：预测综合分 avg_score（三科算术平均，0–100），用于排序与风险分值化。
- **特征**：冻结特征表 12 个背景字段（性别、族裔 one-hot、父母教育 one-hot/有序、午餐 SES 代理、备考课程），
  **不含三科成绩与 avg_score 本身**，不存在标签泄漏。
- **用途**：为公平性审计（fairness_report.md）与业务洞察（阶段 6）提供模型证据。

## 2. 数据与切分

| 指标 | 数值 |
| --- | --- |
| 样本量 | 1,000 |
| 特征数 | 12（冻结契约） |
| 风险人数 / 风险率 | 285 / 28.5% |
| avg_score 均值 ± 标准差 | 67.77 ± 14.26 |
| train / test | 800 / 200 |
| 切分方式 | 按 risk 分层 80/20，random_state=42 |

切分参数与 train/test 的 student_id 清单见 `data/processed/models/classify_split.json`（可完全复现）。
**无泄漏**：5 折 CV 只在训练集内；测试集只做最终评估；模型选择只用 CV。

## 3. 回归任务（目标：avg_score）

| 模型 | CV RMSE | CV MAE | test RMSE | test MAE | test R² |
| --- | ---: | ---: | ---: | ---: | ---: |
| DummyRegressor | 14.47 ± 0.31 | 11.50 | 13.35 | 10.86 | -0.007 |
| Ridge | 12.72 ± 0.53 | 10.31 | 12.13 | 9.73 | 0.169 |
| RandomForestRegressor | 13.67 ± 0.42 | 11.05 | 12.90 | 10.20 | 0.060 |
| GradientBoostingRegressor | 13.30 ± 0.45 | 10.69 | 12.40 | 9.82 | 0.132 |

- 最优回归模型（按 CV RMSE）：**Ridge**。
- 基线 DummyRegressor 的 CV RMSE 约 14.47；
  最优模型 test R² 仅约 0.169，
  说明 12 个背景字段对分数解释力有限，预测用途应定位为「排序参考」而非「精确预测」。

![回归模型对比](figures/04_model_comparison_regression_20261029.png)

## 4. 分类任务（目标：risk）

| 模型 | CV macro-F1 | CV ROC-AUC | test acc | test macro-F1 | test precision | test recall | test ROC-AUC | test PR-AUC |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| DummyClassifier | 0.417 ± 0.001 | 0.500 | 0.715 | 0.417 | 0.000 | 0.000 | 0.500 | 0.285 |
| LogisticRegression | 0.656 ± 0.030 | 0.746 | 0.630 | 0.603 | 0.407 | 0.649 | 0.707 | 0.539 |
| RandomForestClassifier | 0.585 ± 0.032 | 0.666 | 0.605 | 0.580 | 0.383 | 0.632 | 0.670 | 0.432 |
| GradientBoostingClassifier | 0.607 ± 0.041 | 0.703 | 0.715 | 0.612 | 0.500 | 0.351 | 0.667 | 0.458 |

- 最优分类模型（按 CV macro-F1）：**LogisticRegression**，测试集 macro-F1=0.603、
  ROC-AUC=0.707、PR-AUC=0.539。
- 基线 DummyClassifier（全部预测多数类）测试集 macro-F1=0.417，
  最优模型相对基线提升 **+0.186**；代价是准确率下降
  （0.715 → 0.630），
  因为 balanced 类权重偏向召回高风险学生 —— 这是「筛查用途」的合理取舍，不是缺陷。
- 测试集混淆矩阵（阈值 0.5）：[[89, 54], [20, 37]]（行=真实，列=预测，1=高风险）。

![分类模型对比](figures/04_model_comparison_classification_20261029.png)

![混淆矩阵](figures/04_confusion_matrix_20261029.png)

![ROC/PR 曲线](figures/04_roc_pr_curves_20261029.png)

## 5. 概率校准

LogisticRegression 使用 class_weight="balanced"，输出概率整体偏高（未校准），用概率做决策前需注意。
测试集校准表（分位数 5 箱）：

| 预测概率 bin 中点 | n | 平均预测概率 | 实际高风险率 | 偏差（预测-实际） |
| --- | ---: | ---: | ---: | ---: |
| 0.154 | 40 | 0.169 | 0.075 | +0.094 |
| 0.331 | 38 | 0.345 | 0.316 | +0.029 |
| 0.478 | 42 | 0.469 | 0.190 | +0.279 |
| 0.619 | 40 | 0.610 | 0.300 | +0.310 |
| 0.810 | 40 | 0.786 | 0.550 | +0.236 |

- ECE（期望校准误差）=0.192；最大分箱偏差=0.310；
  Brier=0.221。
- 结论：模型概率**高估**风险（预测均值 0.477 vs 实际 0.285）。
  若看板按概率阈值分层，应使用校准后概率或改用排序分位；本阶段不调 test 阈值。

![校准曲线](figures/04_calibration_curve_20261029.png)

## 6. SHAP 归因

- 解释对象：最优分类模型 **LogisticRegression**；解释器 LinearExplainer；
  解释目标：decision_function（LogReg 对数几率，正值推向高风险）；解释样本 n=1,000（全量，用于覆盖小样本子组特征），
  背景样本 n=200（仅取自训练集，随机种子 42）。
- 可加性校验：max |base_value + ΣSHAP − 模型输出| = 4.44e-16。
- **归因口径（关键）**：SHAP 用「条件均值」——仅在该特征出现的样本上平均；
  支持度阈值 support_n ≥ 20。原因：线性 SHAP 的全样本均值会被特征出现率主导，
  导致方向反转（sc_cvs 复盘教训）。
- 方向解读：条件均值 > 0 → 该特征出现时推高「高风险」预测；< 0 → 降低风险预测。

| 排名 | 特征 | 平均绝对 SHAP | 条件均值 SHAP | 出现样本数 | 方向 |
| ---: | --- | ---: | ---: | ---: | --- |
| 1 | lunch_free_reduced | 0.6556 | +0.8934 | 355 | 推高风险 |
| 2 | prep_completed | 0.4821 | -0.6704 | 358 | 降低风险 |
| 3 | gender_male | 0.2928 | +0.3202 | 482 | 推高风险 |
| 4 | edu_level | 0.1514 | -0.0524 | 821 | 降低风险 |
| 5 | race_E | 0.1411 | -0.5147 | 140 | 降低风险 |
| 6 | race_A | 0.1262 | +0.5317 | 89 | 推高风险 |
| 7 | edu_bachelor_or_higher | 0.1185 | -0.3430 | 177 | 降低风险 |
| 8 | edu_some_college | 0.0991 | -0.2168 | 226 | 降低风险 |
| 9 | race_D | 0.0816 | -0.1757 | 262 | 降低风险 |
| 10 | edu_high_school_or_less | 0.0786 | +0.1104 | 375 | 推高风险 |
| 11 | race_C | 0.0494 | +0.0818 | 319 | 推高风险 |
| 12 | race_B | 0.0132 | -0.0321 | 190 | 降低风险 |

- 解读（关联，非因果）：lunch_free_reduced（免费/减价午餐）出现时推高风险，
  prep_completed（完成备考课程）出现时降低风险；族裔/性别特征也携带方向性信号，
  但这是数据中既有群体差异的反映，**不能**解释为个体属性导致成绩差异（见公平性报告「不可因果解读」）。
- 归因值落盘：reports/shap_values.parquet（student_id × feature 长表，含 support_n），可供看板/画像下钻。

![SHAP 全局重要性](figures/04_shap_importance_20261029.png)

## 7. 局限与风险

1. **数据规模与推断力**：1,000 行、12 个背景字段；回归 R² 有限、分类 AUC 约 0.7，模型是辅助筛查工具，不能替代人工判断。
2. **观测数据**：无随机化与学校/地区/年份字段，所有关联（含 SHAP 方向）不能做因果解释。
3. **敏感变量**：race_ethnicity / lunch / parental_education 为敏感变量，禁止个体归因；
   模型使用它们会继承数据中的历史差异，公平性审计已量化（fairness_report.md）。
4. **概率未校准**：class_weight="balanced" 导致概率高估（见第 5 节）。
5. **单次切分**：test 指标依赖一次固定切分（seed 42），模型间差异以 CV 为准。
6. **配置缺省项**：树模型超参（n_estimators=300/200、depth=3、
   lr=0.05）、SHAP 背景量（200）、支持度阈值（20）、
   校准分箱数（5）未在 configs/project.yaml 定义，使用模块默认值（不修改配置）。
7. **阈值**：分类阈值固定 0.5，未在 test 上调优；实际部署应按业务成本确定。

## 8. 产物与复现

| 产物 | 路径 |
| --- | --- |
| 最优分类模型 | data/processed/models/best_classifier.joblib |
| 最优回归模型 | data/processed/models/best_regressor.joblib |
| 切分索引 | data/processed/models/classify_split.json |
| 特征重要性 | data/processed/models/feature_importance.csv |
| SHAP 值长表 | reports/shap_values.parquet |
| 指标 JSON | reports/metrics_student.json |
| 公平性报告 | reports/fairness_report.md |
| 看板接口 | reports/dashboard_data/model.json |

```bash
cd sc_edu
export MPLCONFIGDIR="$PWD/.cache/matplotlib"
../.venv/bin/python scripts/run_pipeline.py --stage model
../.venv/bin/python -m pytest tests/test_modeling.py -q
```

## 9. 校准与决策阈值扩展（P0 追加，2026-10-29）

> 完整分析、表格与 4 张 04b_* 图见 reports/calibration_threshold.md（本卡片只保留摘要，主分析结论与口径不变）。

- **概率校准**（训练集 5 折 OOF 拟合、测试集只评估）：选优 **isotonic**，
  ECE 0.1922 → 0.0521；
  Brier 0.2209 → 0.1851；
  平均预测概率 0.4772 → 0.2984（更接近实际风险率）。
- **决策阈值**：校准后测试集扫描 0.05~0.90，推荐阈值 **0.20**
  （在召回率 >= 0.80 的前提下精确率最高（precision=0.359，recall=0.912）），此时标记 145 人
  （0.7250），precision=0.3586、
  recall=0.9123。
- **分组阈值**：以整体 OOF recall=0.6526 为目标逐组调整后，
  组间 recall 最大差距 **0.5635 → 0.2565**
  （代价：平均 precision 0.4514 → 0.4246）；
  该做法涉及「按敏感属性分别设阈值」，部署前需法律/伦理评审。
- **风险标签阈值**：t=50/55/60/65 的风险率 0.1030 → 0.3990，
  CV macro-F1 在 t=60 处最高（0.6562）；主分析与全站口径仍为 avg_score < 60。
- **向后兼容**：主分析全部章节与字段口径不变；本扩展只新增产物文件、图（04b_*）与 JSON 字段。
