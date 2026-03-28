# 光线追踪中英术语表 (Ray Tracing Glossary)

本术语表收录了《Ray Tracing Gems》一书及配套代码中涉及的核心技术术语，按字母顺序排列。

---

## A

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| Acceleration Structure | 加速结构 | 用于加速光线-场景相交测试的空间数据结构，如 BVH、KD-Tree |
| Accumulation Buffer | 累积缓冲区 | 用于渐进式渲染中多帧结果累加的缓冲区 |
| Alpha Channel | Alpha 通道 | 图像透明度通道 |
| Ambient Occlusion (AO) | 环境光遮蔽 | 近似计算场景中几何体间接遮挡的全局光照技术 |
| Anisotropic | 各向异性 | 属性随方向变化的材质或分布特征 |
| Anti-aliasing (AA) | 抗锯齿 | 减少图像边缘锯齿效应的采样或过滤技术 |
| Any-Hit Shader | 任意命中着色器 | DXR 管线中光线与图元相交但未确定为最近命中时调用的着色器 |
| Aperture | 光圈 | 相机模型中光线通过的开口大小，影响景深效果 |

## B

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| Barycentric Coordinates | 重心坐标 | 三角形内部点的参数化坐标 $(u, v, w)$，满足 $u+v+w=1$ |
| Bilinear Patch | 双线性面片 | 由四个顶点定义的双线性插值曲面 |
| Bit Reversal | 位反转 | 将整数二进制位倒序排列的操作，用于负载均衡的置换算法 |
| Bounding Box | 包围盒 | 包围几何体的轴对齐或定向盒，用于加速相交测试 |
| Bottom-Level Acceleration Structure (BLAS) | 底层加速结构 | DXR 中存储单个几何体加速结构的层级 |
| BRDF (Bidirectional Reflectance Distribution Function) | 双向反射分布函数 | 描述表面反射光分布的四维函数 |
| BSDF (Bidirectional Scattering Distribution Function) | 双向散射分布函数 | 描述表面散射（反射+透射）光分布的函数 |
| Bump Mapping | 凹凸贴图 | 通过扰动法线模拟表面细节的技术 |
| BVH (Bounding Volume Hierarchy) | 层次包围体 | 树状加速结构，节点为包围体，叶节点为几何图元 |

## C

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| Caustics | 焦散 | 光线通过折射或反射后在表面形成的聚焦亮斑 |
| CDF (Cumulative Distribution Function) | 累积分布函数 | 概率分布的累积形式，用于逆变换采样 |
| Circle of Confusion (CoC) | 弥散圆 | 景深模拟中失焦点在像面上扩散形成的圆 |
| Clipping Plane | 裁剪平面 | 用于限制渲染范围的几何平面 |
| Closest-Hit Shader | 最近命中着色器 | DXR 管线中光线找到最近交点后调用的着色器 |
| Concentric Mapping | 同心映射 | 将正方形均匀采样映射到圆盘的保面积变换 |
| Cone Sampling | 锥体采样 | 在指定锥角范围内均匀采样方向的技术 |
| Cosine-Weighted Hemisphere Sampling | 余弦加权半球采样 | 按余弦分布在半球面上采样方向，用于漫反射计算 |
| CUDA (Compute Unified Device Architecture) | CUDA 统一计算设备架构 | NVIDIA 的 GPU 通用计算编程平台 |

## D

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| Deferred Shading | 延迟着色 | 先渲染几何信息到 G-Buffer，再进行光照计算的渲染技术 |
| Denoising | 降噪 | 对蒙特卡洛渲染结果进行噪声去除的后处理技术 |
| Depth Buffer / Z-Buffer | 深度缓冲区 | 存储每个像素深度值的缓冲区 |
| Depth of Field (DoF) | 景深 | 模拟相机聚焦效果的渲染技术 |
| Diffuse Reflection | 漫反射 | 光线在粗糙表面上向各方向均匀散射的现象 |
| Dispatch | 调度 | GPU 计算着色器的启动操作 |
| Dome Master | 穹顶主画面 | 用于天文馆穹幕投影的鱼眼投影格式 |
| DXR (DirectX Raytracing) | DirectX 光线追踪 | Microsoft DirectX 12 中的光线追踪 API 扩展 |

## E

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| Edge Detection | 边缘检测 | 图像处理中识别像素值突变区域的技术 |
| Environment Map | 环境贴图 | 用于模拟远处环境光照的纹理 |
| Extinction Coefficient | 消光系数 | 介质中光强度衰减的速率参数 $\sigma_t = \sigma_a + \sigma_s$ |

## F

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| Field of View (FoV) | 视场角 | 相机可见范围的角度 |
| Fisheye Projection | 鱼眼投影 | 超广角投影方式，可达 180 度或更大视场角 |
| Floating Point (IEEE 754) | 浮点数 (IEEE 754) | 计算机中实数的标准表示格式 |
| Forward Rendering | 前向渲染 | 逐对象进行光照计算的传统渲染流程 |
| Framebuffer | 帧缓冲区 | 存储最终渲染图像的内存区域 |
| Fresnel Equations | 菲涅尔方程 | 描述光在两种介质界面处反射和折射比例的公式 |

## G

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| G-Buffer (Geometry Buffer) | 几何缓冲区 | 延迟渲染中存储几何属性（法线、深度等）的多目标渲染缓冲区 |
| Global Illumination (GI) | 全局光照 | 考虑场景中所有光线交互（直接+间接光照）的渲染方法 |
| GLSL (OpenGL Shading Language) | OpenGL 着色语言 | OpenGL/Vulkan 的着色器编程语言 |

## H

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| Halton Sequence | Halton 序列 | 基于素数底的低差异序列，用于准蒙特卡洛采样 |
| HDR (High Dynamic Range) | 高动态范围 | 超出标准 8 位范围的图像表示或渲染技术 |
| Henyey-Greenstein Phase Function | Henyey-Greenstein 相函数 | 描述介质中光散射方向分布的参数化模型 |
| Hierarchical Warping | 层次变形 | 通过多级细分逐步精细化采样的技术 |
| Hit Group | 命中组 | DXR 管线中由最近命中、任意命中和相交着色器组成的着色器组 |
| HLSL (High-Level Shading Language) | 高级着色语言 | Microsoft DirectX 的着色器编程语言 |
| Hybrid Rendering | 混合渲染 | 结合光栅化和光线追踪的渲染管线 |

## I

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| IEEE 754 | IEEE 754 标准 | 浮点数运算的国际标准 |
| Importance Sampling | 重要性采样 | 按照被积函数的形状分布采样点，降低蒙特卡洛方差的技术 |
| Index of Refraction (IOR) | 折射率 | 描述光在介质中传播速度的参数 |
| Intersection Shader | 相交着色器 | DXR 管线中自定义光线-图元相交测试的着色器 |
| Inverse CDF Method | 逆 CDF 方法 | 通过对累积分布函数求逆来生成特定分布随机样本的方法 |
| Irradiance | 辐照度 | 单位面积上接收到的辐射功率，单位 $W/m^2$ |
| Irradiance Volume | 辐照度体 | 在三维空间中存储辐照度信息的体积数据结构 |

## J

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| Jittering | 抖动 | 在采样位置上添加随机偏移以减少规则采样的走样 |

## K

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| Kernel (Splatting) | 核函数 (喷溅) | 光子映射中将光子能量扩散到周围像素的加权函数 |

## L

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| Lambert's Cosine Law | 朗伯余弦定律 | 理想漫反射面辐射强度与观察角余弦成正比的定律 |
| LCG (Linear Congruential Generator) | 线性同余发生器 | 基于线性递推的伪随机数生成算法 |
| Light Map | 光照贴图 | 预计算的纹理，存储静态场景中表面的光照信息 |
| Lobe (BRDF) | 叶瓣 (BRDF) | BRDF 中的一个反射分量（如漫反射叶瓣、高光叶瓣） |
| Load Balancing | 负载均衡 | 将计算任务均匀分配到多个处理器的技术 |
| Low-Discrepancy Sequence | 低差异序列 | 在空间中均匀分布的确定性序列，如 Halton、Sobol |

## M

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| Microfacet Model | 微面元模型 | 将粗糙表面建模为微小镜面集合的 BRDF 模型 |
| MIP Map | MIP 贴图 | 纹理的多级分辨率金字塔，用于纹理过滤 |
| Miss Shader | 未命中着色器 | DXR 管线中光线未击中任何几何体时调用的着色器 |
| Monte Carlo Integration | 蒙特卡洛积分 | 使用随机采样近似计算积分的数值方法 |
| Motion Vectors | 运动向量 | 描述像素在相邻帧间运动的二维向量场 |
| Multi-Hit | 多重命中 | 在一次光线遍历中收集多个交点的技术 |

## N

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| Nested Volumes | 嵌套体积 | 多个重叠的体积介质，需要确定优先级和边界行为 |
| Normal Distribution | 正态分布 | 以钟形曲线为特征的概率分布，又称高斯分布 |
| Normal Map | 法线贴图 | 存储表面法线扰动信息的纹理 |
| Null Scattering | 空散射 | 体积渲染中引入的虚拟散射事件，使非均匀介质的采样等效于均匀介质 |

## O

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| Octahedral Mapping | 八面体映射 | 将球面方向编码为正方形上的二维坐标的映射方法 |
| Opacity | 不透明度 | 材质阻挡光线传播的程度 |
| OptiX | OptiX | NVIDIA 的 GPU 光线追踪引擎和 API |
| Origin Offset | 原点偏移 | 对光线起点进行微小偏移以避免自相交的技术 |

## P

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| Participating Media | 参与介质 | 与光线交互（吸收、散射、发射）的体积介质，如烟雾、云 |
| Path Tracing | 路径追踪 | 从相机出发追踪光线完整传播路径的全局光照算法 |
| PDF (Probability Density Function) | 概率密度函数 | 描述连续随机变量概率分布的函数 |
| Permutation | 置换 | 元素顺序的重新排列，用于负载均衡中的像素分配 |
| Phase Function | 相函数 | 描述体积散射中光线方向改变概率分布的函数 |
| Phong Distribution | Phong 分布 | 基于余弦幂函数的经验高光分布模型 |
| Photon Mapping | 光子映射 | 通过从光源发射光子追踪并存储交点来计算全局光照的方法 |
| Piecewise Constant | 分段常数 | 在每个区间内取常数值的函数，用于离散化连续分布 |
| Pixel Shader / Fragment Shader | 像素着色器 / 片段着色器 | 光栅化管线中计算每个像素最终颜色的着色器阶段 |
| Progressive Rendering | 渐进式渲染 | 随时间逐步累积和改善图像质量的渲染方式 |

## Q

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| Quasi-Monte Carlo (QMC) | 准蒙特卡洛 | 使用低差异序列替代伪随机数的积分方法 |

## R

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| Radiance | 辐射度 / 辐亮度 | 单位面积单位立体角的辐射功率，单位 $W/(m^2 \cdot sr)$ |
| Radiance Cache | 辐射度缓存 | 在空间离散点上缓存辐射度信息以加速全局光照计算 |
| Radiance Probe | 辐射度探针 | 空间中采样点处存储各方向辐射度的数据结构 |
| Rasterization | 光栅化 | 将几何图元转换为像素片段的传统渲染方法 |
| Ray | 光线 | 由起点 $\mathbf{o}$ 和方向 $\mathbf{d}$ 定义的半直线：$\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ |
| Ray Generation Shader | 光线生成着色器 | DXR 管线入口，为每个像素生成初始光线 |
| Ray Payload | 光线载荷 | 随光线传播携带的自定义数据结构 |
| Ray Tracing | 光线追踪 | 模拟光线传播路径计算图像的渲染方法 |
| Reflection | 反射 | 光线遇到表面后改变方向返回同一介质的现象 |
| Refraction | 折射 | 光线穿过两种介质界面时改变传播方向的现象 |
| Rejection Sampling | 拒绝采样 | 通过在包围域内均匀采样并丢弃不满足条件的样本来生成目标分布的方法 |
| Render Target | 渲染目标 | GPU 渲染输出写入的纹理或缓冲区 |
| Root Signature | 根签名 | DirectX 12 中描述着色器资源绑定布局的对象 |
| Roughness | 粗糙度 | 描述表面微观粗糙程度的参数，影响高光反射的扩散 |

## S

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| Sample Variance | 样本方差 | 衡量采样值分散程度的统计量 |
| Scattering | 散射 | 光与介质交互改变传播方向的现象 |
| Scene Epsilon | 场景 epsilon | 用于避免浮点精度导致自相交的微小偏移量 |
| Screen Space | 屏幕空间 | 以像素坐标系定义的二维空间 |
| Self-Intersection | 自相交 | 光线从表面出发时错误地与同一表面再次相交的问题 |
| Shader Resource View (SRV) | 着色器资源视图 | DirectX 中用于在着色器中读取资源的视图 |
| Shadow Ray | 阴影光线 | 从着色点向光源发射的光线，用于判断是否被遮挡 |
| Signed Distance Field (SDF) | 有符号距离场 | 存储到最近表面距离的三维场，正值在外部，负值在内部 |
| Solid Angle | 立体角 | 三维空间中从一点观察物体所张的角度范围，单位球面度 (sr) |
| Specular Reflection | 镜面反射 | 光线在光滑表面上按入射角等于反射角的定律反射 |
| Splatting | 喷溅 / 泼溅 | 将光子能量扩散到其邻域像素的技术 |
| Stereoscopic Rendering | 立体渲染 | 为左右眼分别渲染图像以产生三维立体效果的技术 |
| Stochastic | 随机 | 涉及随机过程的方法或结果 |

## T

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| Tachyon | Tachyon | 开源并行光线追踪引擎，由 John Stone 开发 |
| TEA (Tiny Encryption Algorithm) | 微型加密算法 | 用于在 GPU 上快速生成高质量伪随机数的加密算法 |
| Temporal Accumulation | 时间累积 | 利用前帧信息改善当前帧渲染质量的技术 |
| Tent Function | 帐篷函数 | 三角形状的分段线性概率分布，用于像素重建滤波器 |
| Texel | 纹素 | 纹理中的基本元素，类似于像素之于图像 |
| Top-Level Acceleration Structure (TLAS) | 顶层加速结构 | DXR 中管理多个实例变换的上层加速结构 |
| Transfer Function | 传递函数 | 将标量数据值映射到颜色和不透明度的函数，常用于体积渲染 |
| Transmittance | 透射率 | 光线穿过介质后剩余强度的比例 |
| Traversal | 遍历 | 在加速结构中查找光线交点的过程 |

## U

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| ULP (Unit in the Last Place) | 最后一位的单位 | 浮点数相邻可表示值之间的最小差值 |
| Unordered Access View (UAV) | 无序访问视图 | DirectX 中允许着色器读写的资源视图 |

## V

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| Variance | 方差 | 随机变量与其期望值偏差的平方的期望 |
| Variance Reduction | 方差缩减 | 降低蒙特卡洛估计器方差的技术集合 |
| Visibility | 可见性 | 判断两点之间是否存在遮挡的几何属性 |
| Volume Rendering | 体积渲染 | 可视化三维标量场的渲染技术，不需要提取等值面 |
| Volumetric Scattering | 体积散射 | 光线在参与介质内部发生的散射现象 |

## W

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| Warping | 变形映射 | 将一种分布的样本变换到另一种分布的数学变换 |
| Woodcock Tracking | Woodcock 追踪 | 体积渲染中使用空散射处理非均匀介质的随机采样方法 |
| World Space | 世界空间 | 场景全局统一的三维坐标系 |

## Z

| 英文术语 | 中文翻译 | 说明 |
|---------|---------|------|
| Z-Fighting | 深度冲突 | 两个几何面距离过近导致深度缓冲区无法正确区分的现象 |

---

> 本术语表随文档持续更新。如发现遗漏术语，请在对应章节文档中标注。
