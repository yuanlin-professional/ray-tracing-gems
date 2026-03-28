# DistributionScheme.cpp — 像素分配置换方案

> 源文件：`Ch_10_A_Simple_Load-Balancing_Scheme_with_High_Scaling_Efficiency/DistributionScheme.cpp`

---

## 1. 文件概述

`DistributionScheme.cpp` 描述了基于位反转置换的像素分配方案伪代码。它将连续的像素索引映射到图像中均匀分布的位置，使得每个处理器处理的像素在空间上分散，从而实现负载均衡。

---

## 2. 算法与数学背景

### 置换公式

将连续像素索引 $i$ 映射为分散的图像像素索引 $j$：

$$f = \lfloor i / s \rfloor, \quad p = i \bmod s$$

$$j = (\text{reverse}(f) \gg \text{bits}) + p$$

其中：
- $n = \text{width} \times \text{height}$：总像素数
- $m = 2^b$：区域数（$b$ 位地址空间）
- $s = \lceil n / m \rceil$：每个区域的像素数
- $\text{bits} = 32 - b$：右移位数，将 32 位反转结果缩放到 $b$ 位

### 处理器分配

每个处理器 $k$ 按相对速度 $w_k$ 处理 $\lfloor w_k \cdot m \rceil$ 个区域，起始区域索引为 $\text{base} = \sum_{l=0}^{k-1} \lfloor w_l \cdot m \rceil$。

---

## 3. 完整代码与逐行注释

```cpp
// 像素分配置换方案伪代码

const unsigned n = image.width() * image.height();  // 总像素数
const unsigned m = 1u << b;                          // 区域数 = 2^b
const unsigned s = (n + m - 1) / m;                  // 每区域像素数（向上取整）
const unsigned bits = (sizeof(unsigned) * CHAR_BIT) - b;  // 32 - b

// 假设处理器 k 的相对速度为 w_k，
// 它处理 ⌊w_k × m⌉ 个区域，
// 起始区域 base = Σ_{l=0}^{k-1} ⌊w_l × m⌉

// 在处理器 k 上，连续像素块中的每个索引 i 经过以下置换分配到图像中：
const unsigned f = i / s;                  // 区域索引
const unsigned p = i % s;                  // 区域内偏移
const unsigned j = (reverse(f) >> bits) + p;  // 置换后的图像像素索引

// 填充像素（超出图像范围）被忽略
if (j < n)
    image[j] = render(j);  // 渲染像素 j 并写入图像
```

---

## 4. 输入与输出

| 变量 | 类型 | 说明 |
|------|------|------|
| `n` | `unsigned` | 图像总像素数 |
| `b` | `unsigned` | 区域地址位数 |
| `m` | `unsigned` | 区域总数 $2^b$ |
| `s` | `unsigned` | 每区域像素数 |
| `i` | `unsigned` | 连续像素索引（处理器本地） |
| `j` | `unsigned` | 置换后的图像像素索引 |

---

## 5. 应用场景

- **分布式光线追踪**：多台机器/多 GPU 协作渲染同一图像
- **异构集群**：不同速度的处理器通过 $w_k$ 权重自适应分配
- **渐进式渲染**：先渲染分散的像素快速产生预览

---

## 6. 关键算法深入分析

### 为什么使用位反转

位反转将连续索引 $0, 1, 2, 3, ...$ 映射为 $0, m/2, m/4, 3m/4, ...$（类似 Van der Corput 序列），使得连续分配的区域在图像中均匀分散。

### 负载均衡原理

由于渲染代价与像素位置相关（天空区域快、复杂几何区域慢），将每个处理器的像素均匀分散可使得每个处理器大概率包含相似比例的"难"和"容易"像素。

### 填充处理

当 $n$ 不是 $m$ 的整数倍时，$s \times m > n$，多余的索引 ($j \geq n$) 被跳过。

---

## 7. 与其他文件的关系

```
BitReversal.cpp ─→ reverse() 函数
        ↓
DistributionScheme.cpp ← 本文件：使用 reverse() 进行像素分配
        ↓
ImageAssembly.cpp ← 渲染完成后的图像重组装
```

---

## 8. 扩展阅读

- **书中章节**：Chapter 10, "A Simple Load-Balancing Scheme with High Scaling Efficiency"
- **Van der Corput 序列**：与位反转密切相关的低差异序列
