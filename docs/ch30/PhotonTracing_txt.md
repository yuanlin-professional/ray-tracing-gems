# PhotonTracing.txt — 屏幕空间光子追踪伪代码

> 源文件：`Ch_30_Caustics_Using_Screen_Space_Photon_Mapping/PhotonTracing.txt`

---

## 1. 文件概述

`PhotonTracing.txt` 是屏幕空间光子映射焦散渲染方法的核心伪代码。它描述了从点光源发射光子并在场景中追踪的完整逻辑，包括光子存储条件判断和折射传播。

---

## 2. 算法与数学背景

### 光子映射 (Photon Mapping)

光子映射是一种两阶段全局光照算法：
1. **光子发射阶段**：从光源发射光子，追踪路径并存储在碰撞点
2. **渲染阶段**：收集每个着色点附近的光子，估计辐照度

### 均匀球面采样

从点光源出发，光子方向通过均匀球面采样 (Uniform Sample Sphere) 生成：

$$\theta = \arccos(1 - 2\xi_1), \quad \phi = 2\pi\xi_2$$

$$\mathbf{d} = (\sin\theta\cos\phi, \; \sin\theta\sin\phi, \; \cos\theta)$$

其中 $\xi_1, \xi_2 \sim U[0,1]$。

### 焦散条件

焦散发生在光线经过镜面/光滑表面折射或反射后照射到漫反射表面。因此，光子需要在经过至少一次镜面交互后的漫反射表面上才被存储。

---

## 3. 完整代码与逐行注释

```pseudocode
void PhotonTracing(float2 LaunchIndex)
{
    // 使用启动索引生成从点光源出发的均匀球面方向
    // LaunchIndex 作为随机数种子，将每个线程映射到一个唯一的光子方向
    Ray = UniformSampleSphere(LaunchIndex.xy);

    // 迭代光子追踪循环
    // MaxIterationCount 限制最大弹射次数，防止无限循环
    for (int i = 0; i < MaxIterationCount; i++)
    {
        // 追踪光线，获取（重建的）表面数据
        // rtTrace 执行加速结构遍历和最近命中测试
        Result = rtTrace(Ray);

        // 检查是否命中场景几何体
        bool bHit = Result.HitT >= 0.0;
        if (!bHit)
            break;  // 光子逃逸场景，终止追踪

        // 根据第 30.3.1 节描述的条件判断是否存储光子
        // 条件包括：表面是否足够漫反射、光子是否经过折射/反射
        if (CheckToStorePhoton(Result))
        {
            // 在当前交点存储光子数据（位置、能量、方向等）
            // 详见第 30.3.1 节的存储策略
            StorePhoton(Result);
        }

        // 如果表面足够光滑（低粗糙度），允许镜面反射
        // 反射的光子可以产生二次焦散效果
        if (Result.Roughness <= RoughnessThresholdForReflection)
        {
            // 追踪反射光线
            FRayHitInfo Result = rtTrace(Ray)
            bool bHit = Result.HitT >= 0.0;
            // 如果反射光线命中且满足存储条件，也存储光子
            if (bHit && CheckToStorePhoton(Result))
                StorePhoton(Result);
        }

        // 通过折射继续传播光子
        // RefractPhoton 根据 Snell 定律计算折射方向并更新光线
        Ray = RefractPhoton(Ray, Result);
    }
}
```

---

## 4. 输入与输出

### 输入

| 参数 | 类型 | 说明 |
|------|------|------|
| `LaunchIndex` | `float2` | 调度线程的二维索引，用作采样种子 |

### 隐式输入

| 变量 | 说明 |
|------|------|
| `MaxIterationCount` | 最大迭代（弹射）次数 |
| `RoughnessThresholdForReflection` | 允许镜面反射的粗糙度阈值 |
| 场景加速结构 | 用于 `rtTrace` 的光线遍历 |

### 输出

通过 `StorePhoton` 将光子写入光子缓冲区（位置、能量、方向等信息）。

---

## 5. 应用场景

- **水面焦散**：光线通过水面折射后在水底形成的亮斑
- **玻璃物体投射的光斑**：光线通过透明物体折射形成的聚焦图案
- **二次焦散**：经过两次或多次镜面交互后产生的复杂光斑

---

## 6. 关键算法深入分析

### 追踪策略

- **最大迭代次数限制**：`MaxIterationCount` 防止在高折射率场景中无限弹射
- **条件存储**：通过 `CheckToStorePhoton` 过滤不需要存储的光子（如光子还在镜面表面上）
- **粗糙度阈值**：`RoughnessThresholdForReflection` 区分镜面和漫反射表面

### 屏幕空间特性

与传统光子映射不同，这里的光子存储在屏幕空间中：
- 光子的位置被投影到屏幕空间
- 光子收集阶段在屏幕空间进行
- 避免了世界空间光子图的内存开销

---

## 7. 与其他文件的关系

本文件为独立伪代码。完整实现需要配合：
- 光子收集/密度估计着色器
- G-Buffer 生成管线
- 最终合成 Pass

---

## 8. 在光线追踪管线中的位置

```
光源数据准备
    ↓
┌─────────────────────────┐
│ PhotonTracing            │  ← 本文件
│ DXR Ray Generation Shader│
│ 发射 + 追踪 + 存储光子    │
└─────────────────────────┘
    ↓
光子密度估计 (Splatting / Gathering)
    ↓
焦散贡献合成到最终图像
```

---

## 9. 技术要点与注意事项

1. **`Result.HitT >= 0.0`**：使用命中距离的符号判断是否有效命中（负值表示未命中）。
2. **反射与折射分支**：反射分支在折射之前执行，确保低粗糙度表面的反射焦散也被捕获。
3. **伪代码简化**：实际实现中 `rtTrace` 需要完整的 DXR TraceRay 调用，包含光线标志和载荷。
4. **性能考虑**：每个线程可能执行多次 `rtTrace`，光子追踪的性能取决于场景复杂度和最大迭代次数。

---

## 10. 扩展阅读

- **书中章节**：Chapter 30, Section 30.3 "Caustics Using Screen Space Photon Mapping"
- **经典光子映射**：Jensen, H.W., "Realistic Image Synthesis Using Photon Mapping," 2001.
- **实时焦散**：Hu, W. et al., "Caustic Triangles for Screen-Space Photon Mapping," *I3D*, 2010.
