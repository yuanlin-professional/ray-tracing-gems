# transferFunction.h 技术文档

## 文件概述

`transferFunction.h` 实现了体渲染中的传递函数 (Transfer Function)，将标量属性值 $v \in [0,1]$ 和距离参数 $t$ 映射为 RGBA 颜色。提供了 3 种预设的颜色映射方案以及可选的红移 (Redshift) 效果。

**源文件路径**: `Ch_29_Efficient_Particle_Volume_Splatting_in_a_Ray_Tracer/optixParticleVolumes/transferFunction.h`

## 算法与数学背景

### 传递函数 (Transfer Function)

传递函数是体渲染的核心组件，定义了标量场到可视化颜色的映射：

$$\text{TF}: v \in [0,1] \rightarrow (R, G, B, \alpha) \in [0,1]^4$$

### 分段线性插值

每种预设使用 2-3 个控制点的分段线性插值：

$$\text{color}(v) = \text{lerp}(c_k, c_{k+1}, t_k)$$

其中 $t_k = (v - v_k) / (v_{k+1} - v_k)$ 为局部参数。

### 红移效果

对于天体物理数据，距离参数 $t$ 可以映射为红移（远处物体偏红）：

$$r = 1 - e^{-t}$$

当 $r < 0.5$ 时从白色渐变到原始颜色，当 $r \geq 0.5$ 时从原始颜色渐变到红色。

## 完整源码与注解

```cuda
#pragma once
#include <optixu/optixu_math_namespace.h>

// 辅助插值函数
inline __device__ float4 lerp4f(float4 a, float4 b, float c)
{
    return a * (1.f - c) + b * c;
}

// 传递函数主入口
inline __device__ float4 tf(float v, float t, const int tf_type)
{
    float4 color;

    if (tf_type == 1)
    {
        // 预设1: 黄(低) → 白(中) → 红(高)
        // 适合正值数据，低值冷色高值暖色
        if (v < .5f)
            color = lerp4f(make_float4(1,1,0,0), make_float4(1,1,1,0.5f), v*2.f);
        else
            color = lerp4f(make_float4(1,1,1,0.5f), make_float4(1,0,0,1), v*2.f-1.f);
    }
    else if (tf_type == 2)
    {
        // 预设2: 蓝(低) → 白(中) → 红(高) [默认]
        // 经典的冷-暖色映射 (Diverging Colormap)
        if (v < .5f)
            color = lerp4f(make_float4(0,0,1,0), make_float4(1,1,1,0.5f), v*2.f);
        else
            color = lerp4f(make_float4(1,1,1,0.5f), make_float4(1,0,0,1), v*2.f-1.f);
    }
    else if (tf_type == 3)
    {
        // 预设3: 品红(低) → 蓝(中低) → 青(中高) → 白(高)
        // 三段式映射，适合含负值的对称数据
        if (v < .33f)
            color = lerp4f(make_float4(1,0,1,0), make_float4(0,0,1,.33f), (v-0.f)*3.f);
        else if (v < .66f)
            color = lerp4f(make_float4(0,0,1,.33f), make_float4(0,1,1,.66f), (v-.33f)*3.f);
        else
            color = lerp4f(make_float4(0,1,0,.5f), make_float4(1,1,1,1), (v-.66f)*3.f);
    }
    else
    {
        // 默认: 蓝(低) → 红(高)
        color = lerp4f(make_float4(0,0,1,0), make_float4(1,0,0,1), v);
    }

    // 红移效果 (可选)
    if (t > 0.f)
    {
        const float alpha = color.w;  // 保存原始alpha
        const float r = 1.f - expf(-t);
        if (r < .5f)
            color = lerp4f(make_float4(1,1,1,0), color, r*2.f);
        else
            color = lerp4f(color, make_float4(1,0,0,1), r*2.f-1.f);
        color.w = alpha;  // 恢复alpha（红移不影响不透明度）
    }

    return color;
}
```

## 关键算法深入分析

### Alpha 与颜色的耦合

传递函数同时输出颜色和不透明度。一个关键设计是 alpha 值从 0 增长到 1 随着属性值增大：

| 预设 | $v=0$ | $v=0.5$ | $v=1.0$ |
|------|-------|---------|---------|
| TF1 | $\alpha=0$ (黄) | $\alpha=0.5$ (白) | $\alpha=1.0$ (红) |
| TF2 | $\alpha=0$ (蓝) | $\alpha=0.5$ (白) | $\alpha=1.0$ (红) |
| TF3 | $\alpha=0$ (品红) | $\alpha\approx0.5$ (青) | $\alpha=1.0$ (白) |

低属性值的粒子几乎透明，高属性值的粒子完全不透明。

## 输入与输出

**输入**:
- `v`: RBF 评估后的标量值 $\in [0,1]$
- `t`: 距离参数（红移用）
- `tf_type`: 预设编号 (1-3)

**输出**: `float4` RGBA 颜色

## 与其他文件的关系

- 被 `raygen.cu` 包含和调用
- `tf_type` 参数由 CPU 端 (`optixParticleVolumes.cpp`) 通过 ImGui 控制

## 技术要点与注意事项

1. `__device__` 限定符表示此函数仅在 GPU 上执行
2. 红移效果不修改 alpha，仅调制 RGB（因为距离不应影响不透明度）
3. 预设2 是经典的"发散色图"(Diverging Colormap)，中间为白色，适合有自然零点的数据
4. 预设3 专为含负值的数据设计（如暗物质密度场）

## 扩展阅读

- "Design Principles for Transfer Functions" (Ljung et al., 2016)
- ColorBrewer 色图设计工具
- "A Survey of Transfer Functions Suitable for Volume Rendering"
