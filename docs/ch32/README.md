# 第32章：基于辐射度缓存的精确实时镜面反射

> **Chapter 32: Accurate Real-Time Specular Reflections with Radiance Caching**

## 章节概述

本章介绍了一种结合 DXR 光线追踪 (Ray Tracing) 和**辐射度缓存 (Radiance Caching)** 的实时镜面反射 (Specular Reflection) 系统。该方法的核心思想是：先通过 DXR 追踪反射光线获取交点位置和距离，然后从预计算的辐射度探针 (Radiance Probe) 或屏幕空间 (Screen Space) 中采样该交点处的辐射度，从而避免在 DXR 中执行昂贵的着色 (shading) 计算。

### 两阶段架构

1. **光线追踪阶段 (Ray Tracing Pass)**：使用 DXR 追踪反射光线，仅记录交点属性（距离、实例 ID、图元 ID、重心坐标），不执行任何着色
2. **辐射度查找阶段 (Radiance Lookup Pass)**：根据交点位置，按优先级从多个来源查找辐射度：
   - 屏幕空间重投影 (Screen-Space Reprojection)
   - 辐射度探针立方体贴图 (Radiance Probe Cube Maps)
   - 天空盒 (Skybox)

## 文件列表

| 文件名 | 说明 | 文档链接 |
|--------|------|----------|
| `RayGen.hlsl` | DXR 光线生成着色器：GGX 重要性采样、光线追踪、存储交点 | [RayGen_hlsl.md](RayGen_hlsl.md) |
| `SamplePrecomputedRadiance.hlsl` | 辐射度查找主逻辑：多源采样、缓存未命中处理、运动估计 | [SamplePrecomputedRadiance_hlsl.md](SamplePrecomputedRadiance_hlsl.md) |
| `SampleRadianceProbe.hlsl` | 单个辐射度探针的采样与验证逻辑 | [SampleRadianceProbe_hlsl.md](SampleRadianceProbe_hlsl.md) |

## 核心技术架构

```
┌────────────────────────────────────────────────────────────────┐
│                 精确实时镜面反射系统                              │
│                                                                │
│  Pass 1: DXR 光线追踪                                          │
│  ┌──────────────────────────────────────────────┐              │
│  │ RayGen.hlsl                                  │              │
│  │   ├── 读取 G-Buffer (粗糙度, 深度)            │              │
│  │   ├── GGX 重要性采样生成反射方向               │              │
│  │   ├── TraceRay (仅求交，不着色)               │              │
│  │   └── 存储: 交点属性 + 光线长度               │              │
│  └──────────────────────────────────────────────┘              │
│                        │                                       │
│                        ▼                                       │
│  Pass 2: 辐射度查找                                             │
│  ┌──────────────────────────────────────────────┐              │
│  │ SamplePrecomputedRadiance.hlsl               │              │
│  │   ├── 对每个样本:                             │              │
│  │   │   ├── 优先级1: 天空盒 (miss)              │              │
│  │   │   ├── 优先级2: 屏幕空间重投影              │              │
│  │   │   ├── 优先级3: 辐射度探针 (SampleRadiance │              │
│  │   │   │            Probe.hlsl)               │              │
│  │   │   └── 优先级4: 标记为缓存未命中            │              │
│  │   ├── 累积加权辐射度                          │              │
│  │   └── 缓存未命中 → 追加到着色队列              │              │
│  └──────────────────────────────────────────────┘              │
│                        │                                       │
│                        ▼                                       │
│  Pass 3: 缓存未命中着色 (可选, 代码未展示)                       │
│  ┌──────────────────────────────────────────────┐              │
│  │ 对缓存未命中的样本执行完整着色                   │              │
│  └──────────────────────────────────────────────┘              │
│                        │                                       │
│                        ▼                                       │
│  最终合成: L_total / w_total + 运动矢量                         │
└────────────────────────────────────────────────────────────────┘
```

## 关键概念

1. **GGX 重要性采样 (GGX Importance Sampling)**：根据 GGX 微表面分布 (Microfacet Distribution) 生成反射方向，集中采样在反射概率最高的方向
2. **辐射度缓存 (Radiance Caching)**：预计算并缓存场景中不同位置的辐射度信息（立方体贴图），避免实时着色的高昂开销
3. **屏幕空间重投影 (Screen-Space Reprojection)**：尝试从当前帧的已渲染图像中查找反射点的颜色，利用已有的着色结果
4. **辐射度探针 (Radiance Probe)**：放置在场景中的立方体贴图探针，存储该位置向各方向看到的辐射度
5. **缓存未命中 (Cache Miss)**：当反射点既不在屏幕上也不在任何探针覆盖范围内时，需要回退到实时着色
6. **粗糙度阈值 (Roughness Threshold)**：低于阈值的表面使用 DXR 精确追踪，高于阈值的使用传统立方体贴图近似
