# 第28章：非均匀体积的光线追踪 (Ray Tracing Inhomogeneous Volumes)

## 章节概述

本章介绍了使用 **CUDA 路径追踪 (Path Tracing)** 渲染**非均匀参与介质 (Inhomogeneous Participating Media)** 的完整实现。参与介质是指光线在传播过程中会被散射 (Scattering) 和吸收 (Absorption) 的介质，如烟雾、云朵、半透明材料等。非均匀意味着介质的密度在空间中不是常数，而是随位置变化。

本章提供了一个完整的、可运行的交互式应用程序，使用 CUDA 进行体积路径追踪，OpenGL 进行显示，支持环境贴图光照和渐进式采样累积。

## 核心算法：Delta 追踪 (Delta Tracking)

也称为 **Woodcock 追踪** 或**空碰撞方法 (Null-Collision Method)**，是处理非均匀介质散射的标准技术。核心思想是：

1. 选取介质中的最大消光系数 $\sigma_{max}$
2. 以 $\sigma_{max}$ 作为均匀采样步长生成候选交互点
3. 在候选点处，以概率 $\sigma(\mathbf{x}) / \sigma_{max}$ 接受该交互
4. 如果拒绝，继续采样下一个候选点

这等价于引入"空碰撞"使介质在统计上变为均匀介质，从而可以使用指数距离采样。

## 文件列表

| 文件 | 说明 | 文档链接 |
|------|------|----------|
| `hdr_loader.h` | HDR 环境贴图加载器 | [hdr_loader_h.md](hdr_loader_h.md) |
| `main.cpp` | 主程序：OpenGL 显示和 CUDA 交互 | [main_cpp.md](main_cpp.md) |
| `volume_kernel.cu` | CUDA 体积路径追踪内核 | [volume_kernel_cu.md](volume_kernel_cu.md) |
| `volume_kernel.h` | 内核参数结构体和接口声明 | [volume_kernel_h.md](volume_kernel_h.md) |

## 技术架构

```
┌────────────────────────────────────────────────────────┐
│                    main.cpp (CPU 端)                    │
│                                                         │
│  ┌───────────┐  ┌──────────────┐  ┌─────────────────┐ │
│  │ OpenGL    │  │ CUDA-GL      │  │ 环境贴图加载    │ │
│  │ 窗口/渲染 │  │ 互操作       │  │ (hdr_loader.h)  │ │
│  └───────────┘  └──────────────┘  └─────────────────┘ │
│         │              │                    │           │
│         v              v                    v           │
│  ┌──────────────────────────────────────────────┐     │
│  │           Kernel_params (volume_kernel.h)     │     │
│  │  相机、分辨率、体积参数、环境贴图...            │     │
│  └──────────────┬───────────────────────────────┘     │
│                 │                                       │
└─────────────────┼───────────────────────────────────────┘
                  │
                  v
┌────────────────────────────────────────────────────────┐
│              volume_kernel.cu (GPU 端)                   │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ 射线-盒交点  │→ │ Delta 追踪   │→ │ 环境光采样   │ │
│  │ 计算         │  │ (散射模拟)   │  │              │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐                    │
│  │ 渐进式累积   │→ │ Reinhard     │ → 显示缓冲区      │
│  │              │  │ 色调映射     │                    │
│  └──────────────┘  └──────────────┘                    │
└────────────────────────────────────────────────────────┘
```

## 关键技术

1. **Delta 追踪 (Delta Tracking)**：处理非均匀消光系数的无偏采样方法
2. **俄罗斯轮盘赌 (Russian Roulette)**：在路径权重过低时概率性终止路径，保持无偏性
3. **各向同性相函数 (Isotropic Phase Function)**：体积散射方向均匀分布在球面上
4. **渐进式渲染 (Progressive Rendering)**：逐帧累积采样结果，渐进收敛到正确结果
5. **Reinhard 色调映射 (Tone Mapping)**：将 HDR 颜色映射到可显示的 LDR 范围
6. **CUDA-OpenGL 互操作**：GPU 渲染结果直接传递给 OpenGL 显示，避免 CPU 回传

## 依赖项

- CUDA Runtime
- OpenGL 3.3+
- GLEW
- GLFW3
- cuRAND（CUDA 随机数库）

## 参考文献

- Woodcock, E.R. et al. *Techniques Used in the GEM Code for Monte Carlo Neutronics Calculations*. 1965
- Novak, J. et al. *Monte Carlo Methods for Volumetric Light Transport Simulation*. Computer Graphics Forum, 2018
- Pharr, M., Jakob, W., Humphreys, G. *PBRT*, Chapter 15 - Volume Scattering
