# ISP-pipeline-Cmodel-DPC-
# DPC模块文档

## 1. 简介

### 1.1 需求及目的

本文档描述了Cmodel中DPC（Dead Pixel Correction，坏点校正）模块的相关算法和实现。通过该模块，可以对RAW图像中的坏点进行检测与校正，有效提高图像质量。团队成员可依据本文档了解DPC模块设计、代码结构和实现细节，并参考此内容自行实现相关功能。

### 1.2 定义及缩略词说明

| 定义 | 说明                 |
| ---- | -------------------- |
| DPC  | Dead Pixel Correction，坏点校正 |

---

## 2. 概述

### 2.1 DPC位置

DPC模块位于ISP图像处理流程的RAW数据阶段，通常在黑电平校正（BLC）之后、自动白平衡（AWB）之前。

---

### 2.2 DPC介绍

DPC模块用于检测并修复图像中的坏点，坏点指那些明显偏离正常像素值的异常点。通过DPC校正，可以避免坏点在后续ISP处理中被放大，提升整体图像质量。

---

### 2.3 图像坏点的来源

- **传感器缺陷**：制造工艺导致的像素点异常。
- **环境干扰**：高温、强光等环境因素对传感器的影响。
- **老化效应**：传感器在长期使用中可能产生的劣化。

---

## 3. DPC算法与实现

### 3.1 坏点检测

#### 算法原理

DPC模块通过比较目标像素与周围邻域像素的差异，检测坏点。若目标像素的值偏离其邻域像素的统计特性（如均值、中值）过大，则判断为坏点。

---

### 3.2 校正方法

#### 方法选择

DPC提供以下三种修复方法：
1. **均值法（Mode 0）**：以上下左右像素的均值替代坏点。
2. **中值滤波法（Mode 1）**：用邻域中值替代坏点。
3. **梯度修复法（Mode 2）**：通过梯度分析选择最佳方向修复坏点。

#### 方法公式

1. **均值法**：
   $$
   P_{\text{new}} = \frac{P_{上下左右}}{4}
   $$
2. **中值滤波法**：
   $$
   P_{\text{new}} = \text{Median}(P_{1}, P_{2}, \ldots, P_{8})
   $$
3. **梯度修复法**：
   $$
   P_{\text{new}} = \frac{P_{i} + P_{j}}{2}, \quad \text{where Gradient(i, j) is minimum}
   $$

---

### 3.3 函数实现

#### 顶层函数`dpc`

**接口定义**：

| 参数        | 说明                           |
| ----------- | ------------------------------ |
| isp_image   | 输入RAW图像数据                |
| isp_param   | ISP顶层参数                    |

**功能描述**：

1. 遍历每个像素点，调用坏点检测函数`dead_pixel_detection`判断是否为坏点。
2. 根据配置的校正方法，调用相应的校正函数。
3. 更新图像数据，完成坏点修复。

---

### 3.4 配置参数

#### 数据结构

```c
typedef struct {
    u8 enable;    // 是否启用DPC模块
    u8 mode;      // 校正模式
} dpc_reg_t;
```

#### 配置映射

| 参数名   | 默认值 | 描述                     |
| -------- | ------ | ------------------------ |
| DPC_EN   | 1      | 是否启用DPC模块          |
| DPC_MODE | 0      | 修复模式：0均值，1中值，2梯度 |

```c
Mapping dpc_cfg_map[] = {
    {"DPC_EN" ,  (int *)&dpc_cfg_reg.enable , 0, 1},
    {"DPC_MODE", (int *)&dpc_cfg_reg.mode   , 0, 2},
    {NULL, NULL, -1, -1}
};
```

---

### 3.5 主要函数

#### 坏点检测

```c
u16 dead_pixel_detection(u16 pixel[9]);
```

- **功能**：判断中心像素是否为坏点。
- **输入**：9邻域像素数组。
- **输出**：1为坏点，0为正常像素。

#### 中值滤波

```c
u16 median_filter(u16 pixel[9]);
```

- **功能**：计算邻域中值，返回修复后的像素值。

#### 梯度修复

```c
u16 gradient_filter(u16 pixel[9]);
```

- **功能**：通过梯度分析选择修复方向，返回修复后的像素值。

---

### 3.6 实现流程

1. 初始化行缓存和窗口。
2. 遍历图像：
   - 滑动窗口，更新邻域像素。
   - 调用坏点检测函数，判断当前像素是否异常。
   - 根据校正模式调用对应修复算法。
3. 更新图像数据。

---

## 4. DPC代码示例

```c
void dpc(ISP_IMAGE *isp_image, const ISP_PARAM *isp_param) {
    printf("start dpc....\n");
    if (!dpc_cfg_reg.enable) {
        printf("DPC module is disabled.\n");
        return;
    }

    int width = isp_image->input_width;
    int height = isp_image->input_height;

    u16 rawWindow[5][5] = {0};
    u16 pix[9];
    u16 dst_t;
    u8 flag;

    for (int row = 0; row < height; row++) {
        for (int col = 0; col < width; col++) {
            // 更新窗口数据
            update_window(rawWindow, isp_image, row, col);

            // 坏点检测
            flag = dead_pixel_detection(rawWindow[2]);
            if (flag == 1) {
                switch (dpc_cfg_reg.mode) {
                    case 0:
                        dst_t = average_filter(pix);
                        break;
                    case 1:
                        dst_t = median_filter(pix);
                        break;
                    case 2:
                        dst_t = gradient_filter(pix);
                        break;
                }
            } else {
                dst_t = pix[0];
            }
            isp_image->BAYER_DAT[row * width + col] = dst_t;
        }
    }
    printf("finish dpc\n");
}
```

---

## 5. 总结

DPC模块通过多种校正算法，有效修复图像中的坏点，改善图像质量。通过可配置的模式选择，DPC模块能够适配不同场景需求，是ISP图像处理流程中的重要环节。
