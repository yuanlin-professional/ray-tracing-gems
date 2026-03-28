# dxrShadows.cpp 技术文档

## 文件概述

`dxrShadows.cpp` 是第 13 章 DXR 实时阴影示例的核心实现文件，包含主应用类 `DXRShadows` 的全部方法实现。该文件编排了一个完整的多通道渲染管线，从场景加载到最终像素输出，涵盖深度预通道、运动向量生成、光线追踪阴影、空间/时域滤波以及最终光照计算。

**源文件路径**: `Ch_13_Ray_Traced_Shadows_Maintaining_Real-Time_Frame_Rates/dxrShadows/dxrShadows.cpp`

## 算法与数学背景

### 时域重投影 (Temporal Reprojection)

每帧保存视图-投影矩阵 (View-Projection Matrix)，下一帧通过该矩阵将当前像素反投影到前一帧屏幕空间：

$$\mathbf{p}_{\text{prev}} = M_{\text{VP,prev}} \cdot M_{\text{V,curr}}^{-1} \cdot \mathbf{p}_{\text{view}}$$

其中 $M_{\text{VP,prev}}$ 为前一帧的 View-Projection 矩阵。

### 反投影重建世界坐标

从深度缓冲重建视空间位置：

$$\mathbf{p}_{\text{view}} = \text{ray}(u,v) \cdot d_{\text{linear}}$$

其中 ray 方向由投影矩阵的逆构造。

### 采样数调度

非自适应模式支持 1-32 SPP，但仅允许特定值：1-8, 12, 16, 20, 24, 28, 32（对应预计算泊松圆盘的分组）。

## 代码结构概览

```
dxrShadows.cpp
├── onGuiRender()           -- GUI 界面
├── loadScene()             -- 场景加载
├── onLoad()                -- 初始化
├── 渲染通道
│   ├── GBufferPass()       -- G-Buffer 深度+法线
│   ├── depthPass()         -- 纯深度预通道
│   ├── motionVectorsPass() -- 运动向量生成
│   ├── shadowMappingPass() -- CSM 阴影贴图
│   ├── updateLightBuffer() -- 光源数据更新
│   ├── rayTracedShadowsPass()      -- DXR 光追阴影
│   ├── variationFilteringPass()     -- 变化度滤波
│   ├── visibilityFilteringPass()    -- 可见性降噪
│   ├── lightingPass()               -- 最终光照
│   ├── debugVisualizationsPass()    -- 调试可视化
│   └── renderSkyBox()               -- 天空盒
├── renderScene()           -- 渲染流程编排
├── onFrameRender()         -- Falcor 帧回调
├── onResizeSwapChain()     -- 交换链重建
├── 工具函数
│   ├── getCurrentFrameDepthFBO() / getPreviousFrameDepthFBO()
│   ├── applyDefine()
│   ├── getPassesPerSceneNum() / getLightsPerPassNum()
│   └── getPackedTexturesPerSceneNum() / getPackedTexturesPerPassNum()
└── WinMain()               -- 程序入口
```

## 逐段代码详解

### onLoad() -- 应用初始化

```cpp
void DXRShadows::onLoad(SampleCallbacks* pSample, RenderContext* pRenderContext)
{
    // 1. 检查 DXR 支持
    // 2. 创建 RT 程序：rayGen + primaryClosestHit/miss + shadowAnyHit/miss
    RtProgram::Desc rtProgDesc;
    rtProgDesc.addShaderLibrary("DXRShadows.rt.hlsl").setRayGen("rayGen");
    rtProgDesc.addHitGroup(0, "primaryClosestHit", "").addMiss(0, "primaryMiss");
    rtProgDesc.addHitGroup(1, "", "shadowAnyHit").addMiss(1, "shadowMiss");

    // 3. 创建各通道着色器
    // 4. 加载泊松圆盘采样纹理 (1D, 6048 texels, RG32Float)
    mpPoisson1to32sppSamples = Texture::create1D(6048, ResourceFormat::RG32Float, ...);

    // 5. 创建深度/混合/光栅化状态
    // 6. 初始化全屏通道 (运动向量、滤波、调试)
    // 7. 设置初始 shader defines
}
```

关键点：RT 程序定义了两组 Hit Group：
- **Hit Group 0** (主光线)：`primaryClosestHit` + miss，用于 Primary Ray 模式求交
- **Hit Group 1** (阴影光线)：`shadowAnyHit` + miss，用于阴影检测（Any Hit 即可终止）

### renderScene() -- 渲染流程编排

```cpp
void DXRShadows::renderScene(RenderContext* pContext, Fbo::SharedPtr pTargetFbo)
{
    // 阶段1: 深度/法线获取
    if (GBuffer模式) GBufferPass(pContext);
    else { depthPass(pContext); motionVectorsPass(pContext, pTargetFbo); }

    // 阶段2: 天空盒
    renderSkyBox(pContext, pTargetFbo);

    // 阶段3: 阴影计算
    if (ShadowMapping) shadowMappingPass(pContext);
    if (RayTracing) {
        for (int pass = 0; pass < getPassesPerSceneNum(); pass++) {
            rayTracedShadowsPass(pContext, pTargetFbo, pass);
            visibilityFilteringPass(pContext, pTargetFbo, pass);
        }
    }

    // 阶段4: 最终光照
    lightingPass(pContext, pTargetFbo);

    // 阶段5: 调试 (可选)
    debugVisualizationsPass(pContext, pTargetFbo);
}
```

### rayTracedShadowsPass() -- DXR 阴影核心

此函数设置 RT 常量缓冲区、输入/输出纹理、执行光线追踪，然后触发变化度滤波：

1. 设置投影参数：`invView`, `view`, `viewportDims`, `tanHalfFovY`, `screenUnprojection`
2. 设置采样参数：`lightSamplesCount`, `maxLightSamplesCount`, `variationThreshold`
3. 双缓冲选择：根据 `mFrameNumber % 2` 选择输入/输出可见性缓存
4. 绑定 UAV 输出纹理（每光源一个可见性 + 一个变化度缓冲）
5. 执行光追（首帧调用 `renderScene`，之后直接调用 `raytrace`）
6. 触发变化度滤波（先 MAX 再 AVERAGE，两次水平+垂直通道）

### onResizeSwapChain() -- 资源重建

当窗口大小或光源数量变化时，重新创建所有纹理和 FBO：

- 每光源创建奇偶帧可见性缓存 (RGBA16Float)
- 每光源创建变化度 + 采样数缓存 (RGBA16Float, 4 mip levels)
- 运动向量缓冲 (RGBA32Float)
- 法线缓冲 (RGBA32Float)
- 滤波中间缓冲区

### 光源批处理辅助函数

```cpp
int getPassesPerSceneNum() {
    return ((lightsInScene - 1) / MAX_LIGHTS_PER_PASS) + 1;  // 向上取整
}
int getPackedTexturesPerPassNum(int passNum) {
    return (lightsPerPass - 1) / 4 + 1;  // 每4个光源打包为1个RGBA纹理
}
```

## 关键算法深入分析

### 变化度双通道滤波策略

对变化度缓冲连续执行两次滤波：
1. **最大值分布** (`MAXIMUM`)：将局部最大变化度扩散到邻近像素，确保高变化度区域的采样不会过早降低
2. **均值模糊** (`AVERAGE`)：平滑变化度场，防止自适应采样在相邻像素间剧烈跳变

每次滤波均为两通道分离式 (Separable)：先水平、再垂直。

### RT 变量设置优化

```cpp
if (!mRtShadowsPass.varsAlreadySetup) {
    mpRaytracingRenderer->renderScene(...);  // 完整设置（仅首次）
    mRtShadowsPass.varsAlreadySetup = true;
} else {
    mRtShadowsPass.pVars->apply(...);       // 快速应用
    pContext->raytrace(...);                  // 直接发射光线
}
```

这个优化跳过了 Falcor 框架中的场景遍历和完整变量绑定流程。

## 输入与输出

- **输入**: Falcor `.fscene` 场景文件（默认 `Arcade/Arcade.fscene`），命令行可指定
- **输出**: 渲染到屏幕的实时阴影画面

## 与其他文件的关系

- 包含 `dxrShadows.h`（类声明）和 `samplesData.h`（采样数据）
- 引用全部 HLSL 着色器文件作为 RT/图形程序
- 依赖 Falcor 框架的 `RtScene`, `RtProgram`, `RtState`, `CascadedShadowMaps` 等

## 在光线追踪管线中的位置

这是 CPU 端的"调度中心"，负责编排所有 GPU 工作，设置每个通道的输入/输出，管理双缓冲状态。

## 技术要点与注意事项

1. 光源数量上限为 32（超出部分会被删除）
2. 非自适应模式下采样数量按 4 的倍数对齐（9-11→12, 13-15→16 等）
3. `WinMain` 支持命令行参数指定初始场景
4. 双缓冲 FBO 通过帧号奇偶选择，确保前帧数据在当前帧可读
5. CSM 阴影贴图使用 EVSM4 滤波模式 + 4 级级联

## 扩展阅读

- Ray Tracing Gems, Chapter 13
- Falcor 框架文档: https://github.com/NVIDIAGameWorks/Falcor
- DXR Helper Library (DXR Fallback Layer)
