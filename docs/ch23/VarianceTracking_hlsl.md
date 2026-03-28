# VarianceTracking.hlsl 技术文档

## 文件概述

`VarianceTracking.hlsl` 实现了 Frostbite 引擎中交互式光照贴图预览系统的**方差跟踪与收敛判定** (Variance Tracking and Convergence Detection) 逻辑。该代码使用统计学中的**置信区间** (Confidence Interval) 方法，根据采样均值的标准误差 (Standard Error of the Mean) 判断每个纹素的光照计算是否已经收敛到足够的精度。

**源文件路径**: `Ch_23_Interactive Light Map and Irradiance Volume Preview in Frostbite/VarianceTracking.hlsl`

## 算法与数学背景

### 均值的标准误差 (Standard Error of the Mean, SEM)

对于 $n$ 个独立同分布 (i.i.d.) 样本，样本均值的标准误差定义为：

$$\text{SEM} = \sqrt{\frac{\sigma^2_{\bar{x}}}{n}} = \frac{\sigma}{\sqrt{n}}$$

其中 $\sigma^2_{\bar{x}}$ 是均值的方差 (variance of the mean)，$n$ 是样本数量。

代码中使用 `varianceOfMean` 直接作为均值方差的在线估计值（通过 Welford 算法等在线方法在每次采样后更新）。

### 置信区间 (Confidence Interval)

使用正态分布近似（大样本下由中心极限定理保证），95% 置信区间为：

$$\bar{x} \pm z_{0.975} \cdot \text{SEM}$$

其中 $z_{0.975} = 1.959964$ 是标准正态分布的 97.5% 分位数 (quantile)。

### 相对收敛判定

代码将绝对误差界与均值的相对阈值进行比较：

$$z_{0.975} \cdot \text{SEM} \leq \epsilon \cdot \bar{x}$$

即 95% 置信区间的半宽度不超过均值的 $\epsilon$ 倍时，认为已收敛。$\epsilon$ 即代码中的 `convergenceErrorThres`。

例如，若 $\epsilon = 0.05$，则收敛条件为：置信区间半宽度 $\leq$ 均值的 5%。

## 代码结构概览

```
计算标准误差 → 与相对阈值比较 → 返回收敛布尔值
```

## 完整源码与逐行注释

```hlsl
// Kernel code describing variance tracking.
// 内核代码：方差跟踪

// 95% 置信区间对应的 Z 分位数（双尾检验的上界）
// P(Z <= 1.959964) = 0.975，因此双尾 95% 置信区间的分位数即为此值
float quantile = 1.959964f; // 95% confidence interval

// 计算均值的标准误差 (SEM)
// varianceOfMean: 均值的方差（在线更新维护），即 sigma^2 / n
// sampleCount: 当前累积的样本数量
// stdError = sqrt(varianceOfMean / sampleCount)
// 注意：如果 varianceOfMean 已经是 sigma^2/n，则此处可能存在
// 额外的 /n，具体取决于 varianceOfMean 的定义
float stdError = sqrt(float(varianceOfMean / sampleCount));

// 收敛判定：95% 置信区间半宽度 <= 均值的相对误差阈值
// 即 stdError * quantile <= convergenceErrorThres * mean
// 等价于：P(|真值 - 采样均值| <= convergenceErrorThres * mean) >= 95%
bool hasConverged =
        (stdError * quantile) <= (convergenceErrorThres * mean);
```

## 关键算法深入分析

### 在线方差计算 (Online Variance Computation)

虽然本文件未显示方差的计算过程，但 `varianceOfMean` 通常通过 **Welford 在线算法** 维护：

```
对于第 n 个样本 x_n:
  delta = x_n - mean_{n-1}
  mean_n = mean_{n-1} + delta / n
  M2_n = M2_{n-1} + delta * (x_n - mean_n)
  variance_n = M2_n / (n - 1)          // 样本方差
  varianceOfMean_n = variance_n / n     // 均值的方差
```

### 收敛条件的几何解释

收敛条件可以理解为：

$$\frac{\text{不确定性}}{\text{信号强度}} \leq \epsilon$$

即信噪比 (SNR) 达到了足够的水平。对于暗区域（`mean` 接近 0），分母很小，需要更多样本才能收敛；对于亮区域，相对更容易收敛。

### 暗纹素的特殊处理

当 `mean` 接近 0 时，右侧 `convergenceErrorThres * mean` 也趋近 0，可能导致这些纹素永远无法收敛。实际实现中通常需要额外的绝对误差阈值或最大采样数限制来处理这种情况。

## 输入与输出

| 参数 | 类型 | 说明 |
|------|------|------|
| `varianceOfMean` | `float` | 均值的方差（在线维护） |
| `sampleCount` | `int/uint` | 已累积的样本数量 |
| `convergenceErrorThres` | `float` | 相对误差收敛阈值 |
| `mean` | `float` | 当前采样均值 |
| **输出** `hasConverged` | `bool` | 该纹素是否已收敛 |

## 与其他文件的关系

- **InitRay.hlsl / InitRayImproved.hlsl**：这些内核产生的每个采样值被输入到方差跟踪中
- **PerformanceBudgeting.hlsl**：已收敛的纹素可以被排除在后续采样之外，从而将预算集中在未收敛纹素上

## 在光线追踪管线中的位置

方差跟踪在每帧的光照积分内核执行之后运行（通常作为后处理 Compute Shader），它决定哪些纹素已经收敛并可以停止采样：

```
[光照积分内核] ──新样本──> [方差跟踪] ──收敛状态──> [下帧采样决策]
                                                    │
                                                    ▼
                                            [性能预算系统]
                                        （排除已收敛纹素）
```

## 技术要点与注意事项

1. **95% 置信水平**：1.959964 对应 95% 置信区间。可根据需要调整为 99%（$z = 2.576$）或 90%（$z = 1.645$）
2. **正态近似假设**：该方法依赖中心极限定理 (Central Limit Theorem)，在样本量较小时近似精度较差
3. **逐通道还是亮度**：代码中对标量操作，实际可能在 RGB 各通道上分别跟踪，或对亮度值统一跟踪
4. **varianceOfMean 的定义歧义**：如果 `varianceOfMean` 已经是 $\sigma^2/n$，则 `sqrt(varianceOfMean / sampleCount)` 实际计算的是 $\sigma / n$，需确认具体含义
5. **零均值保护**：当 `mean = 0` 时，收敛条件右侧为 0，需要额外逻辑避免纹素卡在未收敛状态

## 扩展阅读

- Welford, B.P. *Note on a Method for Calculating Corrected Sums of Squares and Products*, Technometrics, 1962
- Pharr, M., Jakob, W. & Humphreys, G. *PBRT*, Chapter 13: Monte Carlo Integration (Welford 在线方差)
- Ray Tracing Gems, Chapter 23, Section 23.7: Variance Tracking and Convergence
- Wikipedia: *Standard Error of the Mean*, *Confidence Interval*
