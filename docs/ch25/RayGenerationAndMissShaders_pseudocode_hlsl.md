# RayGenerationAndMissShaders.pseudocode.hlsl 技术文档

## 文件概述

**文件路径**: `Ch_25_Hybrid_Rendering_for_Real-Time_Ray_Tracing/RayGenerationAndMissShaders.pseudocode.hlsl`

本文件是**阴影射线生成着色器 (Shadow Ray Generation Shader)** 和**未命中着色器 (Miss Shader)** 的 HLSL 伪代码实现。它展示了混合渲染 (Hybrid Rendering) 管线中如何从 G-Buffer 的深度信息重建世界空间位置，使用 Halton 低差异序列生成采样方向，并通过 DXR 的 `TraceRay()` 发射阴影射线以确定阳光软阴影。

> 注意：本文件标注为"pseudocode"，无法直接编译，但完整展示了阴影射线追踪的逻辑流程。

## 算法与数学背景

### 深度缓冲区位置重建

从深度缓冲区重建世界空间位置的标准流程：

1. 将像素坐标转换为 UV 坐标：$\mathbf{uv} = (\mathbf{pixel} + 0.5) \cdot \mathbf{viewDimensions}^{-1}$
2. UV 转换为裁剪空间 (Clip Space)：$\mathbf{cs} = (\text{uvToCs}(\mathbf{uv}), \text{depth}, 1)$
3. 通过逆变换矩阵转换到世界空间：$\mathbf{ws} = \mathbf{M}_{clipToWorld} \cdot \mathbf{cs}$
4. 透视除法 (Perspective Division)：$\mathbf{position} = \mathbf{ws}.xyz / \mathbf{ws}.w$

### 软阴影与锥形采样

为产生**软阴影 (Soft Shadows)**，不是沿单一光线方向发射射线，而是在以光线方向为轴的锥体 (Cone) 内随机选择方向：

$$\boldsymbol{\omega} = \text{uniformSampleCone}(\xi_1, \xi_2, \cos\theta_{max})$$

其中 $\cos\theta_{max}$ 控制锥体的半角，值越小锥体越宽，阴影越柔和但噪声越大。

### TMin 的自适应计算

$$T_{min} = \max(1.0, \|\mathbf{position}\|) \times 10^{-3}$$

射线起始偏移量与离原点的距离成正比，以处理远距离处浮点精度降低的问题。

## 代码结构概览

```
RayGenerationAndMissShaders.pseudocode.hlsl
├── shadowRaygen()    // 阴影射线生成着色器 [shader("raygeneration")]
└── shadowMiss()      // 阴影射线未命中着色器 [shader("miss")]
```

## 逐段代码详解

### 着色器入口与像素坐标计算

```hlsl
[shader("raygeneration")]
void shadowRaygen()
{
  uint2 launchIndex = DispatchRaysIndex();
  uint2 launchDim = DispatchRaysDimensions();
  uint2 pixelPos = launchIndex +
      uint2(g_pass.launchOffsetX, g_pass.launchOffsetY);
```

- `DispatchRaysIndex()`：当前线程在 DXR 调度网格中的二维索引
- `DispatchRaysDimensions()`：调度网格的尺寸
- `g_pass.launchOffset`：偏移量，支持分块渲染 (Tiled Rendering) 或视口偏移

### 天空像素跳过

```hlsl
  const float depth = g_depth[pixelPos];

  // Skip sky pixels.
  if (depth == 0.0)
  {
    g_output[pixelPos] = float4(0, 0, 0, 0);
    return;
  }
```

深度为 0.0 表示天空（无穷远处），无需计算阴影。在反向 Z 缓冲区 (Reverse-Z) 中，近平面深度为 1.0，远平面为 0.0。

### 世界空间位置重建

```hlsl
  // Compute position from depth buffer.
  float2 uvPos = (pixelPos + 0.5) * g_raytracing.viewDimensions.zw;
  float4 csPos = float4(uvToCs(uvPos), depth, 1);
  float4 wsPos = mul(g_raytracing.clipToWorld, csPos);
  float3 position = wsPos.xyz / wsPos.w;
```

逐步重建过程：
1. **像素 → UV**：加 0.5 取像素中心，乘以 `viewDimensions.zw`（即 `1/width, 1/height`）
2. **UV → 裁剪空间**：`uvToCs()` 将 $[0,1]$ 范围转换为 $[-1,1]$ 范围
3. **裁剪空间 → 世界空间**：使用预计算的 `clipToWorld` 逆矩阵
4. **透视除法**：除以 $w$ 分量

### Halton 序列初始化

```hlsl
  // Initialize the Halton sequence.
  HaltonState hState =
      haltonInit(hState, pixelPos, g_raytracing.frameIndex);
```

初始化 Halton 序列状态，使用像素坐标和帧索引确保每个像素和每帧使用不同的采样序列。

### 随机方向生成

```hlsl
  // Generate random numbers to rotate the Halton sequence.
  uint frameseed =
      randomInit(pixelPos, launchDim.x, g_raytracing.frameIndex);
  float rnd1 = frac(haltonNext(hState) + randomNext(frameseed));
  float rnd2 = frac(haltonNext(hState) + randomNext(frameseed));
```

**Cranley-Patterson 旋转**：将 Halton 序列值与伪随机偏移相加后取小数部分，兼顾低差异性和随机性。

```hlsl
  // Generate a random direction based on the cone angle.
  float3 rndDirection = uniformSampleCone(rnd1, rnd2, cosThetaMax);
```

在以光源方向为轴的锥体内均匀采样一个方向。`cosThetaMax` 决定锥体角度：
- $\cos\theta_{max} = 1.0$：点光源，硬阴影
- $\cos\theta_{max} < 1.0$：面光源模拟，软阴影

### 阴影射线配置

```hlsl
  // Prepare a shadow ray.
  RayDesc ray;
  ray.Origin = position;
  ray.Direction = g_sunLight.L;
  ray.TMin = max(1.0f, length(position)) * 1e-3f;
  ray.TMax = tmax;
  ray.Direction = mul(rndDirection, createBasis(L));
```

注意 `Direction` 被赋值两次——最终值是将锥体内的随机方向通过 `createBasis(L)` 变换到以光源方向 $L$ 为轴的坐标系中。

`TMin` 的自适应设置是关键的**自相交防护 (Self-Intersection Avoidance)** 策略：离原点越远的物体，浮点精度越低，需要更大的偏移量。

### 射线发射

```hlsl
  // Initialize the payload.
  ShadowData shadowPayload;
  shadowPayload.miss = false;

  // Launch a ray.
  TraceRay(rtScene,
      RAY_FLAG_SKIP_CLOSEST_HIT_SHADER |
      RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH,
      RaytracingInstanceMaskAll, HitType_Shadow, SbtRecordStride,
      MissType_Shadow, ray, shadowPayload);
```

DXR 射线追踪标志的性能优化意义：
- `RAY_FLAG_SKIP_CLOSEST_HIT_SHADER`：无需最近命中着色器，节省着色开销
- `RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH`：找到任意命中即停止 BVH 遍历

这两个标志组合是阴影射线的标准优化模式，显著减少 BVH 遍历开销。

### 结果输出

```hlsl
  // If we have missed, the shadow value is white.
  g_output[pixelPos] = shadowPayload.miss ? 1.0f : 0.0f;
}
```

输出二值阴影：
- `1.0`（白色）：射线未命中任何物体，该像素在光照下
- `0.0`（黑色）：射线被遮挡，该像素在阴影中

### 未命中着色器

```hlsl
[shader("miss")]
void shadowMiss(inout ShadowData payload : SV_RayPayload)
{
  payload.miss = true;
}
```

当射线未与任何几何体相交时，DXR 管线调用此着色器。仅需设置 `miss = true` 标志。

## 关键算法深入分析

### DXR 阴影射线的工作流程

```
shadowRaygen()
    │
    ├─ 重建世界空间位置
    ├─ 生成随机方向（Halton + Cranley-Patterson）
    ├─ 配置射线参数
    │
    └─ TraceRay() ──┬── 命中几何体 → payload.miss = false (初始值)
                    │
                    └── 未命中 → shadowMiss() → payload.miss = true
```

### 软阴影的原理

软阴影 (Soft Shadow / Penumbra) 的产生：
1. 每帧每像素发射 1 条射线，方向在锥体内随机
2. 在阴影边缘 (Penumbra) 区域，部分射线命中遮挡物，部分未命中
3. 经过多帧时域累积（使用 `MultiscaleMeanEstimator`），得到 $[0, 1]$ 之间的连续阴影值
4. 锥体角度模拟了太阳等面光源的角直径

## 输入与输出

### 输入（全局资源）
| 名称 | 类型 | 说明 |
|------|------|------|
| `g_depth` | `Texture2D<float>` | 深度缓冲区 |
| `g_pass.launchOffset` | `uint2` | 调度偏移量 |
| `g_raytracing.viewDimensions` | `float4` | 视图维度 (w, h, 1/w, 1/h) |
| `g_raytracing.clipToWorld` | `float4x4` | 裁剪空间到世界空间的变换矩阵 |
| `g_raytracing.frameIndex` | `uint` | 帧编号 |
| `g_sunLight.L` | `float3` | 太阳光方向 |
| `rtScene` | `RaytracingAccelerationStructure` | 加速结构 |

### 输出
| 名称 | 类型 | 说明 |
|------|------|------|
| `g_output[pixelPos]` | `float4` | 阴影值（0 = 阴影, 1 = 光照） |

## 与其他文件的关系

- **`HaltonSampling.hlsl`**：提供 `haltonInit()` 和 `haltonNext()` 函数
- **`AORayGeneration.hlsl`**：结构类似的 AO 射线生成着色器
- **`MultiscaleMeanEstimator.hlsl`**：对阴影结果进行时域降噪
- **`DistanceMetric.cpp`**：降噪中的距离度量

## 在光线追踪管线中的位置

```
┌──────────┐    ┌──────────────────┐    ┌──────────┐
│ G-Buffer │ →  │ Shadow Ray Gen   │ →  │ Miss     │ → 阴影输出
│ (深度)    │    │ [本文件主体]     │    │ Shader   │
└──────────┘    └──────────────────┘    │ [本文件] │
                         │              └──────────┘
                         v
                ┌──────────────────┐
                │ BVH 遍历         │
                │ (硬件加速)       │
                └──────────────────┘
```

## 技术要点与注意事项

1. **深度值判断**：`depth == 0.0` 的浮点比较在反向 Z 缓冲区中是安全的，因为天空深度精确为 0
2. **自适应 TMin**：`max(1.0, length(position)) * 1e-3` 是一种简单但有效的自相交防护策略
3. **单射线软阴影**：每帧仅发射 1 条射线，依赖时域累积获得平滑阴影，这是实时光线追踪的常见策略
4. **SBT 配置**：`HitType_Shadow` 和 `MissType_Shadow` 是 Shader Binding Table (SBT) 中的偏移量，确保使用正确的着色器
5. **伪代码说明**：文件中的 `uniformSampleCone()`, `createBasis()`, `randomInit()`, `randomNext()` 等函数在此文件中未定义，需要外部实现

## 扩展阅读

- Wyman, C. *A Gentle Introduction to DirectX Raytracing*. Ray Tracing Gems, Chapter 3
- Shirley, P. et al. *Monte Carlo Rendering with Natural Illumination*. Pacific Graphics, 2001
- Woo, A. *Fast Ray-Box Intersection*. Graphics Gems, 1990
- DXR 规范: `DispatchRaysIndex()`, `TraceRay()`, `RAY_FLAG_*`
