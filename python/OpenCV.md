# OpenCV模块

## OpenCV安装

```bash
pip install opencv-python
```

##  图像基础处理

### `cv2.imread()`读取图像

####　**参数：**

* **filename**（必填，字符串）：图像路径（支持中文路径需使用 `cv2.imdecode`）
* **flags**（可选，整数）：读取模式，默认 `cv2.IMREAD_COLOR`（彩色图）
  * `cv2.IMREAD_COLOR` 或 `1`：加载彩色图（忽略透明度）
  * `cv2.IMREAD_GRAYSCALE` 或 `0`：加载灰度图
  * `cv2.IMREAD_UNCHANGED` 或 `-1`：保留 Alpha 通道（如 PNG 透明背景）

```python
img = cv2.imread('image.jpg')
```

####　**返回值**：

- **numpy.ndarray**：成功返回图像数据（Numpy数组），失败返回 `None`

### `cv2.imshow()` 显示图像

####　**参数**：

- **winname**（必填，字符串）：窗口名称，需唯一标识
- **mat**（必填，numpy.ndarray）：要显示的图像数据（BGR或灰度格式）


```python
cv2.imshow("show", image) #窗口标签，要显示图像的类
```

### `cv2.waitKey()` 等待键盘输入
#### **参数**：

- **delay**（可选，整数）：等待时间（毫秒），默认 `0` 表示无限等待

```python
cv2.imwrite(0) #无限等待
```

#### **返回值**：

- **按键 ASCII 码**：若按下按键则返回对应 ASCII 值（如 q 返回 113）
- **-1**：超时未按键

### `cv2.destroyAllWindows()` 关闭所有窗口

```python
cv2.destroyAllWindows()
```

### `cv2.destroyWindow()` 关闭指定窗口

### 参数

- **`winname`**（必填，字符串类型）
  需要关闭的窗口名称，必须与创建窗口时通过 `cv2.imshow()` 或 `cv2.namedWindow()` 设置的名称完全一致（区分大小写和特殊字符）

```python
cv2.destroyWindow("show")
```

###　`cv2.imwrite()` 保存图像

####　**参数：**

* **filename**（必填，字符串）：保存路径（需包含扩展名如 .jpg）
* **img**（必填，numpy.ndarray）：图像数据（支持8位/16位单通道或3通道BGR格式）
* **params**（可选，列表/元组）：保存参数，如压缩质量
  * JPEG：[`cv2.IMWRITE_JPEG_QUALITY`, 95]（质量0-100）
  * PNG：[`cv2.IMWRITE_PNG_COMPRESSION`, 3]（压缩级别0-9）

### `cv2.cvtColor()` 图像颜色空间转换

### 参数：

* src：输入图像（支持 uint8、float32 等数据类型）。
* code：颜色转换代码（如 cv2.COLOR_BGR2GRAY）。
* dstCn：输出图像通道数（默认 0，自动推断）。

1. **BGR 转灰度（文档扫描/人脸检测）**

   ```python
   import cv2
   gray_img = cv2.cvtColor(bgr_img, cv2.COLOR_BGR2GRAY)
   ```

2. **BGR 转 HSV（颜色追踪/分割）**

   ```python
   hsv_img = cv2.cvtColor(bgr_img, cv2.COLOR_BGR2HSV)
   lower_red = np.array([0, 50, 50])
   upper_red = np.array([10, 255, 255])
   mask = cv2.inRange(hsv_img, lower_red, upper_red)
   ```

3. **BGR 转 RGB（可视化兼容性）**

   ```python
   rgb_img = cv2.cvtColor(bgr_img, cv2.COLOR_BGR2RGB)
   plt.imshow(rgb_img)  #Matplotlib 显示需 RGB 格式
   ```

4. **BGR 转 Lab（色域分析）**

   ```python
   lab_img = cv2.cvtColor(bgr_img, cv2.COLOR_BGR2Lab)
   ```

## 获取图像基本信息

### `img.shape` 

* 返回一个元组 (height, width, channels)：
  * 高度（height）：图像垂直方向像素数，对应 shape[0]
  * 宽度（width）：图像水平方向像素数，对应 shape[1]
  * 通道数（channels）：彩色图为 3（BGR 三通道），灰度图为 1

```python
import cv2
img = cv2.imread("image.jpg")
height, width, channels = img.shape
```

## 图像算术运算

### `cv2.add()` 图像加法

### `cv2.subtract()` 图像减法

### `cv2.multiply()` 图像乘法

### `cv2.divide()` 图像除法

### `cv2.bitwise_and()` 图像与运算

### `cv2.bitwise_or()` 图像或运算

### `cv2.bitwise_not()` 图像非运算

### `cv2.bitwise_xor()` 图像异或运算

## 创建黑白图片的方法

### 使用numpy

#### 灰度图（单通道）

```python
import numpy as np
gray_img = np.zeros((480, 640), dtype=np.uint8) #黑
gray_img = np.full((480, 640), fill_value=255,dtype=np.uint8) #白
```

#### 彩色图RGB

```python
import numpy as np
color_img = np.zeros((480, 640, 3), dtype=np.uint8) #黑
color_img = np.full((480, 640, 3), fill_value=255,dtype=np.uint8)  #白
```

