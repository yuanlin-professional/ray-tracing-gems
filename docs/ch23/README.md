# 第23章：Frostbite 中的交互式光照贴图与辐照度体积预览

> **Chapter 23: Interactive Light Map and Irradiance Volume Preview in Frostbite**

## 章节概述

本章介绍了在 Frostbite 引擎中实现交互式光照贴图 (Light Map) 和辐照度体积 (Irradiance Volume) 预览的技术方案。该系统利用 DXR (DirectX Raytracing) 实现实时路径追踪 (Path Tracing)，在编辑器中为美术人员提供渐进式的全局光照 (Global Illumination) 预览。核心挑战包括：如何在有限的 GPU 时间预算内高效分配采样资源、如何通过方差跟踪判断收敛状态、以及如何通过改进的光照积分提升收敛速度。

## 文件列表

| 文件名 | 说明 | 文档链接 |
|--------|------|----------|
| `InitRay.hlsl` | 基础版路径追踪光照积分内核 | [InitRay_hlsl.md](InitRay_hlsl.md) |
| `InitRayImproved.hlsl` | 改进版路径追踪光照积分内核（含局部光源直接采样和余弦加权半球采样） | [InitRayImproved_hlsl.md](InitRayImproved_hlsl.md) |
| `PerformanceBudgeting.hlsl` | 性能预算系统，动态调节每帧采样率 | [PerformanceBudgeting_hlsl.md](PerformanceBudgeting_hlsl.md) |
| `VarianceTracking.hlsl` | 方差跟踪与收敛判定 | [VarianceTracking_hlsl.md](VarianceTracking_hlsl.md) |

## 核心技术架构

```
┌─────────────────────────────────────────────┐
│           编辑器交互循环                      │
│                                             │
│  ┌──────────────┐   ┌───────────────────┐   │
│  │ 性能预算系统  │──>│ 采样率 sampleRatio │   │
│  │ (CPU 端)     │   └────────┬──────────┘   │
│  └──────────────┘            │              │
│         ▲                    ▼              │
│         │          ┌─────────────────┐      │
│    GPU 时间反馈    │  光照积分内核    │      │
│         │          │  (GPU 端)       │      │
│         │          └────────┬────────┘      │
│         │                   │               │
│         │                   ▼               │
│         │          ┌─────────────────┐      │
│         └──────────│  方差跟踪       │      │
│                    │  收敛判定       │      │
│                    └─────────────────┘      │
└─────────────────────────────────────────────┘
```

## 关键概念

1. **渐进式渲染 (Progressive Rendering)**：每帧累积少量样本，逐步收敛到最终结果
2. **性能预算 (Performance Budgeting)**：根据 GPU 实际耗时动态调整采样率，保持交互帧率
3. **方差跟踪 (Variance Tracking)**：使用统计学方法判断每个纹素 (texel) 是否已收敛
4. **余弦加权半球采样 (Cosine-Weighted Hemisphere Sampling)**：改进的采样策略，隐式包含 Lambert 余弦项
5. **下一事件估计 (Next Event Estimation)**：在每个路径顶点直接采样光源，加速收敛
