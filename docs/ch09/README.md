# Chapter 9: DXR 中的多重命中光线追踪

> **Multi-Hit Ray Tracing in DXR**

## 概述

本章介绍了在 DXR (DirectX Raytracing) 框架下收集光线沿途多个命中点 (Hit Points) 的两种方法。在许多科学可视化 (Scientific Visualization) 和工业应用中，需要获取光线穿过场景时的前 $N$ 个最近交点（而不仅仅是最近的一个），例如 X 射线模拟、体积渲染 (Volume Rendering) 和透明度排序 (Order-Independent Transparency)。

本章提供了两种策略的 HLSL 着色器实现：

1. **朴素方法 (Naive Approach)**：遍历所有交点，通过插入排序 (Insertion Sort) 维护前 $N$ 个最近命中
2. **Node-C 方法 (Node-C Approach)**：在朴素方法的基础上，利用 BVH 节点剔除 (BVH Node Culling) 优化性能

## 文件列表

| 文件 | 用途 | 文档链接 |
|------|------|---------|
| `NaiveAnyHit.hlsl` | 朴素方法的任意命中着色器 | [NaiveAnyHit_hlsl.md](NaiveAnyHit_hlsl.md) |
| `NaiveIntersect.hlsl` | 朴素方法的自定义相交着色器 | [NaiveIntersect_hlsl.md](NaiveIntersect_hlsl.md) |
| `NodeCAnyHit.hlsl` | Node-C 方法的任意命中着色器 | [NodeCAnyHit_hlsl.md](NodeCAnyHit_hlsl.md) |
| `NodeCIntersect.hlsl` | Node-C 方法的自定义相交着色器 | [NodeCIntersect_hlsl.md](NodeCIntersect_hlsl.md) |

## 核心思想

### 问题定义

给定一条光线和参数 $N$（`gNquery`），找到光线沿途最近的 $N$ 个交点，按距离从近到远排序。

### 朴素方法

- 在任意命中着色器 (Any-Hit Shader) 中对每个候选交点执行插入排序
- 使用全局缓冲区 (Global Buffers) 存储排序后的命中数据
- 始终调用 `IgnoreHit()` 拒绝当前交点以继续遍历
- 缺点：即使已找到 $N$ 个近交点，仍遍历所有可能的 BVH 节点

### Node-C 方法

- 核心优化：当新命中恰好存储在第 $N$ 个位置（最远的有效位置）时，**不调用** `IgnoreHit()`
- 隐式接受该交点，使 `RayTCurrent()` 成为新的光线区间 (Ray Interval) 端点
- DXR 运行时随后会自动剔除所有超出此端点的 BVH 节点
- 效果：随着收集到更近的命中点，搜索范围逐步缩小

### 缓冲区布局

命中数据使用像素交错 (Pixel-Interleaved) 布局，步长 (Stride) 为 `pixelDims.x * pixelDims.y`：

```
缓冲区索引:  pixel_0_hit_0, pixel_1_hit_0, ..., pixel_N_hit_0,
             pixel_0_hit_1, pixel_1_hit_1, ..., pixel_N_hit_1,
             ...
```

这种布局对 GPU 的合并内存访问 (Coalesced Memory Access) 友好。

## 依赖关系

- 所有着色器依赖 DXR 运行时和 HLSL 着色器编译器
- 依赖未列出的辅助函数：`getHitBufferIndex`、`intersectTriangle`、`getDiffuseSurfaceColor`、`getGeometricFaceNormal`
- 依赖全局资源：`gHitT`、`gHitDiffuse`、`gHitNdotV`、`gHitCount`、`gNquery`
- Node-C 的相交着色器和任意命中着色器需配合使用
