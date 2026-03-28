# UniformScaling.hlsl 技术文档

## 文件概述

`UniformScaling.hlsl` 实现了实时光子映射系统中的**均匀核缩放** (Uniform Kernel Scaling) 算法。该函数根据两个因素动态计算光子喷溅核 (splatting kernel) 的基础大小：
1. **基于图块的光子密度估计** (tile-based photon density estimation)：光子越密集，核越小
2. **光线长度衰减** (ray length attenuation)：光线越长（间接弹射越远），核越小

两者相乘得到最终的均匀缩放因子。

**源文件路径**: `Ch_24_Real-Time_Global_Illumination_with_Photon_Mapping/UniformScaling.hlsl`

## 算法与数学背景

### 自适应核大小 (Adaptive Kernel Size)

在核密度估计 (Kernel Density Estimation, KDE) 中，核的大小（带宽）至关重要：
- **核太大**：过度模糊，丢失细节
- **核太小**：噪声过多，不够平滑

自适应方法根据局部数据密度调整核大小：在数据密集的区域使用小核，在稀疏的区域使用大核。

### 密度驱动的核半径 (Equation 24.5)

假设图块 (tile) 在视图空间 (view space) 中的投影面积为 $A_{\text{view}}$，图块内的光子数量为 $n_p$，则理想的核半径为：

$$r = \sqrt{\frac{A_{\text{view}}}{\pi \cdot n_p}}$$

其中 $A_{\text{view}}$ 与光子位置的视图空间深度 $z$ 的平方成正比：

$$A_{\text{view}} = z^2 \cdot C_{\text{tile}}$$

$C_{\text{tile}}$ 为与图块角分辨率相关的常量。

### 动态核缩放 (Equation 24.6)

密度驱动的半径经过限幅 (clamp) 后，乘以图块归一化因子：

$$s_d = \text{clamp}(r, s_{\min}, s_{\max}) \cdot n_{\text{tile}}$$

### 光线长度衰减 (Equation 24.2)

光子弹射的光线越长，其能量扩散越广，核应该相应缩小：

$$s_l = \text{clamp}\left(\frac{l}{l_{\max}}, 0.1, 1.0\right)$$

其中 $l$ 为光线长度，$l_{\max}$ 为最大允许光线长度。

### 最终均匀缩放因子

$$s = s_d \cdot s_l$$

## 代码结构概览

```
加载图块光子数 → 计算视图空间面积 → 密度驱动核半径 (Eq 24.5)
  → 限幅和归一化 (Eq 24.6) → 光线长度衰减 (Eq 24.2) → 返回乘积
```

## 完整源码与逐行注释

```hlsl
float uniform_scaling(float3 pp_in_view, float ray_length)
{
    // 基于图块的光子密度估计：
    // 读取光子投影所在图块中的光子数量
    int n_p = load_number_of_photons_in_tile(pp_in_view);
    // 默认核半径（当无法计算时使用的后备值）
    float r = .1f;

    // 仅当存在有效层 (layers > 0) 时进行密度驱动计算
    if (layers > .0f)
    {
        // Equation 24.5: 基于视图空间面积和光子密度计算核半径
        // a_view: 图块在视图空间中的投影面积
        // = z^2 * TileAreaConstant
        // z = pp_in_view.z (光子在视图空间中的深度)
        // TileAreaConstant: 与图块角分辨率相关的预计算常量
        float a_view = pp_in_view.z * pp_in_view.z * TileAreaConstant;
        // 核半径 = sqrt(面积 / (pi * 光子数))
        // 物理含义：如果光子均匀分布在图块面积内，
        // 每个光子占据的圆形区域的半径
        r = sqrt(a_view / (PI * n_p));
    }

    // Equation 24.6: 动态核缩放
    // 将半径限制在 [DYNAMIC_KERNEL_SCALE_MIN, DYNAMIC_KERNEL_SCALE_MAX]
    // 然后乘以图块归一化因子 n_tile
    float s_d = clamp(r, DYNAMIC_KERNEL_SCALE_MIN,
        DYNAMIC_KERNEL_SCALE_MAX) * n_tile;

    // Equation 24.2: 光线长度衰减
    // 光线越长，间接光照越分散，核应该越小
    // 限制范围为 [0.1, 1.0]，避免完全消失或过大
    float s_l = clamp(ray_length / MAX_RAY_LENGTH, .1f, 1.0f);

    // 最终均匀缩放因子 = 密度驱动缩放 * 光线长度衰减
    return s_d * s_l;
}
```

## 关键算法深入分析

### 密度估计的直觉理解

想象一个屏幕图块中有 $n_p$ 个光子，图块的视图空间面积为 $A$。如果将面积均匀分配给每个光子，每个光子的"领地"面积为 $A/n_p$。对应的圆形半径为：

$$r = \sqrt{\frac{A/n_p}{\pi}} = \sqrt{\frac{A}{\pi \cdot n_p}}$$

这确保了相邻光子的核不会过度重叠（密集区域）或留下空隙（稀疏区域）。

### 视图空间面积的计算

$$A_{\text{view}} = z^2 \cdot C_{\text{tile}}$$

其中 $C_{\text{tile}}$ 与图块的角大小有关：

$$C_{\text{tile}} = \Delta\theta_x \cdot \Delta\theta_y$$

$\Delta\theta$ 为图块在屏幕上对应的角度范围。在透视投影下，相同的屏幕图块在远处对应更大的世界空间面积，因此与 $z^2$ 成正比。

### 光线长度衰减的物理意义

光子在多次弹射后传播的距离越远，其贡献的空间范围越不确定。缩小核可以减少远距离光子的"涂抹"范围，降低错误贡献的风险。$s_l$ 的下限为 0.1，确保即使最远的光子仍有一定的核大小。

### 两个缩放因子的相互作用

$$s = s_d \cdot s_l$$

- 在光子密集、弹射短的区域：$s$ 最小（紧凑精确的核）
- 在光子稀疏、弹射长的区域：$s_d$ 大但 $s_l$ 小，部分抵消
- 在光子稀疏、弹射短的区域：$s$ 最大（覆盖更广的核）

## 输入与输出

| 参数 | 类型 | 说明 |
|------|------|------|
| `pp_in_view` | `float3` | 光子在视图空间中的位置 |
| `ray_length` | `float` | 光子的光线长度 |
| **返回值** | `float` | 均匀缩放因子 $s = s_d \cdot s_l$ |

### 依赖的外部变量/常量

| 变量 | 说明 |
|------|------|
| `layers` | 有效层数（可能与级联 RSM 相关） |
| `TileAreaConstant` | 图块角面积常量 |
| `PI` | 圆周率 $\pi$ |
| `n_tile` | 图块归一化因子 |
| `DYNAMIC_KERNEL_SCALE_MIN/MAX` | 核半径的限幅范围 |
| `MAX_RAY_LENGTH` | 最大光线长度 |

## 与其他文件的关系

- **ValidateAndAddPhoton.hlsl**: 更新 `DensityEstimationBuffer`，本文件通过 `load_number_of_photons_in_tile` 读取
- **KernelModificationForVertexPosition.hlsl**: 调用本函数获取均匀缩放因子，然后进一步进行方向性变形
- **SplattingShaders.hlsl**: 最终使用缩放后的核几何体进行光栅化

## 在光线追踪管线中的位置

本函数在光子喷溅阶段的**顶点着色器**中被间接调用（通过 `kernel_modification_for_vertex_position`）：

```
[光子追踪] → 存储光子 + 密度估计
                │
                ▼
[光子喷溅 VS]
    │
    ├── uniform_scaling() ← 本文件
    │   ├── 读取图块光子密度
    │   └── 计算 s_d * s_l
    │
    └── kernel_modification_for_vertex_position()
        └── 使用 uniform_scaling 结果进行核变形
```

## 技术要点与注意事项

1. **图块对齐**：`load_number_of_photons_in_tile` 需要将视图空间位置正确映射到图块索引，与 `validate_and_add_photon` 中的映射一致
2. **layers 变量**：当 `layers <= 0` 时使用默认半径 0.1，可能表示没有有效的 RSM 级联
3. **零光子处理**：当 `n_p = 0` 时，`sqrt(a_view / (PI * 0))` 会产生无穷大，需要由 clamp 限幅处理
4. **性能考量**：`load_number_of_photons_in_tile` 读取的是上一帧（或当前帧早期阶段）的光子计数，存在一帧延迟
5. **各向同性假设**：本函数计算的是均匀（各向同性）的缩放，方向性调整由 `KernelModificationForVertexPosition.hlsl` 处理

## 扩展阅读

- Silverman, B.W. *Density Estimation for Statistics and Data Analysis*, 1986 (核密度估计理论)
- Ray Tracing Gems, Chapter 24, Section 24.4: Adaptive Kernel Sizing
- Jensen, H.W. *Realistic Image Synthesis Using Photon Mapping*, Chapter 6: Radiance Estimate
