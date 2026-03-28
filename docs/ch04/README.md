# Chapter 4: 天文馆穹顶主相机

> **A Planetarium Dome Master Camera**

## 概述

本章实现了用于天文馆穹幕投影 (Planetarium Dome) 的鱼眼相机 (Fisheye Camera) 光线生成程序，基于 NVIDIA OptiX 光线追踪引擎。该相机模型生成约 180 度视场角的半球投影图像，支持立体渲染 (Stereoscopic Rendering) 和景深 (Depth of Field) 效果。

## 文件列表

| 文件 | 用途 | 文档链接 |
|------|------|---------|
| `boilerplate.cuh` | OptiX 框架代码：缓冲区、随机数、DoF 支持 | [boilerplate_cuh.md](boilerplate_cuh.md) |
| `dome_master_camera.cu` | 穹顶相机光线生成程序 | [dome_master_camera_cu.md](dome_master_camera_cu.md) |

## 核心思想

穹顶主相机 (Dome Master Camera) 将图像中每个像素映射到半球面上的一个方向，实现等距鱼眼投影 (Equidistant Fisheye Projection)。与透视投影不同，像素到投影中心的距离与对应方向的天顶角 $\theta$ 成正比：

$$\theta = d \times \frac{\text{FoV}}{D}$$

其中 $d$ 是像素到视口中心的距离，$D$ 是视口直径。

## 代码来源

代码源自 Tachyon 光线追踪引擎和 VMD 分子可视化软件，由作者 John E. Stone 开发。

## 依赖关系

`dome_master_camera.cu` 包含 `boilerplate.cuh`，后者提供 OptiX 框架声明和辅助函数。
