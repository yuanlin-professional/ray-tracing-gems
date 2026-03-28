# types.cpp — 体积栈数据结构定义

> 源文件：`Ch_11_Automatic_Handling_of_Materials_in_Nested_Volumes/types.cpp`

---

## 1. 文件概述

`types.cpp` 定义了嵌套体积材质追踪所需的数据结构，包括栈元素结构和全局材质数组。

---

## 2. 完整代码与逐行注释

```cpp
struct volume_stack_element
{
    bool topmost : 1,       // 是否为该材质在栈中的最上层实例（1 位）
         odd_parity : 1;    // 该材质出现次数是否为奇数（1 位）
    int  material_idx : 30; // 材质索引（30 位，支持约 10 亿种材质）
};

scene_material *material;   // 全局材质数组指针
```

---

## 3. 数据结构详解

### `volume_stack_element`

| 字段 | 类型 | 位宽 | 说明 |
|------|------|------|------|
| `topmost` | `bool` | 1 位 | 标记此实例是否为该材质在栈中最上层（最后进入）的实例 |
| `odd_parity` | `bool` | 1 位 | 标记该材质在当前栈位置以下出现的次数奇偶性 |
| `material_idx` | `int` | 30 位 | 指向全局 `material` 数组的索引 |

### 位域设计

总共使用 32 位（4 字节），通过位域 (bit field) 紧凑存储，最小化每个栈元素的内存占用。在 GPU 上，较小的数据结构有助于减少寄存器和局部内存压力。

### `topmost` 标志的作用

在嵌套体积中，同一种材质可能多次出现在栈中（如光线从玻璃 A 进入水，再进入另一个玻璃 A）。`topmost` 标志追踪每种材质最新进入的实例，用于在 `Push_and_Load` 中快速找到当前有效的外层材质。

### `odd_parity` 标志的作用

光线穿过材质的表面时，奇数次穿越表示"进入"，偶数次表示"离开"。此标志用于判断当前是进入还是离开材质。

---

## 4. 与其他文件的关系

```
types.cpp ← 本文件：定义栈元素结构
    ↑
push.cpp ← 使用栈元素进行入栈操作
pop.cpp  ← 使用栈元素进行出栈操作
```

---

## 5. 扩展阅读

- **书中章节**：Chapter 11, "Automatic Handling of Materials in Nested Volumes"
