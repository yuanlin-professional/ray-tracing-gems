# transmax.cu 技术文档

## 文件概述

**文件路径**: `Ch_27_Interactive_Ray_Tracing_Techniques_for_High-Fidelity_Scientific_Visualization/transmax.cu`

本文件实现了**透明表面最大穿越次数限制 (Maximum Transparent Surface Traversal Count)** 功能。当光线穿过的透明表面数量超过用户定义的最大值时，着色器不再对该表面进行着色，而是直接发射一条继续方向相同的透射射线 (Transmission Ray)，如同表面不存在一样。这种技术对于科学可视化中密集排列的半透明物体（如分子表面）至关重要，可以有效限制递归深度并控制渲染性能。

## 算法与数学背景

### 透射射线递归问题

在包含大量透明表面的场景中，每条射线可能穿过数十甚至数百个透明表面。如果每个表面都进行完整着色（光照计算、阴影射线、AO 等），计算成本将呈指数级增长：

$$\text{Cost} = O(N^d)$$

其中 $N$ 为每个表面的着色操作数，$d$ 为递归深度。

### 穿越次数限制策略

引入计数器 `transcnt`，初始值为某个最大值 $T_{max}$：

$$\text{transcnt}_{i+1} = \text{transcnt}_i - 1$$

当 $\text{transcnt} < 1$ 时：
- 跳过所有着色计算
- 不增加递归深度
- 直接发射方向相同的透射射线
- 相当于表面"不存在"

这将复杂度从指数降为线性：$O(N \cdot T_{max})$。

## 代码结构概览

```
transmax.cu
├── struct PerRayData_radiance    // 辐射射线负载（含 transcnt）
└── closest_hit_shader()          // 最近命中着色器
```

## 逐段代码详解

### 射线负载结构

```cuda
struct PerRayData_radiance {
  float3 result;     // Final shaded surface color
  int transcnt;      // Transmission ray surface count/depth
  int depth;         // Current ray recursion depth
  // ...
}
```

关键字段：
- `transcnt`：透明表面穿越计数器，每穿过一个透明表面递减
- `depth`：递归深度计数器，用于传统的递归深度限制
- 这两个计数器独立工作，提供双重保护

### 最近命中着色器

```cuda
rtDeclareVariable(PerRayData_radiance, prd, rtPayload, );

RT_PROGRAM void closest_hit_shader( ... ) {
  // Skipping boilerplate closest-hit shader material here ...
```

### 穿越限制核心逻辑

```cuda
  // Do not shade transparent surface if the maximum transcnt has been reached.
  if ((opacity < 1.0) && (transcnt < 1)) {
```
两个条件同时满足时触发跳过逻辑：
1. `opacity < 1.0`：表面是透明的
2. `transcnt < 1`：已达到最大穿越次数

```cuda
    // Spawn transmission ray; shading behaves as if there was no intersection.
    PerRayData_radiance new_prd;
    new_prd.depth = prd.depth; // Do not increment recursion depth.
    new_prd.transcnt = prd.transcnt - 1;
```

关键设计决策：
- **`new_prd.depth = prd.depth`**：不增加递归深度——因为我们跳过了着色，这不应该算作一次"真正的"递归
- **`new_prd.transcnt = prd.transcnt - 1`**：继续递减穿越计数器

```cuda
    // Set/update various other properties of the new ray.

    // Shoot the new transmission ray and return its color
    Ray trans_ray = make_Ray(hit_point, ray.direction,
        radiance_ray_type, scene_epsilon, RT_DEFAULT_MAX);
    rtTrace(root_object, trans_ray, new_prd);
  }
```

发射透射射线：
- `hit_point`：从当前交点出发
- `ray.direction`：方向不变（直线穿过）
- `radiance_ray_type`：使用辐射射线类型（不是阴影射线）
- `scene_epsilon`：偏移防止自相交
- `RT_DEFAULT_MAX`：无最大距离限制

```cuda
  // Otherwise, continue shading this transparent surface hit point normally ...

  // Continue with typical closest-hit shader contents ...
  prd.result = result;
}
```

如果未达到穿越限制，正常进行着色。

## 关键算法深入分析

### depth vs transcnt 的区别

```
光线追踪树结构示意:

            Camera Ray (depth=0, transcnt=T_max)
                │
                ├── 透明表面 1 (depth=1, transcnt=T_max-1)
                │   ├── 反射射线 (depth=2) ← depth 增加
                │   └── 透射射线 (depth=1, transcnt=T_max-2)
                │       ├── 透明表面 2
                │       │   └── 透射射线 (depth=1, transcnt=T_max-3)
                │       ...
                │       └── 透明表面 N (transcnt < 1)
                │           └── 直接穿过 (depth=1, transcnt 继续递减)
                │               不着色，不增加 depth
```

- `depth` 限制**着色递归**的最大深度（反射、折射等需要完整着色的射线）
- `transcnt` 限制**透射穿越**的最大次数（仅计透明表面穿过次数）

### 场景示例

考虑一个密集的分子可视化场景，视线穿过 50 个半透明原子表面：

| 策略 | 着色次数 | 递归深度 | 性能 |
|------|---------|---------|------|
| 无限制 | 50 | 50 | 极慢（可能栈溢出） |
| depth 限制 = 5 | 5 | 5 | 快但丢失信息 |
| transcnt = 5 | 前 5 个着色，其余穿过 | 5 | 快且保持一致性 |

使用 `transcnt` 的优势：前几层透明表面正常着色（提供视觉信息），后面的表面透射穿过（避免过度递归），最终射线到达不透明物体或背景时正常着色。

### 能量守恒考虑

跳过着色的透射操作等效于将表面不透明度设为 0，因此：
- 不会引入额外能量
- 不会丢失背景信息
- 但可能丢失跳过表面的着色贡献

## 输入与输出

### 输入
| 名称 | 类型 | 说明 |
|------|------|------|
| `opacity` | `float` | 当前表面的不透明度 |
| `prd.transcnt` | `int` | 当前透射穿越计数器 |
| `prd.depth` | `int` | 当前递归深度 |
| `hit_point` | `float3` | 表面交点位置 |
| `ray.direction` | `float3` | 入射射线方向 |

### 输出
| 名称 | 类型 | 说明 |
|------|------|------|
| `prd.result` | `float3` | 最终颜色（来自正常着色或透射穿过） |

## 与其他文件的关系

- **`transedge.cu`**：角度依赖透明度产生大量透射射线，需要本文件的穿越限制来保证性能
- **`clipping.cpp`**：裁剪平面可以减少透明表面的数量，间接降低穿越计数
- **`aomaxdist.cu`**：被跳过的透明表面不会参与 AO 遮蔽计算
- **`edgeoutline.cu`**：被跳过的表面不会渲染轮廓线

## 在光线追踪管线中的位置

```
射线 → 命中透明表面 → transcnt >= 1?
                          │
                    ┌─────┴─────┐
                    │ YES       │ NO
                    v           v
              正常着色      跳过着色
              (完整光照)    (直接穿过)
                    │           │
                    v           v
              递归射线      透射射线
              depth++      depth 不变
              transcnt--   transcnt--
```

## 技术要点与注意事项

1. **transcnt 初始值选择**：
   - 值过小（如 1-2）：远处透明物体可能完全看不到
   - 值过大（如 100+）：失去限制作用
   - 典型值：3-10，取决于场景复杂度
2. **depth 不增加的意义**：确保透射穿过不会"消耗"宝贵的递归深度预算，留给真正需要着色的反射/折射射线
3. **射线类型**：透射射线使用 `radiance_ray_type` 而非 `shadow_ray_type`，因为它最终需要返回完整的颜色信息
4. **自相交防护**：`scene_epsilon` 确保透射射线不会重新命中同一表面
5. **栈深度**：尽管 `depth` 不增加，但实际的 OptiX/CUDA 栈深度仍会增加。极端情况下仍可能栈溢出

## 扩展阅读

- Stone, J.E. et al. *Early Experiences Scaling VMD Molecular Visualization and Analysis Jobs on Blue Waters*. XSEDE 2013
- Whitted, T. *An Improved Illumination Model for Shaded Display*. CACM, 1980
- OptiX Programming Guide: Recursive Ray Tracing and Stack Management
