# InitRay.hlsl 技术文档

## 文件概述

`InitRay.hlsl` 是 Frostbite 引擎中交互式光照贴图预览系统的**基础版路径追踪光照积分内核** (Path Tracing Light Integration Kernel)。该文件实现了一个简单的蒙特卡洛路径追踪器 (Monte Carlo Path Tracer)，通过在半球上均匀采样随机方向来计算每个光照贴图纹素 (texel) 的入射辐照度 (irradiance)。

**源文件路径**: `Ch_23_Interactive Light Map and Irradiance Volume Preview in Frostbite/InitRay.hlsl`

## 算法与数学背景

### 渲染方程

该代码实现的是渲染方程 (Rendering Equation) 的蒙特卡洛 (Monte Carlo) 近似：

$$L_o(x, \omega_o) = L_e(x, \omega_o) + \int_{\Omega} f_r(x, \omega_i, \omega_o) \, L_i(x, \omega_i) \, (\omega_i \cdot n) \, d\omega_i$$

其中：
- $L_o$ 为出射辐射度 (outgoing radiance)
- $L_e$ 为自发光辐射度 (emissive radiance)
- $f_r$ 为双向反射分布函数 (BRDF)
- $L_i$ 为入射辐射度 (incoming radiance)
- $\omega_i \cdot n$ 为入射方向与法线的余弦项 (cosine term)

### 蒙特卡洛路径追踪

路径追踪通过递归采样实现上述积分的近似。对于 Lambert 漫反射 (Lambertian Diffuse) 表面，BRDF 为常数 $f_r = \frac{\rho}{\pi}$，其中 $\rho$ 为反照率 (albedo)。

本代码使用均匀半球采样 (uniform hemisphere sampling)，采样 PDF 为：

$$p(\omega) = \frac{1}{2\pi}$$

因此蒙特卡洛估计器为：

$$L \approx \frac{1}{N} \sum_{i=1}^{N} \frac{f_r \cdot L_i(\omega_i) \cdot \cos\theta_i}{p(\omega_i)} = \frac{1}{N} \sum_{i=1}^{N} 2 \rho \cdot L_i(\omega_i) \cdot \cos\theta_i$$

代码中的 `pathThroughput` 累积了 $\rho \cdot \cos\theta$ 项（缺少 $\frac{2\pi}{2\pi}$ 的归一化因子，因为在均匀半球采样下 BRDF 与 PDF 的 $\pi$ 项相互抵消）。

## 代码结构概览

```
初始化光线 → 循环追踪 → [命中判断] → 累积辐射度 → 弹射 → 返回结果
```

文件总共 23 行，实现了一个完整的路径追踪循环。

## 完整源码与逐行注释

```hlsl
// Kernel code describing a simple light integration.
// 内核代码：描述一个简单的光照积分过程

// 从纹素原点 (texelOrigin) 沿随机方向 (randomDirection) 初始化光线
Ray r = initRay(texelOrigin, randomDirection);

// 输出辐射度，初始化为 0（黑色）
float3 outRadiance = 0;
// 路径吞吐量 (path throughput)，初始化为 1（完全传递）
// 用于累积沿路径各次弹射的衰减系数
float3 pathThroughput = 1;

// 主循环：沿路径追踪，直到达到最大深度 (maxDepth)
while (pathVertexCount++ < maxDepth) {
    // 声明光线追踪结果数据结构
    PrimaryRayData rayData;
    // 执行 DXR 光线追踪 (TraceRay)，结果存入 rayData
    TraceRay(r, rayData);

    // 如果光线未命中任何几何体（射向天空）
    if (!rayData.hasHitAnything) {
        // 累积天空穹顶 (sky dome) 的辐射度，乘以路径吞吐量
        outRadiance += pathThroughput * getSkyDome(r.Direction);
        // 路径终止
        break;
    }

    // 命中几何体：累积该命中点的自发光 (emissive) 辐射度
    outRadiance += pathThroughput * rayData.emissive;

    // 更新光线原点到命中点位置
    // hitT 为光线参数 t，表示从原点到命中点的距离
    r.Origin = r.Origin + r.Direction * rayData.hitT;
    // 在命中点法线的半球上均匀采样新的随机方向
    r.Direction = sampleHemisphere(rayData.Normal);
    // 更新路径吞吐量：乘以反照率 (albedo) 和余弦项 cos(theta)
    // albedo 代表表面反射率，cos(theta) 为 Lambert 余弦定律
    pathThroughput *= rayData.albedo * dot(r.Direction, rayData.Normal);
}

// 返回累积的总辐射度
return outRadiance;
```

## 关键算法深入分析

### 路径吞吐量 (Path Throughput)

`pathThroughput` 变量跟踪光线沿路径传播时的能量衰减。每次弹射时：

$$T_{k+1} = T_k \cdot \rho_k \cdot \cos\theta_k$$

其中 $T_0 = 1$，$\rho_k$ 是第 $k$ 次弹射的反照率，$\theta_k$ 是出射方向与法线的夹角。

### 均匀半球采样的问题

使用 `sampleHemisphere()` 进行均匀半球采样会导致较高的方差 (variance)，因为：
1. 掠射角 (grazing angle) 方向的采样权重与法线方向相同，但贡献的辐照度很低
2. 没有对光源进行定向采样，依赖随机碰中光源

这些问题在改进版 `InitRayImproved.hlsl` 中得到了解决。

## 输入与输出

| 参数 | 类型 | 说明 |
|------|------|------|
| `texelOrigin` | `float3` | 光照贴图纹素在世界空间中的位置 |
| `randomDirection` | `float3` | 初始随机方向 |
| `maxDepth` | `int` | 最大路径弹射深度 |
| **返回值** | `float3` | 累积的出射辐射度 (RGB) |

## 与其他文件的关系

- **InitRayImproved.hlsl**: 本文件的改进版本，使用余弦加权采样和局部光源直接采样
- **PerformanceBudgeting.hlsl**: 控制每帧有多少纹素执行本内核
- **VarianceTracking.hlsl**: 跟踪本内核输出的方差，判断收敛状态

## 在光线追踪管线中的位置

本代码运行在 DXR 光线追踪管线的 **Ray Generation Shader** 阶段（或作为 Compute Shader 调用 `TraceRay`）。每个线程处理一个光照贴图纹素，通过多帧渐进累积 (progressive accumulation) 最终收敛。

```
[Ray Generation] ──TraceRay──> [Intersection/Any Hit/Closest Hit] ──返回──> [继续循环或终止]
```

## 技术要点与注意事项

1. **无显式 BRDF 归一化**：代码中 `pathThroughput *= albedo * cos(theta)` 隐含了均匀半球采样 PDF ($\frac{1}{2\pi}$) 与 Lambert BRDF ($\frac{\rho}{\pi}$) 的比值简化
2. **仅支持漫反射**：该积分器仅处理 Lambert 漫反射表面
3. **无俄罗斯轮盘赌**：路径仅通过 `maxDepth` 硬截断终止，可能引入偏差 (bias) 或浪费计算
4. **自发光仅在命中时累积**：天空穹顶仅在光线完全 miss 时采样

## 扩展阅读

- Pharr, M., Jakob, W., & Humphreys, G. *Physically Based Rendering: From Theory to Implementation* (PBRT), Chapter 13-14: Monte Carlo Integration & Path Tracing
- Veach, E. *Robust Monte Carlo Methods for Light Transport Simulation*, 1997
- Ray Tracing Gems, Chapter 23: Interactive Light Map and Irradiance Volume Preview in Frostbite
