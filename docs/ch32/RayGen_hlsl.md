# RayGen.hlsl 技术文档

## 文件概述

`RayGen.hlsl` 实现了基于辐射度缓存的镜面反射系统的**第一阶段：DXR 光线追踪**。该文件包含两个着色器函数：
1. **`rayHit`**：Closest Hit Shader，记录光线与场景的交点信息
2. **`rayGen`**：Ray Generation Shader，根据 G-Buffer 的粗糙度 (roughness) 使用 GGX 重要性采样 (GGX Importance Sampling) 生成反射光线，追踪并存储交点属性

**关键设计**：本阶段的 DXR 追踪**仅求交，不着色** (intersection only, no shading)。辐射度查找在后续阶段完成，这使得 DXR 着色器极其轻量，大幅提升追踪性能。

**源文件路径**: `Ch_32_Accurate_Real-Time_Specular_Reflections_with_Radiance_Caching/RayGen.hlsl`

## 算法与数学背景

### GGX 微表面模型 (GGX Microfacet Model)

GGX（又名 Trowbridge-Reitz）分布是现代 PBR 渲染中最常用的微表面法线分布函数 (Normal Distribution Function, NDF)：

$$D_{GGX}(h) = \frac{\alpha^2}{\pi \left((\alpha^2 - 1) \cos^2\theta_h + 1\right)^2}$$

其中 $\alpha = \text{roughness}^2$，$\theta_h$ 为半向量 (half vector) 与法线的夹角。

### GGX 重要性采样

`ImportanceSampleGGX` 根据 GGX 分布采样半向量 $\mathbf{h}$，然后通过反射公式计算反射方向：

$$\mathbf{r} = 2(\mathbf{v} \cdot \mathbf{h})\mathbf{h} - \mathbf{v}$$

重要性采样确保大部分样本集中在 BRDF 贡献最大的方向，减少方差。

### 粗糙度自适应采样数

`SamplesRequired(roughness)` 根据表面粗糙度返回所需的采样数量：
- **低粗糙度**（接近镜面）：反射波瓣 (lobe) 窄，需要较少样本
- **高粗糙度**（接近漫反射）：反射波瓣宽，需要更多样本

### 粗糙度阈值

`RT_ROUGHNESS_THRESHOLD` 定义了使用 DXR 追踪的粗糙度上限。高于此阈值的表面反射过于模糊，使用传统立方体贴图代理交点 (proxy intersection) 即可获得足够精度。

## 代码结构概览

```
rayHit (Closest Hit):
  记录交点属性 (距离, 实例ID, 图元ID, 重心坐标)

rayGen (Ray Generation):
  读取粗糙度 → 粗糙度阈值判断
    → 构建光线原点 → for 循环 (多样本)
      → GGX 重要性采样方向 → TraceRay → 存储交点和光线长度
```

## 完整源码与逐行注释

```hlsl
// ========================
// 最近命中着色器 (Closest Hit Shader)
// ========================
void rayHit(inout Result result)
{
  // 记录光线长度（从原点到命中点的参数 t）
  result.RayLength = RayTCurrent();
  // 记录命中的实例 ID（用于后续查找该实例的材质/属性）
  result.InstanceId = InstanceId();
  // 记录命中的图元（三角形）索引
  result.PrimitiveId = PrimitiveIndex();
  // 记录命中点的重心坐标 (barycentric coordinates)
  // 用于后续插值顶点属性（UV、法线等）
  result.Barycentrics = barycentrics;
}

// ========================
// 光线生成着色器 (Ray Generation Shader)
// ========================
void rayGen()
{
  // 从 G-Buffer 读取当前像素的粗糙度值
  float roughness = LoadRoughness(GBufferRoughness);
  // 根据粗糙度计算所需的采样数量
  // 低粗糙度 → 少量样本，高粗糙度 → 更多样本
  uint sampleCount = SamplesRequired(roughness);

  // 仅当粗糙度低于阈值时使用 DXR 追踪
  // 高粗糙度表面使用传统方法（如立方体贴图代理）
  if (roughness < RT_ROUGHNESS_THRESHOLD) {
    // 从 G-Buffer 深度重建光线原点（世界空间中的表面位置）
    float3 ray_o = ConstructRayOrigin(GBufferDepth);

    // 对每个样本进行追踪
    for (uint sampleIndex = 0;
          sampleIndex < sampleCount; sampleIndex++) {
      // 使用 GGX 分布进行重要性采样，获取反射方向
      // roughness 参数控制采样分布的宽度
      float3 ray_d = ImportanceSampleGGX(roughness);

      // 执行 DXR 光线追踪
      // 仅记录交点信息（rayHit），不执行着色
      TraceRay(ray_o, ray_d, results);
      // 存储交点属性（实例ID、图元ID、重心坐标）
      // 索引按 (x, y, sampleIndex) 排列
      StoreRayIntersectionAttributes(results, index.xy, sampleIndex);
      // 单独存储光线长度到专用纹理
      // 负值可能表示 miss（射向天空）
      RayLengthTarget[uint3(index.xy, sampleIndex)] = rayLength;
    }
  }
}
```

## 关键算法深入分析

### 仅求交设计 (Intersection-Only Design)

传统 DXR 反射实现在 Closest Hit Shader 中执行完整着色（加载材质、采样纹理、计算光照），这导致：
- Closest Hit Shader 复杂度高
- 内存访问模式不连贯（不同命中点需要加载不同材质）
- 着色器占用率 (occupancy) 低

本方案的 `rayHit` 极其简单，仅记录 4 个标量值。着色在后续的 Compute Shader 阶段完成，可以更好地利用 GPU 资源。

### 多样本策略

每个像素追踪 `sampleCount` 条光线，这些光线由 GGX 重要性采样生成不同的反射方向。多样本的好处：
- 粗糙表面的模糊反射需要多方向采样来近似积分
- 不同样本可能命中不同的辐射度缓存，提供更准确的结果

### 存储布局

交点属性存储在 3D 纹理/缓冲区中：
```
维度: [屏幕宽度] x [屏幕高度] x [最大样本数]
每个元素: (InstanceId, PrimitiveId, Barycentrics, RayLength)
```

光线长度单独存储在 `RayLengthTarget` 纹理中，因为它在后续的辐射度查找阶段被频繁访问。

## 输入与输出

### 输入
| 资源 | 类型 | 说明 |
|------|------|------|
| `GBufferRoughness` | Texture2D | G-Buffer 粗糙度 |
| `GBufferDepth` | Texture2D | G-Buffer 深度 |
| 加速结构 | TLAS | 场景的顶层加速结构 |

### 输出
| 资源 | 类型 | 说明 |
|------|------|------|
| 交点属性缓冲区 | Texture2DArray / Buffer | 实例ID、图元ID、重心坐标 |
| `RayLengthTarget` | Texture2DArray | 每个样本的光线长度 |

## 与其他文件的关系

- **SamplePrecomputedRadiance.hlsl**: 读取本文件输出的交点属性和光线长度，进行辐射度查找
- **SampleRadianceProbe.hlsl**: 被 `SamplePrecomputedRadiance` 调用，使用交点位置采样探针

## 在光线追踪管线中的位置

```
[G-Buffer Pass] → 粗糙度 + 深度
                       │
                       ▼
[DXR Pass: RayGen.hlsl] ← 本文件
    │
    ├── rayGen(): 生成反射光线
    │   └── for (每个样本)
    │       ├── ImportanceSampleGGX()
    │       └── TraceRay()
    │           └── rayHit(): 记录交点
    │
    └── 输出: 交点属性 + 光线长度
              │
              ▼
[Compute Pass: SamplePrecomputedRadiance]
```

## 技术要点与注意事项

1. **轻量 Closest Hit**：`rayHit` 仅有 4 条赋值语句，确保 DXR 着色器表 (shader table) 极其简单
2. **粗糙度阈值的选择**：`RT_ROUGHNESS_THRESHOLD` 通常为 0.3-0.5，平衡追踪质量与性能
3. **sampleCount 的动态性**：`SamplesRequired` 可以根据性能预算动态调整采样数
4. **光线长度的符号约定**：负值光线长度可能表示 miss（用于区分命中和未命中）
5. **rayLength 变量**：代码中使用了 `rayLength` 但其来源未显示，可能从 `results` 结构中提取
6. **barycentrics 变量**：在 `rayHit` 中使用的 `barycentrics` 未显示声明，应为系统值 `SV_IntersectionAttributes`

## 扩展阅读

- Walter, B. et al. *Microfacet Models for Refraction through Rough Surfaces*, EGSR 2007 (GGX 分布)
- Karis, B. *Real Shading in Unreal Engine 4*, SIGGRAPH 2013 Course (GGX 重要性采样实现)
- Ray Tracing Gems, Chapter 32: Accurate Real-Time Specular Reflections with Radiance Caching
- Stachowiak, T. *Stochastic Screen-Space Reflections*, GDC 2015
