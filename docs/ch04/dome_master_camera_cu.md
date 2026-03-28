# dome_master_camera.cu — 穹顶主相机光线生成

> 源文件：`Ch_04_A_Planetarium_Dome_Master_Camera/dome_master_camera.cu`

---

## 1. 文件概述

`dome_master_camera.cu` 实现了天文馆穹顶显示系统的光线生成程序 (Ray Generation Program)，生成等距鱼眼投影 (Equidistant Fisheye Projection) 的光线。支持四种模式的组合：单眼/立体 × 无景深/有景深，通过 C++ 模板在编译时生成优化版本。

---

## 2. 算法与数学背景

### 等距鱼眼投影

穹顶主相机使用等距鱼眼投影，特点是像素到图像中心的距离与对应方向的天顶角 $\theta$ 成线性关系：

$$\theta = \text{hypot}(p_x, p_y)$$

其中 $p_x, p_y$ 是以弧度为单位的像素偏移角度，通过以下公式从像素坐标计算：

$$p = (\text{viewport\_idx} - \text{viewport\_mid}) \times \text{radperpix}$$

### 光线方向计算

给定天顶角 $\theta$ 和像素偏移 $(p_x, p_y)$：

$$\mathbf{d} = \mathbf{U} \frac{\sin\theta}{\theta} p_x + \mathbf{V} \frac{\sin\theta}{\theta} p_y + \mathbf{W} \cos\theta$$

其中 $\frac{\sin\theta}{\theta}$ 是归一化因子，$\mathbf{U}, \mathbf{V}, \mathbf{W}$ 是相机的正交基。

### 立体渲染

立体模式使用上/下分屏 (Over/Under) 布局：
- 上半部分：左眼图像
- 下半部分：右眼图像

眼睛偏移沿垂直于光线方向和穹顶"上方"方向的轴：

$$\mathbf{o}_\text{eye} = \mathbf{o} + \text{eyeshift} \times (\mathbf{d} \times \mathbf{W})$$

### 景深

在穹顶投影中，景深的"上方"和"右方"方向需要特殊计算（不能使用全局 up/right）：

$$\mathbf{up} = -\mathbf{U}\frac{\cos\theta}{\theta}p_x - \mathbf{V}\frac{\cos\theta}{\theta}p_y + \mathbf{W}\sin\theta$$

$$\mathbf{right} = \mathbf{U}\frac{p_y}{\theta} - \mathbf{V}\frac{p_x}{\theta}$$

---

## 3. 代码结构概览

### 函数列表

| 函数 | 类型 | 说明 |
|------|------|------|
| `camera_dome_general<STEREO, DOF>()` | 模板设备函数 | 穹顶相机核心逻辑 |
| `camera_dome_master()` | RT_PROGRAM | 无立体 + 无景深 |
| `camera_dome_master_dof()` | RT_PROGRAM | 无立体 + 有景深 |
| `camera_dome_master_stereo()` | RT_PROGRAM | 有立体 + 无景深 |
| `camera_dome_master_stereo_dof()` | RT_PROGRAM | 有立体 + 有景深 |

---

## 4. 逐段代码详解

### 立体渲染视口设置（第 28-47 行）

```cuda
uint viewport_sz_y, viewport_idx_y;
float eyeshift;
if (STEREO_ON) {
  viewport_sz_y = launch_dim.y >> 1;  // 视口高度减半
  if (launch_index.y >= viewport_sz_y) {
    viewport_idx_y = launch_index.y - viewport_sz_y;
    eyeshift = -0.5f * cam_stereo_eyesep;  // 左眼
  } else {
    viewport_idx_y = launch_index.y;
    eyeshift =  0.5f * cam_stereo_eyesep;  // 右眼
  }
}
```

立体模式将帧缓冲区纵向一分为二，上半部分渲染左眼（负偏移），下半部分渲染右眼（正偏移）。由于 `STEREO_ON` 是模板参数，非立体模式下此分支在编译时被消除。

### 鱼眼投影参数（第 49-63 行）

```cuda
float fov = M_PIf;                        // 180 度视场角
float thetamax = 0.5 * fov;               // 最大天顶角 90 度
float2 viewport_sz = make_float2(launch_dim.x, viewport_sz_y);
float2 radperpix = fov / viewport_sz;     // 弧度/像素
float2 viewport_mid = viewport_sz * 0.5f; // 视口中心
```

- 视场角固定为 $\pi$（180 度），即完整半球
- `radperpix` 将像素距离转换为弧度
- 超出半球范围的像素（圆形视口外）设为黑色

### 随机种子初始化（第 65 行）

```cuda
unsigned int randseed = tea<4>(launch_dim.x*(launch_index.y)+launch_index.x, subframe_count());
```

使用 TEA 算法从像素索引和帧号生成唯一的高质量随机种子。4 轮 TEA 提供足够的随机性。

### 核心光线方向计算（第 82-93 行）

```cuda
float2 p = (viewport_idx - viewport_mid) * radperpix;
float theta = hypotf(p.x, p.y);

if (theta < thetamax) {
  if (theta == 0) {
    ray_direction = cam_W;  // 穹顶天顶方向
  } else {
    float sintheta, costheta;
    sincosf(theta, &sintheta, &costheta);
    float rsin = sintheta / theta;
    ray_direction = cam_U*rsin*p.x + cam_V*rsin*p.y + cam_W*costheta;
  }
}
```

- `hypotf(p.x, p.y)` 计算像素到中心的角度距离 $\theta$
- 天顶 ($\theta = 0$) 特殊处理，避免除零
- `sintheta/theta` 是归一化因子，确保 $(p_x, p_y)$ 到 $(\sin\theta \cos\phi, \sin\theta \sin\phi)$ 的正确映射

### 立体眼偏移（第 94-98 行）

```cuda
if (STEREO_ON) {
  ray_origin += eyeshift * cross(ray_direction, cam_W);
}
```

假设平面穹顶，`cam_W` 也是观众的"上方"方向。眼偏移沿光线方向与上方向的叉积方向（即"左右"方向），实现视差。

### 景深处理（第 100-106 行）

```cuda
if (DOF_ON) {
  float rcos = costheta / theta;
  float3 ray_up    = -cam_U*rcos*p.x  -cam_V*rcos*p.y + cam_W*sintheta;
  float3 ray_right =  cam_U*(p.y/theta) + cam_V*(-p.x/theta);
  dof_ray(ray_origin, ray_origin, ray_direction, ray_direction,
          randseed, ray_up, ray_right);
}
```

为每条光线计算局部的"上方"和"右方"向量（在穹顶投影中随方向变化），然后调用 `dof_ray` 进行薄透镜模型的景深扰动。

### 光线追踪与累积（第 110-122 行）

```cuda
PerRayData_radiance prd;
prd.importance = 1.f;
prd.depth = 0;
prd.transcnt = max_trans;
optix::Ray ray = optix::make_Ray(ray_origin, ray_direction,
                                  radiance_ray_type, scene_epsilon, RT_DEFAULT_MAX);
rtTrace(root_object, ray, prd);
col += prd.result;
alpha += prd.alpha;
```

OptiX 光线追踪调用：创建光线并遍历加速结构。多个 AA 样本的结果累加到 `col` 和 `alpha`。

---

## 5. 关键算法深入分析

### 模板优化

通过 `template<int STEREO_ON, int DOF_ON>` 生成 4 个特化版本，编译器可以消除未使用的代码分支，避免 GPU 上的分支开销。

### 圆形视口处理

当 $\theta \geq \theta_\text{max}$ 时，像素位于圆形视口外，不发射光线也不累积颜色。最终 `accumulate_color` 使用归零的 `col` 和 `alpha`（仅在圆形区域外）。

### 性能特征

- 每像素：1 次 TEA + `aa_samples` × (2 次 RNG + 三角函数 + 光线追踪)
- 光线追踪是主要开销，其余计算均为 $O(1)$

---

## 6. 输入与输出

### 输入（来自 boilerplate.cuh 的全局变量）

| 变量 | 说明 |
|------|------|
| `launch_index` | 当前像素坐标 |
| `launch_dim` | 图像分辨率 |
| `cam_pos, cam_U, cam_V, cam_W` | 相机参数 |
| `cam_stereo_eyesep` | 立体眼间距 |
| `cam_dof_*` | 景深参数 |
| `aa_samples` | AA 采样数 |

### 输出

| 变量 | 说明 |
|------|------|
| `accumulation_buffer[launch_index]` | 累积的 RGBA 颜色 |

---

## 7. 与其他文件的关系

```
boilerplate.cuh
  ├── 全局变量声明
  ├── 随机数生成器 (myrt_rand, tea)
  ├── 抖动函数 (jitter_offset2f, jitter_disc2f)
  ├── 景深函数 (dof_ray)
  └── 累积函数 (accumulate_color)
       ↓
dome_master_camera.cu  ← 本文件
  └── 使用以上所有功能生成穹顶投影光线
```

---

## 8. 在光线追踪管线中的位置

```
OptiX 启动 (rtContextLaunch2D)
       ↓
  ┌──────────────────────────────┐
  │  camera_dome_master*()        │  ← 本文件 (Ray Generation)
  │  生成等距鱼眼投影光线          │
  └──────────────────────────────┘
       ↓
  rtTrace → 遍历 BVH 加速结构
       ↓
  Closest-Hit / Any-Hit / Miss Shaders
       ↓
  返回颜色到 prd.result
       ↓
  累积到 accumulation_buffer
```

---

## 9. 技术要点与注意事项

1. **`hypotf` vs `sqrtf(x*x + y*y)`**：`hypotf` 处理溢出/下溢更安全，但可能稍慢。
2. **`theta == 0` 判断**：浮点精确等于零的判断在此场景合理，因为只有视口正中心的像素会触发。
3. **上/下分屏假设**：立体渲染假设帧缓冲区高度加倍，OpenGL 端需要相应处理。
4. **OptiX 5.x API**：`rtTrace`、`RT_PROGRAM` 等为 OptiX 5 语法；OptiX 7+ 使用不同的编程模型。

---

## 10. 扩展阅读

- **书中章节**：Chapter 4, "A Planetarium Dome Master Camera"
- **鱼眼投影**：Bourke, P., "Computer Generated Angular Fisheye Projections," 2001
- **Tachyon 引擎**：Stone, J.E., "An Efficient Library for Parallel Ray Tracing and Animation," M.S. Thesis, 1998
