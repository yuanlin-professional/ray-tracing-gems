# TentFunctionPDFValue.cpp 文档

## 1. 变换概述

| 项目 | 内容 |
|------|------|
| **名称** | 帐篷函数 PDF 值计算 (Tent Function PDF Value) |
| **用途** | 计算帐篷函数 (tent function) 分布在给定点 $x$ 处的概率密度函数值 |
| **应用场景** | 多重重要性采样 (MIS) 权重计算、滤波器权重评估、采样验证 |

---

## 2. 数学推导

### 2.1 帐篷函数 PDF

半径为 $r$ 的帐篷函数定义为：

$$
f(x) = \begin{cases}
\frac{1}{r} - \frac{|x|}{r^2} & \text{if } |x| < r \\
0 & \text{otherwise}
\end{cases}
$$

### 2.2 验证归一化

$$
\int_{-r}^{r} f(x)\, dx = 2\int_0^r \left(\frac{1}{r} - \frac{x}{r^2}\right) dx = 2\left[\frac{x}{r} - \frac{x^2}{2r^2}\right]_0^r = 2\left(1 - \frac{1}{2}\right) = 1 \quad \checkmark
$$

### 2.3 峰值与边界

- 峰值在 $x = 0$：$f(0) = 1/r$
- 边界值：$f(\pm r) = 0$

---

## 3. 完整代码与逐行注释

```cpp
if (abs(x) >= r) return 0;
// 范围检查：若 |x| >= r，则超出帐篷函数的支撑范围 (support)，PDF 为 0

return 1 / r - abs(x) / (r * r);
// 帐篷函数 PDF = 1/r - |x|/r²
// 1/r：帐篷顶部（x=0处）的值
// abs(x)/(r*r)：线性衰减项，距中心越远 PDF 越小
// 当 x=0 时返回 1/r（最大值）
// 当 |x|→r 时返回趋近 0
```

---

## 4. 输入与输出

### 输入参数

| 参数 | 类型 | 范围 | 说明 |
|------|------|------|------|
| `x` | `float` | $(-\infty, +\infty)$ | 需要评估 PDF 的点 |
| `r` | `float` | $(0, +\infty)$ | 帐篷函数的半径 |

### 输出

| 输出 | 类型 | 说明 |
|------|------|------|
| 返回值 | `float` | 在点 $x$ 处的 PDF 值；$|x| \ge r$ 时返回 0 |

---

## 5. 应用场景

- **MIS 权重计算**：当帐篷采样与其他策略组合时，需要计算帐篷分布在每个样本位置的 PDF
- **滤波器权重**：在图像重建中评估帐篷滤波器对每个样本的贡献权重
- **采样验证**：通过将采样直方图与本 PDF 函数对比来验证 `TentFunction.cpp` 的正确性

---

## 6. 与相关变换的对比

| 函数 | 功能 | 关系 |
|------|------|------|
| **帐篷 PDF (本方法)** | 评估帐篷分布 $f(x)$ | 配套 `TentFunction.cpp` 使用 |
| **帐篷采样** (`TentFunction.cpp`) | 从帐篷分布中生成样本 | 采样函数 |
| **线性 PDF** (`SampleLinearPDFValue.cpp`) | 一维线性分布 PDF | 帐篷函数可视为两段线性 PDF 的拼接 |
| **正态 PDF** (`NormalDistributionPDFValue.cpp`) | 高斯分布 PDF | 类似的钟形但无限支撑、更平滑 |

> **几何理解**：帐篷函数可以看作两个线性函数（从 1 到 0）的对称拼接。左半部分和右半部分各自是 `SampleLinear(u, 1, 0)` 的 PDF（经缩放和平移后）。
