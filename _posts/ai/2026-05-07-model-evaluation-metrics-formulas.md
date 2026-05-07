---
title: 从 AUC 到 Agent Eval：常见模型评估指标与公式速查
date: 2026-05-07 00:00:00 +0800
categories: [人工智能, 机器学习]
tags: [模型评估, Evaluation, Metrics, AUC, F1, RAG, Agent, LLM]
math: true
---

> **核心观点**：评估指标不是一串漂亮数字，而是系统反馈链路中的压缩信号。不同任务的“好”并不相同：分类关心错分代价，回归关心误差形态，排序关心前几名是否有用，LLM 应用还要关心事实性、安全性、格式、工具调用和成本。选错指标，模型可能越优化越偏离真实目标。

## 一、先统一符号

设测试集有 $n$ 条样本：

$$
\mathcal{D} = \{(x_i, y_i)\}_{i=1}^{n}
$$

模型预测值记为 $\hat{y}_i$，预测概率记为 $\hat{p}_i$。对于二分类任务，通常有：

$$
y_i \in \{0, 1\}, \quad \hat{p}_i = P(y_i = 1 \mid x_i)
$$

当把预测概率通过阈值 $t$ 转成类别时：

$$
\hat{y}_i =
\begin{cases}
1, & \hat{p}_i \ge t \\
0, & \hat{p}_i < t
\end{cases}
$$

二分类最基础的是混淆矩阵：

| 真实 / 预测 | 预测为正 | 预测为负 |
|---|---:|---:|
| 真实为正 | TP | FN |
| 真实为负 | FP | TN |

其中：

$$
\begin{aligned}
TP &= \sum_i \mathbf{1}(y_i = 1 \land \hat{y}_i = 1) \\
TN &= \sum_i \mathbf{1}(y_i = 0 \land \hat{y}_i = 0) \\
FP &= \sum_i \mathbf{1}(y_i = 0 \land \hat{y}_i = 1) \\
FN &= \sum_i \mathbf{1}(y_i = 1 \land \hat{y}_i = 0)
\end{aligned}
$$

很多分类指标本质上都是从这四个数派生出来的。

## 二、二分类指标

### 2.1 Accuracy：整体正确率

$$
\text{Accuracy} = \frac{TP + TN}{TP + TN + FP + FN}
$$

Accuracy 适合类别分布相对均衡、错分代价接近的场景。如果正负样本极不均衡，它很容易产生错觉。例如欺诈检测中，99% 的样本都是正常交易，模型永远预测“正常”也可能有 99% Accuracy，但业务价值几乎为零。

### 2.2 Precision：预测为正时有多准

$$
\text{Precision} = \frac{TP}{TP + FP}
$$

Precision 回答的问题是：**模型说是正样本的那些样本里，有多少真的为正？**

适合关注误报代价的场景，例如：

- 推荐系统不希望把无关内容推荐给用户
- 风控系统不希望误伤正常用户
- 告警系统不希望产生太多无效告警

### 2.3 Recall：真实正样本找回了多少

$$
\text{Recall} = \frac{TP}{TP + FN}
$$

Recall 也叫 Sensitivity、True Positive Rate：

$$
\text{TPR} = \text{Recall}
$$

它回答的问题是：**所有真实正样本中，有多少被模型找到了？**

适合关注漏报代价的场景，例如疾病筛查、欺诈检测、故障检测、恶意内容识别。

### 2.4 Specificity：真实负样本排除了多少

$$
\text{Specificity} = \frac{TN}{TN + FP}
$$

Specificity 也叫 True Negative Rate：

$$
\text{TNR} = \text{Specificity}
$$

它关注模型识别负样本的能力。与之对应的是误报率：

$$
\text{FPR} = \frac{FP}{FP + TN} = 1 - \text{Specificity}
$$

漏报率为：

$$
\text{FNR} = \frac{FN}{FN + TP} = 1 - \text{Recall}
$$

### 2.5 F1 与 $F_\beta$：Precision 和 Recall 的折中

F1 是 Precision 和 Recall 的调和平均：

$$
\text{F1} = \frac{2 \cdot \text{Precision} \cdot \text{Recall}}{\text{Precision} + \text{Recall}}
$$

调和平均的特点是会惩罚短板：只要 Precision 或 Recall 很低，F1 就不会高。

更一般的形式是 $F_\beta$：

$$
F_\beta = (1 + \beta^2) \cdot \frac{\text{Precision} \cdot \text{Recall}}{\beta^2 \cdot \text{Precision} + \text{Recall}}
$$

当 $\beta > 1$ 时，更重视 Recall；当 $\beta < 1$ 时，更重视 Precision。

### 2.6 Balanced Accuracy：类别不均衡下的平均能力

$$
\text{Balanced Accuracy} = \frac{\text{Recall} + \text{Specificity}}{2}
$$

它把正类和负类的识别能力平均起来，比普通 Accuracy 更适合类别不均衡场景。

### 2.7 MCC：更稳健的二分类相关系数

Matthews Correlation Coefficient 的公式是：

$$
\text{MCC} =
\frac{TP \cdot TN - FP \cdot FN}
{\sqrt{(TP + FP)(TP + FN)(TN + FP)(TN + FN)}}
$$

MCC 的取值范围是 $[-1, 1]$：

| MCC | 含义 |
|---:|---|
| $1$ | 完美预测 |
| $0$ | 接近随机 |
| $-1$ | 完全相反 |

它同时考虑 TP、TN、FP、FN，在类别不均衡时通常比 Accuracy 更可靠。

### 2.8 Log Loss：概率预测的惩罚

二分类交叉熵，也叫 Log Loss：

$$
\text{Log Loss} =
-\frac{1}{n}\sum_{i=1}^{n}
\left[
y_i \log(\hat{p}_i) + (1 - y_i)\log(1 - \hat{p}_i)
\right]
$$

Log Loss 不只关心类别对不对，还关心模型有多自信。一个错误但非常自信的预测会被严重惩罚。

### 2.9 Brier Score：概率校准误差

$$
\text{Brier Score} = \frac{1}{n}\sum_{i=1}^{n}(\hat{p}_i - y_i)^2
$$

Brier Score 越小越好。它适合评估概率是否可信。例如模型说“80% 会违约”的一批用户中，最终真的大约 80% 违约，说明校准较好。

多分类 Brier Score 通常写成：

$$
\text{Brier Score}_{\text{multi}} =
\frac{1}{n}\sum_{i=1}^{n}\sum_{c=1}^{K}
\left(\mathbf{1}(y_i = c) - \hat{p}_{i,c}\right)^2
$$

注意不同工具库可能会对二分类结果做缩放，使取值范围落在 $[0,1]$；跨工具比较时要确认实现约定。

### 2.10 ECE：Expected Calibration Error

ECE 的核心思想是把样本按预测概率或置信度分成 $M$ 个桶 $B_1, B_2, \dots, B_M$，再比较“模型说的概率”和“真实发生频率”。

如果评估的是二分类正类概率校准，桶内真实正类比例为：

$$
\text{freq}(B_m) = \frac{1}{|B_m|}\sum_{i \in B_m}y_i
$$

桶内平均预测概率为：

$$
\text{conf}(B_m) = \frac{1}{|B_m|}\sum_{i \in B_m}\hat{p}_i
$$

$$
\text{ECE}_{\text{binary}} =
\sum_{m=1}^{M}\frac{|B_m|}{n}
\left|\text{freq}(B_m) - \text{conf}(B_m)\right|
$$

如果评估的是多分类“预测类别置信度”校准，则令：

$$
c_i = \max_c \hat{p}_{i,c}
$$

并使用：

$$
\text{acc}(B_m) = \frac{1}{|B_m|}\sum_{i \in B_m}\mathbf{1}(\hat{y}_i = y_i)
$$

$$
\text{conf}(B_m) = \frac{1}{|B_m|}\sum_{i \in B_m}c_i
$$

$$
\text{ECE}_{\text{confidence}} =
\sum_{m=1}^{M}\frac{|B_m|}{n}
\left|\text{acc}(B_m) - \text{conf}(B_m)\right|
$$

ECE 越小，说明模型给出的概率或置信度越接近真实频率。实际使用时，分桶方式、桶数量、是否使用等频分桶都会影响结果。

### 2.11 ROC-AUC 与 PR-AUC

ROC 曲线横轴是 FPR，纵轴是 TPR：

$$
\text{AUC} = \int_0^1 \text{TPR}(\text{FPR}) \, d\text{FPR}
$$

也可以理解为：

$$
\text{AUC} = P(s^+ > s^-) + \frac{1}{2}P(s^+ = s^-)
$$

其中 $s^+$ 是一个随机正样本的预测分数，$s^-$ 是一个随机负样本的预测分数。也就是说，AUC 衡量模型把正样本排在负样本前面的概率。

PR-AUC 则是 Precision-Recall 曲线下面积：

$$
\text{PR-AUC} = \int_0^1 \text{Precision}(\text{Recall}) \, d\text{Recall}
$$

类别极不均衡时，PR-AUC 往往比 ROC-AUC 更敏感。需要注意的是，很多工具库里的 Average Precision（AP）不是用梯形法对 PR 曲线做线性插值，而是按 Recall 增量对各阈值处的 Precision 加权：

$$
\text{AP} = \sum_n (R_n - R_{n-1})P_n
$$

因此 AP 和“梯形法 PR-AUC”数值可能不同，做实验报告时要写清楚使用的是哪一种。

### 2.12 KS：正负样本分布最大区分度

KS 常用于风控评分卡：

$$
\text{KS} = \max_t |\text{TPR}(t) - \text{FPR}(t)|
$$

它表示在所有阈值 $t$ 上，正负样本累计分布差异最大的地方。

## 三、多分类指标

多分类任务中，类别集合为：

$$
\mathcal{C} = \{1, 2, \dots, K\}
$$

预测概率通常是一个分布：

$$
\hat{\mathbf{p}}_i = [\hat{p}_{i,1}, \hat{p}_{i,2}, \dots, \hat{p}_{i,K}]
$$

且：

$$
\sum_{c=1}^{K}\hat{p}_{i,c} = 1
$$

### 3.1 Accuracy

$$
\text{Accuracy} = \frac{1}{n}\sum_{i=1}^{n}\mathbf{1}(\hat{y}_i = y_i)
$$

### 3.2 Top-K Accuracy

$$
\text{Top-K Accuracy} =
\frac{1}{n}\sum_{i=1}^{n}\mathbf{1}(y_i \in \text{TopK}(\hat{\mathbf{p}}_i))
$$

它适合类别很多、多个候选答案都有参考价值的场景，例如图像分类、搜索建议、意图识别。

如果多个类别预测分数相同，Top-K 的 tie-breaking 规则会影响结果。实际评测时应固定类别顺序和并列分数处理方式。

### 3.3 Confusion Matrix

多分类混淆矩阵 $C \in \mathbb{R}^{K \times K}$：

$$
C_{a,b} = \sum_{i=1}^{n}\mathbf{1}(y_i = a \land \hat{y}_i = b)
$$

其中 $C_{a,b}$ 表示真实类别为 $a$，预测类别为 $b$ 的样本数。对某个类别 $c$，可以把它看成“一对多”的二分类：

$$
\begin{aligned}
TP_c &= C_{c,c} \\
FP_c &= \sum_{a \ne c} C_{a,c} \\
FN_c &= \sum_{b \ne c} C_{c,b}
\end{aligned}
$$

然后得到每类的 Precision、Recall、F1：

$$
\text{Precision}_c = \frac{TP_c}{TP_c + FP_c}
$$

$$
\text{Recall}_c = \frac{TP_c}{TP_c + FN_c}
$$

$$
\text{F1}_c =
\frac{2 \cdot \text{Precision}_c \cdot \text{Recall}_c}
{\text{Precision}_c + \text{Recall}_c}
$$

### 3.4 Macro、Micro、Weighted

Macro F1 是各类别 F1 的简单平均：

$$
\text{Macro F1} = \frac{1}{K}\sum_{c=1}^{K}\text{F1}_c
$$

Weighted F1 按类别样本数加权：

$$
\text{Weighted F1} =
\sum_{c=1}^{K}\frac{n_c}{n}\text{F1}_c
$$

其中 $n_c$ 是类别 $c$ 的真实样本数。

Micro 方法先汇总所有类别的 TP、FP、FN，再计算 Precision、Recall、F1：

$$
\text{Micro Precision} =
\frac{\sum_c TP_c}{\sum_c TP_c + \sum_c FP_c}
$$

$$
\text{Micro Recall} =
\frac{\sum_c TP_c}{\sum_c TP_c + \sum_c FN_c}
$$

$$
\text{Micro F1} =
\frac{2 \cdot \text{Micro Precision} \cdot \text{Micro Recall}}
{\text{Micro Precision} + \text{Micro Recall}}
$$

简单理解：

| 指标 | 更关注什么 |
|---|---|
| Macro | 每个类别同等重要，小类不会被淹没 |
| Weighted | 按样本占比加权，更接近整体表现 |
| Micro | 每个样本同等重要，适合全局统计 |

在单标签多分类任务中，Micro Precision、Micro Recall、Micro F1 与 Accuracy 数值相同；在多标签任务中则不一定相同。

### 3.5 Multi-class Log Loss

多分类交叉熵：

$$
\text{Log Loss} =
-\frac{1}{n}\sum_{i=1}^{n}\sum_{c=1}^{K}
\mathbf{1}(y_i = c)\log \hat{p}_{i,c}
$$

因为只有真实类别对应的指示函数为 1，也可以写成：

$$
\text{Log Loss} =
-\frac{1}{n}\sum_{i=1}^{n}\log \hat{p}_{i,y_i}
$$

## 四、多标签分类指标

多标签任务中，一个样本可以同时属于多个标签。设标签数为 $L$：

$$
\mathbf{y}_i \in \{0,1\}^{L}, \quad \hat{\mathbf{y}}_i \in \{0,1\}^{L}
$$

### 4.1 Hamming Loss

$$
\text{Hamming Loss} =
\frac{1}{nL}\sum_{i=1}^{n}\sum_{\ell=1}^{L}
\mathbf{1}(y_{i,\ell} \ne \hat{y}_{i,\ell})
$$

它统计所有“样本-标签”位置上的错误比例。

### 4.2 Subset Accuracy

$$
\text{Subset Accuracy} =
\frac{1}{n}\sum_{i=1}^{n}
\mathbf{1}(\mathbf{y}_i = \hat{\mathbf{y}}_i)
$$

这个指标非常严格：一个样本的所有标签必须完全预测正确，才算正确。

### 4.3 Jaccard Score

把真实标签集合记为 $Y_i$，预测标签集合记为 $\hat{Y}_i$：

$$
\text{Jaccard}_i =
\frac{|Y_i \cap \hat{Y}_i|}
{|Y_i \cup \hat{Y}_i|}
$$

整体指标为：

$$
\text{Jaccard} =
\frac{1}{n}\sum_{i=1}^{n}\text{Jaccard}_i
$$

如果 $Y_i$ 和 $\hat{Y}_i$ 都为空，则 $|Y_i \cup \hat{Y}_i| = 0$，Jaccard 没有自然定义。工具库通常会提供 `zero_division` 之类的参数，或者返回 0 并给出警告。

## 五、回归指标

回归任务中，真实值和预测值通常是连续变量：

$$
y_i \in \mathbb{R}, \quad \hat{y}_i \in \mathbb{R}
$$

误差为：

$$
e_i = y_i - \hat{y}_i
$$

### 5.1 MAE：平均绝对误差

$$
\text{MAE} = \frac{1}{n}\sum_{i=1}^{n}|y_i - \hat{y}_i|
$$

MAE 的量纲与原始目标一致，解释起来直观，并且对极端值没有平方误差那么敏感。

### 5.2 MSE 与 RMSE

$$
\text{MSE} = \frac{1}{n}\sum_{i=1}^{n}(y_i - \hat{y}_i)^2
$$

$$
\text{RMSE} = \sqrt{\frac{1}{n}\sum_{i=1}^{n}(y_i - \hat{y}_i)^2}
$$

RMSE 会放大大误差，适合大偏差代价很高的场景。

### 5.3 MAPE、SMAPE、WAPE

MAPE：

$$
\text{MAPE}_{\%} =
\frac{100\%}{n}\sum_{i=1}^{n}
\left|\frac{y_i - \hat{y}_i}{y_i}\right|
$$

这是百分比写法。也有很多工具库使用相对值写法，不乘 $100\%$，并用一个很小的 $\epsilon$ 避免除零：

$$
\text{MAPE}_{\text{rel}} =
\frac{1}{n}\sum_{i=1}^{n}
\frac{|y_i - \hat{y}_i|}{\max(\epsilon, |y_i|)}
$$

MAPE 的问题是当 $y_i = 0$ 或接近 0 时会不稳定，所以跨工具比较时一定要确认是否乘了 $100\%$，以及是否使用了 $\epsilon$。

SMAPE：

$$
\text{SMAPE} =
\frac{100\%}{n}\sum_{i=1}^{n}
\frac{2|y_i - \hat{y}_i|}
{|y_i| + |\hat{y}_i|}
$$

当 $|y_i| + |\hat{y}_i| = 0$ 时，SMAPE 也需要额外约定，常见做法是跳过、记为 0，或者在分母加入 $\epsilon$。

WAPE：

$$
\text{WAPE} =
\frac{\sum_{i=1}^{n}|y_i - \hat{y}_i|}
{\sum_{i=1}^{n}|y_i|}
$$

WAPE 在销量预测、流量预测中常见，因为它按总体规模归一化。如果 $\sum_i |y_i| = 0$，这个指标同样没有定义，需要业务上单独处理。

### 5.4 RMSLE

$$
\text{RMSLE} =
\sqrt{
\frac{1}{n}\sum_{i=1}^{n}
\left(\log(1 + \hat{y}_i) - \log(1 + y_i)\right)^2
}
$$

RMSLE 更关注相对误差，且会降低极大值的影响。它常用于目标值跨度很大的非负目标场景，例如销量、播放量、交易额。使用时要确保 $y_i \ge 0$ 且 $\hat{y}_i \ge 0$；否则 $\log(1 + y)$ 可能没有定义。

### 5.5 $R^2$：决定系数

设真实值均值为：

$$
\bar{y} = \frac{1}{n}\sum_{i=1}^{n}y_i
$$

残差平方和：

$$
SS_{\text{res}} = \sum_{i=1}^{n}(y_i - \hat{y}_i)^2
$$

总平方和：

$$
SS_{\text{tot}} = \sum_{i=1}^{n}(y_i - \bar{y})^2
$$

则：

$$
R^2 = 1 - \frac{SS_{\text{res}}}{SS_{\text{tot}}}
$$

$R^2$ 表示模型解释了目标变量方差的多少。它可能为负，意味着模型比直接预测均值还差。如果所有 $y_i$ 都相同，则 $SS_{\text{tot}} = 0$，这个指标需要额外约定；一些工具库会把完美预测处理为 1，把非完美预测处理为 0，以避免返回非有限值。

### 5.6 Explained Variance

$$
\text{Explained Variance} =
1 - \frac{\text{Var}(y - \hat{y})}{\text{Var}(y)}
$$

它关注误差方差，而不是误差均值。如果模型存在整体偏移，Explained Variance 可能仍然较高，因此要和 MAE、RMSE 一起看。当 $\text{Var}(y)=0$ 时也需要额外约定。

### 5.7 Quantile Loss / Pinball Loss

分位数预测中，模型输出的是 $\tau$ 分位数 $\hat{q}_i$。Pinball Loss 为：

$$
L_\tau(y_i, \hat{q}_i) =
\max\left(\tau(y_i - \hat{q}_i), (\tau - 1)(y_i - \hat{q}_i)\right)
$$

整体指标：

$$
\text{Pinball Loss}_\tau =
\frac{1}{n}\sum_{i=1}^{n}L_\tau(y_i, \hat{q}_i)
$$

它适合需求预测、库存预测、风险估计等需要不确定性区间的场景。

### 5.8 Prediction Interval Coverage

如果模型输出预测区间 $[l_i, u_i]$：

$$
\text{Coverage} =
\frac{1}{n}\sum_{i=1}^{n}
\mathbf{1}(l_i \le y_i \le u_i)
$$

例如一个 90% 预测区间，理想情况下应该覆盖约 90% 的真实值。

## 六、排序与推荐指标

排序任务不是只问“预测对不对”，而是问：**排在前面的东西是否真的有用？**

对某个用户或查询 $q$，模型返回一个排序列表：

$$
[d_1, d_2, \dots, d_K]
$$

其中 $\text{rel}_i$ 表示第 $i$ 个结果的相关性。它可以是二值的：

$$
\text{rel}_i \in \{0,1\}
$$

也可以是分级的：

$$
\text{rel}_i \in \{0,1,2,3,\dots\}
$$

下面的 Precision@K、Recall@K、Hit Rate@K、AP、MAP、MRR 默认使用二值相关性。如果只有分级相关性，通常先按阈值二值化，或者直接使用 NDCG 这类原生支持分级相关性的指标。

### 6.1 Precision@K 与 Recall@K

$$
\text{Precision@K} =
\frac{1}{K}\sum_{i=1}^{K}\text{rel}_i
$$

如果该 query 总共有 $R$ 个相关文档：

$$
\text{Recall@K} =
\frac{1}{R}\sum_{i=1}^{K}\text{rel}_i
$$

Precision@K 看前 $K$ 个结果有多纯，Recall@K 看前 $K$ 个结果找回了多少。

如果某个 query 没有任何相关文档，也就是 $R = 0$，Recall@K 需要在评测协议中显式约定，常见做法是跳过这类 query 或记为 0。

### 6.2 Hit Rate@K

$$
\text{Hit Rate@K} =
\mathbf{1}\left(\sum_{i=1}^{K}\text{rel}_i > 0\right)
$$

多个 query 或多个用户时取平均：

$$
\text{Mean Hit Rate@K} =
\frac{1}{Q}\sum_{q=1}^{Q}\text{Hit Rate@K}_q
$$

它只关心前 $K$ 个结果里有没有命中，不关心命中几个。

### 6.3 AP 与 MAP

Average Precision@K：

$$
\text{AP@K} =
\frac{1}{\min(R,K)}
\sum_{i=1}^{K}\text{Precision@i} \cdot \text{rel}_i
$$

其中 $R$ 是相关结果总数。MAP 是多个 query 的平均：

$$
\text{MAP@K} =
\frac{1}{Q}\sum_{q=1}^{Q}\text{AP@K}_q
$$

AP 会奖励“相关结果尽早出现”。

上面的公式是 top-$K$ 截断版本，分母使用 $\min(R,K)$。非截断 AP 通常把求和范围扩展到完整列表，并用 $R$ 归一化。不同工具库可能在截断、重复推荐、空相关集合上有细节差异，做离线对比时要固定同一套实现。

### 6.4 MRR

设第一个相关结果的位置为 $r_q$。如果不存在相关结果，则令 reciprocal rank 为 0：

$$
\text{RR}_q =
\begin{cases}
\frac{1}{r_q}, & \text{存在相关结果} \\
0, & \text{不存在相关结果}
\end{cases}
$$

MRR 为：

$$
\text{MRR} =
\frac{1}{Q}\sum_{q=1}^{Q}\text{RR}_q
$$

它特别适合只需要一个正确答案的场景，例如问答检索、导航型搜索。也有评测协议会直接跳过没有相关结果的 query。

### 6.5 DCG 与 NDCG

DCG@K 的一般写法是：

$$
\text{DCG@K} =
\sum_{i=1}^{K}
\frac{g(\text{rel}_i)}{\log_2(i + 1)}
$$

其中 $g(\cdot)$ 是相关性的 gain 函数。常见有两种写法。

指数增益：

$$
g(\text{rel}_i) = 2^{\text{rel}_i} - 1
$$

线性增益：

$$
g(\text{rel}_i) = \text{rel}_i
$$

IDCG@K 是理想排序下的 DCG@K：

$$
\text{IDCG@K} =
\max_{\pi}
\sum_{i=1}^{K}
\frac{g(\text{rel}_{\pi(i)})}{\log_2(i + 1)}
$$

NDCG@K：

$$
\text{NDCG@K} =
\frac{\text{DCG@K}}{\text{IDCG@K}}
$$

NDCG 支持分级相关性，也会对靠前位置给更高权重，是搜索和推荐中非常常用的指标。使用时要保证 DCG 和 IDCG 采用同一种 gain 写法。如果 $\text{IDCG@K}=0$，说明该 query 在前 $K$ 评价范围内没有可获得的理想增益，NDCG 需要按评测协议记为 0 或跳过。

### 6.6 Coverage、Diversity、Novelty

推荐系统除了准，还要避免只推荐少数热门物品。

Catalog Coverage：

$$
\text{Coverage} =
\frac{|\cup_u \text{Rec}(u)|}{|\mathcal{I}|}
$$

其中 $\mathcal{I}$ 是全量物品集合，$\text{Rec}(u)$ 是给用户 $u$ 推荐过的物品集合。

列表内多样性可以写成：

$$
\text{Diversity@K} =
\frac{2}{K(K-1)}
\sum_{1 \le i < j \le K}
\left(1 - \text{sim}(d_i, d_j)\right)
$$

Novelty 可以用物品流行度的负对数表示：

$$
\text{Novelty@K} =
\frac{1}{K}\sum_{i=1}^{K}
-\log_2 P(d_i)
$$

其中 $P(d_i)$ 可以用物品被交互次数占总交互次数的比例估计。

## 七、RAG 与检索增强生成指标

RAG 可以拆成两段评估：**检索是否找对资料**，以及**生成是否忠实使用资料**。

### 7.1 Retrieval Recall@K

设标准相关文档集合为 $G_q$，检索前 $K$ 个文档为 $R_q^K$：

$$
\text{Retrieval Recall@K} =
\frac{|G_q \cap R_q^K|}{|G_q|}
$$

这个指标回答：应该被找回的资料，有多少进入了上下文。

### 7.2 Retrieval Precision@K

$$
\text{Retrieval Precision@K} =
\frac{|G_q \cap R_q^K|}{K}
$$

它回答：塞进上下文的资料里，有多少真的相关。

### 7.3 Context Precision / Context Utilization

上面的 Retrieval Precision@K 是集合级指标，不考虑相关 chunk 的位置。RAG 评估中还常用 rank-aware 的 Context Precision。设第 $k$ 个检索 chunk 是否相关为：

$$
v_k \in \{0,1\}
$$

则：

$$
\text{Precision@k} =
\frac{\sum_{i=1}^{k}v_i}{k}
$$

$$
\text{Context Precision@K} =
\frac{\sum_{k=1}^{K}\text{Precision@k} \cdot v_k}
{\sum_{k=1}^{K}v_k}
$$

如果 $\sum_k v_k = 0$，需要约定该 query 的分数为 0 或跳过。若 $v_k$ 是通过“检索 chunk 是否支持参考答案”判断的，通常叫 Context Precision；若 $v_k$ 是通过“检索 chunk 是否支持模型生成答案”判断的，RAGAS 等工具也会称为 Context Utilization。它们的公式形式相同，差别在于相关性判断参照的是 reference 还是 response。

### 7.4 Context Recall

Context Recall 关注参考答案中的必要信息是否被检索上下文覆盖。把参考答案拆成可核验声明：

$$
G_q = \{g_1, g_2, \dots, g_r\}
$$

若声明 $g_j$ 能被检索上下文 $C_q$ 支持，记为：

$$
\text{support}(g_j, C_q) = 1
$$

则：

$$
\text{Context Recall} =
\frac{1}{r}\sum_{j=1}^{r}\text{support}(g_j, C_q)
$$

它回答的问题是：标准答案需要的事实，检索阶段是否已经找到了。

### 7.5 Faithfulness / Groundedness

把模型答案拆成若干可核验声明：

$$
A_q = \{a_1, a_2, \dots, a_m\}
$$

若声明 $a_j$ 能被上下文 $C_q$ 支持，记为：

$$
\text{support}(a_j, C_q) = 1
$$

则：

$$
\text{Faithfulness} =
\frac{1}{m}\sum_{j=1}^{m}
\text{support}(a_j, C_q)
$$

幻觉率可以看成它的补集：

$$
\text{Hallucination Rate} =
1 - \text{Faithfulness}
$$

注意 Context Recall 看的是“参考答案是否被上下文覆盖”，Faithfulness 看的是“模型答案是否被上下文支持”。前者主要诊断检索，后者主要诊断生成。

### 7.6 Citation Precision 与 Citation Recall

Citation Precision：

$$
\text{Citation Precision} =
\frac{\text{正确引用数}}{\text{总引用数}}
$$

Citation Recall：

$$
\text{Citation Recall} =
\frac{\text{带有正确引用的关键声明数}}{\text{需要引用的关键声明数}}
$$

在严肃知识问答、法律、医疗、金融等场景中，引用质量经常比语言流畅度更重要。

## 八、LLM 应用与 Agent 指标

LLM 应用的指标更像系统指标，而不只是模型指标。因为最终质量不仅取决于模型，还取决于提示词、上下文、工具、权限、输出解析和用户反馈。

### 8.1 任务准确率

如果每个样本都有可自动判断的标准答案：

$$
\text{Accuracy} =
\frac{1}{n}\sum_{i=1}^{n}
\mathbf{1}(\text{judge}(\hat{a}_i, a_i) = 1)
$$

其中 $\hat{a}_i$ 是模型答案，$a_i$ 是标准答案，$\text{judge}$ 可以是规则、单元测试、人工标注，也可以是经过校准的评审模型。

### 8.2 Exact Match

$$
\text{EM} =
\frac{1}{n}\sum_{i=1}^{n}
\mathbf{1}(\text{normalize}(\hat{a}_i) = \text{normalize}(a_i))
$$

Exact Match 很严格，适合短答案、结构化抽取、代码题单测结果等场景。

### 8.3 格式合规率

对要求 JSON、XML、函数参数、表格、代码块等固定格式的任务：

$$
\text{Format Compliance Rate} =
\frac{1}{n}\sum_{i=1}^{n}
\mathbf{1}(\text{valid\_format}(\hat{a}_i))
$$

这类指标非常工程化，但非常重要。格式错了，内容再好也可能无法进入下游系统。

### 8.4 有害内容比例

$$
\text{Harmful Rate} =
\frac{1}{n}\sum_{i=1}^{n}
\mathbf{1}(\text{harmful}(\hat{a}_i))
$$

如果测试集只包含恶意请求，还可以统计越狱成功率：

$$
\text{Jailbreak Success Rate} =
\frac{\text{产生违规输出的恶意请求数}}
{\text{恶意请求总数}}
$$

### 8.5 拒答正确率与过度拒答率

对应该拒答的请求集合 $\mathcal{U}$：

$$
\text{Correct Refusal Rate} =
\frac{1}{|\mathcal{U}|}
\sum_{i \in \mathcal{U}}
\mathbf{1}(\text{refuse}(\hat{a}_i))
$$

对应该回答的正常请求集合 $\mathcal{S}$：

$$
\text{Over-refusal Rate} =
\frac{1}{|\mathcal{S}|}
\sum_{i \in \mathcal{S}}
\mathbf{1}(\text{refuse}(\hat{a}_i))
$$

安全系统不能只看“拒得够不够多”，还要看是否误伤正常用户。

### 8.6 人工评分

如果人工或评审模型给每个回答打分 $s_i \in [1, 5]$：

$$
\text{Mean Human Score} =
\frac{1}{n}\sum_{i=1}^{n}s_i
$$

如果比较两个系统 A 和 B，可以用胜率：

$$
\text{Win Rate}(A, B) =
\frac{
\sum_{i=1}^{n}\mathbf{1}(A_i \succ B_i)
}{
n
}
$$

也可以排除平局：

$$
\text{Non-tie Win Rate}(A, B) =
\frac{
\sum_i \mathbf{1}(A_i \succ B_i)
}{
\sum_i \mathbf{1}(A_i \succ B_i) + \sum_i \mathbf{1}(B_i \succ A_i)
}
$$

### 8.7 Agent 任务完成率

Agent 的核心指标通常不是某一句回答是否漂亮，而是任务是否完成：

$$
\text{Task Success Rate} =
\frac{\text{成功完成任务数}}{\text{任务总数}}
$$

如果任务有不同难度或不同业务价值，可以加权：

$$
\text{Weighted Success Rate} =
\frac{\sum_{i=1}^{n}w_i \cdot \mathbf{1}(\text{success}_i)}
{\sum_{i=1}^{n}w_i}
$$

### 8.8 工具调用指标

工具选择准确率：

$$
\text{Tool Selection Accuracy} =
\frac{\text{正确选择工具的次数}}
{\text{需要选择工具的决策次数}}
$$

工具调用成功率：

$$
\text{Tool Call Success Rate} =
\frac{\text{成功执行的工具调用次数}}
{\text{工具调用总次数}}
$$

工具参数正确率：

$$
\text{Tool Argument Accuracy} =
\frac{\text{参数正确的工具调用次数}}
{\text{工具调用总次数}}
$$

如果有标准工具调用集合 $E_i$ 和实际工具调用集合 $A_i$，还可以按集合匹配计算 Tool Call F1。这里的“匹配”可以要求工具名和参数完全一致，也可以使用任务自定义的近似匹配函数。

$$
\text{Tool Precision}_i =
\frac{|A_i \cap E_i|}{|A_i|}
$$

$$
\text{Tool Recall}_i =
\frac{|A_i \cap E_i|}{|E_i|}
$$

$$
\text{Tool Call F1}_i =
\frac{2 \cdot \text{Tool Precision}_i \cdot \text{Tool Recall}_i}
{\text{Tool Precision}_i + \text{Tool Recall}_i}
$$

当 $|A_i| = 0$ 或 $|E_i| = 0$ 时，需要像分类指标一样显式约定 zero-division 处理。如果工具调用顺序也重要，应该把顺序是否对齐纳入评分；如果工具会产生真实副作用，还应该记录危险动作拦截率：

$$
\text{Guardrail Block Rate} =
\frac{\text{被 guardrail 拦截的危险动作数}}
{\text{危险动作总数}}
$$

只看拦截率还不够，安全评测也应同时统计误拦截率：

$$
\text{False Block Rate} =
\frac{\text{被错误拦截的安全动作数}}
{\text{安全动作总数}}
$$

### 8.9 错误恢复率

Agent 会遇到工具失败、上下文不足、权限不足、环境异常。恢复能力可以定义为：

$$
\text{Recovery Rate} =
\frac{\text{失败后最终恢复并完成的任务数}}
{\text{发生可恢复失败的任务数}}
$$

### 8.10 过程效率

平均步骤数：

$$
\text{Avg Steps} =
\frac{1}{n}\sum_{i=1}^{n}\text{steps}_i
$$

成功任务平均步骤数：

$$
\text{Avg Steps per Success} =
\frac{\sum_i \text{steps}_i \cdot \mathbf{1}(\text{success}_i)}
{\sum_i \mathbf{1}(\text{success}_i)}
$$

Token 成本：

$$
\text{Token Cost} =
\sum_i
\left(
c_{\text{in}} \cdot \text{input\_tokens}_i
+
c_{\text{out}} \cdot \text{output\_tokens}_i
\right)
$$

单位成功成本：

$$
\text{Cost per Success} =
\frac{\text{Total Cost}}
{\text{成功完成任务数}}
$$

很多 Agent demo 看起来很强，但如果每个简单任务都要十几轮工具调用，线上就可能不可用。

## 九、线上系统指标

模型上线后，还要看系统工程指标。

### 9.1 延迟分位数

设请求延迟随机变量为 $T$，其分布函数为 $F(t)$。$p$ 分位延迟定义为：

$$
\text{Latency}_{p} =
\inf \{t \mid F(t) \ge p\}
$$

常见的是 P50、P95、P99。对用户体验来说，P95/P99 往往比平均延迟更重要。

### 9.2 吞吐量

$$
\text{QPS} =
\frac{\text{请求总数}}{\text{时间窗口秒数}}
$$

### 9.3 错误率

$$
\text{Error Rate} =
\frac{\text{失败请求数}}{\text{请求总数}}
$$

### 9.4 可用性

$$
\text{Availability} =
\frac{\text{Uptime}}{\text{Uptime} + \text{Downtime}}
$$

例如 99.9% 可用性，意味着一年大约有 8.76 小时不可用。

### 9.5 缓存命中率

$$
\text{Cache Hit Rate} =
\frac{\text{缓存命中次数}}{\text{缓存查询总次数}}
$$

对于 LLM 应用，Prompt Cache、RAG Cache、工具结果缓存都可能显著影响成本和延迟。

## 十、怎么选指标

没有一个指标适合所有任务。更好的做法是把指标分成三层：

| 层次 | 作用 | 例子 |
|---|---|---|
| **主指标** | 决定是否真的变好 | Task Success Rate、NDCG@K、Recall、MAE |
| **诊断指标** | 定位为什么变好或变坏 | Tool Call Success Rate、Context Recall、Per-class F1 |
| **护栏指标** | 防止优化方向伤害系统 | Harmful Rate、Latency P99、Cost per Success、Over-refusal Rate |

几个常用选择原则：

- 类别均衡、错分代价接近：优先看 Accuracy。
- 类别不均衡：优先看 PR-AUC、F1、Balanced Accuracy、MCC。
- 漏报代价高：优先看 Recall、F2、FNR。
- 误报代价高：优先看 Precision、Specificity、F0.5。
- 概率要可信：优先看 Log Loss、Brier Score、ECE。
- 回归目标量纲重要：优先看 MAE、RMSE。
- 目标跨度很大：考虑 RMSLE、MAPE、WAPE。
- 排序推荐：优先看 NDCG@K、MAP@K、MRR、Hit Rate@K。
- RAG：拆开看 Retrieval Recall、Retrieval Precision、Faithfulness、Citation Precision/Recall。
- Agent：优先看 Task Success Rate，再看工具调用、恢复、成本、延迟和安全。

最后要记住：**离线指标是代理目标，不是真实目标本身**。一个模型离线 AUC 提升，不代表线上转化率一定提升；一个 Agent 的自动评分变高，也不代表用户真的更信任它。真正可靠的评估，应该把离线评测、线上实验、人工审查和系统监控连成一条闭环。

## 总结

评估指标的本质，是把复杂行为压缩成可比较、可优化、可追踪的信号。

传统机器学习任务中，我们用 Accuracy、F1、AUC、MAE、RMSE、NDCG 这些指标描述预测质量；到了 LLM 和 Agent 应用，评估对象从“模型输出”扩展到了“系统行为”：有没有找对资料、有没有忠实回答、有没有按格式输出、有没有正确调用工具、有没有在高风险动作前停下来。

所以，做模型评估时不要只问“哪个指标最高”，而要先问：

> 这个指标改善时，真实用户价值是否也会改善？

如果答案不确定，就需要补充诊断指标和护栏指标。指标体系不是为了让 dashboard 更热闹，而是为了让系统在持续迭代时，仍然知道自己有没有朝正确方向走。

## 参考资料

- [scikit-learn: Metrics and scoring](https://scikit-learn.org/stable/modules/model_evaluation.html)
- [scikit-learn: Precision / Recall / F-beta](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.fbeta_score.html)
- [scikit-learn: Top-K Accuracy](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.top_k_accuracy_score.html)
- [scikit-learn: Log Loss](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.log_loss.html)
- [scikit-learn: Brier Score Loss](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.brier_score_loss.html)
- [scikit-learn: ROC AUC](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.roc_auc_score.html)
- [scikit-learn: Mean Absolute Percentage Error](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.mean_absolute_percentage_error.html)
- [scikit-learn: Jaccard Score](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.jaccard_score.html)
- [scikit-learn: R2 Score](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.r2_score.html)
- [scikit-learn: DCG / NDCG](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.dcg_score.html)
- [scikit-learn: Average Precision](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.average_precision_score.html)
- [Microsoft Recommenders: Evaluation module](https://recommenders-team.github.io/recommenders/evaluation.html)
- [RAGAS: Context Precision](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_precision/)
- [RAGAS: Context Recall](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_recall/)
- [RAGAS: Faithfulness](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/faithfulness/)
- [RAGAS: Agentic or Tool use metrics](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/agents/)
- [OpenAI: Agent evals](https://platform.openai.com/docs/guides/agent-evals)
- [OpenAI: Graders](https://platform.openai.com/docs/guides/graders/)
- [OpenAI: Trace grading](https://platform.openai.com/docs/guides/trace-grading)
