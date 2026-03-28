# pop.cpp — 材质出栈与栈维护

> 源文件：`Ch_11_Automatic_Handling_of_Materials_in_Nested_Volumes/pop.cpp`

---

## 1. 文件概述

`pop.cpp` 实现了 `Pop` 函数，用于在光线离开体积时从材质栈中移除相应条目并恢复栈状态。根据 `Push_and_Load` 返回的 `leaving_material` 标志决定弹出一个还是两个条目。

---

## 2. 算法与数学背景

### 出栈逻辑

- **普通离开**（`leaving_material = false`）：只弹出栈顶元素
- **真正离开材质**（`leaving_material = true`）：弹出栈顶 + 搜索并移除栈中同一材质的先前实例

### 最顶层标志恢复

弹出后，需要将该材质在栈中剩余的最上层实例的 `topmost` 标志设为 `true`。

---

## 3. 完整代码与逐行注释

```cpp
void Pop(
    const bool leaving_material,             // Push_and_Load 返回的离开标志
    volume_stack_element stack[STACK_SIZE],   // 体积栈
    int &stack_pos)                          // 栈顶指针
{
    // 记录栈顶材质信息，然后弹出
    const scene_material &top = material[stack[stack_pos].material_idx];
    --stack_pos;

    // 如果正在离开材质，需要弹出两个条目
    if(leaving_material) {
        // 从栈顶向下搜索同一材质的上一个实例
        int idx;
        for (idx = stack_pos; idx >= 0; --idx)
            if (material[stack[idx].material_idx] == top)
                break;

        // 防止栈损坏（可能由 Push 中的溢出处理导致）
        if (idx >= 0)
            // 通过向下移动上方元素来删除该条目
            for (int i = idx+1; i <= stack_pos; ++i)
                stack[i-1] = stack[i];
        --stack_pos;  // 栈缩小一个位置
    }

    // 更新该材质剩余实例的 topmost 标志
    // 从栈顶向下找到第一个同材质实例，标记为 topmost
    for (int i = stack_pos; i >= 0; --i)
        if (material[stack[i].material_idx] == top) {
            stack[i].topmost = true;  // 注意：之前一定不是 topmost
            break;
        }
}
```

---

## 4. 输入与输出

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `leaving_material` | `bool` | 输入 | 由 `Push_and_Load` 确定，true 表示真正离开该材质 |
| `stack` | `volume_stack_element[]` | 输入/输出 | 体积栈数组 |
| `stack_pos` | `int&` | 输入/输出 | 栈顶位置 |

---

## 5. 关键算法深入分析

### 数组删除操作

当需要删除栈中间的元素时，通过将上方所有元素下移一位来填充空隙。这是 $O(k)$ 操作，其中 $k$ 是栈深度。

### 栈损坏保护

`if (idx >= 0)` 检查处理了一种边缘情况：如果之前 `Push_and_Load` 中发生栈溢出（覆写了栈顶），可能导致先前实例丢失。此检查防止在这种情况下执行无效删除。

### 两次弹出的原因

`leaving_material = true` 时，栈中存在两个该材质的条目：
1. `Push_and_Load` 刚压入的（栈顶，先弹出）
2. 更早之前进入该材质时压入的（需要搜索并删除）

---

## 6. 与其他文件的关系

```
types.cpp ← 栈元素结构定义
push.cpp  ← Push_and_Load 产生 leaving_material 标志
    ↓
pop.cpp   ← 本文件：使用该标志执行出栈
```

---

## 7. 在光线追踪管线中的位置

```
光线追踪循环中：
  1. 光线与表面相交
  2. Push_and_Load() → 入射/出射材质 + leaving_material
  3. 计算着色/折射
  4. Pop(leaving_material) ← 本文件
  5. 继续追踪下一段光线
```

---

## 8. 技术要点与注意事项

1. **调用顺序**：`Pop` 必须在 `Push_and_Load` 之后调用，且使用同一次 Push 返回的 `leaving_material` 值。
2. **`topmost` 恢复**：注释"must not have been topmost before"表明该实例之前的 `topmost` 标志在 `Push_and_Load` 中被清除了。
3. **性能影响**：数组删除操作涉及内存移动，但栈通常很浅，开销可忽略。

---

## 9. 扩展阅读

- **书中章节**：Chapter 11, "Automatic Handling of Materials in Nested Volumes"
