# DebugPass.ps.hlsl 技术文档

## 文件概述

`DebugPass.ps.hlsl` 实现了调试可视化全屏通道 (Debug Visualization Pass)，可将管线中间数据以可视化形式输出到屏幕。支持深度、视空间位置、世界空间位置、运动向量、变化度、法线、自适应采样数和滤波核大小等多种可视化模式。

**源文件路径**: `Ch_13_Ray_Traced_Shadows_Maintaining_Real-Time_Frame_Rates/dxrShadows/Data/DebugPass.ps.hlsl`

## 算法与数学背景

### 运动向量可视化

在 $64 \times 64$ 像素的块中心绘制线段，从当前位置指向前一帧位置。使用点到线段的最短距离判断是否在线段上：

$$d = \min_{t \in [0,1]} \|\mathbf{p} - (\mathbf{a} + t(\mathbf{b} - \mathbf{a}))\|$$

$$t = \text{clamp}\left(\frac{(\mathbf{p} - \mathbf{a}) \cdot (\mathbf{b} - \mathbf{a})}{|\mathbf{b} - \mathbf{a}|^2}, 0, 1\right)$$

### 变化度热图

将变化度值映射到黄色到品红色的渐变：

$$\text{color} = \text{lerp}((1,1,0.5), (1,0,1), v)$$

### 滤波核大小可视化

将变化度映射到核索引 $k \in [0, 4)$，然后使用颜色编码：

| 范围 | 颜色 | 含义 |
|------|------|------|
| $[0, 0.25)$ | 白→蓝 | 1 tap，几乎无滤波 |
| $[0.25, 0.5)$ | 黄→黄 | 3 taps |
| $[0.5, 0.75)$ | 粉→红 | 5 taps |
| $[0.75, 1.0]$ | 深粉→暗红 | 7-9 taps |

## 代码结构概览

```
DebugPass.ps.hlsl
├── DebugPassCB 常量缓冲
├── 输入纹理声明
├── distanceFromLineSegment()   -- 点到线段距离
├── visualizeMotionVectors()    -- 运动向量可视化
├── getVariationLevelValue()    -- 从打包纹理提取变化度
└── main()                      -- 像素着色器入口 (条件编译分支)
    ├── OUTPUT_DEPTH
    ├── OUTPUT_VIEW_SPACE_POSITION
    ├── OUTPUT_WORLD_SPACE_POSITION
    ├── OUTPUT_MOTION_VECTORS
    ├── OUTPUT_VARIATION_LEVEL
    ├── OUTPUT_NORMALS_WORLD
    ├── OUTPUT_ADAPTIVE_SAMPLES_COUNT
    └── OUTPUT_FILTERING_KERNEL_SIZE
```

## 输入与输出

**输入**: 深度缓冲、法线缓冲、运动向量缓冲、变化度缓冲、采样历史缓冲、采样点数据

**输出**: `SV_TARGET` -- 调试可视化颜色

## 与其他文件的关系

- 包含 `CommonUtils.hlsl` 用于深度/位置重建
- 由 `dxrShadows.cpp::debugVisualizationsPass()` 调度
- 通过 shader defines 选择可视化模式

## 技术要点与注意事项

1. 运动向量可视化使用 $64 \times 64$ 块中心的代表性向量，避免过密的线段
2. 法线可视化需从视空间转换到世界空间：$\mathbf{n}_{\text{world}} = M_V^T \cdot \mathbf{n}_{\text{view}}$
3. 采样数可视化通过 `decodeVariationAndSampleCount` 解码历史缓冲

## 扩展阅读

- GPU-based Debug Visualization Techniques
- 调试渲染管线的最佳实践
