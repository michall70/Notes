- SLAM / VO 代码常用
- 视觉里程计 (Visual Odometry, VO)

> 适用于你那段 Orbbec 双目视觉里程计代码。
包含每个类型在代码里的**实际用途**和**最小示例**。

## 一、图像与矩阵容器（核心）

| 类型 | 一句话说明 | 代码中出现的地方 |
|---|---|---|
| `cv::Mat` | 万能矩阵/图像容器，存灰度图、描述子、旋转矩阵、平移向量等 | 存 IR 图像 `leftImg`，存描述子 `desc`，存 R / t 矩阵 |
| `cv::Matx33d` | 3×3 `double` 矩阵，编译期固定大小（比 `cv::Mat` 轻量） | 相机内参 `K`，累积旋转 `trajR` |

```
// cv::Mat  — 动态大小，最常用
cv::Mat img(height, width, CV_8UC1);         // 单通道灰度图
cv::Mat R   = cv::Mat::eye(3, 3, CV_64F);    // 3×3 单位阵

// cv::Matx33d — 编译期 3×3
cv::Matx33d K(intrinsic.fx, 0, intrinsic.cx,
              0, intrinsic.fy, intrinsic.cy,
              0, 0, 1);
```

## 二、几何 / 坐标类型

| 类型 | 一句话说明 | 代码中出现的地方 |
|---|---|---|
| `cv::Point2f` | 2D 浮点坐标 `(x, y)` | 匹配点坐标 `ptsCur` / `ptsPrev` |
| `cv::Point3f` | 3D 浮点坐标 `(x, y, z)` | 轨迹点 `traj` |
| `cv::Point` | 2D 整数坐标（等价于 `cv::Point2i`） | 画图的像素坐标 |
| `cv::Size` | 宽高 `(width, height)` | 图像尺寸 |

```
cv::Point2f pt(12.3f, 45.6f);    // 特征点的亚像素坐标
cv::Point3f pos(0.1f, 0.2f, 0.3f); // 3D 轨迹点
cv::Point   px(320, 240);         // 整数像素
cv::Size    sz(1280, 800);        // 图像尺寸
```

## 三、颜色 / 绘图参数

| 类型 | 一句话说明 | 代码中出现的地方 |
|---|---|---|
| `cv::Scalar` | BGR 颜色（或 1~4 个 `double` 值） | 画线/画圆/写字颜色 |
| `cv::LINE_AA` | 抗锯齿画线标志 | 轨迹线 `cv::line()` |
| `cv::FONT_HERSHEY_SIMPLEX` | 无衬线字体 | `cv::putText()` 写字 |
| `cv::DrawMatchesFlags::NOT_DRAW_SINGLE_POINTS` | 画匹配时不画孤立点 | `cv::drawMatches()` |

```
// 颜色 = (B, G, R)  ← 注意顺序！
cv::Scalar red(0, 0, 255);
cv::Scalar white(255, 255, 255);
cv::Scalar gray(230, 230, 230);

// 使用
cv::line(img, pt1, pt2, red, 2, cv::LINE_AA);
cv::putText(img, "start", pos, cv::FONT_HERSHEY_SIMPLEX, 0.5, red, 1);
```

## 四、特征与匹配

| 类型 | 一句话说明 | 代码中出现的地方 |
|---|---|---|
| `cv::KeyPoint` | 单个特征点：坐标、方向、尺度、响应值 | `prevKp` / `kp` |
| `cv::DMatch` | 一对匹配关系：queryIdx（当前帧索引）、trainIdx（参考帧索引）、distance（汉明距离） | `goodMatches`、knn 匹配结果 |
| `cv::ORB` | ORB 特征检测器+描述子提取器（智能指针 `cv::Ptr<cv::ORB>`） | `auto orb = cv::ORB::create(...)` |
| `cv::BFMatcher` | 暴力匹配器 | 用汉明距离做匹配 |
| `cv::NORM_HAMMING` | 汉明距离范数，专用于二进制描述子 | `cv::BFMatcher matcher(cv::NORM_HAMMING)` |
| `cv::noArray()` | "空"占位符，表示不输出 mask | `orb->detectAndCompute(img, cv::noArray(), kp, desc)` |

```
// ORB 检测 + 描述
auto orb = cv::ORB::create(2000);
std::vector<cv::KeyPoint> kp;
cv::Mat desc;
orb->detectAndCompute(img, cv::noArray(), kp, desc);

// 匹配
cv::BFMatcher matcher(cv::NORM_HAMMING);
std::vector<std::vector<cv::DMatch>> knn;
matcher.knnMatch(desc, prevDesc, knn, 2);

// 筛选好的匹配（Lowe's ratio test）
for (auto &m : knn) {
    if (m[0].distance < 0.75f * m[1].distance) {
        ptsCur.push_back(kp[ m[0].queryIdx ].pt);   // 当前帧点
        ptsPrev.push_back(prevKp[ m[0].trainIdx ].pt); // 上一帧点
    }
}
```

## 五、对极几何 & 位姿估计

| 类型 | 一句话说明 | 代码中出现的地方 |
|---|---|---|
| `cv::RANSAC` | RANSAC 鲁棒估计算法常量 | `cv::findEssentialMat()` |
| `cv::findEssentialMat()` | 5 点法 + RANSAC 求本质矩阵 **E** | 核心几何计算 |
| `cv::recoverPose()` | 从 **E** 分解出 **R** 和 **t**（方向，无尺度） | 核心几何计算 |

```
// 本质矩阵
cv::Mat inlierMask;
cv::Mat E = cv::findEssentialMat(
    ptsCur, ptsPrev, K,
    cv::RANSAC,   // 鲁棒估计
    0.999,        // 置信度
    1.0,          // 重投影阈值（像素）
    inlierMask    // 内点标记
);

// 恢复位姿
cv::Mat R_rel, t_rel;
int inlierCount = cv::recoverPose(
    E, ptsCur, ptsPrev, K,
    R_rel, t_rel, inlierMask
);
// R_rel, t_rel 表示 "当前帧 → 上一帧" 的变换
```

## 六、常用容器（STL + OpenCV）

| 写法 | 含义 |
|---|---|
| `std::vector<cv::KeyPoint>` | 一帧的所有特征点 |
| `std::vector<cv::DMatch>` | 筛选后的好匹配 |
| `std::vector<std::vector<cv::DMatch>>` | knn 原始结果（每个 query 点有 k 个候选） |
| `std::vector<cv::Point2f>` | 匹配点 2D 坐标集（输给 findEssentialMat） |
| `std::vector<cv::Point3f>` | 3D 轨迹点集 |
| `std::vector<cv::Matx33d>` | 每帧的旋转矩阵 |

## 七、位姿累加关键公式（你的代码中）

```
// recoverPose 给出:  cur 帧 相对于 prev 帧 的变换  (R_rel, t_rel)
// 即: P_prev = R_rel * P_cur + t_rel

// 但我们要累积 "世界 → 当前帧" 的变换，所以取逆：
cv::Mat R_rel_t = R_rel.t();   // R 的转置 = 逆

// 链式累积
t_new = t_accum - R_accum * R_rel_t * t_rel;
R_accum = R_accum * R_rel_t;

// 相机中心在世界坐标系的位置
cv::Mat C = -R_accum.t() * t_accum;   // 3×1
```

> **口诀**：`recoverPose` 给的是 `cur→prev`，累积时要**反转方向**再串起来。

## 八、类型记忆卡片（打印用）

```
┌─────────────────┬──────────────────────────────────┐
│ cv::Mat         │ 图像 / 描述子 / 矩阵               │
│ cv::Matx33d     │ 3×3 double 矩阵（内参 K）          │
│ cv::Point2f     │ (x,y) 浮点坐标                    │
│ cv::Point3f     │ (x,y,z) 3D 点                    │
│ cv::Point       │ (x,y) 整数像素                    │
│ cv::Scalar      │ BGR 颜色                          │
│ cv::KeyPoint    │ 特征点（坐标+方向+尺度）             │
│ cv::DMatch      │ 匹配对（queryIdx, trainIdx, dist) │
│ cv::ORB         │ ORB 检测器/描述子                  │
│ cv::BFMatcher   │ 暴力匹配器                         │
│ cv::noArray()   │ 占位"空 mask"                     │
│ cv::NORM_HAMMING│ 汉明距离                          │
│ cv::RANSAC      │ 随机采样一致性                     │
└─────────────────┴──────────────────────────────────┘
```

> **初学建议**：不用死记，写代码时对照这个表查就行。
写多了自然就记住了 `cv::Mat`、`cv::Point2f`、`cv::KeyPoint` 这三个最常用的。