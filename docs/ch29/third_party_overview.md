# 第三方库概览 (Template C)

## 模块用途

第 29 章的粒子体渲染器依赖多个第三方库来提供窗口管理、GUI 交互、图像 I/O、OpenGL 扩展加载和网格加载等基础功能。这些库通过 `sutil` 支持库间接被主程序使用。

## 文件列表与职责

### GLFW -- 跨平台窗口管理

| 项目 | 说明 |
|------|------|
| **用途** | 创建 OpenGL 窗口、处理键盘/鼠标输入、管理 OpenGL 上下文 |
| **版本** | GLFW 3.x |
| **链接方式** | 外部库链接 (`glfw`) |
| **许可证** | zlib/libpng |
| **在项目中的使用** | `glfwInitialize()`, `glfwRun()`, 键盘/窗口大小回调 |

关键 API 调用：
```cpp
GLFWwindow* window = sutil::initGLFW();
glfwSetKeyCallback(window, keyCallback);
glfwSetWindowSizeCallback(window, windowSizeCallback);
glfwPollEvents();
glfwSwapBuffers(window);
```

### ImGui -- 即时模式 GUI 库

| 项目 | 说明 |
|------|------|
| **用途** | 在渲染窗口内绘制参数控制面板（滑块、复选框等） |
| **版本** | ~1.6x（带 GLFW + OpenGL2 后端） |
| **链接方式** | 外部库链接 (`imgui`) |
| **许可证** | MIT |
| **在项目中的使用** | 粒子半径、属性缩放、不透明度、传递函数等交互控制 |

关键 API 调用：
```cpp
ImGui_ImplGlfwGL2_NewFrame();
ImGui::SliderFloat("radius", &fixed_radius, min, max);
ImGui::SliderFloat("attribute scale", &wScale, 0.1f, 10.f);
ImGui::SliderFloat("sample opacity", &opacity, 0.f, 1.f);
ImGui::SliderInt("transfer function preset", &tf_type, 1, 3);
ImGui::Checkbox("camera rotate", &camera_slow_rotate);
ImGui::Render();
ImGui_ImplGlfwGL2_RenderDrawData(ImGui::GetDrawData());
```

### STB Image Write -- 单头文件图像写入库

| 项目 | 说明 |
|------|------|
| **文件** | `sutil/stb/stb_image_write.h`, `stb_image_write.cpp` |
| **用途** | 将帧缓冲区保存为 PNG/BMP/TGA 图像文件 |
| **版本** | Sean Barrett's stb 系列 |
| **许可证** | Public Domain / MIT |
| **在项目中的使用** | `sutil::writeBufferToFile()` 内部调用 |

关键特性：
- 单头文件实现（定义 `STB_IMAGE_WRITE_IMPLEMENTATION` 后包含即可编译）
- 支持 PNG、BMP、TGA 格式
- 无外部依赖

### TinyOBJLoader -- OBJ 网格加载器

| 项目 | 说明 |
|------|------|
| **文件** | `sutil/tinyobjloader/tiny_obj_loader.h`, `tiny_obj_loader.cc` |
| **用途** | 加载 Wavefront OBJ 格式的三角网格 |
| **版本** | Syoyo Fujita's tinyobjloader |
| **许可证** | MIT |
| **在项目中的使用** | `sutil::Mesh` 和 `sutil::OptiXMesh` 的内部实现 |

关键特性：
- 支持顶点/法线/纹理坐标
- 支持多材质和材质库 (.mtl)
- 单头文件实现

注意：粒子体渲染器不直接加载 OBJ 文件，但该库作为 sutil 库的标准组件被编译。

### RPly -- PLY 格式网格加载器

| 项目 | 说明 |
|------|------|
| **文件** | `sutil/rply-1.01/rply.h`, `rply.c` |
| **用途** | 加载 Stanford PLY 格式的点云和网格数据 |
| **版本** | rply 1.01 (Diego Nehab) |
| **许可证** | MIT |
| **在项目中的使用** | sutil Mesh 加载管线（PLY 格式支持） |

关键特性：
- 纯 C 实现
- 支持 ASCII 和 Binary PLY 格式
- 回调式 API

### GLEW -- OpenGL Extension Wrangler

| 项目 | 说明 |
|------|------|
| **文件** | `sutil/glew.c`, `sutil/GL/glew.h`, `GL/wglew.h`, `GL/glxew.h` |
| **用途** | 在运行时加载 OpenGL 扩展函数指针 |
| **版本** | GLEW 2.x |
| **许可证** | BSD / MIT |
| **在项目中的使用** | `glewInit()` 在窗口创建后调用 |

关键 API：
```cpp
#ifndef __APPLE__
GLenum err = glewInit();
if (err != GLEW_OK) {
    std::cout << "GLEW init failed: " << glewGetErrorString(err) << std::endl;
    exit(EXIT_FAILURE);
}
#endif
```

注意：macOS 上不需要 GLEW（使用系统 OpenGL 框架）。

## 架构关系

```
┌─────────────────────────────────────────────────┐
│          optixParticleVolumes.cpp                │
└────────┬────────────────────────────┬───────────┘
         │                            │
    ┌────▼────┐                 ┌─────▼──────┐
    │  GLFW   │                 │   ImGui    │
    │ (窗口)  │                 │  (GUI)     │
    └────┬────┘                 └─────┬──────┘
         │                            │
    ┌────▼────────────────────────────▼──────────┐
    │              sutil 库                       │
    │  ┌─────────┐  ┌────────┐  ┌──────────────┐│
    │  │  GLEW   │  │  STB   │  │ TinyOBJLoader││
    │  │(GL扩展) │  │(图像IO)│  │  (OBJ加载)   ││
    │  └─────────┘  └────────┘  └──────────────┘│
    │  ┌─────────┐                               │
    │  │  RPly   │                               │
    │  │(PLY加载)│                               │
    │  └─────────┘                               │
    └────────────────────────────────────────────┘
         │
    ┌────▼────┐
    │ OpenGL  │
    │ (显示)  │
    └─────────┘
```

## API 概要

### 各库在渲染循环中的调用顺序

```
1. GLFW: glfwPollEvents()         -- 处理输入事件
2. ImGui: ImGui_ImplGlfwGL2_NewFrame() -- 开始 GUI 帧
3. sutil Camera: process_mouse()   -- 更新相机
4. ImGui: SliderFloat() 等         -- 绘制控制面板
5. OptiX: context->launch()        -- 执行光线追踪
6. sutil: displayBufferGL()        -- 显示结果 (GLEW + OpenGL)
7. ImGui: Render()                 -- 绘制 GUI 覆盖层
8. GLFW: glfwSwapBuffers()         -- 交换前后缓冲
```

### 编译依赖关系 (CMake)

```
optixParticleVolumes
    ├── sutil_sdk (静态库)
    │   ├── OptiX
    │   ├── GLFW
    │   ├── ImGui
    │   └── OpenGL
    └── CUDA (PTX 编译)
```

### 平台兼容性

| 平台 | GLEW | GLFW | ImGui | 备注 |
|------|------|------|-------|------|
| Windows | 需要 (`wglew.h`) | 支持 | 支持 | 需要 `GLEW_BUILD` 宏 |
| Linux | 需要 (`glxew.h`) | 支持 | 支持 | |
| macOS | 不需要 (`__APPLE__` 跳过) | 支持 | 支持 | 使用系统 OpenGL |
