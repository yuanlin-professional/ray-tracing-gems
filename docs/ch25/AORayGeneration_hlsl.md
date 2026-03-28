# AORayGeneration.hlsl 技术文档

## 文件概述

**文件路径**: `Ch_25_Hybrid_Rendering_for_Real-Time_Ray_Tracing/AORayGeneration.hlsl`

本文件实现了**环境遮蔽 (Ambient Occlusion, AO)** 光线生成着色器的核心逻辑。AO 是一种全局光照近似技术，通过计算场景中每个表面点被周围几何体遮挡的程度来模拟环境光的衰减效果。该着色器从 G-Buffer 中获取表面位置和法线信息，沿法线方向的半球内发射多条随机射线，统计未被遮挡的射线比例，作为该像素的 AO 值。

## 算法与数学背景

### 环境遮蔽的数学定义

环境遮蔽可以表达为对法线半球 $\Omega$ 上可见性函数的积分：

$$AO(\mathbf{p}) = \frac{1}{\pi} \int_{\Omega} V(\mathbf{p}, \boldsymbol{\omega}) (\mathbf{n} \cdot \boldsymbol{\omega}) \, d\boldsymbol{\omega}$$

其中：
- $\mathbf{p}$ 为表面点位置
- $\mathbf{n}$ 为表面法线
- $\boldsymbol{\omega}$ 为采样方向
- $V(\mathbf{p}, \boldsymbol{\omega})$ 为可见性函数（未遮挡为 1，遮挡为 0）

### 蒙特卡洛估计

使用 $N$ 条射线的蒙特卡洛 (Monte Carlo) 估计：

$$AO(\mathbf{p}) \approx \frac{1}{N} \sum_{i=1}^{N} V(\mathbf{p}, \boldsymbol{\omega}_i)$$

当采样方向 $\boldsymbol{\omega}_i$ 来自余弦加权半球采样 (Cosine-Weighted Hemisphere Sampling) 时，$(\mathbf{n} \cdot \boldsymbol{\omega})$ 项已被隐含在采样 PDF 中，因此估计器简化为简单的命中/未命中统计。

### 余弦加权半球采样

给定均匀随机数 $\xi_1, \xi_2 \in [0, 1)$，余弦加权采样方向为：

$$\boldsymbol{\omega} = \left(\cos(2\pi\xi_2)\sqrt{\xi_1}, \sin(2\pi\xi_2)\sqrt{\xi_1}, \sqrt{1-\xi_1}\right)$$

## 代码结构概览

由于本文件为部分代码片段（<30 行核心逻辑），以下提供完整源码及逐行注释。

## 逐段代码详解

```hlsl
// 第1-3行：注释说明
// Partial code for AO ray generation shader, truncated for brevity.
// The full shader is otherwise essentially identical to the shadow
// ray generation.
```
说明此代码是 AO 射线生成着色器的核心片段，完整着色器结构与阴影射线生成着色器基本一致。

```hlsl
// 第4行：初始化结果累加器
float result = 0;
```
`result` 用于累加所有射线的遮蔽结果，初始值为 0。

```hlsl
// 第6行：循环发射 numRays 条 AO 射线
for (uint i = 0; i < numRays; i++)
{
```
`numRays` 控制每像素发射的 AO 采样射线数。更多射线意味着更低的噪声但更高的计算成本。

```hlsl
    // 第9-10行：生成随机数
    float rnd1 = frac(haltonNext(hState) + randomNext(frameSeed));
    float rnd2 = frac(haltonNext(hState) + randomNext(frameSeed));
```
使用 **Cranley-Patterson 旋转 (Cranley-Patterson Rotation)** 技术：将 Halton 低差异序列的值与伪随机数相加，再取小数部分。这结合了低差异序列的良好分布特性和随机偏移消除相关性的优点。

```hlsl
    // 第11行：余弦加权半球采样
    float3 rndDir = cosineSampleHemisphere(rnd1, rnd2);
```
根据两个随机数生成余弦加权的半球方向向量。该方向在切线空间 (Tangent Space) 中，z 轴朝上。

```hlsl
    // 第15行：将采样方向从切线空间旋转到世界空间
    float3 rndWorldDir = mul(rndDir, createBasis(gbuffer.worldNormal));
```
`createBasis()` 根据表面法线创建正交基 (Orthonormal Basis)，`mul()` 将切线空间方向变换到世界空间 (World Space)。

```hlsl
    // 第18-19行：创建射线负载
    ShadowData shadowPayload;
    shadowPayload.miss = false;
```
使用与阴影射线相同的负载结构 `ShadowData`。初始假设射线被遮挡（`miss = false`），如果射线未命中任何几何体，未命中着色器 (Miss Shader) 会将其设为 `true`。

```hlsl
    // 第21-25行：配置射线参数
    RayDesc ray;
    ray.Origin = position;           // 射线起点：表面位置
    ray.Direction = rndWorldDir;     // 射线方向：随机半球方向
    ray.TMin = g_aoConst.minRayLength;  // 最小距离（避免自相交）
    ray.TMax = g_aoConst.maxRayLength;  // 最大距离（AO 影响范围）
```
- `TMin`：防止射线与自身表面相交（自遮蔽问题）
- `TMax`：限制 AO 的最大遮挡距离，距离超过此值的遮挡物不计入 AO

```hlsl
    // 第29-37行：发射光线
    TraceRay(g_rtScene,
             RAY_FLAG_SKIP_CLOSEST_HIT_SHADER|
                 RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH,
             RaytracingInstanceMaskAll,
             HitType_Shadow,
             SbtRecordStride,
             MissType_Shadow,
             ray,
             shadowPayload);
```
关键优化标志：
- `RAY_FLAG_SKIP_CLOSEST_HIT_SHADER`：跳过最近命中着色器 (Closest-Hit Shader)，因为 AO 只需知道是否被遮挡
- `RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH`：接受第一个命中即停止搜索，无需寻找最近命中

```hlsl
    // 第39行：累加结果
    result += shadowPayload.miss ? 1 : 0;
}
```
如果射线未命中任何物体（即该方向未被遮挡），累加 1。

```hlsl
// 第42行：归一化
result /= numRays;
```
将累加结果除以总射线数，得到 $[0, 1]$ 范围的 AO 值。1 表示完全无遮挡，0 表示完全被遮挡。

## 关键算法深入分析

### 采样策略：Halton + Cranley-Patterson

采样质量直接影响 AO 的噪声水平。本实现采用两级策略：

1. **Halton 序列**：提供低差异的基础采样点分布，确保空间覆盖的均匀性
2. **Cranley-Patterson 旋转**：添加逐帧随机偏移 `randomNext(frameSeed)`，使得：
   - 相邻像素不会出现相同的采样模式
   - 不同帧之间的采样模式不同，有利于时域累积

### 性能优化

使用 `RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH` 标志是关键性能优化。对于 AO 射线，我们只关心"是否被遮挡"，不需要找到最近的遮挡物，因此 BVH 遍历可以在找到第一个交点时立即终止。

## 输入与输出

### 输入
| 名称 | 类型 | 说明 |
|------|------|------|
| `position` | `float3` | 从 G-Buffer 重建的世界空间表面位置 |
| `gbuffer.worldNormal` | `float3` | G-Buffer 中的世界空间法线 |
| `numRays` | `uint` | 每像素 AO 采样射线数 |
| `hState` | `HaltonState` | Halton 序列状态 |
| `frameSeed` | `uint` | 帧随机种子 |
| `g_aoConst.minRayLength` | `float` | AO 射线最小距离 |
| `g_aoConst.maxRayLength` | `float` | AO 射线最大距离 |
| `g_rtScene` | `RaytracingAccelerationStructure` | 场景加速结构 |

### 输出
| 名称 | 类型 | 说明 |
|------|------|------|
| `result` | `float` | AO 值，$[0, 1]$ 范围，1 表示无遮蔽 |

## 与其他文件的关系

- **`HaltonSampling.hlsl`**：提供 `haltonNext()` 函数用于生成低差异采样序列
- **`RayGenerationAndMissShaders.pseudocode.hlsl`**：完整的射线生成着色器伪代码，本文件是其 AO 变体
- **`MultiscaleMeanEstimator.hlsl`**：AO 结果后续会经过时域降噪处理

## 在光线追踪管线中的位置

```
光栅化 G-Buffer → [AO 射线生成着色器] → BVH 遍历 → Miss Shader → AO 值 → 降噪 → 合成
                   ^^^^^^^^^^^^^^^^^^^
                   本文件实现此阶段
```

本着色器位于 DXR 光线追踪管线的**射线生成阶段 (Ray Generation Stage)**，是混合渲染管线中光线追踪阶段的入口。

## 技术要点与注意事项

1. **自相交问题**：`TMin` 需要设置适当的偏移量以避免射线与自身表面相交
2. **AO 半径选择**：`TMax`（即 `maxRayLength`）决定了 AO 的影响范围，值过大会导致全局过暗，值过小则丢失大尺度遮蔽信息
3. **射线数量与性能**：实时应用中通常使用 1-4 条射线/像素，配合时域累积和空间降噪
4. **负载初始化**：`shadowPayload.miss` 初始化为 `false` 是安全的——如果射线命中 any-hit shader 可能会终止，此时默认为遮蔽

## 扩展阅读

- Pharr, M., Jakob, W., Humphreys, G. *Physically Based Rendering*, Chapter 13 - Monte Carlo Integration
- Shirley, P., Chiu, K. *A Low Distortion Map Between Disk and Square*
- DXR Specification: `TraceRay()` 和射线标志
