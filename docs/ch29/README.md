# 第 29 章 -- 光线追踪器中的高效粒子体渲染

## 模块用途

本章实现了一个基于 **NVIDIA OptiX** 光线追踪引擎的粒子体渲染器 (Particle Volume Renderer)。它使用**径向基函数** (Radial Basis Function, RBF) 对大规模粒子数据进行体积溅射 (Volume Splatting)，通过 BVH 加速结构高效收集光线路径上的粒子，并以前-后排序 (Front-to-Back Sorting) 进行 alpha 合成积分。

核心创新是 **Slab-based 分段遍历** 策略：将光线行进路径分割为多个"厚板" (Slab)，每个厚板内使用固定大小的缓冲区收集粒子，排序后积分，从而在有限 GPU 内存下处理任意数量的粒子。

## 文件列表与职责

### 核心应用 (optixParticleVolumes/)

| 文件 | 类型 | 职责 |
|------|------|------|
| `optixParticleVolumes.cpp` | C++ 主程序 | OptiX 上下文初始化、粒子数据加载、GLFW 窗口管理、ImGui 交互 |
| `commonStructs.h` | C/CUDA 共享头 | `BasicLight`, `PerRayData` 结构体定义，粒子缓冲区大小常量 |
| `raygen.cu` | CUDA 光线生成 | 射线生成、AABB 求交、Slab 分段遍历、粒子排序与积分 |
| `geometry.cu` | CUDA 自定义几何 | 粒子球体求交 (Intersection) 和包围盒 (Bounding Box) 程序 |
| `material.cu` | CUDA 材质 | Any Hit 程序：收集粒子到 per-ray 缓冲区 |
| `constantbg.cu` | CUDA 背景 | Miss 程序：返回恒定背景颜色 |
| `transferFunction.h` | CUDA 设备头 | 传递函数 (Transfer Function) 实现，将标量值映射为 RGBA 颜色 |

### 伪代码 / 书中代码片段

| 文件 | 类型 | 职责 |
|------|------|------|
| `RaygenProgram.cpp` | 伪代码 | 书中列出的简化版 Ray Generation 伪代码 |
| `ParticleIntersect.cpp` | 伪代码 | 书中列出的简化版粒子求交 + Any Hit 伪代码 |

### 设备端公共库 (device_include/)

| 文件 | 类型 | 职责 |
|------|------|------|
| `commonStructs.h` | C/CUDA 共享头 | `BasicLight`, `DirectionalLight`, `ParticleSample`, `PerRayData` 定义 |
| `helpers.h` | CUDA 设备工具 | 颜色转换、Phong 采样、ONB 构建、射线微分、色调映射 |
| `intersection_refinement.h` | CUDA 设备工具 | 交点精化和偏移（避免自交叉） |
| `random.h` | CUDA 设备工具 | TEA 哈希、LCG 伪随机数、Multiply-With-Carry 随机数 |

## 架构关系

```
┌────────────────────────────────────────────────────────────┐
│              optixParticleVolumes.cpp (CPU Host)           │
│                                                            │
│  main() ──→ createContext() ──→ setupParticles()           │
│              ──→ loadParticles() ──→ setupCamera()         │
│              ──→ setupLights() ──→ glfwRun()               │
│                                                            │
│  glfwRun() 循环:                                            │
│    camera.process_mouse() → context->launch(0, w, h)       │
│    → displayBufferGL() → ImGui::Render()                   │
└─────────────────────────┬──────────────────────────────────┘
                          │ OptiX launch
                          ▼
┌────────────────────────────────────────────────────────────┐
│              GPU Device Programs                           │
│                                                            │
│  raygen.cu::raygen_program()                               │
│    │                                                       │
│    ├── Ray-AABB 求交（确定遍历范围）                         │
│    │                                                       │
│    ├── for each slab:                                      │
│    │    ├── rtTrace() ──→ geometry.cu::particle_intersect() │
│    │    │                  └──→ material.cu::any_hit()      │
│    │    │                       (收集到 prd.particles[])    │
│    │    ├── sort(prd.particles)  (冒泡/双调排序)            │
│    │    └── integrate (RBF + Transfer Function + Alpha)     │
│    │         └── transferFunction.h::tf()                  │
│    │                                                       │
│    └── miss → constantbg.cu::miss()                        │
│                                                            │
│  输出 → output_buffer[launch_index]                        │
└────────────────────────────────────────────────────────────┘
```

### 数据流

```
粒子文件 (.raw / .txt)
    │
    ▼
CPU: readFile() → positions[], velocities[], colors[], radii[]
    │
    ▼
OptiX Buffers: positions_buffer (float4: xyz + attribute w)
    │
    ▼
GPU BVH: Bvh8 加速结构 (per-particle 球形包围盒)
    │
    ▼
raygen_program():
    Ray-AABB → Slab 分段 → rtTrace() per slab
                                │
                          particle_intersect()
                          any_hit() → prd.particles[]
                                │
                          Sort → Integrate
                                │
                          tf() → alpha compositing
                                │
                                ▼
                         output_buffer (uchar4 BGRA)
```

## API 概要

### OptiX 程序入口

| 程序 | 文件 | 入口 |
|------|------|------|
| Ray Generation | `raygen.cu` | `raygen_program` |
| Exception | `raygen.cu` | `exception` |
| Miss | `constantbg.cu` | `miss` |
| Intersection | `geometry.cu` | `particle_intersect` |
| Bounding Box | `geometry.cu` | `particle_bounds` |
| Any Hit | `material.cu` | `any_hit` |
| Closest Hit | `material.cu` | `closest_hit` (空实现) |

### 关键参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `fixed_radius` | 100.0 | 粒子世界空间半径 |
| `particlesPerSlab` | 16.0 | 每厚板期望粒子数密度因子 |
| `wScale` | 3.5 | 标量属性缩放因子 |
| `opacity` | 0.5 | 全局不透明度缩放 |
| `tf_type` | 2 | 传递函数预设 (1-3) |
| `PARTICLE_BUFFER_SIZE` | 32 | 每条光线的粒子缓冲区大小 |

### 关键公式

**RBF 评估**:
$$d_{\text{rbf}} = \frac{|\mathbf{c} - \mathbf{p}_{\text{hit}}|}{r/2}$$
$$v = \text{clamp}(w_{\text{scale}} \cdot w \cdot e^{-d_{\text{rbf}}^2}, 0, 1)$$

**Alpha 合成 (Front-to-Back)**:
$$\alpha_{\text{new}} = \alpha_{\text{sample}} \cdot (1 - \alpha_{\text{accum}})$$
$$C_{\text{accum}} += C_{\text{sample}} \cdot \alpha_{\text{new}}$$
$$\alpha_{\text{accum}} += \alpha_{\text{new}}$$
