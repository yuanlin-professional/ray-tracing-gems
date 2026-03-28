# boilerplate.cuh — OptiX 框架与辅助函数

> 源文件：`Ch_04_A_Planetarium_Dome_Master_Camera/boilerplate.cuh`

---

## 1. 文件概述

`boilerplate.cuh` 是穹顶相机实现的 CUDA/OptiX 框架头文件，提供了渲染所需的全局变量声明、随机数生成器、抗锯齿抖动、景深 (DoF) 光线扰动和累积缓冲区管理等功能。该代码源自 Tachyon 光线追踪引擎。

---

## 2. 代码结构概览

### 头文件依赖

| 头文件 | 用途 |
|--------|------|
| `<optix.h>` | OptiX 核心 API |
| `<optixu/optixu_math_namespace.h>` | OptiX 数学工具（float3 等） |
| `<optixu/optixu_matrix_namespace.h>` | 矩阵运算 |
| `<optixu/optixu_aabb_namespace.h>` | 轴对齐包围盒 |

### 全局变量/常量表

| 变量 | 类型 | 说明 |
|------|------|------|
| `framebuffer` | `rtBuffer<uchar4, 2>` | 输出帧缓冲区（8 位 RGBA） |
| `accumulation_buffer` | `rtBuffer<float4, 2>` | 浮点累积缓冲区 |
| `accumulation_normalization_factor` | `float` | 累积归一化因子 |
| `launch_index` | `uint2` | 当前启动线程的二维索引 |
| `launch_dim` | `uint2` | 启动维度（图像分辨率） |
| `progressiveSubframeIndex` | `unsigned int` | 渐进式渲染子帧索引 |
| `accumCount` | `unsigned int` | 累积帧计数 |
| `scene_epsilon` | `float` | 场景 epsilon（自相交避免） |
| `max_depth` | `int` | 最大光线递归深度 |
| `max_trans` | `int` | 最大透明表面数 |
| `cam_zoom` | `float` | 相机缩放因子 |
| `cam_pos` | `float3` | 相机世界坐标位置 |
| `cam_U, cam_V, cam_W` | `float3` | 相机正交基向量 |
| `cam_stereo_eyesep` | `float` | 立体渲染眼间距 |
| `cam_dof_aperture_rad` | `float` | 景深光圈半径 |
| `cam_dof_focal_dist` | `float` | 景深焦距 |
| `aa_samples` | `int` | 抗锯齿采样数 |

### 函数列表

| 函数 | 说明 |
|------|------|
| `subframe_count()` | 返回当前总子帧计数 |
| `myrt_rand(idum)` | 32 位 LCG 伪随机数生成器 |
| `tea<N>(val0, val1)` | TEA 加密算法生成高质量随机种子 |
| `jitter_offset2f(pval, xy)` | 生成 $[-0.5, 0.5)$ 的 AA 抖动偏移 |
| `jitter_disc2f(pval, xy, radius)` | 生成圆盘内的 DoF 抖动偏移 |
| `dof_ray(...)` | 计算景深扰动后的光线原点和方向 |
| `accumulate_color(col, alpha)` | 将颜色写入累积缓冲区 |

### 数据结构

| 结构体 | 说明 |
|--------|------|
| `PerRayData_radiance` | 光线载荷：颜色、透明度、重要性、递归深度、透射计数 |

---

## 3. 逐段代码详解

### 随机数生成器（第 69-89 行）

```cuda
static __device__ __inline__
unsigned int myrt_rand(unsigned int &idum) {
  idum *= 1099087573;
  return idum;
}
```

32 位 LCG 随机数生成器，使用 Fishman (1990) 推荐的乘数 1099087573。该 RNG 速度极快但质量有限：
- 周期：$10^9$
- $10^6$ 样本后开始失败统计测试
- 适合渲染中的快速采样

### TEA 随机种子生成器（第 102-115 行）

```cuda
template<unsigned int N> static __host__ __device__ __inline__
unsigned int tea(unsigned int val0, unsigned int val1) {
  unsigned int v0 = val0, v1 = val1, s0 = 0;
  for(unsigned int n = 0; n < N; n++) {
    s0 += 0x9e3779b9;  // 黄金比例常数
    v0 += ((v1<<4)+0xa341316c)^(v1+s0)^((v1>>5)+0xc8013ea4);
    v1 += ((v0<<4)+0xad90777d)^(v0+s0)^((v0>>5)+0x7e95761e);
  }
  return v0;
}
```

Tiny Encryption Algorithm (TEA) 用于从像素坐标和帧索引生成高质量的初始随机种子。模板参数 `N` 控制加密轮数（更多轮 = 更好的随机性）。

### 圆盘抖动生成器（第 129-151 行）

```cuda
static __device__ __inline__
void jitter_disc2f(unsigned int &pval, float2 &xy, float radius) {
  float   r=(myrt_rand(pval) * MYRT_RAND_MAX_INV);
  float phi=(myrt_rand(pval) * MYRT_RAND_MAX_INV) * 2.0f * M_PIf;
  __sincosf(phi, &xy.x, &xy.y);
  xy *= sqrtf(r) * radius;
}
```

在半径为 `radius` 的圆盘内均匀采样：
- $r \sim U[0,1]$，$\phi \sim U[0, 2\pi)$
- 使用 $\sqrt{r}$ 确保面积均匀分布
- `__sincosf` 是 CUDA 快速近似三角函数

### 景深光线扰动（第 160-169 行）

```cuda
static __device__ __inline__
void dof_ray(const float3 &ray_origin_orig, float3 &ray_origin,
             const float3 &ray_direction_orig, float3 &ray_direction,
             unsigned int &randseed, const float3 &up, const float3 &right) {
  float3 focuspoint = ray_origin_orig + ray_direction_orig * cam_dof_focal_dist;
  float2 dofjxy;
  jitter_disc2f(randseed, dofjxy, cam_dof_aperture_rad);
  ray_origin = ray_origin_orig + dofjxy.x*right + dofjxy.y*up;
  ray_direction = normalize(focuspoint - ray_origin);
}
```

景深模拟的薄透镜模型：
1. 计算焦平面上的聚焦点
2. 在光圈圆盘内随机偏移光线原点
3. 从新原点指向聚焦点得到新方向
4. 焦平面上的点保持清晰，其他距离的点被模糊

### PerRayData_radiance 结构（第 191-197 行）

| 字段 | 类型 | 说明 |
|------|------|------|
| `result` | `float3` | 最终着色颜色 |
| `alpha` | `float` | 透明度（反向传播到帧缓冲区） |
| `importance` | `float` | 递归光线树的重要性权重 |
| `depth` | `int` | 当前递归深度 |
| `transcnt` | `int` | 已穿越的透明表面数 |

---

## 4. 技术要点与注意事项

1. **LCG vs TEA**：LCG 用于快速连续采样，TEA 用于生成高质量初始种子。两者结合平衡了速度和质量。
2. **`__sincosf`**：CUDA 特有的快速近似函数，精度约 $10^{-6}$，比标准 `sinf`/`cosf` 更快。
3. **累积缓冲区**：使用 `float4` 而非 `uchar4`，支持 HDR 渐进式累积。
4. **OptiX API**：`rtDeclareVariable`、`rtBuffer` 等是 OptiX 5.x 的声明宏，在 OptiX 7+ 中已改用不同的 API。

---

## 5. 扩展阅读

- **书中章节**：Chapter 4, "A Planetarium Dome Master Camera"
- **TEA 算法**：Wheeler, D. and Needham, R., "TEA, a Tiny Encryption Algorithm," *Fast Software Encryption*, 1994
- **GPU 随机数**：Zafar, F. et al., "GPU Random Numbers via the Tiny Encryption Algorithm," *HPG*, 2010
