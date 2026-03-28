# NodeCAnyHit.hlsl -- Node-C 优化多重命中任意命中着色器

> 源文件：`Ch_09_Multi-Hit_Ray_Tracing_in_DXR/NodeCAnyHit.hlsl`

---

## 1. 文件概述

`NodeCAnyHit.hlsl` 实现了 Node-C 优化策略的任意命中着色器 (Any-Hit Shader)。Node-C 是对朴素多重命中方法的性能优化，核心思想是：当命中数据被插入到排序数组的**最后一个有效位置**（第 $N-1$ 个）时，选择性地**接受**该交点，使 DXR 运行时将 `RayTCurrent()` 作为新的光线区间端点 (Ray Interval Endpoint)。这将自动触发 BVH 节点剔除 (Node Culling)，跳过所有超出此距离的 BVH 节点。

---

## 2. 算法与数学背景

### Node-C 优化原理

在 DXR 中，任意命中着色器有两种返回方式：
- **`IgnoreHit()`**：拒绝交点，光线区间保持不变，继续遍历
- **隐式返回（不调用 `IgnoreHit()`）**：接受交点，`RayTCurrent()` 成为新的光线区间端点

Node-C 利用第二种方式：当新命中被存储在第 $N-1$ 个位置（排序数组的最远有效位置）时，接受它。此时 `RayTCurrent()` 等于第 $N$ 个最近命中的 $t$ 值，DXR 运行时会将此值作为光线的最大 $t$，从而剔除所有更远的 BVH 节点。

### 性能增益分析

假设场景中有 $M$ 个潜在交点，只需要前 $N$ 个最近的：
- **朴素方法**：必须检查所有 $M$ 个交点
- **Node-C 方法**：随着收集到更近的命中，光线区间不断缩小，大量远处的 BVH 节点被剔除

在稠密场景中，Node-C 的加速比可达数倍。

---

## 3. 代码结构概览

### 函数列表

| 函数 | 类型 | 参数 | 说明 |
|------|------|------|------|
| `mhAnyHitNodeC` | `[shader("anyhit")]` | `mhRayPayload`, `BuiltinIntersectionAttribs` | Node-C 任意命中着色器入口 |

### 与朴素方法的代码差异

本文件共 16 行，其中大部分（第 6 行 OMITTED 标记）与 `NaiveAnyHit.hlsl` 的第 5-37 行等价。唯一的新增逻辑是第 10-15 行的条件性 `IgnoreHit()` 调用。

---

## 4. 逐段代码详解

### 完整源代码（带中文注释）

```hlsl
// DXR 任意命中着色器：Node-C 优化版本
[shader("anyhit")]
void mhAnyHitNodeC(inout mhRayPayload rayPayload,        // 光线负载
     BuiltinIntersectionAttribs attribs)                   // DXR 内建相交属性
{
  // ---- 与朴素方法相同的插入排序逻辑 ----
  // OMITTED: 等价于 NaiveAnyHit.hlsl 的第 5-37 行
  // 包括：获取像素信息、计算缓冲区步长、插入排序、存储命中数据
  // 排序完成后，hi 指向新命中数据被存储的位置

  // ---- Node-C 核心逻辑：选择性 IgnoreHit ----
  // 计算新命中被存储在哪个逻辑位置（第几个命中）
  uint hitPos = hi / hitStride;  // 将缓冲区线性索引转换为命中层级索引

  // 如果命中不在最后一个有效位置，拒绝交点
  if (hitPos != gNquery - 1)
    IgnoreHit();

  // 否则（命中在第 N-1 个位置），隐式返回 —— 接受交点
  // DXR 运行时将 RayTCurrent() 设为新的光线区间端点
  // 这将触发 BVH 节点剔除，跳过所有 t > RayTCurrent() 的节点
}
```

---

## 5. 关键算法深入分析

### 5.1 条件判断的核心逻辑

```hlsl
uint hitPos = hi / hitStride;
if (hitPos != gNquery - 1)
    IgnoreHit();
```

此判断的含义：

| `hitPos` 的值 | 含义 | 行为 |
|--------------|------|------|
| `0` 到 `gNquery - 2` | 新命中被插入到前 $N-1$ 个位置之一 | 调用 `IgnoreHit()`，保持光线区间不变 |
| `gNquery - 1` | 新命中被插入到最后一个有效位置 | **不调用** `IgnoreHit()`，接受交点 |
| `>= gNquery` | 新命中超出有效范围（被丢弃） | 调用 `IgnoreHit()`（走入 `!=` 分支） |

### 5.2 为何只在最后位置接受

只有当命中存储在第 $N-1$ 个位置时才接受，是因为：

1. **`RayTCurrent()` 的值**：此时 `RayTCurrent()` 等于当前候选交点的 $t$ 值
2. **排序后的含义**：如果命中被插入到第 $N-1$ 个位置，说明排序后它是当前第 $N$ 近的命中
3. **剔除效果**：接受此交点意味着告诉 DXR "我不需要比这更远的交点了"，从而启用节点剔除

如果命中被插入到更靠前的位置（`hitPos < gNquery - 1`），接受它会错误地将光线缩短到比第 $N$ 个命中更近的位置，导致遗漏有效命中。

### 5.3 与朴素方法的行为对比

```
场景：N = 3，当前已有命中 [0.5, 1.2, 3.7]

新命中 t = 2.0:
  朴素方法:  插入排序 -> [0.5, 1.2, 2.0], IgnoreHit(), 光线区间不变
  Node-C:    插入排序 -> [0.5, 1.2, 2.0], hitPos = 2 = N-1, 接受!
             新光线区间端点 = 2.0, 剔除所有 t > 2.0 的节点

新命中 t = 0.8:
  朴素方法:  插入排序 -> [0.5, 0.8, 1.2], IgnoreHit(), 光线区间不变
  Node-C:    插入排序 -> [0.5, 0.8, 1.2], hitPos = 1 != N-1, IgnoreHit()
             光线区间不变（但第 3 近命中已从 2.0 变为 1.2）
```

### 5.4 渐进式光线区间缩短

随着遍历进行，每次在第 $N-1$ 个位置接受命中时，光线区间端点都会缩短（因为新的第 $N$ 近命中比旧的更近）。这形成了一个正反馈循环：

$$t_{\text{max}}^{(k+1)} \leq t_{\text{max}}^{(k)}$$

光线区间只会缩短不会延长，剔除效率随遍历推进而提高。

---

## 6. 输入与输出

### 输入

与 `NaiveAnyHit.hlsl` 相同，参见 [NaiveAnyHit_hlsl.md](NaiveAnyHit_hlsl.md) 第 6 节。

### 输出

| 输出 | 说明 |
|------|------|
| 全局缓冲区 | 与朴素方法相同的排序命中数据 |
| 光线区间端点 | 当命中在第 $N-1$ 位置时，隐式更新 `RayTCurrent()` |

---

## 7. 与其他文件的关系

| 相关文件 | 关系 |
|---------|------|
| `NaiveAnyHit.hlsl` | 本文件的基础版本，OMITTED 部分引用其第 5-37 行 |
| `NodeCIntersect.hlsl` | 配套的自定义相交着色器，同样利用光线区间更新实现节点剔除 |
| `NaiveIntersect.hlsl` | 朴素方法的相交着色器，不含节点剔除优化 |

---

## 8. 在光线追踪管线中的位置

```
BVH 遍历
   |
   +----> 包围盒测试通过
   |           |
   |           v
   |      相交测试（内置或自定义）
   |           |
   |           v
   |      ┌──────────────────────────────────┐
   |      │ mhAnyHitNodeC()                   │  <-- 本文件
   |      │ 1. 插入排序（与 Naive 相同）        │
   |      │ 2. if hitPos != N-1: IgnoreHit()  │
   |      │ 3. else: 接受 -> 缩短光线区间       │
   |      └──────────────────────────────────┘
   |           |
   |           v
   |      DXR 运行时根据新的 tmax 剔除远处节点
   |           |
   v           v
  遍历完成（比朴素方法更快，因为跳过了远处节点）
```

---

## 9. 技术要点与注意事项

1. **隐式接受的语义**：在 DXR 中，任意命中着色器不调用 `IgnoreHit()` 也不调用 `AcceptHitAndEndSearch()` 就是隐式接受当前交点。

2. **不能使用 AcceptHitAndEndSearch**：`AcceptHitAndEndSearch()` 会立即终止整个遍历，这与多重命中的目标（收集 $N$ 个命中）相矛盾。Node-C 只需要缩短光线区间，而非终止遍历。

3. **hitPos 的计算**：`hi / hitStride` 利用整数除法将缓冲区线性索引转换为命中层级索引。这要求 `getHitBufferIndex` 的返回值布局与 `hitStride` 对齐。

4. **OMITTED 部分的必要性**：代码中的 OMITTED 标记表示此处需要完整的插入排序实现。在实际项目中，通常通过 `#include` 或宏来共享朴素方法和 Node-C 方法之间的公共代码。

5. **正确性保证**：Node-C 方法的正确性建立在以下前提上：排序始终维护前 $N$ 个最近命中，且光线区间端点始终等于当前第 $N$ 近命中的 $t$ 值。

---

## 10. 扩展阅读

- **书中章节**：Chapter 9, "Multi-Hit Ray Tracing in DXR"，特别是 Node-C 优化部分
- **BVH 节点剔除**：理解 DXR 运行时如何利用光线区间端点进行 BVH 遍历剪枝
- **相关论文**：Gribble, C. P., Naveros, A. A., and Kerzner, E., "Multi-Hit Ray Tracing in DXR," *Ray Tracing Gems*, Chapter 9, 2019.
- **DXR 规范**：Any-Hit Shader 中 `IgnoreHit()` 与隐式接受的行为差异
