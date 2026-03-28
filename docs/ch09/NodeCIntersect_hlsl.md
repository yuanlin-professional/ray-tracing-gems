# NodeCIntersect.hlsl -- Node-C 优化多重命中自定义相交着色器

> 源文件：`Ch_09_Multi-Hit_Ray_Tracing_in_DXR/NodeCIntersect.hlsl`

---

## 1. 文件概述

`NodeCIntersect.hlsl` 实现了 Node-C 优化策略的自定义相交着色器 (Intersection Shader)。与朴素方法的 `NaiveIntersect.hlsl` 相比，本文件增加了关键的光线区间端点更新逻辑：当新命中落入有效范围 $[0, N-1]$ 内时，调用 `ReportHit` 并以排序数组中第 $N$ 个最近命中的 $t$ 值作为参数，从而触发 BVH 节点剔除 (BVH Node Culling)。

---

## 2. 算法与数学背景

### ReportHit 的双重作用

在 DXR 中，`ReportHit(tHit, hitKind, attribs)` 通常用于向运行时报告一个自定义相交。但在 Node-C 方法中，它被巧妙地用于**更新光线区间端点**：

$$t_{\text{reported}} = \text{gHitT}[\text{lastIdx}] = \text{第 } N \text{ 近命中的 } t \text{ 值}$$

当 DXR 运行时接受此报告后，`RayTCurrent()` 将被更新为 $t_{\text{reported}}$，从而缩短光线区间。后续的 BVH 遍历将跳过所有 $t > t_{\text{reported}}$ 的节点。

### 有效范围判断

只有当新命中的插入位置在 $[0, N-1]$ 范围内时（即 `hitPos < gNquery`），才需要调用 `ReportHit`。如果新命中被插到第 $N$ 个位置或更后面，说明它比当前所有 $N$ 个最近命中都远，不会改变排序数组的有效部分，因此无需更新光线区间。

---

## 3. 代码结构概览

### 函数列表

| 函数 | 类型 | 参数 | 说明 |
|------|------|------|------|
| `mhIntersectNodeC` | `[shader("intersection")]` | 无（使用 DXR 系统值） | Node-C 自定义相交着色器入口 |

### 与朴素方法的代码差异

本文件共 21 行，OMITTED 部分与 `NaiveIntersect.hlsl` 的第 9-20 行等价。新增的逻辑是第 13-19 行的条件性 `ReportHit` 调用。

---

## 4. 逐段代码详解

### 完整源代码（带中文注释）

```hlsl
// DXR 自定义相交着色器：Node-C 优化版本
[shader("intersection")]
void mhIntersectNodeC()
{
  HitAttribs hitAttrib;  // 相交属性结构体

  // 执行自定义三角形相交测试
  uint nhits = intersectTriangle(PrimitiveIndex(), hitAttrib);

  if (nhits > 0)  // 相交成功
  {
    // ---- 与朴素方法相同的处理 ----
    // OMITTED: 等价于 NaiveIntersect.hlsl 的第 9-20 行
    // 包括：获取像素信息、计算缓冲区步长、插入排序、更新 gHitCount
    // 排序完成后，hi 指向新命中被存储的缓冲区位置

    // ---- Node-C 核心逻辑：条件性 ReportHit ----
    // 计算新命中被存储在哪个逻辑位置
    uint hitPos = hi / hitStride;

    // 仅当新命中落入有效范围 [0, Nquery-1] 时更新光线区间
    if (hitPos < gNquery)
    {
      // 获取排序后第 N 个最近命中（最后一个有效位置）的缓冲区索引
      uint lastIdx =
          getHitBufferIndex(gNquery - 1, pixelIdx, pixelDims);

      // 以第 N 个最近命中的 t 值调用 ReportHit
      // 这将触发配套的 Any-Hit Shader (mhAnyHitNodeC)
      // 并可能将 RayTCurrent() 更新为 gHitT[lastIdx]
      ReportHit(gHitT[lastIdx], 0, hitAttrib);
    }
  }
}
```

---

## 5. 关键算法深入分析

### 5.1 ReportHit 的参数解析

```hlsl
ReportHit(gHitT[lastIdx], 0, hitAttrib);
```

| 参数 | 值 | 含义 |
|------|-----|------|
| `tHit` | `gHitT[lastIdx]` | 报告的命中距离 = 第 $N$ 个最近命中的 $t$ 值 |
| `hitKind` | `0` | 命中类型标识（用于 Any-Hit Shader 中区分） |
| `attribs` | `hitAttrib` | 相交属性（传递给 Any-Hit Shader） |

注意：这里报告的 $t$ 值不是当前三角形相交的 $t$ 值，而是排序后第 $N$ 近命中的 $t$ 值。这是 Node-C 方法的关键技巧。

### 5.2 ReportHit 与 NodeCAnyHit 的协作

`ReportHit` 调用后，DXR 运行时会触发配套的任意命中着色器 `mhAnyHitNodeC`。在该着色器中：

1. 如果命中被存储在第 $N-1$ 个位置（`hitPos == gNquery - 1`），隐式接受
2. DXR 运行时将 `gHitT[lastIdx]` 作为新的 `RayTCurrent()`
3. 后续遍历的有效范围缩短为 $[t_{\min}, \text{gHitT[lastIdx]}]$

### 5.3 hitPos 条件判断的必要性

```hlsl
if (hitPos < gNquery)
```

| `hitPos` 值 | 含义 | 行为 |
|------------|------|------|
| `0` 到 `gNquery - 1` | 新命中在有效范围内 | 调用 `ReportHit` 更新光线区间 |
| `>= gNquery` | 新命中在有效范围外 | 不调用 `ReportHit`，光线区间不变 |

当 `hitPos >= gNquery` 时，新命中比所有已存储的 $N$ 个命中都远，不会影响最终结果。此时排序数组中第 $N$ 个位置的 $t$ 值没有变化，无需更新光线区间。

### 5.4 与 NaiveIntersect 的对比

| 特性 | NaiveIntersect | NodeCIntersect |
|------|---------------|----------------|
| 插入排序 | 有 | 有（相同） |
| gHitCount 更新 | 有 | 有（OMITTED 中） |
| ReportHit 调用 | 无 | 条件性调用 |
| 光线区间更新 | 不更新 | 缩短为第 $N$ 近命中的 $t$ |
| BVH 节点剔除 | 无 | 有（通过缩短光线区间） |

### 5.5 lastIdx 的计算

```hlsl
uint lastIdx = getHitBufferIndex(gNquery - 1, pixelIdx, pixelDims);
```

`lastIdx` 指向排序数组中第 $N$ 个位置（索引 $N-1$）。由于数组按 $t$ 值从小到大排序，`gHitT[lastIdx]` 始终是当前 $N$ 个最近命中中距离最远的那个。

---

## 6. 输入与输出

### 输入

| 来源 | 说明 |
|------|------|
| `PrimitiveIndex()` | 当前测试的图元索引 |
| `DispatchRaysIndex()` | 当前像素索引 |
| `DispatchRaysDimensions()` | 分派维度 |
| `gHitT[lastIdx]` | 排序后第 $N$ 个最近命中的 $t$ 值 |

### 输出

| 输出 | 说明 |
|------|------|
| 全局缓冲区 | 与朴素方法相同的排序命中数据和命中计数 |
| `ReportHit` | 向 DXR 运行时报告新的光线区间端点 |

---

## 7. 与其他文件的关系

| 相关文件 | 关系 |
|---------|------|
| `NodeCAnyHit.hlsl` | 配套的任意命中着色器，被本文件的 `ReportHit` 触发 |
| `NaiveIntersect.hlsl` | 本文件的基础版本，OMITTED 部分引用其第 9-20 行 |
| `NaiveAnyHit.hlsl` | 朴素方法的任意命中着色器，提供插入排序的完整实现参考 |

### 配合关系图

```
mhIntersectNodeC()
   |
   +---> 插入排序（与 Naive 相同）
   |
   +---> ReportHit(gHitT[lastIdx], ...)
              |
              v
         mhAnyHitNodeC()           <-- NodeCAnyHit.hlsl
              |
              +---> hitPos != N-1 ? IgnoreHit()
              |
              +---> hitPos == N-1 ? 隐式接受 -> 缩短光线区间
```

---

## 8. 在光线追踪管线中的位置

```
BVH 遍历
   |
   +----> 包围盒测试通过
   |           |
   |           v
   |      ┌──────────────────────────────────┐
   |      │ mhIntersectNodeC()                │  <-- 本文件
   |      │ 1. intersectTriangle()            │
   |      │ 2. 插入排序 + gHitCount++         │
   |      │ 3. if hitPos < N:                 │
   |      │      ReportHit(gHitT[lastIdx])    │
   |      └──────────────────────────────────┘
   |           |
   |           v (ReportHit 触发)
   |      ┌──────────────────────────────────┐
   |      │ mhAnyHitNodeC()                   │
   |      │ 条件性 IgnoreHit / 接受            │
   |      └──────────────────────────────────┘
   |           |
   |           v
   |      DXR 运行时可能缩短光线区间
   |      剔除远处的 BVH 节点
   |
   v
  遍历完成（高效，因为远处节点被剔除）
```

---

## 9. 技术要点与注意事项

1. **ReportHit 的非传统用法**：通常 `ReportHit` 用于报告实际的几何相交，但这里用于传递排序后的第 $N$ 近 $t$ 值。这是一种对 DXR API 的创造性使用。

2. **ReportHit 与 IgnoreHit 的交互**：`ReportHit` 触发 Any-Hit Shader 后，如果 Any-Hit Shader 调用 `IgnoreHit()`，则报告被拒绝，光线区间不变。只有当 Any-Hit Shader 隐式接受时，光线区间才会更新。

3. **线程安全**：插入排序和 `ReportHit` 都在单条光线的上下文中执行，不存在跨线程竞争。

4. **OMITTED 部分**：与 `NaiveIntersect.hlsl` 一样，OMITTED 部分必须在实际代码中补全。本文件额外需要保留 `hi`、`hitStride`、`pixelIdx`、`pixelDims` 变量供后续的 Node-C 逻辑使用。

5. **gHitT[lastIdx] 的时序**：`ReportHit` 使用的 `gHitT[lastIdx]` 是在插入排序完成后读取的，因此它反映了包含当前新命中在内的排序结果。

6. **与 NodeCAnyHit 的匹配**：本文件必须与 `NodeCAnyHit.hlsl` 配合使用。如果误配 `NaiveAnyHit.hlsl`，由于后者始终调用 `IgnoreHit()`，`ReportHit` 永远被拒绝，光线区间永不缩短，优化失效。

---

## 10. 扩展阅读

- **书中章节**：Chapter 9, "Multi-Hit Ray Tracing in DXR"，Node-C 方法的详细推导和性能分析
- **DXR ReportHit 规范**：Microsoft DirectX Raytracing Functional Spec - Intersection Shader, `ReportHit` intrinsic
- **BVH 遍历与剪枝**：理解光线区间如何影响 BVH 遍历的早期终止行为
- **作者网站**：完整的可执行示例和额外资源见 [http://www.rtvtk.org/~cgribble/research/DXR-MultiHitRayTracing/](http://www.rtvtk.org/~cgribble/research/DXR-MultiHitRayTracing/)
