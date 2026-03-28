# raygen.cu 技术文档

## 文件概述

`raygen.cu` 实现了 OptiX 的 Ray Generation 程序和 Exception 程序。这是粒子体渲染算法的核心，包含射线生成、场景 AABB 求交、Slab 分段遍历、粒子排序和 RBF 体积积分的完整流程。

**源文件路径**: `Ch_29_Efficient_Particle_Volume_Splatting_in_a_Ray_Tracer/optixParticleVolumes/raygen.cu`

## 算法与数学背景

### Slab-based 分段遍历

将光线在场景 AABB 内的行进区间 $[t_{\text{enter}}, t_{\text{exit}}]$ 分割为等间距的 Slab：

$$\text{slab\_spacing} = \text{PARTICLE\_BUFFER\_SIZE} \times \text{particlesPerSlab} \times r$$

每个 Slab 的参数范围为 $[t_{\text{slab}}, t_{\text{slab}} + \text{slab\_spacing}]$。

### Ray-AABB 求交

使用 Slab Method (平板法)：

$$t_0 = \frac{\text{bbox\_max} - \mathbf{o}}{\mathbf{d}}, \quad t_1 = \frac{\text{bbox\_min} - \mathbf{o}}{\mathbf{d}}$$

$$t_{\text{enter}} = \max(0, \max(t_{\min,x}, t_{\min,y}, t_{\min,z}))$$
$$t_{\text{exit}} = \min(t_{\max,x}, t_{\max,y}, t_{\max,z})$$

### RBF 评估

对于粒子中心 $\mathbf{c}$，射线上采样点 $\mathbf{p}$：

$$d = \frac{2 \cdot |\mathbf{c} - \mathbf{p}|}{r}$$

$$v = \text{clamp}\left(w_{\text{scale}} \cdot w \cdot e^{-d^2}, 0, 1\right)$$

其中 $w$ 为粒子的标量属性，$w_{\text{scale}}$ 为用户控制的缩放因子。

### 前-后 Alpha 合成

$$\alpha_{\text{new}} = \alpha_{\text{sample}} \cdot (1 - \alpha_{\text{accum}})$$
$$C_{\text{accum}} \mathrel{+}= C_{\text{sample}} \cdot \alpha_{\text{new}}$$
$$\alpha_{\text{accum}} \mathrel{+}= \alpha_{\text{new}}$$

当 $\alpha_{\text{accum}} \geq 0.97$ 时提前终止（Early Ray Termination）。

### 排序算法

根据 `PARTICLE_BUFFER_SIZE` 选择排序算法：
- $\leq 64$：冒泡排序 (Bubble Sort) -- 简单，对小数组高效
- $> 64$：双调排序 (Bitonic Sort) -- $O(n\log^2 n)$，对 GPU 更友好

## 代码结构概览

```
raygen.cu
├── 变量声明 (eye, U, V, W, buffers, ...)
├── raygen_program()
│   ├── 射线生成 (pinhole camera)
│   ├── Ray-AABB 求交
│   ├── Slab 循环
│   │   ├── rtTrace() -- 收集粒子
│   │   ├── 排序 (bubble/bitonic)
│   │   └── RBF 积分 + alpha 合成
│   └── 写入输出缓冲
└── exception()
```

## 逐段代码详解

### 射线生成

```cuda
RT_PROGRAM void raygen_program()
{
    size_t2 screen = output_buffer.size();
    unsigned int seed = tea<16>(screen.x*launch_index.y+launch_index.x, frame);

    // 针孔相机射线
    float2 d = (make_float2(launch_index) + make_float2(0,0)) / make_float2(screen) * 2.f - 1.f;
    float3 ray_origin = eye;
    float3 ray_direction = normalize(d.x*U + d.y*V + W);
```

### Ray-AABB 求交

```cuda
    float3 t0, t1, tmin, tmax;
    t0 = (bbox_max - ray_origin) / ray_direction;
    t1 = (bbox_min - ray_origin) / ray_direction;
    tmax = fmaxf(t0, t1);
    tmin = fminf(t0, t1);
    float tenter = fmaxf(0.f, fmaxf(tmin.x, fmaxf(tmin.y, tmin.z)));
    float texit = fminf(tmax.x, fminf(tmax.y, tmax.z));
```

### Slab 遍历循环

```cuda
    if (tenter < texit)
    {
        float tbuffer = 0.f;

        while(tbuffer < texit && result_alpha < 0.97f)
        {
            prd.tail = 0;
            ray.tmin = fmaxf(tenter, tbuffer);
            ray.tmax = fminf(texit, tbuffer + slab_spacing);

            if (ray.tmax > tenter)  // 保持光线连贯性
            {
                rtTrace(top_object, ray, prd);  // 收集粒子

                // === 排序 ===
                int N = prd.tail;
                // 冒泡排序 (PARTICLE_BUFFER_SIZE <= 64)
                for(int i=0; i<N; i++)
                    for(int j=0; j < N-i-1; j++)
                    {
                        const float2 tmp = prd.particles[i];
                        if(tmp.x < prd.particles[j].x) {
                            prd.particles[i] = prd.particles[j];
                            prd.particles[j] = tmp;
                        }
                    }

                // === RBF 积分 ===
                const float inv_radius_scale = 2.f / fixed_radius;
                for(int i=0; i<prd.tail; i++) {
                    float trbf = prd.particles[i].x;
                    int idx = __float_as_int(prd.particles[i].y);
                    float3 hit = ray.origin + ray.direction * trbf;

                    float4 pos = positions_buffer[idx];
                    float3 offset = make_float3(pos.x, pos.y, pos.z) - hit;
                    float drbf = length(offset) * inv_radius_scale;
                    drbf = fmaxf(0.f, fminf(1.f, wScale * pos.w * exp(-drbf*drbf)));

                    float4 color = tf(drbf, trbf * redshiftScale, tf_type);
                    float alpha = color.w * opacity;
                    float alpha_1msa = alpha * (1.0 - result_alpha);
                    result += make_float3(color) * alpha_1msa;
                    result_alpha += alpha_1msa;
                }
            }
            tbuffer += slab_spacing;
        }
    }
```

### 写入帧缓冲

```cuda
    float4 acc_val = make_float4(result, 0.f);
    output_buffer[launch_index] = make_color(make_float3(acc_val));
    accum_buffer[launch_index] = acc_val;
}
```

## 关键算法深入分析

### 光线连贯性优化

```cuda
if (ray.tmax > tenter)  // 仅当 Slab 在 AABB 内时发射
```

这个条件确保了在光线尚未进入 AABB 时不进行无效的 BVH 遍历，但仍然推进 `tbuffer`。这使得相邻像素的光线在同一时间处理相同的 Slab，提高 GPU warp 的执行效率。

### 红移 (Redshift) 效果

对于天体物理可视化，粒子的距离可以映射为红移效应：

$$\text{redshiftScale} = \frac{\text{redshift}}{|\text{bbox}|}$$

传递函数 `tf()` 接收此参数来调制颜色。

### 早期终止

当累积不透明度 $\alpha \geq 0.97$ 时退出循环，避免对已基本不透明的像素继续计算。

## 输入与输出

**输入**: 相机参数、场景 AABB、粒子缓冲区、渲染参数

**输出**: `output_buffer` (uchar4 BGRA) + `accum_buffer` (float4)

## 与其他文件的关系

- 调用 `rtTrace()` 触发 `geometry.cu` 和 `material.cu`
- 使用 `transferFunction.h::tf()` 进行标量-颜色映射
- 使用 `helpers.h::make_color()` 进行颜色格式转换
- 使用 `random.h::tea()` 生成伪随机种子

## 在光线追踪管线中的位置

管线入口：每个像素执行一次，驱动整个渲染流程。

## 技术要点与注意事项

1. 双调排序需要将数组长度补齐到 2 的幂，超出部分填充 $t = 10^{20}$
2. 冒泡排序虽然理论复杂度差，但对 GPU 小数组（32元素）实际性能优异
3. `__float_as_int` / `__int_as_float` 是零开销的位重解释
4. 子像素抖动 (jitter) 设为 `(0, 0)`，即无抗锯齿；可启用随机抖动提高质量
5. 当 `play == true` 时支持动画播放（帧序列切换）

## 扩展阅读

- "Volume Rendering" (Levoy, 1988)
- "Deep Image Compositing" (Hillman, SIGGRAPH)
- "Bitonic Sort on GPU" (Batcher, 1968; Nvidia CUDA Samples)
