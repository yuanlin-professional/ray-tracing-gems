# Ray Tracing Gems 代码文档

本文档为《Ray Tracing Gems: High-Quality and Real-Time Rendering with DXR and Other APIs》(Apress, 2019) 配套代码仓库的中文详细技术文档。

---

## 术语表

- [光线追踪中英术语表](glossary.md) — 约 100 个核心术语的中英对照及释义

---

## 章节索引

### 第一部分：光线追踪基础 (Part I: Ray Tracing Basics)

| 章节 | 主题 | 文件数 | 文档目录 |
|------|------|--------|---------|
| Chapter 4 | 天文馆穹顶主相机 (Planetarium Dome Master Camera) | 2 | [ch04/](ch04/README.md) |
| Chapter 6 | 快速鲁棒的自相交避免方法 (Avoiding Self-Intersection) | 1 | [ch06/](ch06/README.md) |

### 第二部分：相交与效率 (Part II: Intersections and Efficiency)

| 章节 | 主题 | 文件数 | 文档目录 |
|------|------|--------|---------|
| Chapter 8 | 双线性面片的几何求交 (Ray-Bilinear Patch Intersections) | 2 | [ch08/](ch08/README.md) |
| Chapter 9 | DXR 中的多重命中光线追踪 (Multi-Hit Ray Tracing in DXR) | 4 | [ch09/](ch09/README.md) |
| Chapter 10 | 高扩展效率的简单负载均衡方案 (Load-Balancing Scheme) | 3 | [ch10/](ch10/README.md) |
| Chapter 11 | 嵌套体积材质的自动处理 (Nested Volumes) | 3 | [ch11/](ch11/README.md) |
| Chapter 12 | 凹凸终结器问题的微面元阴影函数 (Bump Terminator) | 1 | [ch12/](ch12/README.md) |

### 第三部分：光线追踪中的阴影与光照 (Part III: Shadows and Lighting)

| 章节 | 主题 | 文件数 | 文档目录 |
|------|------|--------|---------|
| Chapter 13 | 保持实时帧率的光线追踪阴影 (Ray Traced Shadows) | 14 | [ch13/](ch13/README.md) |

### 第四部分：采样 (Part IV: Sampling)

| 章节 | 主题 | 文件数 | 文档目录 |
|------|------|--------|---------|
| Chapter 15 | 采样的重要性 (On the Importance of Sampling) | 2 | [ch15/](ch15/README.md) |
| Chapter 16 | 采样变换集锦 (Sample Transformations Zoo) | 27 | [ch16/](ch16/README.md) |

### 第五部分：光照贴图与全局光照 (Part V: Light Maps and GI)

| 章节 | 主题 | 文件数 | 文档目录 |
|------|------|--------|---------|
| Chapter 23 | Frostbite 中的交互式光照贴图预览 (Light Map Preview) | 4 | [ch23/](ch23/README.md) |
| Chapter 24 | 基于光子映射的实时全局光照 (Photon Mapping GI) | 6 | [ch24/](ch24/README.md) |
| Chapter 25 | 实时光线追踪的混合渲染 (Hybrid Rendering) | 6 | [ch25/](ch25/README.md) |
| Chapter 26 | 延迟混合路径追踪 (Deferred Hybrid Path Tracing) | 1 | [ch26/](ch26/README.md) |

### 第六部分：科学可视化与体积渲染 (Part VI: Visualization and Volumes)

| 章节 | 主题 | 文件数 | 文档目录 |
|------|------|--------|---------|
| Chapter 27 | 高保真科学可视化的交互式光线追踪 (Scientific Visualization) | 6 | [ch27/](ch27/README.md) |
| Chapter 28 | 非均匀体积的光线追踪 (Inhomogeneous Volumes) | 4 | [ch28/](ch28/README.md) |
| Chapter 29 | 光线追踪器中的高效粒子体积喷溅 (Particle Volume Splatting) | 15 | [ch29/](ch29/README.md) |
| Chapter 30 | 屏幕空间光子映射焦散 (Screen Space Photon Mapping Caustics) | 1 | [ch30/](ch30/README.md) |

### 第七部分：高级渲染技术 (Part VII: Advanced Techniques)

| 章节 | 主题 | 文件数 | 文档目录 |
|------|------|--------|---------|
| Chapter 32 | 基于辐射度缓存的精确实时镜面反射 (Radiance Caching Reflections) | 3 | [ch32/](ch32/README.md) |

---

## 技术栈概览

| 技术 | 用途 | 涉及章节 |
|------|------|---------|
| **CUDA / OptiX** | NVIDIA GPU 光线追踪 | Ch 4, 6, 8, 27, 28, 29 |
| **HLSL / DXR** | DirectX 光线追踪着色器 | Ch 9, 13, 23, 24, 25, 32 |
| **GLSL** | OpenGL/Vulkan 着色器 | Ch 26 |
| **C++** | 主机端逻辑、算法实现 | Ch 10, 11, 12, 15, 16, 25, 27, 28, 29 |

---

## 文档规范

- **语言**：简体中文，英文术语首次出现时括注
- **代码引用**：保留英文函数名/变量名，用反引号标记
- **数学公式**：LaTeX `$$...$$` 块级，`$...$` 行内
- **代码块**：按源文件语言标注 (cuda/hlsl/cpp/glsl)
- **行号引用**：格式为"（第 X-Y 行）"

---

## 已知勘误

以下勘误来自仓库 README，已在对应章节文档中标注：

| 文件 | 问题 | 状态 |
|------|------|------|
| Ch 16 `SampleLinear.cpp` | 缺少 `float u` 参数 | 已在文档中标注 |
| Ch 16 `ConcentricSquareMapping.cpp` | `a==0 && b==0` 时除零 | 已在文档中标注 |
| Ch 16 `PhongDistribution.cpp` | 应使用 `2+s` 而非 `1+s` | 已在文档中标注 |

---

> 本文档基于 Ray Tracing Gems (Apress, 2019) 配套代码生成。书中更多理论背景请参阅原著。
