# DepthPass.ps.hlsl 技术文档

## 文件概述

`DepthPass.ps.hlsl` 实现了最简单的深度预通道 (Depth Pre-Pass) 像素着色器。它仅执行材质采样以处理 alpha-test 透明度，不输出任何颜色数据，仅写入深度缓冲。

**源文件路径**: `Ch_13_Ray_Traced_Shadows_Maintaining_Real-Time_Frame_Rates/dxrShadows/Data/DepthPass.ps.hlsl`

## 完整源码与注解

```hlsl
__import DefaultVS;      // 导入 Falcor 默认顶点着色器
__import ShaderCommon;    // 导入 Falcor 公共着色数据
__import Shading;         // 导入材质/着色系统

void main(VertexOut vOut)
{
    // 准备着色数据（触发纹理采样，处理 alpha-test）
    // 不输出任何颜色，仅写入深度缓冲
    prepareShadingData(vOut, gMaterial, gCamera.posW);
}
```

## 算法与数学背景

深度预通道是一种经典的渲染优化技术。在光照通道之前先填充深度缓冲，光照通道使用 `DepthFunc::Equal` 测试，可以：

1. 消除 overdraw（重复绘制同一像素）
2. 提前剔除被遮挡片元
3. 为后续的深度依赖操作（运动向量、阴影）提供完整深度

## 输入与输出

**输入**: 顶点属性（位置、法线等）

**输出**: 仅 `SV_Depth`（由硬件自动写入深度缓冲）

## 与其他文件的关系

- 在 `DepthPrePass` 模式下使用，与 `MotionVectors.ps.hlsl` 配合
- 与 `DepthAndNormalsPass.ps.hlsl`（G-Buffer 模式）互斥
- 由 `dxrShadows.cpp::depthPass()` 调度

## 在光线追踪管线中的位置

位于管线最前端，为后续所有依赖深度的通道准备数据。

## 技术要点与注意事项

1. `prepareShadingData` 会采样漫反射贴图，确保 alpha-test 物体的深度正确
2. 无返回值 (`void main`)，不写入任何颜色目标
3. 配合 `DepthStencilState::LessEqual` 使用

## 扩展阅读

- Early-Z / Z Pre-Pass 优化技术
- GPU 深度缓冲硬件加速原理
