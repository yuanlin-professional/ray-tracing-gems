# rqx.cu -- OptiX 双线性面片求交程序

> 源文件：`Ch_08_Cool_Patches_A_Geometric_Approach_to_Ray-Bilinear_Patch_Intersections/rqx.cu`

---

## 1. 文件概述

`rqx.cu` 实现了在 NVIDIA OptiX 光线追踪框架中对双线性面片 (Bilinear Patch) 进行光线求交的自定义相交程序 (Intersection Program)。该文件定义了一个 `RT_PROGRAM` 函数 `intersectPatch`，在 OptiX 的加速结构 (Acceleration Structure) 遍历过程中被调用。

算法核心是将光线-双线性面片的求交问题转化为一个关于参数 $u$ 的一元二次方程 (Quadratic Equation)，利用混合积 (Mixed Product / Scalar Triple Product) 构造系数，然后对有效根求解 $v$ 和 $t$ 参数。

---

## 2. 算法与数学背景

### 双线性面片的参数化

双线性面片由四个角点定义，面片上任意一点为：

$$\mathbf{q}(u, v) = (1-u)(1-v)\mathbf{q}_{00} + u(1-v)\mathbf{q}_{10} + uv\mathbf{q}_{11} + (1-u)v\mathbf{q}_{01}$$

### 求交方程推导

光线方程为 $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$。将光线代入面片方程并利用标量三重积 (Scalar Triple Product) 消元，得到：

$$a + bu + cu^2 = 0$$

其中：
- $a = [(\mathbf{q}_{00} - \mathbf{o}) \times \mathbf{d}] \cdot \mathbf{e}_{00}$
- $c = \mathbf{q}_n \cdot \mathbf{d}$，其中 $\mathbf{q}_n = (\mathbf{q}_{10} - \mathbf{q}_{00}) \times (\mathbf{q}_{01} - \mathbf{q}_{11})$
- $b = [(\mathbf{q}_{10} - \mathbf{o}) \times \mathbf{d}] \cdot \mathbf{e}_{11} - a - c$

注意 $b$ 的计算采用了间接方式：先计算 $a + b + c$ 的值，再减去 $a$ 和 $c$。

### 韦达定理求根

为提高数值稳定性 (Numerical Stability)，使用韦达定理 (Viete's Formula)：

$$u_1 = \frac{-b - \text{sign}(b)\sqrt{b^2 - 4ac}}{2c}, \quad u_2 = \frac{a}{c \cdot u_1}$$

这避免了当 $b$ 和 $\sqrt{\Delta}$ 符号相同时的灾难性抵消 (Catastrophic Cancellation)。

### v 参数的计算

对每个有效根 $u_i \in [0,1]$，在面片上确定两个点：
- $\mathbf{p}_a = \text{lerp}(\mathbf{q}_{00}, \mathbf{q}_{10}, u_i)$：底边上的插值点
- $\mathbf{p}_b = \text{lerp}(\mathbf{e}_{00}, \mathbf{e}_{11}, u_i)$：实际为 $\mathbf{p}_b - \mathbf{p}_a$ 方向向量

然后通过辅助叉积 (Cross Product) 求解 $v$ 和 $t$：

$$\mathbf{n} = \mathbf{d} \times \mathbf{p}_b$$
$$t = \frac{(\mathbf{n} \times \mathbf{p}_a) \cdot \mathbf{p}_b}{|\mathbf{n}|^2}, \quad v = \frac{(\mathbf{n} \times \mathbf{p}_a) \cdot \mathbf{d}}{|\mathbf{n}|^2}$$

### 几何法线与着色法线

- 几何法线 (Geometric Normal)：$\mathbf{n}_g = \frac{\partial \mathbf{q}}{\partial u} \times \frac{\partial \mathbf{q}}{\partial v}$
- 着色法线 (Shading Normal)：通过四个顶点法线的双线性插值得到

---

## 3. 代码结构概览

### 函数列表

| 函数 | 类型 | 参数 | 说明 |
|------|------|------|------|
| `intersectPatch` | `RT_PROGRAM void` | `int prim_idx` | OptiX 自定义相交程序入口 |

### 核心数据结构

| 名称 | 类型 | 说明 |
|------|------|------|
| `patchdata` | `rtBuffer` | OptiX 缓冲区，存储所有面片数据 |
| `PatchData` | struct | 包含面片角点系数和顶点法线 |
| `irec` | struct | 相交记录，包含法线、纹理坐标、图元索引 |
| `ray` | `Ray` | OptiX 内建光线变量 |

---

## 4. 逐段代码详解

### 完整源代码（带中文注释）

```cuda
// OptiX 自定义相交程序：对一个双线性面片执行光线求交测试
RT_PROGRAM void intersectPatch(int prim_idx) {
    // ray 是通过 rtDeclareVariable(Ray,ray,rtCurrentRay,) 声明的 OptiX 内建变量
    const PatchData& patch = patchdata[prim_idx]; // 从 OptiX 缓冲区获取当前面片数据
    const float3* q = patch.coefficients();       // 获取 4 个角点 + 预计算的 "法线" qn

    // 读取四个角点坐标
    float3 q00 = q[0], q10 = q[1], q11 = q[2], q01 = q[3];

    // 计算边向量
    // q01 ─────────── q11
    //  |                |
    //  | e00        e11 |   预计算：
    //  |                |   qn = cross(q10-q00, q01-q11)
    //  |      e10       |
    // q00 ─────────── q10
    float3 e10 = q10 - q00; // 底边向量
    float3 e11 = q11 - q10; // 右边向量
    float3 e00 = q01 - q00; // 左边向量
    float3 qn  = q[4];      // 预计算的叉积 cross(e10, q01-q11)

    // 将角点平移到以光线原点为中心的坐标系（减少浮点误差）
    q00 -= ray.origin;
    q10 -= ray.origin;

    // 构造一元二次方程 a + bu + cu^2 = 0 的系数
    float a = dot(cross(q00, ray.direction), e00); // 标量三重积 [q00 x d] . e00
    float c = dot(qn, ray.direction);              // qn . d
    float b = dot(cross(q10, ray.direction), e11); // 先计算 a+b+c = [q10 x d] . e11
    b -= a + c;                                    // 再减去 a 和 c 得到 b

    // 计算判别式
    float det = b*b - 4*a*c;
    if (det < 0) return;      // 判别式 < 0 表示光线与面片不相交（参见书中图 8.5）
    det = sqrt(det);           // CUDA 使用 -use_fast_math 编译选项

    float u1, u2;              // 两个根（u 参数）
    float t = ray.tmax, u, v;  // 记录最小 t > 0 的解

    // 求解二次方程
    if (c == 0) {                            // c == 0 时面片退化为梯形，只有一个根
        u1  = -a/b; u2 = -1;                // 线性方程只有一个解
    } else {                                 // 一般情况（Stanford 模型中 c != 0）
        u1  = (-b - copysignf(det, b))/2;   // 数值稳定的根（避免灾难性抵消）
        u2  = a/u1;                          // 韦达定理：u1*u2 = a/c
        u1 /= c;
    }

    // 检查第一个根 u1
    if (0 <= u1 && u1 <= 1) {                  // u1 在面片参数范围内？
        float3 pa = lerp(q00, q10, u1);        // 底边上的插值点（参见书中图 8.4）
        float3 pb = lerp(e00, e11, u1);        // 实际为方向向量 pb - pa
        float3 n  = cross(ray.direction, pb);  // 辅助法线
        det = dot(n, n);                       // |n|^2，用于归一化
        n = cross(n, pa);                      // 更新 n 用于计算 t 和 v
        float t1 = dot(n, pb);                 // t * |n|^2
        float v1 = dot(n, ray.direction);      // v * |n|^2
        if (t1 > 0 && 0 <= v1 && v1 <= det) { // t1 > 0 且 v1 在 [0,1] 内
            t = t1/det; u = u1; v = v1/det;    // 若 t1 > ray.tmax，将被
        }                                      // rtPotentialIntersection 拒绝
    }

    // 检查第二个根 u2（需要 t2 < t1，即比第一个根更近）
    if (0 <= u2 && u2 <= 1) {                  // u2 在面片参数范围内？
        float3 pa = lerp(q00, q10, u2);        // u1 可能已有有效解
        float3 pb = lerp(e00, e11, u2);        // 所以需要 0 < t2 < t1
        float3 n  = cross(ray.direction, pb);
        det = dot(n, n);
        n = cross(n, pa);
        float t2 = dot(n, pb)/det;             // 直接计算归一化后的 t2
        float v2 = dot(n, ray.direction);
        if (0 <= v2 && v2 <= det && t > t2 && t2 > 0) { // 更近且有效
            t = t2; u = u2; v = v2/det;
        }
    }

    // 向 OptiX 报告潜在的相交
    if (rtPotentialIntersection(t)) {
        // 填充相交记录结构体
        // 最近命中着色器中将对法线进行归一化

        // 计算几何法线：偏导数的叉积
        float3 du = lerp(e10, q11 - q01, v);  // dq/du = lerp(e10, q11-q01, v)
        float3 dv = lerp(e00, e11, u);        // dq/dv = lerp(e00, e11, u)
        irec.geometric_normal = cross(du, dv); // 几何法线

        // 着色法线（可选）
        #if defined(SHADING_NORMALS)
        const float3* vn = patch.vertex_normals; // 四个顶点的法线
        irec.shading_normal = lerp(lerp(vn[0],vn[1],u),  // 双线性插值
                                   lerp(vn[3],vn[2],u),v);
        #else
        irec.shading_normal = irec.geometric_normal;      // 无顶点法线时使用几何法线
        #endif

        irec.texcoord = make_float3(u, v, 0);  // 纹理坐标 (u, v)
        irec.id = prim_idx;                     // 图元索引
        rtReportIntersection(0u);               // 向 OptiX 报告相交，材质索引为 0
    }
}
```

---

## 5. 关键算法深入分析

### 5.1 二次方程系数的几何含义

系数 $a$、$b$、$c$ 分别对应光线方向与面片边向量的标量三重积：

| 系数 | 公式 | 几何含义 |
|------|------|---------|
| $a$ | $[(\mathbf{q}_{00}-\mathbf{o}) \times \mathbf{d}] \cdot \mathbf{e}_{00}$ | 光线与面片左下角、左边构成的混合积 |
| $c$ | $\mathbf{q}_n \cdot \mathbf{d}$ | 预计算叉积与光线方向的点积 |
| $b$ | 间接计算 | 通过 $a + b + c$ 的关系间接得到 |

$b$ 的间接计算方式避免了直接计算可能带来的额外误差：先计算 $a + b + c = [(\mathbf{q}_{10}-\mathbf{o}) \times \mathbf{d}] \cdot \mathbf{e}_{11}$，然后 $b = (a+b+c) - a - c$。

### 5.2 copysignf 的稳定求根技巧

```cuda
u1 = (-b - copysignf(det, b)) / 2;
```

`copysignf(det, b)` 返回 `det` 的绝对值并取 `b` 的符号。这确保分子中 $-b$ 和 $-\text{sign}(b) \cdot |\sqrt{\Delta}|$ 同号，从而使分子的绝对值最大化，避免两个接近值相减的精度损失。第二个根通过韦达公式 $u_1 \cdot u_2 = a/c$ 得到。

### 5.3 v 和 t 的统一求解

对于给定的 $u_i$，面片上对应的直线段可表示为：

$$\mathbf{q}(u_i, v) = \mathbf{p}_a + v \cdot \mathbf{p}_b$$

求解光线与此线段的交点等价于一个光线-线段求交问题，通过两次叉积操作高效完成。关键观察：在除以 $|\mathbf{n}|^2$ 之前先检查 $v$ 的范围，避免了除法运算。

### 5.4 退化情况处理

当 $c = 0$ 时，面片退化为梯形 (Trapezoid)，二次方程退化为线性方程，只有一个根 $u_1 = -a/b$。代码通过设置 $u_2 = -1$（超出 $[0,1]$ 范围）来跳过第二个根的检查。

### 5.5 几何法线计算

面片在点 $(u, v)$ 处的偏导数为：

$$\frac{\partial \mathbf{q}}{\partial u} = \text{lerp}(\mathbf{e}_{10},\, \mathbf{q}_{11} - \mathbf{q}_{01},\, v)$$

$$\frac{\partial \mathbf{q}}{\partial v} = \text{lerp}(\mathbf{e}_{00},\, \mathbf{e}_{11},\, u)$$

几何法线为两个偏导数的叉积。法线在最近命中着色器 (Closest-Hit Shader) 中归一化。

---

## 6. 输入与输出

### 输入

| 参数 | 类型 | 说明 |
|------|------|------|
| `prim_idx` | `int` | 图元索引，用于从缓冲区获取面片数据 |
| `ray` | `Ray` (OptiX 内建) | 当前光线，包含 `origin`、`direction`、`tmax` |
| `patchdata` | `rtBuffer` | OptiX 缓冲区，存储面片的角点坐标和预计算数据 |

### 输出

通过 `irec` 结构体和 OptiX API 报告：

| 字段 | 类型 | 说明 |
|------|------|------|
| `irec.geometric_normal` | `float3` | 交点处的几何法线（未归一化） |
| `irec.shading_normal` | `float3` | 交点处的着色法线（未归一化） |
| `irec.texcoord` | `float3` | 纹理坐标 $(u, v, 0)$ |
| `irec.id` | `int` | 面片的图元索引 |

---

## 7. 与其他文件的关系

| 相关文件 | 关系 |
|---------|------|
| `cpu_test.cpp` | CPU 端的测试实现，`intersectPatchWorldCoordinates` 函数与本文件算法完全一致 |
| OptiX 最近命中着色器 | 接收本文件报告的相交记录，执行着色计算 |
| OptiX 主机端代码 | 负责创建 `patchdata` 缓冲区和设置加速结构 |

---

## 8. 在光线追踪管线中的位置

```
OptiX 加速结构遍历 (BVH Traversal)
       |
       v
  包围盒测试 (Bounding Box Test) 通过
       |
       v
  ┌─────────────────────────┐
  │  intersectPatch(prim_idx) │  <-- 本文件：自定义相交程序
  │  求解 u, v, t 参数         │
  │  计算法线和纹理坐标         │
  │  调用 rtReportIntersection │
  └─────────────────────────┘
       |
       v
  OptiX 判断是否为最近相交
       |
       v
  最近命中着色器 (Closest-Hit Shader)
  归一化法线 -> 着色计算
```

在 OptiX 管线中，本文件充当**自定义相交程序** (Intersection Program)，取代内置的三角形相交测试。当加速结构遍历找到潜在相交的包围盒时，调用此程序执行精确的光线-面片求交。

---

## 9. 技术要点与注意事项

1. **预计算 $\mathbf{q}_n$**：叉积 $(\mathbf{q}_{10} - \mathbf{q}_{00}) \times (\mathbf{q}_{01} - \mathbf{q}_{11})$ 在主机端预计算并存入 `q[4]`，避免在每次求交时重复计算。

2. **原点平移**：将 $\mathbf{q}_{00}$ 和 $\mathbf{q}_{10}$ 平移到以光线原点为中心的坐标系，减少大坐标值的浮点精度损失。

3. **延迟除法**：$v$ 和 $t$ 的归一化（除以 $\text{det} = |\mathbf{n}|^2$）推迟到范围检查之后，避免不必要的除法运算。

4. **-use_fast_math**：CUDA 编译选项 `-use_fast_math` 使 `sqrt` 等函数使用快速但精度较低的硬件实现，对光线追踪场景通常足够。

5. **两根的非对称处理**：第一个根 $u_1$ 只检查 $t_1 > 0$，而第二个根 $u_2$ 额外检查 $t_2 < t$（$t$ 可能已被第一个根更新），确保选择最近的有效交点。

6. **材质索引为 0**：`rtReportIntersection(0u)` 中的 `0u` 是材质索引，表示使用第一个绑定的材质。

7. **法线不归一化**：报告的法线是未归一化的，需要在最近命中着色器中进行归一化处理。

---

## 10. 扩展阅读

- **书中章节**：Chapter 8, "Cool Patches: A Geometric Approach to Ray/Bilinear Patch Intersections"
- **OptiX 文档**：NVIDIA OptiX Programming Guide - Custom Intersection Programs
- **韦达定理数值稳定性**：Press et al., *Numerical Recipes*, Section 5.6, "Quadratic and Cubic Equations"
- **相关章节**：Chapter 7 "Precision Improvements for Ray/Sphere Intersection" 涉及类似的数值稳定性技巧
