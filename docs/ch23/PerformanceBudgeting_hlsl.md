# PerformanceBudgeting.hlsl 技术文档

## 文件概述

`PerformanceBudgeting.hlsl` 实现了 Frostbite 引擎中交互式光照贴图预览系统的**性能预算系统** (Performance Budgeting System)。该系统运行在 CPU 端（主机代码），通过动态调节每帧的采样比率 (sample ratio)，确保 GPU 光线追踪 (Ray Tracing) 的耗时不超过预设的时间预算，从而维持编辑器的交互帧率。

**源文件路径**: `Ch_23_Interactive Light Map and Irradiance Volume Preview in Frostbite/PerformanceBudgeting.hlsl`

## 算法与数学背景

### 反馈控制系统

性能预算系统本质上是一个**反馈控制器 (Feedback Controller)**。它测量上一帧 GPU 光线追踪的实际耗时，与目标预算进行比较，然后调整下一帧的采样率：

$$\text{sampleRatio}_{n+1} = f(\text{sampleRatio}_n, \text{timeSpent}_n, \text{budget})$$

### 阻尼因子与加速因子

- **阻尼因子 (Damping Factor)** $\alpha = 0.9$：当超出预算时，将采样率缩减为原来的 $\alpha$ 倍，避免剧烈波动
- **加速因子 (Boost Factor)**：$\beta = \text{clamp}(0.25, 1.0, \frac{\text{budget}}{\text{timeSpent}})$，当严重超预算时提供更强的收缩

超预算时：$r_{n+1} = r_n \cdot \alpha \cdot \beta$

低于预算时：$r_{n+1} = r_n / \alpha$（即放大约 11%）

### 稳定区间

为避免在目标值附近来回振荡，系统定义了一个**稳定区间** (stable area)，宽度为预算的 15%：

$$|\text{timeSpent} - \text{budget}| \leq 0.15 \times \text{budget}$$

当实际耗时落在稳定区间内时，不调整采样率。

## 代码结构概览

```
获取上帧状态 → 计算加速因子 → [判断是否在稳定区间外]
  → 超预算：缩减采样率 / 低于预算：提升采样率
  → 限制采样率范围
```

## 完整源码与逐行注释

```hlsl
// Host code for computing the sample ratio used by the performance budgeting system.
// 主机端代码：计算性能预算系统使用的采样比率

// 目标时间预算：16 毫秒（对应约 60 FPS 的帧预算）
const float tracingBudgetInMs = 16.0f;
// 阻尼因子：90%，经验值，用于平滑调整，防止振荡
const float dampingFactor = 0.9f;                  // 90% (empirical)
// 稳定区间：预算的 15%，在此范围内不调整采样率
const float stableArea = tracingBudgetInMs*0.15f;  // 15% of the budget

// 获取上一帧使用的采样比率（0.0 ~ 1.0）
float sampleRatio = getLastFrameRatio();
// 获取上一帧 GPU 光线追踪实际耗时（毫秒）
float timeSpentTracing = getGPUTracingTime();
// 计算加速因子：预算 / 实际耗时，限制在 [0.25, 1.0] 范围
// 当严重超预算时（timeSpent >> budget），boostFactor 趋近 0.25，加速缩减
// 当轻微超预算或未超预算时，boostFactor 接近或等于 1.0
float boostFactor =
        clamp(0.25f, 1.0f, tracingBudgetInMs / timeSpentTracing);

// 仅在实际耗时偏离预算超过稳定区间时才调整
if (abs(timeSpentTracing - tracingBudgetInMs) > stableArea)
    if (traceTime > tracingBudgetInMs)
        // 超预算：缩减采样率（乘以阻尼因子和加速因子）
        sampleRatio *= dampingFactor * boostFactor;
    else
        // 低于预算：提升采样率（除以阻尼因子，即乘以 ~1.11）
        sampleRatio /= dampingFactor;

// 将采样率限制在 [0.001, 1.0] 范围内
// 最低 0.1% 保证至少有少量采样在进行
// 最高 100% 表示所有纹素都参与采样
sampleRatio = clamp(0.001f, 1.0f, sampleRatio);
```

## 关键算法深入分析

### 采样率的物理含义

`sampleRatio` 表示每帧实际执行光线追踪的纹素占总纹素的比例：
- `sampleRatio = 1.0`：所有纹素都参与追踪（GPU 负载最大）
- `sampleRatio = 0.5`：仅一半纹素参与追踪
- `sampleRatio = 0.001`：仅 0.1% 的纹素参与追踪（几乎暂停）

较低的采样率意味着更慢的收敛速度，但保证了交互帧率。

### 调整策略的非对称性

系统的调整策略是**非对称**的：
- **下调**（超预算）：$r \times 0.9 \times \beta$，其中 $\beta \in [0.25, 1.0]$
  - 最激进下调：$r \times 0.225$（降至原来的 22.5%）
  - 最温和下调：$r \times 0.9$
- **上调**（低于预算）：$r / 0.9 \approx r \times 1.111$

这种非对称设计是有意为之：超预算时需要快速响应以避免卡顿，而上调时则保守增长以避免突然的性能尖峰。

### 控制环路的稳定性

```
            ┌─────────────────────────────────┐
            │                                 │
            ▼                                 │
 ┌──────────────────┐  采样率   ┌──────────┐  │ GPU耗时
 │ 调整 sampleRatio │────────>│ GPU 追踪 │──┘
 └──────────────────┘          └──────────┘
         ▲
         │ 稳定区间过滤
```

稳定区间（$\pm 2.4$ ms）充当了死区 (dead zone)，防止在平衡点附近的微小波动引起不必要的调整。

## 输入与输出

| 参数/变量 | 类型 | 说明 |
|-----------|------|------|
| `tracingBudgetInMs` | `float` | GPU 追踪时间预算（毫秒），常量 16.0 |
| `dampingFactor` | `float` | 阻尼因子，常量 0.9 |
| `getLastFrameRatio()` | `float` | 上一帧的采样比率 |
| `getGPUTracingTime()` | `float` | 上一帧的 GPU 追踪耗时（毫秒） |
| **输出** `sampleRatio` | `float` | 当前帧的采样比率 [0.001, 1.0] |

## 与其他文件的关系

- **InitRay.hlsl / InitRayImproved.hlsl**：`sampleRatio` 决定了每帧有多少纹素执行这些光照积分内核
- **VarianceTracking.hlsl**：已收敛的纹素可以从采样中排除，与性能预算系统协同工作

## 在光线追踪管线中的位置

本代码运行在 **CPU 端**（主机代码），在每帧的光线追踪分派 (dispatch) 之前执行。它根据上一帧的 GPU 性能反馈计算新的采样率，然后将该比率传递给 GPU 端的分派逻辑。

```
[CPU] 读取上帧 GPU 时间 → 计算新 sampleRatio → 设置分派参数
                                                    │
                                                    ▼
                                            [GPU] 执行光线追踪
                                                    │
                                                    ▼
                                            [GPU→CPU] 时间戳回读
```

## 技术要点与注意事项

1. **变量名不一致**：代码中同时使用了 `timeSpentTracing` 和 `traceTime`，这可能是原文中的笔误，两者应指同一变量
2. **clamp 参数顺序**：HLSL 标准 `clamp(x, min, max)` 中，代码写作 `clamp(0.25f, 1.0f, value)` 实际效果等同于 `clamp(value, 0.25, 1.0)`，注意此处可能使用了自定义的 clamp 函数
3. **16ms 预算假设**：16ms 对应约 60 FPS，但实际 Frostbite 编辑器中预算应该只是帧时间的一部分（留余量给其他渲染工作）
4. **帧间延迟**：GPU 时间的回读通常有 1-2 帧延迟，这使得控制循环需要足够的阻尼来避免过度响应
5. **收敛与交互的平衡**：低采样率下收敛极慢，但保证编辑器可交互；高采样率加速收敛但可能卡顿

## 扩展阅读

- Ray Tracing Gems, Chapter 23, Section 23.6: Performance Budgeting
- Astrom, K.J. & Murray, R.M. *Feedback Systems: An Introduction for Scientists and Engineers* —— 反馈控制理论基础
- GPU profiling and timing queries in DirectX 12
