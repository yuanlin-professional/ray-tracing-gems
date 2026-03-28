# intersection_refinement.h 技术文档

## 文件概述

`intersection_refinement.h` 提供了交点精化 (Intersection Refinement) 和偏移 (Offset) 工具函数，用于解决光线追踪中常见的自交叉 (Self-Intersection) 问题。虽然粒子体渲染本身不直接使用这些函数，但它们作为 OptiX SDK 标准设备端库的一部分被包含。

**源文件路径**: `Ch_29_Efficient_Particle_Volume_Splatting_in_a_Ray_Tracer/device_include/intersection_refinement.h`

## 算法与数学背景

### 平面交点精化

给定射线上的初始交点 $\mathbf{p}_0$ 和三角形所在平面（由法线 $\hat{n}$ 和平面上一点 $\mathbf{q}$ 定义），计算精化后的参数 $t'$：

$$t' = -\frac{\hat{n} \cdot (\mathbf{p}_0 - \mathbf{q})}{\hat{n} \cdot \mathbf{d}}$$

然后更新交点：$\mathbf{p}' = \mathbf{p}_0 + t' \cdot \mathbf{d}$

### 整数偏移法 (Integer Offset)

通过操作浮点数的整数表示来实现微小偏移，避免浮点精度问题：

$$p'_x = \begin{cases}
p_x + \epsilon \cdot n_x & \text{if } |p_x| < \epsilon \\
\text{int\_as\_float}(\text{float\_as\_int}(p_x) + \text{copysign}(\delta, p_x) \cdot n_x) & \text{otherwise}
\end{cases}$$

其中 $\epsilon = 10^{-4}$, $\delta = 8192$。这种方法比简单加减 epsilon 更稳定。

## 代码结构概览

```
intersection_refinement.h
├── intersectPlane()               -- 射线-平面求交
├── offset()                       -- 整数偏移法
└── refine_and_offset_hitpoint()   -- 精化 + 双面偏移
```

## 逐段代码详解

### 整数偏移法

```cuda
static __device__ __inline__ float3 offset(const float3& hit_point, const float3& normal)
{
    const float epsilon = 1.0e-4f;
    const float offset  = 4096.0f * 2.0f;

    float3 offset_point = hit_point;

    // 对每个分量：
    // 如果值非常小（接近0），直接加 epsilon * normal
    // 否则，通过整数运算微调浮点表示
    if ((__float_as_int(hit_point.x) & 0x7fffffff) < __float_as_int(epsilon)) {
        offset_point.x += epsilon * normal.x;
    } else {
        offset_point.x = __int_as_float(
            __float_as_int(offset_point.x) +
            int(copysign(offset, hit_point.x) * normal.x));
    }
    // y, z 分量同理...
}
```

### 精化 + 双面偏移

```cuda
static __device__ __inline__ void refine_and_offset_hitpoint(
    const float3& original_hit, const float3& direction,
    const float3& normal, const float3& plane_point,
    float3& back_hit, float3& front_hit)
{
    // 1. 精化交点
    float refined_t = intersectPlane(original_hit, direction, normal, plane_point);
    float3 refined = original_hit + refined_t * direction;

    // 2. 双面偏移（反射面和折射面）
    if (dot(direction, normal) > 0.0f) {
        back_hit  = offset(refined,  normal);
        front_hit = offset(refined, -normal);
    } else {
        back_hit  = offset(refined, -normal);
        front_hit = offset(refined,  normal);
    }
}
```

## 输入与输出

**输入**: 射线参数、交点位置、表面法线

**输出**: 精化后的前面/背面偏移交点

## 与其他文件的关系

- 来自 OptiX SDK 标准设备端库
- 粒子体渲染不直接使用，但作为通用工具被包含
- 对于需要反射/折射的场景（如半透明粒子），可能会用到

## 技术要点与注意事项

1. 整数偏移法利用 IEEE 754 浮点表示，使偏移量自适应于交点值的量级
2. `__float_as_int` 和 `__int_as_float` 是 CUDA 内置的位重解释函数
3. 近原点区域（$|p| < \epsilon$）使用不同策略，避免整数运算的退化
4. 此技术在 Ray Tracing Gems 第 6 章有详细讨论

## 扩展阅读

- Ray Tracing Gems, Chapter 6: "A Fast and Robust Method for Avoiding Self-Intersection"
- "Robust BVH Ray Traversal" (Woop et al.)
- Watertight Ray-Triangle Intersection
