# DepthAndNormalsPass.ps.hlsl 技术文档

## 文件概述

`DepthAndNormalsPass.ps.hlsl` 实现了 G-Buffer 通道的像素着色器，同时输出运动向量、线性深度和视空间法线。这是深度/法线源的 "GBufferPass" 模式的实现，相比纯深度预通道 + 后处理法线重建，能提供更精确的法线数据。

**源文件路径**: `Ch_13_Ray_Traced_Shadows_Maintaining_Real-Time_Frame_Rates/dxrShadows/Data/DepthAndNormalsPass.ps.hlsl`

## 算法与数学背景

### 反向重投影 (Reverse Reprojection)

利用 Falcor 提供的 `prevPosH`（前一帧的裁剪空间坐标），将当前帧像素反投影到前一帧屏幕空间：

$$\text{prevUV} = \frac{\mathbf{p}_{\text{prev},xy} / w_{\text{prev}}}{2} \cdot (1, -1) + 0.5$$

### 重投影有效性验证

两项检查：
1. **越界检查**: 前帧 UV 不在 $[0, 1]^2$ 范围内时标记无效
2. **深度一致性检查**: 重投影深度与采样深度的相对差异超过阈值时标记无效：

$$\left|1 - \frac{d_{\text{reproj}}}{d_{\text{sampled}}}\right| > \delta_{\max}$$

### 视空间法线变换

顶点法线从世界空间变换到视空间：

$$\mathbf{n}_{\text{view}} = \text{normalize}((M_V^{-T}) \cdot \mathbf{n}_{\text{world}})$$

其中 $M_V^{-T}$ 即 `normalMatrix = transpose(inverse(viewMatrix))`。

## 完整源码与注解

```hlsl
__import DefaultVS;
__import ShaderCommon;
__import Shading;
#include "CommonUtils.hlsl"

cbuffer GBufferPassCB
{
    float4x4 normalMatrix;           // 法线变换矩阵 = transpose(inverse(view))
    float farOverNear;               // 远/近平面比值
    float maxReprojectionDepthDifference; // 重投影深度差阈值 (默认 0.003)
};

Texture2D previousDepthBuffer;       // 前帧深度缓冲
SamplerState bilinearSampler;

struct PsOut
{
    float4 motionAndDepth : SV_TARGET0; // xy = 前帧UV, z = 当前线性深度
    float4 normal : SV_TARGET1;         // xyz = 视空间法线
};

PsOut main(VertexOut vOut)
{
    PsOut output;

    ShadingData sd = prepareShadingData(vOut, gMaterial, gCamera.posW);

    // 跳过透明物体
    if (sd.opacity < 1.0f) discard;

    // 计算当前帧线性深度
    float3 pixelViewSpacePosition = mul(float4(vOut.posW, 1.0f), gCamera.viewMat).xyz;
    float currentLinearDepth = length(pixelViewSpacePosition);

    // 计算前帧重投影 UV
    float previousReprojectedLinearDepth01 =
        getNormalizedLinearDepth(vOut.prevPosH.z / vOut.prevPosH.w, farOverNear);
    float2 previousUvs = ((vOut.prevPosH.xy / vOut.prevPosH.w) * float2(0.5f, -0.5f)) + 0.5f;

    // 验证1: 越界检查
    if (previousUvs.x > 1.0f || previousUvs.x < 0.0f) previousUvs = INVALID_UVS;
    if (previousUvs.y > 1.0f || previousUvs.y < 0.0f) previousUvs = INVALID_UVS;

    // 验证2: 深度一致性检查
    float previousSampledLinearDepth01 =
        getNormalizedLinearDepth(previousDepthBuffer.Sample(bilinearSampler, previousUvs).r, farOverNear);
    if (abs(1.0f - (previousReprojectedLinearDepth01 / previousSampledLinearDepth01))
        > maxReprojectionDepthDifference)
        previousUvs = INVALID_UVS;

    output.motionAndDepth = float4(previousUvs, currentLinearDepth, 0.0f);
    output.normal = float4(normalize(mul(float4(normalize(vOut.normalW), 0.0f), normalMatrix).xyz), 0.0f);

    return output;
}
```

## 输入与输出

**输入**: 顶点属性 (位置、法线、前帧裁剪坐标)、前帧深度缓冲

**输出**:
- `SV_TARGET0` (motionAndDepth): `float4(prevUV.x, prevUV.y, linearDepth, 0)`
- `SV_TARGET1` (normal): `float4(normalView.xyz, 0)`

## 与其他文件的关系

- 作为 G-Buffer 通道的替代方案，与 `DepthPass.ps.hlsl` + `MotionVectors.ps.hlsl` 互斥
- 输出被 `DXRShadows.rt.hlsl` 和 `VisibilityFilteringPass.ps.hlsl` 使用

## 技术要点与注意事项

1. G-Buffer 法线直接来自顶点插值，精度优于从深度重建
2. `prevPosH` 由 Falcor 的 `DefaultVS` 自动计算（基于前帧 MVP 矩阵）
3. 线性深度使用 `length(viewSpacePos)` 而非 Z 分量，为欧几里得距离
4. 透明物体被 `discard`，不参与阴影计算

## 扩展阅读

- Deferred Rendering / G-Buffer 设计模式
- Temporal Reprojection in Real-Time Rendering
