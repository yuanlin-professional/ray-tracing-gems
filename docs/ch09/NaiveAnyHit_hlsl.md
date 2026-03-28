# NaiveAnyHit.hlsl -- 朴素多重命中任意命中着色器

> 源文件：`Ch_09_Multi-Hit_Ray_Tracing_in_DXR/NaiveAnyHit.hlsl`

---

## 1. 文件概述

`NaiveAnyHit.hlsl` 实现了 DXR (DirectX Raytracing) 中用于多重命中光线追踪 (Multi-Hit Ray Tracing) 的朴素任意命中着色器 (Any-Hit Shader)。该着色器在光线遍历 BVH (Bounding Volume Hierarchy) 时对每个候选交点调用，通过插入排序 (Insertion Sort) 将命中数据维护在全局缓冲区中，始终保持前 $N$ 个最近命中按距离排序。

核心策略：始终调用 `IgnoreHit()` 拒绝交点，使光线继续遍历所有可能的几何体，确保不遗漏任何候选命中。

---

## 2. 算法与数学背景

### 插入排序

对于每个新的候选命中点 $t_{\text{val}}$，在已排序的命中数组中从后向前查找插入位置：

1. 从数组末尾（第 $\min(N_{\text{hits}}, N_{\text{query}})$ 个位置）开始
2. 将所有 $t > t_{\text{val}}$ 的元素向后移动一个位置
3. 在空出的位置插入新命中数据

时间复杂度为 $O(N)$，其中 $N$ = `gNquery`（查询的命中数量）。

### 像素交错缓冲区

缓冲区使用步长 (Stride) 寻址，步长为像素总数 $W \times H$：

$$\text{index}(hitIdx, pixelIdx) = hitIdx \times (W \times H) + \text{pixelLinearIdx}$$

相邻像素的同一命中层级数据在内存中连续，有利于 GPU 合并访问 (Coalesced Access)。

---

## 3. 代码结构概览

### 函数列表

| 函数 | 类型 | 参数 | 说明 |
|------|------|------|------|
| `mhAnyHitNaive` | `[shader("anyhit")]` | `mhRayPayload`, `BuiltinIntersectionAttribs` | 朴素任意命中着色器入口 |

### 依赖的全局资源

| 名称 | 类型 | 说明 |
|------|------|------|
| `gHitT` | `RWBuffer<float>` | 存储命中点的光线参数 $t$ |
| `gHitDiffuse` | `RWBuffer<float4>` | 存储命中点的漫反射颜色 |
| `gHitNdotV` | `RWBuffer<float>` | 存储命中点的法线-视线夹角余弦值 |
| `gNquery` | `uint` | 需要收集的最大命中数 $N$ |

### 依赖的辅助函数

| 函数 | 说明 |
|------|------|
| `getHitBufferIndex(hitIdx, pixelIdx, pixelDims)` | 计算像素交错缓冲区中的线性索引 |
| `getDiffuseSurfaceColor(primIdx)` | 获取指定图元的漫反射颜色 |
| `getGeometricFaceNormal(primIdx)` | 获取指定图元的几何面法线 |

---

## 4. 逐段代码详解

### 完整源代码（带中文注释）

```hlsl
// DXR 任意命中着色器：在每次候选相交时被调用
[shader("anyhit")]
void mhAnyHitNaive(inout mhRayPayload rayPayload,       // 光线负载（包含命中计数）
                   BuiltinIntersectionAttribs attribs)    // DXR 内建相交属性
{
  // ---- 获取当前候选交点信息 ----
  uint2 pixelIdx  = DispatchRaysIndex();         // 当前像素的二维索引 (x, y)
  uint2 pixelDims = DispatchRaysDimensions();    // 光线分派的总像素维度 (W, H)
  uint  hitStride = pixelDims.x*pixelDims.y;     // 缓冲区步长 = W * H（像素总数）
  float tval      = RayTCurrent();               // 当前候选交点的光线参数 t

  // ---- 插入排序：找到候选交点的插入位置 ----
  // hi 指向当前应检查的位置（从数组末尾开始）
  uint hi = getHitBufferIndex(min(rayPayload.nhits, gNquery),
                              pixelIdx, pixelDims);
  uint lo = hi - hitStride;  // lo 指向 hi 的前一个位置

  // 从后向前扫描，将 t 值更大的元素向后移动
  while (hi > 0 && tval < gHitT[lo])
  {
    // 将 lo 位置的数据移动到 hi 位置
    gHitT      [hi] = gHitT      [lo];    // 移动 t 值
    gHitDiffuse[hi] = gHitDiffuse[lo];    // 移动漫反射颜色
    gHitNdotV  [hi] = gHitNdotV  [lo];    // 移动法线-视线余弦值

    // 继续向前检查
    hi -= hitStride;   // hi 前移一个位置
    lo -= hitStride;   // lo 前移一个位置
  }

  // ---- 获取当前命中点的表面属性 ----
  uint   primIdx = PrimitiveIndex();                       // 当前图元索引
  float4 diffuse = getDiffuseSurfaceColor(primIdx);        // 漫反射颜色
  float3 Ng      = getGeometricFaceNormal(primIdx);        // 几何面法线

  // ---- 在找到的位置存储命中数据 ----
  // 注意：可能存储在第 Nquery 个位置（超出有效范围），
  // 这些数据会在后续被更近的命中覆盖或被忽略
  gHitT      [hi] = tval;                                   // 存储 t 值
  gHitDiffuse[hi] = diffuse;                                // 存储漫反射颜色
  gHitNdotV  [hi] =
      abs(dot(normalize(Ng), normalize(WorldRayDirection()))); // |N . D|

  // 递增总命中计数
  ++rayPayload.nhits;

  // 拒绝该交点并继续遍历（保持光线区间不变）
  IgnoreHit();
}
```

---

## 5. 关键算法深入分析

### 5.1 插入排序的缓冲区操作

插入排序使用步长寻址在像素交错缓冲区上操作。假设 `hitStride = 1000`（1000 个像素），当前像素的命中数据分布在索引 `pixelLinearIdx, pixelLinearIdx + 1000, pixelLinearIdx + 2000, ...` 处。

```
缓冲区位置:    [0]    [1000]   [2000]   [3000]   <- 像素 0 的命中层
存储的 t 值:   0.5    1.2      3.7      空

新命中 tval = 0.8:
  -> 3.7 移到 [3000]（已在位）
  -> 1.2 移到 [2000]
  -> 0.8 插入到 [1000]
结果:          0.5    0.8      1.2      3.7
```

### 5.2 为何始终调用 IgnoreHit()

调用 `IgnoreHit()` 有两个效果：
1. **拒绝当前交点**：DXR 不会将其视为有效相交
2. **保持光线区间不变**：光线继续以原始 `[tmin, tmax]` 范围遍历

这确保所有可能的交点都能被检测到，代价是遍历了可能不必要的 BVH 节点。Node-C 方法（见 [NodeCAnyHit_hlsl.md](NodeCAnyHit_hlsl.md)）通过选择性接受交点来优化此行为。

### 5.3 命中数据的三个缓冲区

| 缓冲区 | 存储内容 | 用途 |
|--------|---------|------|
| `gHitT` | 光线参数 $t$ | 排序依据，用于深度排序 |
| `gHitDiffuse` | 漫反射颜色 `float4` | 用于后续合成着色 |
| `gHitNdotV` | $\|\mathbf{N} \cdot \mathbf{D}\|$ | 法线与光线方向的夹角余弦值，用于光照计算 |

### 5.4 溢出处理

当 `rayPayload.nhits >= gNquery` 时，`min(rayPayload.nhits, gNquery)` 返回 `gNquery`，新命中从第 `gNquery` 个位置开始插入。如果新命中比所有已存储的命中都远，它将被存储在第 `gNquery` 个位置（超出有效范围 $[0, N-1]$），实际上被丢弃。如果新命中较近，它会将一个旧命中推到第 `gNquery` 个位置，从而有效地替换最远的命中。

---

## 6. 输入与输出

### 输入

| 参数 | 类型 | 说明 |
|------|------|------|
| `rayPayload` | `inout mhRayPayload` | 光线负载，包含 `nhits`（已检测到的命中总数） |
| `attribs` | `BuiltinIntersectionAttribs` | DXR 内建相交属性（重心坐标等） |
| DXR 系统值 | - | `DispatchRaysIndex()`、`RayTCurrent()`、`PrimitiveIndex()` 等 |

### 输出（通过全局缓冲区）

| 缓冲区 | 更新方式 | 说明 |
|--------|---------|------|
| `gHitT` | 插入排序写入 | 排序后的 $t$ 值数组 |
| `gHitDiffuse` | 插入排序写入 | 排序后的漫反射颜色数组 |
| `gHitNdotV` | 插入排序写入 | 排序后的法线-视线余弦值数组 |
| `rayPayload.nhits` | 递增 | 累计命中总数 |

---

## 7. 与其他文件的关系

| 相关文件 | 关系 |
|---------|------|
| `NaiveIntersect.hlsl` | 配套的自定义相交着色器，引用本文件的插入排序逻辑 |
| `NodeCAnyHit.hlsl` | Node-C 优化版本，在本文件基础上添加选择性接受逻辑 |
| `NodeCIntersect.hlsl` | Node-C 配套的自定义相交着色器 |

---

## 8. 在光线追踪管线中的位置

```
应用程序调用 DispatchRays()
       |
       v
  光线生成着色器 (Ray Generation Shader)
  调用 TraceRay()
       |
       v
  BVH 遍历（DXR 运行时管理）
       |
       +----> 包围盒测试通过
       |           |
       |           v
       |      ┌──────────────────────────┐
       |      │ NaiveIntersect.hlsl       │  自定义相交着色器
       |      │ 执行三角形相交测试          │
       |      │ 调用 ReportHit()          │
       |      └──────────────────────────┘
       |           |
       |           v
       |      ┌──────────────────────────┐
       |      │ NaiveAnyHit.hlsl          │  <-- 本文件
       |      │ 插入排序维护前 N 个命中     │
       |      │ 调用 IgnoreHit()          │
       |      └──────────────────────────┘
       |           |
       |           v
       |      继续遍历下一个候选交点 ...
       |
       v
  遍历完成 -> Miss Shader 或 Closest-Hit Shader
       |
       v
  从 gHitT/gHitDiffuse/gHitNdotV 读取前 N 个命中
  执行最终着色合成
```

---

## 9. 技术要点与注意事项

1. **全局缓冲区的竞争**：由于每条光线只写入自己像素对应的缓冲区位置，不同光线之间不存在写竞争。

2. **IgnoreHit 的语义**：`IgnoreHit()` 是 DXR 内建函数，只能在任意命中着色器中调用。调用后着色器立即返回，DXR 运行时继续遍历。

3. **nhits 的含义**：`rayPayload.nhits` 记录的是总命中数（可能超过 `gNquery`），不是有效存储的命中数。有效命中数为 `min(nhits, gNquery)`。

4. **法线-视线计算**：`abs(dot(normalize(Ng), normalize(WorldRayDirection())))` 取绝对值是因为背面法线可能与光线方向同向，取绝对值确保结果始终为正。

5. **性能瓶颈**：朴素方法的主要性能问题不在于排序本身（$O(N)$ 通常较快），而在于光线遍历了所有 BVH 节点，包括那些距离远于第 $N$ 个命中的节点。Node-C 方法正是为了解决此问题。

---

## 10. 扩展阅读

- **书中章节**：Chapter 9, "Multi-Hit Ray Tracing in DXR"
- **DXR 规范**：Microsoft DirectX Raytracing (DXR) Functional Spec - Any-Hit Shader
- **Node-C 优化**：Gribble, C. P., "Multi-Hit Ray Tracing in DXR," *Ray Tracing Gems*, Chapter 9, 2019.
- **相关应用**：透明度排序 (Order-Independent Transparency)、X 射线模拟、体积渲染中的多层命中收集
