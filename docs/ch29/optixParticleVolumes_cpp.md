# optixParticleVolumes.cpp 技术文档

## 文件概述

`optixParticleVolumes.cpp` 是第 29 章粒子体渲染器的 CPU 端主程序。它负责 OptiX 上下文初始化、粒子数据文件的读取和缓冲区管理、GLFW 窗口创建、相机/光源设置、ImGui 交互界面，以及主渲染循环的驱动。

**源文件路径**: `Ch_29_Efficient_Particle_Volume_Splatting_in_a_Ray_Tracer/optixParticleVolumes/optixParticleVolumes.cpp`

## 算法与数学背景

### 粒子属性归一化

对于 `.raw` 文件，粒子的第四分量 $w$（标量属性）被归一化到 $[0, 1]$：

$$w_{\text{norm}} = w \cdot \frac{1}{w_{\max} - w_{\min}}$$

对于有负值的数据（如暗物质密度），使用带偏移的归一化：

$$w_{\text{norm}} = w \cdot \frac{0.5}{|w_{\text{extremum}}|} + 0.5$$

### 默认粒子半径

如果未指定半径，按粒子密度自动计算：

$$r = \frac{|\text{bbox}|}{N^{1/3}}$$

其中 $N$ 为粒子总数，$|\text{bbox}|$ 为包围盒对角线长度。

### Slab 间距

```cpp
float slab_spacing = PARTICLE_BUFFER_SIZE * particlesPerSlab * fixed_radius;
```

## 代码结构概览

```
optixParticleVolumes.cpp
├── 数据结构
│   ├── ParticleFrameData   -- 单帧粒子数据缓存
│   ├── ParticlesBuffer     -- OptiX 缓冲区集合
│   └── RenderBuffers       -- 渲染缓冲状态
├── 工具函数
│   ├── ptxPath()           -- PTX 文件路径构造
│   ├── parseFloat()        -- 文本解析
│   ├── get_min/get_max()   -- 向量极值
│   └── fillBuffers()       -- CPU→GPU 缓冲区传输
├── OptiX 设置
│   ├── createContext()     -- 上下文、输出缓冲、RT程序
│   ├── createMaterialPrograms() -- Any Hit 程序
│   ├── createOptiXMaterial()    -- 材质创建
│   ├── createBoundingBoxProgram/IntersectionProgram()
│   ├── setupParticles()    -- 几何、材质、加速结构
│   ├── setupCamera()       -- 相机初始化
│   └── setupLights()       -- 三点光源
├── 数据加载
│   ├── readFile()          -- 读取 .raw / .txt 粒子文件
│   ├── loadParticles()     -- 带缓存的粒子加载
│   └── setParticlesBaseName() -- 序列文件名解析
├── GLFW 回调
│   ├── keyCallback()       -- 键盘事件
│   └── windowSizeCallback() -- 窗口缩放
├── 渲染循环
│   ├── glfwRun()           -- 主渲染循环 + ImGui
│   └── updateCamera()      -- 相机矩阵更新
└── main()                  -- 命令行解析 + 启动
```

## 逐段代码详解

### createContext() -- OptiX 上下文初始化

```cpp
void createContext(int usage_report_level, UsageReportLogger* logger)
{
    context = Context::create();
    context->setRayTypeCount(2);        // radiance + shadow
    context->setEntryPointCount(1);      // 单入口

    // 输出缓冲 (PBO 或普通缓冲)
    Buffer output_buffer = sutil::createOutputBuffer(context, RT_FORMAT_UNSIGNED_BYTE4, width, height, use_pbo);

    // 累积缓冲 (GPU-local, float4)
    Buffer accum_buffer = context->createBuffer(RT_BUFFER_INPUT_OUTPUT | RT_BUFFER_GPU_LOCAL,
        RT_FORMAT_FLOAT4, width, height);

    // Ray Generation + Exception 程序
    context->setRayGenerationProgram(0,
        context->createProgramFromPTXFile(ptxPath("raygen.cu"), "raygen_program"));
    context->setExceptionProgram(0,
        context->createProgramFromPTXFile(ptxPath("raygen.cu"), "exception"));

    // Miss 程序 (恒定背景色)
    context->setMissProgram(0,
        context->createProgramFromPTXFile(ptxPath("constantbg.cu"), "miss"));
}
```

### setupParticles() -- 粒子几何设置

```cpp
void setupParticles()
{
    // 创建 4 个缓冲区 (初始大小 0, 后续动态设置)
    buffers.positions  = context->createBuffer(RT_BUFFER_INPUT, RT_FORMAT_FLOAT4, 0);
    buffers.velocities = context->createBuffer(RT_BUFFER_INPUT, RT_FORMAT_FLOAT3, 0);
    buffers.colors     = context->createBuffer(RT_BUFFER_INPUT, RT_FORMAT_FLOAT3, 0);
    buffers.radii      = context->createBuffer(RT_BUFFER_INPUT, RT_FORMAT_FLOAT,  0);

    // 创建自定义几何 (Custom Geometry)
    geometry = context->createGeometry();
    geometry->setBoundingBoxProgram(createBoundingBoxProgram(context));
    geometry->setIntersectionProgram(createIntersectionProgram(context));

    // 创建几何实例 + 几何组 + BVH 加速结构
    geometry_group = context->createGeometryGroup();
    geometry_group->addChild(geom_instance);
    Acceleration accel = context->createAcceleration("Bvh8");
    geometry_group->setAcceleration(accel);

    context["top_object"]->set(geometry_group);
}
```

### readFile() -- 粒子文件读取

支持两种格式：
1. **`.raw` 二进制**: 每粒子 16 字节 (xyz + scalar)，无速度/颜色/半径
2. **`.txt` 文本**: 每行 `x y z vx vy vz [r g b] [radius]`，速度模长作为标量属性

### glfwRun() -- 主渲染循环

```cpp
while (!glfwWindowShouldClose(window))
{
    glfwPollEvents();
    ImGui_ImplGlfwGL2_NewFrame();

    // 鼠标交互 (sutil::Camera)
    if (!io.WantCaptureMouse)
        camera.process_mouse(mouseX, mouseY, ...);

    // 自动旋转
    if (camera_slow_rotate) camera.rotate(1.f, 0.f);

    // ImGui 参数面板
    ImGui::SliderFloat("radius", &fixed_radius, ...);
    ImGui::SliderFloat("attribute scale", &wScale, ...);
    ImGui::SliderFloat("sample opacity", &opacity, ...);
    ImGui::SliderInt("transfer function preset", &tf_type, ...);

    // 发射光线
    context["frame"]->setUint(accumulation_frame++);
    context->launch(0, camera.width(), camera.height());

    // 显示结果
    sutil::displayBufferGL(getOutputBuffer());
    ImGui::Render();
    glfwSwapBuffers(window);
}
```

## 关键算法深入分析

### 帧数据缓存

使用 `std::map<int, ParticleFrameData>` 缓存已加载的帧数据，支持动画序列的快速切换而无需重新解析文件。

### BVH 更新策略

当粒子半径改变时，标记加速结构为脏 (dirty) 并启用 refit：
```cpp
accel->setProperty("refit", "1");
accel->markDirty();
```

Refit 比完全重建更快，但仅适用于包围盒微小变化的情况。

## 输入与输出

**输入**:
- 粒子数据文件 (.raw 或 .txt)
- 命令行参数 (半径、透明度、传递函数等)

**输出**:
- GLFW 窗口中的实时渲染画面
- 可选的 PNG 截图 (按 S 键)

## 与其他文件的关系

- 通过 PTX 路径引用全部 `.cu` GPU 程序
- 使用 `sutil` 库进行窗口管理、缓冲区显示、相机控制
- 包含 `commonStructs.h` 共享数据结构

## 在光线追踪管线中的位置

CPU 端的主控制器，负责：数据准备 → OptiX 上下文配置 → 渲染循环驱动 → 结果显示。

## 技术要点与注意事项

1. `RT_BUFFER_GPU_LOCAL` 标志使累积缓冲仅存在于 GPU 内存
2. 默认数据文件为 `darksky_1M.xyz`（100 万粒子的暗物质模拟数据）
3. 三点光源：两个弱光 + 一个主光（带阴影），位置按场景大小缩放
4. 当命令行指定 `-f` 输出文件时，渲染单帧并退出（无窗口模式）
5. 粒子属性 $w$ 的归一化方式取决于数据是否含负值

## 扩展阅读

- NVIDIA OptiX Programming Guide
- Ray Tracing Gems, Chapter 29: "Efficient Particle Volume Splatting in a Ray Tracer"
- "Deep Image Compositing" (Hillman, SIGGRAPH 2012)
