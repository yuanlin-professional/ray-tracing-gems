# DistanceMetric.cpp 技术文档

## 文件概述

**文件路径**: `Ch_25_Hybrid_Rendering_for_Real-Time_Ray_Tracing/DistanceMetric.cpp`

本文件实现了时域降噪 (Temporal Denoising) 中的**距离度量权重 (Distance Metric Weight)** 计算。这是一个极短的代码片段（仅 2 行），用于衡量当前像素颜色值与历史均值的偏差程度，并将偏差转换为指数衰减的权重值。该权重用于控制时域滤波器对当前样本的接受程度——偏差越大（可能是"萤火虫"噪点或场景变化），权重越低。

## 算法与数学背景

### 标准化距离

将当前像素 RGB 颜色值与历史均值的差异，按历史标准差进行标准化：

$$\mathbf{d} = \frac{\mathbf{c}_{rgb} - \boldsymbol{\mu}_{rgb}}{\boldsymbol{\sigma}_{rgb}}$$

其中：
- $\mathbf{c}_{rgb}$ 为当前像素的 RGB 颜色值
- $\boldsymbol{\mu}_{rgb}$ 为历史 RGB 均值
- $\boldsymbol{\sigma}_{rgb}$ 为历史 RGB 标准差

### 指数权重

将标准化距离的亮度 (Luminance) 转换为指数衰减权重：

$$w = 2^{-10 \cdot \text{luma}(\mathbf{d})}$$

其中 $\text{luma}(\mathbf{d})$ 是对标准化距离向量的亮度感知加权（通常为 $0.299 d_r + 0.587 d_g + 0.114 d_b$）。

当 $\text{luma}(\mathbf{d}) = 0$ 时，$w = 1$（完全接受）；当距离增大时，权重指数级下降。

## 代码结构概览

本文件仅包含 2 行代码，以下提供完整源码及逐行注释。

## 逐段代码详解

```cpp
// 第1行：计算标准化距离向量
dist = (rgb - rgb_mean) / rgb_deviation;
```
- `rgb`：当前像素的 RGB 颜色值（`float3` 类型）
- `rgb_mean`：该像素位置的历史 RGB 均值（来自时域累积）
- `rgb_deviation`：该像素位置的历史 RGB 标准差
- `dist`：标准化距离向量，表示当前颜色偏离历史均值的标准差倍数

这是经典的 **z-score** 标准化。例如，`dist = (2.0, 0.5, 0.1)` 表示红色通道偏离 2 个标准差，绿色偏离 0.5 个标准差，蓝色偏离 0.1 个标准差。

```cpp
// 第2行：将距离转换为指数衰减权重
w = exp2(-10 * luma(dist));
```
- `luma(dist)`：计算标准化距离向量的亮度感知加权值
- `exp2(x)` = $2^x$：以 2 为底的指数函数
- `-10`：衰减速率系数，控制权重对偏差的敏感度

权重行为特性：
| `luma(dist)` | $w = 2^{-10 \cdot \text{luma}}$ | 含义 |
|---------------|------|------|
| 0 | 1.0 | 完全匹配，完全接受 |
| 0.1 | 0.5 | 轻微偏差 |
| 0.3 | 0.125 | 中等偏差 |
| 1.0 | ~0.001 | 大偏差，几乎拒绝 |

## 关键算法深入分析

### 时域降噪中的作用

在时域降噪管线中，该距离度量用于：

1. **萤火虫抑制 (Firefly Suppression)**：异常亮的采样点产生大的标准化距离，权重趋近于 0
2. **场景变化检测**：当场景发生明显变化时（如物体移动），新颜色值与历史均值差异大，权重下降，防止"拖影"(Ghosting)
3. **自适应混合**：在颜色稳定区域，权重高，历史信息充分利用；在变化剧烈区域，权重低，更多依赖新数据

### 系数 -10 的设计考量

系数 $-10$ 是经验值，选择原则为：
- 使得 1 个标准差内的偏差保持合理权重（~0.001 在 1$\sigma$ 处可能过于激进，实际应用中 `luma(dist)` 可能使用绝对值）
- 超过 1 个标准差的偏差被大幅抑制
- 使用 `exp2` 而非 `exp` 是 GPU 上的高效实现（GPU 硬件原生支持 `exp2`）

## 输入与输出

### 输入
| 名称 | 类型 | 说明 |
|------|------|------|
| `rgb` | `float3` | 当前像素 RGB 颜色值 |
| `rgb_mean` | `float3` | 历史 RGB 均值 |
| `rgb_deviation` | `float3` | 历史 RGB 标准差 |

### 输出
| 名称 | 类型 | 说明 |
|------|------|------|
| `dist` | `float3` | 标准化距离向量 (z-score) |
| `w` | `float` | 权重值，$\in [0, 1]$，用于时域混合 |

## 与其他文件的关系

- **`MultiscaleMeanEstimator.hlsl`**：提供 `rgb_mean` 和 `variance`（标准差的平方），是距离度量的输入来源
- **`AORayGeneration.hlsl`** / **`RayGenerationAndMissShaders.pseudocode.hlsl`**：产生需要降噪的原始光线追踪结果

## 在光线追踪管线中的位置

```
光线追踪输出 (带噪声) → 时域降噪 → [距离度量权重计算] → 加权混合 → 降噪输出
                                     ^^^^^^^^^^^^^^^^^^^
                                     本文件实现此步骤
```

距离度量权重在时域降噪的**混合阶段 (Blending Stage)** 中使用，决定当前帧与历史帧的混合比例。

## 技术要点与注意事项

1. **除零保护**：实际实现中 `rgb_deviation` 需要添加小的 epsilon（如 `max(1e-5, rgb_deviation)`），防止零标准差导致除零错误
2. **颜色空间选择**：在 RGB 空间计算距离可能对色调变化不够敏感，某些实现会使用 YCoCg 颜色空间
3. **GPU 优化**：`exp2()` 是 GPU 上的单周期指令，比 `exp()` 更高效
4. **亮度加权函数**：`luma()` 通常使用 ITU-R BT.601 标准权重 $(0.299, 0.587, 0.114)$，与 `MultiscaleMeanEstimator.hlsl` 中使用的权重一致

## 扩展阅读

- Salvi, M. *An Excursion in Temporal Supersampling*. GDC 2016
- Karis, B. *High Quality Temporal Supersampling*. SIGGRAPH 2014 Course Notes
- Schied, C. et al. *Spatiotemporal Variance-Guided Filtering*. HPG 2017
