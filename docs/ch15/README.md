# Chapter 15: 采样的重要性

> **On the Importance of Sampling**

## 概述

本章讨论了蒙特卡洛渲染中采样策略的重要性，展示了环境光遮蔽 (AO) 采样器的实现和样本方差估计方法。良好的采样策略可以显著降低蒙特卡洛积分的方差，提高图像质量。

## 文件列表

| 文件 | 用途 | 文档链接 |
|------|------|---------|
| `ao_sampler.cpp` | 余弦加权环境光遮蔽采样器 | [ao_sampler_cpp.md](ao_sampler_cpp.md) |
| `estimate_sample_variance.cpp` | 在线样本方差估计 | [estimate_sample_variance_cpp.md](estimate_sample_variance_cpp.md) |

## 核心思想

- **余弦加权采样**：AO 采样器使用余弦加权半球采样，利用 Malley's method 将均匀随机数变换为余弦分布方向
- **方差估计**：通过单 Pass 算法高效计算样本方差，用于评估渲染收敛程度

## 依赖关系

两个文件相互独立。
