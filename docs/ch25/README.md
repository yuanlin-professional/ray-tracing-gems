# 第25章：实时光线追踪的混合渲染 (Hybrid Rendering for Real-Time Ray Tracing)

## 章节概述

本章介绍了一种**混合渲染 (Hybrid Rendering)** 方法，将传统的光栅化 (Rasterization) 管线与实时光线追踪 (Real-Time Ray Tracing) 相结合，以在保持实时性能的同时获得高质量的光影效果。该方法利用 G-Buffer 中的深度和法线信息来生成光线追踪所需的几何数据，从而避免全场景光线追踪的高昂开销。

## 核心思想

混合渲染的核心思想是：
1. 使用光栅化完成主视图 (Primary View) 的渲染，生成 G-Buffer（包括深度、法线、材质等信息）
2. 基于 G-Buffer 信息，使用 DXR (DirectX Raytracing) 发射二次光线（阴影射线、环境遮蔽射线、折射射线等）
3. 将光线追踪结果与光栅化结果融合，得到最终图像

## 文件列表

| 文件 | 说明 | 文档链接 |
|------|------|----------|
| `AORayGeneration.hlsl` | 环境遮蔽 (Ambient Occlusion) 光线生成着色器 | [AORayGeneration_hlsl.md](AORayGeneration_hlsl.md) |
| `DistanceMetric.cpp` | 时域降噪中的距离度量权重计算 | [DistanceMetric_cpp.md](DistanceMetric_cpp.md) |
| `HaltonSampling.hlsl` | Halton 低差异序列采样实现 | [HaltonSampling_hlsl.md](HaltonSampling_hlsl.md) |
| `MultiscaleMeanEstimator.hlsl` | 多尺度均值估计器，用于时域滤波 | [MultiscaleMeanEstimator_hlsl.md](MultiscaleMeanEstimator_hlsl.md) |
| `RayGenerationAndMissShaders.pseudocode.hlsl` | 阴影射线生成与未命中着色器伪代码 | [RayGenerationAndMissShaders_pseudocode_hlsl.md](RayGenerationAndMissShaders_pseudocode_hlsl.md) |
| `TransmittanceDetermination.hlsl` | 透射 (Transmittance) 方向确定代码 | [TransmittanceDetermination_hlsl.md](TransmittanceDetermination_hlsl.md) |

## 技术架构

```
┌────────────────────────────────────────────┐
│           光栅化阶段 (Rasterization)        │
│  生成 G-Buffer: 深度、法线、材质 ID 等       │
└────────────────┬───────────────────────────┘
                 │
                 v
┌────────────────────────────────────────────┐
│        光线追踪阶段 (Ray Tracing)           │
│  ┌──────────────┐  ┌──────────────────┐    │
│  │ 阴影射线生成  │  │  AO 射线生成     │    │
│  │ (Shadow Ray)  │  │ (AO Ray Gen)     │    │
│  └──────────────┘  └──────────────────┘    │
│  ┌──────────────┐                          │
│  │ 折射射线生成  │                          │
│  │ (Refraction)  │                          │
│  └──────────────┘                          │
└────────────────┬───────────────────────────┘
                 │
                 v
┌────────────────────────────────────────────┐
│        降噪与滤波 (Denoising & Filtering)   │
│  ┌──────────────────────────────────────┐  │
│  │ 多尺度均值估计器 + 距离度量权重       │  │
│  │ (MultiscaleMeanEstimator + Distance)  │  │
│  └──────────────────────────────────────┘  │
└────────────────┬───────────────────────────┘
                 │
                 v
┌────────────────────────────────────────────┐
│              最终合成输出                    │
└────────────────────────────────────────────┘
```

## 关键算法

1. **Halton 低差异序列**：用于生成高质量的准随机采样方向，提供比伪随机数更好的覆盖性
2. **余弦加权半球采样**：用于 AO 射线方向生成，使采样分布符合余弦权重的 BRDF
3. **多尺度均值估计**：时域降噪算法，自适应地混合历史帧和当前帧数据
4. **Cranley-Patterson 旋转**：将 Halton 序列与随机偏移结合，消除结构化采样模式

## 参考文献

- Pharr, M., Jakob, W., Humphreys, G. *Physically Based Rendering: From Theory to Implementation (PBRT)*
- DXR (DirectX Raytracing) 规范
