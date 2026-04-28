---
title: 客户流失生存分析报告
author: 夏浩喆
date: 2026-04-28
layout: post
---

# 客户流失生存分析报告

## 项目概述

本项目基于 Databricks Industry Solutions 的生存分析解决方案，使用 **IBM Telco Customer Churn** 数据集进行客户流失的生存分析。数据集包含 7,043 条电信客户记录，涵盖人口统计、服务套餐、使用行为及订阅状态。

**生存分析** 关注“事件发生的时间”。在本案例中：
- **事件**：客户流失（Churn）
- **生存时间**：客户在网时长（Tenure）

我们采用了三种主流方法：
1. **Kaplan-Meier** 非参数估计  
2. **Cox 比例风险** 半参数模型  
3. **加速失效时间（AFT）** 全参数模型  

最后，基于模型输出计算客户终身价值（CLV），为业务决策提供量化依据。

---

## 数据准备

### 数据来源
- IBM Telco Customer Churn 数据集（公开）
- 原始记录数：7,043，字段数：21

### 关键字段

| 字段 | 类型 | 说明 |
|------|------|------|
| customerID | String | 客户唯一标识 |
| tenure | Int | 在网月数（生存时间） |
| Churn | String (Yes/No) | 是否流失（事件指示器） |
| Contract | String | 合约类型（月付/年付/两年付） |
| InternetService | String | 互联网服务类型 |
| MonthlyCharges | Decimal | 月费用 |
| TotalCharges | Decimal | 总费用 |

### 清洗与筛选步骤
1. 使用 `urllib` 下载原始 CSV  
2. 用 PySpark 显式定义 Schema 加载  
3. `Churn` 字段映射为二进制（Yes→1, No→0）  
4. **仅保留月付合约且有互联网服务的客户**（流失风险最高，且互联网特征是关键变量）  

最终得到 **Silver 表**，作为后续分析的基础。

---

## Kaplan-Meier 生存分析

### 方法原理
Kaplan-Meier 估计量是非参数方法，能够处理右删失数据。生存函数 \( S(t) \) 表示客户存活超过时间 \( t \) 的概率。

### 总体生存曲线
- 所有客户起始生存概率为 100%  
- 生存概率随时间单调下降  
- **中位生存时间 ≈ 34 个月**（50% 的客户在此前流失）  
- 置信区间随时间变宽，反映远期预测的不确定性  

### 分层生存曲线与 Log‑Rank 检验

对多个分类变量分别绘制 KM 曲线，并执行 Log‑Rank 检验。部分关键结果如下：

| 协变量 | 显著 (p<0.05) | 观察 |
|--------|---------------|------|
| gender | 否 | 男女无显著差异 |
| SeniorCitizen | 是 | 老年客户流失更快 |
| Partner | 是 | 有伴侣的客户留存更久 |
| Dependents | 是 | 有家属的客户留存更久 |
| InternetService | 是 | Fiber optic 用户流失最快 |
| OnlineSecurity | 是 | 有安全服务的客户留存更久 |
| TechSupport | 是 | 有技术支持的客户留存更久 |
| PaperlessBilling | 是 | 电子账单用户流失更快 |
| PaymentMethod | 是 | 自动扣款用户留存更久 |

> **重要结论**：服务类特征（Online Security, Tech Support, Online Backup, Device Protection）对流失有显著影响，是预测流失的核心变量。Gender 和 Phone Service 无显著差异。

---

## Cox 比例风险模型

### 方法原理
Cox 模型是半参数回归模型，假设风险比（Hazard Ratio）随时间恒定。  
\( h(t|X) = h_0(t) \cdot \exp(\beta X) \)  
- \( \exp(\beta) > 1 \)：流失风险升高  
- \( \exp(\beta) < 1 \)：流失风险降低  

### 特征编码
对 5 个分类变量进行独热编码（drop_first），避免多重共线性：
- Dependents
- InternetService
- OnlineBackup
- TechSupport
- PaperlessBilling

### 模型结果

| 协变量 | exp(coef) | p值 | 解释 |
|--------|-----------|-----|------|
| dependents_Yes | < 1 | <0.005 | 有家属降低风险 |
| internetService_DSL | < 1 | <0.005 | DSL 比 Fiber optic 风险低 |
| onlineBackup_Yes | < 1 | <0.005 | 在线备份降低风险 |
| techSupport_Yes | < 1 | <0.005 | 技术支持降低风险 |
| paperlessBilling_Yes | > 1 | <0.005 | 电子账单增加风险 |


### 比例风险假设检验
- **Schoenfeld 残差检验**：3 个变量违反假设（p < 0.05）  
- **Log‑log KM 曲线**：曲线不完全平行，尤其在早期和晚期

**处理建议**：
- 若目标为预测而非推断，可忽略违反，关注损失指标  
- 对违反变量进行分层（Stratification）  
- 引入时间依赖变量（Extended Cox）  
- 改用 AFT 模型

---

## 加速失效时间（AFT）模型

### 方法原理
AFT 是参数模型，直接假设生存时间服从特定分布（此处使用 **Log‑Logistic**）。  
模型形式：\( \log(T) = \mu + \gamma X + \sigma \epsilon \)  
- \( \exp(\gamma) > 1 \) → 延长生存时间  
- \( \exp(\gamma) < 1 \) → 缩短生存时间  

### 模型结果
所有选定协变量均显著（p < 0.005），部分示例如下：

| 协变量 | exp(coef) | 解释 |
|--------|-----------|------|
| partner_Yes | > 1 | 有伴侣延长留存 |
| internetService_DSL | > 1 | DSL 用户比 Fiber 留存更久 |
| techSupport_Yes | > 1 | 技术支持大幅延长生命周期 |
| deviceProtection_Yes | > 1 | 设备保护延长留存 |

### 假设检验
- **Log‑odds 诊断图**：曲线基本呈直线 → Log‑Logistic 分布选择合理  
- **曲线不完全平行** → 比例优势假设部分违反  

> AFT 模型尽管假设部分违背，但作为 CPH 的补充，仍具预测价值。

---

## 客户终身价值（CLV）

### 计算方法
利用 Cox 模型的 `predict_survival_function()` 输出每个客户的未来生存概率。  
每月预期利润 = \( S(t) \times \text{MonthlyCharges} \)  
净现值（NPV）= \( \frac{\text{预期利润}}{(1+r)^t} \)（月贴现率 \( r = 1\% \)）  
累计 CLV 为 36 个月 NPV 之和。

### 结果示例（按 TechSupport 分组）
- **有技术支持**的客户：36 个月累计 NPV 显著更高（约高出 40%）  
- **无技术支持**的客户：前期收入衰减更快

> **业务应用**：客户获取成本（CAC）不应超过预测 CLV。生存分析可为营销预算分配提供量化依据。

---

## 模型对比与总结

| 方法 | 类型 | 优势 | 局限 | 适用场景 |
|------|------|------|------|----------|
| Kaplan‑Meier | 非参数 | 直观，无需分布假设 | 仅单变量，无法预测 | 探索性分析 |
| Cox PH | 半参数 | 多变量，灵活 | 比例风险假设易违反 | 多因素建模、推断 |
| AFT | 全参数 | 直接建模生存时间 | 需指定分布 | 分布已知时精确建模 |

### 核心结论
1. **服务类特征**（TechSupport, OnlineBackup, DeviceProtection）显著降低流失风险；Fiber optic 用户流失最快。
2. **Gender 和 PhoneService** 对流失无显著影响，不适合作为预测变量。
3. **Cox 模型违反比例风险假设**，建议采用分层或时间依赖变量改进。
4. **生存分析可计算 CLV**，直接指导营销预算与客户留存策略。

---

## 参考文献与工具
- Databricks Industry Solutions: [survival-analysis](https://github.com/databricks-industry-solutions/survival-analysis)
- IBM Telco Customer Churn 数据集（公开）
- PySpark, Lifelines, Matplotlib, Seaborn

---

*报告生成日期：2026年4月28日*
