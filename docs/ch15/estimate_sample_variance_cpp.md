# estimate_sample_variance.cpp — 样本方差估计

> 源文件：`Ch_15_On_the_Importance_of_Sampling/estimate_sample_variance.cpp`

---

## 1. 文件概述

`estimate_sample_variance.cpp` 实现了样本方差 (Sample Variance) 的单 Pass 估计方法。该函数在渲染中用于评估蒙特卡洛积分的收敛程度。

---

## 2. 算法与数学背景

### 样本方差公式

对于 $n$ 个样本 $x_1, x_2, \ldots, x_n$，无偏样本方差为：

$$s^2 = \frac{1}{n-1} \sum_{i=1}^{n} (x_i - \bar{x})^2$$

展开后得到计算友好的形式：

$$s^2 = \frac{1}{n-1} \left( \sum_{i=1}^{n} x_i^2 - \frac{(\sum_{i=1}^{n} x_i)^2}{n} \right)$$

即：

$$s^2 = \frac{\sum x_i^2}{n-1} - \frac{(\sum x_i)^2}{n(n-1)}$$

---

## 3. 完整代码与逐行注释

```cpp
float estimate_sample_variance(float samples[], int n) {
   float sum = 0, sum_sq = 0;              // 样本和与样本平方和
   for (int i = 0; i < n; ++i) {
     sum += samples[i];                     // 累加样本值
     sum_sq += samples[i] * samples[i];     // 累加样本值的平方
   }
   return sum_sq / (n - 1) -               // Σx²/(n-1)
          sum * sum / ((n - 1) * n);        // - (Σx)²/[n(n-1)]
}
```

---

## 4. 输入与输出

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `samples` | `float[]` | 输入 | 样本值数组 |
| `n` | `int` | 输入 | 样本数量（必须 > 1） |
| **返回值** | `float` | 输出 | 无偏样本方差 $s^2$ |

---

## 5. 关键算法深入分析

### 数值稳定性

此公式在 $n$ 较大且样本均值远离零时可能存在数值精度问题（"catastrophic cancellation"），因为 $\sum x_i^2$ 和 $(\sum x_i)^2/n$ 可能都很大但差很小。对于渲染中的方差估计（通常 $n$ 较小、值域有限），此问题不显著。

### 与 Welford 算法的对比

Welford 在线算法的数值稳定性更好，但需要维护更多状态。本实现更简洁，适合样本数不大的场景。

### 应用

- **自适应采样**：方差大的像素分配更多样本
- **渲染收敛判断**：方差低于阈值时停止采样
- **降噪权重**：方差信息指导空间/时间滤波

---

## 6. 与其他文件的关系

本文件为独立工具函数，可用于任何蒙特卡洛估计器的方差计算。

---

## 7. 扩展阅读

- **书中章节**：Chapter 15, "On the Importance of Sampling"
- **Welford 算法**：Welford, B.P., "Note on a Method for Calculating Corrected Sums of Squares and Products," *Technometrics*, 4(3), 1962
