# device_include/commonStructs.h 技术文档

## 文件概述

`device_include/commonStructs.h` 是设备端 (GPU) 公共结构体定义文件，与 `optixParticleVolumes/commonStructs.h` 不同，它使用了更清晰的 `ParticleSample` 结构体（而非 `float2` 编码），并定义了 `BasicLight` 和 `DirectionalLight` 光源结构体。此文件更接近书中的伪代码风格。

**源文件路径**: `Ch_29_Efficient_Particle_Volume_Splatting_in_a_Ray_Tracer/device_include/commonStructs.h`

## 完整源码与注解

```cpp
// 基本光源
struct BasicLight
{
#if defined(__cplusplus)
    typedef optix::float3 float3;  // C++ 环境下使用 optix::float3
#endif
    float3 pos;          // 位置
    float3 color;        // 颜色
    int casts_shadow;    // 是否投射阴影
};

// 方向光（带面积采样基向量）
struct DirectionalLight
{
#if defined(__cplusplus)
    typedef optix::float3 float3;
#endif
    float3 direction;    // 光源方向
    float radius;        // 面积光半径
    float3 v0;           // 面积采样基向量1
    float3 v1;           // 面积采样基向量2
    float3 color;        // 颜色
    int casts_shadow;    // 是否投射阴影
};
```

## 与 optixParticleVolumes/commonStructs.h 的对比

| 方面 | device_include 版本 | optixParticleVolumes 版本 |
|------|--------------------|-----------------------------|
| `BasicLight` padding | 无 | 有 (32 字节对齐) |
| `DirectionalLight` | 有 | 无 |
| `ParticleSample` 类型 | 无 (在 ParticleIntersect.cpp 中) | 无 (使用 float2) |
| `PerRayData.particles` | 无 | `float2[32]` |
| C++/CUDA 兼容 | `#if defined(__cplusplus)` | `optix::float3` 直接使用 |

## 输入与输出

- **输入**: 无（纯定义文件）
- **输出**: 被 GPU 和 CPU 代码共享使用

## 与其他文件的关系

- 被 `helpers.h`、`intersection_refinement.h` 等设备端头文件间接使用
- 被 `ParticleIntersect.cpp`（书中伪代码）引用

## 技术要点与注意事项

1. `#if defined(__cplusplus)` 保护确保在纯 C 环境（如 CUDA 设备编译）中不使用 C++ 命名空间
2. `DirectionalLight` 中的 `v0`, `v1` 用于面积光采样，但在粒子体渲染中未直接使用
3. 此文件来自 OptiX SDK 的通用示例库，不专属于粒子体渲染

## 扩展阅读

- OptiX SDK 示例中的 commonStructs.h 设计模式
