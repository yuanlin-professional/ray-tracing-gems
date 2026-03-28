# CommonUtils.hlsl 技术文档

## 文件概述

`CommonUtils.hlsl` 是 GPU 端的通用工具函数库，被所有 HLSL 着色器共享。它提供了深度缓冲操作、视空间位置重建、法线重建、可见性历史计算、变化度计算以及多光源数据打包/解包等核心工具函数。

**源文件路径**: `Ch_13_Ray_Traced_Shadows_Maintaining_Real-Time_Frame_Rates/dxrShadows/Data/CommonUtils.hlsl`

## 算法与数学背景

### 线性深度归一化

DirectX 深度缓冲存储的是非线性深度 $z$，需转换为线性深度 $d_{\text{lin}} \in [0, 1]$：

$$d_{\text{lin}} = \frac{1}{(1 - f/n) \cdot z + f/n}$$

其中 $f$ 为远平面距离，$n$ 为近平面距离。

### 反投影重建视空间坐标

从屏幕 UV 和线性深度重建视空间三维坐标：

$$\mathbf{p}_{\text{view}} = \text{ray}(u,v) \cdot d_{\text{lin}}$$

其中 ray 方向通过投影矩阵逆元素构造：

$$\text{ray} = \left( \begin{bmatrix} u' \\ v' \\ 1 \end{bmatrix} \cdot \mathbf{s}_1 + \mathbf{s}_2 \right) \cdot s_{2w}$$

$\mathbf{s}_1, \mathbf{s}_2$ 为预计算的反投影参数。

### 从深度缓冲重建法线

使用十字形 (Cross-Pattern) 有限差分法：

$$\mathbf{n} = \text{normalize}(\Delta x \times \Delta y)$$

其中 $\Delta x, \Delta y$ 分别选取左/右、上/下邻域中距离较近的差分向量，以减少深度不连续处的伪影。

### 时域可见性过滤

4 帧历史可见性的 Box Filter (盒式滤波)：

$$v_{\text{filtered}} = \frac{1}{4}(h_0 + h_1 + h_2 + h_3) = \mathbf{h} \cdot (0.25, 0.25, 0.25, 0.25)$$

### 变化度计算

基于 4 帧历史的最大-最小差值：

$$\text{variation} = |v_{\max} - v_{\min}|$$

## 代码结构概览

```
CommonUtils.hlsl
├── #include "shaderCommon.h"
├── getViewSpacePositionUsingDepth()    -- 反投影重建
├── getNormalizedLinearDepth()          -- 线性深度转换
├── calculateFilteredVariation()         -- 变化度计算
├── getViewSpacePosition()              -- 从深度纹理获取视空间坐标
├── calculateNormal()                   -- 从深度重建法线
├── calculateVisibilityFromHistory()     -- 时域 Box Filter
├── lightSamplesIndices[33]             -- 采样索引表
├── pack4ToTexture()                    -- 4通道打包
├── unpackFrom4Texture()                -- 4通道解包
├── encodeVariationAndSampleCount()      -- 编码变化度+采样数
└── decodeVariationAndSampleCount()      -- 解码变化度+采样数
```

## 逐段代码详解

### 反投影重建

```hlsl
inline float3 getViewSpacePositionUsingDepth(
    float2 uv, float pixelDepth,
    float4 screenUnprojection, float4 screenUnprojection2)
{
    // 将 UV 映射到 NDC [-1, 1]
    float4 screenPosition = float4(uv * 2.0f - 1.0f, 1.0f, 1.0f);
    // 使用预计算的投影矩阵逆元素构造视空间射线方向
    float3 ray = (screenPosition.xyz * screenUnprojection.xyz
                  + screenUnprojection2.xyz) * screenUnprojection2.w;
    // 乘以线性深度得到视空间位置
    return ray * pixelDepth;
}
```

### 法线重建（最优差分选择）

```hlsl
float3 calculateNormal(Texture2D depths, ...) {
    float3 center = getViewSpacePosition(...);
    float3 up    = getViewSpacePosition(uvs + float2(0, dy), ...);
    float3 right = getViewSpacePosition(uvs + float2(dx, 0), ...);
    float3 down  = getViewSpacePosition(uvs + float2(0, -dy), ...);
    float3 left  = getViewSpacePosition(uvs + float2(-dx, 0), ...);

    // 选择更短的差分向量（减少深度边缘伪影）
    float3 xSample = (length(left - center) < length(center - right))
                   ? (left - center) : (center - right);
    float3 ySample = (length(up - center) < length(center - down))
                   ? (up - center) : (center - down);

    return normalize(cross(xSample, ySample));
}
```

### 采样索引表

```hlsl
static const int lightSamplesIndices[33] = {
    0,                  // 0 SPP (unused)
    0,                  // 1 SPP: offset 0
    36,                 // 2 SPP: offset 36 (= 36*1)
    36 + 72,            // 3 SPP: offset 108 (= 36*1 + 36*2)
    36 + 72 + 108,      // 4 SPP: offset 216
    ...
    // 9-12 共享偏移（9,10,11 跳转到 12 SPP 的数据）
    // 13-16 共享偏移
    // ...直到 32 SPP
};
```

### 变化度 + 采样数编码

将浮点变化度和整数采样数打包到单个 `float4` 中：

```hlsl
float4 encodeVariationAndSampleCount(float4 newHistory, int4 samplesCounts) {
    // 变化度缩放到 [0, 0.99) 的小数部分
    // 采样数 + 1 作为整数部分（+1 以正确编码 0）
    return (newHistory * 0.99f) + float4(samplesCounts) + float4(1.0f);
}

float4 decodeVariationAndSampleCount(float4 history, out int4 samplesCounts) {
    samplesCounts = int4(trunc(history)) - int4(1);  // 还原采样数
    return frac(history) / 0.99f;                      // 还原变化度
}
```

## 输入与输出

- **输入**: 深度缓冲、投影参数、可见性历史
- **输出**: 视空间坐标、法线、滤波可见性、变化度

## 与其他文件的关系

被以下所有着色器文件 `#include`：
- `DXRShadows.rt.hlsl`
- `DebugPass.ps.hlsl`
- `DepthAndNormalsPass.ps.hlsl`
- `ForwardRenderer.ps.hlsl`
- `MotionVectors.ps.hlsl`
- `VariationFilteringPass.ps.hlsl`
- `VisibilityFilteringPass.ps.hlsl`

## 在光线追踪管线中的位置

此文件是全部 GPU 着色器的公共基础设施层，为深度重建、滤波和数据编码提供统一实现。

## 技术要点与注意事项

1. `screenUnprojection` 参数从投影矩阵逆的特定元素提取，避免在 GPU 端做完整矩阵乘法
2. 法线重建使用最小差分选择策略，在深度边缘处显著优于简单的前向/后向差分
3. 变化度编码利用 IEEE 浮点的精度，将整数和小数部分分别存储
4. `pack4ToTexture` / `unpackFrom4Texture` 使用 if-else 链而非数组索引，因为 HLSL 编译器对动态索引的展开优化不够可靠

## 扩展阅读

- "Reconstructing Position from Depth" -- GPU Gems 系列
- "Edge-Avoiding A-Trous Wavelet Transform for Fast Global Illumination Filtering"
