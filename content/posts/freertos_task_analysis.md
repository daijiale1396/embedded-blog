---
title: "初次点亮 STM32"
date: 2025-11-12
---

这是你博客的第一篇文章，用来记录那些让 MCU 发光的瞬间。

```c
#include "stm32f1xx_hal.h"

int main(void)
{
    HAL_Init();
    while (1) {
        HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);
        HAL_Delay(500);
    }
}
