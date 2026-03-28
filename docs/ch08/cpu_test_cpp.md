# cpu_test.cpp -- 双线性面片求交 CPU 测试程序

> 源文件：`Ch_08_Cool_Patches_A_Geometric_Approach_to_Ray-Bilinear_Patch_Intersections/cpu_test/cpu_test/cpu_test.cpp`

---

## 1. 文件概述

`cpu_test.cpp` 是双线性面片 (Bilinear Patch) 光线求交算法的 CPU 端测试程序，共 215 行（含版权声明和空行）。该文件同时实现了两种求交方法：

1. **世界坐标方法** (`intersectPatchWorldCoordinates`)：与 `rqx.cu` 中的 GPU 算法完全一致
2. **光线中心坐标方法** (`intersectPatchRayCentricCoordinates`)：在光线对齐的坐标系中执行求交，数学表达式更简洁

此外还提供了坐标变换的辅助函数 `donb` 和 `transform`。`main` 函数用已知数据验证两种方法产生相同的结果。

---

## 2. 算法与数学背景

### 世界坐标方法

与 `rqx.cu` 相同，通过混合积 (Scalar Triple Product) 构造一元二次方程并求解。详见 [rqx_cu.md](rqx_cu.md) 的算法部分。

### 光线中心坐标方法

将所有面片顶点变换到以光线原点为原点、光线方向为 $z$ 轴的坐标系中。设变换后的顶点为 $\mathbf{q}'_i = (x_i, y_i, z_i)$，则：

- 光线在此坐标系中简化为从原点出发沿 $z$ 轴正方向的射线
- 求交问题的 $xy$ 分量化为二维问题：在 $z = 0$ 平面上找面片的 $(u, v)$ 参数
- $t$ 参数直接由 $z$ 分量给出

#### 简化后的二次方程系数

在光线中心坐标系中，方程 $a + bu + cu^2 = 0$ 的系数简化为：

$$a = q'_{00,y} \cdot q'_{01,x} - q'_{00,x} \cdot q'_{01,y}$$

$$c = (q'_{00,y} - q'_{10,y})(q'_{01,x} - q'_{11,x}) + (q'_{00,x} - q'_{10,x})(q'_{11,y} - q'_{01,y})$$

$$b = q'_{10,y} \cdot q'_{11,x} - q'_{10,x} \cdot q'_{11,y} - a - c$$

注意这些只涉及二维的 $x$、$y$ 分量，因为光线方向已被消除。

#### v 和 t 的计算

$$v = \frac{p_{d,x} \cdot p_{o,x} + p_{d,y} \cdot p_{o,y}}{p_{d,x}^2 + p_{d,y}^2}$$

$$t = p_{o,z} - v \cdot p_{d,z}$$

其中 $\mathbf{p}_o$ 和 $\mathbf{p}_d$ 分别为面片在给定 $u$ 值处的线段端点和方向。

### 正交基构造 (Duff et al. 2017)

`donb` 函数使用 Duff 等人在 2017 年提出的改进方法，从一个方向向量构造完整的正交基 (Orthonormal Basis)。该方法：
- 避免了经典方法中的 `if` 分支（当输入接近某坐标轴时的退化处理）
- 仅使用一次 `copysignf` 处理 $z$ 分量的符号
- 数值精度优于 Frisvad (2012) 的方法

---

## 3. 代码结构概览

### 函数列表

| 函数 | 返回类型 | 参数 | 说明 |
|------|---------|------|------|
| `intersectPatchWorldCoordinates` | `void` | `q[5]`, `ray`, `t`, `u`, `v` | 世界坐标系下的面片求交 |
| `donb` | `void` (inline) | `raydir`, `axis1`, `axis2` | 从光线方向构造正交基 |
| `transform` | `void` (inline) | `center`, `axis1`, `axis2`, `axis3`, `q[]`, `n` | 将顶点变换到新坐标系 |
| `intersectPatchRayCentricCoordinates` | `void` | `q[4]`, `t`, `u`, `v` | 光线中心坐标系下的面片求交 |
| `main` | `int` | 无 | 测试入口，验证两种方法一致性 |

### 模块依赖

```
main()
  |
  +---> intersectPatchWorldCoordinates()  // 方法一：世界坐标
  |
  +---> donb()                            // 构造正交基
  |       |
  |       v
  +---> transform()                       // 坐标变换
  |       |
  |       v
  +---> intersectPatchRayCentricCoordinates()  // 方法二：光线中心坐标
```

---

## 4. 逐段代码详解

### 头文件与命名空间（第 30-34 行）

```cuda
#define NOMINMAX
#include <optixu/optixu_math_stream_namespace.h>
using namespace optix;

#include <iostream>
```

- `NOMINMAX`：阻止 Windows 头文件定义 `min`/`max` 宏，避免与 STL 冲突
- 引入 OptiX 数学工具库，提供 `float3`、`dot`、`cross`、`lerp` 等函数
- 代码仅使用 OptiX 的数学工具，不使用 OptiX 光线追踪 API

### IDE 调试输出（第 37-51 行）

```cuda
#if defined(_MSC_BUILD)
#include <windows.h>
struct ide_stream : std::streambuf {
    int overflow(int s) {
        OutputDebugStringA((LPCSTR)&s); // 输出字符到 Visual Studio IDE
        int c = putchar(s);             // 同时输出到控制台
        std::cout << std::flush;        // 立即刷新
        return c;
    }
    ide_stream() {
        std::cout.rdbuf(this);          // 重定向 cout
        std::cerr.rdbuf(this);          // 重定向 cerr
    }
} ide_stream;
#endif
```

仅在 MSVC 编译器下生效，将标准输出同时重定向到 Visual Studio 的 Output 窗口和控制台。

### intersectPatchWorldCoordinates 函数（第 55-104 行）

```cuda
void intersectPatchWorldCoordinates(const float3* q, const Ray& ray,
                                     float& t, float& u, float& v) {
    // 初始化 t 为无穷大，表示尚未找到有效交点
    t = std::numeric_limits<float>::infinity();

    float3 q00 = q[0], q10 = q[1], q11 = q[2], q01 = q[3];

    // 计算边向量
    float3 e10 = q10 - q00; // 底边
    float3 e11 = q11 - q10; // 右边
    float3 e00 = q01 - q00; // 左边
    float3 qn  = q[4];      // 预计算的 cross(q10-q00, q01-q11)

    // 平移到以光线原点为中心的坐标系
    q00 -= ray.origin;
    q10 -= ray.origin;

    // 构造二次方程系数 a + bu + cu^2 = 0
    float a = dot(cross(q00, ray.direction), e00);
    float c = dot(qn, ray.direction);
    float b = dot(cross(q10, ray.direction), e11); // 先计算 a+b+c
    b -= a + c;                                    // 再得到 b

    // 判别式检查
    float det = b*b - 4*a*c;
    if (det < 0) return;
    det = sqrt(det);

    float u1, u2;

    // 求根
    if (c == 0) {                            // 梯形退化情况
        u1  = -a/b; u2 = -1;
    } else {                                 // 一般情况
        u1  = (-b - copysignf(det, b))/2;   // 稳定根
        u2  = a/u1;                          // 韦达公式
        u1 /= c;
    }

    // 检查第一个根 u1
    if (0 <= u1 && u1 <= 1) {
        float3 pa = lerp(q00, q10, u1);
        float3 pb = lerp(e00, e11, u1);
        float3 n  = cross(ray.direction, pb);
        det = dot(n, n);
        n = cross(n, pa);
        float t1 = dot(n, pb);
        float v1 = dot(n, ray.direction);
        if (t1 > 0 && 0 <= v1 && v1 <= det) {
            t = t1/det; u = u1; v = v1/det;
        }
    }

    // 检查第二个根 u2
    if (0 <= u2 && u2 <= 1) {
        float3 pa = lerp(q00, q10, u2);
        float3 pb = lerp(e00, e11, u2);
        float3 n  = cross(ray.direction, pb);
        det = dot(n, n);
        n = cross(n, pa);
        float t2 = dot(n, pb)/det;
        float v2 = dot(n, ray.direction);
        if (0 <= v2 && v2 <= det && t > t2 && t2 > 0) {
            t = t2; u = u2; v = v2/det;
        }
    }
}
```

此函数与 `rqx.cu` 中的 `intersectPatch` 在算法上完全一致。主要区别：
- 初始化 `t` 为无穷大（GPU 版本使用 `ray.tmax`）
- 不调用 OptiX 的 `rtPotentialIntersection` / `rtReportIntersection`
- 不计算法线

### donb 函数（第 106-115 行）

```cuda
inline void donb(const float3& raydir, float3& axis1, float3& axis2) {
    // 使用 Duff et al. "Building an Orthonormal Basis, Revisited" (2017)
    // http://jcgt.org/published/0006/01/01/
    const float3& axis3 = raydir;              // z 轴 = 光线方向
    float sign = copysignf(1.0f, axis3.z);     // z 分量的符号
    const float a = -1.0f / (sign + axis3.z);  // 辅助变量
    const float b = axis3.x * axis3.y * a;     // 辅助变量

    // 构造正交基的另外两个轴
    axis1 = make_float3(1.0f + sign * axis3.x * axis3.x * a,
                        sign * b,
                        -sign * axis3.x);
    axis2 = make_float3(b,
                        sign + axis3.y * axis3.y * a,
                        -axis3.y);
}
```

从光线方向 `raydir` 构造一组正交归一化基底 (Orthonormal Basis)。输出的 `axis1`、`axis2` 与 `raydir`（作为 `axis3`）构成右手坐标系。

该算法由 Duff 等人于 2017 年在 JCGT 上发表，特点是：
- 不含条件分支（仅 `copysignf` 隐含符号判断）
- 所有输入方向均有良好数值精度
- 时间复杂度 $O(1)$

### transform 函数（第 118-125 行）

```cuda
inline void transform(const float3& center, const float3& axis1,
                      const float3& axis2, const float3& axis3,
                      float3* q, int n = 4) {
    for (int i = 0; i < n; i++) {
        float3 cr = q[i] - center;   // 平移到新原点
        q[i].x = dot(cr, axis1);     // 投影到 axis1 方向
        q[i].y = dot(cr, axis2);     // 投影到 axis2 方向
        q[i].z = dot(cr, axis3);     // 投影到 axis3（光线方向）
    }
}
```

将 `n` 个顶点从世界坐标系变换到以 `center` 为原点、以 `(axis1, axis2, axis3)` 为基底的坐标系。变换后光线方向变为 $(0, 0, 1)$，光线原点变为 $(0, 0, 0)$。

### intersectPatchRayCentricCoordinates 函数（第 127-176 行）

```cuda
void intersectPatchRayCentricCoordinates(const float3* q,
                                          float& t, float& u, float& v) {
    t = std::numeric_limits<float>::infinity();

    // 二次方程系数（仅使用 x, y 分量，因为光线方向已经是 z 轴）
    float a = q[0].y*q[3].x - q[0].x*q[3].y;        // 2D 叉积 q00 x q01
    float c = (q[0].y - q[1].y)*(q[3].x - q[2].x)   // 展开的二次项系数
            + (q[0].x - q[1].x)*(q[2].y - q[3].y);
    float b = q[1].y*q[2].x - q[1].x*q[2].y;        // 2D 叉积 q10 x q11
    b -= a + c;                                       // 与世界坐标方法相同的间接计算

    float det = b*b - 4*a*c;
    if (det < 0) return;
    det = sqrt(det);

    float u1, u2;

    // 求根（与世界坐标方法相同）
    if (c == 0) {
        u1 = -a/b; u2 = -1;
    } else {
        u1  = (-b - copysignf(det, b))/2;
        u2  = a/u1;
        u1 /= c;
    }

    // 检查第一个根 u1
    if (0 <= u1 && u1 <= 1) {
        float3 po =      lerp(q[0], q[1], u1);  // 底边上的插值点
        float3 pd = po - lerp(q[3], q[2], u1);  // 方向向量（po 到顶边）
        det       = pd.x*pd.x + pd.y*pd.y;      // 2D 距离的平方
        float v1  = pd.x*po.x + pd.y*po.y;      // 2D 点积（v 的分子）
        if (0 <= v1 && v1 <= det) {
            v1 /= det;
            float t1 = po.z - v1 * pd.z;         // t 直接由 z 分量得到
            if (t1 > 0) {
                t = t1; u = u1; v = v1;
            }
        }
    }

    // 检查第二个根 u2
    if (0 <= u2 && u2 <= 1) {
        float3 po =      lerp(q[0], q[1], u2);
        float3 pd = po - lerp(q[3], q[2], u2);
        det       = pd.x*pd.x + pd.y*pd.y;
        float v2  = pd.x*po.x + pd.y*po.y;
        if (0 <= v2 && v2 <= det) {
            v2 /= det;
            float t2 = po.z - v2 * pd.z;
            if (t > t2 && t2 > 0) {               // 比第一个根更近
                t = t2; u = u2; v = v2;
            }
        }
    }
}
```

与世界坐标方法的关键区别：
- 二次方程系数仅涉及顶点的 $x$、$y$ 分量（二维叉积），无需光线方向
- $v$ 参数通过二维点积计算，无需三维叉积
- $t$ 参数直接由 $z$ 分量给出：$t = p_{o,z} - v \cdot p_{d,z}$

### main 函数（第 179-215 行）

```cuda
int main() {
    Ray ray;
    // 使用已知的精确 u, v, t 值初始化，然后反向求解验证
    ray.direction = normalize(make_float3(-3, 4, -12));
    float3 intersection = make_float3(0.648, 0.368, 0.748);
    ray.origin = intersection - ray.direction;  // 确保 t = 1.0 处为交点

    // 定义双线性面片的四个角点
    float3 q[5];
    q[0] = make_float3(  0,   0,    1);    // q00
    q[1] = make_float3(  1,   0,  0.5);    // q10
    q[2] = make_float3(1.2,   1,  0.8);    // q11
    q[3] = make_float3(  0, 0.8, 0.85);    // q01
    q[4] = cross(q[1] - q[0], q[3] - q[2]); // qn：预计算叉积

    // 方法一：世界坐标求交
    float t, u, v;
    intersectPatchWorldCoordinates(q, ray, t, u, v);
    std::cout << std::endl;
    std::cout << "world coordinates      : ";
    std::cout << "u = " << u << "; v = " << v << "; t = " << t
              << "; intersection = " << ray.origin + t * ray.direction
              << std::endl;

    // 构造正交基并将面片变换到光线中心坐标系
    float3 axis1, axis2;
    donb(ray.direction, axis1, axis2);
    transform(ray.origin, axis1, axis2, ray.direction, q);  // 注意：不需要 q[4]

    // 方法二：光线中心坐标求交
    intersectPatchRayCentricCoordinates(q, t, u, v);
    std::cout << "ray-centric coordinates: ";
    std::cout << "u = " << u << "; v = " << v << "; t = " << t
              << "; intersection = " << ray.origin + t * ray.direction
              << std::endl;
    std::cout << std::endl;
}
```

---

## 5. 关键算法深入分析

### 5.1 两种坐标系方法的等价性

世界坐标方法和光线中心坐标方法在数学上是等价的。坐标变换不改变光线与面片的几何关系，只是将计算从三维简化为二维：

| 特性 | 世界坐标方法 | 光线中心坐标方法 |
|------|-------------|-----------------|
| 二次方程系数 | 3D 混合积（标量三重积） | 2D 叉积 |
| v 参数计算 | 3D 叉积 + 点积 | 2D 点积 |
| t 参数计算 | 3D 点积 + 除法 | 直接读取 $z$ 分量 |
| 预计算需求 | $\mathbf{q}_n$ 叉积 | 坐标变换矩阵 |
| 适用场景 | GPU 单图元求交 | 批量求交或 CPU 实现 |

### 5.2 光线中心坐标的优势

在光线中心坐标系中，光线简化为：
- 原点：$(0, 0, 0)$
- 方向：$(0, 0, 1)$

这意味着：
- 求交的 "投影" 部分只需要处理 $xy$ 平面上的二维几何
- $t$ 参数就是交点的 $z$ 坐标
- 系数计算从标量三重积（6 次乘法 + 加减法）简化为二维叉积（2 次乘法 + 1 次减法）

### 5.3 坐标变换的开销

变换 4 个顶点需要：
- 4 次平移（减法）
- 4 次矩阵-向量乘法（各 3 次点积）

总计约 48 次浮点运算。当对同一条光线测试多个面片时，变换矩阵可复用，分摊成本。

### 5.4 测试数据的设计

`main` 函数使用精心设计的测试数据：
- 面片是非平面四边形（四个角点的 $z$ 坐标各不相同）
- 光线方向 $(-3, 4, -12)$ 经归一化后不与任何坐标轴对齐
- 交点位于面片内部（非边界或角点），确保两个方法都能正确求解
- 已知交点用于反推光线原点，使验证更加可靠

---

## 6. 输入与输出

### intersectPatchWorldCoordinates

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `q` | `const float3*` | 输入 | 面片数据：`q[0..3]` 为角点，`q[4]` 为预计算叉积 |
| `ray` | `const Ray&` | 输入 | 光线（原点 + 方向） |
| `t` | `float&` | 输出 | 光线参数 $t$（无交点时为 `inf`） |
| `u` | `float&` | 输出 | 面片参数 $u \in [0, 1]$ |
| `v` | `float&` | 输出 | 面片参数 $v \in [0, 1]$ |

### intersectPatchRayCentricCoordinates

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `q` | `const float3*` | 输入 | 已变换到光线中心坐标系的面片角点 `q[0..3]` |
| `t` | `float&` | 输出 | 光线参数 $t$（无交点时为 `inf`） |
| `u` | `float&` | 输出 | 面片参数 $u \in [0, 1]$ |
| `v` | `float&` | 输出 | 面片参数 $v \in [0, 1]$ |

### main 函数预期输出

```
world coordinates      : u = 0.6; v = 0.4; t = 1; intersection = 0.648 0.368 0.748
ray-centric coordinates: u = 0.6; v = 0.4; t = 1; intersection = 0.648 0.368 0.748
```

两种方法输出相同的 $(u, v, t)$ 值和交点坐标。

---

## 7. 与其他文件的关系

| 相关文件 | 关系 |
|---------|------|
| `rqx.cu` | GPU 端实现；`intersectPatchWorldCoordinates` 是其 CPU 等价实现 |
| `garq-single-quad-example.nb` | Mathematica 笔记本，提供相同的测试数据和数值验证 |
| `garq-single-quad-example.pdf` | 上述 Mathematica 笔记本的 PDF 导出 |
| OptiX 数学库 | 提供 `float3`、`dot`、`cross`、`lerp` 等基础函数 |

---

## 8. 在光线追踪管线中的位置

```
  ┌────────────────────────────┐
  │  donb() + transform()      │  坐标变换（可选）
  └────────────────────────────┘
       |                  |
       v                  v
  ┌─────────────────┐  ┌──────────────────────────┐
  │ 世界坐标方法      │  │ 光线中心坐标方法            │
  │ (与 rqx.cu 一致) │  │ (简化的 2D 求交)           │
  └─────────────────┘  └──────────────────────────┘
       |                  |
       v                  v
     (u, v, t)         (u, v, t)
       |                  |
       +--------+---------+
                |
                v
       交点验证 / 着色计算
```

本文件是独立的 CPU 测试程序，不直接参与光线追踪管线。其主要作用是：
1. 验证 `rqx.cu` 中 GPU 算法的正确性
2. 演示光线中心坐标方法的实现
3. 作为将算法移植到其他平台的参考代码

---

## 9. 技术要点与注意事项

1. **原地修改**：`intersectPatchWorldCoordinates` 中修改了 `q00` 和 `q10`（减去光线原点），但由于它们是局部变量副本，不影响原始数据。

2. **transform 的破坏性**：`transform` 函数直接修改传入的 `q` 数组。调用后原始世界坐标数据会被覆盖。`main` 中先调用世界坐标方法，再执行变换。

3. **q[4] 不需要变换**：`transform` 默认只变换 4 个顶点（`n = 4`），因为光线中心坐标方法不需要预计算的 $\mathbf{q}_n$ 向量。

4. **pragma warning(disable: 4305)**：禁止 MSVC 关于 `double` 到 `float` 截断的警告，因为 `make_float3` 中的字面量默认为 `double`。

5. **copysignf 的可移植性**：`copysignf` 是 C99/C++11 标准函数，在所有现代编译器中可用。在 CUDA 中也有对应的设备端实现。

6. **float 精度**：使用 `float`（单精度）而非 `double`，与 GPU 端保持一致。对于大多数光线追踪应用，单精度已足够。

7. **光线中心坐标方法的数值注意**：当光线方向接近 $z$ 轴的负方向时，`donb` 中的 `sign + axis3.z` 可能接近零。Duff et al. 的方法通过 `copysignf` 确保此情况下仍有良好的数值精度。

---

## 10. 扩展阅读

- **书中章节**：Chapter 8, "Cool Patches: A Geometric Approach to Ray/Bilinear Patch Intersections"
- **正交基构造**：Duff, T., Burgess, J., Christensen, P., Hery, C., Kensler, A., Liani, M., and Villemin, R., "Building an Orthonormal Basis, Revisited," *Journal of Computer Graphics Techniques*, Vol. 6, No. 1, 2017.
- **Mathematica 验证**：同目录下的 `garq-single-quad-example.nb` 提供了符号计算验证
- **双线性面片概述**：Ramsey, S. D., Potter, K., and Hansen, C., "Ray Bilinear Patch Intersections," *Journal of Graphics Tools*, 2004.
