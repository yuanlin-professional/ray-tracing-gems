# DXRShadows.rt.hlsl 技术文档

## 文件概述

`DXRShadows.rt.hlsl` 是本章最核心的 HLSL 着色器文件，实现了完整的 DXR 光线追踪阴影管线。它包含 Ray Generation（光线生成）、Closest Hit（最近命中）、Any Hit（任意命中）和 Miss（未命中）四类着色器程序，以及自适应采样、采样矩阵剔除和面光源圆盘采样等关键算法。

**源文件路径**: `Ch_13_Ray_Traced_Shadows_Maintaining_Real-Time_Frame_Rates/dxrShadows/Data/DXRShadows.rt.hlsl`

## 算法与数学背景

### 面光源圆盘采样

对于球形面光源，在光源圆盘上使用泊松圆盘采样来近似面积积分。构造 TBN 正交基：

$$\hat{n} = \text{normalize}(\mathbf{p}_{\text{light}} - \mathbf{p}_{\text{origin}})$$
$$\hat{b}_1 = \text{normalize}(\hat{v} - \hat{n} \cdot (\hat{v} \cdot \hat{n}))$$
$$\hat{b}_2 = \hat{n} \times \hat{b}_1$$

其中 $\hat{v}$ 为归一化的视空间像素位置，作为"up"参考向量。

每个采样方向：
$$\mathbf{d}_i = \mathbf{p}_{\text{light}} + \text{TBN} \cdot (s_{i,x} \cdot r, s_{i,y} \cdot r, 0) - \mathbf{p}_{\text{origin}}$$

可见性近似为：
$$V \approx 1 - \frac{1}{N}\sum_{i=1}^{N} \mathbb{1}[\text{hit}_i]$$

### 软方向光近似

将软方向光转换为虚拟球形光源：

$$\mathbf{p}_{\text{virtual}} = \mathbf{p}_{\text{pixel}} - \hat{\mathbf{d}}_{\text{light}} \cdot \frac{1}{\tan(\omega)}$$

其中 $\omega$ 为光源的立体角参数，虚拟球光半径固定为 1。

### 自适应采样策略

基于变化度 (Variation) 反馈控制采样数：

$$n_{\text{next}} = \begin{cases}
n_{\text{prev}} - 1 & \text{if } v < v_{\text{threshold}} \text{ 且连续3帧稳定} \\
n_{\text{prev}} + 1 & \text{if } v > v_{\text{threshold}} \\
n_{\text{prev}} & \text{otherwise}
\end{cases}$$

采样数被限制在 $[1, n_{\max}]$ 范围内，其中 $n_{\max}$ 由质量等级决定（标准=5, 高=8）。

### 采样矩阵剔除

使用 $4 \times 4$ 掩码矩阵，按 `cullingBlockSize` 大小的像素块分配帧号：

$$\text{mask} = \begin{bmatrix} 0 & 3 & 2 & 1 \\ 2 & 1 & 0 & 3 \\ 0 & 3 & 2 & 1 \\ 1 & 2 & 0 & 3 \end{bmatrix}$$

当像素块的掩码值 $\neq$ 当前帧号，且变化度为零时，跳过该像素的阴影采样。

## 代码结构概览

```
DXRShadows.rt.hlsl
├── 输入/输出声明
│   ├── Texture2D inputVisibilityCache[8]  -- 输入可见性缓存
│   ├── RWTexture2D outputVisibilityCache[8] -- 输出可见性缓存
│   ├── RWTexture2D outputFilteredVisibilityBuffer[2] -- 滤波后可见性
│   └── RWTexture2D outputFilteredVariationBuffer[2]  -- 变化度缓冲
├── PerFrameCB 常量缓冲
├── Payload 结构体
│   ├── PrimaryRayData   -- 主光线负载
│   └── ShadowRayData    -- 阴影光线负载
├── 着色器程序
│   ├── shadowMiss()           -- 阴影光线未命中
│   ├── shadowAnyHit()         -- 阴影光线任意命中
│   ├── primaryMiss()          -- 主光线未命中
│   ├── primaryClosestHit()    -- 主光线最近命中（递归）
│   ├── checkLightHit()        -- 单条阴影光线检测
│   ├── sampleDiskLight()      -- 面光源圆盘采样
│   ├── sampleLight()          -- 光源采样分发
│   ├── getWorldSpacePixelPosition() -- 世界坐标重建
│   └── rayGen()               -- 光线生成主入口
```

## 逐段代码详解

### Payload 结构体

```hlsl
struct PrimaryRayData {
    float4 color;  // 累积颜色（用于透明物体穿透）
    int depth;     // 当前递归深度
    float hitT;    // 命中距离
};

struct ShadowRayData {
    bool hit;      // 是否命中遮挡物
};
```

### shadowAnyHit -- 阴影光线命中

```hlsl
[shader("anyhit")]
void shadowAnyHit(inout ShadowRayData hitData, in BuiltInTriangleIntersectionAttributes attribs)
{
    hitData.hit = true;  // 任何命中即标记为遮挡
}
```

配合 `RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH` 标志，一旦命中即终止遍历，最大化性能。

### sampleDiskLight -- 核心圆盘采样

```hlsl
float sampleDiskLight(uint lightIndex, float3 lightCenter, float lightRadius,
    float3 viewSpacePos, float3 origin, uint2 launchIndex,
    bool isDirectional, const int samplesCount, const int frameNumber)
{
    float result = 1.0f;

    // 1. 构造 TBN 基
    float3 n = normalize(lightCenter - origin);
    float3 rvec = normalize(viewSpacePos);
    float3 b1 = normalize(rvec - n * dot(rvec, n));
    float3 b2 = cross(n, b1);
    float3x3 tbn = float3x3(b1, b2, n);

    // 2. 计算采样起始索引（基于帧号 + 屏幕位置）
    int pixelIdx = dot(int2(fmod(float2(launchIndex), 3)), int2(1, 3));
    int startIdx = lightSamplesIndices[samplesCount]
                 + (pixelIdx * samplesCount)
                 + int(frameNumber * (samplesCount * 9));

    float weight = 1.0f / float(samplesCount);

    // 3. 发射采样光线
    [unroll]
    for (int i = 0; i < samplesCount; i++) {
        float2 disk = samples.Load(int2(startIdx + i, 0)).xy * lightRadius;
        float3 dir = lightCenter + mul(float3(disk, 0), tbn) - origin;
        // ... 设置光线并追踪 ...
        if (rayResult.hit) result -= weight;
    }
    return result;
}
```

### rayGen -- 主入口

核心逻辑流程：
1. 计算屏幕 UV 和加载运动向量
2. 读取前帧变化度缓冲（当前位置 + 重投影位置 + 全局最低 mip）
3. 重建世界坐标（3种模式：深度预通道 / G-Buffer / Primary Ray）
4. 对每个光源：
   a. 解码历史变化度和采样数
   b. 检查采样矩阵剔除
   c. 自适应采样数调整
   d. 执行采样（或复用历史值）
   e. 更新可见性历史、变化度和采样数缓存
5. 打包输出滤波后的可见性和变化度

## 关键算法深入分析

### 时域重投影与历史管理

可见性历史为 4 帧滑动窗口 `float4(current, prev1, prev2, prev3)`。重投影成功时：
```hlsl
float4 newHistory = float4(visibilityResult,
    visibilityHistoryReprojected.r,
    visibilityHistoryReprojected.g,
    visibilityHistoryReprojected.b);
```

重投影失败时，用当前帧结果填充全部历史：
```hlsl
float4 newHistory = float4(visibilityResult);
```

### 全局采样数限制

当全局变化度（最低 mip 级别的平均值）超过阈值时，渐进降低采样数：

$$n = \text{lerp}(n, 1, (v_{\text{global}} - \theta) \cdot \text{falloff})$$

这保证了在高复杂度场景中维持帧率。

## 输入与输出

**输入**:
- `inputVisibilityCache[8]`: 前帧可见性历史
- `inputFilteredVariationBuffer[2]`: 前帧滤波变化度
- `motionAndDepthBuffer`: 运动向量 + 线性深度
- `samples`: 1D 泊松圆盘采样纹理

**输出** (UAV):
- `outputVisibilityCache[8]`: 更新后的可见性历史
- `outputVariationAndSampleCountCache[8]`: 变化度 + 采样数
- `outputFilteredVisibilityBuffer[2]`: 当前帧滤波可见性
- `outputFilteredVariationBuffer[2]`: 当前帧变化度

## 与其他文件的关系

- 包含 `CommonUtils.hlsl` 获取工具函数
- 使用 `shaderCommon.h` 定义的常量和结构体
- 由 `dxrShadows.cpp` 中的 `rayTracedShadowsPass()` 调度执行
- 输出被 `VisibilityFilteringPass.ps.hlsl` 和 `ForwardRenderer.ps.hlsl` 消费

## 在光线追踪管线中的位置

这是管线的核心计算阶段，位于深度/运动向量生成之后、空间滤波和光照之前。

## 技术要点与注意事项

1. 使用 `[unroll]` 提示编译器展开采样循环，避免动态分支开销
2. `RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH` 标志对阴影光线至关重要
3. Primary Ray 模式支持递归追踪透明物体（最大深度由 `maxRayRecursionDepth` 控制）
4. 采样索引计算依赖 `launchIndex % 3` 的空间交错，确保 $3 \times 3$ 邻域内无采样重叠
5. `pack4ToTexture` 被手动内联以避免 DXC 编译器的验证错误

## 扩展阅读

- Ray Tracing Gems, Chapter 13
- "Stochastic Soft Shadow Mapping" (Heidrich et al.)
- "Adaptive Temporal Antialiasing" (Karis, SIGGRAPH 2014)
