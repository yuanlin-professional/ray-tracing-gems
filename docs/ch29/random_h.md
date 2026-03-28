# random.h 技术文档

## 文件概述

`random.h` 提供了 GPU 和 CPU 端的伪随机数生成器 (PRNG) 集合，包括 TEA 哈希、线性同余发生器 (LCG)、Multiply-With-Carry (MWC) 等算法。在粒子体渲染中，`tea<16>()` 用于生成每像素的随机种子。

**源文件路径**: `Ch_29_Efficient_Particle_Volume_Splatting_in_a_Ray_Tracer/device_include/random.h`

## 算法与数学背景

### TEA (Tiny Encryption Algorithm) 哈希

TEA 是一种对称加密算法，此处用作哈希函数将 (像素索引, 帧号) 映射为伪随机种子：

```
for n = 0 to N-1:
    s0 += 0x9e3779b9  (黄金比例的定点表示)
    v0 += ((v1<<4) + 0xa341316c) ^ (v1+s0) ^ ((v1>>5) + 0xc8013ea4)
    v1 += ((v0<<4) + 0xad90777d) ^ (v0+s0) ^ ((v0>>5) + 0x7e95761e)
return v0
```

`tea<16>` 表示执行 16 轮 Feistel 网络，提供足够的随机性。

### LCG (Linear Congruential Generator)

$$x_{n+1} = (a \cdot x_n + c) \mod 2^{32}$$

常量 $a = 1664525$, $c = 1013904223$ (Numerical Recipes 推荐值)。

输出仅取高 24 位（通过 `& 0x00FFFFFF`）以避免低位的周期性。

### MWC (Multiply-With-Carry)

使用 4 个状态变量的 MWC 随机数发生器，周期约为 $2^{125}$：

$$\text{sum} = 2111111111 \cdot r_3 + 1492 \cdot r_2 + 1776 \cdot r_1 + 5115 \cdot r_0 + \text{carry}$$

## 完整源码关键部分

```cuda
// TEA 哈希：将两个整数混合成一个伪随机数
template<unsigned int N>
static __host__ __device__ __inline__ unsigned int tea(unsigned int val0, unsigned int val1)
{
    unsigned int v0 = val0, v1 = val1, s0 = 0;
    for (unsigned int n = 0; n < N; n++)
    {
        s0 += 0x9e3779b9;
        v0 += ((v1<<4)+0xa341316c) ^ (v1+s0) ^ ((v1>>5)+0xc8013ea4);
        v1 += ((v0<<4)+0xad90777d) ^ (v0+s0) ^ ((v0>>5)+0x7e95761e);
    }
    return v0;
}

// LCG 伪随机数：返回 [0, 2^24) 的无符号整数
static __host__ __device__ __inline__ unsigned int lcg(unsigned int &prev)
{
    const unsigned int LCG_A = 1664525u;
    const unsigned int LCG_C = 1013904223u;
    prev = (LCG_A * prev + LCG_C);
    return prev & 0x00FFFFFF;
}

// 生成 [0, 1) 的浮点随机数
static __host__ __device__ __inline__ float rnd(unsigned int &prev)
{
    return ((float) lcg(prev) / (float) 0x01000000);
}

// 种子旋转：用帧号扰动种子
static __host__ __device__ __inline__ unsigned int rot_seed(unsigned int seed, unsigned int frame)
{
    return seed ^ frame;
}
```

## 在粒子体渲染中的使用

在 `raygen.cu` 中：
```cuda
unsigned int seed = tea<16>(screen.x * launch_index.y + launch_index.x, frame);
```

这为每个像素在每一帧生成唯一的随机种子，用于子像素抖动（如果启用）。

## 输入与输出

**输入**: 像素索引、帧号、种子状态

**输出**: 伪随机数序列

## 与其他文件的关系

- 被 `raygen.cu` 包含和使用
- 来自 OptiX SDK 标准设备端库

## 技术要点与注意事项

1. `tea<16>` 的 16 轮保证了足够的随机质量，但也有一定计算开销
2. LCG 状态通过引用传递（`unsigned int &prev`），支持连续多次采样
3. MWC 仅在 Host 端（`__host__`）可用，不在 GPU 上运行
4. `0x9e3779b9` 是 $\lfloor 2^{32} / \phi \rfloor$（黄金比例），用于消除系统性偏差
5. `rnd()` 的输出范围是 $[0, 1)$，不含 1.0

## 扩展阅读

- "Tiny Encryption Algorithm" (Wheeler & Needham, 1994)
- "Numerical Recipes" -- LCG 参数选择
- "Hash Functions for GPU Rendering" (Jarzynski & Olano, 2020)
