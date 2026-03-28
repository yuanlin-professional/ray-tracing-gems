# Chapter 12: 凹凸终结器问题的微面元阴影函数

> **A Microfacet-Based Shadowing Function to Solve the Bump Terminator Problem**

## 概述

本章解决了凹凸贴图 (Bump Mapping) 中的"终结器问题" (Bump Terminator Problem)——当凹凸贴图扰动后的法线与光线方向几乎垂直时，会在物体表面产生不自然的硬边界阴影。本章提出了一种基于微面元 (Microfacet) 阴影函数的解决方案。

## 文件列表

| 文件 | 用途 | 文档链接 |
|------|------|---------|
| `terminator.cpp` | 微面元阴影函数实现 | [terminator_cpp.md](terminator_cpp.md) |

## 核心思想

通过计算几何法线 $\mathbf{N}$ 与凹凸法线 $\mathbf{N}_\text{bump}$ 之间的偏差角来估计一个 $\alpha^2$ 粗糙度参数，然后使用 Smith 阴影函数对光照强度进行衰减，使得终结器区域的过渡更加平滑自然。

## 依赖关系

本文件为独立实现，不依赖其他章节代码。
