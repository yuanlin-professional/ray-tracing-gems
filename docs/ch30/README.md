# Chapter 30: 屏幕空间光子映射焦散

> **Caustics Using Screen Space Photon Mapping**

## 概述

本章介绍了一种基于屏幕空间光子映射 (Screen Space Photon Mapping) 的焦散 (Caustics) 渲染方法。代码展示了光子追踪 (Photon Tracing) 的核心伪代码逻辑，包括从点光源发射光子、追踪光子路径、在合适表面上存储光子、以及通过折射继续传播。

## 文件列表

| 文件 | 用途 | 文档链接 |
|------|------|---------|
| `PhotonTracing.txt` | 光子追踪核心逻辑伪代码 | [PhotonTracing_txt.md](PhotonTracing_txt.md) |

## 核心思想

焦散效应是光线通过折射或反射在表面产生的聚焦亮斑（如水底的光斑）。本方法在屏幕空间执行光子映射：
1. 从点光源均匀发射光子
2. 追踪光子路径（碰撞、存储、折射）
3. 低粗糙度表面允许反射传播
4. 折射后的光子继续追踪

## 依赖关系

本文件为独立的伪代码，不依赖其他章节代码。
