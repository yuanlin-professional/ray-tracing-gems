# samplePiecewiseConstantArray.cpp 文档

## 1. 变换概述

| 项目 | 内容 |
|------|------|
| **名称** | 分段常数数组采样 (Piecewise Constant Array Sampling) |
| **用途** | 给定一组分段常数 PDF 值，构建 CDF 并使用二分查找 (binary search) 从该分布中采样 |
| **应用场景** | 环境贴图行采样、1D 光源分布采样、任意离散/分段常数分布的采样 |

---

## 2. 数学推导

### 2.1 分段常数 PDF

给定 PDF 值数组 $p_0, p_1, \ldots, p_{n-1}$（未归一化），第 $i$ 段 $[i/n, (i+1)/n]$ 上的 PDF 值为常数 $p_i$。

### 2.2 CDF 构建

归一化后的 CDF：

$$
C_0 = 0, \quad C_k = \frac{\sum_{i=0}^{k-1} p_i}{\sum_{i=0}^{n-1} p_i}, \quad k = 1, \ldots, n
$$

CDF 数组长度为 $n+1$（比 PDF 多一个元素），$C_0 = 0$，$C_n = 1$。

### 2.3 逆 CDF（二分查找）

给定 $u \in [0,1)$，找到最大的 $k$ 使得 $C_k \le u$：

$$
k = \text{upper\_bound}(C, u) - 1
$$

然后在该段内线性映射得到重映射的 $u$：

$$
u_{\text{remapped}} = \frac{u - C_k}{C_{k+1} - C_k}
$$

---

## 3. 完整代码与逐行注释

```cpp
vector<float> makePiecewiseConstantCDF(vector<float> pdf) {
// 根据 PDF 数组构建归一化 CDF

    float total = 0.0;
    // 累计总和初始化为 0

    // CDF 比 PDF 多一个元素
    vector<float> cdf { 0.0 };
    // CDF 的第一个元素为 0（C_0 = 0）

    // 计算累积和
    for (auto value : pdf) cdf.push_back(total += value);
    // 每个 CDF 元素 = 前面所有 PDF 值的累积和
    // total 在循环中不断累加

    // 归一化
    for (auto& value : cdf) value /= total;
    // 将所有 CDF 值除以总和，使 CDF 范围为 [0, 1]
    // 最后一个元素 cdf[n] = 1.0

    return cdf;
}

int samplePiecewiseConstantArray(float u, vector<float> cdf,
            float* uRemapped)
{
// 使用 CDF 进行逆变换采样

    // 使用（已排序的）CDF 找到采样值 u 左侧的数据点
    int offset = upper_bound(cdf.begin(), cdf.end(), u) -
    cdf.begin() - 1;
    // upper_bound 返回第一个大于 u 的位置
    // 减去 begin() 得到索引，再减 1 得到左侧区间的起始索引
    // 即找到满足 cdf[offset] <= u < cdf[offset+1] 的 offset

    *uRemapped = (u - cdf[offset]) / (cdf[offset+1] - cdf[offset]);
    // 在区间 [cdf[offset], cdf[offset+1]) 内线性重映射 u 到 [0,1)
    // 该值可用于后续的连续采样（如在选中区间内的精确位置）

    return offset;
    // 返回被选中的区间索引
}
```

---

## 4. 输入与输出

### `makePiecewiseConstantCDF`

| 参数 | 类型 | 说明 |
|------|------|------|
| `pdf` | `vector<float>` | 输入：分段常数 PDF 值数组（未归一化） |
| 返回值 | `vector<float>` | 输出：归一化 CDF 数组（长度 = pdf.size() + 1） |

### `samplePiecewiseConstantArray`

| 参数 | 类型 | 范围 | 说明 |
|------|------|------|------|
| `u` | `float` | $[0, 1)$ | 均匀随机变量 |
| `cdf` | `vector<float>` | $[0, 1]$ | 由 `makePiecewiseConstantCDF` 生成的 CDF |
| `*uRemapped` | `float*` | $[0, 1)$ | 输出：区间内的重映射随机数 |
| 返回值 | `int` | — | 输出：选中的区间索引 |

---

## 5. 应用场景

- **环境贴图行采样**：将环境贴图每行的总亮度构成 1D 分段常数分布，先采样行再采样列
- **1D 光强分布**：对沿线的光源亮度变化进行采样
- **频谱采样**：对光谱功率分布 (SPD, Spectral Power Distribution) 采样波长
- **天空模型**：对天空辐照度 (irradiance) 的角度分布采样

---

## 6. 与相关变换的对比

| 方法 | 预处理 | 查找方式 | 时间复杂度 | 特点 |
|------|--------|---------|-----------|------|
| **分段常数 CDF (本方法)** | CDF 构建 $O(n)$ | 二分查找 `upper_bound` | $O(\log n)$ | 标准且高效；CDF 数组紧凑 |
| **线性搜索** (`SampleDiscrete.cpp`) | 求和 $O(n)$ | 线性扫描 | $O(n)$ | 更简单但慢 |
| **层级扭曲** (`HierarchicalWarping.cpp`) | 树构建 $O(n)$ | 树遍历 | $O(\log n)$ | 更灵活但内存开销大 |
| **别名方法 (Alias Method)** | $O(n)$ 表构建 | 直接查表 | $O(1)$ | 最快但构建复杂 |

> **两步法**：本文件提供了两个函数——`makePiecewiseConstantCDF` 用于预处理（通常只做一次），`samplePiecewiseConstantArray` 用于运行时采样（可调用多次）。这种分离设计在分布不变的场景（如静态环境贴图）中非常高效。
