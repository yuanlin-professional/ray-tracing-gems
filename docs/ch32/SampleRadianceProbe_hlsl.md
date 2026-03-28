# SampleRadianceProbe.hlsl 技术文档

## 文件概述

`SampleRadianceProbe.hlsl` 实现了基于辐射度缓存的镜面反射系统中**单个辐射度探针 (Radiance Probe) 的采样与验证**逻辑。该函数判断反射命中点是否位于探针的覆盖范围内，验证探针是否有足够的分辨率来表示该命中点，并通过深度比较排除被遮挡的情况，最终返回探针记录的辐射度。

**源文件路径**: `Ch_32_Accurate_Real-Time_Specular_Reflections_with_Radiance_Caching/SampleRadianceProbe.hlsl`

## 算法与数学背景

### 辐射度探针 (Radiance Probe)

辐射度探针是放置在场景中特定位置的**立方体贴图** (Cube Map)，记录从该位置向所有方向观察到的辐射度和深度。每个探针包含：
- **辐射度立方体贴图** (`cube.Radiance`)：6 个面，存储 RGB 颜色
- **深度立方体贴图** (`cube.Depth`)：6 个面，存储到最近表面的深度
- **位置** (`cube.Position`)：探针在世界空间中的位置
- **覆盖半径** (`cube.RadiusSqr`)：探针有效覆盖范围
- **分辨率** (`cube.Resolution`)：立方体贴图的面分辨率

### 范围检查 (Range Check)

第一步检查命中点是否在探针的覆盖球体内：

$$\|\mathbf{p}_{\text{hit}} - \mathbf{p}_{\text{probe}}\|^2 \leq r_{\text{probe}}^2$$

### 分辨率适配检查 (Resolution Adequacy Check)

即使命中点在探针范围内，如果探针的分辨率不足以准确表示该命中点（命中点太近或处于探针边缘），采样结果会不准确。

函数 `ProbabilityToSampleSameTexel` 估计从当前查找方向采样时，相邻像素映射到**同一个立方体贴图纹素** (texel) 的概率：

$$p = P(\text{同一纹素})$$

如果 $p < \text{ResolutionThreshold}$，说明分辨率足够精细（不同的查找方向映射到不同的纹素），可以安全采样。

### 遮挡检查 (Occlusion Check)

探针看到的表面不一定是光线命中的表面（可能被其他物体遮挡）。通过比较命中点到探针的距离与探针深度图中记录的距离来验证：

$$|z_{\text{hit}} - z_{\text{probe}}| \cdot \frac{1}{\cos\alpha} < T \cdot \frac{z_{\text{hit}}}{R}$$

其中：
- $z_{\text{hit}}$：命中点在探针立方体面上的深度（沿主轴方向的分量）
- $z_{\text{probe}}$：探针深度图中记录的深度
- $\cos\alpha$：探针观察方向与立方体面法线的夹角余弦
- $T$：遮挡阈值因子 (`OcclusionThresholdFactor`)
- $R$：探针分辨率

右侧阈值与 $z_{\text{hit}} / R$ 成正比，意味着远处允许更大的深度误差（因为分辨率对应的空间范围更大）。

## 代码结构概览

```
范围检查 (距离 <= 半径)
  → 分辨率检查 (同纹素概率 < 阈值)
    → 计算采样方向和深度
      → 遮挡检查 (深度差 < 阈值)
        → 采样辐射度 → 返回 true
  → 任一检查失败 → 返回 false
```

## 完整源码与逐行注释

```hlsl
bool SampleRadianceProbe(uint probeIndex,
                         float3 hitPos,
                         out float3 radiance)
{
  // 加载探针数据（位置、半径、分辨率、立方体贴图引用）
  CubeMap cube = LoadCube(probeIndex);

  // 计算命中点相对于探针位置的偏移向量
  float3 fromCube = hitPos - cube.Position;
  // 计算距离的平方（避免开方，用于范围检查）
  float distSqr = dot(fromCube, fromCube);

  // 范围检查：命中点是否在探针的覆盖球体内
  if (distSqr <= cube.RadiusSqr) {

    // 确定命中点对应的立方体贴图面
    // MaxDir 返回绝对值最大的轴方向：(1,0,0), (0,1,0) 或 (0,0,1)
    float3 cubeFace = MaxDir(fromCube); // (1,0,0), (0,1,0) or (0,0,1)

    // 计算命中点在该立方体面法线方向上的深度
    // hitZInCube = 命中点到探针中心在主轴方向上的距离
    float hitZInCube = dot(cubeFace, fromCube);

    // 分辨率检查：估计从该方向采样时采到相同纹素的概率
    // p 越小说明分辨率越充足（不同方向映射到不同纹素）
    float p = ProbabilityToSampleSameTexel(cube, hitZInCube, hitPos);

    // 仅当分辨率足够时继续
    if (p < ResolutionThreshold) {

      // 计算命中点到探针中心的实际距离
      float distanceFromCube = sqrt(distSqr);
      // 归一化方向向量，用于采样立方体贴图
      float3 sampleDir = fromCube / distanceFromCube;

      // 从探针的深度立方体贴图采样该方向的深度
      // SampleLevel(..., 0): 使用 mip level 0（最高分辨率）
      // ZInCube: 将原始深度值转换为线性深度
      float zSeenByCube =
            ZInCube(cube.Depth.SampleLevel(Sampler, sampleDir, 0));

      // 计算 1/cos(观察角) 修正因子
      // 观察角 = 采样方向与立方体面法线的夹角
      // distanceFromCube / hitZInCube = 斜距 / 轴向距离 = 1/cos(alpha)
      float cosCubeViewAngleRcp = distanceFromCube / hitZInCube;

      // 遮挡检查：比较命中点深度与探针深度图深度
      // |hitZ - probeZ| * (1/cos) 将深度差转换为沿视线方向的实际距离
      float dist = abs(hitZInCube - zSeenByCube) * cosCubeViewAngleRcp;

      // 允许的深度误差阈值：与 hitZ/分辨率 成正比
      // 远处的点允许更大的误差（因为单个纹素对应更大的空间范围）
      if (dist <
            OcclusionThresholdFactor * hitZInCube / cube.Resolution) {

        // 所有检查通过：从探针的辐射度立方体贴图采样
        radiance = cube.Radiance.SampleLevel(Sampler, sampleDir, 0);
        return true;  // 成功采样
      }
    }
  }
  // 任一检查未通过：探针无法为该命中点提供有效辐射度
  return false;
}
```

## 关键算法深入分析

### 三层验证机制

算法使用三层嵌套的检查来确保采样质量：

```
┌─────────────────────────────────────────────┐
│ 第1层：范围检查 (distSqr <= RadiusSqr)      │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │ 第2层：分辨率检查 (p < threshold)     │   │
│  │                                      │   │
│  │  ┌───────────────────────────────┐   │   │
│  │  │ 第3层：遮挡检查 (dist < thr)  │   │   │
│  │  │                               │   │   │
│  │  │  ✓ 采样辐射度并返回 true      │   │   │
│  │  │                               │   │   │
│  │  └───────────────────────────────┘   │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

每一层从粗到细地排除不适合的情况：
1. **范围检查**：O(1)，仅需一次点积和比较
2. **分辨率检查**：估计采样精度，防止欠采样伪影
3. **遮挡检查**：需要纹理采样，但仅在前两层通过后执行

### 1/cos 修正的几何意义

```
         探针中心
            │
            │ hitZInCube (轴向距离)
            │
            ├─────────── 立方体面
           /
          /  distanceFromCube (实际距离)
         /
        /  alpha (观察角)
       命中点
```

深度图存储的是沿立方体面法线方向的深度。当采样方向偏离法线时（$\alpha > 0$），需要将深度差乘以 $1/\cos\alpha$ 来获得沿实际视线方向的距离差：

$$d_{\text{actual}} = |z_{\text{hit}} - z_{\text{probe}}| \cdot \frac{1}{\cos\alpha}$$

### 自适应遮挡阈值

遮挡阈值 $T \cdot z_{\text{hit}} / R$ 随距离线性增大：

- **近处**（$z$ 小）：阈值小，深度匹配要求严格
- **远处**（$z$ 大）：阈值大，允许更多深度误差

这是因为立方体贴图的单个纹素在远处对应更大的空间范围，深度精度自然降低。

### 分辨率适配性

`ProbabilityToSampleSameTexel` 函数（未在本文件中展示）估计了在当前查找配置下，微小的方向变化是否会映射到不同的纹素。当命中点非常靠近探针时，探针立方体贴图的单个纹素覆盖的空间范围很小，概率较高（分辨率不足）。当命中点在探针覆盖范围的中远距离时，分辨率通常足够。

## 输入与输出

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `probeIndex` | `uint` | in | 探针索引 |
| `hitPos` | `float3` | in | 反射命中点的世界空间位置 |
| `radiance` | `float3` | out | 采样到的辐射度（仅在返回 true 时有效） |
| **返回值** | `bool` | - | 是否成功采样 |

### 依赖的外部资源

| 资源 | 说明 |
|------|------|
| `LoadCube(index)` | 加载探针的所有数据 |
| `cube.Depth` | 探针深度立方体贴图 |
| `cube.Radiance` | 探针辐射度立方体贴图 |
| `Sampler` | 纹理采样器（通常为线性插值） |
| `OcclusionThresholdFactor` | 遮挡检查的阈值因子 |
| `ResolutionThreshold` | 分辨率检查的阈值 |

## 与其他文件的关系

- **SamplePrecomputedRadiance.hlsl**: 在循环中调用本函数，对每个探针逐一尝试
- **RayGen.hlsl**: 提供反射命中点位置（通过光线原点 + 方向 * 长度计算）

## 在光线追踪管线中的位置

本函数作为辐射度查找阶段的子模块被调用：

```
[SamplePrecomputedRadiance]
    │
    ├── SampleSkybox()           (miss)
    ├── SampleScreen()           (屏幕空间)
    └── SampleRadianceProbe()    ← 本文件
         │
         ├── 范围检查
         ├── 分辨率检查
         ├── 深度纹理采样
         ├── 遮挡检查
         └── 辐射度纹理采样
```

## 技术要点与注意事项

1. **平方距离优化**：范围检查使用 `distSqr <= RadiusSqr` 避免昂贵的 `sqrt` 运算，仅在通过检查后才计算 `sqrt(distSqr)`
2. **MaxDir 的离散性**：`MaxDir` 返回三个轴向之一 (1,0,0) / (0,1,0) / (0,0,1)，在立方体面边界处可能导致不连续
3. **SampleLevel 而非 Sample**：使用 `SampleLevel(..., 0)` 强制使用最高分辨率 mip level，避免自动 mip 选择引入的模糊
4. **线性深度转换**：`ZInCube()` 将非线性深度值转换为线性深度，确保深度比较的正确性
5. **探针放置策略**：探针的位置和覆盖半径由离线工具或自动放置算法决定，影响覆盖率和采样质量
6. **边界情况**：`hitZInCube` 可能接近 0（命中点几乎在探针中心），此时 `cosCubeViewAngleRcp` 趋于无穷，需要确保 `hitZInCube` 有足够的最小值保护

## 扩展阅读

- McGuire, M. et al. *Real-Time Global Illumination using Precomputed Light Field Probes*, I3D 2017
- Hooker, J. *Volumetric Global Illumination at Treyarch*, SIGGRAPH 2016
- Ray Tracing Gems, Chapter 32, Section 32.5: Sampling Radiance Probes
- Ramamoorthi, R. & Hanrahan, P. *An Efficient Representation for Irradiance Environment Maps*, SIGGRAPH 2001
