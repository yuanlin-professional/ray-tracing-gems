# helpers.h 技术文档

## 文件概述

`helpers.h` 是 OptiX SDK 的设备端工具函数库，提供了颜色空间转换、Phong 光照模型采样、正交法线基 (ONB) 构建、射线微分计算和色调映射 (Tone Mapping) 等通用 GPU 工具函数。

**源文件路径**: `Ch_29_Efficient_Particle_Volume_Splatting_in_a_Ray_Tracer/device_include/helpers.h`

## 算法与数学背景

### 颜色格式转换 (make_color)

将 $[0,1]^3$ 的 float3 颜色转换为 BGRA uchar4：

$$\text{uchar4}(B, G, R, 255) = (\lfloor c_b \cdot 255.99 \rfloor, \lfloor c_g \cdot 255.99 \rfloor, \lfloor c_r \cdot 255.99 \rfloor, 255)$$

注意 BGRA 顺序（非 RGBA），这是 OpenGL PBO 格式的要求。

### Phong 采样

在 Phong 光照模型的镜面反射瓣中采样方向：

$$\cos\theta = s_y^{1/(e+1)}$$
$$\phi = 2\pi s_x$$

其中 $s_x, s_y$ 为均匀随机数，$e$ 为 Phong 指数。

### ONB 构建 (create_onb)

从法线向量构建正交法线基 (Orthonormal Basis)：

$$W = \hat{n}$$
$$U = \text{normalize}(W \times (0,1,0))$$
$$V = W \times U$$

当 $W$ 接近 $(0,1,0)$ 时退化为 $U = W \times (1,0,0)$。

### Reinhard 色调映射

$$Y_{\text{rel}} = \frac{a \cdot Y}{Y_{\text{avg}}}$$
$$Y_{\text{mapped}} = Y_{\text{rel}} \cdot \frac{1 + Y_{\text{rel}} / Y_{\max}^2}{1 + Y_{\text{rel}}}$$

其中 $a = 0.04$ 为曝光调节参数。

### 色彩空间转换

支持 RGB $\leftrightarrow$ XYZ $\leftrightarrow$ Yxy 之间的完整转换链：

$$\text{RGB} \xrightarrow{M_1} \text{XYZ} \xrightarrow{} \text{Yxy} \xrightarrow{\text{tonemap}} \text{Yxy'} \xrightarrow{} \text{XYZ'} \xrightarrow{M_1^{-1}} \text{RGB'}$$

## 代码结构概览

```
helpers.h
├── make_color()                    -- float3 → uchar4 BGRA
├── sample_phong_lobe() (2个重载)   -- Phong 采样
├── get_phong_lobe_pdf()            -- Phong PDF
├── create_onb() (2个重载)          -- ONB 构建
├── differential_transfer_origin()   -- 射线微分传递
├── differential_generation_direction() -- 针孔相机射线微分
├── differential_reflect_direction()    -- 反射射线微分
├── differential_refract_direction()    -- 折射射线微分
├── Yxy2XYZ() / XYZ2rgb()              -- 色彩空间转换
├── rgb2Yxy() / Yxy2rgb()              -- 色彩空间转换
└── tonemap()                           -- Reinhard 色调映射
```

## 逐段代码详解

### make_color -- 颜色格式转换

```cuda
static __device__ __inline__ optix::uchar4 make_color(const optix::float3& c)
{
    return optix::make_uchar4(
        static_cast<unsigned char>(__saturatef(c.z)*255.99f),  // B
        static_cast<unsigned char>(__saturatef(c.y)*255.99f),  // G
        static_cast<unsigned char>(__saturatef(c.x)*255.99f),  // R
        255u);                                                  // A
}
```

`__saturatef` 将值裁剪到 $[0, 1]$，防止溢出。

### 射线微分

射线微分 (Ray Differentials) 用于纹理过滤 (Mipmap LOD 选择)：

```cuda
// 反射射线的方向微分
float3 differential_reflect_direction(float3 dPdx, float3 dDdx, float3 dNdP,
                                       float3 D, float3 N)
{
    float3 dNdx = dNdP * dPdx;
    float dDNdx = dot(dDdx, N) + dot(D, dNdx);
    return dDdx - 2*(dot(D,N)*dNdx + dDNdx*N);
}
```

## 输入与输出

- **输入**: 各函数参数（颜色值、向量、矩阵等）
- **输出**: 变换后的值

## 与其他文件的关系

- 被 `raygen.cu` 包含（主要使用 `make_color`）
- 来自 OptiX SDK 标准示例库

## 在光线追踪管线中的位置

通用工具层，被多个 GPU 程序共享使用。在粒子体渲染中主要使用 `make_color()` 进行最终颜色输出。

## 技术要点与注意事项

1. `__host__ __device__` 双限定符允许函数在 CPU 和 GPU 上都可调用
2. BGRA 顺序是历史原因（OpenGL 和 DirectX 的差异）
3. 色调映射中的 $a = 0.04$ 是较低的曝光值，适合暗场景
4. 射线微分函数在粒子体渲染中未使用，但作为通用库的一部分被包含
5. 所有函数都是 `static inline`，编译器会内联展开

## 扩展阅读

- "Ray Tracing with Ray Differentials" (Igehy, SIGGRAPH 1999)
- "Photographic Tone Reproduction for Digital Images" (Reinhard et al., 2002)
- CIE XYZ 色彩空间标准
