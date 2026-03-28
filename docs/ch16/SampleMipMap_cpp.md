# SampleMipMap.cpp 文档

## 1. 变换概述

| 项目 | 内容 |
|------|------|
| **名称** | MipMap 层级采样 (MipMap Hierarchical Sampling) |
| **用途** | 利用纹理的 MipMap 层级结构，按纹素 (texel) 亮度值成比例地采样一个纹素坐标 |
| **应用场景** | 环境贴图 (environment map) 重要性采样、发光纹理 (emissive texture) 采样、光源选择 |

---

## 2. 数学推导

### 2.1 基本思想

MipMap 是纹理的金字塔结构：
- Mip 级别 0：原始分辨率 $N \times N$
- Mip 级别 $k$：$N/2^k \times N/2^k$
- 最高 Mip 级别：$1 \times 1$（整个纹理的总和/平均值）

### 2.2 层级采样过程

从最粗的 $2 \times 2$ Mip 级别开始，逐级向下细化：

在每一级，当前纹素被分为 4 个子纹素（2x2）。先水平方向选择左/右，再垂直方向选择上/下：

**水平方向选择**：
- $\text{left} = T(x, y) + T(x, y+1)$（左列两个纹素之和）
- $\text{right} = T(x+1, y) + T(x+1, y+1)$（右列两个纹素之和）
- $p_L = \frac{\text{left}}{\text{left} + \text{right}}$

若 $u_0 < p_L$，选左列并重映射 $u_0' = u_0 / p_L$；否则选右列。

**垂直方向选择**（在已选的列内）：
- $p_{\text{lower}} = T(x, y) / \text{column\_sum}$

若 $u_1 < p_{\text{lower}}$，选下行并重映射 $u_1' = u_1 / p_{\text{lower}}$；否则选上行。

### 2.3 PDF 计算

最终纹素 $(x, y)$ 被选中的概率（PDF）为：

$$
\text{pdf} = \frac{T(x, y, 0)}{T(0, 0, \text{maxMip})}
$$

即该纹素值占总纹理总和的比例。

---

## 3. 完整代码与逐行注释

```cpp
int2 SampleMipMap(Texture& T, float u[2], float *pdf)
{
    // 从最粗 mip 级别（2×2）遍历到最细级别（原始分辨率）
    // load(x,y,mip) 加载一个纹素（mip 0 为最大的 2 的幂级别）
    int x = 0, y = 0;
    // 初始纹素坐标为 (0,0)

    for (int mip = T.maxMip()-1; mip >= 0; --mip) {
    // 从次最粗级别开始（因为最粗级别是 1×1，只有一个值）
    // 每级将分辨率翻倍

        x <<= 1; y <<= 1;
        // 坐标左移1位 = ×2，映射到更细一级的对应位置

        float left = T.load(x, y, mip) + T.load(x, y+1, mip);
        // 左列：当前位置及其上方纹素的值之和

        float right = T.load(x+1, y, mip) + T.load(x+1, y+1, mip);
        // 右列：右侧及其上方纹素的值之和

        float probLeft = left / (left + right);
        // 选择左列的概率

        if (u[0] < probLeft) {
            // 选择左列
            u[0] /= probLeft;
            // 重映射 u[0] 到 [0,1)

            float probLower = T.load(x, y, mip) / left;
            // 在左列中选择下方纹素的概率

            if (u[1] < probLower) {
                u[1] /= probLower;
                // 选择下方纹素，重映射 u[1]
            }
            else {
                y++;
                // 选择上方纹素
                u[1] = (u[1] - probLower) / (1.0f - probLower);
                // 重映射 u[1]
            }
        }
        else {
            x++;
            // 选择右列
            u[0] = (u[0] - probLeft) / (1.0f - probLeft);
            // 重映射 u[0]

            float probLower = T.load(x, y, mip) / right;
            // 在右列中选择下方纹素的概率

            if (u[1] < probLower) {
                u[1] /= probLower;
                // 选择下方纹素
            }
            else {
                y++;
                // 选择上方纹素
                u[1] = (u[1] - probLower) / (1.0f - probLower);
                // 重映射 u[1]
            }
        }
    }
    // 已找到概率正比于归一化纹素值的纹素 (x,y)
    // 计算 PDF 并返回坐标
    *pdf = T.load(x, y, 0) / T.load(0, 0, T.maxMip());
    // PDF = 该纹素值 / 总纹理值（最粗 mip 的唯一纹素即为总和）

    return int2(x, y);
    // 返回被选中的纹素坐标
}
```

---

## 4. 输入与输出

### 输入参数

| 参数 | 类型 | 范围 | 说明 |
|------|------|------|------|
| `T` | `Texture&` | — | 纹理对象，需提供 `load(x,y,mip)` 和 `maxMip()` 接口 |
| `u[0]` | `float` | $[0, 1)$ | 均匀随机变量，控制水平方向选择 |
| `u[1]` | `float` | $[0, 1)$ | 均匀随机变量，控制垂直方向选择 |

### 输出

| 输出 | 类型 | 说明 |
|------|------|------|
| 返回值 | `int2` | 被选中的纹素坐标 $(x, y)$ |
| `*pdf` | `float` | 选中该纹素的概率密度 |

---

## 5. 应用场景

- **环境贴图重要性采样**：按照环境贴图的亮度分布采样入射光方向，大幅降低全局光照计算的方差
- **发光纹理采样**：对带有纹理的面光源，按纹理亮度采样发光点
- **光子映射中的光子发射**：根据发光分布从面光源上发射光子
- **自适应采样**：在误差图 (error map) 上按误差大小分配采样资源

---

## 6. 与相关变换的对比

| 方法 | 预处理 | 采样复杂度 | 采样精度 | 内存开销 |
|------|--------|-----------|---------|---------|
| **MipMap 层级采样 (本方法)** | MipMap 构建 $O(N^2)$ | $O(\log N)$ | 纹素级别 | MipMap 本身（约 $4/3 \times N^2$） |
| **2D CDF 表** | $O(N^2)$ | $O(\log N)$ 二分查找 | 纹素级别 | 额外 $O(N^2)$ CDF 存储 |
| **拒绝采样** (`TextureRejectionSampling.cpp`) | 无 | 期望 $O(1/\bar{b})$ | 连续（不限纹素） | 无 |
| **层级扭曲** (`HierarchicalWarping.cpp`) | 二叉树 $O(N^2)$ | $O(\log N^2)$ | 叶节点级别 | 二叉树节点 |

> **GPU 实现提示**：在 GPU 上，MipMap 已经是纹理硬件原生支持的结构，无需额外预处理。`T.load(x, y, mip)` 可直接使用 `texelFetch` 指令实现，使得本方法特别适合 GPU 实现。
