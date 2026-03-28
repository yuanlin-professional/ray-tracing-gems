# clipping.cpp 技术文档

## 文件概述

**文件路径**: `Ch_27_Interactive_Ray_Tracing_Techniques_for_High-Fidelity_Scientific_Visualization/clipping.cpp`

本文件实现了 Tachyon 光线追踪引擎中的**用户定义裁剪平面 (User-Defined Clipping Planes)** 功能。裁剪平面允许用户"切开"场景中的物体，暴露其内部结构——这在科学可视化中至关重要，例如查看蛋白质复合体的内部、显示分子截面等。文件提供了四种射线-物体相交处理函数，分别处理有无裁剪和有无阴影的组合场景。

## 算法与数学背景

### 裁剪平面方程

裁剪平面使用标准平面方程：

$$ax + by + cz > d$$

其中 $(a, b, c)$ 为平面法线方向，$d$ 为到原点的有符号距离。当交点满足此不等式时，该交点被"裁剪"（即丢弃）。

裁剪平面数据存储为 `float` 数组，每个平面占 4 个连续值：
- `planes[i*4 + 0]` = $a$
- `planes[i*4 + 1]` = $b$
- `planes[i*4 + 2]` = $c$
- `planes[i*4 + 3]` = $d$

### 射线交点位置计算

给定射线 $\mathbf{r}(t) = \mathbf{o} + t \mathbf{d}$，交点在参数 $t$ 处的世界空间位置为：

$$\mathbf{hit} = \mathbf{o} + t \cdot \mathbf{d}$$

在代码中由宏 `RAYPNT(hit, ray, t)` 实现。

## 代码结构概览

```
clipping.cpp
├── add_regular_intersection()          // 无裁剪，普通射线
├── add_clipped_intersection()          // 有裁剪，普通射线
├── add_shadow_intersection()           // 无裁剪，阴影射线
└── add_clipped_shadow_intersection()   // 有裁剪，阴影射线
```

四个函数形成一个 2x2 的组合矩阵：

|  | 无裁剪 | 有裁剪 |
|--|--------|--------|
| **普通射线** | `add_regular_intersection()` | `add_clipped_intersection()` |
| **阴影射线** | `add_shadow_intersection()` | `add_clipped_shadow_intersection()` |

## 逐段代码详解

### add_regular_intersection() - 无裁剪普通相交

```cpp
void add_regular_intersection(flt t, const object * obj, ray * ry) {
  if (t > EPSILON) {
    if (t < ry->maxdist) {
      ry->maxdist = t;
      ry->intstruct.num = 1;
      ry->intstruct.closest.obj = obj;
      ry->intstruct.closest.t = t;
    }
  }
}
```

最简单的相交处理：
1. **`t > EPSILON`**：排除过近的交点（防止自相交）
2. **`t < ry->maxdist`**：检查是否比当前最近交点更近
3. 更新 `maxdist` 和最近交点信息

### add_clipped_intersection() - 有裁剪普通相交

```cpp
void add_clipped_intersection(flt t, const object * obj, ray * ry) {
  if (t > EPSILON) {
    if (t < ry->maxdist) {

      /* handle clipped object tests */
      if (obj->clip != NULL) {
        vector hit;
        int i;

        RAYPNT(hit, (*ry), t);    /* find the hit point */
        for (i=0; i<obj->clip->numplanes; i++) {
          if ((obj->clip->planes[i * 4    ] * hit.x +
               obj->clip->planes[i * 4 + 1] * hit.y +
               obj->clip->planes[i * 4 + 2] * hit.z) >
               obj->clip->planes[i * 4 + 3]) {
            return; /* hit point was clipped */
          }
        }
      }

      ry->maxdist = t;
      ry->intstruct.num = 1;
      ry->intstruct.closest.obj = obj;
      ry->intstruct.closest.t = t;
    }
  }
}
```

在 `add_regular_intersection()` 的基础上增加裁剪平面测试：

1. **检查物体是否有裁剪数据**：`obj->clip != NULL`
2. **计算交点位置**：`RAYPNT(hit, ray, t)` 即 $\mathbf{hit} = \mathbf{origin} + t \cdot \mathbf{direction}$
3. **遍历所有裁剪平面**：
   - 计算 $ax + by + cz$ 并与 $d$ 比较
   - 若 $ax + by + cz > d$，交点在裁剪平面"外侧"，丢弃该交点
4. **通过所有裁剪测试后**：正常记录交点

### add_shadow_intersection() - 无裁剪阴影相交

```cpp
void add_shadow_intersection(flt t, const object * obj, ray * ry) {
  if (t > EPSILON) {
    if (t < ry->maxdist) {
      if (!(obj->tex->flags & RT_TEXTURE_SHADOWCAST)) {
        if (ry->scene->shadowfilter)
          ry->intstruct.shadowfilter *= (1.0 - obj->tex->opacity);
        return;
      }

      ry->maxdist = t;
      ry->intstruct.num = 1;
      ry->flags |= RT_RAY_FINISHED;
    }
  }
}
```

阴影射线的特殊逻辑：

1. **非投影物体处理**：如果物体不投射阴影（`RT_TEXTURE_SHADOWCAST` 未设置）：
   - 启用阴影滤波时，按透明度衰减阴影：$\text{filter} \times= (1 - \text{opacity})$
   - 直接返回，继续追踪
2. **投射阴影的物体**：设置 `RT_RAY_FINISHED` 标志，立即终止射线——阴影射线只需知道"是否被完全遮挡"

### add_clipped_shadow_intersection() - 有裁剪阴影相交

```cpp
void add_clipped_shadow_intersection(flt t, const object * obj, ray * ry) {
  if (t > EPSILON) {
    if (t < ry->maxdist) {
      if (!(obj->tex->flags & RT_TEXTURE_SHADOWCAST)) {
        if (ry->scene->shadowfilter)
          ry->intstruct.shadowfilter *= (1.0 - obj->tex->opacity);
        return;
      }

      /* handle clipped object tests */
      if (obj->clip != NULL) {
        vector hit;
        int i;

        RAYPNT(hit, (*ry), t);
        for (i=0; i<obj->clip->numplanes; i++) {
          if ((obj->clip->planes[i * 4    ] * hit.x +
               obj->clip->planes[i * 4 + 1] * hit.y +
               obj->clip->planes[i * 4 + 2] * hit.z) >
               obj->clip->planes[i * 4 + 3]) {
            return; /* hit point was clipped */
          }
        }
      }

      ry->maxdist = t;
      ry->intstruct.num = 1;
      ry->flags |= RT_RAY_FINISHED;
    }
  }
}
```

结合了裁剪和阴影的完整逻辑：
1. 处理非投影物体的透明度衰减
2. 对投影物体执行裁剪平面测试
3. 被裁剪的交点不产生阴影
4. 通过裁剪测试的交点终止射线

## 关键算法深入分析

### 裁剪平面的数据组织

```
obj->clip->planes[] 内存布局:
┌──────────────────────────────────────────────┐
│ plane 0: [a0, b0, c0, d0]                   │
│ plane 1: [a1, b1, c1, d1]                   │
│ ...                                          │
│ plane N: [aN, bN, cN, dN]                   │
└──────────────────────────────────────────────┘
```

使用连续 `float` 数组（而非结构体数组）可以更好地利用缓存行和 SIMD 指令。

### 逻辑关系：多平面裁剪

多个裁剪平面之间使用**逻辑或 (OR)** 关系——交点只要位于任何一个裁剪平面的"外侧"就被丢弃。这等价于保留所有裁剪平面"内侧"的**交集 (Intersection)** 区域。

### 阴影滤波的累积模型

```
初始 shadowfilter = 1.0
    ├── 遇到透明物体 1 (opacity=0.3): filter *= (1-0.3) = 0.7
    ├── 遇到透明物体 2 (opacity=0.5): filter *= (1-0.5) = 0.35
    └── 最终阴影强度 = 0.35 (35% 的光通过)
```

这实现了透明物体对阴影的累积衰减效果。

## 输入与输出

### 通用输入
| 名称 | 类型 | 说明 |
|------|------|------|
| `t` | `flt` | 射线参数（到交点的距离） |
| `obj` | `const object *` | 被交物体指针 |
| `ry` | `ray *` | 射线数据（包含当前状态） |

### 射线状态（输入/输出）
| 字段 | 类型 | 说明 |
|------|------|------|
| `ry->maxdist` | `flt` | 当前最远有效距离 |
| `ry->intstruct.closest` | `intersection` | 最近交点信息 |
| `ry->intstruct.shadowfilter` | `float` | 阴影衰减累积值 |
| `ry->flags` | `int` | 射线状态标志 |

## 与其他文件的关系

- **`aomaxdist.cu`**：AO 射线也可能需要裁剪处理
- **`transmax.cu`**：透明射线的裁剪需要特殊处理
- **`edgeoutline.cu`**：裁剪截面的边缘可能需要描边

## 在光线追踪管线中的位置

```
射线发射 → 几何求交 → [相交处理函数] → 选择最近交点 → 着色
                       ^^^^^^^^^^^^^^^^
                       本文件实现此阶段

函数选择逻辑:
  if (shadow_ray)
    if (has_clipping) → add_clipped_shadow_intersection()
    else             → add_shadow_intersection()
  else
    if (has_clipping) → add_clipped_intersection()
    else             → add_regular_intersection()
```

## 技术要点与注意事项

1. **函数指针选择**：Tachyon 在场景初始化时根据是否启用裁剪预先选择对应的函数指针，避免运行时分支
2. **EPSILON 的选择**：`EPSILON` 值需要足够小以不丢失有效交点，又足够大以避免自相交
3. **裁剪与 CSG 的区别**：注释明确说明"no CSG"——这些函数不处理构造实体几何 (Constructive Solid Geometry)，仅做简单裁剪
4. **阴影射线提前终止**：`RT_RAY_FINISHED` 标志允许 Tachyon 跳过后续几何体测试，这是重要的性能优化
5. **透明度阈值**：代码中没有透明度阈值检查，极低透明度的物体仍会被处理

## 扩展阅读

- Tachyon Ray Tracing Library: https://jedi.ks.uiuc.edu/~johns/raytracer/
- Stone, J.E. *An Efficient Library for Parallel Ray Tracing and Animation*. Master's Thesis, 1998
- Roth, S.D. *Ray Casting for Modeling Solids*. Computer Graphics and Image Processing, 1982
