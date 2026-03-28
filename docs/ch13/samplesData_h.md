# samplesData.h 技术文档

## 文件概述

`samplesData.h` 是泊松圆盘采样数据模块的头文件，声明了获取预计算采样数据的函数接口。文件极为简洁，仅含一个函数声明。

**源文件路径**: `Ch_13_Ray_Traced_Shadows_Maintaining_Real-Time_Frame_Rates/dxrShadows/samplesData.h`

## 完整源码与注解

```hlsl
#pragma once

#include "Falcor.h"

// 返回指向预计算泊松圆盘采样数据的指针
// 数据包含 6048 个 float2 采样点，覆盖 1-32 SPP
// 用于创建 GPU 端 1D 纹理，供阴影光线采样使用
const float2* getPoissonDisk1to32spp();
```

## 算法与数学背景

函数返回的数据用于泊松圆盘采样 (Poisson Disk Sampling)，详见 `samplesData_cpp.md`。

## 输入与输出

- **输入**: 无
- **输出**: 指向 6048 个 `float2` 的常量指针

## 与其他文件的关系

- 被 `dxrShadows.cpp` 包含并调用
- 由 `samplesData.cpp` 实现

## 在光线追踪管线中的位置

提供阴影光线方向采样的数据接口层。

## 技术要点与注意事项

1. 使用 Falcor 的 `float2` 类型（等同于 `glm::vec2`）
2. 返回的是静态数组指针，生命周期覆盖整个程序运行期

## 扩展阅读

- 参见 `samplesData_cpp.md` 获取详细数据格式说明
