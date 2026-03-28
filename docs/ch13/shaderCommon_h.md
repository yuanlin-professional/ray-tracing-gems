# shaderCommon.h 技术文档

## 文件概述

`shaderCommon.h` 是 CPU 与 GPU 之间共享的常量和数据结构定义文件。它定义了光源类型枚举、光源信息结构体，以及贯穿整个管线的关键常量。该文件同时被 C++ 代码和 HLSL 着色器包含。

**源文件路径**: `Ch_13_Ray_Traced_Shadows_Maintaining_Real-Time_Frame_Rates/dxrShadows/shaderCommon.h`

## 完整源码与注解

```cpp
// 单个 RT 通道处理的最大光源数量
#define MAX_LIGHTS_PER_PASS 8

// 每通道打包纹理数量 = MAX_LIGHTS_PER_PASS / 4
// 每 4 个光源的可见性值打包到一个 RGBA 纹理的 4 个通道中
#define MAX_LIGHTS_PER_PASS_QUARTER 2

// 分离式模糊方向常量
#define BLUR_HORIZONTAL 1
#define BLUR_VERTICAL 2

// 变化度滤波算子类型
#define MAXIMUM 1    // 最大值分布（扩散高变化度区域）
#define AVERAGE 2    // 均值模糊（平滑变化度场）

// 无效光源索引，表示"显示全部光源"
#define INVALID_LIGHT_INDEX 65535

// 无效 UV 坐标，表示重投影失败
#define INVALID_UVS float2(-10.0f, -10.0f)

// 光源类型枚举
#define POINT_LIGHT 1              // 点光源（硬阴影）
#define SPHERICAL_LIGHT 2          // 球形面光源（软阴影）
#define DIRECTIONAL_HARD_LIGHT 3   // 硬方向光（硬阴影）
#define DIRECTIONAL_SOFT_LIGHT 4   // 软方向光（有角度范围的软阴影）

// 光源信息结构体，CPU 端填充后传递给 GPU
struct ShadowsLightInfo {
    float3 position;    // 光源世界空间位置（点光/球光）
    float3 direction;   // 光源方向（方向光）
    float size;         // 光源大小：球光半径 / 方向光立体角
    int type;           // 光源类型（上述枚举之一）
};
```

## 算法与数学背景

### 光源类型与阴影计算

| 类型 | 阴影方法 | 采样方式 |
|------|----------|----------|
| `POINT_LIGHT` | 单条阴影光线 | 无采样 |
| `SPHERICAL_LIGHT` | 圆盘采样 | 泊松圆盘 + TBN 变换 |
| `DIRECTIONAL_HARD_LIGHT` | 单条平行光线 | 无采样 |
| `DIRECTIONAL_SOFT_LIGHT` | 虚拟球光采样 | 构造虚拟球光 + 泊松圆盘 |

对于软方向光，通过 `size` 参数（立体角 $\omega$）构造虚拟球光：

$$\mathbf{p}_{\text{light}} = \mathbf{p}_{\text{pixel}} - \hat{\mathbf{d}} \cdot \frac{1}{\tan(\omega)}$$

其中 $\hat{\mathbf{d}}$ 为归一化光源方向。

### 可见性打包

4 个光源的可见性值 $[v_0, v_1, v_2, v_3]$ 分别存储在 RGBA 纹理的 R/G/B/A 通道中。8 个光源需要 2 个纹理。

## 输入与输出

- **输入**: 无（纯定义文件）
- **输出**: 被所有 C++ 和 HLSL 文件使用的共享常量

## 与其他文件的关系

- 被 `dxrShadows.h`（C++端）包含
- 被 `CommonUtils.hlsl`（GPU端）包含
- 被所有需要光源类型或常量的着色器间接使用

## 在光线追踪管线中的位置

此文件是 CPU-GPU 数据桥梁的基础，定义了管线各阶段共享的"语言"。

## 技术要点与注意事项

1. 使用 `#define` 而非 C++ enum，确保 HLSL 兼容性
2. `float3` 在 C++ 端为 `glm::vec3`，在 HLSL 端为原生 `float3`
3. `INVALID_UVS` 使用 $(-10, -10)$ 远超 $[0,1]$ 范围，可靠标记无效重投影
4. `ShadowsLightInfo` 结构体内存布局需在 CPU/GPU 端保持一致

## 扩展阅读

- HLSL/C++ 共享头文件设计模式
- DXR Constant Buffer 对齐规则
