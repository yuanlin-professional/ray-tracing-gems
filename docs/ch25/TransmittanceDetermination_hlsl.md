# TransmittanceDetermination.hlsl 技术文档

## 文件概述

**文件路径**: `Ch_25_Hybrid_Rendering_for_Real-Time_Ray_Tracing/TransmittanceDetermination.hlsl`

本文件实现了**透射方向确定 (Transmittance Direction Determination)** 的代码片段，用于在光线追踪中处理透明/半透明材质（如玻璃）的折射射线生成。代码根据射线是从空气进入玻璃还是从玻璃射出到空气，选择正确的折射率比值，并使用 HLSL 内建的 `refract()` 函数计算折射方向。

## 算法与数学背景

### 斯涅尔定律 (Snell's Law)

光线在两种不同介质的交界面发生折射，遵循斯涅尔定律：

$$n_1 \sin\theta_1 = n_2 \sin\theta_2$$

其中：
- $n_1, n_2$ 为两种介质的折射率 (Index of Refraction, IOR)
- $\theta_1$ 为入射角
- $\theta_2$ 为折射角

### HLSL refract() 函数

HLSL 的 `refract(I, N, eta)` 函数计算折射向量：

$$\mathbf{T} = \eta \mathbf{I} + \left(\eta \cos\theta_i - \cos\theta_t\right) \mathbf{N}$$

其中 $\eta = n_1 / n_2$，$\mathbf{I}$ 为入射方向（指向表面），$\mathbf{N}$ 为法线。

当 $\sin^2\theta_t = \eta^2(1 - \cos^2\theta_i) > 1$ 时，发生**全内反射 (Total Internal Reflection)**，此时 `refract()` 返回零向量。

### 正面与背面判断

对于封闭透明物体（如玻璃球），光线会两次穿过表面：
- **正面命中 (Front Face)**：从空气进入玻璃，$\eta = n_{air} / n_{glass}$
- **背面命中 (Back Face)**：从玻璃射出到空气，$\eta = n_{glass} / n_{air}$

## 代码结构概览

本文件是一个简短代码片段（8 行），以下提供完整源码及逐行注释。

## 逐段代码详解

```hlsl
// 第1-2行：注释说明
// If we are going from air to glass or glass to air,
// choose the correct index of refraction ratio.
```

```hlsl
// 第3行：判断是否为背面命中
bool isBackFace = (HitKind() == HIT_KIND_TRIANGLE_BACK_FACE);
```
- `HitKind()`：DXR 内建函数，返回当前命中的三角形面朝向
- `HIT_KIND_TRIANGLE_BACK_FACE`：表示射线从三角形背面进入
- 背面命中意味着射线正在从物体内部射出

```hlsl
// 第4行：根据面朝向选择正确的折射率比值
float ior = isBackFace ? iorGlass / iorAir : iorAir / iorGlass;
```
- **背面（从玻璃到空气）**：$\eta = n_{glass} / n_{air}$
  - 例如：$1.5 / 1.0 = 1.5$
- **正面（从空气到玻璃）**：$\eta = n_{air} / n_{glass}$
  - 例如：$1.0 / 1.5 \approx 0.667$

典型折射率值：
| 材质 | IOR |
|------|-----|
| 空气 | 1.0 |
| 水 | 1.33 |
| 玻璃 | 1.5 |
| 钻石 | 2.42 |

```hlsl
// 第6-8行：创建折射射线
RayDesc refractionRay;
refractionRay.Origin = worldPosition;
refractionRay.Direction = refract(worldRayDir, worldNormal, ior);
```
- `worldPosition`：世界空间中的交点位置
- `worldRayDir`：入射射线方向（世界空间）
- `worldNormal`：表面法线（世界空间），对于背面应已翻转
- `refract()`：返回折射方向向量

## 关键算法深入分析

### 折射率比值的选择逻辑

```
空气 (n=1.0)          玻璃 (n=1.5)
                │
   ----→        │        正面命中
  入射射线      │        eta = iorAir / iorGlass = 0.667
                │   ----→  折射射线
                │
                │
                │        背面命中
  ←----         │        eta = iorGlass / iorAir = 1.5
  折射射线      │   ←----  入射射线（从内部）
                │
```

### 全内反射处理

当光线从高折射率介质射向低折射率介质（如从玻璃到空气），入射角超过**临界角 (Critical Angle)** 时，发生全内反射：

$$\theta_c = \arcsin\left(\frac{n_{air}}{n_{glass}}\right) \approx 41.8°$$

此时 `refract()` 返回 `(0, 0, 0)`，实际实现中应检测此情况并改为执行反射。本代码片段未展示全内反射的处理逻辑。

### 菲涅尔效应 (Fresnel Effect)

完整的透明材质渲染还需考虑菲涅尔效应——入射角越大，反射比例越高。本代码片段仅展示折射部分，实际实现中通常配合菲涅尔系数来混合反射和折射。

## 输入与输出

### 输入
| 名称 | 类型 | 说明 |
|------|------|------|
| `iorGlass` | `float` | 玻璃材质折射率（如 1.5） |
| `iorAir` | `float` | 空气折射率（如 1.0） |
| `worldPosition` | `float3` | 射线-表面交点的世界空间位置 |
| `worldRayDir` | `float3` | 入射射线方向（世界空间） |
| `worldNormal` | `float3` | 表面法线（世界空间） |

### 输出
| 名称 | 类型 | 说明 |
|------|------|------|
| `refractionRay` | `RayDesc` | 折射射线（含起点和方向） |

## 与其他文件的关系

- **`RayGenerationAndMissShaders.pseudocode.hlsl`**：折射射线可在阴影射线之外额外发射，两者共用相同的场景加速结构
- **`HaltonSampling.hlsl`**：折射方向本身是确定性的，但粗糙透射可能需要随机扰动

## 在光线追踪管线中的位置

```
射线 → BVH 遍历 → 最近命中着色器 (Closest-Hit) → [透射方向确定] → 递归 TraceRay()
                                                   ^^^^^^^^^^^^^^^^^^
                                                   本文件实现此步骤
```

本代码位于**最近命中着色器 (Closest-Hit Shader)** 内部，在检测到透明材质时计算折射射线方向，随后递归调用 `TraceRay()` 继续追踪。

## 技术要点与注意事项

1. **法线方向**：对于背面命中，`worldNormal` 应已翻转（指向射线来源方向），否则 `refract()` 的结果不正确
2. **全内反射检测**：需要检测 `refract()` 返回的零向量并回退到反射
3. **递归深度**：透射射线会导致递归，DXR 管线有最大递归深度限制（通常 1-31）
4. **自相交**：折射射线的 `TMin` 需要适当偏移以避免与同一表面再次相交
5. **色散 (Dispersion)**：本实现使用单一 IOR，不考虑色散效应。模拟色散需要按波长分别追踪
6. **嵌套介质**：代码假设简单的空气-玻璃两层结构。对于嵌套透明物体（如水中的玻璃杯），需要更复杂的介质栈管理

## 扩展阅读

- Pharr, M., Jakob, W., Humphreys, G. *PBRT*, Chapter 8 - Reflection Models (Specular Transmission)
- Heckbert, P. *Adaptive Radiosity Textures for Bidirectional Ray Tracing*. SIGGRAPH 1990
- DXR Specification: `HitKind()`, `HIT_KIND_TRIANGLE_BACK_FACE`
- Whitted, T. *An Improved Illumination Model for Shaded Display*. CACM, 1980
