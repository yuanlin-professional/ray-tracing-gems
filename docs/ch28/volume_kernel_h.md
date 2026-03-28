# volume_kernel.h 技术文档

## 文件概述

**文件路径**: `Ch_28_Ray_Tracing_Inhomogeneous_Volumes/volume_kernel.h`

本文件是 CUDA 体积路径追踪内核的**头文件 (Header File)**，定义了内核参数结构体 `Kernel_params` 和内核函数的声明。它是 CPU 端（`main.cpp`）与 GPU 端（`volume_kernel.cu`）之间的**接口契约 (Interface Contract)**，确保两端对数据布局和函数签名的理解一致。

## 算法与数学背景

### 针孔相机模型

`Kernel_params` 中的相机参数定义了一个针孔相机 (Pinhole Camera)：

给定像素坐标 $(px, py)$，射线方向为：

$$\mathbf{d} = \text{normalize}(\mathbf{cam\_dir} \cdot f + \mathbf{cam\_right} \cdot px + \mathbf{cam\_up} \cdot py \cdot \text{aspect})$$

其中 $f$ 为焦距 (`cam_focal`)，`aspect` 为宽高比。

### Delta 追踪参数

- `max_extinction`：$\sigma_{max}$，Delta 追踪的上界消光系数
- `albedo`：$\rho = \sigma_s / \sigma_t$，单次散射反照率

## 代码结构概览

由于本文件较短（<30 行有效代码），以下提供完整源码及逐行注释。

## 逐段代码详解

```cpp
// 第1-3行：文件注释
//
// CUDA volume path tracing kernel interface
//
```

### Kernel_params 结构体

```cpp
struct Kernel_params {
    // Display
    uint2 resolution;               // 渲染分辨率 (宽, 高)
    float exposure_scale;           // 曝光缩放因子 (2^exposure)
    unsigned int *display_buffer;   // LDR 显示缓冲区指针 (BGRA8 格式)
```

**显示参数组**：
- `resolution`：像素维度，用于线程索引边界检查和 UV 计算
- `exposure_scale`：用户可调的曝光值，以 2 的幂次缩放
- `display_buffer`：CUDA-GL 互操作的共享缓冲区，每像素 32 位

```cpp
    // Progressive rendering state
    unsigned int iteration;         // 当前渲染帧编号 (从 0 开始)
    float3 *accum_buffer;           // HDR 累积缓冲区指针
```

**渐进式渲染状态**：
- `iteration`：帧计数器，用于计算运行平均 $\bar{C}_n = \bar{C}_{n-1} + (C_n - \bar{C}_{n-1})/(n+1)$
- `accum_buffer`：每像素 `float3`，存储所有历史帧的 HDR 累积均值

```cpp
    // Limit on path length
    unsigned int max_interactions;  // 路径最大散射次数 (如 1024)
```

**路径长度限制**：防止高反照率、密集介质中的无限循环。

```cpp
    // Camera
    float3 cam_pos;                 // 相机位置 (世界空间)
    float3 cam_dir;                 // 相机朝向 (单位向量)
    float3 cam_right;               // 相机右向量 (单位向量)
    float3 cam_up;                  // 相机上向量 (单位向量)
    float  cam_focal;               // 相机焦距
```

**相机参数组**：
- `cam_pos`：相机在世界空间的位置
- `cam_dir`：相机看向的方向
- `cam_right`, `cam_up`：构成相机的正交坐标系
- `cam_focal`：焦距，$f = 1 / \tan(\text{FOV}/2)$。对于 90 度 FOV，$f = 1.0$

```cpp
    // Environment
    unsigned int environment_type;  // 0 = 渐变天空, 1 = HDR 环境贴图
    cudaTextureObject_t env_tex;    // 环境贴图的 CUDA 纹理对象
```

**环境光照参数**：
- 类型 0 是简单的 Y 方向灰度渐变
- 类型 1 使用通过 `hdr_loader.h` 加载的 HDR 环境贴图

```cpp
    // Volume definition
    unsigned int volume_type;       // 0 = Menger 海绵, 1 = 螺旋管
    float max_extinction;           // 最大消光系数 σ_max
    float albedo; // sigma / kappa  // 散射反照率 ρ = σ_s / σ_t
};
```

**体积定义参数**：
- `volume_type`：选择程序化体积函数
- `max_extinction`：Delta 追踪的关键参数，必须 >= 体积中实际的最大消光系数
- `albedo`：注释中的 "sigma / kappa" 表示散射系数与消光系数之比

### 内核函数声明

```cpp
extern "C" __global__ void volume_rt_kernel(
   const Kernel_params kernel_params);
```

- `extern "C"`：禁止 C++ 名称修饰 (Name Mangling)，使得 CPU 端可以通过 `cudaLaunchKernel()` 查找函数符号
- `__global__`：CUDA 内核函数标记，可从 CPU 端调用，在 GPU 端执行
- `const Kernel_params kernel_params`：按值传递参数结构体（CUDA 内核参数通过常量内存传递，效率高）

## 关键算法深入分析

### 参数传递策略

CUDA 内核参数通过**常量内存 (Constant Memory)** 传递，有以下约束：
- 总参数大小限制为 4KB（对于大多数 GPU）
- 常量内存有缓存，同一 warp 内的线程读取同一参数几乎无开销

`Kernel_params` 结构体的内存布局分析：

| 字段 | 类型 | 大小（字节） |
|------|------|------------|
| `resolution` | `uint2` | 8 |
| `exposure_scale` | `float` | 4 |
| `display_buffer` | `uint*` | 8（64位指针） |
| `iteration` | `uint` | 4 |
| `accum_buffer` | `float3*` | 8 |
| `max_interactions` | `uint` | 4 |
| `cam_pos` | `float3` | 12 |
| `cam_dir` | `float3` | 12 |
| `cam_right` | `float3` | 12 |
| `cam_up` | `float3` | 12 |
| `cam_focal` | `float` | 4 |
| `environment_type` | `uint` | 4 |
| `env_tex` | `cudaTextureObject_t` | 8 |
| `volume_type` | `uint` | 4 |
| `max_extinction` | `float` | 4 |
| `albedo` | `float` | 4 |
| **总计** | | **约 112 字节** |

远小于 4KB 限制，适合按值传递。

### max_extinction 的物理意义

$\sigma_{max}$ 是 Delta 追踪的**虚拟均匀介质密度**。它决定了：
- **采样步长**：平均步长 $\bar{t} = 1/\sigma_{max}$
- **接受率**：平均接受率 $\bar{p} = \bar{\sigma}_t / \sigma_{max}$

如果 $\sigma_{max}$ 设得过大：
- 采样步长过小，需要更多步骤才能穿过体积
- 大量空碰撞被拒绝，效率降低

如果 $\sigma_{max}$ 设得过小（< 实际最大值）：
- **结果错误**——密度超过 $\sigma_{max}$ 的区域被截断

## 输入与输出

本文件为头文件，不直接处理 I/O。它定义的接口被以下文件使用：

### 被包含者
| 文件 | 用途 |
|------|------|
| `main.cpp` | 初始化和设置 `Kernel_params` |
| `volume_kernel.cu` | 读取 `Kernel_params` 执行渲染 |

## 与其他文件的关系

```
volume_kernel.h ←──── 定义接口 ────→ volume_kernel.cu
       ↑                                    ↑
       │                                    │
  main.cpp ─── 设置参数 ─── cudaLaunchKernel() ───→ 调用内核
       │
       └── hdr_loader.h (加载环境贴图到 env_tex)
```

## 在光线追踪管线中的位置

```
                    ┌───────────────────────┐
                    │   volume_kernel.h     │
                    │   (接口定义)           │
                    └───────┬───────────────┘
                            │
               ┌────────────┼────────────┐
               │            │            │
               v            v            v
          main.cpp    volume_kernel.cu   (未来扩展)
         (CPU 端)      (GPU 端)
```

作为头文件，它是整个系统的**中央接口定义**。

## 技术要点与注意事项

1. **内存对齐**：CUDA 结构体可能因对齐规则产生填充字节 (Padding)。`float3` 在 CUDA 中按 4 字节对齐（不是 16 字节），因此结构体中 `float3` 后面可能有填充
2. **`extern "C"` 的必要性**：不加此限定符，C++ 编译器会修饰函数名，导致 `cudaLaunchKernel()` 无法找到内核
3. **按值传递 vs 指针传递**：内核参数按值传递到常量内存。缓冲区通过指针间接访问全局内存
4. **`cudaTextureObject_t`**：这是一个 64 位的不透明句柄 (Opaque Handle)，封装了纹理的所有属性
5. **albedo 的注释**："sigma / kappa" 中的 sigma 指散射系数 $\sigma_s$，kappa 指消光系数 $\sigma_t = \sigma_s + \sigma_a$
6. **线程安全**：`display_buffer` 和 `accum_buffer` 的写入是线程安全的，因为每个线程写不同的像素

## 扩展阅读

- CUDA C Programming Guide: Kernel Parameters and Constant Memory
- NVIDIA CUDA Runtime API: `cudaTextureObject_t`
- Novak, J. et al. *Monte Carlo Methods for Volumetric Light Transport Simulation*. 2018
