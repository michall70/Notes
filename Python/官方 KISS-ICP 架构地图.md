# 参考 · 官方 KISS-ICP 架构地图

> 安装位置：`~/anaconda3/envs/pyslam/lib/python3.11/site-packages/kiss_icp/`
> 版本：**1.3.0**　总共 3620 行 Python + 1 个 965 KB 的 C++ `.so`

---

## 0. 一句话

> **`KissICP` 就是你手搓的 `Odemetry`。**
> 连方法名都一样：`register_frame`。
>
> 你觉得"看不懂架构"，是因为**官方把"跑数据集"那一套也打包进来了** ——
> 而那一套你在线建图**根本用不上**。

---

## 1. 五层地图

```
┌─────────────────────────────────────────────────────────────┐
│ 第 5 层  tools/cmd.py          235 行   命令行入口            │  ✗ 不用
│          kiss_icp_pipeline 这个命令跑的就是它                │
├─────────────────────────────────────────────────────────────┤
│ 第 4 层  pipeline.py           217 行   OdometryPipeline      │  ✗ 不用
│          职责：遍历整个数据集、存 poses.txt、算 ATE、画图      │
├─────────────────────────────────────────────────────────────┤
│ 第 3 层  kiss_icp.py            80 行   KissICP ★★★          │  ✓ 只用这个
│          职责：一帧进，一帧出。= 你的 Odemetry                 │
├─────────────────────────────────────────────────────────────┤
│ 第 2 层  5 个"零件"（每个都是一个薄薄的 Python 壳）            │  △ 大概看看
│          preprocess.py    51 行   Preprocessor               │
│          voxelization.py  30 行   voxel_down_sample          │
│          threshold.py     58 行   AdaptiveThreshold          │
│          registration.py  65 行   Registration               │
│          mapping.py       68 行   VoxelHashMap               │
├─────────────────────────────────────────────────────────────┤
│ 第 1 层  pybind/*.so      965 KB  C++ 真身                   │  ✗ 看不到源码
│          Eigen（矩阵）+ TBB(多线程) + nanoflann(KD树)         │
└─────────────────────────────────────────────────────────────┘
```

**你要动的只有第 3 层。** 上面两层是"跑数据集评估指标"用的脚手架。

---

## 2. 为什么"一层套一层"让人懵

因为**每个零件都有两副面孔**：一个 Python 名字，一个 C++ 身体。

**命名规律（背下来，一眼看穿）：**

| Python 侧（你看得懂的） | C++ 侧（`.so` 里的） |
|---|---|
| `module.Registration` | `kiss_icp_pybind._Registration` |
| `module.Preprocessor` | `kiss_icp_pybind._Preprocessor` |
| `module.VoxelHashMap` | `kiss_icp_pybind._VoxelHashMap` |
| `module.AdaptiveThreshold` | `kiss_icp_pybind._AdaptiveThreshold` |
| `voxel_down_sample()` | `kiss_icp_pybind._voxel_down_sample` |

> **`_` 开头 = C++ 的东西。** 没有 `_` = Python 的东西。

**Python 那层有多薄？看 `registration.py` 全文 65 行：**

```python
def get_registration(config):
    return Registration(...)                       # 工厂函数：从 config 造

class Registration:
    def __init__(self, ...):
        self._registration = kiss_icp_pybind._Registration(...)   # 造 C++ 对象

    def align_points_to_map(self, points, voxel_map, ...):
        return self._registration._align_points_to_map(
            points=kiss_icp_pybind._Vector3dVector(points),       # numpy -> C++
            ...
        )
```

**整个文件里，数学运算一行都没有。** 它只干三件事：造对象、转格式、调 C++。

> **Python 层是"登记簿"，不是"引擎"。**

---

## 3. 唯一的"关卡"：`_Vector3dVector`

每次 numpy 进出 C++，都要过这一道：

```python
kiss_icp_pybind._Vector3dVector(points)     # numpy (N,3) -> std::vector<Eigen::Vector3d>
```

它就是 `std::vector<Eigen::Vector3d>`。要求：

- 形状 `(N, 3)`
- 元素类型 `float64`（double）
- 内存连续

**喂 `float32` 也能跑**（pybind 会帮你转），但**多一次拷贝**。要快就直接给 `float64`。

---

## 4. `KissICP` 内部 5 个零件 —— 对照你手搓的版本

```python
class KissICP:
    def register_frame(self, frame, timestamps):
        frame = self.preprocessor.preprocess(frame, timestamps, self.last_delta)  # ① 去畸变+裁剪
        source, frame_downsample = self.voxelize(frame)                          # ② 降采样
        sigma = self.adaptive_threshold.get_threshold()                          # ③ 自适应阈值
        initial_guess = self.last_pose @ self.last_delta                         # ④ 预测初值
        new_pose = self.registration.align_points_to_map(...)                    # ⑤ ICP
        self.adaptive_threshold.update_model_deviation(...)                      # ⑥ 更新阈值
        self.local_map.update(frame_downsample, new_pose)                        # ⑦ 更新地图
        self.last_delta = np.linalg.inv(self.last_pose) @ new_pose
        self.last_pose = new_pose
        return frame, source
```

| # | 官方零件 | 你的 `Odemetry` 里对应什么 | 差别 |
|---|---|---|---|
| ① | `Preprocessor` | `preprocess()` + `_deskew()` | 你的是**逐点 Python 循环**；人家是 C++ |
| ② | `voxel_down_sample` ×2 | **你没有这一步** | ⚠️ 这是你慢的大头 |
| ③ | `AdaptiveThreshold` | 你可能写死了阈值 | 人家会自适应 |
| ④ | `last_pose @ last_delta` | 你有 | 一样 |
| ⑤ | `Registration` | 你的 ICP 迭代 | 纯 Python/numpy vs C++ + KD树 + 多线程 |
| ⑦ | `VoxelHashMap` | 你的地图 | 字典 vs 体素哈希 + C++ |

**为什么官方一帧 23 ms，你一帧 2 s？**

1. **没有体素降采样** → 12 万个点全部拿去算 ICP
2. **去畸变是逐点 Python 循环** → 12 万次 Python 循环
3. **ICP 内层是 Python/numpy** → 每个迭代都是 Python 开销

官方降采样之后：**123586 点 → 2729 点**，ICP 只在这 2729 个点上跑。

---

## 5. 最小用法（实测跑通）

```python
import numpy as np
from kiss_icp.config import load_config      # ← 用它，不要直接 KISSConfig()
from kiss_icp.kiss_icp import KissICP

cfg = load_config(None)          # 会自动把 voxel_size 填成 max_range/100
cfg.data.deskew = False          # 我们的数据没有逐点时间戳

odom = KissICP(config=cfg)

# 每收到一帧：
pts = ...                                        # (N, 3)，float64 最好
frame, source = odom.register_frame(pts, np.array([]))
pose = odom.last_pose                            # 4x4，世界坐标 -> 当前帧
```

**实测（你的 `bridge/*.npy`，12 帧 × 2 轮）**

```
中位数 23.1 ms  ->  约 43 Hz
最小   12.9 ms
最大   78~84 ms（数据不连续、需要大跨度配准的那一帧）
```

---

## 6. 坑清单

| 症状 | 原因 | 修法 |
|---|---|---|
| `TypeError: incompatible constructor arguments ... voxel_size=None` | 直接 `KISSConfig()`，`voxel_size` 是 `None`（只有 `load_config()` 会替你算） | 用 `load_config(None)`，或手动 `cfg.mapping.voxel_size = cfg.data.max_range/100` |
| `deskew=True` 但 `timestamps` 是空的 | 我们的点云没有逐点时间戳（没有 per-point time 字段） | `cfg.data.deskew = False` |
| 喂进去形状不对 | 要 `(N, 3)` | 从 `(N,4)` 取 `[:, :3]` |
| 慢 | 喂了 float32 / 没做连续 | `.astype(np.float64)`，必要时 `.copy()` |
| 一帧耗时忽大忽小 | 数据本身不连续（跳帧）时，ICP 要搜更大范围 | 正常现象 |

---

## 7. 你可能想调的东西（都在 `config` 里）

```python
cfg.data.max_range      = 100.0    # 超过这个距离的点扔掉（跟雷达匹配！）
cfg.data.min_range      = 0.0
cfg.data.deskew         = False    # 无逐点时间戳时必须关
cfg.mapping.voxel_size  = None     # None -> 自动 = max_range/100
cfg.mapping.max_points_per_voxel = 20
cfg.registration.max_num_iterations = 500
cfg.registration.convergence_criterion = 0.0001
cfg.registration.max_num_threads = 0   # 0 = 自动（用满所有核）
cfg.adaptive_threshold.fixed_threshold = None   # 填个数字就变固定阈值
cfg.adaptive_threshold.initial_threshold = 2.0
cfg.adaptive_threshold.min_motion_th = 0.1
```

**对 Go2 的激光雷达，`max_range` 一定要改。** 官方默认 100 m，而 Utlidar 实际有效距离没那么远 —— 不改的话，体素尺寸也跟着被算错（`voxel_size = max_range/100`）。

---

## 8. 文件行数索引（想读源码从这看）

| 文件 | 行数 | 值得读吗 |
|---|---|---|
| `kiss_icp.py` | 80 | ★★★ **全读**，就这一个类是你要用的 |
| `config/config.py` | 48 | ★★★ 全读，所有可调参数 |
| `config/parser.py` | 90 | ★★ 看 `load_config` 那 15 行 |
| `preprocess.py` | 51 | ★★ 看 `preprocess()` 怎么转 C++ |
| `registration.py` | 65 | ★★ 看 `align_points_to_map()` |
| `mapping.py` | 68 | ★ 看 `VoxelHashMap` 有哪些方法 |
| `threshold.py` | 58 | ★ 两种阈值 |
| `voxelization.py` | 30 | ★ 一眼看完 |
| `pipeline.py` | 217 | ✗ 跑数据集用的，别浪费时间 |
| `datasets/*.py` | 2000+ | ✗ 全是"怎么读各种格式的数据集" |
| `tools/*.py` | 500+ | ✗ 进度条、可视化、命令行 |

---

## 9. 下一步：把 `Odemetry` 换成 `KissICP`

在 `pts_receiver.py` 里：

```python
# 原来
from odometry import Odemetry
odom = Odemetry(config)

# 换成
from kiss_icp.config import load_config
from kiss_icp.kiss_icp import KissICP
cfg = load_config(None)
cfg.data.deskew = False
odom = KissICP(config=cfg)
```

`register_frame(pts, np.array([]))` 的调用形式基本不用改 —— 因为你当初手搓时就是照着它的接口写的。