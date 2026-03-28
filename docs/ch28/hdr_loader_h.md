# hdr_loader.h 技术文档

## 文件概述

**文件路径**: `Ch_28_Ray_Tracing_Inhomogeneous_Volumes/hdr_loader.h`

本文件实现了 **Radiance HDR (.hdr) 图像格式** 的加载器，用于读取高动态范围 (High Dynamic Range, HDR) 环境贴图。HDR 环境贴图在体积路径追踪中作为光源使用——提供来自各个方向的自然光照。该文件支持 Radiance 的 **RGBE 编码** 和 **行程长度编码 (Run-Length Encoding, RLE)** 压缩格式。

## 算法与数学背景

### RGBE 编码格式

Greg Ward 设计的 RGBE 格式使用 4 个字节（32 位）存储一个 HDR 像素：

$$\text{color}_c = (E_c + 0.5) \cdot 2^{e - 128 - 8}, \quad c \in \{R, G, B\}$$

其中：
- $E_c \in [0, 255]$：各通道的尾数 (Mantissa)
- $e \in [0, 255]$：共享指数 (Shared Exponent)

解码公式的变体（本代码实现）：

$$\text{color}_c = (E_c + 0.5) \cdot 2^{(e - 9) \cdot 2^{23} \text{ (as float exponent)}}$$

通过直接操作 IEEE 754 浮点数的指数位来高效计算。

### 行程长度编码 (RLE)

Radiance 的 HDR 文件使用特殊的 RLE 格式压缩扫描线。每个扫描线的 R、G、B、E 四个通道分别独立压缩：

- **游程 (Run)**：字节值 > 128 表示后续的一个字节重复 $(n - 128)$ 次
- **非游程 (Non-Run)**：字节值 <= 128 表示后续的 $n$ 个字节是原始数据

## 代码结构概览

```
hdr_loader.h
├── find_identifier()              // 头部标识符匹配
├── struct hdr_image_header_t      // 图像头部信息
├── enum hdr_error_code_t          // 错误代码
├── hdr_read_image_header()        // 读取图像头部
├── hdr_read_image_scanline()      // 读取未编码扫描线
├── hdr_read_image_scanline_rle()  // 读取 RLE 编码扫描线
├── hdr_rgbe_to_color()            // RGBE → float RGB 转换
├── hdr_read_image_pixels_float()  // 读取完整图像为浮点数组
└── load_hdr_float4()              // 顶层加载接口
```

## 逐段代码详解

### 头部信息结构

```cpp
typedef struct {
    unsigned int rx, ry; /* resolution */
    float gamma;
    float exposure;
    int xyz; /* color space is XYZ or RGB? */
    char comment[128];
} hdr_image_header_t;
```

HDR 文件头部包含：
- 分辨率 (`rx`, `ry`)
- Gamma 校正值和曝光度
- 颜色空间标识（XYZ 或 RGB）

### 错误代码枚举

```cpp
typedef enum {
    HDR_SUCCESS = 0,
    HDR_ERROR_INVALID_HEADER = 1,
    HDR_ERROR_READING_HEADER = 2,
    HDR_ERROR_READING_PIXELS = 3,
    HDR_ERROR_ALLOCATION_FAILURE = 5,
    HDR_ERROR_WRITING_HEADER = 6,
    HDR_ERROR_WRITING_PIXELS = 7,
    HDR_ERROR_NUM_CODES = 9
} hdr_error_code_t;
```

### hdr_read_image_header() - 头部解析

```cpp
static hdr_error_code_t hdr_read_image_header(
    hdr_image_header_t * const header, FILE * const fp)
{
    rewind(fp);
    char linebuf[LINE_SIZE];

    const char *line = fgets(linebuf, LINE_SIZE, fp);
    if (!line) return HDR_ERROR_INVALID_HEADER;

    if (line[0] != '#' && line[1] != '?')
        return HDR_ERROR_INVALID_HEADER;
```

HDR 文件以 `#?` 开头（通常为 `#?RADIANCE` 或 `#?RGBE`）。

```cpp
    while (1) {
        line = fgets(linebuf, LINE_SIZE, fp);
        if (line[0] == '#') continue;  // 跳过注释

        const char *c;
        if ((c = find_identifier(line, "EXPOSURE=")))
            // 解析曝光度
        else if ((c = find_identifier(line, "GAMMA=")))
            // 解析 Gamma 值
        else if ((c = find_identifier(line, "FORMAT=")))
            // 解析格式（32-bit_rle_rgbe 或 32-bit_rle_xyze）
        else if (line[0] == '-' || line[0] == '+')
            // 解析分辨率行（如 "-Y 1024 +X 2048"）
    }
}
```

头部解析按行逐个字段处理，支持 EXPOSURE、GAMMA、FORMAT 等标准字段。

### hdr_read_image_scanline_rle() - RLE 扫描线读取

```cpp
static hdr_error_code_t hdr_read_image_scanline_rle(
    unsigned char *const rgbe_buf,
    const unsigned int len,
    FILE *const fp)
{
    int c = fgetc(fp);
    if (c != 2) {  /* not really rle? */
        ungetc(c, fp);
        return hdr_read_image_scanline(rgbe_buf, len, fp);
    }
```

RLE 扫描线以字节 `2` 开头。如果不是，回退到未编码读取。

```cpp
    // 验证 RLE 标记
    rgbe_buf[1] = fgetc(fp); /* green */
    rgbe_buf[2] = fgetc(fp); /* blue */
    c = fgetc(fp);

    if (rgbe_buf[1] != 2 || (rgbe_buf[2] & 128)) { /* not rle? */
        // 前4个字节是普通像素数据，读取剩余部分
        ...
    }
```

完整的 RLE 标记验证：第二个字节也必须是 2，且第三个字节的最高位必须为 0。

```cpp
    for (unsigned int i = 0; i < 4; ++i) /* 4 个通道 (R,G,B,E) */
    {
        for (unsigned pos = 0; pos < len;) /* 扫描线内位置 */
        {
            int num = fgetc(fp);
            if (num > 128) /* run start? */
            {
                num &= 127;
                const int val = fgetc(fp);
                for (int j = 0; j < num; ++j) {
                    rgbe_buf[pos * 4 + i] = val;
                    ++pos;
                }
            }
            else /* non-run */
            {
                for (int j = 0; j < num; ++j) {
                    rgbe_buf[pos * 4 + i] = fgetc(fp);
                    ++pos;
                }
            }
        }
    }
```

RLE 解码的核心循环：
- 外层循环：4 个通道分别处理（平面存储格式）
- 内层循环：逐游程/非游程解码
- 游程标记：字节值 > 128 表示游程，长度 = 值 & 127

### hdr_rgbe_to_color() - RGBE 到浮点转换

```cpp
static void hdr_rgbe_to_color(
    float rgb[3],
    const unsigned char *const rgbe)
{
    if (rgbe[3] == 0)
        rgb[0] = rgb[1] = rgb[2] = 0.0f;
    else
    {
        union {
            float f;
            unsigned int i;
        } u;

        u.i = (((int)rgbe[3] - 9) << 23) & 0x7f800000;

        rgb[0] = (float)(rgbe[0] + 0.5f) * u.f;
        rgb[1] = (float)(rgbe[1] + 0.5f) * u.f;
        rgb[2] = (float)(rgbe[2] + 0.5f) * u.f;
    }
}
```

高效的 RGBE 解码：
- 指数为 0 时（`rgbe[3] == 0`）：颜色为黑色
- 通过 `union` 直接操作 IEEE 754 浮点数的指数位字段
- `(rgbe[3] - 9) << 23`：将 RGBE 指数转换为 IEEE 754 指数
  - RGBE 的指数偏移为 128 + 8 = 136，IEEE 754 偏移为 127
  - 差值为 136 - 127 = 9
- `& 0x7f800000`：掩码保留指数位，清除符号和尾数
- 最终结果等价于 $2^{e-136}$
- 乘以 $(E_c + 0.5)$ 得到浮点颜色值

### load_hdr_float4() - 顶层加载接口

```cpp
static bool load_hdr_float4(float **pixels, unsigned int *rx,
    unsigned int *ry, const char *filename)
{
    FILE *fp = fopen(filename, "rb");
    // 读取头部
    hdr_read_image_header(&header, fp);
    // 分配 float4 缓冲区
    *pixels = (float *)calloc(header.rx * header.ry, sizeof(float) * 4);
    // 读取像素数据
    hdr_read_image_pixels_float(&header, *pixels, fp, true);
    // 返回分辨率
    *rx = header.rx;
    *ry = header.ry;
    return true;
}
```

外部接口，以 RGBA（`float4`）格式加载完整 HDR 图像。

## 关键算法深入分析

### IEEE 754 位操作技巧

```
RGBE 字节 [R, G, B, E] 的解码:

E = rgbe[3]（共享指数）

IEEE 754 float 位布局:
┌──┬────────┬───────────────────────┐
│S │EEEEEEEE│MMMMMMMMMMMMMMMMMMMMMMM│
│1 │  8 位   │       23 位           │
└──┴────────┴───────────────────────┘

操作: ((E - 9) << 23) & 0x7f800000
  → 构造一个纯指数的浮点数 = 2^(E-9-127) = 2^(E-136)

最终: color_c = (rgbe[c] + 0.5) * 2^(E-136)
```

这种位操作技巧避免了 `pow()` 或 `ldexp()` 函数调用，在循环中非常高效。

### RLE 格式的平面存储

与传统的 RGBE 像素交错存储不同，RLE 格式将每个通道分别存储：

```
传统: R0 G0 B0 E0 R1 G1 B1 E1 R2 G2 B2 E2 ...
RLE:  R0 R1 R2 ... G0 G1 G2 ... B0 B1 B2 ... E0 E1 E2 ...
```

平面存储使每个通道内的相邻值更相似，提高了 RLE 压缩效率。

## 输入与输出

### load_hdr_float4 输入
| 名称 | 类型 | 说明 |
|------|------|------|
| `filename` | `const char*` | HDR 文件路径 |

### load_hdr_float4 输出
| 名称 | 类型 | 说明 |
|------|------|------|
| `pixels` | `float**` | RGBA float 像素数据指针 |
| `rx` | `unsigned int*` | 图像宽度 |
| `ry` | `unsigned int*` | 图像高度 |
| 返回值 | `bool` | 加载是否成功 |

## 与其他文件的关系

- **`main.cpp`**：调用 `load_hdr_float4()` 加载环境贴图，并上传到 CUDA 纹理
- **`volume_kernel.cu`**：使用加载的环境贴图作为背景光源

## 在光线追踪管线中的位置

```
HDR 文件 (.hdr) → [hdr_loader.h] → float4 像素数据 → CUDA 纹理对象 → GPU 环境采样
                  ^^^^^^^^^^^^^^^^^
                  本文件实现此步骤
```

该加载器是离线预处理步骤，在程序启动时执行一次。

## 技术要点与注意事项

1. **文件格式兼容性**：支持标准 Radiance `.hdr` 格式，包括 RLE 和非 RLE 变体
2. **颜色空间**：支持 RGB 和 XYZ 颜色空间标识，但代码中未做 XYZ→RGB 的转换
3. **内存分配**：使用 `calloc` 分配并清零缓冲区，`float4` 格式的第四通道（Alpha）默认为 0
4. **RLE 边界情况**：宽度小于 8 或大于 0x7fff 的扫描线不使用 RLE
5. **错误处理**：函数返回错误代码但不打印诊断信息，调用者需检查返回值
6. **内存泄漏风险**：`hdr_read_image_pixels_float()` 内部分配的 `rgbe_buf` 在错误路径中正确释放

## 扩展阅读

- Ward, G. *Real Pixels*. Graphics Gems II, Chapter 15, 1991
- Ward, G. *The RADIANCE Lighting Simulation and Rendering System*. SIGGRAPH 1994
- Radiance File Format Specification: https://radsite.lbl.gov/radiance/refer/filefmts.pdf
