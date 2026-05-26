没问题，对于刚开始接触SLAM（Simultaneous Localization and Mapping，同时定位与地图构建）和双目相机的大一同学来说，直接面对庞大的OpenCV库确实容易感到迷茫。我为你梳理了一份 **“即查即用”的C++接口速查表 (Cheat Sheet)**，它围绕你的目标定制，包含了必需的代码和足够的细节，希望能帮你快速上手。

### 🚀 写在前面：给初学者的心法

在开始写代码前，了解这几个核心心法能帮你避开不少常见坑，事半功倍：

1.  **命名空间 (`cv`)**: 几乎所有OpenCV的函数和类都在 `cv` 这个命名空间里。所以你会看到 `cv::Mat`, `cv::imread()` 这样的写法。
2.  **图像基石 (`cv::Mat`)**：`cv::Mat` 是OpenCV中存储图像数据的基本结构。它像一个矩阵，包含了图像的所有像素值。例如，读入一张图片，在程序里就是一个 `Mat` 对象。
3.  **颜色王者 (BGR vs RGB)**：这一点对新手特别重要——OpenCV默认的颜色通道顺序是**BGR（蓝、绿、红）**，而不是我们通常说的RGB。在使用 `cv::imshow()` 显示图片，或与其他库交互时，一定要留意这个差异。
4.  **数据门槛 (CV_8U)**: 很多OpenCV函数都要求输入的图像是 `CV_8U` (即8位无符号整数，像素值范围0-255) 格式。如果你的图像不是这个类型，需要先用 `Mat.convertTo()` 进行转换。
5.  **官方文档是你最好的朋友**：OpenCV的官方文档非常详尽，有大量的教程和可运行的例子，能帮你理解更复杂的用法和最新变化。

### 📖 OpenCV 4.12.0 C++ 核心速查表

#### 🗂️ 第一部分：基础数据结构与图像I/O

这是所有计算机视觉程序的“起手式”，用来处理图像的读取、显示和保存。

| 操作目标       | 代码示例                                                   | 核心参数说明                                                                                           | 备注                                                              |
| :--------- | :----------------------------------------------------- | :----------------------------------------------------------------------------------------------- | :-------------------------------------------------------------- |
| **加载图像**   | `cv::Mat img = cv::imread("path/to/img.jpg");`         | `const String& filename`: 图像路径。<br>`int flags = cv::IMREAD_COLOR`: 读取模式，详见下方“可选参数详解”。            | 如果路径错误，`Mat` 对象会是空的，可以用 `img.empty()` 来检查。                      |
| **创建纯色图**  | `cv::Mat img(480, 640, CV_8UC3, cv::Scalar(0,255,0));` | `int rows, int cols`: 行数(高)、列数(宽)。<br>`int type`: 数据类型，如 `CV_8UC3`。<br>`const Scalar& s`: BGR颜色。 | 这里创建了一张640x480的绿色图片。                                            |
| **显示图像**   | `cv::imshow("Window Name", img);`<br>`cv::waitKey(0);` | `const String& winname`: 窗口名称。<br>`InputArray mat`: 要显示的图像。                                      | **`cv::waitKey(n)` 是必须的**，它让图像窗口保持显示，`n` 是等待毫秒数，`0` 表示无限等待直到按键。 |
| **保存图像**   | `cv::imwrite("output.jpg", img);`                      | `const String& filename`: 输出路径。<br>`InputArray img`: 要保存的图像。                                     | 会根据文件后缀名自动选择编码格式。                                               |
| **转换色彩空间** | `cv::cvtColor(img_bgr, img_gray, cv::COLOR_BGR2GRAY);` | `InputArray src, OutputArray dst`: 源/目标图。<br>`int code`: 转换代码，详见下方“可选参数详解”。                      | 这是将彩色图转灰度图最常用的操作。                                               |

**`cv::imread()` 的 flags 可选参数详解**:

-   **`cv::IMREAD_UNCHANGED` (或 -1)**: 按原样读取，包含Alpha（透明度）通道。
-   **`cv::IMREAD_GRAYSCALE` (或 0)**: 始终将图像转换为灰度图。
-   **`cv::IMREAD_COLOR` (或 1, 默认值)**: 始终转换为3通道BGR彩色图。

**`cv::cvtColor()` 的 code 常用参数详解**:

-   **`cv::COLOR_BGR2GRAY`**: BGR转灰度图。
-   **`cv::COLOR_BGR2RGB`**: BGR转RGB。用于与其他库交互。
-   **`cv::COLOR_BGR2HSV`**: BGR转HSV，常用于颜色分割。
-   **`cv::COLOR_GRAY2BGR`**: 灰度转BGR。
-   **`cv::COLOR_BGR2Lab`**: BGR转Lab色彩空间。

#### 🖼️ 第二部分：核心图像处理 (`imgproc`)

掌握 `Mat` 和像素级操作是理解更高级算法的基础。

| 操作目标 | 代码示例 | 核心参数说明 |
| :--- | :--- | :--- |
| **访问并修改像素** | `// 假设是3通道彩色图` <br>`cv::Vec3b &pixel = img.at<cv::Vec3b>(row, col);` <br>`pixel[0] = 255; // B` <br>`pixel[1] = 0; // G` <br>`pixel[2] = 0; // R` | `row, col`: 像素的行、列坐标。<br>`cv::Vec3b` 表示三通道uchar。<br>`pixel[0]` 是Blue通道。 |
| **获取图像属性** | `int w = img.cols;`<br>`int h = img.rows;`<br>`int ch = img.channels();` | `rows` 和 `cols` 返回行(高)和列(宽)数。<br>`channels()` 返回通道数，灰度图为1，BGR图为3。 |
| **创建子图/ROI** | `cv::Rect roi(x, y, width, height);`<br>`cv::Mat img_roi = img(roi);` | **注意：`img_roi` 和 `img` 共享同一块数据**。修改 `img_roi` 也会改变原图。 |
| **图像缩放** | `cv::resize(src, dst, cv::Size(320, 240));` | `cv::Size dsize`: 目标尺寸 (宽, 高)。<br>`double fx=0, fy=0`: 缩放因子。<br>`int interpolation`: 插值方法，如 `cv::INTER_LINEAR` (默认，线性插值)。 |

#### 🔍 第三部分：特征提取与匹配 (SLAM核心)

这是SLAM前端视觉里程计的基础，用于找到不同图像中的对应点。

| 操作目标 | 代码示例 | 核心参数说明 | 备注 |
| :--- | :--- | :--- | :--- |
| **ORB特征** | `auto orb = cv::ORB::create();`<br>`std::vector<cv::KeyPoint> kps;`<br>`cv::Mat descs;`<br>`orb->detectAndCompute(img, cv::noArray(), kps, descs);` | `int nfeatures=500`: 最多保留的特征点数量。<br>`float scaleFactor=1.2f`: 图像金字塔的缩放比例。<br>`int nlevels=8`: 金字塔层数。 | 最经典、高效的特征点，非常适合SLAM入门。 |
| **特征匹配** | `cv::BFMatcher matcher(cv::NORM_HAMMING);`<br>`std::vector<cv::DMatch> matches;`<br>`matcher.match(descs1, descs2, matches);` | `int normType`: 距离度量。ORB描述子用 `cv::NORM_HAMMING`。<br>`bool crossCheck=false`: 设为 `true` 可获得更稳定的匹配，但速度稍慢。 | 暴力匹配（Brute-Force Matcher）会计算所有描述子对之间的距离，然后找最相似的。 |

#### 👁️ 第四部分：双目视觉与SLAM专用模块

这部分是你双目SLAM项目的核心，直接关系到如何从两张图中算出深度。

| 操作目标              | 代码示例                                                                                                                                                 | 核心参数说明                                                                                                                                        | 备注                                                             |
| :---------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------- |
| **双目立体匹配 (SGBM)** | `auto sgbm = cv::StereoSGBM::create(`<br>`    0, 64, 9, 8*9*9, 32*9*9);`<br>`cv::Mat disparity;`<br>`sgbm->compute(img_left, img_right, disparity);` | `minDisparity`: 最小视差，通常为0。<br>`numDisparities`: 最大视差与最小视差之差，必须能被16整除。<br>`blockSize`: 匹配块大小，奇数，推荐5,7,9。<br>`P1, P2`: 控制视差平滑度的参数。              | `StereoSGBM` 能生成稠密视差图。`img_left` 和 `img_right` **必须是已校正的灰度图**。 |
| **估计2D-2D位姿**     | `cv::findEssentialMat(kps1, kps2, K, cv::RANSAC, 0.999, 1.0, inliers);`                                                                              | `InputArray points1, points2`: 匹配的点集。<br>`InputArray K`: 相机内参矩阵。<br>`int method`: 异常值剔除方法，`cv::RANSAC` 很常用。<br>`double threshold`: RANSAC的阈值。 | 利用对极几何，从两帧图像的2D匹配点中恢复相机的旋转和平移。                                 |
| **三角化求3D点**       | `cv::triangulatePoints(P1, P2, pts1, pts2, pts4D);`                                                                                                  | `InputArray P1, P2`: 两个相机的投影矩阵 (3x4)。<br>`InputArray pts1, pts2`: 来自两幅图的匹配点。<br>`OutputArray pts4D`: 输出的4D齐次坐标。                               | 得到齐次坐标后，需要除以第四个分量得到真实的3D坐标。                                    |
