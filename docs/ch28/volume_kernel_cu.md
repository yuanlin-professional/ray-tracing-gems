# volume_kernel.cu 技术文档

## 文件概述

**文件路径**: `Ch_28_Ray_Tracing_Inhomogeneous_Volumes/volume_kernel.cu`

本文件是整个体积路径追踪系统的核心，实现了完整的 **CUDA 体积路径追踪内核 (Volume Path Tracing Kernel)**。它包含射线-包围盒求交、非均匀介质中的 **Delta 追踪 (Delta Tracking)** 散射采样、**各向同性相函数 (Isotropic Phase Function)** 散射方向采样、**俄罗斯轮盘赌 (Russian Roulette)** 路径终止、环境光查询、渐进式累积和色调映射等完整的体积渲染管线。

## 算法与数学背景

### 体积渲染方程 (Volume Rendering Equation)

光线在参与介质中的传播遵循辐射传输方程 (Radiative Transfer Equation, RTE)：

$$(\boldsymbol{\omega} \cdot \nabla) L(\mathbf{x}, \boldsymbol{\omega}) = -\sigma_t(\mathbf{x}) L(\mathbf{x}, \boldsymbol{\omega}) + \sigma_s(\mathbf{x}) \int_{S^2} f_p(\mathbf{x}, \boldsymbol{\omega}', \boldsymbol{\omega}) L(\mathbf{x}, \boldsymbol{\omega}') \, d\boldsymbol{\omega}'$$

其中：
- $\sigma_t(\mathbf{x})$：**消光系数 (Extinction Coefficient)**，= 吸收 + 散射
- $\sigma_s(\mathbf{x})$：**散射系数 (Scattering Coefficient)**
- $f_p$：**相函数 (Phase Function)**

**反照率 (Albedo)**：$\rho = \sigma_s / \sigma_t$，表示散射事件发生的概率。

### Delta 追踪 (Woodcock Tracking)

对于非均匀介质 $\sigma_t(\mathbf{x})$，定义虚拟均匀介质 $\sigma_{max} = \max_{\mathbf{x}} \sigma_t(\mathbf{x})$。

采样步骤：
1. 从指数分布采样距离：$t = -\ln(1 - \xi) / \sigma_{max}$
2. 在候选点 $\mathbf{x}' = \mathbf{x} + t\boldsymbol{\omega}$ 处，以概率 $\sigma_t(\mathbf{x}') / \sigma_{max}$ 接受
3. 拒绝则继续步骤 1

这是**接受-拒绝采样 (Rejection Sampling)** 的一种应用，通过引入"空碰撞"使采样过程与均匀介质等价。

### 俄罗斯轮盘赌

当路径权重 $w$ 降低到某个阈值 $w_{threshold}$ 以下时：
- 以概率 $q = 1 - 5w$ 终止路径
- 以概率 $1 - q = 5w$ 继续路径，权重重新设为 $w_{threshold} = 0.2$

这保持了估计器的**无偏性 (Unbiasedness)**：

$$E[\text{contribution}] = (1-q) \cdot \frac{w}{1-q} = w$$

### 各向同性相函数

均匀球面采样：

$$\boldsymbol{\omega} = (\cos\phi \sin\theta, \sin\phi \sin\theta, \cos\theta)$$

其中 $\phi = 2\pi\xi_1$，$\cos\theta = 1 - 2\xi_2$。

## 代码结构概览

```
volume_kernel.cu
├── 3D 向量数学运算符重载
├── intersect_volume_box()      // 射线-AABB 求交
├── in_volume()                 // 体积包含测试
├── get_extinction()            // 获取消光系数（体积密度函数）
├── sample_interaction()        // Delta 追踪采样
├── trace_volume()              // 完整的体积路径追踪
└── volume_rt_kernel()          // CUDA 入口内核
```

## 逐段代码详解

### 向量数学工具

```cuda
__device__ inline float3 operator+(const float3& a, const float3& b) { ... }
__device__ inline float3 operator-(const float3& a, const float3& b) { ... }
__device__ inline float3 operator*(const float3& a, const float s) { ... }
__device__ inline float3 operator/(const float3& a, const float s) { ... }
__device__ inline void operator+=(float3& a, const float3& b) { ... }
__device__ inline void operator*=(float3& a, const float& s) { ... }
__device__ inline float3 normalize(const float3 &d) { ... }
__device__ inline float dot(const float3 &u, const float3 &v) { ... }
```

CUDA 的 `float3` 类型没有内建运算符重载，需要手动实现。

### 随机数生成

```cuda
#include <curand_kernel.h>
typedef curandStatePhilox4_32_10_t Rand_state;
#define rand(state) curand_uniform(state)
```

使用 cuRAND 库的 **Philox 4x32-10** 伪随机数生成器，这是一种基于计数器的 PRNG，特别适合 GPU 并行环境。

### intersect_volume_box() - 射线-盒求交

```cuda
__device__ inline bool intersect_volume_box(
    float &tmin, const float3 &raypos, const float3 &raydir)
{
    const float x0 = (-0.5f - raypos.x) / raydir.x;
    const float y0 = (-0.5f - raypos.y) / raydir.y;
    const float z0 = (-0.5f - raypos.z) / raydir.z;
    const float x1 = ( 0.5f - raypos.x) / raydir.x;
    const float y1 = ( 0.5f - raypos.y) / raydir.y;
    const float z1 = ( 0.5f - raypos.z) / raydir.z;

    tmin = fmaxf(fmaxf(fmaxf(fminf(z0,z1), fminf(y0,y1)), fminf(x0,x1)), 0.0f);
    const float tmax = fminf(fminf(fmaxf(z0,z1), fmaxf(y0,y1)), fmaxf(x0,x1));
    return (tmin < tmax);
}
```

经典的 **Slab Method** 射线-AABB 求交算法：
- 包围盒范围：$[-0.5, 0.5]^3$（单位立方体）
- 对每个轴计算进入和离开的 $t$ 值
- $t_{min} = \max(t_{enter,x}, t_{enter,y}, t_{enter,z}, 0)$
- $t_{max} = \min(t_{exit,x}, t_{exit,y}, t_{exit,z})$
- 当 $t_{min} < t_{max}$ 时相交

### in_volume() - 体积包含测试

```cuda
__device__ inline bool in_volume(const float3 &pos)
{
    return fmaxf(fabsf(pos.x), fmaxf(fabsf(pos.y), fabsf(pos.z))) < 0.5f;
}
```

使用 $L^{\infty}$ 范数（切比雪夫距离）检查点是否在 $[-0.5, 0.5]^3$ 立方体内。

### get_extinction() - 消光系数查询

```cuda
__device__ inline float get_extinction(
    const Kernel_params &kernel_params, const float3 &p)
{
    if (kernel_params.volume_type == 0) {
        // Menger 海绵分形
        float3 pos = p + make_float3(0.5f, 0.5f, 0.5f);
        const unsigned int steps = 3;
        for (unsigned int i = 0; i < steps; ++i) {
            pos *= 3.0f;
            const int s =
                ((int)pos.x & 1) + ((int)pos.y & 1) + ((int)pos.z & 1);
            if (s >= 2) return 0.0f;
        }
        return kernel_params.max_extinction;
    } else {
        // 螺旋管状结构
        const float r = 0.5f * (0.5f - fabsf(p.y));
        const float a = (float)(M_PI * 8.0) * p.y;
        const float dx = (cosf(a) * r - p.x) * 2.0f;
        const float dy = (sinf(a) * r - p.z) * 2.0f;
        return powf(fmaxf((1.0f - dx*dx - dy*dy), 0.0f), 8.0f)
               * kernel_params.max_extinction;
    }
}
```

两种程序化体积类型：

**类型 0：Menger 海绵 (Menger Sponge)**
- 经典的三维分形结构
- 3 层递归：将空间划分为 $3 \times 3 \times 3$ 的子块
- 在每层中，如果坐标中有 2 个或以上轴的索引为奇数，则该子块为空（消光系数 = 0）
- 其余子块密度为 $\sigma_{max}$

**类型 1：螺旋管 (Helical Tube)**
- 沿 Y 轴的螺旋曲线
- 管的半径随 Y 从中心向外递减
- 使用距离场的高次幂产生平滑边界

### sample_interaction() - Delta 追踪

```cuda
__device__ inline bool sample_interaction(
    Rand_state &rand_state,
    float3 &ray_pos,
    const float3 &ray_dir,
    const Kernel_params &kernel_params)
{
    float t = 0.0f;
    float3 pos;
    do {
        t -= logf(1.0f - rand(&rand_state)) / kernel_params.max_extinction;

        pos = ray_pos + ray_dir * t;
        if (!in_volume(pos))
            return false;

    } while (get_extinction(kernel_params, pos) <
             rand(&rand_state) * kernel_params.max_extinction);

    ray_pos = pos;
    return true;
}
```

Delta 追踪的核心实现：

1. **指数距离采样**：$t_{step} = -\ln(1 - \xi) / \sigma_{max}$
   - 累加 $t$（不是重置），这等价于从均匀介质中连续采样
2. **出界检测**：如果位置超出体积范围，返回 `false`（射线逃逸）
3. **接受-拒绝**：以概率 $\sigma_t(\mathbf{x}) / \sigma_{max}$ 接受
   - `get_extinction(pos) < rand() * max_extinction` 表示拒绝
   - 条件为 false 时（即 $\sigma \geq \xi \cdot \sigma_{max}$），退出循环，接受交互

### trace_volume() - 体积路径追踪

```cuda
__device__ inline float3 trace_volume(
    Rand_state &rand_state,
    float3 &ray_pos,
    float3 &ray_dir,
    const Kernel_params &kernel_params)
{
    float t0;
    float w = 1.0f;  // 路径权重
    if (intersect_volume_box(t0, ray_pos, ray_dir)) {
        ray_pos += ray_dir * t0;  // 移动到盒表面

        unsigned int num_interactions = 0;
        while (sample_interaction(rand_state, ray_pos, ray_dir, kernel_params))
        {
            // 路径长度限制
            if (num_interactions++ >= kernel_params.max_interactions)
                return make_float3(0.0f, 0.0f, 0.0f);

            // 应用反照率
            w *= kernel_params.albedo;

            // 俄罗斯轮盘赌
            if (w < 0.2f) {
                if (rand(&rand_state) > w * 5.0f)
                    return make_float3(0.0f, 0.0f, 0.0f);
                w = 0.2f;
            }

            // 各向同性相函数采样
            const float phi = (float)(2.0 * M_PI) * rand(&rand_state);
            const float cos_theta = 1.0f - 2.0f * rand(&rand_state);
            const float sin_theta = sqrtf(1.0f - cos_theta * cos_theta);
            ray_dir = make_float3(
                cosf(phi) * sin_theta,
                sinf(phi) * sin_theta,
                cos_theta);
        }
    }
```

完整的路径追踪循环：
1. 射线与包围盒求交
2. Delta 追踪采样交互点
3. 每次交互：乘以反照率、俄罗斯轮盘赌、采样新方向
4. 循环直到射线逃逸体积或被终止

```cuda
    // 查询环境光
    if (kernel_params.environment_type == 0) {
        // 渐变天空
        const float f = (0.5f + 0.5f * ray_dir.y) * w;
        return make_float3(f, f, f);
    } else {
        // HDR 环境贴图
        const float4 texval = tex2D<float4>(
            kernel_params.env_tex,
            atan2f(ray_dir.z, ray_dir.x) * (float)(0.5 / M_PI) + 0.5f,
            acosf(fmaxf(fminf(ray_dir.y, 1.0f), -1.0f)) * (float)(1.0 / M_PI));
        return make_float3(texval.x * w, texval.y * w, texval.z * w);
    }
}
```

环境光查询：
- **类型 0（渐变天空）**：简单的 Y 方向渐变灰度
- **类型 1（HDR 环境贴图）**：使用球坐标将方向映射到纹理 UV：
  - $u = \frac{\text{atan2}(d_z, d_x)}{2\pi} + 0.5$（经度）
  - $v = \frac{\arccos(d_y)}{\pi}$（纬度）

### volume_rt_kernel() - CUDA 入口内核

```cuda
extern "C" __global__ void volume_rt_kernel(
    const Kernel_params kernel_params)
{
    const unsigned int x = blockIdx.x * blockDim.x + threadIdx.x;
    const unsigned int y = blockIdx.y * blockDim.y + threadIdx.y;
    if (x >= kernel_params.resolution.x || y >= kernel_params.resolution.y)
        return;

    // 初始化 PRNG
    const unsigned int idx = y * kernel_params.resolution.x + x;
    Rand_state rand_state;
    curand_init(idx, 0, kernel_params.iteration * 4096, &rand_state);
```

每个 CUDA 线程处理一个像素。PRNG 使用像素索引作为种子，`iteration * 4096` 作为偏移，确保每帧不同。

```cuda
    // 针孔相机射线生成
    const float pr = (2.0f * ((float)x + rand(&rand_state)) * inv_res_x - 1.0f);
    const float pu = (2.0f * ((float)y + rand(&rand_state)) * inv_res_y - 1.0f);
    const float aspect = (float)kernel_params.resolution.y * inv_res_x;
    float3 ray_pos = kernel_params.cam_pos;
    float3 ray_dir = normalize(
        kernel_params.cam_dir * kernel_params.cam_focal +
        kernel_params.cam_right * pr +
        kernel_params.cam_up * aspect * pu);
```

**针孔相机 (Pinhole Camera)** 射线生成：
- 像素坐标添加 $[0, 1)$ 随机抖动实现抗锯齿
- 映射到 $[-1, 1]$ 的 NDC 空间
- 射线方向 = 相机方向 * 焦距 + 水平偏移 + 垂直偏移

```cuda
    // 渐进式累积
    if (kernel_params.iteration == 0)
        kernel_params.accum_buffer[idx] = value;
    else
        kernel_params.accum_buffer[idx] = kernel_params.accum_buffer[idx] +
            (value - kernel_params.accum_buffer[idx]) / (float)(kernel_params.iteration + 1);
```

使用在线均值公式避免浮点精度损失。

```cuda
    // Reinhard 色调映射 + Gamma 校正
    float3 val = kernel_params.accum_buffer[idx] * kernel_params.exposure_scale;
    val.x *= (1.0f + val.x * 0.1f) / (1.0f + val.x);
    val.y *= (1.0f + val.y * 0.1f) / (1.0f + val.y);
    val.z *= (1.0f + val.z * 0.1f) / (1.0f + val.z);
    const unsigned int r = (unsigned int)(255.0f *
        fminf(powf(fmaxf(val.x, 0.0f), (float)(1.0 / 2.2)), 1.0f));
    // ... g, b 类似
    kernel_params.display_buffer[idx] = 0xff000000 | (r << 16) | (g << 8) | b;
}
```

后处理流水线：
1. **曝光缩放**：乘以 $2^{exposure}$
2. **Reinhard 色调映射**：$L' = L \cdot \frac{1 + 0.1L}{1 + L}$
3. **Gamma 校正**：$L_{sRGB} = L'^{1/2.2}$
4. **量化**：浮点 → 8 位整数
5. **打包**：BGRA 格式的 32 位像素（与 OpenGL 的 `GL_BGRA` 格式对应）

## 关键算法深入分析

### Delta 追踪的正确性

Delta 追踪等价于在均匀介质 $\sigma_{max}$ 中引入"空碰撞"：

$$\sigma_t(\mathbf{x}) + \sigma_{null}(\mathbf{x}) = \sigma_{max}$$

其中 $\sigma_{null}(\mathbf{x}) = \sigma_{max} - \sigma_t(\mathbf{x}) \geq 0$。

空碰撞不改变光线状态，因此：
- 真实碰撞概率：$\sigma_t / \sigma_{max}$（accept）
- 空碰撞概率：$\sigma_{null} / \sigma_{max}$（reject, continue）

### 路径追踪中的能量传输

```
Camera → Volume → Scatter → Volume → Scatter → ... → Environment
  |        w=1      w*=ρ                               w * L_env
  |
  v
  Final color = w * L_environment
```

最终颜色为路径权重 $w$ 乘以环境光 $L_{env}$：

$$L = \rho^k \cdot L_{env}$$

其中 $k$ 为散射次数。

### 两种体积类型的对比

| 特性 | Menger 海绵 | 螺旋管 |
|------|------------|--------|
| 密度类型 | 二值（0 或 $\sigma_{max}$） | 连续渐变 |
| 分形维数 | $\approx 2.727$ | N/A |
| Delta 追踪效率 | 高（大区域空碰撞率低） | 中等（平滑衰减） |
| 视觉效果 | 多孔海绵结构 | 柔和的螺旋烟雾 |

## 输入与输出

### 内核输入（通过 Kernel_params）
| 名称 | 类型 | 说明 |
|------|------|------|
| `resolution` | `uint2` | 渲染分辨率 |
| `cam_pos/dir/right/up` | `float3` | 相机参数 |
| `cam_focal` | `float` | 焦距 |
| `max_extinction` | `float` | 最大消光系数 $\sigma_{max}$ |
| `albedo` | `float` | 散射反照率 $\rho$ |
| `max_interactions` | `uint` | 最大交互次数 |
| `volume_type` | `uint` | 体积类型（0 或 1） |
| `environment_type` | `uint` | 环境类型（0 或 1） |
| `env_tex` | `cudaTextureObject_t` | 环境贴图纹理 |
| `iteration` | `uint` | 当前帧编号 |
| `exposure_scale` | `float` | 曝光缩放因子 |

### 内核输出
| 名称 | 类型 | 说明 |
|------|------|------|
| `accum_buffer` | `float3*` | HDR 累积缓冲区 |
| `display_buffer` | `unsigned int*` | LDR 显示缓冲区（BGRA8） |

## 与其他文件的关系

- **`volume_kernel.h`**：定义 `Kernel_params` 结构体和内核声明
- **`main.cpp`**：调用 `volume_rt_kernel` 并提供参数
- **`hdr_loader.h`**：提供环境贴图数据

## 在光线追踪管线中的位置

```
volume_rt_kernel() (每像素一个线程)
    │
    ├── 初始化 PRNG
    ├── 生成相机射线
    ├── trace_volume()
    │   ├── 射线-盒求交
    │   ├── Delta 追踪循环
    │   │   ├── 指数距离采样
    │   │   ├── 接受/拒绝测试
    │   │   ├── 反照率加权
    │   │   ├── 俄罗斯轮盘赌
    │   │   └── 各向同性散射
    │   └── 环境光查询
    ├── 渐进式累积
    └── 色调映射 + 输出
```

## 技术要点与注意事项

1. **PRNG 初始化**：`curand_init(idx, 0, iteration * 4096, &rand_state)` 中的 4096 跳跃确保帧间不重复，但初始化开销较大
2. **max_extinction 的选择**：值过大导致接受率低（效率降低），值过小导致结果错误（密度截断）
3. **反照率 vs 吸收**：$\rho = 0.8$ 意味着每次散射事件有 80% 的能量被散射，20% 被吸收
4. **俄罗斯轮盘赌阈值**：$w < 0.2$ 且 $p_{survive} = 5w$，当 $w = 0.2$ 时存活率 100%，$w = 0.04$ 时存活率 20%
5. **max_interactions 安全阀**：1024 的硬限制防止无限循环（高反照率、密集介质中可能发生）
6. **数值稳定性**：`logf(1.0f - rand())` 中 `rand()` 的值域为 $(0, 1)$，不会产生 $\log(0)$
7. **环境贴图的 clamp**：`fmaxf(fminf(ray_dir.y, 1.0f), -1.0f)` 防止 `acosf` 的输入超出 $[-1, 1]$ 导致 NaN

## 扩展阅读

- Woodcock, E.R. et al. *Techniques Used in the GEM Code for Monte Carlo Neutronics Calculations*. ANL-7050, 1965
- Novak, J. et al. *Monte Carlo Methods for Volumetric Light Transport Simulation*. Computer Graphics Forum, 2018
- Pharr, M., Jakob, W., Humphreys, G. *PBRT*, Chapter 15 - Light Transport II: Volume Rendering
- Arvo, J., Kirk, D. *Particle Transport and Image Synthesis*. SIGGRAPH 1990
- Szirmay-Kalos, L. et al. *Unbiased Estimators to Render Procedurally Generated Inhomogeneous Participating Media*. Computer Graphics Forum, 2017
