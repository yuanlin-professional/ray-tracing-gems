# Chapter 26: 延迟混合路径追踪

> **Deferred Hybrid Path Tracing**

## 概述

本章介绍了一种延迟混合路径追踪 (Deferred Hybrid Path Tracing) 技术，结合了光栅化的 G-Buffer 和光线追踪的路径追踪。代码片段展示了如何根据表面粗糙度和曲率估计反射叶瓣 (Lobe) 的屏幕空间足迹 (Footprint)，用于指导采样核大小。

## 文件列表

| 文件 | 用途 | 文档链接 |
|------|------|---------|
| `LobeFootprintEstimation.glsl` | 反射叶瓣足迹估计 | [LobeFootprintEstimation_glsl.md](LobeFootprintEstimation_glsl.md) |

## 核心思想

在延迟混合路径追踪中，需要确定每个像素的反射采样区域大小。这个区域取决于：
1. 表面粗糙度 (Roughness) — 粗糙度越高，叶瓣越宽
2. 光线传播距离 — 距离越远，足迹越大
3. 表面曲率 (Curvature) — 凸面缩小足迹，凹面扩大足迹
4. 视角 (NoV) — 掠射角导致叶瓣拉伸

## 依赖关系

本文件为独立的 GLSL 代码片段，依赖 G-Buffer 中的法线、深度和粗糙度信息。
