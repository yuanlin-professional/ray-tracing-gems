# HaltonSampling.hlsl 技术文档

## 文件概述

**文件路径**: `Ch_25_Hybrid_Rendering_for_Real-Time_Ray_Tracing/HaltonSampling.hlsl`

本文件实现了基于 **Halton 序列 (Halton Sequence)** 的低差异采样系统，用于为光线追踪中的随机方向生成高质量的准随机数 (Quasi-Random Numbers)。Halton 序列是一种低差异序列 (Low-Discrepancy Sequence)，相比伪随机数具有更均匀的空间分布特性，能在相同采样数下提供更低的蒙特卡洛积分方差。

本实现修改自 PBRT (Physically Based Rendering Toolkit) 的 Halton 采样器，针对 GPU 实时渲染进行了适配。

## 算法与数学背景

### Halton 序列的定义

Halton 序列是一种多维低差异序列，每个维度使用不同的质数基底。给定基底 $b$ 和索引 $i$，对应的 **基数反转函数 (Radical Inverse Function)** $\phi_b(i)$ 定义为：

如果 $i$ 的 $b$ 进制表示为 $i = \sum_{k=0}^{m} d_k \cdot b^k$，则：

$$\phi_b(i) = \sum_{k=0}^{m} d_k \cdot b^{-(k+1)}$$

例如，对于 $b = 2$，$i = 5 = (101)_2$：

$$\phi_2(5) = 1 \cdot 2^{-1} + 0 \cdot 2^{-2} + 1 \cdot 2^{-3} = 0.625$$

### 多维 Halton 序列

$d$ 维 Halton 序列的第 $i$ 个点为：

$$\mathbf{x}_i = (\phi_{p_1}(i), \phi_{p_2}(i), \ldots, \phi_{p_d}(i))$$

其中 $p_1, p_2, \ldots, p_d$ 是前 $d$ 个质数（2, 3, 5, 7, 11, ...）。

### Halton 索引的像素映射

为了在屏幕空间上获得良好的分层采样效果，使用逆基数反转函数将像素坐标映射到 Halton 序列索引：

$$\text{index} = (\phi_2^{-1}(x \bmod 256) \cdot 76545 + \phi_3^{-1}(y \bmod 256) \cdot 110080) \bmod m + i \cdot 186624$$

## 代码结构概览

```
HaltonSampling.hlsl
├── struct HaltonState          // Halton 序列状态
├── haltonInit()                // 初始化 Halton 状态
├── haltonSample()              // 单次基数反转采样
├── haltonNext()                // 获取下一维度的采样值
├── haltonIndex()               // 计算像素对应的 Halton 索引
├── halton2Inverse()            // 基底 2 的逆基数反转
└── halton3Inverse()            // 基底 3 的逆基数反转
```

## 逐段代码详解

### HaltonState 结构体

```hlsl
struct HaltonState
{
  uint dimension;       // 当前采样维度（从 2 开始，0 和 1 用于像素坐标）
  uint sequenceIndex;   // 序列中的全局索引
};
```

状态结构体跟踪当前使用的维度和序列索引。`dimension` 从 2 开始，因为维度 0 和 1 已用于像素位置的分层。

### haltonInit() - 初始化

```hlsl
void haltonInit(inout HaltonState hState,
                int x, int y,           // 像素坐标
                int path, int numPaths, // 路径索引和总路径数
                int frameId,            // 帧编号
                int loop)               // 序列循环周期
{
  hState.dimension = 2;  // 从维度 2 开始
  hState.sequenceIndex = haltonIndex(x, y,
        (frameId * numPaths + path) % (loop * numPaths));
}
```

初始化逻辑：
- `dimension = 2`：维度 0, 1 已用于像素位置
- 使用 `haltonIndex()` 将像素坐标和时间信息映射到全局序列索引
- `(frameId * numPaths + path) % (loop * numPaths)`：确保在 `loop` 帧后序列循环，避免索引溢出

### haltonSample() - 基数反转采样

```hlsl
float haltonSample(uint dimension, uint index)
{
  int base = 0;

  // 根据维度选择质数基底
  switch (dimension)
  {
  case 0:  base = 2;   break;
  case 1:  base = 3;   break;
  case 2:  base = 5;   break;
  [...] // 维度 0-31，使用前 32 个质数
  case 31: base = 131; break;
  default: base = 2;   break;
  }

  // 计算基数反转值（Radical Inverse）
  float a = 0;
  float invBase = 1.0f / float(base);

  for (float mult = invBase;
       sampleIndex != 0; sampleIndex /= base, mult *= invBase)
  {
    a += float(sampleIndex % base) * mult;
  }

  return a;
}
```

实现基数反转函数 $\phi_b(i)$：
1. 根据维度选择对应的质数基底（维度 0 → 2, 维度 1 → 3, 维度 2 → 5, ...）
2. 循环提取索引在该基底下的每一位数字
3. 将数字乘以递减的权重（$b^{-1}, b^{-2}, \ldots$）并累加

### haltonNext() - 获取下一维度采样值

```hlsl
float haltonNext(inout HaltonState state)
{
  return haltonSample(state.dimension++, state.sequenceIndex);
}
```

从当前维度获取采样值，然后自增维度。每次调用返回 Halton 序列在下一个维度的值，实现多维准随机数生成。

### haltonIndex() - 像素到序列索引的映射

```hlsl
// Modified from [pbrt]
uint haltonIndex(uint x, uint y, uint i)
{
  return ((halton2Inverse(x % 256, 8) * 76545 +
      halton3Inverse(y % 256, 6) * 110080) % m_increment) + i * 186624;
}
```

此函数将 2D 像素坐标映射到 1D Halton 序列索引，确保：
- 相邻像素使用序列中间隔较远的索引，避免相关性
- `x % 256` 和 `y % 256`：将像素坐标限制在 256x256 的分层块内
- `halton2Inverse` 和 `halton3Inverse`：分别计算基底 2 和基底 3 的逆基数反转
- 魔术常数 `76545`, `110080`, `186624`, `m_increment` 源自 PBRT 的 Halton 采样器设计

### halton2Inverse() - 基底 2 的逆基数反转（位反转）

```hlsl
// Modified from [pbrt]
uint halton2Inverse(uint index, uint digits)
{
  index = (index << 16) | (index >> 16);
  index = ((index & 0x00ff00ff) << 8) | ((index & 0xff00ff00) >> 8);
  index = ((index & 0x0f0f0f0f) << 4) | ((index & 0xf0f0f0f0) >> 4);
  index = ((index & 0x33333333) << 2) | ((index & 0xcccccccc) >> 2);
  index = ((index & 0x55555555) << 1) | ((index & 0xaaaaaaaa) >> 1);
  return index >> (32 - digits);
}
```

基底 2 的逆基数反转等价于**位反转 (Bit Reversal)**，使用经典的分治位交换算法：
1. 交换高 16 位和低 16 位
2. 在各 16 位半段中交换高低 8 位
3. 在各 8 位字节中交换高低 4 位
4. 在各 4 位中交换高低 2 位
5. 在各 2 位中交换高低 1 位
6. 右移保留 `digits` 位

这是 $O(1)$ 的位反转实现，在 GPU 上非常高效。

### halton3Inverse() - 基底 3 的逆基数反转

```hlsl
// Modified from [pbrt]
uint halton3Inverse(uint index, uint digits)
{
  uint result = 0;
  for (uint d = 0; d < digits; ++d)
  {
    result = result * 3 + index % 3;
    index /= 3;
  }
  return result;
}
```

基底 3 的逆基数反转使用迭代方法：
- 逐位提取三进制表示的数字
- 将数字按反转顺序重新组合

## 关键算法深入分析

### 低差异序列 vs 伪随机数

| 特性 | 伪随机数 | Halton 序列 |
|------|----------|-------------|
| 分布均匀性 | 统计上均匀 | 确定性均匀 |
| 聚集现象 | 可能出现 | 极少 |
| 收敛速度 | $O(N^{-1/2})$ | $O(N^{-1} \log^d N)$ |
| 相关性 | 低（但存在） | 确定性模式 |

对于蒙特卡洛积分，Halton 序列的收敛速度理论上优于纯随机采样。

### Cranley-Patterson 旋转

在 `AORayGeneration.hlsl` 中使用的 `frac(haltonNext(hState) + randomNext(frameSeed))` 实现了 Cranley-Patterson 旋转。这种技术：
- 保持了低差异序列的均匀分布特性
- 消除了确定性采样模式导致的视觉伪影
- 使不同像素和不同帧使用不同的采样偏移

## 输入与输出

### haltonInit 输入
| 名称 | 类型 | 说明 |
|------|------|------|
| `x, y` | `int` | 像素坐标 |
| `path` | `int` | 当前路径索引 |
| `numPaths` | `int` | 每像素总路径数 |
| `frameId` | `int` | 当前帧编号 |
| `loop` | `int` | 序列循环周期 |

### haltonNext 输出
| 名称 | 类型 | 说明 |
|------|------|------|
| 返回值 | `float` | $[0, 1)$ 范围的准随机采样值 |

## 与其他文件的关系

- **`AORayGeneration.hlsl`**：调用 `haltonNext()` 生成 AO 射线方向的随机数
- **`RayGenerationAndMissShaders.pseudocode.hlsl`**：调用 `haltonInit()` 和 `haltonNext()` 初始化采样序列

## 在光线追踪管线中的位置

```
初始化 HaltonState → [haltonInit] →
  多次调用 [haltonNext] → 生成采样方向 →
    配合 Cranley-Patterson 旋转 → 射线方向
```

Halton 采样系统是整个混合渲染管线的**采样基础设施**，为所有需要随机方向的光线追踪操作（阴影、AO、反射、折射）提供高质量的采样序列。

## 技术要点与注意事项

1. **维度限制**：支持最多 32 维（case 0-31），超出后默认使用基底 2，这可能导致与维度 0 的相关性
2. **高维退化**：Halton 序列在高维度时分布质量下降，维度超过 ~20 后建议使用其他序列（如 Sobol）
3. **数值精度**：`haltonSample()` 中的循环使用 `float` 精度，对于很大的索引值可能损失精度
4. **像素分块**：`x % 256` 和 `y % 256` 意味着分层效果在 256x256 的块内有效，超出后模式会重复
5. **GPU 性能**：`switch` 语句在 GPU 上可能导致线程发散，但由于所有像素在同一维度使用相同基底，实际影响较小

## 扩展阅读

- Halton, J.H. *On the efficiency of certain quasi-random sequences of points in evaluating multi-dimensional integrals*. Numerische Mathematik 2, 1960
- Pharr, M., Jakob, W., Humphreys, G. *PBRT*, Chapter 7 - Sampling and Reconstruction
- Cranley, R., Patterson, T.N.L. *Randomization of Number Theoretic Methods for Multiple Integration*. SIAM J. Numer. Anal., 1976
- Keller, A. *Quasi-Monte Carlo Methods for Photorealistic Image Synthesis*. Ph.D. Thesis, 1998
