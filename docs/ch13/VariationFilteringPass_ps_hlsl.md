# VariationFilteringPass.ps.hlsl 技术文档

## 文件概述

`VariationFilteringPass.ps.hlsl` 实现了变化度缓冲区 (Variation Buffer) 的空间滤波通道。该着色器支持两种算子模式：**最大值分布** (Maximum Distribution) 和 **均值模糊** (Average Blur)，均使用水平/垂直分离式通道实现。

**源文件路径**: `Ch_13_Ray_Traced_Shadows_Maintaining_Real-Time_Frame_Rates/dxrShadows/Data/VariationFilteringPass.ps.hlsl`

## 算法与数学背景

### 最大值分布 (Maximum Distribution)

在半径为 2 的窗口内取最大值，将高变化度区域扩展到周围像素：

$$v_{\text{out}}(x,y) = \max_{|i| \leq 2} v_{\text{in}}(x+i, y)$$

这确保高变化度区域的边界像素也获得足够的采样数。

### 均值模糊 (Average Blur)

在半径为 4 的窗口内做均匀加权平均：

$$v_{\text{out}}(x,y) = \frac{1}{9}\sum_{i=-4}^{4} v_{\text{in}}(x+i, y)$$

这平滑了变化度场，防止自适应采样在相邻像素间跳变。

### 分离式执行顺序

在一个完整的滤波周期中，按以下顺序执行 4 次通道：

1. MAX, 水平方向
2. MAX, 垂直方向
3. AVG, 水平方向
4. AVG, 垂直方向

## 完整源码与注解

```hlsl
// 核心滤波函数
inline float4 blurPass(int lightBufferIndex, int2 pos) {
    const int halfKernelSize = 4;      // 均值模糊: 9 taps
    float4 result = float4(0.0f);
    float tapWeight = 1.0f / float(halfKernelSize * 2 + 1);

    [unroll]
    for (int i = -halfKernelSize; i <= halfKernelSize; ++i) {
        int2 offset = (BLUR_HORIZONTAL) ? pos + int2(i, 0) : pos + int2(0, i);
        result += inputBuffer[lightBufferIndex].Load(int3(offset, 0)) * tapWeight;
    }
    return result;
}

inline float4 distributeMaxPass(int lightBufferIndex, int2 pos) {
    const int halfKernelSize = 2;      // 最大值: 5 taps
    float4 result = float4(0.0f);

    [unroll]
    for (int i = -halfKernelSize; i <= halfKernelSize; ++i) {
        int2 offset = (BLUR_HORIZONTAL) ? pos + int2(i, 0) : pos + int2(0, i);
        result = max(result, inputBuffer[lightBufferIndex].Load(int3(offset, 0)));
    }
    return result;
}
```

## 输入与输出

**输入**: `inputBuffer[2]` -- 打包的变化度纹理（每个 RGBA 存储 4 个光源的变化度）

**输出**:
- `SV_TARGET0` (variationLevel0to3): 光源 0-3 的滤波变化度
- `SV_TARGET1` (variationLevel4to7): 光源 4-7 的滤波变化度

## 与其他文件的关系

- 由 `dxrShadows.cpp::variationFilteringPass()` 调度
- 处理 `DXRShadows.rt.hlsl` 输出的原始变化度缓冲
- 滤波后的变化度被回传给 `DXRShadows.rt.hlsl` 用于自适应采样
- 同时被 `VisibilityFilteringPass.ps.hlsl` 用于确定滤波核大小

## 技术要点与注意事项

1. 最大值分布使用较小核 (5 taps)，均值模糊使用较大核 (9 taps)
2. 滤波完成后会生成 Mipmap，供全局变化度估计使用
3. 变化度是 per-light 的，每个光源独立滤波
4. `OPERATOR` 和 `BLUR_PASS_DIRECTION` 由 CPU 端通过 `addDefine` 动态设置

## 扩展阅读

- "Adaptive Shadow Maps" (Fernando et al.)
- 分离式高斯滤波 (Separable Gaussian Filter)
