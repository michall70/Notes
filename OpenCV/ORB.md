### warm-up ORB
`{}` 局部作用域
```cpp
// Warm-up: ORB lazily allocates internal buffers on the first call (~2.7s).
// Use random noise so FAST finds keypoints and triggers full allocation.
{                                                     // 1. 左花括号：进入局部作用域
    cv::Mat dummy(cv::Size(1280, 800), CV_8UC1);      // 2. 创建 1280×800 单通道 8 位无符号整数图像
    cv::randu(dummy, 0, 256);                         // 3. 用随机数填充，每个像素值 0~255 均匀分布
    std::vector<cv::KeyPoint> wk;                     // 4. 临时关键点容器
    cv::Mat                   wd;                     // 5. 临时描述子矩阵
    orb->detectAndCompute(dummy, cv::noArray(), wk, wd); // 6. 执行检测+描述，触发所有内部初始化
}                                                     // 7. 右花括号：退出作用域，dummy/wk/wd 全部销毁
```


### ORB cheatsheet
```cpp
OpenCV ORB — Cheat Sheet

1. cv::ORB::create(...) — Create the detector
cv::Ptr<cv::ORB> orb = cv::ORB::create(
    int     nfeatures      = 500,    // max keypoints to keep
    float   scaleFactor    = 1.2f,   // pyramid shrink ratio (1.2 = ×0.83 each level)
    int     nlevels        = 8,      // pyramid levels
    int     edgeThreshold  = 31,     // ignore border pixels (must ≥ patchSize)
    int     firstLevel     = 0,      // start level (0 = full-res)
    int     WTA_K          = 2,      // bits per descriptor element (2, 3, or 4)
    int     scoreType      = cv::ORB::HARRIS_SCORE,  // or FAST_SCORE
    int     patchSize      = 31,     // descriptor patch size
    int     fastThreshold  = 20      // FAST corner threshold (lower = more)
);

When to tweak what:
/You want...	                    /Change
More keypoints	                   ↓ fastThreshold, ↑ nfeatures
Faster detection	               scoreType = FAST_SCORE, ↑ fastThreshold
Multi-scale robustness	           ↑ nlevels, ↓ scaleFactor (closer to 1.0)
Fewer border false-positives	   ↑ edgeThreshold

2. detectAndCompute(...) — Find features + compute descriptors
orb->detectAndCompute(
    cv::Mat        image,        // input: 8-bit grayscale (CV_8UC1)
    cv::Mat        mask,         // optional: only process where mask>0 (use cv::noArray() for none)
    vector<KeyPoint> &keypoints, // OUTPUT: detected corners
    cv::Mat        &descriptors  // OUTPUT: CV_8U matrix, rows=keypoints, cols=32
);

3. cv::KeyPoint — What you get back
keypoint.pt.x       // x pixel coordinate
keypoint.pt.y       // y pixel coordinate
keypoint.size       // diameter of the feature (scale)
keypoint.angle      // orientation in degrees (-1 = not computed)
keypoint.response   // how strong / confident the feature is
keypoint.octave     // which pyramid level it was found on (0 = full-res)

4. cv::drawKeypoints(...) — Visualize
cv::drawKeypoints(
    cv::Mat        image,        // input: original image
    vector<KeyPoint> keypoints,  // keypoints to draw
    cv::Mat        &outImg,      // OUTPUT: color image with drawings
    cv::Scalar     color,        // cv::Scalar::all(-1) = random per keypoint
    int            flags = cv::DrawMatchesFlags::DEFAULT
);
Flags:
- DEFAULT — small red circles only
- DRAW_RICH_KEYPOINTS — circle + orientation line + size (informative)
- DRAW_OVER_OUTIMG — draw on existing outImg instead of clearing it
  
4. Matching across 2 images (next step when you're ready)
cv::BFMatcher matcher(cv::NORM_HAMMING);       // brute-force Hamming distance
std::vector<cv::DMatch> matches;
matcher.match(descriptors1, descriptors2, matches);
cv::Mat matchImg;
cv::drawMatches(img1, kp1, img2, kp2, matches, matchImg);
NORM_HAMMING is mandatory for ORB (binary descriptors). DMatch.queryIdx / .trainIdx / .distance tell you which feature matched which.
```

### 2. 参数详细解释
作为初学者，理解这些参数能帮你更好地调优算法。以下是详细拆解：
#### ① `edgeThreshold` (边缘阈值)
*   **解释：** 靠近图像边缘多少像素的区域**不**检测特征点。
*   **为什么需要它：** 计算特征描述子（BRIEF）需要取特征点周围的一个窗口（Patch）。如果特征点太靠近边界，窗口就会出界。
*   **你的设置：** `31`。这意味着图像最边缘 31 像素的带状区域会被忽略。通常这个值应该和 `patchSize` 保持一致。

#### ② `WTA_K`
*   **解释：** 在计算 BRIEF 描述子时，产生一个 bit 位所需要的像素点个数。
*   **细节：** 
    *   `2`：比较两个像素点（点 A 和点 B），如果 A > B 则为 1，否则为 0。这是最常用的标准模式。汉明距离匹配。
    *   `3` 或 `4`：会比较更多点，比如选 A, B, C 三个点，看谁最亮。这种方式产生的描述子信息量更大，对噪声更鲁棒。但不再是标准的二进制字符串，会影响后续匹配速度。
*   **你的设置：** `2`。这是单目 SLAM 的标准配置，保证了二进制描述子的特性，可以使用汉明距离（Hamming Distance）极速匹配。 **生成序列：** ORB 默认选了 256 对这样的点。把这 256 次比较的结果排成一排，就得到了一个 **256 位（32 字节）的二进制字符串**。这就是描述子。
**结论：** 在 SLAM 中，为了保证实时性，我们几乎永远设置 `WTA_K = 2`。


#### ③ `scoreType` (评分方式)
当你要求 ORB 只保留 500 个点，但它一口气检测出了 2000 个点时，它必须给这些点打分，把分高的留下。

##### 1. Harris Score（哈里斯评分）
*   **原理：** 它是基于图像**梯度（亮度变化率）**的。它会看这个点在水平和垂直方向的亮度变化。
*   **直观理解：** 想象你在一个漆黑的屋子里用手电筒照。
    *   如果照在**白墙**上（平坦区），你左右移动手电，光斑亮度没变化——分数极低。
    *   如果照在**一条直线**上（边缘），你沿着线动没变化，垂直线动有变化——分数中等。
    *   如果照在**桌角**上（角点），你往任何方向动，光斑覆盖的亮度都会剧烈变化——**分数极高**。
*   **评价：** 数学严谨，选出来的点非常“硬”，在相机运动时稳定性极佳，是 SLAM 的首选。

##### 2. FAST Score（FAST 评分）
*   **原理：** FAST 算法检测角点的方法是：看一个点周围的一圈像素（通常是 16 个点），如果有连续 N 个点都比中心点明显亮（或明显暗），那这就是个角点。
*   **评分计算：** 它的分数通常是：中心点与这周围 N 个点之间**亮度的差值总和**。
*   **直觉理解：** 它只看“对比度有多强”。如果一个黑点在白纸上，它的 FAST 分数就很高。
*   **评价：** 计算速度比 Harris 快得多，但它对噪声比较敏感。比如图像里的一点噪点，可能因为对比度高而被当成高分特征点，但实际上它并不稳定。


#### ④ `patchSize` (特征块大小)
*   **解释：** 计算描述子时所采用的像素区域大小。
*   **细节：** 算法会在这个 $31 \times 31$ 的窗口内通过特殊的点对采样来生成 256 位的二进制编码。
*   **注意：** 如果你把这个值调大，描述子包含的信息更多，但也更容易受到透视畸变（角度倾斜过大）的影响。
*   **你的设置：** `31`。这是 ORB 论文推荐的经典值，适配大多数场景。

#### ⑤ `fastThreshold` (FAST 阈值)
*   **解释：** **“灵敏度”**。判断一个点是否是角点时，该点与周围像素的亮度差值。
*   **效果：**
    *   **调低（如 5-10）：** 会检测出密密麻麻的点，哪怕是地毯纹理、墙面噪声也会被当作特征点。
    *   **调高（如 40-50）：** 只有对比度非常强烈的角（如黑白交界处）才会被选上，点会很少。
*   **你的设置：** `20`。这是一个中规中矩的值。**注意：** 对于红外相机，由于图像对比度通常比可见光低，如果你发现特征点太少，可以尝试把这个值降到 `10` 左右。
