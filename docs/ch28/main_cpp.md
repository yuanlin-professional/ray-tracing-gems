# main.cpp 技术文档

## 文件概述

**文件路径**: `Ch_28_Ray_Tracing_Inhomogeneous_Volumes/main.cpp`

本文件是体积路径追踪 (Volume Path Tracer) 应用程序的主入口，实现了一个交互式的实时体积渲染程序。它负责：
1. 初始化 OpenGL 窗口和上下文
2. 设置 CUDA 与 OpenGL 的互操作 (Interop)
3. 加载 HDR 环境贴图
4. 管理相机控制和用户交互
5. 在主循环中调度 CUDA 体积渲染内核并显示结果

程序支持鼠标旋转相机、滚轮缩放、键盘切换体积类型和环境贴图、调节曝光度等交互操作。

## 算法与数学背景

### 球坐标相机模型

相机使用球坐标 (Spherical Coordinates) 系统：

$$\mathbf{cam\_dir} = \begin{pmatrix} -\sin\phi \sin\theta \\ -\cos\theta \\ -\cos\phi \sin\theta \end{pmatrix}$$

$$\mathbf{cam\_right} = \begin{pmatrix} \cos\phi \\ 0 \\ -\sin\phi \end{pmatrix}$$

$$\mathbf{cam\_up} = \begin{pmatrix} -\sin\phi \cos\theta \\ \sin\theta \\ -\cos\phi \cos\theta \end{pmatrix}$$

其中 $\phi$ 为方位角 (Azimuth)，$\theta$ 为极角 (Polar Angle)。

相机位置：

$$\mathbf{cam\_pos} = -\mathbf{cam\_dir} \cdot d_{base} \cdot 0.95^{zoom}$$

### 渐进式累积

第 $n$ 帧的累积平均：

$$\bar{C}_n = \bar{C}_{n-1} + \frac{C_n - \bar{C}_{n-1}}{n+1}$$

这是数值稳定的在线均值计算公式（避免了对所有历史值求和导致的浮点溢出）。

### Reinhard 色调映射

$$L_d = L \cdot \frac{1 + L \cdot 0.1}{1 + L}$$

其中 $L$ 为 HDR 亮度值，$L_d$ 为映射后的 LDR 值。系数 0.1 控制高光压缩程度。

## 代码结构概览

```
main.cpp
├── check_success()          // 错误检查宏
├── init_opengl()            // 初始化 GLFW + GLEW
├── init_cuda()              // 初始化 CUDA（与 OpenGL 互操作）
├── add_shader()             // 编译添加 GLSL 着色器
├── create_shader_program()  // 创建 GL 着色器程序
├── create_quad()            // 创建全屏四边形
├── struct Window_context    // 窗口回调上下文
├── handle_scroll()          // 鼠标滚轮回调
├── handle_key()             // 键盘回调
├── handle_mouse_button()    // 鼠标按钮回调
├── handle_mouse_pos()       // 鼠标移动回调
├── resize_buffers()         // 窗口大小变化时重新分配缓冲区
├── create_environment()     // 创建 CUDA 环境贴图纹理
├── update_camera()          // 更新相机参数
└── main()                   // 主函数
```

## 逐段代码详解

### 错误检查宏

```cpp
#define check_success(expr) \
    do { \
        if(!(expr)) { \
            fprintf(stderr, "Error in file %s, line %u: \"%s\".\n", \
                __FILE__, __LINE__, #expr); \
            exit(EXIT_FAILURE); \
        } \
    } while(false)
```

通用的错误检查宏，将表达式字符串化（`#expr`）输出到错误信息中，便于调试。

### init_opengl() - OpenGL 初始化

```cpp
static GLFWwindow *init_opengl()
{
    check_success(glfwInit());
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
    glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);

    GLFWwindow *window = glfwCreateWindow(
        1024, 1024, "volume path tracer", NULL, NULL);
    // ...
    glfwSwapInterval(0);  // 禁用 VSync，最大化帧率
    return window;
}
```

创建 OpenGL 3.3 Core Profile 窗口，禁用垂直同步以获得最大渲染吞吐量（渐进式渲染需要尽可能多的帧来累积样本）。

### init_cuda() - CUDA 初始化

```cpp
static void init_cuda()
{
    int cuda_devices[1];
    unsigned int num_cuda_devices;
    check_success(cudaGLGetDevices(&num_cuda_devices, cuda_devices,
        1, cudaGLDeviceListAll) == cudaSuccess);
    check_success(cudaSetDevice(cuda_devices[0]) == cudaSuccess);
}
```

使用 `cudaGLGetDevices()` 查找支持 OpenGL 互操作的 CUDA 设备，确保 CUDA 和 OpenGL 使用同一 GPU。

### create_shader_program() - GL 着色器

```cpp
static GLuint create_shader_program()
{
    const char *vert =
        "#version 330\n"
        "in vec3 Position;\n"
        "out vec2 TexCoord;\n"
        "void main() {\n"
        "    gl_Position = vec4(Position, 1.0);\n"
        "    TexCoord = 0.5 * Position.xy + vec2(0.5);\n"
        "}\n";

    const char *frag =
        "#version 330\n"
        "in vec2 TexCoord;\n"
        "out vec4 FragColor;\n"
        "uniform sampler2D TexSampler;\n"
        "void main() {\n"
        "    FragColor = texture(TexSampler, TexCoord);\n"
        "}\n";
    // ...
}
```

极简的着色器程序：顶点着色器将 NDC 坐标转换为纹理坐标，片段着色器进行纹理采样。这些着色器仅用于将 CUDA 渲染的纹理显示到屏幕上。

### Window_context 和回调函数

```cpp
struct Window_context
{
    int zoom_delta;
    bool moving;
    double move_start_x, move_start_y;
    double move_dx, move_dy;
    float exposure;
    unsigned int config_type;
};
```

通过 GLFW 的用户指针 (User Pointer) 机制传递上下文。

回调功能汇总：
| 回调 | 功能 |
|------|------|
| `handle_scroll` | 鼠标滚轮 → 缩放 |
| `handle_key` | 键盘 ESC 退出，`[/]` 调节曝光，空格切换配置 |
| `handle_mouse_button` | 左键拖拽开始/结束 |
| `handle_mouse_pos` | 鼠标移动 → 相机旋转 |

### resize_buffers() - 缓冲区重分配

```cpp
static void resize_buffers(
    float3 **accum_buffer_cuda,
    cudaGraphicsResource_t *display_buffer_cuda,
    int width, int height, GLuint display_buffer)
{
    // 重新分配 OpenGL PBO
    glBindBuffer(GL_PIXEL_UNPACK_BUFFER, display_buffer);
    glBufferData(GL_PIXEL_UNPACK_BUFFER, width * height * 4,
        NULL, GL_DYNAMIC_COPY);

    // 注册/重新注册 CUDA-GL 互操作
    cudaGraphicsGLRegisterBuffer(display_buffer_cuda, display_buffer,
        cudaGraphicsRegisterFlagsWriteDiscard);

    // 重新分配 CUDA 累积缓冲区
    cudaMalloc(accum_buffer_cuda, width * height * sizeof(float3));
}
```

当窗口大小变化时，需要重新分配所有缓冲区：
- **GL PBO (Pixel Buffer Object)**：用于 CUDA-GL 互操作的显示缓冲区
- **CUDA 累积缓冲区**：`float3` 格式，存储渐进式累积的 HDR 颜色

### create_environment() - 环境贴图加载

```cpp
static bool create_environment(
    cudaTextureObject_t *env_tex,
    cudaArray_t *env_tex_data,
    const char *envmap_name)
{
    float *pixels;
    unsigned int rx, ry;
    load_hdr_float4(&pixels, &rx, &ry, envmap_name);

    // 创建 CUDA 数组并复制数据
    cudaMallocArray(env_tex_data, &channel_desc, rx, ry);
    cudaMemcpyToArray(*env_tex_data, 0, 0, pixels, ...);

    // 配置纹理参数
    tex_desc.addressMode[0] = cudaAddressModeWrap;   // U 方向环绕
    tex_desc.addressMode[1] = cudaAddressModeClamp;  // V 方向钳制
    tex_desc.filterMode = cudaFilterModeLinear;       // 双线性插值
    tex_desc.normalizedCoords = 1;                     // 归一化坐标

    cudaCreateTextureObject(env_tex, &res_desc, &tex_desc, NULL);
}
```

将 HDR 图像加载为 CUDA 纹理对象，配置了：
- U 方向环绕寻址（水平方向 360 度无缝衔接）
- V 方向钳制寻址（极点不环绕）
- 双线性插值滤波

### update_camera() - 相机更新

```cpp
static void update_camera(
    Kernel_params &kernel_params,
    double phi, double theta,
    float base_dist, int zoom)
{
    kernel_params.cam_dir.x = float(-sin(phi) * sin(theta));
    kernel_params.cam_dir.y = float(-cos(theta));
    kernel_params.cam_dir.z = float(-cos(phi) * sin(theta));
    // ... cam_right, cam_up 类似

    const float dist = float(base_dist * pow(0.95, double(zoom)));
    kernel_params.cam_pos = -cam_dir * dist;
}
```

球坐标系相机：
- $\phi$：由鼠标水平拖拽控制
- $\theta$：由鼠标垂直拖拽控制，限制在 $[0, \pi]$
- 缩放：每次滚轮操作将距离乘以 0.95

### main() - 主循环

```cpp
int main(const int argc, const char* argv[])
{
    // 初始化
    window = init_opengl();
    init_cuda();

    // 初始化内核参数
    kernel_params.cam_focal = float(1.0 / tan(90.0 / 2.0 * (2.0 * M_PI / 360.0)));
    kernel_params.max_interactions = 1024;
    kernel_params.max_extinction = 100.0f;
    kernel_params.albedo = 0.8f;

    // 加载环境贴图（如果命令行提供）
    if (argc >= 2)
        env_tex = create_environment(&kernel_params.env_tex, &env_tex_data, argv[1]);
```

初始化关键参数：
- `cam_focal`：90 度 FOV 对应的焦距
- `max_interactions = 1024`：路径最大交互次数
- `max_extinction = 100.0`：最大消光系数
- `albedo = 0.8`：散射反照率

```cpp
    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();

        // 处理用户输入（相机旋转、缩放、参数切换等）
        // 输入变化时重置 iteration = 0

        // 映射 GL 缓冲区供 CUDA 访问
        cudaGraphicsMapResources(1, &display_buffer_cuda, 0);
        cudaGraphicsResourceGetMappedPointer(&p, &size_p, display_buffer_cuda);
        kernel_params.display_buffer = reinterpret_cast<unsigned int *>(p);

        // 启动 CUDA 内核
        dim3 threads_per_block(16, 16);
        dim3 num_blocks((width + 15) / 16, (height + 15) / 16);
        void *params[] = { &kernel_params };
        cudaLaunchKernel((const void *)&volume_rt_kernel,
            num_blocks, threads_per_block, params);
        ++kernel_params.iteration;

        // 取消映射并显示
        cudaGraphicsUnmapResources(1, &display_buffer_cuda, 0);

        // 将 PBO 数据上传到纹理并绘制全屏四边形
        glBindBuffer(GL_PIXEL_UNPACK_BUFFER, display_buffer);
        glTexSubImage2D(GL_TEXTURE_2D, 0, 0, 0, width, height,
            GL_BGRA, GL_UNSIGNED_BYTE, NULL);
        glDrawArrays(GL_TRIANGLES, 0, 6);

        glfwSwapBuffers(window);
    }
```

主循环的关键步骤：
1. **事件处理**：检测用户输入，更新相机和参数
2. **缓冲区映射**：将 GL PBO 映射为 CUDA 可写指针
3. **内核调度**：以 16x16 的线程块启动体积渲染内核
4. **渐进累积**：每帧 `iteration++`，内核使用此值计算运行平均
5. **纹理更新**：通过 PBO 高效地将 CUDA 结果传递给 GL 纹理
6. **全屏绘制**：使用简单的纹理采样着色器显示结果

## 关键算法深入分析

### CUDA-OpenGL 互操作流程

```
┌───────────────────────────────────────────────────┐
│                   CPU (main loop)                  │
│                                                    │
│  1. cudaGraphicsMapResources()                     │
│     ↓                                              │
│  2. cudaGraphicsResourceGetMappedPointer()         │
│     → 获取 CUDA 可写指针                            │
│     ↓                                              │
│  3. cudaLaunchKernel(volume_rt_kernel)             │
│     → GPU 渲染到映射的缓冲区                        │
│     ↓                                              │
│  4. cudaGraphicsUnmapResources()                   │
│     → 缓冲区回归 GL 控制                            │
│     ↓                                              │
│  5. glTexSubImage2D(GL_PIXEL_UNPACK_BUFFER)        │
│     → PBO 数据上传到纹理（GPU 内存拷贝，无 CPU 参与）│
│     ↓                                              │
│  6. glDrawArrays() → 绘制全屏四边形                 │
└───────────────────────────────────────────────────┘
```

整个数据流从未离开 GPU，避免了 GPU→CPU→GPU 的往返。

### 渐进式渲染的重置策略

以下事件触发渲染重置（`iteration = 0`）：
- 相机旋转/缩放
- 窗口大小变化
- 体积类型切换
- 环境贴图开关切换

重置后所有累积数据丢失，从第一帧重新开始。

## 输入与输出

### 命令行参数
| 参数 | 说明 |
|------|------|
| `argv[1]` | （可选）HDR 环境贴图文件路径 |

### 键盘控制
| 按键 | 功能 |
|------|------|
| ESC | 退出程序 |
| `[` / 数字键盘 `-` | 降低曝光度 |
| `]` / 数字键盘 `+` | 增加曝光度 |
| Space | 切换体积类型 / 环境贴图 |

### 鼠标控制
| 操作 | 功能 |
|------|------|
| 左键拖拽 | 旋转相机 |
| 滚轮 | 缩放 |

## 与其他文件的关系

- **`hdr_loader.h`**：被包含并调用 `load_hdr_float4()` 加载环境贴图
- **`volume_kernel.h`**：提供 `Kernel_params` 结构体和 `volume_rt_kernel` 声明
- **`volume_kernel.cu`**：CUDA 内核实现，通过 `cudaLaunchKernel()` 调用

## 在光线追踪管线中的位置

```
用户交互 → main.cpp（CPU 端调度） → volume_kernel.cu（GPU 端渲染） → OpenGL 显示
           ^^^^^^^^^^^^^^^^^^^^^^^
           本文件实现此部分
```

本文件是整个应用程序的**主控制器**，管理所有资源的生命周期和渲染循环。

## 技术要点与注意事项

1. **VSync 禁用**：`glfwSwapInterval(0)` 确保渲染不被显示器刷新率限制
2. **FOV 计算**：`1.0 / tan(90 / 2 * pi / 180)` = `1.0 / tan(45°)` = `1.0`，即 90 度 FOV
3. **线程块大小**：16x16 = 256 线程/块，是 CUDA 的常用配置
4. **BGRA 格式**：使用 `GL_BGRA` 格式，因为 CUDA 内核以 `0xff000000 | (r<<16) | (g<<8) | b` 打包
5. **内存管理**：窗口大小变化时正确释放旧缓冲区再分配新缓冲区
6. **错误处理不完整**：`init_opengl()` 中窗口创建失败后未调用 `exit()`

## 扩展阅读

- CUDA-OpenGL Interop Guide: https://docs.nvidia.com/cuda/cuda-runtime-api/group__CUDART__OPENGL.html
- GLFW Documentation: https://www.glfw.org/documentation.html
- Reinhard, E. et al. *Photographic Tone Reproduction for Digital Images*. SIGGRAPH 2002
