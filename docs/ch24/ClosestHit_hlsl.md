# ClosestHit.hlsl 技术文档

## 文件概述

`ClosestHit.hlsl` 实现了实时光子映射系统的**最近命中着色器** (Closest Hit Shader)。当光子光线与场景几何体相交时，该着色器负责处理光子与表面的交互：加载表面属性、执行俄罗斯轮盘赌 (Russian Roulette) 决定是否继续弹射、计算出射方向和功率、以及将有效光子存储到缓冲区。

**源文件路径**: `Ch_24_Real-Time_Global_Illumination_with_Photon_Mapping/ClosestHit.hlsl`

## 算法与数学背景

### 光子-表面交互

当光子命中表面时，需要进行以下决策：
1. **存储**：将光子的功率存储在命中点（用于后续密度估计）
2. **弹射**：根据表面 BRDF 采样出射方向，传播衰减后的功率
3. **吸收**：通过俄罗斯轮盘赌概率性终止路径

### 俄罗斯轮盘赌 (Russian Roulette)

为了在有限弹射次数下保持无偏性 (unbiased)，使用俄罗斯轮盘赌：

$$P_{\text{continue}} = \min(1, \max(\rho_r, \rho_g, \rho_b))$$

其中 $\rho$ 为表面反照率 (albedo)。继续弹射时，功率需要除以继续概率以补偿：

$$\text{power}_{\text{out}} = \frac{\text{power}_{\text{in}} \cdot f_r}{P_{\text{continue}}}$$

这确保了期望值 $E[\text{power}_{\text{out}}] = \text{power}_{\text{in}} \cdot f_r$，保持无偏。

### 功率传递

光子功率在每次弹射后按 BRDF 衰减：

$$\Phi_{\text{out}} = \Phi_{\text{in}} \cdot f_r(\omega_i, \omega_o) \cdot \cos\theta_o$$

其中 $\Phi$ 为光通量 (radiant flux / power)。

## 代码结构概览

```
[DXR Closest Hit 入口]
  → 加载表面属性
  → 计算命中位置
  → 解码入射功率
  → 初始化 RNG
  → 俄罗斯轮盘赌（决定弹射/吸收）
  → 打包状态到 Payload
  → 验证并存储光子
```

## 完整源码与逐行注释

```hlsl
// DXR 最近命中着色器属性声明
[shader("closesthit")]
void closestHitShader(inout Payload p : SV_RayPayload,
    in IntersectionAttributes attribs : SV_IntersectionAttributes)
{
    // 从命中的三角形属性加载表面数据
    // 包括：法线、反照率、材质参数等
    // attribs 包含重心坐标 (barycentric coordinates) 用于插值
    surface_attributes surface = LoadSurface(attribs);

    // 获取世界空间的光线方向（入射方向）
    float3 ray_direction = WorldRayDirection();
    // 计算命中点的世界空间位置：原点 + 方向 * 距离
    float3 hit_pos = WorldRayOrigin() + ray_direction * t;
    // 从 RGBE 5999 格式解码入射光子功率
    float3 incoming_power = from_rbge5999(p.power);
    // 初始化出射功率为 0
    float3 outgoing_power = .0f;

    // 初始化随机数生成器状态
    RandomStruct r;
    r.seed = p.random.x;  // 种子
    r.key = p.random.y;   // 密钥（用于计数器模式 RNG）

    // 俄罗斯轮盘赌检查
    float3 outgoing_direction = .0f;
    float3 store_power = .0f;
    // russian_roulette 函数执行以下操作：
    //   1. 根据表面反照率决定是否继续弹射
    //   2. 如果继续：计算出射方向和衰减后的功率 (outgoing_power)
    //   3. 计算需要存储的功率 (store_power)
    //   4. 返回 bool: true = 继续弹射, false = 路径终止
    bool keep_going = russian_roulette(incoming_power, ray_direction,
        surface, r, outgoing_power, out_going_direction, store_power);

    // 将更新后的状态打包回 Payload
    // 包括：更新的 RNG key、出射功率（RGBE 编码）、
    // 出射方向（half 精度）、keep_going（影响 t 值）
    repack_the_state_to_payload(r.key, outgoing_power,
    outgoing_direction, keep_going);

    // 验证光子有效性并添加到光子缓冲区
    // 检查命中点是否在相机视锥内、法线是否朝向相机
    validate_and_add_photon(surface, hit_pos, store_power,
            ray_direction, t);
}
```

## 关键算法深入分析

### 存储功率 vs 出射功率

代码中区分了两种功率：
- **`store_power`**：存储到光子缓冲区的功率，用于密度估计时的辐照度计算
- **`outgoing_power`**：传递给下一次弹射的功率，已经过 BRDF 衰减和俄罗斯轮盘赌补偿

这种分离使得存储的光子功率代表的是**入射功率**（到达该点的功率），而非弹射后的衰减功率。

### RNG 状态管理

使用 `seed` + `key` 的二元组维护随机数生成器状态，这是典型的计数器模式 (counter-based) RNG 设计（如 Philox 或 Squares 算法）：
- `seed`：基于线程 ID 或像素位置的固定种子
- `key`：每次使用后递增的计数器

这种设计确保了不同弹射间的随机数独立且不重复。

### 命中距离 t 的双重用途

变量 `t` 在代码中有两个用途：
1. 计算命中位置：`hit_pos = origin + direction * t`
2. 传递给 `validate_and_add_photon` 用于光线长度相关的核缩放

## 输入与输出

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `p` | `Payload` | inout | 光子有效载荷（包含功率、方向、RNG、弹射数等） |
| `attribs` | `IntersectionAttributes` | in | 交叉点属性（重心坐标） |
| 光子缓冲区 | UAV | out | 通过 `validate_and_add_photon` 写入 |

## 与其他文件的关系

- **RayGeneration.hlsl**: 调用 `TraceRay` 后触发本着色器；本着色器更新 Payload 后返回循环
- **ValidateAndAddPhoton.hlsl**: 本文件直接调用 `validate_and_add_photon()` 存储光子
- **SplattingShaders.hlsl**: 读取本着色器存储的光子数据进行渲染

## 在光线追踪管线中的位置

本文件是 DXR 管线中的 **Closest Hit Shader**，在光线与场景最近交点确定后执行：

```
[Ray Generation] ── TraceRay ──> [加速结构遍历]
                                      │
                                      ▼
                              [Closest Hit Shader] ← 本文件
                                      │
                              ┌───────┼───────┐
                              │               │
                        更新 Payload     存储光子
                        (返回 RayGen)    (写入缓冲区)
```

## 技术要点与注意事项

1. **变量名不一致**：代码中 `out_going_direction`（带下划线）与 `outgoing_direction`（不带下划线）可能是原文笔误
2. **t 的来源**：代码中使用了 `t` 但未调用 `RayTCurrent()`，可能是通过 Payload 的 `p.t` 或全局变量访问
3. **RGBE 编码解码**：`from_rbge5999` 和 `to_rgbe5999` 的精度适用于光子功率的 HDR 范围，但在极端值时可能丢失精度
4. **表面加载**：`LoadSurface(attribs)` 内部需要根据重心坐标插值顶点属性（法线、UV 等），并加载纹理
5. **所有弹射都存储光子**：与仅在最终弹射存储不同，本方案在每次弹射都存储光子（如果通过验证），这提供了更密集的光子分布

## 扩展阅读

- Jensen, H.W. *Realistic Image Synthesis Using Photon Mapping*, Chapter 3: Photon Tracing
- Arvo, J. & Kirk, D. *Particle Transport and Image Synthesis*, SIGGRAPH 1990 (俄罗斯轮盘赌的理论基础)
- Ray Tracing Gems, Chapter 24, Section 24.3: Photon Tracing with DXR
