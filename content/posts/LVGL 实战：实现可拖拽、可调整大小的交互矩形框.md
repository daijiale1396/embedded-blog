---
title: "LVGL 实战：实现可拖拽、可调整大小的交互矩形框"
date: 2025-11-12T21:41:11-08:00
---

# LVGL 实战：实现可拖拽、可调整大小的交互矩形框

在嵌入式 GUI 开发中，经常需要实现类似 "选区框" 的交互元素 —— 支持拖拽移动位置，通过边角按钮调整大小，且有边界和最小尺寸限制。本文基于 LVGL 框架，解析如何实现这一功能，代码可直接用于智能设备的截图框、区域选择等场景。

## 一、功能概述

本次实现的交互矩形框具有以下特性：

- 核心矩形框（`drag_box`）：支持鼠标 / 触摸拖拽，可在屏幕内自由移动
- 四角控制按钮：分别位于矩形框的左上、右上、左下、右下，拖动可调整矩形大小
- 边界限制：矩形框和控制按钮不会超出屏幕范围（480×320）
- 尺寸限制：矩形框有最小宽高（40px），避免调整至过小不可见
- 实时反馈：拖拽和调整过程中，矩形框与控制按钮位置实时联动更新

## 二、核心实现思路

实现的核心是**事件驱动 + 坐标约束**：

1. 通过 LVGL 的`LV_EVENT_PRESSING`事件实时捕获拖拽动作
2. 用`lv_indev_get_vect`获取拖拽位移，计算新坐标 / 尺寸
3. 设计约束函数，确保位置和尺寸在合理范围内
4. 四角按钮与矩形框位置联动，通过`lv_obj_align_to`和`lv_obj_set_pos`保持相对位置

整体架构分为：矩形框创建、四角按钮创建、拖拽逻辑、大小调整逻辑、约束逻辑 5 个模块。

## 三、代码解析

### 1. 宏定义与全局变量

```c
// 屏幕分辨率
#define SCREEN_WIDTH 480
#define SCREEN_HEIGHT 320

// 核心矩形框对象
lv_obj_t *drag_box = NULL;
// 四角控制按钮（0:左上,1:右上,2:左下,3:右下）
lv_obj_t *corner_circles_btn[4];

// 矩形框初始属性
uint8_t drag_box_width  = 100;  // 初始宽度
uint8_t drag_box_height = 100; // 初始高度
int16_t drag_box_pos_x = 0;    // 初始X坐标
int16_t drag_box_pos_y = 0;    // 初始Y坐标
```

全局变量用于存储矩形框和控制按钮的对象指针，以及初始尺寸和位置，方便在事件回调中修改。

### 2. 四角按钮位置更新

```c
static void update_corner_pos(void)
{
    // 基于矩形框位置，对齐四角按钮（向外偏移15px，避免重叠）
    lv_obj_align_to(corner_circles_btn[0], drag_box, LV_ALIGN_OUT_TOP_LEFT, -15, 15);     // 左上角
    lv_obj_align_to(corner_circles_btn[1], drag_box, LV_ALIGN_OUT_TOP_RIGHT, 15, 15);     // 右上角
    lv_obj_align_to(corner_circles_btn[2], drag_box, LV_ALIGN_OUT_BOTTOM_LEFT, -15, -15); // 左下角
    lv_obj_align_to(corner_circles_btn[3], drag_box, LV_ALIGN_OUT_BOTTOM_RIGHT, 15, -15); // 右下角
}
```

通过`lv_obj_align_to`函数，使四角按钮始终 "附着" 在矩形框的四个角落，`LV_ALIGN_OUT_XXX`表示 "外部对齐"，最后两个参数是偏移量（避免按钮与矩形框边缘重叠）。

### 3. 矩形框拖拽逻辑

```c
static void drag_box_event_cb(lv_event_t *e) // 拖动事件回调
{
    lv_obj_t *obj        = lv_event_get_target(e); // 获取事件目标（矩形框）
    lv_indev_t *indev    = lv_indev_get_act();     // 获取当前输入设备（触摸/鼠标）
    lv_event_code_t code = lv_event_get_code(e);   // 获取事件类型

    if (indev == NULL) return; // 输入设备无效则退出

    if (obj == drag_box && code == LV_EVENT_PRESSING)
    {
        // 获取拖拽位移（相对于上一帧的移动量）
        lv_point_t vect;
        lv_indev_get_vect(indev, &vect);

        // 计算新坐标（当前位置 + 位移）
        lv_coord_t start_x = lv_obj_get_x(obj);
        lv_coord_t start_y = lv_obj_get_y(obj);
        lv_coord_t new_x   = start_x + vect.x;
        lv_coord_t new_y   = start_y + vect.y;

        // 边界约束：不超出屏幕范围
        if (new_x < 0) new_x = 0;
        if (new_y < 0) new_y = 0;
        // 右侧边界：矩形框右边缘不超过屏幕右边界
        if (new_x + lv_obj_get_width(obj) > SCREEN_WIDTH) {
            new_x = SCREEN_WIDTH - lv_obj_get_width(obj);
        }
        // 下边界：矩形框下边缘不超过屏幕下边界
        if (new_y + lv_obj_get_height(obj) > SCREEN_HEIGHT) {
            new_y = SCREEN_HEIGHT - lv_obj_get_height(obj);
        }

        // 更新矩形框位置
        lv_obj_align(drag_box, LV_ALIGN_OUT_TOP_LEFT, new_x, new_y);
        // 记录最新坐标
        drag_box_pos_x = lv_obj_get_x(drag_box);
        drag_box_pos_y = lv_obj_get_y(drag_box);

        // 刷新布局并更新四角按钮位置
        lv_obj_update_layout(lv_scr_act());
        update_corner_pos();
    }
}
```

**核心逻辑**：

- 监听`LV_EVENT_PRESSING`事件（持续按下时触发），实时获取拖拽位移
- 计算新坐标后，通过边界检查确保矩形框不超出屏幕
- 移动矩形框后，调用`update_corner_pos`让四角按钮跟随移动

### 4. 拖动四角按钮调整矩形框大小逻辑

```c
static void drag_circles_event_cb(lv_event_t *e)
{
    lv_obj_t *target     = lv_event_get_target(e); // 被拖拽的按钮
    lv_indev_t *indev    = lv_indev_get_act();
    lv_event_code_t code = lv_event_get_code(e);

    if (code == LV_EVENT_PRESSING)
    {
        // 获取拖拽位移
        lv_point_t vect;
        lv_indev_get_vect(indev, &vect);

        // 计算按钮新坐标
        lv_coord_t new_x = lv_obj_get_x(target) + vect.x;
        lv_coord_t new_y = lv_obj_get_y(target) + vect.y;

        // 按钮边界约束（不超出屏幕）
        if (new_x < 0) new_x = 0;
        if (new_y < 0) new_y = 0;
        if (new_x + lv_obj_get_width(target)/2 > SCREEN_WIDTH) {
            new_x = SCREEN_WIDTH - lv_obj_get_width(target)/2;
        }
        if (new_y + lv_obj_get_height(target)/2 > SCREEN_HEIGHT) {
            new_y = SCREEN_HEIGHT - lv_obj_get_height(target)/2;
        }

        // 按钮位置约束（不与其他按钮重叠，保留最小间距）
        constrain_corner_buttons(target, &new_x, &new_y, 40);
        // 更新按钮位置
        lv_obj_set_pos(target, new_x, new_y);

        // 根据不同按钮，调整矩形框大小
        if (target == corner_circles_btn[0]) { // 左上角按钮：调整宽高（向左/上缩小，向右/下放大）
            lv_coord_t new_width  = lv_obj_get_width(drag_box) - vect.x;
            lv_coord_t new_height = lv_obj_get_height(drag_box) - vect.y;
            // 限制最小尺寸
            constrain_drag_box_size(&new_width, &new_height, 40);
            lv_obj_set_size(drag_box, new_width, new_height);
            // 保持矩形框与按钮位置联动
            lv_obj_align(drag_box, LV_ALIGN_OUT_TOP_LEFT, 
                        new_x + lv_obj_get_width(target)/2,  // 矩形框X = 按钮X + 按钮半宽
                        new_y + lv_obj_get_height(target)/2); // 矩形框Y = 按钮Y + 按钮半高
            // 同步其他按钮位置
            lv_obj_set_pos(corner_circles_btn[1], lv_obj_get_x(corner_circles_btn[1]), new_y);
            lv_obj_set_pos(corner_circles_btn[2], new_x, lv_obj_get_y(corner_circles_btn[2]));
        }
        // 省略右上、左下、右下按钮的类似逻辑（原理相同，方向不同）
    }
}
```

**核心逻辑**：

- 拖动不同角落的按钮时，根据位移计算矩形框的新宽高（如左上角按钮左移→宽度减小，上移→高度减小）
- 通过`constrain_drag_box_size`确保矩形框不小于最小尺寸（40px）
- 调整矩形框大小后，同步更新其他角落按钮的位置，保持整体联动

### 5. 约束函数

```c
// 限制四角按钮位置（避免重叠）
static void constrain_corner_buttons(lv_obj_t *target, lv_coord_t *new_x, lv_coord_t *new_y, uint16_t min_width) 
{
    // 获取所有按钮当前坐标
    lv_coord_t x0 = lv_obj_get_x(corner_circles_btn[0]);
    lv_coord_t y0 = lv_obj_get_y(corner_circles_btn[0]);
    // ... 省略其他按钮坐标获取

    if (target == corner_circles_btn[0]) {
        // 左上角按钮不能越过右上角和左下角按钮（保留min_width间距）
        if (*new_x > x1 - min_width) *new_x = x1 - min_width;
        if (*new_y > y2 - min_width) *new_y = y2 - min_width;
    }
    // 省略其他按钮的约束逻辑
}

// 限制矩形框最小尺寸
static void constrain_drag_box_size(uint16_t *width, uint16_t *height, uint16_t min_size)
{
    if (*width < min_size) *width = min_size;
    if (*height < min_size) *height = min_size;
}
```

约束函数是交互体验的关键，避免出现按钮重叠、矩形框过小等异常情况。

### 6. 初始化函数

```c
// 创建四角按钮
void create_corner_circles(void) 
{
    for (int i = 0; i < 4; i++)
    {
        corner_circles_btn[i] = lv_btn_create(lv_scr_act());
        lv_obj_set_size(corner_circles_btn[i], 30, 30); // 按钮大小30×30px
        lv_obj_set_style_radius(corner_circles_btn[i], 15, LV_PART_MAIN); // 圆形（半径=15）
        lv_obj_set_style_border_width(corner_circles_btn[i], 1, LV_PART_MAIN); // 边框
        lv_obj_set_style_bg_opa(corner_circles_btn[i], LV_OPA_50, LV_PART_MAIN); // 半透明
        lv_obj_move_foreground(corner_circles_btn[i]); // 确保按钮在最上层（可点击）
        // 添加拖拽事件
        lv_obj_add_event_cb(corner_circles_btn[i], drag_circles_event_cb, LV_EVENT_PRESSING, NULL);
    }
    update_corner_pos(); // 初始对齐
}

// 创建矩形框
void create_drag_box(void)
{
    // 初始化屏幕（禁用滚动）
    lv_obj_set_scrollbar_mode(lv_scr_act(), LV_SCROLLBAR_MODE_OFF);
    lv_obj_clear_flag(lv_scr_act(), LV_OBJ_FLAG_SCROLLABLE);

    // 创建矩形框
    drag_box = lv_obj_create(lv_scr_act());
    lv_obj_align(drag_box, LV_ALIGN_CENTER, 0, 0); // 初始居中
    lv_obj_set_size(drag_box, drag_box_width, drag_box_height);
    lv_obj_set_style_bg_opa(drag_box, LV_OPA_0, LV_PART_MAIN); // 透明背景
    lv_obj_set_style_border_width(drag_box, 1, LV_PART_MAIN); // 红色边框
    lv_obj_set_style_border_color(drag_box, lv_color_make(255, 0, 0), LV_PART_MAIN);
    lv_obj_add_flag(drag_box, LV_OBJ_FLAG_CLICKABLE); // 允许点击
    lv_obj_add_event_cb(drag_box, drag_box_event_cb, LV_EVENT_ALL, NULL); // 添加拖拽事件

    // 创建四角按钮
    create_corner_circles();
}
```

初始化函数完成对象创建、样式设置和事件绑定，确保矩形框和按钮初始状态正确。

## 四、效果展示

运行代码后，可实现以下交互效果：

- 点击矩形框并拖动，框体随手指 / 鼠标平滑移动，到达屏幕边缘后停止
- 点击任意角落的圆形按钮并拖动：
  - 左上按钮：向左拖动→框体变窄，向上拖动→框体变高（整体向左上缩小）
  - 右下按钮：向右拖动→框体变宽，向下拖动→框体变高（整体向右下放大）
  - 所有调整均有最小尺寸限制（40px），不会无限缩小
- 按钮始终跟随矩形框移动，且不会相互重叠或超出屏幕

## 五、优化建议

1. **添加动画效果**：在调整大小和位置时，通过`lv_anim`添加过渡动画，减少视觉跳变

   ```c
   // 示例：为矩形框位置变化添加动画
   lv_anim_t anim;
   lv_anim_init(&anim);
   lv_anim_set_var(&anim, drag_box);
   lv_anim_set_values(&anim, start_x, new_x);
   lv_anim_set_time(&anim, 50); // 50ms过渡
   lv_anim_set_exec_cb(&anim, (lv_anim_exec_xcb_t)lv_obj_set_x);
   lv_anim_start(&anim);
   ```

2. **优化坐标计算**：当前通过`lv_indev_get_vect`获取位移，可改为直接读取绝对坐标（`lv_indev_get_point`），减少累积误差

3. **支持比例调整**：添加 "保持宽高比" 功能，拖动角落时按比例缩放矩形框

4. **视觉反馈增强**：拖动时可将边框颜色加深或加粗，明确当前处于调整状态

## 六、总结

本文实现的交互矩形框核心是**利用 LVGL 的事件驱动模型，通过实时坐标计算和约束逻辑，实现低内存占用的交互功能**。该方案仅使用 1 个矩形框 + 4 个按钮（共 5 个对象），适合内存受限的嵌入式设备，可直接用于截图工具、区域选择、控件调整等场景。





存在bug :当拖动超出屏幕大小时候，矩形框会大小异常，原因是lvgl底层会获取到超出屏幕大小的坐标，并且发出警告

[Warn]  (14.243, +30)    indev_pointer_proc: X is -20 which is smaller than zero        (in lv_indev.c line #358)
[Warn]  (14.808, +565)   indev_pointer_proc: X is -6 which is smaller than zero         (in lv_indev.c line #358)

[Warn]  (22.207, +34)    indev_pointer_proc: Y is -70 which is smaller than zero        (in lv_indev.c line #366)
[Warn]  (25.815, +3608)  indev_pointer_proc: Y is -12 which is smaller than zero        (in lv_indev.c line #366)

![image-20250821143357890](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20250821143357890.png)