# sutil 支持库概览 (Template C)

## 模块用途

`sutil`（Sample Utilities）是 NVIDIA OptiX SDK 提供的**示例应用支持库**，封装了 OptiX 示例程序常用的基础设施功能。它将窗口管理、相机控制、缓冲区显示、网格加载、纹理处理、光照设置等通用功能抽象为可复用的 API，使示例程序能够专注于光线追踪算法的核心逻辑。

在第 29 章的粒子体渲染器中，`sutil` 提供了 GLFW 窗口初始化、缓冲区-GL 互操作 (PBO) 显示、相机交互、FPS 计数器等功能。

## 文件列表与职责

| 文件 | 职责 |
|------|------|
| `sutil.h` / `sutil.cpp` | 核心 API：缓冲区创建/显示、窗口初始化、文件写入、相机计算、错误处理 |
| `sutilapi.h` | DLL 导出/导入宏定义 (`SUTILAPI`) |
| `Camera.h` / `Camera.cpp` | 交互式相机：鼠标旋转/缩放/平移，自动更新 OptiX 变量 |
| `Arcball.h` / `Arcball.cpp` | 弧球旋转 (Arcball Rotation)：2D 鼠标输入→3D 旋转矩阵 |
| `Mesh.h` / `Mesh.cpp` | 三角网格数据结构和加载器（支持 OBJ 格式） |
| `OptiXMesh.h` / `OptiXMesh.cpp` | OptiX 专用网格：将 Mesh 数据上传为 OptiX Buffer/Geometry |
| `HDRLoader.h` / `HDRLoader.cpp` | HDR 格式图像加载器 |
| `PPMLoader.h` / `PPMLoader.cpp` | PPM 格式图像加载器 |
| `SunSky.h` / `SunSky.cpp` | Hosek-Wilkie 天空模型 |
| `phong.h` / `phong.cu` | Phong 材质的 CUDA 着色器 |
| `triangle_mesh.cu` | 三角网格的 CUDA 求交着色器 |
| `glew.c` / `GL/glew.h` | GLEW (OpenGL Extension Wrangler) -- OpenGL 扩展加载 |
| `GL/wglew.h` / `GL/glxew.h` | 平台特定 GLEW 头文件 |
| `glext.h` | OpenGL 扩展声明 |
| `stb/stb_image_write.h` / `.cpp` | STB 单头文件图像写入库 |
| `tinyobjloader/tiny_obj_loader.h` / `.cc` | TinyOBJLoader -- OBJ 网格加载器 |
| `rply-1.01/rply.h` / `rply.c` | RPly -- PLY 格式网格加载器 |
| `CMakeLists.txt` | CMake 构建脚本 |

## 架构关系

```
┌──────────────────────────────────────────────┐
│         optixParticleVolumes.cpp             │
│              (示例应用)                       │
└────────────┬─────────────────────────────────┘
             │ 使用
             ▼
┌──────────────────────────────────────────────┐
│              sutil 库                         │
│                                              │
│  ┌─────────────────────────────────────────┐ │
│  │         sutil.h / sutil.cpp             │ │
│  │  - createOutputBuffer()                 │ │
│  │  - displayBufferGL()                    │ │
│  │  - writeBufferToFile()                  │ │
│  │  - initGLFW()                           │ │
│  │  - calculateCameraVariables()           │ │
│  │  - displayFps()                         │ │
│  └────────────┬────────────────────────────┘ │
│               │ 依赖                         │
│  ┌────────────┼────────────────────────────┐ │
│  │ Camera.h   │  Arcball.h    Mesh.h       │ │
│  │ OptiXMesh.h│  HDRLoader.h  PPMLoader.h  │ │
│  │ SunSky.h   │                            │ │
│  └────────────┼────────────────────────────┘ │
│               │ 第三方                       │
│  ┌────────────┼────────────────────────────┐ │
│  │  GLEW      │  stb_image   tinyobjloader │ │
│  │  rply      │                            │ │
│  └────────────┴────────────────────────────┘ │
└──────────────────────────────────────────────┘
             │ 链接
             ▼
    OptiX + GLFW + ImGui + OpenGL
```

## API 概要

### sutil 核心函数

| 函数 | 说明 |
|------|------|
| `sutil::createOutputBuffer()` | 创建 OptiX 输出缓冲区（可选 PBO 互操作） |
| `sutil::resizeBuffer()` | 调整缓冲区大小及底层 GL 对象 |
| `sutil::displayBufferGL()` | 将 OptiX 缓冲区内容显示到 OpenGL 视口 |
| `sutil::writeBufferToFile()` | 将缓冲区保存为图像文件（PNG/PPM） |
| `sutil::initGLFW()` | 初始化 GLFW 窗口和 ImGui |
| `sutil::calculateCameraVariables()` | 计算针孔相机的 U/V/W 基向量 |
| `sutil::displayFps()` | 在 ImGui 中显示 FPS 计数器 |
| `sutil::samplesDir()` / `sutil::samplesPTXDir()` | 获取示例数据和 PTX 文件路径 |
| `sutil::loadTexture()` | 加载纹理为 OptiX TextureSampler |
| `sutil::currentTime()` | 获取高精度当前时间 |

### Camera 类

| 方法 | 说明 |
|------|------|
| `Camera(width, height, eye, lookat, up, ...)` | 构造函数，绑定到 OptiX 变量 |
| `process_mouse(x, y, left, right, middle)` | 处理鼠标输入，更新相机 |
| `rotate(dx, dy)` | 增量旋转 |
| `resize(w, h)` | 窗口大小变化 |

### Arcball 类

| 方法 | 说明 |
|------|------|
| `Arcball(center, radius)` | 构造弧球 |
| `rotate(from, to)` | 计算增量旋转矩阵 |

### 构建依赖 (CMakeLists.txt)

```cmake
target_link_libraries(sutil_sdk
    optix       # NVIDIA OptiX
    glfw        # 窗口管理
    imgui       # GUI 系统
    ${OPENGL_LIBRARIES}
)
```

### CUDA PTX 编译

`CMakeLists.txt` 使用 `CUDA_COMPILE_PTX()` 编译 `.cu` 文件为 PTX 中间表示，供 OptiX 在运行时 JIT 编译为目标 GPU 的机器码。

## 在第 29 章中的具体使用

1. **窗口初始化**: `sutil::initGLFW()` 创建 GLFW 窗口
2. **输出缓冲**: `sutil::createOutputBuffer()` 创建 PBO 支持的输出缓冲区
3. **缓冲区显示**: `sutil::displayBufferGL()` 将光追结果绘制到屏幕
4. **相机控制**: `sutil::Camera` 处理鼠标交互和自动旋转
5. **文件保存**: `sutil::writeBufferToFile()` 将帧保存为 PNG
6. **FPS 显示**: `sutil::displayFps()` 在 ImGui 中显示帧率
7. **PTX 路径**: `sutil::samplesPTXDir()` 定位编译后的 CUDA PTX 文件
