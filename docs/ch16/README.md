# 第16章 采样变换集锦 (Sample Transformations Zoo)

> 本章来自 *Ray Tracing Gems* 第16章，提供了光线追踪与渲染中常用的采样变换 (sample transformation) 代码片段集合。每个变换将均匀随机变量 (uniform random variable) 映射为所需的分布 (distribution) 或几何形状上的采样点。

---

## 文件总览

本章包含 **27 个 `.cpp` 代码片段**，按主题分为以下六大类：

---

### 一、基础分布 (Basic Distributions)

| 文件名 | 说明 | 文档链接 |
|--------|------|----------|
| `BoxMullerTransform.cpp` | Box-Muller 变换：将均匀分布转换为正态分布 | [BoxMullerTransform_cpp.md](BoxMullerTransform_cpp.md) |
| `NormalDistribution.cpp` | 正态分布采样（基于逆误差函数） | [NormalDistribution_cpp.md](NormalDistribution_cpp.md) |
| `NormalDistributionPDFValue.cpp` | 正态分布概率密度函数值计算 | [NormalDistributionPDFValue_cpp.md](NormalDistributionPDFValue_cpp.md) |
| `TentFunction.cpp` | 帐篷函数采样 | [TentFunction_cpp.md](TentFunction_cpp.md) |
| `TentFunctionPDFValue.cpp` | 帐篷函数 PDF 值计算 | [TentFunctionPDFValue_cpp.md](TentFunctionPDFValue_cpp.md) |

---

### 二、半球/球面采样 (Hemisphere / Sphere Sampling)

| 文件名 | 说明 | 文档链接 |
|--------|------|----------|
| `CosineWeightedHemisphereToZAxis.cpp` | 余弦加权半球采样（Z轴方向） | [CosineWeightedHemisphereToZAxis_cpp.md](CosineWeightedHemisphereToZAxis_cpp.md) |
| `CosineWeightedHemisphereToAVector.cpp` | 余弦加权半球采样（任意法线方向） | [CosineWeightedHemisphereToAVector_cpp.md](CosineWeightedHemisphereToAVector_cpp.md) |
| `ConeSampling.cpp` | 圆锥采样 | [ConeSampling_cpp.md](ConeSampling_cpp.md) |
| `PhongDistribution.cpp` | Phong 分布采样 | [PhongDistribution_cpp.md](PhongDistribution_cpp.md) |

---

### 三、映射方法 (Mapping Methods)

| 文件名 | 说明 | 文档链接 |
|--------|------|----------|
| `PolarMapping.cpp` | 极坐标映射：均匀采样圆盘 | [PolarMapping_cpp.md](PolarMapping_cpp.md) |
| `LatitudeLongitudeMapping.cpp` | 经纬度映射：均匀采样球面 | [LatitudeLongitudeMapping_cpp.md](LatitudeLongitudeMapping_cpp.md) |
| `ConcentricSquareMapping.cpp` | Shirley-Chiu 同心正方形映射 | [ConcentricSquareMapping_cpp.md](ConcentricSquareMapping_cpp.md) |
| `OctahedralConcentricMapping.cpp` | 八面体同心映射：正方形到球面 | [OctahedralConcentricMapping_cpp.md](OctahedralConcentricMapping_cpp.md) |

---

### 四、纹理/离散采样 (Texture / Discrete Sampling)

| 文件名 | 说明 | 文档链接 |
|--------|------|----------|
| `SampleDiscrete.cpp` | 离散分布采样 | [SampleDiscrete_cpp.md](SampleDiscrete_cpp.md) |
| `SampleLinear.cpp` | 线性插值采样 | [SampleLinear_cpp.md](SampleLinear_cpp.md) |
| `SampleLinearPDFValue.cpp` | 线性插值采样的 PDF 值 | [SampleLinearPDFValue_cpp.md](SampleLinearPDFValue_cpp.md) |
| `SampleBilinearInterpolation.cpp` | 双线性插值采样 | [SampleBilinearInterpolation_cpp.md](SampleBilinearInterpolation_cpp.md) |
| `SampleBilinearInterpolationPDFValue.cpp` | 双线性插值采样的 PDF 值 | [SampleBilinearInterpolationPDFValue_cpp.md](SampleBilinearInterpolationPDFValue_cpp.md) |
| `SampleMipMap.cpp` | MipMap 层级采样 | [SampleMipMap_cpp.md](SampleMipMap_cpp.md) |
| `TextureRejectionSampling.cpp` | 纹理拒绝采样 | [TextureRejectionSampling_cpp.md](TextureRejectionSampling_cpp.md) |
| `samplePiecewiseConstantArray.cpp` | 分段常数数组采样 | [samplePiecewiseConstantArray_cpp.md](samplePiecewiseConstantArray_cpp.md) |

---

### 五、介质采样 (Medium / Volume Sampling)

| 文件名 | 说明 | 文档链接 |
|--------|------|----------|
| `HomogeneousMedia.cpp` | 均匀介质自由程采样 | [HomogeneousMedia_cpp.md](HomogeneousMedia_cpp.md) |
| `InhomogeneousMedia.cpp` | 非均匀介质采样（Woodcock 追踪） | [InhomogeneousMedia_cpp.md](InhomogeneousMedia_cpp.md) |
| `HenyeyGreensteinPhaseFunction.cpp` | Henyey-Greenstein 相位函数采样 | [HenyeyGreensteinPhaseFunction_cpp.md](HenyeyGreensteinPhaseFunction_cpp.md) |

---

### 六、几何采样 (Geometric Sampling)

| 文件名 | 说明 | 文档链接 |
|--------|------|----------|
| `TriangleQuadFlipping.cpp` | 三角形采样（四边形翻折法） | [TriangleQuadFlipping_cpp.md](TriangleQuadFlipping_cpp.md) |
| `TriangleQuadWarping.cpp` | 三角形采样（四边形扭曲法） | [TriangleQuadWarping_cpp.md](TriangleQuadWarping_cpp.md) |
| `HierarchicalWarping.cpp` | 层级扭曲采样（二叉树） | [HierarchicalWarping_cpp.md](HierarchicalWarping_cpp.md) |

---

## 勘误说明 (Errata)

| 文件 | 问题描述 |
|------|---------|
| `SampleLinear.cpp` | 缺少 `float u` 参数声明——原书代码片段中 `u` 未在函数签名中出现，实际使用时需确认参数传入 |
| `ConcentricSquareMapping.cpp` | 当 `a == 0 && b == 0` 时存在除零风险——代码仅处理了 `b == 0` 的情况，未处理 `a` 同时为零的情况 |
| `PhongDistribution.cpp` | Phong 指数应使用 `2 + s` 而非 `1 + s`——代码中 `pow(1-u[0], 1/(2+s))` 对应正确的 $(s+2)/(2\pi)$ 归一化 |

---

## 参考资源

- *Ray Tracing Gems*, Chapter 16 - Sample Transformations Zoo
- [Sample Zoo 代码仓库](https://github.com/Atrix256/SampleZoo)（Alan Wolfe 维护）
- Matt Pharr - Adventures in Sampling Points on Triangles: [Part 1](https://pharr.org/matt/blog/2019/02/27/triangle-sampling-1.html), [Part 1.5](https://pharr.org/matt/blog/2019/03/13/triangle-sampling-1.5.html)
- [Evenly Distributing Points in a Triangle](http://extremelearning.com.au/evenly-distributing-points-in-a-triangle/) (Martin Roberts)

---

## 使用说明

每个文档遵循 **Template B（紧凑格式）** 结构：

1. **变换概述** — 名称、用途、应用场景
2. **数学推导** — PDF → CDF → 逆 CDF 完整推导（LaTeX 公式）
3. **完整代码与逐行注释** — 源代码 + 中文逐行注释
4. **输入与输出** — 参数表
5. **应用场景** — 实际渲染用途
6. **与相关变换的对比** — 同类变换的比较
