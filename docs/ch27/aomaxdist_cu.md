# aomaxdist.cu 技术文档

## 文件概述

**文件路径**: `Ch_27_Interactive_Ray_Tracing_Techniques_for_High-Fidelity_Scientific_Visualization/aomaxdist.cu`

本文件实现了带**最大遮蔽距离限制 (Maximum Occluder Distance)** 的**环境遮蔽 (Ambient Occlusion, AO)** 着色功能。该代码使用 NVIDIA OptiX 光线追踪框架编写，通过发射阴影射线并将最大追踪距离限制为 `ao_maxdist`，实现局部化的 AO 效果。这对于科学可视化中密集排列的分子结构特别有用——仅计算近距离的遮蔽，避免远距离物体过度影响 AO 结果。

## 算法与数学背景

### 距离限制环境遮蔽

标准 AO 积分中引入最大距离 $d_{max}$：

$$AO(\mathbf{p}) = \frac{1}{\pi} \int_{\Omega} V_{d_{max}}(\mathbf{p}, \boldsymbol{\omega}) (\mathbf{n} \cdot \boldsymbol{\omega}) \, d\boldsymbol{\omega}$$

其中可见性函数被修改为：

$$V_{d_{max}}(\mathbf{p}, \boldsymbol{\omega}) = \begin{cases} 0 & \text{if ray hits within } [0, d_{max}] \\ 1 & \text{otherwise} \end{cases}$$

### 蒙特卡洛估计

使用 $N$ 条阴影射线进行估计：

$$AO \approx \frac{1}{N} \sum_{i=1}^{N} (\mathbf{n} \cdot \boldsymbol{\omega}_i) \cdot V_{d_{max}}(\mathbf{p}, \boldsymbol{\omega}_i)$$

## 代码结构概览

```
aomaxdist.cu
├── struct PerRayData_radiance    // 辐射射线负载
├── struct PerRayData_shadow      // 阴影射线负载
├── shade_ambient_occlusion()     // AO 着色函数
└── closest_hit_shader()          // 最近命中着色器
```

## 逐段代码详解

### 射线负载结构体

```cuda
struct PerRayData_radiance {
  float3 result;     // Final shaded surface color
  // ...
}

struct PerRayData_shadow {
  float3 attenuation;  // 阴影衰减因子
};
```

- `PerRayData_radiance`：主射线（辐射射线）的负载，携带最终着色颜色
- `PerRayData_shadow`：阴影射线的负载，使用 `float3` 衰减值支持彩色阴影（如透过彩色玻璃的阴影）

### OptiX 变量声明

```cuda
rtDeclareVariable(PerRayData_radiance, prd, rtPayload, );
rtDeclareVariable(PerRayData_shadow, prd_shadow, rtPayload, );
rtDeclareVariable(float, ao_maxdist, , ); // max AO occluder distance
```

OptiX 的变量声明宏：
- `rtPayload`：射线负载语义
- `ao_maxdist`：用户可配置的 AO 最大遮蔽距离参数

### AO 着色函数

```cuda
static __device__ float3 shade_ambient_occlusion(
    float3 hit, float3 N, float aoimportance) {
  // Skipping boilerplate AO shadowing material here ...

  for (int s=0; s<ao_samples; s++) {
    Ray aoray;
    // Skipping boilerplate AO shadowing material here ...
    aoray = make_Ray(hit, dir, shadow_ray_type, scene_epsilon, ao_maxdist);
```

关键行 `make_Ray(hit, dir, shadow_ray_type, scene_epsilon, ao_maxdist)`：
- `hit`：射线起点（表面交点）
- `dir`：随机采样的半球方向
- `shadow_ray_type`：使用阴影射线类型
- `scene_epsilon`：最小射线距离（防止自相交）
- **`ao_maxdist`**：最大射线距离——这是本文件的核心参数

```cuda
    shadow_prd.attenuation = make_float3(1.0f);
    rtTrace(root_shadower, ambray, shadow_prd);
    inten += ndotambl * shadow_prd.attenuation;
  }

  return inten * lightscale;
}
```

- 初始化衰减为 1.0（完全无阴影）
- `rtTrace()`：OptiX 的射线追踪调用
- 如果射线在 `ao_maxdist` 范围内命中遮挡物，`shadow_prd.attenuation` 会被修改（通常变为 0 或减小）
- 累加贡献：$\text{inten} += (\mathbf{n} \cdot \boldsymbol{\omega}) \cdot \text{attenuation}$

### 最近命中着色器中的 AO 集成

```cuda
RT_PROGRAM void closest_hit_shader( ... ) {
  // Skipping boilerplate closest-hit shader material here ...

  // Add ambient occlusion diffuse lighting, if enabled.
  if (AO_ON && ao_samples > 0) {
    result *= ao_direct;
    result += ao_ambient * col * p_Kd *
        shade_ambient_occlusion(hit_point, N, fogmod * p_opacity);
  }

  // Continue with typical closest-hit shader contents ...

  prd.result = result; // Pass the resulting color back up the tree.
}
```

AO 集成逻辑：
1. `result *= ao_direct`：将直接光照部分乘以直接光系数
2. `ao_ambient * col * p_Kd * shade_ambient_occlusion(...)`：
   - `ao_ambient`：环境光强度
   - `col`：材质颜色
   - `p_Kd`：漫反射系数
   - `shade_ambient_occlusion()`：AO 遮蔽因子
3. `fogmod * p_opacity`：AO 重要性权重，考虑雾效和透明度

## 关键算法深入分析

### ao_maxdist 参数的影响

| `ao_maxdist` 值 | 效果 | 适用场景 |
|------------------|------|----------|
| 小（如场景尺寸的 1%） | 仅显示非常局部的遮蔽 | 紧密排列的分子 |
| 中等 | 适度的全局感 | 一般场景 |
| 大（近无穷） | 完整 AO，类似全局光照 | 建筑、大尺度场景 |

对于科学可视化中密集排列的球体模型（如蛋白质），小的 `ao_maxdist` 值可以：
- 突出原子之间的空间关系
- 避免远处不相关的几何体影响遮蔽
- 提高性能（射线更早终止）

### 阴影射线的衰减模型

使用 `float3 attenuation` 而非简单的 `bool miss` 允许：
- **半透明遮蔽**：透明物体可以部分遮挡 AO 射线
- **彩色遮蔽**：有色透明物体可以给 AO 添加色调
- **叠加衰减**：多个半透明物体的衰减可以叠加

## 输入与输出

### 输入
| 名称 | 类型 | 说明 |
|------|------|------|
| `hit` | `float3` | 表面交点位置 |
| `N` | `float3` | 表面法线 |
| `aoimportance` | `float` | AO 重要性权重 |
| `ao_maxdist` | `float` | 最大 AO 遮蔽距离 |
| `ao_samples` | `int` | AO 采样射线数 |

### 输出
| 名称 | 类型 | 说明 |
|------|------|------|
| `prd.result` | `float3` | 最终着色颜色（含 AO） |

## 与其他文件的关系

- **`edgeoutline.cu`**：边缘描边效果可与 AO 叠加使用
- **`transedge.cu`** / **`transmax.cu`**：透明物体的 AO 处理需要考虑衰减
- **`clipping.cpp`**：裁剪平面影响 AO 射线的遮蔽判断

## 在光线追踪管线中的位置

```
主射线 → BVH 遍历 → closest_hit_shader() → shade_ambient_occlusion()
                                                    │
                                                    ├── AO 阴影射线 1 (TMax = ao_maxdist)
                                                    ├── AO 阴影射线 2
                                                    ├── ...
                                                    └── AO 阴影射线 N
                                                          │
                                                          v
                                                    累加 → AO 值
```

## 技术要点与注意事项

1. **ao_maxdist 的场景依赖性**：该参数需要根据场景尺度调整。分子可视化中通常设为几个原子半径
2. **性能优化**：限制最大距离可以显著减少 BVH 遍历深度，提高 AO 计算性能
3. **aoimportance 加权**：远处或透明物体的 AO 重要性降低，可用于自适应采样
4. **场景类型**：OptiX 的 `root_shadower` 允许使用单独的加速结构进行阴影计算
5. **采样质量**：实际实现中应使用分层采样或低差异序列生成 `dir`

## 扩展阅读

- Pharr, M. *Ambient Occlusion*. GPU Gems, Chapter 17
- Zhukov, S. et al. *An Ambient Light Illumination Model*. EGSR 1998
- Stone, J.E. et al. *GPU-Accelerated Molecular Visualization on Petascale Supercomputing Platforms*. UltraVis 2013
- NVIDIA OptiX Programming Guide
