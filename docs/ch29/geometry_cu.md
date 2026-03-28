# geometry.cu 技术文档

## 文件概述

`geometry.cu` 实现了 OptiX 的自定义几何程序 (Custom Geometry Programs)，包括粒子球体的**求交程序** (Intersection Program) 和**包围盒程序** (Bounding Box Program)。这两个程序定义了粒子作为隐式几何体在光线追踪中的行为。

**源文件路径**: `Ch_29_Efficient_Particle_Volume_Splatting_in_a_Ray_Tracer/optixParticleVolumes/geometry.cu`

## 算法与数学背景

### 粒子求交 -- 投影法

本实现使用一种**近似投影法**而非标准的射线-球体解析求交。对于粒子中心 $\mathbf{c}$ 和射线 $\mathbf{o} + t\mathbf{d}$：

1. 计算 $t$ 为射线原点到粒子中心的距离：$t = |\mathbf{c} - \mathbf{o}|$
2. 在 $t$ 处采样射线位置：$\mathbf{p} = \mathbf{o} + t\mathbf{d}$
3. 检查 $\mathbf{p}$ 是否在粒子球体内：$|\mathbf{c} - \mathbf{p}| < r$

这不是精确的射线-球体求交（精确方法需要求解二次方程），但对于 RBF 体渲染来说足够精确，因为 RBF 在球体边缘的贡献趋近于零。

注意：这里的 $t$ 值实际上不是射线参数，而是到粒子中心的欧几里得距离。

### 包围盒计算

每个粒子的 AABB 为以粒子中心为中心、半径为 `fixed_radius` 的正方体：

$$\text{AABB}_{\min} = \mathbf{c} - (r, r, r), \quad \text{AABB}_{\max} = \mathbf{c} + (r, r, r)$$

## 完整源码与注解

```cuda
#include <optix.h>
#include <optixu/optixu_math_namespace.h>
#include <optixu/optixu_aabb_namespace.h>

using namespace optix;

rtBuffer<float4> positions_buffer;  // 粒子位置 (xyz) + 标量属性 (w)

// 自定义属性变量：传递给 Any Hit/Closest Hit
rtDeclareVariable(float2, particle_rbf, attribute particle_rbf, );
rtDeclareVariable(optix::Ray, ray, rtCurrentRay, );
rtDeclareVariable(float, fixed_radius, , );

// 求交程序：每个粒子调用一次
RT_PROGRAM void particle_intersect(int primIdx)
{
    const float4 pos = positions_buffer[primIdx];
    const float3 pos3 = make_float3(pos.x, pos.y, pos.z);

    // 计算射线原点到粒子中心的距离作为 t 值
    const float t = length(pos3 - ray.origin);

    // 在 t 处的射线位置
    const float3 samplePos = ray.origin + ray.direction * t;

    // 检查采样点是否在粒子半径内
    if ((length(pos3 - samplePos) < fixed_radius) && rtPotentialIntersection(t))
    {
        // 报告求交：传递 t 和粒子索引
        particle_rbf.x = t;
        particle_rbf.y = __int_as_float(primIdx);  // 位重解释存储索引
        rtReportIntersection(0);  // material index 0
    }
}

// 包围盒程序：BVH 构建时调用
RT_PROGRAM void particle_bounds(int primIdx, float result[6])
{
    const float4 position = positions_buffer[primIdx];
    const float radius = fixed_radius;

    optix::Aabb *aabb = (optix::Aabb *) result;

    aabb->m_min.x = position.x - radius;
    aabb->m_min.y = position.y - radius;
    aabb->m_min.z = position.z - radius;

    aabb->m_max.x = position.x + radius;
    aabb->m_max.y = position.y + radius;
    aabb->m_max.z = position.z + radius;
}
```

## 关键算法深入分析

### 为什么不用标准射线-球体求交

标准射线-球体求交需要求解二次方程：

$$t = \frac{-(\mathbf{d} \cdot \Delta) \pm \sqrt{(\mathbf{d} \cdot \Delta)^2 - |\Delta|^2 + r^2}}{1}$$

但在 RBF 体渲染中：
1. 我们需要的不是"表面交点"而是"投影距离"，用于计算 RBF 值
2. 近似法更简单、更快
3. RBF 核函数本身是连续衰减的，精确交点并不重要

### rtPotentialIntersection 机制

`rtPotentialIntersection(t)` 向 OptiX 报告一个候选交点，OptiX 内部会与 `ray.tmin`/`ray.tmax` 比较。如果 $t$ 在范围内，则继续调用 `rtReportIntersection`。

## 输入与输出

**输入**: `positions_buffer` (float4), `fixed_radius`, 当前射线

**输出**: `particle_rbf` 属性 (float2: t + primIdx)

## 与其他文件的关系

- 被 `optixParticleVolumes.cpp::createBoundingBoxProgram()` 和 `createIntersectionProgram()` 加载
- 输出的 `particle_rbf` 属性被 `material.cu::any_hit()` 读取

## 在光线追踪管线中的位置

BVH 遍历阶段：OptiX 遍历加速结构时，对每个候选叶节点调用 `particle_intersect` 进行精确测试。

## 技术要点与注意事项

1. `__int_as_float(primIdx)` 是 CUDA 的位重解释函数，不会改变二进制表示
2. 所有粒子使用相同的 `fixed_radius`（均匀半径），简化了 BVH 构建
3. 包围盒程序仅在 BVH 构建时调用，不影响运行时性能
4. `rtReportIntersection(0)` 的参数 0 是材质索引

## 扩展阅读

- OptiX Custom Geometry Documentation
- "Intersection-Free Ray-Marching" 技术
- 隐式表面 (Implicit Surface) 的光线求交方法
