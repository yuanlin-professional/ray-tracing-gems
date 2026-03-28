# KernelModificationForVertexPosition.hlsl 技术文档

## 文件概述

`KernelModificationForVertexPosition.hlsl` 实现了实时光子映射系统中的**核形状修改** (Kernel Shape Modification) 算法。该函数将光子喷溅的圆形核 (circular kernel) 变形为**椭圆形核** (elliptical kernel)，使其沿光照方向拉伸。这种变形可以更准确地反映光子在表面上的空间分布，特别是在掠射角 (grazing angle) 光照条件下。

**源文件路径**: `Ch_24_Real-Time_Global_Illumination_with_Photon_Mapping/KernelModificationForVertexPosition.hlsl`

## 算法与数学背景

### 椭圆核变形的动机

当光线以掠射角到达表面时，其照射的面积会沿入射方向拉伸。一个正圆形的核无法正确表示这种拉伸分布，导致在掠射角处出现不自然的"圆形"伪影。椭圆核通过沿光照方向拉伸来匹配实际的照射形状。

### 坐标系分解

算法将核几何体的顶点位置分解为三个正交分量：
1. **法线方向** ($\hat{n}$)：表面法线方向的分量
2. **光照方向** ($\hat{u}$)：光照方向在表面切平面上的投影
3. **切线方向** ($\hat{t}$)：与法线和光照方向都垂直的方向

### 光照方向缩放 (Equation 24.7)

沿光照方向的缩放因子为：

$$s_{\text{light}} = \min\left(\frac{1}{\cos\theta}, C_{\max}\right)$$

其中 $\theta$ 为光照方向与法线的夹角。当 $\theta \to 90°$ 时 $\cos\theta \to 0$，缩放趋于无穷大，因此需要 $C_{\max}$ 作为上限。

### 核顶点变换 (Equation 24.8)

对于核几何体的每个顶点 $\mathbf{v}$：

$$\mathbf{v}_{\text{scaled}} = (\mathbf{v} \cdot \hat{u}) \cdot \hat{u} \cdot s_{\text{light}} \cdot s_{\text{uniform}} + (\mathbf{v} - (\mathbf{v} \cdot \hat{u}) \hat{u} - (\mathbf{v} \cdot \hat{n}) \hat{n}) \cdot s_{\text{uniform}} + (\mathbf{v} \cdot \hat{n}) \hat{n} \cdot K_{\text{compress}}$$

其中：
- 第一项：光照方向分量，同时受光照缩放和均匀缩放
- 第二项：切线方向分量，仅受均匀缩放
- 第三项：法线方向分量，受核压缩因子控制（控制核厚度）

### 椭圆面积

变形后的椭圆面积为：

$$A = \pi \cdot s_{\text{uniform}}^2 \cdot s_{\text{light}}$$

## 代码结构概览

```
计算均匀缩放 → 分解顶点到 (n, u, t) 坐标系
  → 计算光照方向缩放 (Eq 24.7) → 分别缩放各分量 (Eq 24.8)
  → 组合顶点位置 → 计算椭圆面积
```

## 完整源码与逐行注释

```hlsl
kernel_output kernel_modification_for_vertex_position(float3 vertex,
    float3 n, float3 light, float3 pp_in_view, float ray_length)
{
    kernel_output o;
    // 计算均匀缩放因子（基于密度估计和光线长度）
    float scaling_uniform = uniform_scaling(pp_in_view, ray_length);

    // 归一化光照方向向量
    float3 l = normalize(light);
    // 计算顶点在法线方向上的投影长度 cos(alpha) = dot(n, vertex)
    // 注意：这里应为标量，但代码使用了 float3
    float3 cos_alpha = dot(n, vertex);
    // 顶点在法线方向上的投影向量
    float3 projected_v_to_n = cos_alpha * n;
    // 光照方向与法线的余弦值（限制为非负：饱和/saturate）
    float cos_theta = saturate(dot(n, l));
    // 光照方向在法线上的投影向量
    float3 projected_l_to_n = cos_theta * n;

    // 计算光照方向在表面切平面上的投影（归一化）
    // u = normalize(l - (l·n)n) = 光照方向的切平面分量的单位向量
    float3 u = normalize(l - projected_l_to_n);

    // Equation 24.7: 光照方向缩放因子
    // 1/cos(theta): 掠射角时拉伸更大
    // MAX_SCALING_CONSTANT: 上限，防止极端掠射角时缩放过大
    o.light_shaping_scale = min(1.0f/cos_theta, MAX_SCALING_CONSTANT);

    // 将顶点分解到三个正交方向：
    // 1. u 方向（光照切平面投影方向）的分量
    float3 projected_v_to_u = dot(u, vertex) * u;
    // 2. t 方向（与 u 和 n 都垂直的切线方向）的分量
    //    = vertex - u 方向分量 - n 方向分量
    float3 projected_v_to_t = vertex - projected_v_to_u;
    //    减去残留的法线分量，确保完全在切平面内
    projected_v_to_t -= dot(projected_v_to_t, n) * n;

    // Equation 24.8: 分别缩放各方向分量
    // u 方向：同时受光照缩放和均匀缩放
    // (光照缩放使核沿光照方向拉伸)
    float3 scaled_u = projected_v_to_u * light_shaping_scale *
        scaling_Uniform;
    // t 方向：仅受均匀缩放（不拉伸）
    float3 scaled_t = projected_v_to_t * scaling_uniform;
    // 最终顶点位置 = u 分量 + t 分量 + 法线分量（压缩）
    // KernelCompress: 核在法线方向的压缩因子（控制核"厚度"）
    o.vertex_position = scaled_u + scaled_t +
        (KernelCompress * projected_v_to_n);

    // 椭圆面积 = pi * r_minor * r_major
    // = pi * scaling_uniform^2 * light_shaping_scale
    // scaling_uniform^2: 基础圆面积的缩放
    // * light_shaping_scale: 沿光照方向的拉伸
    o.ellipse_area = PI * o.scaling_uniform * o.scaling_uniform *
        o.light_shaping_scale;

    return o;
}
```

## 关键算法深入分析

### 坐标系构建

算法构建了一个以表面法线 $\hat{n}$ 为基础的正交坐标系 $(\hat{u}, \hat{t}, \hat{n})$：

```
        n (法线)
        │
        │
        │──────── u (光照切平面投影方向)
       /
      /
     t (副切线方向)
```

- $\hat{u}$ = 光照方向在切平面上的投影的单位向量
- $\hat{t}$ = 与 $\hat{u}$ 和 $\hat{n}$ 都垂直的方向

### 掠射角拉伸的几何直觉

想象一个手电筒从侧面照射桌面：
- 正面照射 ($\theta \approx 0$)：光斑接近圆形 → $s_{\text{light}} \approx 1$
- 侧面照射 ($\theta \approx 75°$)：光斑沿照射方向拉伸 → $s_{\text{light}} \approx 3.86$
- 几乎平行 ($\theta \to 90°$)：光斑极度拉伸 → $s_{\text{light}} \to C_{\max}$

### 面积计算的推导

原始核为半径 $r = s_{\text{uniform}}$ 的圆，面积为 $\pi r^2$。椭圆化后：
- 短轴 $= s_{\text{uniform}}$（切线方向不变）
- 长轴 $= s_{\text{uniform}} \cdot s_{\text{light}}$（光照方向拉伸）

椭圆面积 $= \pi \cdot a \cdot b = \pi \cdot s_{\text{uniform}}^2 \cdot s_{\text{light}}$

该面积在 VS 中用于归一化功率：$E = \Phi / A$。

### 法线方向的核压缩

`KernelCompress * projected_v_to_n` 控制核在法线方向上的"厚度"。这在深度测试中很重要：
- 厚度过大：可能与其他表面错误重叠
- 厚度过小：核可能无法覆盖到 G-Buffer 表面（被深度裁剪掉）

## 输入与输出

| 参数 | 类型 | 说明 |
|------|------|------|
| `vertex` | `float3` | 核几何体的顶点位置（模型空间） |
| `n` | `float3` | 表面法线（世界空间） |
| `light` | `float3` | 光照方向（世界空间，指向光源） |
| `pp_in_view` | `float3` | 光子在视图空间中的位置 |
| `ray_length` | `float` | 光线长度 |

| 输出字段 | 类型 | 说明 |
|----------|------|------|
| `vertex_position` | `float3` | 变形后的顶点偏移（世界空间） |
| `light_shaping_scale` | `float` | 光照方向缩放因子 |
| `ellipse_area` | `float` | 椭圆核面积 |

## 与其他文件的关系

- **UniformScaling.hlsl**: 本函数内部调用 `uniform_scaling()` 获取基础缩放因子
- **SplattingShaders.hlsl**: VS 调用本函数，使用返回的 `vertex_position` 和 `ellipse_area`

## 在光线追踪管线中的位置

本函数在光子喷溅阶段的顶点着色器中被调用：

```
[光子喷溅 VS]
    │
    ├── unpack_photon()  → 获取光子数据
    │
    └── kernel_modification_for_vertex_position() ← 本文件
         │
         ├── uniform_scaling()      → 基础缩放
         │
         ├── 坐标系分解 (n, u, t)
         │
         ├── light_shaping_scale    → Eq 24.7
         │
         └── vertex_position        → Eq 24.8
              │
              ▼
         投影到裁剪空间
```

## 技术要点与注意事项

1. **变量名不一致**：代码中混用 `scaling_uniform`、`scaling_Uniform`（大写 U）和 `o.scaling_uniform`，实际应为同一变量
2. **cos_theta = 0 的退化情况**：当光照方向与法线完全垂直时，$\cos\theta = 0$，此时 $1/\cos\theta = \infty$。`saturate` 确保 $\cos\theta \geq 0$，但 $= 0$ 仍会导致除零，需要由 `MAX_SCALING_CONSTANT` 保护
3. **cos_alpha 的类型**：代码中 `float3 cos_alpha = dot(n, vertex)` 使用了 `float3` 但 `dot` 返回标量，这可能依赖于 HLSL 的隐式广播 (broadcast)
4. **u 方向退化**：当光照方向恰好平行于法线时（$\theta = 0$），`l - projected_l_to_n` 为零向量，`normalize(0)` 未定义。实际实现需要特殊处理
5. **light_shaping_scale 未用 o. 前缀**：`scaled_u` 使用了 `light_shaping_scale` 而非 `o.light_shaping_scale`，可能是笔误

## 扩展阅读

- Schregle, R. *Bias Compensation for Photon Maps*, Computer Graphics Forum, 2003 (椭圆核的理论基础)
- Jensen, H.W. *Realistic Image Synthesis Using Photon Mapping*, Chapter 6: Kernel Functions
- Ray Tracing Gems, Chapter 24, Section 24.4: Kernel Shape Modification
