# dxrShadows.h 技术文档

## 文件概述

`dxrShadows.h` 是 DXR 实时阴影示例的核心头文件，声明了主应用类 `DXRShadows` 及其全部辅助结构体和枚举。该类继承自 Falcor 框架的 `Renderer` 基类，管理完整的多通道阴影渲染管线。

**源文件路径**: `Ch_13_Ray_Traced_Shadows_Maintaining_Real-Time_Frame_Rates/dxrShadows/dxrShadows.h`

## 算法与数学背景

本文件本身不包含算法实现，但定义了管线中所有通道的参数和状态。关键数学参数包括：

- **最大重投影深度差** $\delta_{\text{reproj}} = 0.003$：用于判断时域重投影 (Temporal Reprojection) 是否有效
- **滤波深度差容差** $\epsilon_{\text{min}} = 0.003$, $\epsilon_{\text{max}} = 0.02$：空间滤波中边缘检测的深度阈值
- **法线点积阈值** $\theta_{\text{norm}} = 0.9$：用于法线感知的边缘保持滤波
- **目标变化度** $v_{\text{target}} = 0.1$：自适应采样的收敛目标

## 代码结构概览

```
dxrShadows.h
├── 辅助结构体
│   ├── FullScreenPassContext    // 全屏通道上下文
│   ├── RasterizationPassContext // 光栅化通道上下文
│   ├── RaytracingPassContext    // 光追通道上下文
│   ├── ShadowMappingPass        // 阴影贴图通道状态
│   └── SkyBoxPass               // 天空盒通道状态
├── 枚举类型
│   ├── Filtering                // 滤波类型
│   ├── DebugVisualizations      // 调试可视化模式
│   ├── ShadowsAlgorithm         // 阴影算法选择
│   └── DepthNormalsSourceType   // 深度/法线来源
└── DXRShadows 类
    ├── Falcor 回调 (public)
    └── 私有成员 (按通道分组)
```

## 逐段代码详解

### 辅助结构体

```cpp
// 全屏通道 (Full-Screen Pass) 上下文：将 GraphicsVars 与 FullScreenPass 绑定
struct FullScreenPassContext {
    GraphicsVars::SharedPtr pVars;
    FullScreenPass::UniquePtr pPass;
};

// 标准光栅化通道上下文
struct RasterizationPassContext {
    GraphicsVars::SharedPtr pVars;
    GraphicsProgram::SharedPtr pProgram;
};

// 光追通道上下文：增加 varsAlreadySetup 标志用于优化
struct RaytracingPassContext {
    RtProgramVars::SharedPtr pVars;
    RtProgram::SharedPtr pProgram;
    bool varsAlreadySetup = false;  // 避免每帧重复设置 RT 变量
};
```

### 枚举类型

```cpp
enum class ShadowsAlgorithm {
    RayTracing = 1,     // DXR 光线追踪阴影
    ShadowMapping = 2,  // 级联阴影贴图 (CSM)
    Off = 3             // 关闭阴影
};

enum class DepthNormalsSourceType {
    DepthPrePass = 1,   // 纯深度预通道 + 从深度重建法线
    GBufferPass = 2,    // G-Buffer 通道直接输出法线
    PrimaryRays = 3     // 使用主光线求交获取位置
};
```

### DXRShadows 类关键成员

**双缓冲 FBO 系统**：
```cpp
Fbo::SharedPtr mpEvenDepthPassFbo;  // 偶数帧深度
Fbo::SharedPtr mpOddDepthPassFbo;   // 奇数帧深度
Fbo::SharedPtr mpEvenGBufferPassFbo;
Fbo::SharedPtr mpOddGBufferPassFbo;
```

**可见性缓冲区 (每光源一个)**：
```cpp
std::vector<Texture::SharedPtr> mRtVisibilityCacheOdd;   // 奇数帧缓存
std::vector<Texture::SharedPtr> mRtVisibilityCacheEven;  // 偶数帧缓存
std::vector<Texture::SharedPtr> mRtOutputFilteredVisibilityBuffer; // 滤波后输出
std::vector<Texture::SharedPtr> mOutputVariationAndSampleCountCache; // 变化度 + 采样数
```

## 关键算法深入分析

### 多通道光源批处理

每个 RT 通道最多处理 8 个光源。当场景中有 $N$ 个光源时，需要 $\lceil N/8 \rceil$ 个通道：

$$\text{passes} = \left\lfloor \frac{N-1}{8} \right\rfloor + 1$$

每 4 个光源的可见性值打包到一个 RGBA 纹理的四个通道中，因此 8 个光源需要 2 个纹理。

## 输入与输出

- **输入**: Falcor `.fscene` 场景文件
- **输出**: 带阴影的实时渲染画面

## 与其他文件的关系

- 被 `dxrShadows.cpp` 包含，实现所有声明的方法
- 包含 `shaderCommon.h`，共享 CPU/GPU 常量
- 包含 `Falcor.h` 和 `FalcorExperimental.h`，依赖 Falcor 框架

## 在光线追踪管线中的位置

此文件定义了整个渲染管线的"骨架"--全部通道的状态和参数。它是 CPU 端控制逻辑的核心声明。

## 技术要点与注意事项

1. `varsAlreadySetup` 优化：首次渲染后跳过 Falcor RT 变量的完整设置，直接调用 `pContext->raytrace()`
2. 每光源独立的可见性缓冲区使得滤波和自适应采样可以独立控制
3. 最大支持 32 个光源（受采样数据和缓冲区限制）

## 扩展阅读

- Ray Tracing Gems, Chapter 13: "Ray Traced Shadows: Maintaining Real-Time Frame Rates"
- NVIDIA Falcor 实时渲染框架文档
- DXR (DirectX Raytracing) 规范
