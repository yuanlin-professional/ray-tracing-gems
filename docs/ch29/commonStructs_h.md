# commonStructs.h (optixParticleVolumes) 技术文档

## 文件概述

`commonStructs.h`（位于 `optixParticleVolumes/` 目录下）定义了 CPU 和 GPU 共享的核心数据结构，包括光源结构体 `BasicLight`、光线负载 `PerRayData` 以及关键常量 `PARTICLE_BUFFER_SIZE`。

**源文件路径**: `Ch_29_Efficient_Particle_Volume_Splatting_in_a_Ray_Tracer/optixParticleVolumes/commonStructs.h`

## 完整源码与注解

```cpp
#pragma once

#include <optixu/optixu_vector_types.h>

// 每条光线的粒子缓冲区大小
// 这是 Slab-based 算法的核心参数
// 决定了每次 BVH 遍历能收集的最大粒子数
const int PARTICLE_BUFFER_SIZE = 32;

// 基本光源结构体
struct BasicLight
{
    optix::float3 pos;      // 光源世界空间位置
    optix::float3 color;    // 光源颜色 (RGB)
    int           casts_shadow;  // 是否投射阴影
    int           padding;       // 填充至 32 字节（2的幂对齐）
};

// 每条光线的数据负载 (Per-Ray Data)
struct PerRayData
{
    optix::float3 result;   // 累积颜色结果
    int           tail;     // particles[] 数组的当前末尾索引
    // 粒子缓冲区：每个元素为 float2(t, int_as_float(primIdx))
    optix::float2 particles[PARTICLE_BUFFER_SIZE];
};
```

## 算法与数学背景

### PARTICLE_BUFFER_SIZE 选择

缓冲区大小需要在内存消耗和遍历效率之间平衡：

- **太小** (< 16)：需要更多 Slab 分段，增加 BVH 遍历次数
- **太大** (> 64)：每条光线的 payload 过大，降低 GPU 寄存器利用率
- **推荐值**：32 (Turing 架构) 或更大 (Volta 架构，寄存器更多)

### particles[] 编码

每个粒子样本使用 `float2` 编码：
- `.x` = 光线参数 $t$（到射线原点的距离）
- `.y` = `__int_as_float(primIdx)`（粒子索引的位重解释）

使用 `__int_as_float` 而非类型转换，避免精度损失。

## 输入与输出

- **输入**: 无（纯定义文件）
- **输出**: 被 CPU 和 GPU 代码共享使用

## 与其他文件的关系

- 被 `optixParticleVolumes.cpp` 包含（CPU 端）
- 被 `raygen.cu` 和 `material.cu` 包含（GPU 端）

## 技术要点与注意事项

1. `BasicLight` 使用 `padding` 填充至 32 字节，确保 GPU 端对齐访问效率
2. `PerRayData` 的大小为 $12 + 4 + 32 \times 8 = 272$ 字节，需要注意 OptiX payload 大小限制
3. `PARTICLE_BUFFER_SIZE` 影响 Slab 间距和排序算法选择

## 扩展阅读

- OptiX Payload Size Considerations
- GPU Register Pressure and Occupancy
