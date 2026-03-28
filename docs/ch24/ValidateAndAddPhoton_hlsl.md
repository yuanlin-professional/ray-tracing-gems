# ValidateAndAddPhoton.hlsl 技术文档

## 文件概述

`ValidateAndAddPhoton.hlsl` 实现了实时光子映射系统中的**光子验证与存储** (Photon Validation and Storage) 逻辑。在光子命中表面后，该函数检查命中点是否位于相机视锥 (camera frustum) 内且法线朝向相机，然后将通过验证的光子写入光子缓冲区 (photon buffer)，并更新基于图块 (tile-based) 的密度估计缓冲区 (density estimation buffer)。

**源文件路径**: `Ch_24_Real-Time_Global_Illumination_with_Photon_Mapping/ValidateAndAddPhoton.hlsl`

## 算法与数学背景

### 视锥裁剪 (Frustum Culling)

只有落在相机视锥内的光子才会被存储，因为只有这些光子对当前帧的渲染有贡献。视锥外的光子即使计算了也不会影响最终图像。

### 法线方向检查

`is_normal_direction_to_camera(normal)` 确保存储的光子位于面向相机的表面上。背对相机的表面上的光子在光子喷溅 (splatting) 阶段不可见，存储它们是浪费。

### 基于图块的密度估计 (Tile-Based Density Estimation)

屏幕被划分为固定大小的图块 (tile)，每个图块维护一个光子计数器。该计数在光子喷溅阶段用于自适应核缩放 (adaptive kernel scaling)，使得光子密集的区域使用较小的核，光子稀疏的区域使用较大的核。

### 原子操作 (Atomic Operations)

由于多个光子追踪线程可能同时写入同一个图块计数器或全局光子索引，需要使用原子加法 `InterlockedAdd` 确保线程安全。

## 代码结构概览

```
验证光子 → [视锥内 & 法线朝向相机]
  → 计算图块索引
  → 原子分配光子缓冲区索引
  → 打包并存储光子数据
  → 原子更新图块密度计数
```

## 完整源码与逐行注释

```hlsl
void validate_and_add_photon(Surface_attributes surface,
    float3 position_in_world, float3 power,
    float3 incoming_direction, float t)
{
    // 验证条件 1：光子命中点是否在相机视锥内
    // 验证条件 2：表面法线是否朝向相机（背面剔除）
    if (is_in_camera_frustum(position) &&
        is_normal_direction_to_camera(surface.normal))
    {
        // 将世界空间位置映射到屏幕空间图块索引
        // 展平为一维索引用于缓冲区寻址
        uint tile_index =
            get_tile_index_in_flattened_buffer(position_in_world);

        uint photon_index;
        // 使用原子加法在间接绘制参数缓冲区 (DrawArgumentBuffer) 中
        // 分配一个新的光子索引
        // 偏移 4 字节处是实例计数 (instance count)，
        // 每添加一个光子就递增 1
        // photon_index 接收递增前的值作为新光子的唯一索引
        DrawArgumentBuffer.InterlockedAdd(4, 1, photon_index);

        // 将光子数据打包并存储到光子缓冲区的对应位置
        // 存储内容包括：世界空间位置、功率、法线、入射方向、光线长度
        add_photon_to_buffer(position_in_world, power, surface.normal,
            power, incoming_direction, photon_index, t);

        // 更新基于图块的光子密度估计
        // 在对应图块的计数器上原子加 1
        // tile_i * 4：每个计数器占 4 字节 (uint)
        DensityEstimationBuffer.InterlockedAdd(tile_i * 4, 1);
    }
}
```

## 关键算法深入分析

### 间接绘制 (Indirect Draw) 与光子索引分配

`DrawArgumentBuffer` 是一个间接绘制参数缓冲区 (Indirect Draw Argument Buffer)，其布局通常为：

```
偏移 0: VertexCountPerInstance (顶点数，固定)
偏移 4: InstanceCount (实例数 = 光子数，通过原子加法递增)
偏移 8: StartVertexLocation
偏移 12: StartInstanceLocation
```

通过 `InterlockedAdd(4, 1, photon_index)`：
- 在偏移 4（InstanceCount）处原子加 1
- 返回加之前的值作为 `photon_index`
- 这样每个光子获得唯一的连续索引

在后续的光子喷溅阶段，GPU 使用 `DrawInstancedIndirect` 读取该缓冲区，自动绘制正确数量的光子实例。

### 双重存储的 power 参数

注意 `add_photon_to_buffer` 中 `power` 参数出现了两次：
```hlsl
add_photon_to_buffer(position_in_world, power, surface.normal,
    power, incoming_direction, photon_index, t);
```
这可能是原始代码的简化，实际中两个 power 参数可能分别代表不同的功率（如原始功率和衰减后功率）。

### 图块密度估计的用途

存储在 `DensityEstimationBuffer` 中的每图块光子计数，在 `UniformScaling.hlsl` 中通过 `load_number_of_photons_in_tile()` 读取，用于计算自适应核半径：

$$r = \sqrt{\frac{A_{\text{view}}}{\pi \cdot n_p}}$$

其中 $n_p$ 为图块内的光子数量。

## 输入与输出

| 参数 | 类型 | 说明 |
|------|------|------|
| `surface` | `Surface_attributes` | 命中表面属性（法线等） |
| `position_in_world` | `float3` | 光子命中点的世界空间位置 |
| `power` | `float3` | 光子功率 (RGB) |
| `incoming_direction` | `float3` | 光子入射方向 |
| `t` | `float` | 光线长度 |
| **输出** | `DrawArgumentBuffer` | 更新间接绘制参数（光子计数） |
| **输出** | 光子缓冲区 | 存储光子数据 |
| **输出** | `DensityEstimationBuffer` | 更新图块光子密度计数 |

## 与其他文件的关系

- **ClosestHit.hlsl**: 调用本函数存储光子
- **SplattingShaders.hlsl**: 读取光子缓冲区（由本函数写入）和间接绘制参数
- **UniformScaling.hlsl**: 读取密度估计缓冲区（由本函数更新）

## 在光线追踪管线中的位置

本函数在 Closest Hit Shader 内部被调用，属于光子追踪阶段的末端操作：

```
[Closest Hit]
    │
    ├── russian_roulette() ──> 决定弹射
    ├── repack_payload()   ──> 更新 Payload
    └── validate_and_add_photon() ← 本文件
         ├── DrawArgumentBuffer.InterlockedAdd ──> 分配光子索引
         ├── add_photon_to_buffer()             ──> 存储光子数据
         └── DensityEstimationBuffer.InterlockedAdd ──> 更新密度
```

## 技术要点与注意事项

1. **变量名不一致**：`position` vs `position_in_world`，`tile_index` vs `tile_i` 可能是原文简化导致的不一致
2. **原子操作性能**：大量线程同时执行 `InterlockedAdd` 会导致争用 (contention)，特别是在光子密集的图块中
3. **缓冲区溢出保护**：代码未显示光子缓冲区大小检查，实际实现中需要确保 `photon_index` 不超过缓冲区容量
4. **背面光子的丢弃**：仅存储面向相机的光子可能在某些场景中丢失有用的间接光照（如从背面反射到可见面的光）
5. **图块大小选择**：图块大小需要在密度估计精度和存储开销之间平衡

## 扩展阅读

- Ray Tracing Gems, Chapter 24, Section 24.4: Photon Storage and Density Estimation
- Stürzlinger, W. & Bastos, R. *Interactive Rendering of Globally Illuminated Glossy Scenes*, Eurographics Rendering Workshop, 1997
- Microsoft DirectX 12: Indirect Drawing and ExecuteIndirect
