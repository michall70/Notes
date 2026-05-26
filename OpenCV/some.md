### convertTo
```c++
void Mat::convertTo(
    OutputArray m,      // 输出的目标矩阵
    int rtype,          // 输出矩阵的类型（比如 CV_8UC1）
    double alpha = 1,   // 缩放因子（默认为1）
    double beta = 0     // 偏移量（默认为0）
) const;

depth32F.convertTo(depthNorm, CV_8UC1,
                   255.0 / (maxVal - minVal),              // alpha
                   -minVal * 255.0 / (maxVal - minVal));   // beta
```


### 灰度转色彩
```c++
void cv::applyColorMap(
    InputArray src,        // 输入灰度图（必须为 CV_8UC1，灰度值 0~255）
    OutputArray dst,       // 输出彩色图（CV_8UC3，BGR 顺序）
    int colormap           // 颜色映射表的 ID
);

cv::Mat depthColor;
cv::applyColorMap(depthNorm, depthColor, cv::COLORMAP_JET);
```
- **`cv::COLORMAP_JET`**：OpenCV 预置的一种“蓝-青-绿-黄-红”渐变的色彩表。  
    具体的对应关系（近似）是：
    - 灰度 0 → 深蓝色 (B:128, G:0, R:0)
    - 灰度 64 → 青色 (B:0, G:255, R:255)
    - 灰度 127 → 绿色 (B:0, G:255, R:0)
    - 灰度 191 → 黄色 (B:0, G:255, R:255)
    - 灰度 255 → 深红色 (B:0, G:0, R:128)
所以，在没有做反转之前：
- 深度值最小（离相机最近）的地方 → 灰度 0 → 蓝色
- 深度值最大（离相机最远）的地方 → 灰度 255 → 红色  
    这就是你看到的“越远越红”的来源。

灰度按位取反
`cv::bitwise_not(depthNorm, depthNormInv);`


### `cv::Matx33d` — 固定 3×3 双精度矩阵
它的大小在编译时就固定了，不能像 `cv::Mat` 那样随意 resize，但访问更快、写法更简洁。