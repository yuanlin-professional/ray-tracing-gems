# push.cpp — 材质入栈与材质确定

> 源文件：`Ch_11_Automatic_Handling_of_Materials_in_Nested_Volumes/push.cpp`

---

## 1. 文件概述

`push.cpp` 实现了 `Push_and_Load` 函数，这是嵌套体积处理的核心操作。当光线穿越表面时，此函数将新材质压入栈，并确定当前界面的入射和出射材质。

---

## 2. 算法与数学背景

### 嵌套体积的进入/离开判定

光线穿越体积表面时，需要判断是"进入"还是"离开"该体积：
- **奇数次穿越**（`odd_parity = true`）：进入材质
- **偶数次穿越**（`odd_parity = false`）：离开材质

### 材质确定规则

| 情况 | 入射材质 | 出射材质 |
|------|---------|---------|
| 进入 (`odd_parity`) | 栈中最顶层的外层材质 | 新压入的材质 |
| 离开且非重叠区 | 被弹出的材质 | 栈中最顶层的外层材质 |
| 离开重叠区 | 出射材质（跳过此界面） | 栈中最顶层的外层材质 |

---

## 3. 代码结构概览

### 函数签名

```cpp
void Push_and_Load(
    int &incident_material_idx,    // [输出] 入射侧材质索引
    int &outgoing_material_idx,    // [输出] 出射侧材质索引
    bool &leaving_material,        // [输出] 是否正在离开材质
    const int material_idx,        // [输入] 被穿越几何体的材质索引
    volume_stack_element stack[STACK_SIZE],  // [输入/输出] 体积栈
    int &stack_pos)                // [输入/输出] 栈顶指针
```

---

## 4. 逐段代码详解

### 阶段 1：搜索先前实例（第 9-19 行）

```cpp
bool odd_parity = true;
int prev_same;
for (prev_same = stack_pos; prev_same >= 0; --prev_same)
    if (material[material_idx] == material[stack[prev_same].material_idx]) {
        stack[prev_same].topmost = false;
        odd_parity = !stack[prev_same].odd_parity;
        break;
    }
```

- 从栈顶向下搜索同一材质的先前实例
- 若找到，将其 `topmost` 标志清除（新实例将成为最新的）
- 奇偶性取反：若先前为奇数次，现在为偶数次（离开），反之亦然
- 若未找到（`prev_same < 0`），`odd_parity` 保持 `true`（首次进入）

### 阶段 2：查找当前外层材质（第 22-28 行）

```cpp
int idx;
for (idx = stack_pos; idx >= 0; --idx)
    if ((material[stack[idx].material_idx] != material[material_idx]) &&
        (stack[idx].odd_parity && stack[idx].topmost))
        break;
```

- 从栈顶向下搜索满足三个条件的栈元素：
  1. 与当前材质不同
  2. 出现奇数次（`odd_parity` 为 true）
  3. 是该材质的最上层实例（`topmost` 为 true）
- 这找到的是光线当前实际所处的"外层"材质
- 注释提到 `idx` 总是 >= 0，因为相机始终处于某个体积中

### 阶段 3：压入新材质（第 30-35 行）

```cpp
if (stack_pos < STACK_SIZE - 1)
    ++stack_pos;
stack[stack_pos].material_idx = material_idx;
stack[stack_pos].odd_parity = odd_parity;
stack[stack_pos].topmost = true;
```

- 栈溢出保护：若栈满则不递增指针（覆写栈顶，可能丢失信息）
- 新元素设为最上层实例

### 阶段 4：确定入射/出射材质（第 37-52 行）

```cpp
if (odd_parity) { // 进入材质
    incident_material_idx = stack[idx].material_idx;
    outgoing_material_idx = material_idx;
} else { // 离开材质
    outgoing_material_idx = stack[idx].material_idx;
    if(idx < prev_same)
        incident_material_idx = material_idx;
    else
        incident_material_idx = outgoing_material_idx;
}
leaving_material = !odd_parity;
```

- **进入时**：入射侧是外层材质，出射侧是新材质
- **离开时**：
  - 若外层材质在先前同材质实例之下（`idx < prev_same`），说明不在重叠区，正常设置
  - 否则在重叠区，设置入射=出射（标记此界面应被跳过）

---

## 5. 关键算法深入分析

### 时间复杂度

$O(k)$ — 其中 $k$ 是栈深度（嵌套体积层数），通常很小（< 10）。

### 重叠处理

当两个相同材质的体积重叠时，重叠区内的界面应被忽略（因为两侧材质相同）。通过设置 `incident_material_idx = outgoing_material_idx` 来标记这种情况。

### 栈溢出处理

当嵌套层数超过 `STACK_SIZE` 时，最底层的材质信息会丢失。`Pop` 函数中的"保护"逻辑处理了这种退化情况。

---

## 6. 输入与输出

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `incident_material_idx` | `int&` | 输出 | 光线来自的材质索引 |
| `outgoing_material_idx` | `int&` | 输出 | 光线进入的材质索引 |
| `leaving_material` | `bool&` | 输出 | 是否正在离开材质（供 Pop 使用） |
| `material_idx` | `int` | 输入 | 被穿越几何体的材质索引 |
| `stack` | `volume_stack_element[]` | 输入/输出 | 体积栈数组 |
| `stack_pos` | `int&` | 输入/输出 | 栈顶位置 |

---

## 7. 与其他文件的关系

```
types.cpp ← 栈元素结构定义
    ↓
push.cpp  ← 本文件：进入表面时调用
    ↓
pop.cpp   ← 离开表面后调用（使用 leaving_material 参数）
```

---

## 8. 在光线追踪管线中的位置

```
光线与表面相交
       ↓
  Push_and_Load()  ← 确定入射/出射材质
       ↓
  使用入射/出射材质计算 Fresnel、折射方向等
       ↓
  生成反射/折射光线
       ↓
  [折射光线继续追踪...]
       ↓
  Pop()  ← 离开该体积时恢复栈状态
```

---

## 9. 技术要点与注意事项

1. **材质比较**：使用 `material[a] == material[b]` 而非索引比较，允许不同索引引用相同材质。
2. **相机体积**：假设栈初始化时包含相机所在的体积材质（通常为空气/真空）。
3. **GPU 实现考虑**：线性搜索在栈深度很小时可接受，但极深嵌套可能影响性能。

---

## 10. 扩展阅读

- **书中章节**：Chapter 11, "Automatic Handling of Materials in Nested Volumes"
- **Schmidt, C. and Budge, B.**, "Simple Nested Dielectrics in Ray Traced Images," *JGT*, 7(2), 2002.
