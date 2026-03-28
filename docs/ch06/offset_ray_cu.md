# offset_ray.cu — 自适应光线原点偏移

> 源文件：`Ch_06_A_Fast_and_Robust_Method_for_Avoiding_Self-Intersection/offset_ray.cu`

---

## 1. 文件概述

`offset_ray.cu` 实现了一种基于 IEEE 754 浮点数位操作的光线原点偏移函数 `offset_ray`，用于在光线追踪中避免自相交 (Self-Intersection) 问题。该方法在书中第 6.2.2 节中详细描述。

与传统的固定 epsilon 偏移不同，此方法根据交点坐标的浮点精度自适应调整偏移量，在极大和极小尺度下都能正确工作。

---

## 2. 算法与数学背景

### 自相交问题

当从表面点 $\mathbf{p}$ 沿法线 $\mathbf{n}$ 方向发射二次光线时，由于浮点运算的有限精度，计算得到的交点 $\mathbf{p}$ 可能位于表面的错误一侧，导致光线错误地与原表面相交。

### IEEE 754 整数位偏移原理

IEEE 754 单精度浮点数具有一个重要性质：对于正浮点数，将其位模式解释为整数后加 1，等价于将浮点数增加 1 ULP (Unit in the Last Place)。

$$\text{float\_as\_int}(x) + 1 \Rightarrow x + \text{ULP}(x)$$

本方法利用这一性质，通过整数加减来实现与浮点数量级自适应的偏移：

- **远离原点区域** ($|p| \geq \frac{1}{32}$)：使用整数位偏移，偏移量 = $256 \times n$ 个 ULP
- **靠近原点区域** ($|p| < \frac{1}{32}$)：使用固定浮点偏移，偏移量 = $\frac{n}{65536}$

### 常量选择

| 常量 | 值 | 用途 |
|------|-----|------|
| `origin()` | $1/32$ | 切换阈值：整数偏移与浮点偏移的分界点 |
| `float_scale()` | $1/65536$ | 近原点区域的浮点偏移比例 |
| `int_scale()` | $256$ | 远原点区域的整数偏移缩放因子 |

---

## 3. 代码结构概览

### 函数列表

| 函数 | 返回类型 | 参数 | 说明 |
|------|---------|------|------|
| `origin()` | `float` | 无 | 返回切换阈值常量 $1/32$ |
| `float_scale()` | `float` | 无 | 返回浮点偏移比例 $1/65536$ |
| `int_scale()` | `float` | 无 | 返回整数偏移缩放因子 $256$ |
| `offset_ray()` | `float3` | `p`, `n` | 计算偏移后的光线原点 |

---

## 4. 逐段代码详解

### 常量定义（第 3-5 行）

```cuda
constexpr float origin()      { return 1.0f / 32.0f; }
constexpr float float_scale() { return 1.0f / 65536.0f; }
constexpr float int_scale()   { return 256.0f; }
```

三个 `constexpr` 函数定义了算法的核心常量：
- `origin()`：当坐标分量绝对值小于此阈值时，切换到浮点偏移模式
- `float_scale()`：近原点区域的偏移缩放因子
- `int_scale()`：远原点区域的整数位偏移缩放因子

### offset_ray 函数（第 8-19 行）

```cuda
float3 offset_ray(const float3 p, const float3 n)
{
  int3 of_i(int_scale() * n.x, int_scale() * n.y, int_scale() * n.z);
```

- 参数 `p`：表面交点坐标
- 参数 `n`：表面法线（从表面出射方向，对于出射光线指向外侧，对于入射光线已翻转）
- `of_i`：将法线分量乘以 256 并截断为整数，作为整数偏移量

```cuda
  float3 p_i(
     int_as_float(float_as_int(p.x) + ((p.x < 0) ? -of_i.x : of_i.x)),
     int_as_float(float_as_int(p.y) + ((p.y < 0) ? -of_i.y : of_i.y)),
     int_as_float(float_as_int(p.z) + ((p.z < 0) ? -of_i.z : of_i.z)));
```

- 将 `p` 的每个分量的浮点位模式解释为整数
- 根据分量的符号决定加或减偏移量（负数需要反向偏移以远离表面）
- 将结果整数位模式转回浮点数
- 这一步实现了自适应的 ULP 级偏移

```cuda
  return float3(fabsf(p.x) < origin() ? p.x+float_scale()*n.x : p_i.x,
                fabsf(p.y) < origin() ? p.y+float_scale()*n.y : p_i.y,
                fabsf(p.z) < origin() ? p.z+float_scale()*n.z : p_i.z);
}
```

- 对每个分量独立判断：
  - 若绝对值 < $1/32$（靠近原点），使用固定浮点偏移 $p + n/65536$
  - 否则，使用整数位偏移结果 `p_i`
- 靠近原点时整数偏移可能不够精确（ULP 过小），因此切换到固定偏移

---

## 5. 关键算法深入分析

### 为什么需要两种模式

- **整数偏移模式**：偏移量正比于浮点数的 ULP，即正比于 $|p| \times 2^{-23}$。当 $|p|$ 很大时偏移量也大，很小时偏移量也小，自适应于数值精度。
- **浮点偏移模式**：当 $|p|$ 接近 0 时，ULP 极小（次正规数区域），整数偏移可能不足以避免自相交。此时使用固定偏移量 $n/65536$ 更安全。

### 时间复杂度

$O(1)$ — 仅涉及常数次算术运算和位操作。

### 数值精度

- 整数偏移引入的误差约为 $256 \times \text{ULP}(p) \approx |p| \times 2^{-15}$
- 浮点偏移引入的误差为 $|n|/65536 \approx 1.5 \times 10^{-5}$（假设法线单位化）
- 两者在切换点 $|p| = 1/32$ 处量级匹配

### 性能优化

- 使用 `constexpr` 函数确保常量在编译时计算
- `int_as_float` / `float_as_int` 在 CUDA 中映射为零开销的类型转换内联函数
- 无分支条件编译为条件移动指令，避免 GPU warp 分歧

---

## 6. 输入与输出

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `p` | `float3` | 输入 | 表面交点世界坐标 |
| `n` | `float3` | 输入 | 表面法线（已根据光线方向翻转，指向出射侧） |
| **返回值** | `float3` | 输出 | 偏移后的光线原点 |

---

## 7. 与其他文件的关系

本文件为独立实现，可直接集成到任何 CUDA/OptiX 光线追踪器中。Chapter 29 的 `intersection_refinement.h` 中也包含类似的相交精化技术。

---

## 8. 在光线追踪管线中的位置

```
光线-表面相交检测
       ↓
  计算交点 p 和法线 n
       ↓
  ┌─────────────────┐
  │  offset_ray(p,n) │  ← 本文件
  └─────────────────┘
       ↓
  偏移后的点作为二次光线原点
       ↓
  发射反射/折射/阴影光线
```

在 OptiX / DXR 管线中，此函数通常在最近命中着色器 (Closest-Hit Shader) 或任意命中着色器 (Any-Hit Shader) 中调用，用于生成二次光线的原点。

---

## 9. 技术要点与注意事项

1. **法线方向要求**：`n` 必须指向光线出射侧。对于反射光线，`n` 指向表面外侧；对于折射光线，`n` 指向表面内侧（已翻转）。
2. **float_as_int / int_as_float**：这些函数执行位重解释 (bit reinterpretation)，不是数值类型转换。在 CUDA 中对应 `__float_as_int()` 和 `__int_as_float()`。
3. **对次正规数的处理**：靠近原点的浮点偏移模式隐式处理了次正规数 (denormalized numbers) 区域。
4. **不适用于非 IEEE 754 平台**：算法依赖 IEEE 754 浮点数的位布局。

---

## 10. 扩展阅读

- **书中章节**：Chapter 6, Section 6.2.2 "A Fast and Robust Method for Avoiding Self-Intersection"
- **相关论文**：Wächter, C., and Binder, N., "A Fast and Robust Method for Avoiding Self-Intersection," *Ray Tracing Gems*, Chapter 6, 2019.
- **IEEE 754 标准**：IEEE Standard for Floating-Point Arithmetic (IEEE 754-2008)
