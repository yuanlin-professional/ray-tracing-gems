# VisibilityFilteringPass.ps.hlsl 技术文档

## 文件概述

`VisibilityFilteringPass.ps.hlsl` 实现了阴影可见性缓冲区 (Visibility Buffer) 的空间降噪 (Spatial Denoising) 通道。这是阴影质量的最终保证环节，使用**深度/法线感知的自适应高斯滤波**消除光线追踪阴影中的残留噪声，同时保持深度和法线边缘处的锐利阴影。

**源文件路径**: `Ch_13_Ray_Traced_Shadows_Maintaining_Real-Time_Frame_Rates/dxrShadows/Data/VisibilityFilteringPass.ps.hlsl`

## 算法与数学背景

### 自适应核大小选择

滤波核大小基于局部变化度 $v$ 自适应选择。变化度越高（阴影越"噪声"），使用越大的核：

$$k = v / v_{\max} \cdot (4 - \epsilon)$$

核大小从 1 tap（无滤波）到 9 taps（最大滤波），通过线性插值实现平滑过渡。

### 高斯权重

预计算了 5 组高斯权重（1/3/5/7/9 taps），在相邻两组之间插值：

$$w_i = \text{lerp}(\text{gauss}[k_{\text{low}}][i], \text{gauss}[k_{\text{high}}][i], \text{frac}(k))$$

### 边缘保持 (Edge-Preserving)

通过深度和法线双重检测防止模糊穿越几何边界：

**深度检测**：
$$\text{valid} = \left|1 - \frac{d_{\text{tap}}}{d_{\text{center}}}\right| < \epsilon_d$$

其中 $\epsilon_d$ 基于法线 Z 分量自适应调整：
$$\epsilon_d = \text{lerp}(\epsilon_{\max}, \epsilon_{\min}, |n_z|)$$

**法线检测**：
$$\text{valid} = \hat{\mathbf{n}}_{\text{tap}} \cdot \hat{\mathbf{n}}_{\text{center}} > \theta_n$$

### 权重补偿

被拒绝的 tap 的权重从总权重中扣除，确保结果归一化：

$$V_{\text{filtered}} = \frac{\sum_{\text{valid}} w_i \cdot V_i}{1 - \sum_{\text{invalid}} w_i}$$

## 代码结构概览

```
VisibilityFilteringPass.ps.hlsl
├── FilteringPassCB 常量缓冲
├── gauss[45] -- 预计算高斯权重 (5组 x 9 taps)
├── isValidTap() -- 深度/法线边缘检测
├── blurPass() -- 核心高斯模糊
├── selectBlurPassSize() -- 根据变化度选择核大小
├── visibilityFilteringPass() -- 入口分发
└── main() -- 像素着色器入口
```

## 逐段代码详解

### 高斯权重表

```hlsl
static const float gauss[45] = {
    // 1 tap:  [0,0,0,0, 1.0, 0,0,0,0]
    // 3 taps: [0,0,0, 0.197, 0.606, 0.197, 0,0,0]
    // 5 taps: [0,0, 0.054, 0.244, 0.403, 0.244, 0.054, 0,0]
    // 7 taps: [0, 0.024, 0.098, 0.227, 0.301, 0.227, 0.098, 0.024, 0]
    // 9 taps: [0.014, 0.048, 0.117, 0.201, 0.241, 0.201, 0.117, 0.048, 0.014]
};
```

每组 9 个权重，较小的核将外围权重设为 0。

### 核心模糊函数

```hlsl
inline float4 blurPass(const int halfBlurSize, int4 kernelOffsets,
    float4 kernelBlends, int bufferIdx, int2 pos)
{
    float4 acc = 0.0f;
    float4 weight = 1.0f;

    float centerDepth = getNormalizedLinearDepth(depthBuffer.Load(int3(pos, 0)).x, farOverNear);
    float3 centerNormal = normalize(normalsBuffer.Load(int3(pos, 0)).xyz);

    [unroll]
    for (int i = -halfBlurSize; i <= halfBlurSize; ++i) {
        int2 offset = (HORIZONTAL) ? pos + int2(i, 0) : pos + int2(0, i);

        float tapDepth = getNormalizedLinearDepth(depthBuffer.Load(int3(offset, 0)).x, farOverNear);
        float3 tapNormal = normalize(normalsBuffer.Load(int3(offset, 0)).xyz);
        float4 tapShadow = inputBuffer[bufferIdx].Load(int3(offset, 0));

        // 在两组高斯权重之间插值（per-light 独立核大小）
        float4 tapWeightLow = float4(gauss[offsets.x+i], gauss[offsets.y+i], ...);
        float4 tapWeightHigh = float4(gauss[offsets.x+i+9], ...);
        float4 tapWeight = lerp(tapWeightLow, tapWeightHigh, kernelBlends);

        if (isValidTap(tapDepth, centerDepth, tapNormal, centerNormal))
            acc += tapShadow * tapWeight;
        else
            weight -= tapWeight; // 扣除无效 tap 的权重
    }

    return acc / weight;
}
```

注意：`int4 kernelOffsets` 和 `float4 kernelBlends` 意味着 **每个 RGBA 通道对应的光源可以使用不同大小的滤波核**。

## 输入与输出

**输入**:
- `depthBuffer`: 深度缓冲
- `normalsBuffer`: 法线缓冲
- `inputBuffer[2]`: 打包的可见性纹理
- `variationBuffer[2]`: 打包的变化度纹理

**输出**:
- `SV_TARGET0` (visibility0to3): 光源 0-3 的滤波可见性
- `SV_TARGET1` (visibility4to7): 光源 4-7 的滤波可见性

## 与其他文件的关系

- 消费 `DXRShadows.rt.hlsl` 的可见性输出和 `VariationFilteringPass.ps.hlsl` 的变化度
- 输出被 `ForwardRenderer.ps.hlsl` 最终消费
- 由 `dxrShadows.cpp::visibilityFilteringPass()` 调度

## 在光线追踪管线中的位置

位于光线追踪和最终光照之间，是阴影管线的最后一个降噪环节。

## 技术要点与注意事项

1. 每光源独立的核大小选择使得高变化度的光源获得更多平滑，低变化度的保持锐利
2. 深度容差 $\epsilon_d$ 基于法线 Z 分量调整：表面朝向摄像机时使用更严格的阈值，避免过度模糊
3. 9 taps 的滤波半径为 4 像素，配合时域 4 帧 + 空间 $3 \times 3$ 交错，总有效范围约 $12 \times 12$
4. `FILTERING_OFF` 宏可禁用滤波，直接输出原始数据（调试用途）
5. 分离式执行：先水平再垂直，使用中间临时纹理

## 扩展阅读

- "A-Trous Wavelet Transform" for Image Denoising
- "Spatiotemporal Variance-Guided Filtering" (SVGF)
- "Edge-Avoiding A-Trous Wavelet Transform for fast Global Illumination Filtering"
