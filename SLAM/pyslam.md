你说得对，我上一轮说错了两个地方，先道歉。核实后的修正：

1. **`py_module/`** 里面确实是 21 个 `.h` 头文件 + 1 个 `cpp_core_module.cpp`。这些头文件每个定义一段 pybind11 绑定，`cpp_core_module.cpp` 把它们 `#include` 组装在一起。
2. **`config.yaml` 没有 `VIEWER_ENABLED`**。关闭可视化窗口的方式是命令行参数 `--headless`，不是改 YAML。

下面我用核实后的正确信息，重新回答你的真正问题：**pyslam 到底能换什么、怎么换？**

---

# 第二讲：Pyslam 的模块化架构 & 算法替换指南

## 2.1 整体架构：四层可插拔设计

```
main_slam.py                    ← 入口脚本（你基本不改这里）
    │
    ├── ★ 特征提取/匹配 ─────── FeatureTrackerConfigs.ORB2
    │                              ↑ 可换：SIFT, SuperPoint, XFEAT, ALIKED, DISK, LoFTR…
    │
    ├── ★ 闭环检测 ──────────── LoopDetectorConfigs.DBOW3
    │                              ↑ 可换：VLAD, NetVLAD, CosPlace, iBoW, 或 None 禁用
    │
    ├── ★ 深度估计 ──────────── DepthEstimatorType.DEPTH_PRO
    │                              ↑ 可换：DepthAnythingV2/V3, RAFT-Stereo, CREStereo…
    │                              （需设置 kUseDepthEstimatorInFrontEnd=True）
    │
    └── ★ 语义分割 ──────────── Parameters.kSemanticSegmentationType
                                   ↑ 可换：DeepLabV3, Segformer, CLIP, YOLO, Detic, ODISE…
```

**每个组件**都有一个独立的：
- **配置文件**（Python 字典或类，定义该组件的可选型号）
- **工厂函数**（根据配置字符串创建具体实例）
- **枚举类型**（列出所有支持的型号）

---

## 2.2 怎么换？—— 每种算法的替换点

### ① 换特征提取器（最常用的改动）

**文件**：`main_slam.py`，第 162 行附近

```python
feature_tracker_config = FeatureTrackerConfigs.ORB2    # ← 改这一行就行
```

改成其他配置（完整列表在 `pyslam/local_features/feature_tracker_configs.py`）：

| 配置名 | 描述 | 速度快/慢 |
|---|---|---|
| `ORB2` | 默认，OpenCV ORB | 最快 |
| `SIFT` | 传统 SIFT | 快 |
| `SUPERPOINT` | 深度学习特征 | 中等 |
| `XFEAT` | 轻量 CNN 特征 | 中等 |
| `LIGHTGLUE` | SuperPoint + LightGlue 匹配 | 慢但准 |
| `LOFTR` | 无检测器，直接匹配 | 慢 |
| `DISK` | 端到端学习特征 | 慢 |

**工作原理**：
- `feature_tracker_configs.py` 中每个配置是一个字典，指定 `detector_type` + `descriptor_type` + `tracker_type`
- `feature_tracker_factory()` 根据这三个字段创建对应的实例
- 每种特征类型在 `pyslam/local_features/` 下有一个独立文件（`feature_superpoint.py`、`feature_disk.py`…）

**改完后直接 `./main_slam.py` 即可，无需编译。**

### ② 换闭环检测方法

**文件**：`main_slam.py`，第 167 行附近

```python
loop_detection_config = LoopDetectorConfigs.DBOW3    # ← 改这里
```

可用选项：

| 配置名 | 需要词袋? | 特点 |
|---|---|---|
| `DBOW2` | ✅ 需要 ORB 词汇表 | 经典方法 |
| `DBOW3` | ✅ 需要 ORB 词汇表 | DBoW2 升级版 |
| `VLAD` | ✅ 需要 ORB 词汇表 | 向量聚合 |
| `IBOW` | ❌ 无需 | 增量式，无需预训练 |
| `IBOW_INDEPENDENT` | ❌ 无需 | 独立特征管道 |
| `NETVLAD` | ❌ 无需 | 深度学习全局描述子 |
| `COSPLACE` | ❌ 无需 | 场景识别 |
| `EIGENPLACES` | ❌ 无需 | 场景识别 |
| `None` | - | **完全禁用闭环** |

配置定义在 `pyslam/loop_closing/loop_detector_configs.py`。

### ③ 换深度估计模型

需要同时设置两个地方：

**文件**：`pyslam/config_parameters.py`

```python
kUseDepthEstimatorInFrontEnd = True    # 开启前端深度估计
```

**文件**：`main_slam.py`，第 231 行附近

```python
depth_estimator_type = DepthEstimatorType.DEPTH_PRO    # ← 改这里
```

| 选项 | 适用场景 | 需要 CUDA? |
|---|---|---|
| `DEPTH_PRO` | 单目，高精度 | 推荐 |
| `DEPTH_ANYTHING_V2` | 单目 | 推荐 |
| `DEPTH_ANYTHING_V3` | 单目 | 推荐 |
| `RAFT_STEREO` | **双目**输入 | 推荐 |
| `CREStereo` | **双目**输入 | 推荐 |

**注意**：单目 SLAM 用深度估计时，深度尺度和 SLAM 尺度会打架。官方建议只在 **non-monocular**（即本来就有深度信息）时后端用，或者 mono 时只在前端用。

### ④ 切换传感器类型（mono / stereo / RGBD）

**文件**：`config.yaml`，每个数据集段下的 `sensor_type`

```yaml
KITTI_DATASET:
  sensor_type: stereo    # 可改为 mono

TUM_DATASET:
  sensor_type: rgbd      # 可改为 mono
```

这会影响到：
- 是否读深度图
- 三角化方式是双目 vs 单目
- 关键帧筛选逻辑
- 可视化中的尺度

### ⑤ 运行模式：后台模式 vs 可视化模式

```bash
./main_slam.py                   # 正常模式（弹出 3D 可视化窗口）
./main_slam.py --headless        # 后台模式（无窗口，适合服务器）
```

---

## 2.3 具体改法举例 —— 三步走

**例子：把特征从 ORB2 换成 SuperPoint + LightGlue 匹配**

**第一步**：改 `main_slam.py`

```python
# 第 162 行
# feature_tracker_config = FeatureTrackerConfigs.ORB2
feature_tracker_config = FeatureTrackerConfigs.LIGHTGLUE    # ← 改这行
```

**第二步**：确认你安装了 LightGlue（第一次运行时会自动检查，缺啥装啥）

**第三步**：运行

```bash
./main_slam.py
```

你会看到：
- 特征点数量从 ORB 的 2000 变为 SuperPoint 的默认数量
- 匹配质量更高（但每帧处理时间变长）
- 地图更稀疏但更稳定

---

## 2.4 配置文件的正确用法

**原本的配置文件结构（不要再搞混了）：**

| 配置文件 | 作用 | 你会改它的频率 |
|---|---|---|
| `config.yaml` | 选数据集、传感器类型 | 每次换数据集 |
| `config.yaml → GLOBAL_PARAMETERS:` | 覆盖 `config_parameters.py` 的值 | 偶尔 |
| `main_slam.py` 开头的几个常量 | 选特征、闭环、深度模型 | 每次换算法 |
| `pyslam/config_parameters.py` | 所有 SLAM 参数（特征数、阈值、线程） | 调参时 |
| `settings/TUM1.yaml` 等 | 相机内参、畸变系数 | 换相机时 |

**`GLOBAL_PARAMETERS:` 的正确用法**（在 `config.yaml` 末尾）：

```yaml
GLOBAL_PARAMETERS:
  kNumFeatures: 1000
  kDoVolumetricIntegration: True
  kLogsFolder: /tmp/pyslam_logs
```

这等价于改了 `config_parameters.py`，但不会污染代码版本。

---

## 2.5 下一讲预告

这次你知道了：
- ✅ pyslam 的四个可替换组件（特征、闭环、深度、语义）
- ✅ 每个组件的替换点和配置方式
- ✅ 具体的代码修改位置
- ✅ `config.yaml` 和 `main_slam.py` 各自的职责

**下一讲内容**：我们可以选一个具体的特征提取器（比如 SuperPoint），打开 `pyslam/local_features/feature_superpoint.py`，一行一行看它的实现，然后你自己写一个简单的特征提取器插进去替换它——这样你就真正懂了"论文算法怎么塞进 SLAM 框架"。你有兴趣的话，告诉我你想从哪个组件开始深入？