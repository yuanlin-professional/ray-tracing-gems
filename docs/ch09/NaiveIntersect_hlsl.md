# NaiveIntersect.hlsl -- 朴素多重命中自定义相交着色器

> 源文件：`Ch_09_Multi-Hit_Ray_Tracing_in_DXR/NaiveIntersect.hlsl`

---

## 1. 文件概述

`NaiveIntersect.hlsl` 实现了 DXR (DirectX Raytracing) 中朴素多重命中方法的自定义相交着色器 (Intersection Shader)。该着色器负责执行自定义的三角形相交测试 (Triangle Intersection Test)，在相交成功时执行插入排序并更新命中计数缓冲区 (Hit Count Buffer)。

本文件的核心插入排序逻辑与 `NaiveAnyHit.hlsl` 等价（代码中标记为 OMITTED），主要区别在于：作为自定义相交着色器，它直接在相交测试后处理命中数据，并通过 `gHitCount` 缓冲区记录命中次数。

---

## 2. 算法与数学背景

### 自定义相交着色器的作用

在 DXR 中，自定义相交着色器 (Intersection Shader) 用于非三角形几何体或需要特殊处理的三角形相交。它替代 DXR 的内置三角形相交测试，允许开发者在相交测试中加入额外逻辑。

本文件中使用自定义相交着色器的目的是：
1. 在相交测试时直接执行插入排序（而非通过 `ReportHit` 触发 Any-Hit Shader）
2. 维护独立的命中计数缓冲区 `gHitCount`

### 命中计数

`gHitCount` 缓冲区记录每个像素检测到的总命中数。与 `rayPayload.nhits` 不同，这个计数存储在全局缓冲区中而非光线负载中，便于后续阶段读取。

---

## 3. 代码结构概览

### 函数列表

| 函数 | 类型 | 参数 | 说明 |
|------|------|------|------|
| `mhIntersectNaive` | `[shader("intersection")]` | 无（使用 DXR 系统值） | 朴素自定义相交着色器入口 |

### 依赖的辅助函数

| 函数 | 说明 |
|------|------|
| `intersectTriangle(primIdx, hitAttrib)` | 执行三角形相交测试，返回命中数 |
| `getHitBufferIndex(hitIdx, pixelIdx, pixelDims)` | 计算像素交错缓冲区索引 |

### 依赖的全局资源

| 名称 | 类型 | 说明 |
|------|------|------|
| `gHitT` | `RWBuffer<float>` | 命中点 $t$ 值缓冲区 |
| `gHitDiffuse` | `RWBuffer<float4>` | 漫反射颜色缓冲区 |
| `gHitNdotV` | `RWBuffer<float>` | 法线-视线余弦值缓冲区 |
| `gHitCount` | `RWBuffer<uint>` | 命中计数缓冲区 |
| `gNquery` | `uint` | 查询的最大命中数 $N$ |

---

## 4. 逐段代码详解

### 完整源代码（带中文注释）

```hlsl
// DXR 自定义相交着色器：执行三角形相交并维护命中排序
[shader("intersection")]
void mhIntersectNaive()
{
  HitAttribs hitAttrib;  // 相交属性结构体（包含 tval 等）

  // 执行自定义三角形相交测试
  uint nhits = intersectTriangle(PrimitiveIndex(), hitAttrib);

  if (nhits > 0)  // 相交成功
  {
    // ---- 获取像素信息和缓冲区参数 ----
    uint2 pixelIdx  = DispatchRaysIndex();          // 当前像素索引
    uint2 pixelDims = DispatchRaysDimensions();     // 光线分派维度
    uint  hitStride = pixelDims.x*pixelDims.y;      // 缓冲区步长
    float tval      = hitAttrib.tval;               // 相交点的 t 值

    // ---- 插入排序 ----
    // 计算插入的起始位置
    uint hi = getHitBufferIndex(min(nhits, gNquery),
                                pixelIdx, pixelDims);
    // OMITTED: 与 NaiveAnyHit.hlsl 的第 13-35 行等价
    // 执行与 mhAnyHitNaive 相同的插入排序和命中数据存储逻辑

    // ---- 更新命中计数缓冲区 ----
    uint hcIdx = getHitBufferIndex(0, pixelIdx, pixelDims); // 第 0 层的索引即为像素线性索引
    ++gHitCount[hcIdx];  // 递增该像素的命中计数
  }
}
```

### 省略部分说明

代码注释中标记的 "OMITTED" 部分等价于 `NaiveAnyHit.hlsl` 中的第 13-35 行，包含：
1. 从后向前的插入排序循环（移动 `gHitT`、`gHitDiffuse`、`gHitNdotV`）
2. 获取当前命中点的漫反射颜色和几何法线
3. 在排序找到的位置存储命中数据

详细实现参见 [NaiveAnyHit_hlsl.md](NaiveAnyHit_hlsl.md) 第 4 节。

---

## 5. 关键算法深入分析

### 5.1 Intersection Shader vs Any-Hit Shader

本文件的方法与 `NaiveAnyHit.hlsl` 在功能上等价，但工作流程不同：

| 特性 | Any-Hit 方案 | Intersection 方案（本文件） |
|------|-------------|-------------------------|
| 调用时机 | 内置相交测试通过后 | 替代内置相交测试 |
| 相交测试 | DXR 内置 | 自定义 `intersectTriangle` |
| 命中计数 | `rayPayload.nhits` | `gHitCount` 全局缓冲区 |
| 适用场景 | 标准三角形几何 | 自定义几何或需要额外控制 |

### 5.2 gHitCount 的索引计算

```hlsl
uint hcIdx = getHitBufferIndex(0, pixelIdx, pixelDims);
```

`hitIdx = 0` 时返回的是像素的基础线性索引（`pixelIdx.y * pixelDims.x + pixelIdx.x`），因此 `gHitCount` 实际上是一个按像素索引的一维数组。

### 5.3 nhits 参数的双重含义

`intersectTriangle` 返回的 `nhits` 在此有两层含义：
1. 作为返回值：非零表示相交成功
2. 传入 `getHitBufferIndex` 时：表示当前已积累的命中数，用于确定插入排序的起始位置

---

## 6. 输入与输出

### 输入

| 来源 | 说明 |
|------|------|
| `PrimitiveIndex()` | DXR 系统值，当前测试的图元索引 |
| `DispatchRaysIndex()` | DXR 系统值，当前像素索引 |
| `DispatchRaysDimensions()` | DXR 系统值，分派维度 |

### 输出

| 缓冲区 | 更新方式 | 说明 |
|--------|---------|------|
| `gHitT` | 插入排序写入 | 排序后的命中 $t$ 值 |
| `gHitDiffuse` | 插入排序写入 | 排序后的漫反射颜色 |
| `gHitNdotV` | 插入排序写入 | 排序后的法线-视线余弦值 |
| `gHitCount` | 递增 | 像素级命中计数 |

---

## 7. 与其他文件的关系

| 相关文件 | 关系 |
|---------|------|
| `NaiveAnyHit.hlsl` | 包含本文件省略的插入排序完整实现 |
| `NodeCIntersect.hlsl` | Node-C 优化版本，在此基础上添加光线区间更新 |
| `NodeCAnyHit.hlsl` | Node-C 的任意命中着色器配套实现 |

---

## 8. 在光线追踪管线中的位置

```
BVH 遍历
   |
   +----> 包围盒测试通过
   |           |
   |           v
   |      ┌──────────────────────────┐
   |      │ mhIntersectNaive()        │  <-- 本文件
   |      │ 1. intersectTriangle()    │
   |      │ 2. 插入排序（OMITTED）     │
   |      │ 3. ++gHitCount            │
   |      └──────────────────────────┘
   |           |
   |           v
   |      继续遍历（不调用 ReportHit，
   |      插入排序已在此完成）
   |
   v
  遍历完成 -> 读取排序后的命中数据
```

注意：本文件的自定义相交着色器直接完成了命中数据的排序和存储，不需要通过 `ReportHit` 触发任意命中着色器。这与 `NaiveAnyHit.hlsl` 的工作模式不同，后者需要 DXR 内置相交测试先触发 `ReportHit`。

---

## 9. 技术要点与注意事项

1. **OMITTED 标记**：代码中的 OMITTED 部分并非可选，而是书中为避免重复而省略的完整插入排序逻辑。实际使用时必须补全。

2. **intersectTriangle 的实现**：该辅助函数未在本文件中定义。它应执行标准的光线-三角形相交测试（如 Moller-Trumbore 算法），返回命中数并填充 `hitAttrib`。

3. **不调用 ReportHit**：与典型的自定义相交着色器不同，本文件直接处理命中数据而非通过 `ReportHit` 报告给 DXR 运行时。这是因为多重命中的管理完全由用户代码控制。

4. **gHitCount vs nhits**：`gHitCount` 是持久化的全局缓冲区计数，而 `nhits` 是 `intersectTriangle` 的返回值。两者在功能上互补。

---

## 10. 扩展阅读

- **书中章节**：Chapter 9, "Multi-Hit Ray Tracing in DXR"
- **自定义相交着色器**：DXR Functional Spec - Intersection Shader Stage
- **配套实现**：参见 [NaiveAnyHit_hlsl.md](NaiveAnyHit_hlsl.md) 了解完整的插入排序实现
- **Moller-Trumbore 算法**：Moller, T., and Trumbore, B., "Fast, Minimum Storage Ray-Triangle Intersection," *Journal of Graphics Tools*, 1997.
