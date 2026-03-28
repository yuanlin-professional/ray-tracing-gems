# MultiscaleMeanEstimator.hlsl 技术文档

## 文件概述

**文件路径**: `Ch_25_Hybrid_Rendering_for_Real-Time_Ray_Tracing/MultiscaleMeanEstimator.hlsl`

本文件实现了**多尺度均值估计器 (Multiscale Mean Estimator)**，这是一种自适应时域滤波算法 (Adaptive Temporal Filtering Algorithm)，用于对光线追踪产生的噪声结果进行时域降噪 (Temporal Denoising)。该算法维护多个时间尺度的统计量（短期均值、长期均值、方差等），能够自动检测场景变化并调整混合速率，同时抑制萤火虫噪点 (Firefly Noise)。

## 算法与数学背景

### 指数移动平均 (Exponential Moving Average, EMA)

时域滤波的基础是 EMA：

$$\bar{x}_t = (1 - \alpha) \bar{x}_{t-1} + \alpha \cdot x_t = \text{lerp}(\bar{x}_{t-1}, x_t, \alpha)$$

其中 $\alpha$ 为混合因子 (Blend Factor)。$\alpha$ 越小，滤波窗口越长，平滑效果越好但对变化响应越慢。

### 在线方差估计 (Welford's Method)

使用改进的 Welford 方法估计方差：

$$\delta_1 = x_t - \bar{x}_{t-1}$$
$$\bar{x}_t = \text{lerp}(\bar{x}_{t-1}, x_t, \alpha)$$
$$\delta_2 = x_t - \bar{x}_t$$
$$\sigma^2_t = \text{lerp}(\sigma^2_{t-1}, \delta_1 \cdot \delta_2, \alpha_v)$$

### 多尺度设计

算法使用两个时间尺度：
- **短期均值 (Short-Term Mean)**：$\alpha = 0.08$，快速响应变化
- **长期均值 (Long-Term Mean)**：自适应 $\alpha$（通过 `catchUpBlend` 和 `vbbr`），提供稳定的低噪声估计

当两者差异大时，表示场景发生变化，长期均值加快追赶速度。

## 代码结构概览

```
MultiscaleMeanEstimator.hlsl
├── struct MultiscaleMeanEstimatorData   // 持久化状态数据
└── MultiscaleMeanEstimator()            // 核心估计函数
```

## 逐段代码详解

### 状态数据结构

```hlsl
struct MultiscaleMeanEstimatorData
{
  float3 mean;           // 长期均值（最终输出）
  float3 shortMean;      // 短期均值（快速响应）
  float vbbr;            // 基于方差的混合速率缩减因子
  float3 variance;       // 方差估计
  float inconsistency;   // 不一致性度量（场景变化指示器）
};
```

每个像素维护此结构体的持久化状态，跨帧传递。

### 函数签名与局部变量解包

```hlsl
float3 MultiscaleMeanEstimator(float3 y,
  inout MultiscaleMeanEstimatorData data,
  float shortWindowBlend = 0.08f)
{
  float3 mean = data.mean;
  float3 shortMean = data.shortMean;
  float vbbr = data.vbbr;
  float3 variance = data.variance;
  float inconsistency = data.inconsistency;
```

- `y`：当前帧的新采样值
- `shortWindowBlend = 0.08`：短期均值的 EMA 混合因子（约 12 帧的有效窗口）

### 萤火虫抑制

```hlsl
  // Suppress fireflies.
  {
    float3 dev = sqrt(max(1e-5, variance));
    float3 highThreshold = 0.1 + shortMean + dev * 8;
    float3 overflow = max(0, y - highThreshold);
    y -= overflow;
  }
```

**萤火虫 (Firefly)** 是光线追踪中偶然产生的极高亮度采样点。抑制策略：
1. 计算标准差 $\sigma = \sqrt{\max(10^{-5}, \sigma^2)}$
2. 设置高阈值：$\text{threshold} = 0.1 + \bar{x}_{short} + 8\sigma$
3. 将超过阈值的部分截断：$y \leftarrow y - \max(0, y - \text{threshold})$

阈值设置在短期均值上方 8 个标准差处（加上 0.1 的基础余量），这是一个极为宽松的裁剪——只有极端异常值才会被截断。

### 短期均值与方差更新

```hlsl
  float3 delta = y - shortMean;
  shortMean = lerp(shortMean, y, shortWindowBlend);
  float3 delta2 = y - shortMean;

  // This should be a longer window than shortWindowBlend to avoid bias.
  float varianceBlend = shortWindowBlend * 0.5;
  variance = lerp(variance, delta * delta2, varianceBlend);
  float3 dev = sqrt(max(1e-5, variance));
```

使用 Welford 方法的 EMA 变体更新方差：
- $\delta_1 = y - \bar{x}_{short,old}$（更新前的偏差）
- $\bar{x}_{short} = \text{lerp}(\bar{x}_{short}, y, 0.08)$
- $\delta_2 = y - \bar{x}_{short,new}$（更新后的偏差）
- $\sigma^2 = \text{lerp}(\sigma^2, \delta_1 \cdot \delta_2, 0.04)$

方差窗口长度（$0.04$）是短期均值窗口（$0.08$）的一半，注释解释这是为了避免方差因短期均值变化而产生偏差。

### 不一致性检测

```hlsl
  float3 shortDiff = mean - shortMean;

  float relativeDiff = dot( float3(0.299, 0.587, 0.114),
        abs(shortDiff) / max(1e-5, dev) );
  inconsistency = lerp(inconsistency, relativeDiff, 0.08);
```

衡量长期均值与短期均值的差异（以标准差为单位）：

$$\text{relativeDiff} = \text{luma}\left(\frac{|\bar{x}_{long} - \bar{x}_{short}|}{\max(10^{-5}, \sigma)}\right)$$

- 使用亮度权重 $(0.299, 0.587, 0.114)$（ITU-R BT.601 标准）
- `inconsistency` 通过 EMA 平滑，避免单帧抖动
- 当场景稳定时，$\text{relativeDiff} \to 0$；场景变化时，值增大

### 自适应混合速率计算

```hlsl
  float varianceBasedBlendReduction =
        clamp( dot( float3(0.299, 0.587, 0.114),
        0.5 * shortMean / max(1e-5, dev) ), 1.0/32, 1 );
```

**基于方差的混合速率缩减 (Variance-Based Blend Reduction, VBBR)**：

$$\text{vbbr}_{\text{target}} = \text{clamp}\left(\text{luma}\left(\frac{0.5 \cdot \bar{x}_{short}}{\max(10^{-5}, \sigma)}\right), \frac{1}{32}, 1\right)$$

这是**信噪比 (SNR)** 的一种度量：
- 当 SNR 高（信号远大于噪声）时，$\text{vbbr} \to 1$，长期均值可以快速追赶
- 当 SNR 低（噪声大）时，$\text{vbbr} \to 1/32$，长期均值缓慢更新以累积更多样本

```hlsl
  float3 catchUpBlend = clamp(smoothstep(0, 1,
        relativeDiff * max(0.02, inconsistency - 0.2)), 1.0/256, 1);
  catchUpBlend *= vbbr;
```

**追赶混合因子 (Catch-Up Blend)**：

$$\text{catchUp} = \text{clamp}\left(\text{smoothstep}\left(0, 1, \text{relativeDiff} \cdot \max(0.02, \text{inconsistency} - 0.2)\right), \frac{1}{256}, 1\right) \cdot \text{vbbr}$$

设计逻辑：
- `inconsistency - 0.2`：不一致性低于 0.2 的阈值时，认为处于稳定状态
- `max(0.02, ...)`：确保最低追赶速度
- `smoothstep`：平滑过渡，避免突变
- 最终乘以 `vbbr`，在低 SNR 条件下降低追赶速度

### 长期均值更新与状态输出

```hlsl
  vbbr = lerp(vbbr, varianceBasedBlendReduction, 0.1);
  mean = lerp(mean, y, saturate(catchUpBlend));

  // Output
  data.mean = mean;
  data.shortMean = shortMean;
  data.vbbr = vbbr;
  data.variance = variance;
  data.inconsistency = inconsistency;

  return mean;
}
```

- `vbbr` 自身也通过 EMA 平滑更新（$\alpha = 0.1$）
- 长期均值使用自适应的 `catchUpBlend` 更新
- 返回长期均值作为最终降噪结果

## 关键算法深入分析

### 双尺度架构

```
输入 y ──┬── 萤火虫抑制 ──┬── 短期均值更新 (α=0.08)
         │                │
         │                ├── 方差更新 (α=0.04)
         │                │
         │                ├── 不一致性检测
         │                │
         │                └── 自适应混合速率计算
         │
         └── 长期均值更新 (自适应 α) ──── 输出
```

短期均值快速跟踪信号变化，长期均值在短期均值的指导下自适应地更新，实现了**低噪声与快速响应**的平衡。

### 与传统 TAA 的比较

| 特性 | 传统 TAA | 多尺度均值估计器 |
|------|---------|----------------|
| 混合因子 | 固定（如 0.05） | 自适应 |
| 方差跟踪 | 无 | 有 |
| 萤火虫处理 | 颜色空间裁剪 | 基于统计的阈值 |
| 场景变化检测 | 运动向量 | 双均值不一致性 |

## 输入与输出

### 输入
| 名称 | 类型 | 说明 |
|------|------|------|
| `y` | `float3` | 当前帧的新采样值 (RGB) |
| `data` | `MultiscaleMeanEstimatorData` | 上一帧的持久化状态 |
| `shortWindowBlend` | `float` | 短期均值混合因子，默认 0.08 |

### 输出
| 名称 | 类型 | 说明 |
|------|------|------|
| 返回值 | `float3` | 降噪后的颜色值（长期均值） |
| `data` | `MultiscaleMeanEstimatorData` | 更新后的持久化状态 |

## 与其他文件的关系

- **`AORayGeneration.hlsl`**：产生需要降噪的 AO 值
- **`RayGenerationAndMissShaders.pseudocode.hlsl`**：产生需要降噪的阴影值
- **`DistanceMetric.cpp`**：使用本算法产生的均值和方差计算距离度量权重

## 在光线追踪管线中的位置

```
光线追踪原始输出 → [多尺度均值估计器] → 降噪结果 → 最终合成
                   ^^^^^^^^^^^^^^^^^^
                   本文件实现此阶段
```

该算法运行在每一帧的**后处理降噪阶段**，对每个像素独立执行。

## 技术要点与注意事项

1. **状态持久化**：`MultiscaleMeanEstimatorData` 必须跨帧存储，通常使用全屏缓冲区
2. **初始化**：首帧应将 `data` 清零，算法会自动收敛
3. **运动补偿**：该算法不包含运动补偿 (Motion Compensation)，实际使用中需要配合运动向量重投影 (Reprojection)
4. **每通道独立**：方差和均值按 RGB 三通道独立计算，但不一致性和 VBBR 使用亮度加权合并
5. **数值稳定性**：多处使用 `max(1e-5, ...)` 防止除零和负方差
6. **参数调优**：`shortWindowBlend = 0.08` 对应约 12.5 帧的有效窗口，可根据帧率和噪声水平调整

## 扩展阅读

- Welford, B.P. *Note on a Method for Calculating Corrected Sums of Squares and Products*. Technometrics, 1962
- Schied, C. et al. *Spatiotemporal Variance-Guided Filtering: Real-Time Reconstruction for Path-Traced Global Illumination*. HPG 2017
- Mara, M. et al. *Toward Practical Real-Time Photon Mapping*. SIGGRAPH 2017
- Yang, L. et al. *An Improved Shading Cache for Modern GPUs*. HPG 2016
