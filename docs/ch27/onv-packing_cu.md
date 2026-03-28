# onv-packing.cu 技术文档

## 文件概述

**文件路径**: `Ch_27_Interactive_Ray_Tracing_Techniques_for_High-Fidelity_Scientific_Visualization/onv-packing.cu`

本文件实现了**八面体法线向量编码 (Octahedron Normal Vector Encoding, ONV)** 的打包 (Packing) 和解包 (Unpacking) 功能。该技术将三维单位法线向量（3 个 `float`，96 位）压缩为单个 32 位无符号整数，实现 3:1 的压缩比。这在需要存储大量法线数据的场景中可以显著减少内存占用和带宽消耗，例如科学可视化中的大规模分子模型。

## 算法与数学背景

### 八面体映射 (Octahedral Mapping)

单位球面上的法线向量 $\mathbf{n} = (n_x, n_y, n_z)$ 满足 $\|\mathbf{n}\|_2 = 1$。八面体映射利用 $L^1$ 范数将球面映射到八面体表面，然后展开为二维正方形。

#### 编码过程

1. **投影到八面体表面**（$L^1$ 归一化）：

$$\mathbf{p} = \frac{\mathbf{n}}{|n_x| + |n_y| + |n_z|}$$

此时 $\mathbf{p}$ 位于 $L^1$ 单位八面体上。

2. **展开为二维**：
   - 上半球（$n_z \geq 0$）：直接使用 $(p_x, p_y)$
   - 下半球（$n_z < 0$）：使用镜像翻折：

$$\text{encoded}_x = \text{sign}(n_x) \cdot (1 - |p_y|)$$
$$\text{encoded}_y = \text{sign}(n_y) \cdot (1 - |p_x|)$$

3. **量化为 16 位整数**：将 $[-1, 1]$ 映射到 $[0, 65535]$

#### 解码过程

1. **反量化**：$[0, 65535] \to [-1, 1]$
2. **重建三维坐标**：
   - $n_z = 1 - |n_x| - |n_y|$
   - 若 $n_z < 0$（下半球），应用逆翻折
3. **归一化**（因量化误差，需重新归一化）

### 量化误差分析

使用 16 位分量（每轴 65536 级），最大角度误差约为：

$$\delta\theta \approx \frac{1}{65535} \cdot \frac{180°}{\pi} \approx 0.00087°$$

对于大多数渲染应用，这个精度完全足够。

## 代码结构概览

```
onv-packing.cu
├── OctDecode()           // 八面体解码（float2 → float3）
├── OctEncode()           // 八面体编码（float3 → float2）
├── convfloat2uint32()    // float2 → uint32 量化
├── convuint32float2()    // uint32 → float2 反量化
├── packNormal()          // 高层封装：float3 → uint
└── unpackNormal()        // 高层封装：uint → float3
```

## 逐段代码详解

### OctDecode() - 八面体解码

```cuda
static __host__ __device__ __inline__ float3 OctDecode(float2 projected) {
  float3 n = make_float3(projected.x,
                         projected.y,
                         1.0f - (fabsf(projected.x) + fabsf(projected.y)));
```
从二维坐标重建三维坐标：$n_z = 1 - |p_x| - |p_y|$

```cuda
  if (n.z < 0.0f) {
    float oldX = n.x;
    n.x = copysignf(1.0f - fabsf(n.y), oldX);
    n.y = copysignf(1.0f - fabsf(oldX), n.y);
  }
  return n;
}
```
下半球的逆翻折操作：
- $n_x = \text{sign}(n_x) \cdot (1 - |n_y|)$
- $n_y = \text{sign}(n_y) \cdot (1 - |n_x|)$

`copysignf(mag, sign)` 返回具有 `sign` 符号的 `mag` 的绝对值。

### OctEncode() - 八面体编码

```cuda
static __host__ __device__ __inline__ float2 OctEncode(float3 n) {
  const float invL1Norm = 1.0f / (fabsf(n.x) + fabsf(n.y) + fabsf(n.z));
  float2 projected;
  if (n.z < 0.0f) {
    projected = 1.0f - make_float2(fabsf(n.y), fabsf(n.x)) * invL1Norm;
    projected.x = copysignf(projected.x, n.x);
    projected.y = copysignf(projected.y, n.y);
  } else {
    projected = make_float2(n.x, n.y) * invL1Norm;
  }
  return projected;
}
```

编码过程：
1. 计算 $L^1$ 范数的倒数
2. 上半球：简单投影 $(\frac{n_x}{L^1}, \frac{n_y}{L^1})$
3. 下半球：应用翻折映射，保持符号

### convfloat2uint32() - 量化

```cuda
static __host__ __device__ __inline__ uint convfloat2uint32(float2 f2) {
  f2 = f2 * 0.5f + 0.5f;  // [-1,1] → [0,1]
  uint packed;
  packed = ((uint) (f2.x * 65535)) | ((uint) (f2.y * 65535) << 16);
  return packed;
}
```

将二维浮点数打包为 32 位整数：
- x 分量占低 16 位
- y 分量占高 16 位

### convuint32float2() - 反量化

```cuda
static __host__ __device__ __inline__ float2 convuint32float2(uint packed) {
  float2 f2;
  f2.x = (float)((packed      ) & 0x0000ffff) / 65535;
  f2.y = (float)((packed >> 16) & 0x0000ffff) / 65535;
  return f2 * 2.0f - 1.0f;  // [0,1] → [-1,1]
}
```

反向解包过程，使用位掩码和移位提取两个 16 位分量。

### packNormal() 和 unpackNormal() - 高层接口

```cuda
static __host__ __device__ __inline__ uint packNormal(const float3& normal) {
  float2 octf2 = OctEncode(normal);
  return convfloat2uint32(octf2);
}

static __host__ __device__ __inline__ float3 unpackNormal(uint packed) {
  float2 octf2 = convuint32float2(packed);
  return OctDecode(octf2);
}
```

简洁的高层封装，将整个编解码流水线串联。

## 关键算法深入分析

### 八面体映射的几何直觉

```
         +y
          ▲
         /│\
        / │ \
       /  │  \
      /   │   \
  -x ─────┼─────── +x    ← 上半球投影
      \   │   /           （z ≥ 0 的部分直接投影到 xy 平面）
       \  │  /
        \ │ /
         \│/
          ▼
         -y

下半球（z < 0）通过翻折映射到正方形的四个角
```

### 与其他法线压缩方案的比较

| 方案 | 压缩后大小 | 编码复杂度 | 解码复杂度 | 精度 |
|------|-----------|-----------|-----------|------|
| 未压缩 float3 | 96 位 | N/A | N/A | 完美 |
| 球坐标 (θ, φ) | 32-64 位 | 三角函数 | 三角函数 | 中等 |
| **八面体 (ONV)** | **32 位** | **加减乘** | **加减乘** | **高** |
| Stereographic | 32 位 | 乘除 | 乘除 | 中等 |

八面体编码的主要优势是编解码只需简单的加减乘运算，无需三角函数，GPU 上非常高效。

### `__host__ __device__` 双用途

所有函数都标记为 `__host__ __device__`，可以同时在 CPU 和 GPU 上使用：
- CPU 端：预处理阶段打包法线到几何缓冲区
- GPU 端：渲染阶段实时解包法线

## 输入与输出

### packNormal
| 名称 | 类型 | 说明 |
|------|------|------|
| `normal` | `float3` | 单位法线向量 |
| 返回值 | `uint` | 32 位压缩法线 |

### unpackNormal
| 名称 | 类型 | 说明 |
|------|------|------|
| `packed` | `uint` | 32 位压缩法线 |
| 返回值 | `float3` | 解码后的法线向量（近似单位长度） |

## 与其他文件的关系

- **`edgeoutline.cu`**：边缘描边需要法线信息，可从压缩格式解码获取
- **`aomaxdist.cu`**：AO 着色需要法线，压缩存储可节省 G-Buffer 带宽
- **`transedge.cu`**：透明度计算需要法线的点积

## 在光线追踪管线中的位置

```
几何预处理阶段：
  法线数据 (float3) → [packNormal()] → 压缩数据 (uint32) → 几何缓冲区

渲染阶段：
  几何缓冲区 → [unpackNormal()] → 法线数据 (float3) → 着色计算
```

法线压缩是一种**数据预处理和运行时解码**技术，贯穿整个渲染管线。

## 技术要点与注意事项

1. **输入归一化**：`OctEncode` 假设输入法线已归一化。非单位法线会产生错误结果
2. **解码后归一化**：由于量化误差，`OctDecode` 的结果可能不是严格的单位向量，需要时可追加归一化
3. **精度分配**：当前实现 x、y 各 16 位。如果精度需求不对称，可以调整分配
4. **`copysignf` 的零值处理**：当分量为零时，`copysignf` 的行为可能依赖实现（正零/负零）
5. **SNORM vs UNORM**：先映射到 $[0,1]$ 再量化为 UNORM 可以避免有符号整数的复杂性
6. **内存对齐**：32 位打包法线天然满足 4 字节对齐，适合 GPU 内存访问模式

## 扩展阅读

- Meyer, Q., Suessmuth, J., Sussner, G. et al. *On Floating-Point Normal Vectors*. Computer Graphics Forum, 2010
- Cigolle, Z.H. et al. *A Survey of Efficient Representations for Independent Unit Vectors*. JCGT, 2014
- Praun, H., Hoppe, H. *Spherical Parametrization and Remeshing*. SIGGRAPH 2003
