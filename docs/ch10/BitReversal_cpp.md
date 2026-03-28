# BitReversal.cpp — 32 位整数位反转

> 源文件：`Ch_10_A_Simple_Load-Balancing_Scheme_with_High_Scaling_Efficiency/BitReversal.cpp`

---

## 1. 文件概述

`BitReversal.cpp` 实现了高效的 32 位无符号整数位反转 (Bit Reversal) 函数。位反转是负载均衡置换方案的核心操作，将整数的二进制位顺序完全倒转。

---

## 2. 算法与数学背景

### 位反转定义

对于 $n$ 位整数 $x = \sum_{k=0}^{n-1} b_k \cdot 2^k$，位反转定义为：

$$\text{reverse}(x) = \sum_{k=0}^{n-1} b_k \cdot 2^{n-1-k}$$

### 分治交换策略

该实现使用经典的"蝶形交换" (Butterfly Swap) 方法，通过 5 次掩码操作完成 32 位反转：

1. 交换相邻的 1 位组
2. 交换相邻的 2 位组
3. 交换相邻的 4 位组
4. 交换相邻的 8 位组
5. 交换相邻的 16 位组

---

## 3. 完整代码与逐行注释

```cpp
// 基于掩码的位反转实现
// 参考：http://aggregate.org/MAGIC/#Bit%20Reversal

unsigned reverse(unsigned x) // 假设 32 位整数
{
    // 步骤 1：交换相邻的 1 位
    // 0xaaaaaaaa = 10101010... (偶数位掩码)
    // 0x55555555 = 01010101... (奇数位掩码)
    x = ((x & 0xaaaaaaaa) >> 1) | ((x & 0x55555555) << 1);

    // 步骤 2：交换相邻的 2 位组
    // 0xcccccccc = 11001100... (高 2 位组掩码)
    // 0x33333333 = 00110011... (低 2 位组掩码)
    x = ((x & 0xcccccccc) >> 2) | ((x & 0x33333333) << 2);

    // 步骤 3：交换相邻的 4 位组（半字节）
    // 0xf0f0f0f0 = 11110000... (高半字节掩码)
    // 0x0f0f0f0f = 00001111... (低半字节掩码)
    x = ((x & 0xf0f0f0f0) >> 4) | ((x & 0x0f0f0f0f) << 4);

    // 步骤 4：交换相邻的字节
    x = ((x & 0xff00ff00) >> 8) | ((x & 0x00ff00ff) << 8);

    // 步骤 5：交换高低 16 位
    return (x >> 16) | (x << 16);
}
```

---

## 4. 输入与输出

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `x` | `unsigned` | 输入 | 待反转的 32 位无符号整数 |
| **返回值** | `unsigned` | 输出 | 位反转后的 32 位无符号整数 |

**示例**：`reverse(0x00000001)` = `0x80000000`，`reverse(0x12345678)` = `0x1E6A2C48`

---

## 5. 关键算法深入分析

### 时间复杂度

$O(\log n)$ — 对于 $n$ 位整数，需要 $\log_2 n$ 步交换操作。32 位需要 5 步。

### 空间复杂度

$O(1)$ — 仅使用常数个寄存器。

### 与硬件指令的对比

某些 CPU 提供专用位反转指令（如 ARM 的 `RBIT`），但 x86 架构无直接支持。此掩码方法是跨平台的便携实现。

---

## 6. 与其他文件的关系

```
BitReversal.cpp
  ← DistributionScheme.cpp (调用 reverse() 进行像素分配)
  ← ImageAssembly.cpp (调用 reverse() 进行图像重组装)
```

---

## 7. 扩展阅读

- **书中章节**：Chapter 10, "A Simple Load-Balancing Scheme with High Scaling Efficiency"
- **位操作技巧**：Anderson, S., "Bit Twiddling Hacks," Stanford Graphics, 2005.
- **参考来源**：http://aggregate.org/MAGIC/#Bit%20Reversal
