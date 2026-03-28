# 第 13 章 -- 光线追踪阴影：保持实时帧率

## 模块用途

本章实现了一套完整的 **DXR 实时光线追踪阴影** (Ray Traced Shadows) 管线，支持多光源、自适应采样 (Adaptive Sampling)、时域滤波 (Temporal Filtering) 和空间降噪 (Spatial Denoising)。其核心目标是在保持实时帧率的前提下，生成高质量的软阴影效果。

系统基于 NVIDIA **Falcor** 实时渲染框架构建，使用 DirectX Raytracing (DXR) API 进行阴影光线追踪，并通过多通道 (Multi-Pass) 管线对原始阴影结果进行过滤和降噪。

## 文件列表与职责

| 文件 | 类型 | 职责 |
|------|------|------|
| `dxrShadows.h` | C++ 头文件 | 主应用类 `DXRShadows` 声明，包含全部渲染通道上下文、状态及枚举定义 |
| `dxrShadows.cpp` | C++ 源文件 | 主应用逻辑：场景加载、多通道渲染流程编排、GUI 交互 |
| `samplesData.h` | C++ 头文件 | 泊松圆盘 (Poisson Disk) 采样数据接口声明 |
| `samplesData.cpp` | C++ 源文件 | 预计算的 1-32 SPP 泊松圆盘采样点数据 (6048 个 float2) |
| `shaderCommon.h` | C/HLSL 共享头 | CPU/GPU 共享常量、光源类型枚举、`ShadowsLightInfo` 结构体 |
| `CommonUtils.hlsl` | HLSL 工具库 | GPU 端通用工具函数：深度重建、法线计算、可见性历史、数据打包 |
| `DXRShadows.rt.hlsl` | HLSL 光追着色器 | 核心 RT 着色器：ray generation、hit/miss 程序、自适应采样、多光源阴影 |
| `DepthPass.ps.hlsl` | HLSL 像素着色器 | 深度预通道 (Depth Pre-Pass)，仅写入深度 |
| `DepthAndNormalsPass.ps.hlsl` | HLSL 像素着色器 | G-Buffer 通道，输出运动向量 + 线性深度 + 视空间法线 |
| `MotionVectors.ps.hlsl` | HLSL 像素着色器 | 全屏运动向量生成，基于深度缓冲重投影 |
| `VariationFilteringPass.ps.hlsl` | HLSL 像素着色器 | 变化度缓冲区滤波 (最大值分布 + 均值模糊) |
| `VisibilityFilteringPass.ps.hlsl` | HLSL 像素着色器 | 可见性缓冲区高斯滤波降噪，支持深度/法线感知边缘保持 |
| `ForwardRenderer.ps.hlsl` | HLSL 像素着色器 | 最终光照通道，结合阴影因子进行 BRDF 着色 |
| `DebugPass.ps.hlsl` | HLSL 像素着色器 | 调试可视化通道：深度、法线、运动向量、变化度、采样数等 |

## 架构关系

```
┌─────────────────────────────────────────────────────────────────┐
│                        DXRShadows 主循环                         │
│  onFrameRender()                                                │
│       │                                                         │
│       ├── updateLightBuffer()         // 更新光源信息            │
│       │                                                         │
│       ├── renderScene()                                         │
│       │    ├── GBufferPass() / depthPass()   // 深度 + 法线     │
│       │    ├── motionVectorsPass()            // 运动向量        │
│       │    ├── renderSkyBox()                 // 天空盒          │
│       │    │                                                    │
│       │    ├── [Shadow Pass 选择]                                │
│       │    │   ├── shadowMappingPass()        // CSM 阴影贴图   │
│       │    │   └── for each 8-light pass:                       │
│       │    │       ├── rayTracedShadowsPass() // DXR 光追阴影   │
│       │    │       ├── variationFilteringPass() // 变化度滤波    │
│       │    │       └── visibilityFilteringPass() // 可见性降噪   │
│       │    │                                                    │
│       │    ├── lightingPass()                 // 最终光照        │
│       │    └── debugVisualizationsPass()      // 调试输出        │
│       │                                                         │
│       └── 存储当前帧 ViewProjection 矩阵 → 下帧使用             │
└─────────────────────────────────────────────────────────────────┘
```

### 数据流

```
深度缓冲 (Depth)
    │
    ├──→ 运动向量缓冲 (Motion Vectors) ──→ DXR 光追通道
    │                                         │
    │                                    ┌────┴─────┐
    │                              可见性缓冲   变化度缓冲
    │                              (per-light)  (per-light)
    │                                    │          │
    │                              ┌─────┴──┐  变化度滤波
    │                              │ 时域    │  (Max + Avg)
    │                              │ 历史    │      │
    │                              │ 混合    │  ┌───┴───┐
    │                              └────┬───┘  │ Mipmap │
    │                                   │      └───┬───┘
    │                              可见性滤波       │
    │                              (深度/法线感知)  │
    │                                   │      自适应采样
    │                                   ▼      反馈控制
法线缓冲 (Normals) ──→ 最终光照通道 (Lighting)
                              │
                              ▼
                         渲染输出
```

### 关键设计决策

1. **每通道最多处理 8 个光源** (`MAX_LIGHTS_PER_PASS`)：每 4 个光源的可见性打包进一个 RGBA 纹理，减少纹理绑定数量
2. **双缓冲乒乓策略** (Ping-Pong)：奇偶帧交替使用不同的缓冲区集合，支持时域重投影
3. **采样矩阵剔除** (Sampling Matrix Culling)：对变化度为零的区域，按 $4 \times 4$ 棋盘格模式跳过采样
4. **自适应采样**：基于历史变化度自动调节每像素采样数 (1-8 SPP)

## API 概要

### 主类 `DXRShadows`

继承自 `Falcor::Renderer`，实现以下 Falcor 回调：

| 方法 | 功能 |
|------|------|
| `onLoad()` | 初始化 RT 程序、光栅化通道、全屏四边形通道 |
| `onFrameRender()` | 每帧主渲染逻辑 |
| `onResizeSwapChain()` | 交换链大小变化时重建 FBO 和纹理 |
| `onGuiRender()` | ImGui 界面绘制 |
| `loadScene()` | 加载 Falcor 场景文件 (.fscene) |

### 着色器入口

| 着色器 | 入口函数 |
|--------|----------|
| `DXRShadows.rt.hlsl` | `rayGen`, `primaryClosestHit`, `primaryMiss`, `shadowAnyHit`, `shadowMiss` |
| `ForwardRenderer.ps.hlsl` | `vs`, `ps` |
| 其他全屏通道 | `main` |

### 共享常量 (`shaderCommon.h`)

- `MAX_LIGHTS_PER_PASS = 8` -- 单通道最大光源数
- `INVALID_LIGHT_INDEX = 65535` -- 表示"全部光源"
- 光源类型: `POINT_LIGHT(1)`, `SPHERICAL_LIGHT(2)`, `DIRECTIONAL_HARD_LIGHT(3)`, `DIRECTIONAL_SOFT_LIGHT(4)`
