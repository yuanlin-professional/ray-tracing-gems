# RayGeneration.hlsl 技术文档

## 文件概述

`RayGeneration.hlsl` 实现了实时光子映射 (Real-Time Photon Mapping) 系统的**光线生成着色器** (Ray Generation Shader)。该文件包含两部分：
1. **Payload 结构体**：定义了光子在追踪过程中携带的所有状态信息
2. **rayGen 函数**：实现了光子追踪的主循环，从反射阴影贴图 (RSM) 读取初始状态，然后通过 DXR `TraceRay` 进行多次弹射

**源文件路径**: `Ch_24_Real-Time_Global_Illumination_with_Photon_Mapping/RayGeneration.hlsl`

## 算法与数学背景

### 光子映射 (Photon Mapping)

经典光子映射分为两个阶段：
1. **光子追踪 (Photon Tracing)**：从光源发射光子，在场景中弹射并存储
2. **密度估计 (Density Estimation)**：在渲染时通过收集附近光子估计辐照度

本文件实现的是第一阶段。光子从 RSM 获取初始状态，然后通过 DXR 在场景中弹射。

### 反射阴影贴图 (Reflective Shadow Map, RSM)

RSM 是从光源视角渲染的 G-Buffer，每个像素存储：
- 世界空间位置 (position)
- 法线 (normal)
- 光通量 (flux/power)

RSM 的每个像素相当于一个已经完成第一次弹射的光子，避免了从光源直接追踪第一段路径的开销。

### 光子弹射控制

光子路径的终止条件：
1. 达到最大弹射次数 `MAX_BOUNCE_COUNT`
2. 光线长度 `t` 为 0（表示光线未命中或被俄罗斯轮盘赌终止）

## 代码结构概览

```
Payload 定义 → rayGen 入口
  → 读取 RSM 初始状态
  → while 循环（弹射次数 & 光线长度检查）
    → 计算光线原点和方向
    → TraceRay (触发 ClosestHit/Miss)
    → 递增弹射计数
```

## 完整源码与逐行注释

```hlsl
// 光子有效载荷 (Payload) 结构体
// 在 TraceRay 调用中在 RayGeneration 和 ClosestHit 之间传递
struct Payload
{
    // 下一条光线的方向（half4 节省带宽，第 4 个分量为填充）
    half4 direction;
    // 随机数生成器 (RNG) 状态：seed 和 key
    // 使用两个 uint 维护可重复的伪随机序列
    uint2 random;
    // 打包的光子功率 (power)，使用 RGBE 5999 格式压缩为单个 uint
    // 节省有效载荷空间（DXR 有效载荷越小性能越好）
    uint power;
    // 光线长度 t：命中距离。t = 0 表示路径终止（miss 或轮盘赌终止）
    float t;
    // 当前弹射次数
    uint bounce;
};

// DXR 光线生成着色器入口点
[shader("raygeneration")]
void rayGen()
{
    Payload p;
    RayDesc ray;

    // 从反射阴影贴图 (RSM) 读取光子的初始状态
    // 包括：初始位置（通过 direction 和 t 编码）、功率、随机种子等
    ReadRSMSamplePosition(p);

    // 主循环：持续追踪光子直到路径终止
    // 终止条件：
    //   1. bounce >= MAX_BOUNCE_COUNT（达到最大弹射次数）
    //   2. t == 0（光线 miss 或被俄罗斯轮盘赌终止）
    while (p.bounce < MAX_BOUNCE_COUNT && p.t != 0)
    {
        // 从有效载荷中的状态计算世界空间的光线原点
        // 使用上一次命中的 t 值和方向重建命中位置
        ray.Origin = get_hit_position_in_world(p, ray);
        // 光线方向从有效载荷获取（由 ClosestHit 设置为弹射方向）
        ray.Direction = p.direction.xyz;

        // 执行 DXR 光线追踪
        // gRtScene: 顶层加速结构 (Top-Level Acceleration Structure, TLAS)
        // RAY_FLAG_FORCE_OPAQUE: 跳过 any-hit 着色器，所有几何体视为不透明
        // 0xFF: 实例掩码 (instance mask)，与所有实例匹配
        // 0, 1, 0: hit group 索引偏移、步长、miss shader 索引
        TraceRay(gRtScene, RAY_FLAG_FORCE_OPAQUE, 0xFF, 0, 1, 0, ray, p);
        // 递增弹射计数
        p.bounce++;
    }
}
```

## 关键算法深入分析

### Payload 的压缩设计

DXR 中有效载荷 (Payload) 的大小直接影响性能，因此需要精心压缩：

| 字段 | 大小 | 压缩策略 |
|------|------|----------|
| `direction` | 8 字节 | 使用 `half4`（16 位半精度浮点） |
| `random` | 8 字节 | 两个 `uint` 维护 RNG 状态 |
| `power` | 4 字节 | RGBE 5999 格式（RGB 各 9 位尾数 + 5 位共享指数） |
| `t` | 4 字节 | 标准 `float` |
| `bounce` | 4 字节 | `uint` |
| **总计** | **28 字节** | |

### 循环结构的设计

传统递归路径追踪使用递归的 `TraceRay` 调用（在 Closest Hit 中再次调用 `TraceRay`）。本实现使用**迭代式循环** (iterative loop)，将递归展开为 Ray Generation 中的 `while` 循环：

```
传统递归:  RayGen → TraceRay → ClosestHit → TraceRay → ClosestHit → ...
本实现:    RayGen → while { TraceRay → ClosestHit(更新 Payload) → 回到 while }
```

迭代式设计的优势：
1. 避免 DXR 递归深度限制
2. 减少栈空间使用
3. 更易于控制循环终止条件

### RAY_FLAG_FORCE_OPAQUE 的含义

设置 `RAY_FLAG_FORCE_OPAQUE` 标志后，所有几何体被视为不透明 (opaque)，跳过 Any Hit Shader 的执行。这简化了光子追踪（不处理半透明物体），并提升了追踪性能。

## 输入与输出

| 参数/资源 | 类型 | 说明 |
|-----------|------|------|
| `gRtScene` | `RaytracingAccelerationStructure` | 顶层加速结构 (TLAS) |
| RSM 纹理 | 纹理资源 | 通过 `ReadRSMSamplePosition` 读取 |
| `MAX_BOUNCE_COUNT` | 编译时常量 | 最大弹射次数 |
| **输出** | 通过 ClosestHit 中的 `validate_and_add_photon` | 光子缓冲区 |

## 与其他文件的关系

- **ClosestHit.hlsl**: `TraceRay` 命中几何体时调用，处理光子与表面的交互，更新 Payload
- **ValidateAndAddPhoton.hlsl**: 在 ClosestHit 中调用，将有效光子写入缓冲区
- **SplattingShaders.hlsl**: 使用存储的光子进行喷溅渲染
- **UniformScaling.hlsl / KernelModificationForVertexPosition.hlsl**: 光子喷溅时计算核大小和形状

## 在光线追踪管线中的位置

本文件是 DXR 管线的**入口点**，即 Ray Generation Shader：

```
[Ray Generation: rayGen()]
    │
    ├── ReadRSMSamplePosition() ──读取 RSM──> 初始化 Payload
    │
    └── while 循环
         │
         └── TraceRay()
              ├── [Miss Shader] ──> p.t = 0，终止循环
              └── [Closest Hit: closestHitShader()]
                   ├── 俄罗斯轮盘赌
                   ├── 更新 Payload（方向、功率、RNG）
                   └── validate_and_add_photon()
```

## 技术要点与注意事项

1. **半精度方向**：`half4 direction` 使用 FP16，方向精度约为 3 位小数，对于光子弹射通常足够
2. **RNG 状态传递**：随机数状态通过 Payload 在弹射间传递，确保不同弹射的随机数独立
3. **RGBE 编码**：`from_rbge5999()` / `to_rgbe5999()` 使用共享指数的 HDR 颜色压缩，节省 Payload 空间
4. **光线长度复用**：`t` 字段既用于重建命中位置，又作为终止标志（0 = 终止）
5. **无 Miss Shader 显示**：代码中未展示 Miss Shader，但它应将 `p.t` 设为 0 以终止循环

## 扩展阅读

- Jensen, H.W. *Realistic Image Synthesis Using Photon Mapping*, A K Peters, 2001
- Ray Tracing Gems, Chapter 24: Real-Time Global Illumination with Photon Mapping
- Microsoft DXR Specification: Ray Generation Shader, TraceRay intrinsic
