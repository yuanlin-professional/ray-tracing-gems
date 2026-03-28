# SamplePrecomputedRadiance.hlsl 技术文档

## 文件概述

`SamplePrecomputedRadiance.hlsl` 实现了基于辐射度缓存的镜面反射系统的**第二阶段：辐射度查找** (Radiance Lookup)。该函数对每个像素的所有反射样本，按优先级从多个辐射度来源进行查找：天空盒 (Skybox)、屏幕空间重投影 (Screen-Space Reprojection)、辐射度探针 (Radiance Probe)。对于无法从缓存中获取辐射度的样本，标记为**缓存未命中** (cache miss) 并追加到后续的实时着色队列中。

**源文件路径**: `Ch_32_Accurate_Real-Time_Specular_Reflections_with_Radiance_Caching/SamplePrecomputedRadiance.hlsl`

## 算法与数学背景

### 加权辐射度累积

最终的反射辐射度通过加权平均计算：

$$L_{\text{reflection}} = \frac{\sum_{i=0}^{N-1} w_i \cdot L_i}{\sum_{i=0}^{N-1} w_i}$$

其中 $w_i$ 是第 $i$ 个样本的 GGX 采样权重，$L_i$ 是该样本的辐射度。代码分别累积分子 `L_total` 和分母 `w_total`，最终在后处理中相除。

### GGX 重要性采样权重

`ImportanceSampleGGX` 返回采样权重 `sampleWeight`，等于：

$$w = \frac{f_r(\omega_i, \omega_o) \cdot \cos\theta_i}{p(\omega_i)}$$

其中 $f_r$ 为 GGX BRDF，$p$ 为采样 PDF。在重要性采样下，该权重的方差远低于均匀采样。

### 辐射度来源的优先级

查找顺序按精度和开销的权衡排列：

1. **天空盒** (rayLength < 0 表示 miss)：最简单，直接采样环境贴图
2. **屏幕空间重投影** (`SampleScreen`)：利用已有的着色结果，精度高但覆盖有限
3. **辐射度探针** (`SampleRadianceProbe`)：预计算的立方体贴图，覆盖范围广但精度较低
4. **缓存未命中**：标记样本，后续通过完整着色处理

### 缓存未命中掩码 (Cache Miss Mask)

使用位掩码 (bitmask) 紧凑地记录哪些样本未命中缓存：

$$\text{cacheMissMask} \mid= (1 \ll \text{sampleId})$$

通过 `bitcount()` 计算未命中数量，避免额外的计数器。

### 反射运动估计 (Reflection Motion Estimation)

为了支持时域抗锯齿 (Temporal Anti-Aliasing, TAA) 和时域去噪 (Temporal Denoising)，需要计算反射像素的运动向量。算法使用最可能的反射方向（`NdotH` 最小的方向，即最接近镜面反射的方向）及其光线长度来近似反射点的运动。

## 代码结构概览

```
读取粗糙度和光线原点 → 初始化累积变量
→ for 循环 (每个样本):
    → GGX 重要性采样获取方向和权重
    → 累积权重
    → 读取光线长度
    → 追踪最可能的反射方向 (minNdotH)
    → 辐射度查找:
        → miss → 天空盒
        → 低粗糙度 → 屏幕空间 → 探针 → 标记未命中
        → 高粗糙度 → 立方体贴图代理
    → 累积加权辐射度
→ 处理缓存未命中（追加到着色队列）
→ 输出: L_total, w_total, 运动向量
```

## 完整源码与逐行注释

```hlsl
void SamplePrecomputedRadiance()
{
  // 从 G-Buffer 读取粗糙度
  float roughness = LoadRoughness(GBufferRoughness);
  // 从 G-Buffer 深度重建世界空间位置（即光线原点）
  float3 rayOrigin = ConstructRayOrigin(GBufferDepth);
  // 累积的加权辐射度（分子）
  float3 L_total = float3(0, 0, 0); // Stochastic reflection
  // 累积的权重和（分母）
  float3 w_total = float3(0, 0, 0); // Sum of weights
  // 主反射方向的光线长度近似（用于运动估计）
  float primaryRayLengthApprox;
  // 追踪 NdotH 最小值（最接近镜面反射的方向）
  float minNdotH = 2.0;  // 初始化为大于 1 的值
  // 缓存未命中的位掩码
  uint cacheMissMask = 0;

  // 遍历每个反射样本
  for (uint sampleId = 0;
        sampleId < RequiredSampleCount(roughness); sampleId++) {
    float3 sampleWeight;
    float NdotH;
    // GGX 重要性采样：获取反射方向、采样权重、N·H 值
    // NdotH = dot(Normal, HalfVector)
    // 越小的 NdotH 表示越偏离法线的反射，即越接近掠射
    float3 rayDir =
          ImportanceSampleGGX(roughness, sampleWeight, NdotH);
    // 累积采样权重
    w_total += sampleWeight;

    // 从第一阶段的输出读取该样本的光线长度
    float rayLength = RayLengthTexture[uint3(threadId, sampleId)];

    // 追踪最可能的反射方向（NdotH 最小 = 最接近镜面反射）
    // 用于后续的反射运动估计
    if (NdotH < minNdotH)
    {
      minNdotH = NdotH;
      primaryRayLengthApprox = rayLength;
    }

    // 辐射度初始化为 0（缓存未命中时保持 0）
    float3 radiance = 0; // For cache misses, this will remain 0.

    // 光线长度 < 0 表示 miss（光线未命中任何几何体）
    if (rayLength < 0)
      // 从天空盒/环境贴图采样辐射度
      radiance = SampleSkybox(rayDir);
    // 低粗糙度：使用 DXR 追踪结果进行精确查找
    else if (roughness < RT_ROUGHNESS_THRESHOLD) {
      // 根据光线原点 + 方向 * 长度计算命中点位置
      float3 hitPos = rayOrigin + rayLength * rayDir;

      // 优先级1：尝试屏幕空间重投影
      // 如果命中点可以在当前帧的屏幕上找到，直接使用该像素颜色
      if (!SampleScreen(hitPos, radiance)) {
        // 优先级2：遍历所有辐射度探针
        uint c;
        for (c = 0; c < CubeMapCount; c++)
          // 尝试从第 c 个探针采样辐射度
          // 如果成功（命中点在探针范围内且通过验证），跳出循环
          if (SampleRadianceProbe(c, hitPos, radiance)) break;

        // 如果所有探针都未能提供辐射度
        if (c == CubeMapCount)
          // 标记该样本为缓存未命中
          cacheMissMask |= (1 << sampleId); // Sample was not found.
      }
    }
    // 高粗糙度：使用传统立方体贴图代理交点（无需精确追踪）
    else
      radiance = SampleCubeMapsWithProxyIntersection(rayDir);

    // 累积加权辐射度
    L_total += sampleWeight * radiance;
  }

  // ============ 后处理 ============

  // 统计缓存未命中的样本数量
  uint missCount = bitcount(cacheMissMask);
  // 将未命中信息追加到着色输入队列
  // 后续的着色 Pass 将对这些样本执行完整着色
  AppendToRayShadeInput(missCount, threadId, cacheMissMask);

  // 输出累积的加权辐射度（分子）
  L_totalTexture[threadId] = L_total;
  // 输出累积的权重和（分母）
  w_totalTexture[threadId] = w_total;

  // 计算反射运动向量
  // 使用最可能的反射方向 (primaryReflectionDir) 和对应的光线长度
  // 近似反射点在帧间的运动，用于 TAA 的运动补偿
  ReflectionMotionTexture[threadId] =
        CalculateMotion(primaryReflectionDir, primaryRayLengthApprox);
}
```

## 关键算法深入分析

### 多层级辐射度缓存查找

查找策略形成了一个**回退链** (fallback chain)：

```
rayLength < 0 ?
  ├── 是 → SampleSkybox()        [零开销]
  └── 否 → roughness < threshold ?
              ├── 是 → SampleScreen() ?
              │          ├── 成功 → 使用屏幕颜色   [低开销]
              │          └── 失败 → SampleRadianceProbe() ?
              │                      ├── 成功 → 使用探针数据 [中开销]
              │                      └── 失败 → 标记 cache miss [高开销]
              └── 否 → SampleCubeMapsWithProxyIntersection() [低开销]
```

这种设计使得大部分像素可以通过低开销的方法获取辐射度，只有少数难以处理的样本才需要昂贵的实时着色。

### 缓存未命中的延迟处理

缓存未命中的样本不会在本 Pass 中着色，而是通过 `AppendToRayShadeInput` 追加到一个着色队列 (shader queue)。这样做的好处：
1. **避免分支发散** (branch divergence)：所有线程执行相同的查找逻辑，不着色的线程不会拖慢着色的线程
2. **批量着色**：未命中样本可以在专门的 Pass 中批量处理，提高着色器利用率
3. **可控开销**：可以限制每帧处理的未命中数量，保持帧率稳定

### NdotH 与主反射方向

在 GGX 微表面模型中，$N \cdot H$（法线与半向量的点积）决定了反射的"集中程度"：
- $N \cdot H \approx 1$：反射方向接近镜面反射方向（最可能发生）
- $N \cdot H \ll 1$：反射方向偏离较远（概率低）

代码追踪 `minNdotH`（实际上应为最大的反射概率方向）用于选择**主反射方向**来计算运动向量。这里的逻辑可能存在混淆 -- 通常 `NdotH` 最大的样本对应最可能的反射方向，但代码寻找最小值。这可能是因为 `ImportanceSampleGGX` 返回的 `NdotH` 在不同上下文中有不同含义。

### 分离的权重累积

`L_total` 和 `w_total` 分别存储到纹理中，最终的反射颜色在后处理 Pass 中计算：

$$L_{\text{final}} = \frac{L_{\text{total}} + L_{\text{miss}}}{w_{\text{total}}}$$

其中 $L_{\text{miss}}$ 是缓存未命中样本在着色后补充的辐射度。

## 输入与输出

### 输入
| 资源 | 类型 | 说明 |
|------|------|------|
| `GBufferRoughness` | Texture2D | 粗糙度 |
| `GBufferDepth` | Texture2D | 深度 |
| `RayLengthTexture` | Texture2DArray | 每样本光线长度（来自 Pass 1） |
| 辐射度探针 | CubeMap 数组 | 预计算的辐射度立方体贴图 |
| 屏幕颜色缓冲 | Texture2D | 当前帧的已渲染颜色（用于屏幕空间重投影） |

### 输出
| 资源 | 类型 | 说明 |
|------|------|------|
| `L_totalTexture` | RWTexture2D<float3> | 累积的加权辐射度 |
| `w_totalTexture` | RWTexture2D<float3> | 累积的权重和 |
| `ReflectionMotionTexture` | RWTexture2D | 反射运动向量 |
| 着色队列 | AppendBuffer | 缓存未命中样本的信息 |

## 与其他文件的关系

- **RayGen.hlsl**: 提供本文件读取的光线长度和交点属性
- **SampleRadianceProbe.hlsl**: 本文件在循环中调用，对单个探针进行采样验证

## 在光线追踪管线中的位置

本文件运行在**Compute Shader** 阶段，位于 DXR 追踪之后、最终合成之前：

```
[DXR Pass: RayGen.hlsl]
    │
    ▼ 交点属性 + 光线长度
[Compute Pass: SamplePrecomputedRadiance.hlsl] ← 本文件
    │
    ├── L_total, w_total → [合成 Pass] → 反射颜色
    │
    ├── 运动向量 → [TAA] → 时域稳定
    │
    └── 缓存未命中 → [着色 Pass] → 补充辐射度
                              │
                              └──→ [合成 Pass]
```

## 技术要点与注意事项

1. **采样数一致性**：`RequiredSampleCount(roughness)` 必须与 `RayGen.hlsl` 中的 `SamplesRequired(roughness)` 返回相同值，否则读取的光线长度会错位
2. **位掩码限制**：`cacheMissMask` 为 `uint`（32位），最多支持 32 个样本。对于大多数实时应用来说足够
3. **L_total 中的零值样本**：缓存未命中的样本贡献 `sampleWeight * 0 = 0`，在后续补充着色后需要正确地加回
4. **屏幕空间重投影的局限**：反射点被遮挡、超出屏幕范围、或在上一帧不可见时，`SampleScreen` 会失败
5. **探针遍历顺序**：从索引 0 到 `CubeMapCount-1` 线性遍历，找到第一个有效探针即停止。实际实现可能按距离排序以提高命中率

## 扩展阅读

- Stachowiak, T. *Stochastic Screen-Space Reflections*, GDC 2015 (屏幕空间反射)
- McAuley, S. et al. *Rendering in "Call of Duty: WWII"*, SIGGRAPH 2018 (辐射度缓存实践)
- Ray Tracing Gems, Chapter 32, Section 32.4: Sampling Precomputed Radiance
- Karis, B. *Real Shading in Unreal Engine 4*, SIGGRAPH 2013 (GGX 重要性采样权重推导)
