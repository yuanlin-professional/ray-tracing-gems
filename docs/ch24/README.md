# 第24章：基于光子映射的实时全局光照

> **Chapter 24: Real-Time Global Illumination with Photon Mapping**

## 章节概述

本章介绍了一种使用 DXR (DirectX Raytracing) 实现**实时光子映射 (Real-Time Photon Mapping)** 的全局光照 (Global Illumination, GI) 方案。该方法结合了反射阴影贴图 (Reflective Shadow Map, RSM) 作为初始光子源、DXR 光线追踪进行光子弹射、以及基于光栅化的光子喷溅 (Photon Splatting) 进行密度估计与渲染。

与传统离线光子映射不同，本方案针对实时性能进行了大量优化，包括：
- 基于 RSM 的光子发射（避免从光源直接追踪）
- 自适应核缩放 (Adaptive Kernel Scaling) 以平衡精度与覆盖
- 基于图块 (tile) 的光子密度估计
- 利用光栅化管线进行光子喷溅

## 文件列表

| 文件名 | 说明 | 文档链接 |
|--------|------|----------|
| `RayGeneration.hlsl` | 光子追踪的光线生成着色器 (Ray Generation Shader)，含 Payload 结构定义 | [RayGeneration_hlsl.md](RayGeneration_hlsl.md) |
| `ClosestHit.hlsl` | 最近命中着色器，处理光子与表面的交互 | [ClosestHit_hlsl.md](ClosestHit_hlsl.md) |
| `ValidateAndAddPhoton.hlsl` | 光子验证与存储，含基于图块的密度估计 | [ValidateAndAddPhoton_hlsl.md](ValidateAndAddPhoton_hlsl.md) |
| `SplattingShaders.hlsl` | 光子喷溅的顶点着色器 (VS) 和像素着色器 (PS) | [SplattingShaders_hlsl.md](SplattingShaders_hlsl.md) |
| `UniformScaling.hlsl` | 均匀核缩放：基于密度估计和光线长度的自适应核大小 | [UniformScaling_hlsl.md](UniformScaling_hlsl.md) |
| `KernelModificationForVertexPosition.hlsl` | 核形状修改：将圆形核变形为椭圆以适应光照方向 | [KernelModificationForVertexPosition_hlsl.md](KernelModificationForVertexPosition_hlsl.md) |

## 核心技术架构

```
┌───────────────────────────────────────────────────────────┐
│                   实时光子映射管线                          │
│                                                           │
│  Phase 1: 光子发射                                        │
│  ┌──────────┐                                             │
│  │   RSM    │── 读取光子初始状态 (位置、方向、功率)          │
│  └──────────┘                                             │
│       │                                                   │
│       ▼                                                   │
│  Phase 2: 光子追踪 (DXR)                                  │
│  ┌──────────────┐    ┌───────────────┐                    │
│  │ RayGeneration │──>│  ClosestHit   │── 俄罗斯轮盘赌     │
│  │   (循环弹射)  │<──│  (表面交互)    │── 存储光子          │
│  └──────────────┘    └───────────────┘                    │
│       │                      │                            │
│       │                      ▼                            │
│       │              ┌────────────────┐                   │
│       │              │ValidateAndAdd  │── 视锥裁剪        │
│       │              │   Photon       │── 密度估计更新     │
│       │              └────────────────┘                   │
│       ▼                                                   │
│  Phase 3: 光子喷溅 (光栅化)                                │
│  ┌──────────────┐    ┌───────────────┐                    │
│  │UniformScaling│──>│  Kernel       │                     │
│  │(核大小计算)   │    │ Modification │                     │
│  └──────────────┘    └──────┬────────┘                    │
│                             │                             │
│                             ▼                             │
│                     ┌───────────────┐                     │
│                     │  Splatting    │── VS: 变形+投影      │
│                     │  Shaders     │── PS: 深度测试+累积   │
│                     └───────────────┘                     │
└───────────────────────────────────────────────────────────┘
```

## 关键概念

1. **反射阴影贴图 (Reflective Shadow Map, RSM)**：从光源视角渲染的 G-Buffer，提供光子的初始位置、法线、功率等信息
2. **俄罗斯轮盘赌 (Russian Roulette)**：概率性终止低能量光子路径，保持无偏性
3. **光子喷溅 (Photon Splatting)**：将每个光子作为几何体绘制到屏幕上，累积光照贡献
4. **自适应核缩放 (Adaptive Kernel Scaling)**：根据光子密度和光线长度动态调整喷溅核的大小
5. **核形状修改 (Kernel Shape Modification)**：将圆形核变形为椭圆形，以更好地匹配光照方向分布
6. **基于图块的密度估计 (Tile-Based Density Estimation)**：使用屏幕空间图块统计光子数量，用于自适应核大小计算
