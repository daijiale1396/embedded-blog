---
title: "LVGL 实战：基于复用机制实现无限滚动列表"
date: 2025-11-12T21:41:11-08:00
---

# LVGL 实战：基于复用机制实现无限滚动列表

在嵌入式 GUI 开发中，内存资源往往比较宝贵。当需要展示大量数据（如 30 条以上）时，直接创建对应数量的 UI 元素会导致内存占用过高，甚至引发系统卡顿或崩溃。本文将介绍如何使用 LVGL 的虚拟列表复用机制，仅通过少量子对象实现无限滚动效果，适用于智能手表、小型物联网设备等资源受限场景。

## 一、核心功能与应用场景

本次实现的无限滚动列表具有以下特点：

- 仅使用 6 个复用的文本标签（`ITEM_COUNT=6`），即可展示 30 条数据（可扩展为无限循环）
- 支持垂直滚动，滚动时动态更新内容，视觉上实现无缝衔接
- 关闭弹性滚动效果，避免边缘反弹干扰
- 每条数据高度 22px，间距 25px，适配 240×216 分辨率的显示区域

适用于需要展示日志、列表项、状态信息等场景，尤其适合 RAM 不足 128KB 的嵌入式设备。

## 二、实现原理：虚拟列表与对象复用

传统列表会为每条数据创建一个独立的 UI 对象，当数据量超过 10 条时就可能占用大量内存。而**虚拟列表**通过以下机制优化：

1. **对象复用**：只创建可见区域 + 1 的子对象（本例 6 个），滚动时重复使用这些对象
2. **动态更新**：根据滚动偏移计算当前应显示的数据索引，更新复用对象的位置和内容
3. **无限循环**：通过取模运算（`data_idx % 30`）实现数据循环展示，突破数据量限制

核心思路：用时间换空间，通过滚动事件触发的少量计算，换取内存占用的大幅降低。

## 三、完整代码解析

### 1. 宏定义与全局变量

```c
// 复用对象数量（可见区域能显示的最大数量）
#define ITEM_COUNT 6       
// 单条数据高度（像素）
#define ITEM_HEIGHT 22     
// 总数据量（实际可通过取模实现无限）
#define TOTAL_DATA 30      
// 子对象间距（像素）
#define ITEM_INTERVAL 25    
// 单条数据总占用高度（高度+间距）
#define ITEM_TOTAL_HEIGHT (ITEM_HEIGHT + ITEM_INTERVAL)

// 存储复用的子对象
static lv_obj_t *item_list[ITEM_COUNT];  
// 模拟30条数据
static const char *data[30] = {
    "1", "2", "3", "4", "5", "6", "7", "8", "9", "10",
    "11", "12", "13", "14", "15", "16", "17", "18", "19", "20",
    "21", "22", "23", "24", "25", "26", "27", "28", "29", "30"
};

// 主容器对象
lv_obj_t *widget = NULL;
```

- `ITEM_TOTAL_HEIGHT` 是关键计算量，用于确定每条数据的位置间隔
- 数据数组`data`可替换为实际业务数据（如传感器日志、设备列表等）

### 2. 核心函数：更新可见项（`update_visible_items`）

```c
// 更新可见区域的子对象内容和位置
static void update_visible_items(void) {
    // 获取当前滚动偏移量（Y方向）
    lv_coord_t scroll_y = lv_obj_get_scroll_y(widget);
    int32_t first_visible = 0;  // 第一条可见数据的索引

    // 计算first_visible（核心逻辑）
    if (scroll_y < ITEM_HEIGHT) {
        // 滚动偏移小于单条高度时，显示第0条
        first_visible = 0;
    } else {
        // 偏移超过单条高度时，通过总高度计算索引
        first_visible = ((scroll_y - ITEM_HEIGHT) / ITEM_TOTAL_HEIGHT) + 1;
    }

#if 0  // 无限循环写法
    for(int i = 0; i < ITEM_COUNT; i++) {
        int32_t data_idx = first_visible + i;  // 滑动窗口的第 i  ~ i+4 数据索引，永远显示5个   
        lv_obj_set_y(item_list[i], data_idx * ITEM_TOTAL_HEIGHT);
        lv_label_set_text(item_list[i], data[data_idx % 30]);  // 取模则为无限循环
        lv_obj_clear_flag(item_list[i], LV_OBJ_FLAG_HIDDEN);
    }
#endif

#if 1  // 有限循环写法
    for(int i = 0; i < ITEM_COUNT; i++) {
        int32_t data_idx = first_visible + i;  // 滑动窗口的第 i  ~ i+4 数据索引，永远显示5个
        if(data_idx >= TOTAL_DATA) break; 
        lv_obj_set_y(item_list[i], data_idx * ITEM_TOTAL_HEIGHT);
        lv_label_set_text(item_list[i], data[data_idx]);
        lv_obj_clear_flag(item_list[i], LV_OBJ_FLAG_HIDDEN);
    }
#endif
}
```

**关键计算解析**：

- `scroll_y`：当前滚动的像素偏移（顶部已滚动出视野的距离）
- `first_visible`：通过偏移量计算当前应该显示的第一条数据索引，确保滚动时内容 “跟随” 移动
- 取模运算`data_idx % 30`：实现数据循环，当`data_idx`超过 29 时自动回到 0

### 3. 滚动事件回调

```c
static void scroll_event_cb(lv_event_t *e) {
    lv_event_code_t code = lv_event_get_code(e);
    // 滚动时触发更新
    if(code == LV_EVENT_SCROLL) {
        update_visible_items();
    }
}
```

通过监听`LV_EVENT_SCROLL`事件，在用户滚动列表时实时更新复用对象的内容，确保显示正确的数据。

### 4. 初始化函数

```c
/**
 * @brief 无限滑动效果初始化
 */
void test_scroll(void) {  
    // 创建主容器（滚动区域）
    widget = lv_obj_create(lv_scr_act());
    lv_obj_set_size(widget, 240, 216);  // 可视区域大小
    lv_obj_center(widget);
    lv_obj_set_style_pad_all(widget, 0, LV_PART_MAIN);  // 去除内边距
    // 关闭弹性滚动（边缘无反弹）
    lv_obj_clear_flag(widget, LV_OBJ_FLAG_SCROLL_ELASTIC);

    // 创建复用的子对象（文本标签）
    for(int i = 0; i < ITEM_COUNT; i++) {
        item_list[i] = lv_label_create(widget);
        lv_obj_set_size(item_list[i], 212, ITEM_HEIGHT);  // 标签大小
        lv_obj_set_style_border_width(item_list[i], 1, LV_PART_MAIN);  // 边框区分
        lv_obj_align(item_list[i], LV_ALIGN_TOP_MID, 0, ITEM_TOTAL_HEIGHT * i);  // 初始位置
        lv_obj_set_style_text_align(item_list[i], LV_TEXT_ALIGN_CENTER, LV_PART_MAIN);  // 文字居中
        lv_label_set_text(item_list[i], data[i]);  // 初始文本
    }

    // 添加滚动事件监听
    lv_obj_add_event_cb(widget, scroll_event_cb, LV_EVENT_SCROLL, NULL);
}
```

初始化步骤说明：

1. 创建主容器并设置大小、位置，关闭弹性滚动
2. 预创建 6 个文本标签作为复用对象，设置初始位置和样式
3. 绑定滚动事件，确保滚动时触发内容更新

## 四、效果展示与调试技巧

### 预期效果

- 初始显示数据 1-6，垂直滚动时内容平滑更新
- 滚动到末尾后自动循环显示 1-30（通过取模实现）
- 内存占用稳定（仅 6 个标签对象），无内存泄漏

### 调试建议

1. 打印`scroll_y`和`first_visible`值，验证索引计算是否正确
2. 调整`ITEM_COUNT`值（建议为可见数量 + 1），观察滚动是否有空白
3. 若出现内容闪烁，可在`update_visible_items`中添加防抖动判断（如仅当`first_visible`变化时更新）

## 五、优化方向

1. **动态调整复用数量**：根据容器高度自动计算`ITEM_COUNT`（如`216 / ITEM_HEIGHT + 1`）
2. **添加滚动动画**：通过`lv_anim`实现平滑过渡，减少视觉跳变
3. **支持水平滚动**：修改`lv_obj_set_scroll_dir`为水平方向，调整`X`坐标计算逻辑
4. **数据懒加载**：结合实际业务场景，滚动时动态从 Flash/SPIFFS 加载数据，而非预存数组

## 六、总结

本文介绍的无限滚动列表通过对象复用机制，在资源受限的嵌入式设备中高效展示大量数据，核心是通过滚动偏移计算实现少量对象的动态更新。这种思路不仅适用于 LVGL，也可迁移到其他 GUI 框架（如 Qt for MCUs、LittlevGL），帮助开发者在内存与性能之间找到平衡。

如果你在实现过程中遇到问题，欢迎在评论区交流调试经验！