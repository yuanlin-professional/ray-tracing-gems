# MotionVectors.ps.hlsl 技术文档

## 文件概述

`MotionVectors.ps.hlsl` 实现了基于深度缓冲的全屏运动向量生成 (Motion Vectors Pass)。当使用"深度预通道"模式（而非 G-Buffer 模式）时，该着色器负责计算每个像素从当前帧到前一帧的屏幕空间位移，同时重建视空间法线。

**源文件路径**: `Ch_13_Ray_Traced_Shadows_Maintaining_Real-Time_Frame_Rates/dxrShadows/Data/MotionVectors.ps.hlsl`

## 算法与数学背景

### 深度缓冲重投影

1. 从当前帧深度缓冲读取深度 $z$，转换为线性深度 $d$
2. 使用反投影重建视空间位置 $\mathbf{p}_{\text{view}}$
3. 应用运动向量矩阵 $M_{\text{motion}} = M_{\text{VP,prev}} \cdot M_{\text{V,curr}}^{-1}$ 得到前帧裁剪空间坐标
4. 转换为前帧屏幕 UV

$$\mathbf{p}_{\text{prev,clip}} = M_{\text{motion}} \cdot \begin{bmatrix} \mathbf{p}_{\text{view}} \\ 1 \end{bmatrix}$$

$$\text{prevUV} = \frac{\mathbf{p}_{\text{prev,clip},xy}}{w} \cdot (0.5, -0.5) + 0.5$$

### 深度差分法线重建

使用 `calculateNormal()` 从深度缓冲重建视空间法线（4 邻域十字形差分）。

## 完整源码与注解

```hlsl
__import ShaderCommon;
__import Shading;
#include "CommonUtils.hlsl"

cbuffer MotionVectorsPassCB
{
    float4x4 motionVectorsMatrix; // = prevVP * invCurrentView
    float4 screenUnprojection;     // 反投影参数1
    float4 screenUnprojection2;    // 反投影参数2
    float2 texelSizeRcp;           // 1/width, 1/height
    float farOverNear;             // 远/近平面比值
    float maxReprojectionDepthDifference; // 深度差阈值
};

Texture2D currentDepthBuffer;
Texture2D previousDepthBuffer;
SamplerState bilinearSampler;

struct PsOut
{
    float4 motionAndDepth : SV_TARGET0; // xy=前帧UV, z=线性深度
    float4 normal : SV_TARGET1;         // xyz=视空间法线
};

PsOut main(float2 uvs : TEXCOORD, float4 pos : SV_POSITION)
{
    PsOut output;

    // 1. 读取并线性化当前帧深度
    float currentLinearDepth01 = getNormalizedLinearDepth(
        currentDepthBuffer.Load(int3(pos.xy, 0)).r, farOverNear);

    // 2. 反投影到视空间
    float3 pixelViewSpacePosition = getViewSpacePositionUsingDepth(
        uvs, currentLinearDepth01, screenUnprojection, screenUnprojection2) * float3(1, -1, 1);

    // 3. 投影到前帧裁剪空间
    float4 prevPos = mul(float4(pixelViewSpacePosition, 1.0f), motionVectorsMatrix);
    prevPos.xyz /= prevPos.w;
    float prevDepth01 = getNormalizedLinearDepth(prevPos.z, farOverNear);

    // 4. 转换为前帧 UV
    float2 previousUvs = (prevPos.xy * float2(0.5f, -0.5f)) + 0.5f;

    // 5. 有效性验证
    if (previousUvs.x > 1.0f || previousUvs.x < 0.0f) previousUvs = INVALID_UVS;
    if (previousUvs.y > 1.0f || previousUvs.y < 0.0f) previousUvs = INVALID_UVS;

    float prevSampledDepth01 = getNormalizedLinearDepth(
        previousDepthBuffer.Sample(bilinearSampler, previousUvs).r, farOverNear);
    if (abs(1.0f - (prevDepth01 / prevSampledDepth01)) > maxReprojectionDepthDifference)
        previousUvs = INVALID_UVS;

    output.motionAndDepth = float4(previousUvs, currentLinearDepth01, 0.0f);

    // 6. 从深度缓冲重建法线
    output.normal = float4(calculateNormal(currentDepthBuffer, uvs, pos.xy,
        farOverNear, screenUnprojection, screenUnprojection2, texelSizeRcp), 0.0f);

    return output;
}
```

## 输入与输出

**输入**: 当前帧和前帧深度缓冲

**输出**:
- `SV_TARGET0`: 运动向量 + 线性深度
- `SV_TARGET1`: 从深度重建的视空间法线

## 与其他文件的关系

- 与 `DepthPass.ps.hlsl` 配合使用（深度预通道模式）
- 与 `DepthAndNormalsPass.ps.hlsl` 互斥（G-Buffer 模式直接输出）
- 输出被 `DXRShadows.rt.hlsl` 的 `motionAndDepthBuffer` 使用

## 技术要点与注意事项

1. Y 轴翻转 `float3(1, -1, 1)` 因为纹理坐标系与摄像机坐标系的 Y 轴方向相反
2. 运动向量矩阵在 CPU 端预计算为 `prevVP * inv(currentView)`
3. 从深度重建的法线精度低于 G-Buffer 直接输出的法线，但节省一个完整的场景绘制
4. 双线性采样 (`bilinearSampler`) 用于前帧深度的亚像素采样

## 扩展阅读

- "GPU-Based Importance Sampling" (Chapter on Motion Vectors)
- Temporal Antialiasing 中的运动向量生成
