---
title: "TIM120P项目总结"
date: 2025-11-12T21:41:11-08:00
---

# TIM120P项目总结



### 一.遇到的问题

#### 1.点击返回触发编辑（内存重用）

​	在重复的创建删除控件时，由于未将指针置空，导致给返回按键分配的地址与原先的编辑按键地址一致，所以导致点击返回触发编辑。

触发之前：

![企业微信截图_1753265000481](C:\Users\Admin\Documents\WXWork\1688855322660095\Cache\Image\2025-07\企业微信截图_1753265000481.png)

触发之后：

#### ![企业微信截图_17532648703785](C:\Users\Admin\Documents\WXWork\1688855322660095\Cache\Image\2025-07\企业微信截图_17532648703785.png)

​	因此，在删除控件之后，一定要将指针置空，避免野指针。同时另外一个问题：lv_obj_is_valid偶尔失效，也是这个原因。

<[Lv_obj_is_valid() doesn't work 100% of the time - General discussion - LVGL Forum](https://forum.lvgl.io/t/lv-obj-is-valid-doesnt-work-100-of-the-time/10658)>

#### 2.滑动触发点击

​	这个错误的原因是，事件一直采用的LV_EVENT_RELEASED，而不是LV_EVENT_CLICKED,在LVGL底层，当无滑动松手时，才会触发CLICKED,而滑动结束会触发一次RELEAED，因此会导致滑动触发点击。



#### 3.使用fmt无法显示浮点数

​	在调用`lv_label_set_text_fmt()` 函数打印[浮点数](https://so.csdn.net/so/search?q=浮点数&spm=1001.2101.3001.7020)时，默认的会显示’f' 在lv_conf.h中找到：\#define LV_SPRINTF_CUSTOM 0，改为 #define LV_SPRINTF_CUSTOM 1

#### 4.无法实现超长选项列表的roller控件

​	在lv_conf.h中打开宏 LV_USE_LARGE_COORD 即可。



不错的lvgl开源项目：[No-Chicken/OV-Watch: A powerful Smart Watch based on STM32, FreeRTOS, LVGL.](https://github.com/No-Chicken/OV-Watch)