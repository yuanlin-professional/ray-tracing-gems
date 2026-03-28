# SplattingShaders.hlsl 技术文档

## 文件概述

`SplattingShaders.hlsl` 实现了实时光子映射系统中的**光子喷溅着色器** (Photon Splatting Shaders)，包含顶点着色器 (Vertex Shader, VS) 和像素着色器 (Pixel Shader, PS)。光子喷溅是将每个光子的光照贡献"涂抹"到屏幕上附近像素的过程，通过光栅化管线 (rasterization pipeline) 实现，避免了传统光子映射中昂贵的 k-最近邻 (k-NN) 搜索。

**源文件路径**: `Ch_24_Real-Time_Global_Illumination_with_Photon_Mapping/SplattingShaders.hlsl`

## 算法与数学背景

### 光子喷溅 (Photon Splatting)

传统光子映射使用 kd-tree 进行最近邻搜索来收集光子。光子喷溅则反转了这一过程：不是从着色点搜索附近光子，而是将每个光子作为一个几何体（通常是圆盘或球体）渲染到屏幕上，利用 GPU 光栅化硬件进行高效的空间分配。

对于每个光子，其对表面点的辐照度贡献为：

$$E(x) = \frac{\Phi}{A_{\text{kernel}}}$$

其中 $\Phi$ 为光子功率 (power)，$A_{\text{kernel}}$ 为核 (kernel) 的面积（此处为椭圆面积）。

### 深度测试与裁剪

光子喷溅需要验证光子与 G-Buffer 表面的深度一致性。如果光子的深度与 G-Buffer 深度差异过大，说明光子不在该表面上，应被裁剪 (clip)。

### 功率加权方向

为了在喷溅过程中保留光照方向信息（用于后续的方向性滤波或法线验证），像素着色器输出功率加权的方向向量。

## 代码结构概览

```
顶点着色器 (VS):
  读取光子数据 → 核形状变形 → 投影到裁剪空间 → 计算功率密度

像素着色器 (PS):
  深度测试 → 裁剪过远光子 → 输出功率和加权方向
```

## 完整源码与逐行注释

```hlsl
// ========================
// 顶点着色器 (Vertex Shader)
// ========================
void VS(
    float3 Position : SV_Position,   // 核几何体的顶点位置（模型空间）
    uint instanceID : SV_InstanceID, // 实例 ID = 光子索引
    out vs_to_ps Output)             // 输出到像素着色器
{
    // 从光子缓冲区读取并解包光子数据
    // instanceID 对应 validate_and_add_photon 中分配的 photon_index
    unpacked_photon up = unpack_photon(PhotonBuffer[instanceID]);

    // 光子在世界空间的位置
    float3 photon_position = up.position;
    // 将光子位置变换到视图空间 (view space)
    float3 photon_position_in_view = mul(WorldToViewMatrix,
    float4(photon_position, 1)).xyz;

    // 计算核形状变形：基于光子位置、法线、光照方向、视图空间位置、光线长度
    // 返回变形后的顶点位置和椭圆面积
    // -up.direction: 取反因为 direction 存储的是入射方向，核变形需要出射方向
    kernel_output o = kernel_modification_for_vertex_position(Position,
    up.normal, -up.direction, photon_position_in_view, up.ray_length);

    // 将变形后的顶点位置加到光子世界位置上
    // pp 应为 photon_position 的简写
    float3 p = pp + o.vertex_position;

    // 变换到裁剪空间 (clip space) 用于光栅化
    Output.Position = mul(WorldToViewClipMatrix, float4(p, 1));
    // 计算功率密度 = 光子功率 / 椭圆核面积
    // 这是辐照度的近似：E = Phi / A
    Output.Power = up.power / o.ellipse_area;
    // 传递光照方向（取反为出射方向）
    Output.Direction = -up.direction;
}

// ========================
// 像素着色器 (Pixel Shader)
// ========================
// earlydepthstencil: 启用早期深度模板测试，
// 提前剔除被遮挡的片元以提升性能
[earlydepthstencil]
void PS(
vs_to_ps Input,
// 输出 0：RGB 功率 + 加权方向的 X 分量
out float4 OutputColorXYZAndDirectionX : SV_Target,
// 输出 1：加权方向的 YZ 分量
out float2 OutputDirectionYZ : SV_Target1)
{
    // 从 G-Buffer 深度纹理读取当前像素的深度
    float depth = DepthTexture[Input.Position.xy];
    // 将 G-Buffer 深度转换为线性深度
    float gbuffer_linear_depth = LinearDepth(ViewConstants, depth);
    // 将光子核片元的深度转换为线性深度
    float kernel_linear_depth = LinearDepth(ViewConstants,
        Input.Position.z);
    // 计算 G-Buffer 表面深度与光子核深度的差异
    float d = abs(gbuffer_linear_depth - kernel_linear_depth);

    // 深度裁剪：如果深度差异超过阈值则丢弃此片元
    // KernelCompress: 核压缩因子（控制核的法线方向厚度）
    // MAX_DEPTH: 最大允许深度差
    // clip(x): x < 0 时丢弃片元
    clip(d > (KernelCompress * MAX_DEPTH) ? -1 : 1);

    // 光子功率（辐照度密度，已在 VS 中除以椭圆面积）
    float3 power = Input.Power;
    // 计算总功率（RGB 分量之和），用作方向权重
    float total_power = dot(power.xyz, float3(1.0f, 1.0f, 1.0f));
    // 功率加权方向：total_power * Direction
    // 用于后续判断光子贡献的主方向
    float3 weighted_direction = total_power * Input.Direction;

    // 输出 MRT (Multiple Render Targets):
    // Target0: (power.r, power.g, power.b, weighted_direction.x)
    OutputColorXYZAndDirectionX = float4(power, weighted_direction.x);
    // Target1: (weighted_direction.y, weighted_direction.z)
    OutputDirectionYZ = weighted_direction.yz;
}
```

## 关键算法深入分析

### 实例化绘制 (Instanced Drawing)

光子喷溅使用 GPU 实例化绘制 (Instanced Draw)：
- **基础几何体**：一个小圆盘/多边形（`Position` 参数）
- **实例数据**：每个光子一个实例（通过 `instanceID` 从 `PhotonBuffer` 索引）

`DrawInstancedIndirect` 从 `DrawArgumentBuffer` 读取实例数量（由 `validate_and_add_photon` 中的原子操作设置），无需 CPU 回读。

### 深度裁剪的原理

```
     相机
      │
      │    G-Buffer 表面
      │  ────────────────
      │         ↑ d (深度差)
      │  - - - - - - - -  光子核
      │
```

当光子核的深度与 G-Buffer 表面深度差异超过阈值时，说明光子位于不同的表面上（如墙壁后面），应丢弃。`KernelCompress * MAX_DEPTH` 定义了允许的最大深度差。

### MRT 输出布局

使用两个渲染目标 (Multiple Render Targets, MRT) 输出 5 个通道：

| Target | 通道 | 内容 |
|--------|------|------|
| SV_Target0 | RGBA | (R功率, G功率, B功率, 加权方向X) |
| SV_Target1 | RG | (加权方向Y, 加权方向Z) |

这种紧凑的布局减少了带宽使用。功率和方向的累积使用加法混合 (additive blending) 模式。

### 加法混合 (Additive Blending)

光子喷溅的渲染目标使用加法混合模式 (BlendState = ADD)，使得多个光子的贡献在同一像素上自动累加：

$$E_{\text{pixel}} = \sum_{i} \frac{\Phi_i}{A_i}$$

## 输入与输出

### 顶点着色器输入
| 参数 | 类型 | 说明 |
|------|------|------|
| `Position` | `float3` | 核几何体的模型空间顶点 |
| `instanceID` | `uint` | 光子索引 |
| `PhotonBuffer` | 结构化缓冲区 | 光子数据 |

### 像素着色器输出
| 输出 | 类型 | 说明 |
|------|------|------|
| `OutputColorXYZAndDirectionX` | `float4` | RGB 功率 + 方向X |
| `OutputDirectionYZ` | `float2` | 方向YZ |

## 与其他文件的关系

- **ValidateAndAddPhoton.hlsl**: 写入 `PhotonBuffer` 和 `DrawArgumentBuffer`（本文件读取）
- **UniformScaling.hlsl**: 在 `kernel_modification_for_vertex_position` 内部调用，计算核大小
- **KernelModificationForVertexPosition.hlsl**: 直接被 VS 调用，计算核形状变形

## 在光线追踪管线中的位置

本文件运行在**光栅化管线** (rasterization pipeline) 中，而非光线追踪管线。它是光子映射系统第二阶段（密度估计/渲染）的核心：

```
[阶段1: 光子追踪 (DXR)]
    │
    ▼ 光子缓冲区

[阶段2: 光子喷溅 (光栅化)] ← 本文件
    │
    ├── VS: 读取光子 → 核变形 → 投影
    │
    ├── 光栅化: 生成片元
    │
    └── PS: 深度测试 → 输出功率和方向
         │
         ▼ 加法混合

[光子贡献纹理] → 与直接光照合成
```

## 技术要点与注意事项

1. **pp 变量**：`float3 p = pp + o.vertex_position` 中的 `pp` 应为 `photon_position` 的简写或别名
2. **earlydepthstencil**：PS 使用 `[earlydepthstencil]` 属性启用保守的早期深度测试，但由于内部使用 `clip()` 可能导致该优化部分失效
3. **加法混合的限制**：加法混合不支持负值，因此功率值必须始终非负
4. **方向编码**：功率加权方向可以在后处理中归一化（除以总功率）以恢复主方向
5. **带宽考量**：MRT 输出需要足够的渲染目标格式精度（通常为 R16G16B16A16_FLOAT）

## 扩展阅读

- Lavignotte, F. & Paulin, M. *Scalable Photon Splatting for Global Illumination*, 2003
- McGuire, M. & Luebke, D. *Hardware-Accelerated Global Illumination by Image Space Photon Mapping*, 2009
- Ray Tracing Gems, Chapter 24, Section 24.5: Photon Splatting
