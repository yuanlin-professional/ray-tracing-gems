# Chapter 8: 双线性面片的几何求交方法

> **Cool Patches: A Geometric Approach to Ray-Bilinear Patch Intersections**

## 概述

本章介绍了一种基于几何方法的光线-双线性面片 (Ray-Bilinear Patch) 求交算法。双线性面片 (Bilinear Patch) 由四个角点定义，是比三角形更灵活的图元 (Primitive)，可以表达非平面的四边形曲面。该算法将求交问题归结为一个关于参数 $u$ 的一元二次方程 (Quadratic Equation)，然后对每个有效根计算参数 $v$ 和光线参数 $t$。

本章提供了两种实现：一种在世界坐标系 (World Coordinates) 中工作，另一种在光线中心坐标系 (Ray-Centric Coordinates) 中工作。两种方法产生相同的结果，但后者在坐标变换后数学表达式更简洁。

## 文件列表

| 文件 | 用途 | 文档链接 |
|------|------|---------|
| `rqx.cu` | OptiX GPU 求交程序 | [rqx_cu.md](rqx_cu.md) |
| `cpu_test.cpp` | CPU 测试程序（同时实现世界坐标和光线中心坐标方法） | [cpu_test_cpp.md](cpu_test_cpp.md) |

## 核心思想

### 双线性面片的定义

一个双线性面片由四个角点 $\mathbf{q}_{00}, \mathbf{q}_{10}, \mathbf{q}_{11}, \mathbf{q}_{01}$ 定义，面片上任意一点可以用双线性插值 (Bilinear Interpolation) 表示：

$$\mathbf{q}(u, v) = (1-u)(1-v)\mathbf{q}_{00} + u(1-v)\mathbf{q}_{10} + uv\mathbf{q}_{11} + (1-u)v\mathbf{q}_{01}$$

其中 $u, v \in [0, 1]$。

### 面片的拓扑结构

```
q01 ─────────── q11
 |                |
 | e00        e11 |
 |                |
 |      e10       |
q00 ─────────── q10
```

其中边向量 (Edge Vectors) 为：
- $\mathbf{e}_{10} = \mathbf{q}_{10} - \mathbf{q}_{00}$（底边）
- $\mathbf{e}_{00} = \mathbf{q}_{01} - \mathbf{q}_{00}$（左边）
- $\mathbf{e}_{11} = \mathbf{q}_{11} - \mathbf{q}_{10}$（右边）

### 求交方程

将光线 $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ 代入面片方程，消去参数 $v$ 和 $t$ 后，得到关于 $u$ 的一元二次方程：

$$a + bu + cu^2 = 0$$

其中系数由面片角点和光线参数的混合积 (Mixed Product) 得到。求解此方程后，对每个有效根 $u \in [0, 1]$，再计算对应的 $v \in [0, 1]$ 和 $t > 0$。

### 数值稳定性

使用韦达定理 (Viete's Formula) 计算第二个根，避免灾难性抵消 (Catastrophic Cancellation)：
- 先计算数值稳定的根 $u_1$
- 然后通过 $u_1 \cdot u_2 = a/c$ 得到第二个根 $u_2$

## 依赖关系

- `rqx.cu` 依赖 NVIDIA OptiX 光线追踪框架
- `cpu_test.cpp` 依赖 OptiX 数学库（仅使用 `float3`、`dot`、`cross`、`lerp` 等基础数学函数）
- 两个文件在算法层面相互独立，`cpu_test.cpp` 额外实现了光线中心坐标方法
