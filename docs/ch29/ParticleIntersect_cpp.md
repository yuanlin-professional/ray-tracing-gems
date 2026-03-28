# ParticleIntersect.cpp 技术文档

## 文件概述

`ParticleIntersect.cpp` 是 Ray Tracing Gems 书中第 29 章的**伪代码清单** (Pseudocode Listing)，展示了粒子求交 (Intersection) 和 Any Hit 收集程序的简化版本。它与 `optixParticleVolumes/geometry.cu` 和 `optixParticleVolumes/material.cu` 的功能对应，但使用了不同的数据结构名称以便于教学。

**源文件路径**: `Ch_29_Efficient_Particle_Volume_Splatting_in_a_Ray_Tracer/ParticleIntersect.cpp`

## 完整源码与注解

```cpp
// 自定义属性：传递给 Any Hit
rtDeclareVariable(ParticleSample, hit_particle, attribute hit_particle, );

// 粒子求交程序
RT_PROGRAM void particle_intersect(int primIdx)
{
    // 获取粒子中心坐标 (从 float4 提取 xyz)
    const float3 center = make_float3(particles_buffer[primIdx]);

    // 近似 t 值: 射线原点到粒子中心的距离
    const float t = length(center - ray.origin);

    // 在 t 处的射线采样位置
    const float3 sample_pos = ray.origin + ray.direction * t;

    // 检查采样点是否在粒子半径内
    const float3 offset = center - sample_pos;
    if (dot(offset, offset) < radius * radius &&
        rtPotentialIntersection(t))
    {
        hit_particle.t = t;           // 距离
        hit_particle.id = primIdx;    // 粒子索引 (直接用 uint)
        rtReportIntersection(0);
    }
}

// Any Hit 程序：收集粒子
RT_PROGRAM void any_hit()
{
    if (prd.tail < PARTICLE_BUFFER_SIZE) {
        // 将粒子样本添加到缓冲区
        prd.particles[prd.tail++] = hit_particle;
        // 忽略交点，继续遍历
        rtIgnoreIntersection();
    }
    // 缓冲区满时：接受交点，终止遍历
}
```

## 与实际实现的差异

| 方面 | 伪代码 (本文件) | 实际实现 (`geometry.cu` + `material.cu`) |
|------|-----------------|------------------------------------------|
| 粒子样本类型 | `ParticleSample{t, id}` | `float2(t, __int_as_float(primIdx))` |
| 距离检查 | `dot(offset, offset) < r*r` | `length(pos3 - samplePos) < r` |
| 半径变量 | `radius` | `fixed_radius` |
| 缓冲区大小 | 31 (Turing) / 255 (Volta) | 32 |

## 输入与输出

**输入**: 粒子缓冲区、当前射线、粒子半径

**输出**: `prd.particles[]` 中的粒子样本

## 与其他文件的关系

- 对应 `geometry.cu::particle_intersect()` 和 `material.cu::any_hit()` 的简化版
- 使用 `device_include/commonStructs.h` 中的 `ParticleSample` 结构体

## 技术要点与注意事项

1. 伪代码使用 `dot(offset, offset)` 替代 `length()`，避免不必要的开方运算
2. 书中建议 `PARTICLE_BUFFER_SIZE` 在 Turing 架构上设为 31（适配寄存器限制）
3. 实际实现中使用 `__int_as_float` 编码索引，此处使用 `uint id` 更清晰

## 扩展阅读

- Ray Tracing Gems, Chapter 29, Listing 29-1 和 29-2
