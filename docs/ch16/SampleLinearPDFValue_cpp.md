# SampleLinearPDFValue.cpp 文档

## 1. 变换概述

| 项目 | 内容 |
|------|------|
| **名称** | 线性插值采样 PDF 值计算 (Linear Interpolation Sampling PDF Value) |
| **用途** | 计算线性插值采样在给定点 $x$ 处的概率密度函数值 |
| **应用场景** | 多重重要性采样 (MIS) 权重计算、蒙特卡洛估计器的 PDF 评估、采样算法正确性验证 |

---

## 2. 数学推导

### 2.1 线性 PDF

在 $[0,1]$ 上由端点 $a$（$x=0$处）和 $b$（$x=1$处）定义的归一化线性 PDF：

$$
f(x) = \frac{\text{lerp}(x, a, b)}{(a+b)/2} = \frac{2[(1-x)a + xb]}{a+b}
$$

其中 $\text{lerp}(x, a, b) = (1-x)a + xb$。

### 2.2 验证归一化

$$
\int_0^1 f(x)\, dx = \frac{2}{a+b} \int_0^1 [(1-x)a + xb]\, dx = \frac{2}{a+b} \cdot \frac{a+b}{2} = 1 \quad \checkmark
$$

### 2.3 范围外

当 $x < 0$ 或 $x > 1$ 时，$f(x) = 0$。

---

## 3. 完整代码与逐行注释

```cpp
if (x < 0 || x > 1) return 0;
// 范围检查：若 x 不在 [0,1] 内，PDF 值为 0

return lerp(x, a, b) / ((a + b) / 2);
// PDF = lerp(x, a, b) / ((a+b)/2)
// lerp(x, a, b) = (1-x)*a + x*b：在 x 处的线性插值值
// (a+b)/2：归一化常数（线性函数在 [0,1] 上的平均值）
// 等价于 2*lerp(x,a,b) / (a+b)
```

---

## 4. 输入与输出

### 输入参数

| 参数 | 类型 | 范围 | 说明 |
|------|------|------|------|
| `x` | `float` | $(-\infty, +\infty)$ | 需要评估 PDF 的点 |
| `a` | `float` | $[0, +\infty)$ | $x=0$ 处的分布值 |
| `b` | `float` | $[0, +\infty)$ | $x=1$ 处的分布值 |

### 输出

| 输出 | 类型 | 说明 |
|------|------|------|
| 返回值 | `float` | 在点 $x$ 处的 PDF 值；$x \notin [0,1]$ 时返回 0 |

---

## 5. 应用场景

- **MIS 权重**：组合线性采样与其他策略时，评估线性策略的 PDF
- **蒙特卡洛权重**：计算 $w_i = f(x_i) / \text{pdf}(x_i)$ 时需要 PDF 值
- **自检验证**：通过直方图测试验证 `SampleLinear` 采样的正确性——采样频率应与 PDF 值成正比

---

## 6. 与相关变换的对比

| 函数 | 功能 | 关系 |
|------|------|------|
| **线性 PDF (本方法)** | 评估 $f(x)$ | 配套 `SampleLinear` 使用 |
| **线性采样** (`SampleLinear.cpp`) | 从线性分布中生成样本 | 采样函数 |
| **双线性 PDF** (`SampleBilinearInterpolationPDFValue.cpp`) | 二维 PDF 评估 | 本方法的二维推广 |
| **帐篷 PDF** (`TentFunctionPDFValue.cpp`) | 帐篷分布 PDF 评估 | 由两段线性 PDF 拼接而成 |

> **数值注意**：当 $a + b = 0$（即 $a = b = 0$）时，存在除零风险。在实际使用中应确保至少一个端点值为正。
