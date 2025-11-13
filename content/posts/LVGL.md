---
title: "LVGL"
date: 2025-11-12T21:41:11-08:00
---

# LVGL

#### 1.防止样式重复初始化方式

多次调用 `lv_style_init()` 会重复分配内存导致内存泄漏

通过检查样式对象的属性计数器 `prop_cnt` 决定操作：

- 若 `prop_cnt > 1`，说明样式已初始化并设置过属性，调用 `lv_style_reset()` 清除属性（保留初始化状态）。
- 否则，调用 `lv_style_init()` 完成首次初始化。

```c
void ui_init_style(lv_style_t *style)

{

  if (style->prop_cnt > 1)
  {
      lv_style_reset(style);
  }
  else
  {
      lv_style_init(style);
  }
}
```



#### 2.lv_obj_is_valid()失效原因

1. **`lv_obj_is_valid()` 的工作原理**
   - 该函数通过检查对象头部的 "magic number"（一个特殊标记）来判断对象是否有效。
   - 当对象被删除时，LVGL 会清除这个 magic number，理论上使 `lv_obj_is_valid()` 返回 `false`。
2. **失效原因**
   - 如果内存被重新分配给新对象，而新对象的 magic number 与旧对象相同，就会导致误判。
   - 在内存碎片化严重的情况下，这种冲突更容易发生。
3. **设计限制**
   - LVGL 的对象验证机制依赖于内存管理的一致性。如果内存被重复使用且未正确初始化，验证就可能失效。

解决方案：在删除对象后立即将指针置为 `NULL`，并在访问前同时检查指针是否为 `NULL` 和 `lv_obj_is_valid()` 的返回值