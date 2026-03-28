# Chapter 10: 高扩展效率的简单负载均衡方案

> **A Simple Load-Balancing Scheme with High Scaling Efficiency**

## 概述

本章提出了一种基于位反转置换 (Bit Reversal Permutation) 的图像空间负载均衡方案。当多个处理器协同进行光线追踪渲染时，该方案将连续的像素块映射到图像的不同区域，确保每个处理器处理的像素在图像空间中均匀分布，从而平衡不同区域的渲染负载差异。

## 文件列表

| 文件 | 用途 | 文档链接 |
|------|------|---------|
| `BitReversal.cpp` | 32 位整数位反转实现 | [BitReversal_cpp.md](BitReversal_cpp.md) |
| `DistributionScheme.cpp` | 像素分配与置换方案 | [DistributionScheme_cpp.md](DistributionScheme_cpp.md) |
| `ImageAssembly.cpp` | 渲染后图像重组装 | [ImageAssembly_cpp.md](ImageAssembly_cpp.md) |

## 核心思想

光线追踪中不同像素的渲染时间差异极大（简单背景 vs 复杂反射）。简单的按行/块分配会导致处理器负载不均。本方案使用位反转置换将连续索引映射到图像中均匀分布的位置，无需预先知道每个像素的渲染代价。

## 算法概要

1. **位反转函数**：$\text{reverse}(f)$ 将整数 $f$ 的二进制位倒序
2. **像素置换**：像素索引 $i$ 映射为 $j = \text{reverse}(i/s) \gg \text{bits} + i \bmod s$
3. **自逆性质**：置换是自逆的 (involutory)，即 $\sigma(\sigma(i)) = i$，简化了重组装

## 依赖关系

三个文件互相关联，`BitReversal.cpp` 被 `DistributionScheme.cpp` 和 `ImageAssembly.cpp` 使用。
