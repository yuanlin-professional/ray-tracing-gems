# ForwardRenderer.ps.hlsl 技术文档

## 文件概述

`ForwardRenderer.ps.hlsl` 实现了最终光照通道 (Lighting Pass) 的顶点和像素着色器。它读取预计算的阴影可见性缓冲，结合 Falcor 的 PBR 材质系统对每个光源进行着色计算。支持光线追踪阴影、阴影贴图和无阴影三种模式。

**源文件路径**: `Ch_13_Ray_Traced_Shadows_Maintaining_Real-Time_Frame_Rates/dxrShadows/Data/ForwardRenderer.ps.hlsl`

## 算法与数学背景

### 光照计算

对每个光源 $l$ 的贡献：

$$L_o = \sum_{l=0}^{N-1} V_l \cdot f(\omega_i, \omega_o) \cdot L_l \cdot |\cos\theta_l|$$

其中 $V_l$ 为阴影可见性因子（0=完全遮挡, 1=完全可见），$f$ 为 BRDF。

### 可见性读取

光线追踪模式下，每 4 个光源的可见性打包在一个 RGBA 纹理中：

$$V_l = \text{channel}[l \mod 4] \text{ of } \text{gVisibilityBuffers}[\lfloor l/4 \rfloor]$$

阴影贴图模式下，每个光源有独立的可见性纹理。

## 代码结构概览

```hlsl
ForwardRenderer.ps.hlsl
├── PerFrameCB 常量缓冲
│   ├── CsmData gCsmData          -- CSM 数据
│   ├── camVpAtLastCsmUpdate      -- CSM 更新时的VP矩阵
│   ├── gVisibilityBuffers[8]     -- 可见性纹理数组
│   ├── visibilityBufferToDisplay -- 调试：指定显示某个光源
│   └── lights[16]                -- 光源数据 (Falcor LightData)
├── vs() -- 顶点着色器（计算CSM深度）
├── getRtShadowValue() -- 获取阴影可见性
└── ps() -- 像素着色器（光照累加）
```

## 逐段代码详解

### 顶点着色器

```hlsl
MainVsOut vs(VertexIn vIn) {
    MainVsOut vsOut;
    vsOut.vsData = defaultVS(vIn);
    // 计算 CSM 阴影所需的深度值（使用上次 CSM 更新时的 VP 矩阵）
    vsOut.shadowsDepthC = mul(float4(vsOut.vsData.posW, 1), camVpAtLastCsmUpdate).z;
    return vsOut;
}
```

### 像素着色器核心

```hlsl
float4 ps(MainVsOut vOut, float4 pixelCrd : SV_POSITION) : SV_TARGET
{
    ShadingData sd = prepareShadingData(vOut.vsData, gMaterial, gCamera.posW);
    float4 finalColor = float4(0, 0, 0, 1);

    // 读取打包的可见性缓冲（光追模式）
    #ifdef USE_RAYTRACED_SHADOWS
    float4 vis0To3 = gVisibilityBuffers[0].Load(int3(pos, 0)).rgba;
    float4 vis4To7 = gVisibilityBuffers[1].Load(int3(pos, 0)).rgba;
    #endif

    [unroll]
    for (uint l = 0; l < _LIGHT_COUNT; l++) {
        float shadowFactor = getRtShadowValue(vis0To3, vis4To7, l, pos);

        // 调试模式：仅显示指定光源
        if (visibilityBufferToDisplay != INVALID_LIGHT_INDEX)
            shadowFactor = (visibilityBufferToDisplay == l) ? shadowFactor : 1.0f;

        #ifdef DISPLAY_SHADOWS_ONLY
            finalColor.rgb += shadowFactor / float(_LIGHT_COUNT);
        #else
            if (shadowFactor > 0.0f)
                finalColor.rgb += evalMaterial(sd, lights[l], shadowFactor).color.rgb;
        #endif
    }

    // 发光体和 lightmap
    finalColor.rgb += sd.emissive + sd.diffuse * sd.lightMap.rgb;
    return finalColor;
}
```

## 输入与输出

**输入**: 顶点属性、材质纹理、可见性缓冲纹理、光源数据

**输出**: `SV_TARGET` -- 最终颜色

## 与其他文件的关系

- 消费 `DXRShadows.rt.hlsl` 或 `CascadedShadowMaps` 的可见性输出
- 使用 `VisibilityFilteringPass.ps.hlsl` 滤波后的可见性缓冲
- 由 `dxrShadows.cpp::lightingPass()` 调度
- 配合 `DepthFunc::Equal` 使用预填充的深度缓冲

## 技术要点与注意事项

1. 使用 `DepthFunc::Equal` 确保仅对最前面的片元着色（Zero Overdraw）
2. `DISPLAY_SHADOWS_ONLY` 模式直接输出阴影因子的灰度值，便于调试
3. 当 `shadowFactor > 0` 时才调用 `evalMaterial`，跳过完全阴影区域的 BRDF 计算
4. 光源数据使用 Falcor 的 `LightData` 结构，由 CPU 端填充

## 扩展阅读

- Falcor PBR 材质系统
- Forward Rendering vs Deferred Rendering
- "Shadows Only" 调试技巧在阴影算法开发中的应用
