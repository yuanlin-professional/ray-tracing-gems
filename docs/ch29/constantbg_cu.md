# constantbg.cu 技术文档

## 文件概述

`constantbg.cu` 实现了 OptiX 的 Miss 程序 (Miss Program)，当光线未命中场景中的任何几何体时被调用。它简单地将光线结果设置为恒定背景颜色。

**源文件路径**: `Ch_29_Efficient_Particle_Volume_Splatting_in_a_Ray_Tracer/optixParticleVolumes/constantbg.cu`

## 完整源码与注解

```cuda
#include <optix_world.h>

// CPU 端设置的背景颜色变量
// 默认值: float3(0.07, 0.11, 0.17) -- 深蓝灰色
rtDeclareVariable(float3, bg_color, , );

// 光线负载结构体 (与 raygen.cu 中使用的 PerRayData 不同)
// 此 Miss 程序用于 radiance 射线类型
struct PerRayData_radiance
{
    float3 result;       // 光线颜色结果
    float importance;    // 重要性权重（用于递归光线裁剪）
    int depth;           // 递归深度
};

rtDeclareVariable(PerRayData_radiance, prd_radiance, rtPayload, );

// Miss 程序：光线未命中任何几何体
RT_PROGRAM void miss()
{
    prd_radiance.result = bg_color;
}
```

## 算法与数学背景

Miss 程序是光线追踪管线中的"兜底"环节。对于粒子体渲染，大部分像素可能穿越了粒子场但累积了不满 100% 的不透明度，最终的颜色将是：

$$C_{\text{final}} = C_{\text{particles}} + (1 - \alpha_{\text{particles}}) \cdot C_{\text{bg}}$$

注意：在本实现中，`raygen.cu` 直接将背景混合为黑色（不使用此 Miss 程序的返回值），因为 Raygen 内部已经处理了 AABB 测试。此 Miss 程序主要服务于未命中 AABB 的光线。

## 输入与输出

**输入**: `bg_color` -- CPU 端设置的背景颜色

**输出**: `prd_radiance.result` -- 写入光线负载的颜色

## 与其他文件的关系

- 在 `optixParticleVolumes.cpp::createContext()` 中注册：
  ```cpp
  context->setMissProgram(0, context->createProgramFromPTXFile(ptxPath("constantbg.cu"), "miss"));
  context["bg_color"]->setFloat(0.07f, 0.11f, 0.17f);
  ```

## 技术要点与注意事项

1. 使用 `PerRayData_radiance` 而非 `PerRayData`（后者包含粒子缓冲区），因为 Miss 程序不需要粒子数据
2. `bg_color` 可通过 ImGui 界面或命令行参数修改
3. 实际渲染中，此函数很少被调用，因为 raygen 内部已处理了 AABB 外的光线

## 扩展阅读

- OptiX Miss Program Documentation
- 环境映射 (Environment Map) 作为 Miss 程序的替代实现
