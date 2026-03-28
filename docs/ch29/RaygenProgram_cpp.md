# RaygenProgram.cpp 技术文档

## 文件概述

`RaygenProgram.cpp` 是 Ray Tracing Gems 书中第 29 章的**伪代码清单**，展示了 Slab-based 粒子体渲染算法的 Ray Generation 程序的简化版本。它清晰地展示了算法的核心结构：Ray-AABB 求交、Slab 分段、收集-排序-积分循环。

**源文件路径**: `Ch_29_Efficient_Particle_Volume_Splatting_in_a_Ray_Tracer/RaygenProgram.cpp`

## 完整源码与注解

```cpp
// 伪代码：书中使用的数据结构
// ParticleSample { float t; uint id; };
// PerRayData { int tail; int pad; ParticleSample particles[PARTICLE_BUFFER_SIZE]; };

RT_PROGRAM raygen_program()
{
    optix::Ray ray;
    PerRayData prd;

    generate_ray(launch_index, camera);   // 针孔相机射线生成
    optix::Aabb aabb(volume_bbox_min, volume_bbox_max);

    // Ray-AABB 求交
    float tenter, texit;
    intersect_Aabb(ray, aabb, tenter, texit);

    float3 result_color = make_float3(0.f);
    float result_alpha = 0.f;

    if (tenter < texit)
    {
        // Slab 间距 = 缓冲区大小 * 密度因子 * 半径
        const float slab_spacing =
            PARTICLE_BUFFER_SIZE * particlesPerSlab * radius;
        float tslab = 0.f;

        // Slab 遍历循环
        while (tslab < texit && result_alpha < 0.97f)
        {
            prd.tail = 0;                              // 重置缓冲区
            ray.tmin = fmaxf(tenter, tslab);           // Slab 起始
            ray.tmax = fminf(texit, tslab + slabWidth); // Slab 结束

            if (ray.tmax > tenter)  // 连贯性优化
            {
                // 阶段1: 收集 (BVH 遍历 + Any Hit)
                rtTrace(top_object, ray, prd);

                // 阶段2: 排序 (按 t 值前-后排序)
                sort(prd.particles, prd.tail);

                // 阶段3: 积分 (RBF 评估 + Alpha 合成)
                for (int i = 0; i < prd.tail; i++) {
                    float drbf = evaluate_rbf(prd.particles[i]);
                    float4 color_sample = transfer_function(drbf);

                    // 前-后 Alpha 合成
                    float alpha_1msa = color_sample.w * (1.0 - result_alpha);
                    result_color += alpha_1msa * make_float3(
                        color_sample.x, color_sample.y, color_sample.z);
                    result_alpha += alpha_1msa;
                }
            }
            tslab += slab_spacing;  // 推进到下一个 Slab
        }
    }

    output_buffer[launch_index] = make_color(result_color);
}
```

## 算法与数学背景

### 三阶段管线 (Collect-Sort-Integrate)

这是本章算法的核心架构：

1. **收集 (Collect)**: 通过 `rtTrace()` 遍历 BVH，Any Hit 程序将所有在当前 Slab 内命中的粒子存入 `prd.particles[]`
2. **排序 (Sort)**: 按 $t$ 值从近到远排序，确保前-后合成的正确性
3. **积分 (Integrate)**: 遍历排序后的粒子列表，计算 RBF 值，查询传递函数，进行 alpha 合成

### 正确性条件

Slab 分段的正确性依赖于：
- 每个 Slab 内收集的粒子被完整排序
- 相邻 Slab 之间的积分结果通过 alpha 合成正确拼接
- Slab 间距需大于粒子直径（否则同一粒子可能被两个 Slab 重复计算）

## 与实际实现的差异

| 方面 | 伪代码 | 实际实现 (`raygen.cu`) |
|------|--------|------------------------|
| 射线生成 | `generate_ray()` | 手动计算 `d.x*U + d.y*V + W` |
| AABB 求交 | `intersect_Aabb()` | 内联 Slab Method |
| 排序 | `sort()` | 冒泡排序/双调排序 |
| RBF 评估 | `evaluate_rbf()` | 内联高斯 RBF |
| 传递函数 | `transfer_function()` | `tf()` from `transferFunction.h` |

## 输入与输出

**输入**: 相机参数、场景 AABB、BVH 加速结构

**输出**: `output_buffer[]` -- 像素颜色

## 与其他文件的关系

- 对应 `raygen.cu::raygen_program()` 的简化版
- 使用 `device_include/commonStructs.h` 中的数据结构

## 技术要点与注意事项

1. 伪代码中的 `slabWidth` 与实际实现中的 `slab_spacing` 对应
2. 注意伪代码最后一行有语法错误：`make_color( result_color ) )`（多了一个右括号）
3. 实际实现增加了红移效果和累积缓冲等细节

## 扩展阅读

- Ray Tracing Gems, Chapter 29, Listing 29-3
- "Volume Rendering Integral" (Max, 1995)
