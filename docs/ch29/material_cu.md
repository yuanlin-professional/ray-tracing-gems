# material.cu 技术文档

## 文件概述

`material.cu` 实现了 OptiX 的 Any Hit 程序，这是粒子体渲染管线中"收集"阶段的核心。当光线与粒子相交时，Any Hit 程序将粒子信息存入光线的 per-ray 缓冲区，然后**忽略该交点**继续遍历，从而在一次 BVH 遍历中收集所有命中的粒子。

**源文件路径**: `Ch_29_Efficient_Particle_Volume_Splatting_in_a_Ray_Tracer/optixParticleVolumes/material.cu`

## 完整源码与注解

```cuda
#include <optix.h>
#include <optixu/optixu_math_namespace.h>
#include "commonStructs.h"

using namespace optix;

// 从 geometry.cu 传递的自定义属性
rtDeclareVariable(float2, particle_rbf, attribute particle_rbf, );
// 光线负载
rtDeclareVariable(PerRayData, prd, rtPayload, );

// Any Hit 程序：收集粒子到缓冲区
RT_PROGRAM void any_hit()
{
    // 检查缓冲区是否已满
    if (prd.tail < PARTICLE_BUFFER_SIZE)
    {
        // 存储粒子样本 (t值, 粒子索引)
        prd.particles[prd.tail] = particle_rbf;
        prd.tail++;

        // 关键：忽略此交点，继续遍历 BVH
        rtIgnoreIntersection();
    }
    // 当缓冲区满时，不调用 rtIgnoreIntersection()
    // 这将接受交点并终止遍历 (默认行为)
}

// Closest Hit 程序：空实现
RT_PROGRAM void closest_hit()
{
    // 不需要 Closest Hit 逻辑
    // 所有工作在 Any Hit 中完成
}
```

## 算法与数学背景

### Any Hit 收集模式

这是本章算法的关键创新之一。传统光线追踪中，Any Hit 用于透明度测试。这里它被重新用途为**粒子收集器**：

1. 每次命中时，将粒子的 $t$ 值和索引存入缓冲区
2. 调用 `rtIgnoreIntersection()` 告诉 OptiX "忽略这个交点"
3. OptiX 继续遍历 BVH，查找更多交点
4. 当缓冲区满时（`tail >= PARTICLE_BUFFER_SIZE`），不调用 `rtIgnoreIntersection()`，OptiX 终止遍历

### 缓冲区满时的行为

当 `prd.tail >= PARTICLE_BUFFER_SIZE` 时：
- 不执行任何操作
- OptiX 默认行为是"接受交点"
- 这将终止当前 Slab 的 BVH 遍历
- Raygen 程序推进到下一个 Slab 继续处理

## 输入与输出

**输入**:
- `particle_rbf` (float2): 从 `geometry.cu` 传递的粒子 $t$ 值和索引
- `prd` (PerRayData): 当前光线的负载

**输出**:
- `prd.particles[prd.tail]`: 存储的粒子样本
- `prd.tail`: 递增的缓冲区尾指针

## 与其他文件的关系

- 读取 `geometry.cu` 设置的 `particle_rbf` 属性
- 修改 `raygen.cu` 创建的 `PerRayData` 负载
- 在 `optixParticleVolumes.cpp::createMaterialPrograms()` 中加载

## 在光线追踪管线中的位置

位于 BVH 遍历的命中测试环节。对于每个 Slab：
1. `rtTrace()` 启动遍历
2. 对每个候选粒子，OptiX 调用 `geometry.cu::particle_intersect()`
3. 求交成功后，调用 `material.cu::any_hit()` 收集粒子
4. 缓冲区满或遍历完成后返回 Raygen

## 技术要点与注意事项

1. **`rtIgnoreIntersection()` 是核心**：没有它，OptiX 会在找到最近交点后停止
2. 缓冲区大小限制意味着密集区域可能丢失粒子，Slab 机制缓解了这个问题
3. `closest_hit()` 虽然为空，但在某些 OptiX 版本中仍需声明
4. 粒子收集顺序不确定（取决于 BVH 遍历顺序），因此后续需要排序
5. 此技术类似于 "rasterization-order independent transparency" 在光追中的对应

## 扩展阅读

- OptiX Any Hit Program Semantics
- "Order-Independent Transparency" (OIT) 技术
- A-Buffer 算法
