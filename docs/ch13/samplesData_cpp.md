# samplesData.cpp 技术文档

## 文件概述

`samplesData.cpp` 包含预计算的泊松圆盘采样点 (Poisson Disk Samples) 数据，用于软阴影的面光源采样。该文件定义了一个包含 6048 个 `float2` 的静态数组，覆盖 1 到 32 SPP（Samples Per Pixel，每像素采样数）的完整采样集。

**源文件路径**: `Ch_13_Ray_Traced_Shadows_Maintaining_Real-Time_Frame_Rates/dxrShadows/samplesData.cpp`

## 算法与数学背景

### 泊松圆盘采样 (Poisson Disk Sampling)

泊松圆盘采样是一种蓝噪声 (Blue Noise) 分布的二维采样方法，其特点是：

- 任意两个采样点之间的最小距离不小于某个阈值 $r$
- 在单位圆/方形上均匀分布
- 相比均匀随机采样，能显著减少聚集和空隙

数学定义：对于采样集 $S = \{s_1, s_2, \ldots, s_n\}$：
$$\forall i \neq j: \|s_i - s_j\| \geq r$$

### 时空采样交错

每个 SPP 级别的采样数据按 **帧号** $\times$ **屏幕像素位置** 进行交错分组：

- 4 帧时域交错（帧号 0-3）
- 每帧内 $3 \times 3 = 9$ 种空间位置变体

因此每个 SPP 级别 $n$ 对应 $n \times 9 \times 4 = 36n$ 个采样点。

总采样点数：
$$\sum_{n=1}^{8} 36n + \sum_{k \in \{12,16,20,24,28,32\}} 36k = 6048$$

## 代码结构概览

```cpp
#include "samplesData.h"
#include "Falcor.h"

static const float2 poissonDisk1to32spp[6048] {
    // 1 SPP: 36 个采样点 (4帧 x 9位置 x 1样本)
    // Frame 0: 9个float2
    float2(-0.573172f, -0.532465f),
    ...
    // Frame 1: 9个float2
    ...
    // 2 SPP: 72 个采样点 (4帧 x 9位置 x 2样本)
    ...
    // 最高到 32 SPP
};

const float2* getPoissonDisk1to32spp() {
    return poissonDisk1to32spp;
}
```

## 逐段代码详解

### 数据组织

采样数据按 SPP 级别递增排列，每个级别内部按以下层次组织：

```
SPP 级别 n:
  Frame 0:
    像素位置 0: n 个 float2 采样点
    像素位置 1: n 个 float2 采样点
    ...
    像素位置 8: n 个 float2 采样点
  Frame 1:
    ...
  Frame 2:
    ...
  Frame 3:
    ...
```

对应 `CommonUtils.hlsl` 中的索引表 `lightSamplesIndices[33]`：
```
lightSamplesIndices[1] = 0       // 1 SPP 起始
lightSamplesIndices[2] = 36      // 2 SPP 起始 = 36*1
lightSamplesIndices[3] = 108     // 3 SPP 起始 = 36*1 + 36*2
...
```

### 采样点值域

所有采样点的 $(x, y)$ 坐标在 $[-1, 1]^2$ 范围内，表示单位圆盘上的偏移。在 GPU 端会被缩放到光源的世界空间半径。

## 关键算法深入分析

### 时空采样交错消除固定噪声

通过在时间和空间维度上交错使用不同的采样子集：
1. **时间交错**：连续 4 帧使用不同采样子集，时域滤波后等效于更高的采样率
2. **空间交错**：$3 \times 3$ 邻域内每个像素使用不同的采样子集，空间滤波后进一步提升有效采样率

实际效果：4 SPP 配合时空交错 + 滤波，可以达到接近 $4 \times 4 \times 9 = 144$ 个独立采样点的视觉质量。

## 输入与输出

- **输入**: 无（编译时常量数据）
- **输出**: `getPoissonDisk1to32spp()` 返回指向静态数组的指针，用于创建 1D 纹理

## 与其他文件的关系

- 在 `dxrShadows.cpp` 的 `onLoad()` 中被调用，创建 1D 纹理：
  ```cpp
  mpPoisson1to32sppSamples = Texture::create1D(6048, ResourceFormat::RG32Float, 1, 1, getPoissonDisk1to32spp(), ...);
  ```
- 在 `DXRShadows.rt.hlsl` 中通过 `samples.Load(int2(index, 0))` 读取

## 在光线追踪管线中的位置

采样数据位于管线的输入端，为阴影光线的方向偏移提供高质量分布，直接影响软阴影的视觉质量和收敛速度。

## 技术要点与注意事项

1. 数组大小 6048 是精确计算的，对应全部 SPP 级别的采样需求
2. 该数据以 `RG32Float` 格式上传为 1D 纹理，每个 texel 存储一个 `float2` 采样点
3. 非 4 的倍数的 SPP 级别（如 9-11）在 GPU 端会被跳过或四舍五入到最近的合法值
4. 数据为离线预计算（使用 Robert Bridson 的泊松圆盘算法或类似方法）

## 扩展阅读

- Robert Bridson, "Fast Poisson Disk Sampling in Arbitrary Dimensions", SIGGRAPH 2007
- "A Survey of Blue-Noise Sampling and its Applications"
- Ray Tracing Gems, Chapter 13, Section 13.4 (Sampling Strategies)
