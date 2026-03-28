# ImageAssembly.cpp — 渲染后图像重组装

> 源文件：`Ch_10_A_Simple_Load-Balancing_Scheme_with_High_Scaling_Efficiency/ImageAssembly.cpp`

---

## 1. 文件概述

`ImageAssembly.cpp` 实现了渲染完成后的图像重组装逻辑。由于像素在渲染阶段被置换分配，完成后需要将像素恢复到正确位置。该置换具有自逆 (involutory) 性质，可通过简单的成对交换原地完成。

---

## 2. 算法与数学背景

### 自逆置换 (Involutory Permutation)

本方案的关键优势在于置换函数 $\sigma$ 是自逆的：

$$\sigma(\sigma(i)) = i$$

即：像素 $i$ 映射到位置 $j$，像素 $j$ 也映射到位置 $i$。因此重组装只需遍历所有 $(i, j)$ 对并交换即可。

---

## 3. 完整代码与逐行注释

```cpp
// 图像重组装

// 将像素索引 i 映射为置换后的索引 j
const unsigned f = i / s;
const unsigned p = i % s;
const unsigned j = (reverse(f) >> bits) + p;

// 置换是自逆的：像素 j 置换回到像素 i
// 因此原地反向置换只需交换置换对
if (j > i)                     // 避免重复交换和自映射
    swap(image[i], image[j]);  // 交换像素位置
```

---

## 4. 输入与输出

| 变量 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `i` | `unsigned` | 输入 | 当前遍历的像素索引 |
| `j` | `unsigned` | 计算 | 置换目标索引 |
| `image[]` | 像素数组 | 输入/输出 | 原地交换重组装 |

---

## 5. 关键算法深入分析

### 为什么 `j > i`

- `j == i`：像素映射到自身，无需操作
- `j > i`：当前遍历到 $(i, j)$ 对，执行交换
- `j < i`：该对已经在遍历到 $j$ 时处理过，跳过以避免重复交换

### 时间复杂度

$O(n)$ — 每个像素恰好被访问一次或两次。

### 空间复杂度

$O(1)$ — 原地交换，不需要额外缓冲区。

---

## 6. 与其他文件的关系

```
渲染阶段                      重组装阶段
DistributionScheme.cpp  →  ImageAssembly.cpp
(置换分配渲染)              (原地反向置换)
        ↑                        ↑
   BitReversal.cpp          BitReversal.cpp
   (reverse 函数)            (reverse 函数)
```

---

## 7. 扩展阅读

- **书中章节**：Chapter 10, "A Simple Load-Balancing Scheme with High Scaling Efficiency"
- **自逆置换**：满足 $\sigma = \sigma^{-1}$ 的置换，也称对合 (involution)
