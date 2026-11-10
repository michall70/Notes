# 第三部分：SLAM主循环——跟踪与建图

## 1. 系统架构概览

aceSLAM采用**并行跟踪与建图（Parallel Tracking and Mapping）**策略，这是PTAM [Klein2007PTAM] 提出的经典架构。

```
┌─────────────────────────────────────────────────────────────┐
│                     RGB-D输入流                              │
│                  {I^t, D^t}_{t=0,1,2,...}                   │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│              线程1: 特征提取 (异步，持续运行)                  │
│              输出: F_t = {(f_i, x_i, d_i)}_{i=1}^M         │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│              线程2: SLAM主循环                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  优化循环 (Optimization Cycle)                       │   │
│  │  1. 采样优化窗口 W                                    │   │
│  │  2. 对W中的帧进行重定位                                │   │
│  │  3. 采样特征进行地图更新                                │   │
│  │  4. 梯度下降更新TriMLP                                │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│              输出: 相机轨迹 P = {P^0, P^1, ...}             │
│                   隐式地图 M (TriMLP网络权重)                │
└─────────────────────────────────────────────────────────────┘
```

## 2. 核心数据结构

### 2.1 帧的数据结构

```python
class Frame:
    def __init__(self, timestamp, rgb, depth, K):
        self.timestamp = timestamp
        
        # 原始数据
        self.rgb = rgb           # H x W x 3
        self.depth = depth       # H x W
        self.K = K               # 3 x 3 内参矩阵
        
        # 特征提取后
        self.features = None     # M x D 特征向量
        self.pixel_coords = None # M x 2 像素坐标
        self.local_coords = None # M x 3 局部3D坐标
        
        # 位姿估计后
        self.pose = None         # 4 x 4 齐次变换矩阵 P^t ∈ SE(3)
        self.inlier_ratio = None # λ^t 内点比例
        
        # 元信息
        self.is_keyframe = False
        self.feature_extractor = None  # 'ace' 或 'superpoint'
```

### 2.2 地图的数据结构

```python
class ImplicitMap:
    def __init__(self):
        # 核心: TriMLP网络
        self.scr_network = TriMLP(...)  # 场景坐标回归网络
        
        # 关键帧集合
        self.keyframes = []      # List[Frame]
        
        # 所有帧的位姿
        self.poses = {}          # {timestamp: P^t}
        
        # 场景范围
        self.scene_bounds = (...)  # (x_min, x_max, y_min, y_max, z_min, z_max)
```

## 3. 位姿估计：通过重定位

aceSLAM的关键创新：**完全依赖重定位来估计相机位姿**，而不是传统的光流或特征匹配。

### 3.1 重定位的数学原理

给定帧t的特征集合 `F^t = {(f_i^t, y_i^t)}_{i=1}^M`，其中：
- `f_i^t` 是第i个特征的描述子
- `y_i^t` 是对应的3D局部坐标（从深度图反投影得到）

**步骤1**：用当前地图M预测全局坐标
```python
# 固定地图M，预测每个特征的全局3D坐标
y_hat_i^t = M(f_i^t)  # TriMLP前向传播

# 得到预测的全局坐标集合
Y_hat^t = {y_hat_i^t}_{i=1}^M
```

**步骤2**：求解刚体对齐问题
```python
# 目标: 找到位姿P^t，使得预测的全局坐标与局部坐标对齐
# 数学: P^t = argmin_{P ∈ SE(3)} Σ_i ||y_hat_i^t - P * y_i^t||^2

# 这是一个经典的点集配准问题
# 可以用Kabsch-Umeyama算法高效求解
```

### 3.2 RANSAC处理外点

由于预测的全局坐标可能有错误（外点），使用RANSAC：

```python
def ransac_relocalization(y_hat, y_local, num_hypotheses=128, threshold=0.05):
    """
    RANSAC重定位
    
    参数:
        y_hat: 预测的全局坐标 (N, 3)
        y_local: 局部坐标 (N, 3)
        num_hypotheses: 候选位姿数量
        threshold: 内点阈值（米）
    
    返回:
        best_pose: 最佳位姿 (4, 4)
        best_inlier_ratio: 最佳内点比例
    """
    N = y_hat.shape[0]
    best_pose = None
    best_inlier_ratio = 0
    
    for _ in range(num_hypotheses):
        # 1. 随机采样3个点对
        indices = np.random.choice(N, 3, replace=False)
        y_hat_sample = y_hat[indices]
        y_local_sample = y_local[indices]
        
        # 2. 求解位姿（用Kabsch算法）
        R, t = kabsch_algorithm(y_local_sample, y_hat_sample)
        pose = create_homogeneous_matrix(R, t)
        
        # 3. 计算内点比例
        y_hat_transformed = (pose[:3, :3] @ y_hat.T).T + pose[:3, 3]
        errors = np.linalg.norm(y_hat_transformed - y_local, axis=1)
        inlier_ratio = np.mean(errors < threshold)
        
        # 4. 更新最佳假设
        if inlier_ratio > best_inlier_ratio:
            best_inlier_ratio = inlier_ratio
            best_pose = pose
    
    # 5. 用所有内点重新估计位姿
    y_hat_transformed = (best_pose[:3, :3] @ y_hat.T).T + best_pose[:3, 3]
    errors = np.linalg.norm(y_hat_transformed - y_local, axis=1)
    inliers = errors < threshold
    
    # 只用内点重新计算位姿
    R, t = kabsch_algorithm(y_local[inliers], y_hat[inliers])
    best_pose = create_homogeneous_matrix(R, t)
    
    return best_pose, best_inlier_ratio
```

### 3.3 内点比例的意义

```python
# 内点比例 λ^t 反映了位姿估计的质量
λ^t = 1/N * Σ_i 1(r_i^t(M, P^t) < τ)

# 其中:
# r_i^t = ||M(f_i^t) - P^t * y_i^t||^2  是残差
# τ 是阈值
# 1(·) 是指示函数

# λ^t 的含义:
# λ^t ≈ 1.0: 几乎所有特征都匹配，位姿估计很可靠
# λ^t ≈ 0.5: 只有一半特征匹配，可能有问题
# λ^t ≈ 0.1: 几乎没有匹配，位姿估计不可靠
```

## 4. 关键帧选择策略

不是每帧都成为关键帧，选择标准：

### 4.1 时间间隔条件

```python
def should_be_keyframe_by_time(frame, last_keyframe, min_time_gap=0.5):
    """如果距离上一关键帧时间超过阈值，成为关键帧"""
    time_gap = frame.timestamp - last_keyframe.timestamp
    return time_gap > min_time_gap
```

### 4.2 内点比例条件

```python
def should_be_keyframe_by_quality(frame, inlier_threshold=0.5):
    """如果内点比例低于阈值，成为关键帧（需要更多训练）"""
    return frame.inlier_ratio < inlier_threshold
```

### 4.3 完整的关键帧选择

```python
def select_keyframe(frame, keyframes, time_gap_threshold=0.5, 
                    inlier_threshold=0.5):
    """
    判断是否将当前帧设为关键帧
    
    条件1: 距离上一关键帧时间超过阈值
    条件2: 内点比例低于阈值（需要更多训练）
    """
    if len(keyframes) == 0:
        # 第一帧总是关键帧
        return True
    
    last_keyframe = keyframes[-1]
    
    # 条件1: 时间间隔
    time_condition = (frame.timestamp - last_keyframe.timestamp) > time_gap_threshold
    
    # 条件2: 位姿质量
    quality_condition = frame.inlier_ratio < inlier_threshold
    
    return time_condition or quality_condition
```

## 5. 优化窗口选择

每个优化周期，从所有关键帧中选择一个子集进行优化：

### 5.1 优化窗口的组成

```python
class OptimizationWindow:
    def __init__(self, W_L=5, W_G=10):
        """
        参数:
            W_L: 最近关键帧数量（必须包含）
            W_G: 额外采样的关键帧数量
        """
        self.W_L = W_L
        self.W_G = W_G
        
    def sample_window(self, keyframes, latest_frame, alpha=1.0):
        """
        采样优化窗口
        
        组成:
        1. 最新的W_L个关键帧（强制包含）
        2. 最新帧（无论是否关键帧）
        3. 额外W_G个关键帧（按概率采样）
        """
        window = []
        
        # 1. 最新的W_L个关键帧
        recent_kfs = keyframes[-self.W_L:]
        window.extend(recent_kfs)
        
        # 2. 最新帧
        window.append(latest_frame)
        
        # 3. 额外采样W_G个关键帧
        if len(keyframes) > self.W_L:
            remaining_kfs = keyframes[:-self.W_L]
            
            # 计算每个关键帧的采样概率
            probabilities = []
            for kf in remaining_kfs:
                # 概率 ∝ 1/|P| + α(1-λ)
                # |P|: 当前关键帧总数
                # λ: 该关键帧的内点比例
                prob = 1.0 / len(keyframes) + alpha * (1 - kf.inlier_ratio)
                probabilities.append(prob)
            
            # 归一化概率
            probabilities = np.array(probabilities)
            probabilities = probabilities / probabilities.sum()
            
            # 采样
            num_samples = min(self.W_G, len(remaining_kfs))
            sampled_indices = np.random.choice(
                len(remaining_kfs), 
                size=num_samples, 
                replace=False, 
                p=probabilities
            )
            
            for idx in sampled_indices:
                window.append(remaining_kfs[idx])
        
        return window
```

### 5.2 采样概率的直觉

```python
# 概率公式: p^t ∝ 1/|P| + α(1-λ^t)

# 第一项 1/|P|: 鼓励均匀覆盖
# - 早期关键帧少，每个被选中的概率高
# - 后期关键帧多，每个被选中的概率低

# 第二项 α(1-λ^t): 偏好质量差的关键帧
# - λ^t 小（位姿估计差）→ (1-λ^t) 大 → 概率高
# - λ^t 大（位姿估计好）→ (1-λ^t) 小 → 概率低

# α 控制平衡:
# α=0: 只考虑均匀覆盖
# α→∞: 只考虑质量差的帧
# α=1: 平衡两者
```

## 6. 特征采样与地图更新

### 6.1 特征采样策略

```python
def sample_features(frame, total_features, beta=1.0):
    """
    从帧中采样特征用于地图更新
    
    采样概率与帧的内点比例成反比
    """
    M = len(frame.features)  # 该帧的特征总数
    
    # 计算每个特征的采样概率
    # p_i^t ∝ 1/Σ|F^i| + β(1-λ^t)
    prob_per_feature = 1.0 / total_features + beta * (1 - frame.inlier_ratio)
    
    # 所有特征概率相同（因为来自同一帧）
    probabilities = np.ones(M) * prob_per_feature
    probabilities = probabilities / probabilities.sum()
    
    # 采样
    num_samples = min(256, M)  # 每帧最多采样256个特征
    sampled_indices = np.random.choice(M, size=num_samples, replace=False, p=probabilities)
    
    return sampled_indices
```

### 6.2 地图更新的损失函数

```python
def compute_mapping_loss(model, features, local_coords, poses):
    """
    计算地图更新的损失
    
    损失函数 (公式13):
    L = Σ_{(f_i^j, y_i^j, P_i^j) ∈ S_W} ||M(f_i^j) - P_i^j * y_i^j||^2
    
    参数:
        model: TriMLP网络
        features: 采样的特征 (batch, D)
        local_coords: 对应的局部坐标 (batch, 3)
        poses: 对应的位姿 (batch, 4, 4)
    """
    # 1. 用当前网络预测全局坐标
    predicted_global = model(features)  # (batch, 3)
    
    # 2. 将局部坐标变换到全局坐标系
    # P * y = R * y + t
    R = poses[:, :3, :3]  # (batch, 3, 3)
    t = poses[:, :3, 3]   # (batch, 3)
    
    target_global = torch.bmm(R, local_coords.unsqueeze(-1)).squeeze(-1) + t
    
    # 3. 计算L2损失
    loss = torch.mean((predicted_global - target_global) ** 2)
    
    return loss
```

### 6.3 完整的地图更新流程

```python
def update_map(model, optimizer, window, batch_size=1024, num_batches=4):
    """
    更新隐式地图
    
    参数:
        model: TriMLP网络
        optimizer: 优化器
        window: 优化窗口中的帧
        batch_size: 批大小
        num_batches: 每个优化周期的批次数
    """
    model.train()
    
    # 收集所有可用的特征
    all_features = []
    all_local_coords = []
    all_poses = []
    
    for frame in window:
        if frame.features is None:
            continue
        
        # 采样特征
        indices = sample_features(frame, total_features=10000)
        
        all_features.append(frame.features[indices])
        all_local_coords.append(frame.local_coords[indices])
        all_poses.append(np.tile(frame.pose, (len(indices), 1, 1)))
    
    # 合并所有数据
    all_features = np.concatenate(all_features, axis=0)
    all_local_coords = np.concatenate(all_local_coords, axis=0)
    all_poses = np.concatenate(all_poses, axis=0)
    
    # 随机打乱
    indices = np.random.permutation(len(all_features))
    all_features = all_features[indices]
    all_local_coords = all_local_coords[indices]
    all_poses = all_poses[indices]
    
    # 分批训练
    total_samples = len(all_features)
    
    for batch_idx in range(num_batches):
        # 采样一个mini-batch
        start_idx = (batch_idx * batch_size) % total_samples
        end_idx = min(start_idx + batch_size, total_samples)
        
        batch_features = torch.tensor(all_features[start_idx:end_idx], 
                                       dtype=torch.float32).cuda()
        batch_local = torch.tensor(all_local_coords[start_idx:end_idx], 
                                    dtype=torch.float32).cuda()
        batch_poses = torch.tensor(all_poses[start_idx:end_idx], 
                                    dtype=torch.float32).cuda()
        
        # 计算损失
        loss = compute_mapping_loss(model, batch_features, batch_local, batch_poses)
        
        # 反向传播
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
```

## 7. 完整的SLAM主循环

```python
class ACE_SLAM:
    def __init__(self, config):
        # 初始化组件
        self.feature_extractor = load_feature_extractor(config.feature_type)
        self.scr_network = TriMLP(config).cuda()
        self.optimizer = torch.optim.Adam(self.scr_network.parameters(), lr=1e-4)
        
        # 关键帧管理
        self.keyframes = []
        self.latest_frame = None
        
        # 优化窗口
        self.window_sampler = OptimizationWindow(W_L=5, W_G=10)
        
        # 参数
        self.time_gap_threshold = 0.5  # 秒
        self.inlier_threshold = 0.5
        self.ransac_hypotheses = 128
        self.inlier_distance_threshold = 0.05  # 米
        
    def process_frame(self, timestamp, rgb, depth, K):
        """处理一帧RGB-D数据"""
        
        # 1. 创建帧对象
        frame = Frame(timestamp, rgb, depth, K)
        
        # 2. 特征提取（异步，这里简化为同步）
        features, pixel_coords = self.feature_extractor(rgb)
        local_coords = unproject_to_3d(pixel_coords, depth, K)
        
        frame.features = features
        frame.pixel_coords = pixel_coords
        frame.local_coords = local_coords
        
        # 3. 重定位：估计位姿
        if len(self.keyframes) > 0:
            # 用当前地图预测全局坐标
            self.scr_network.eval()
            with torch.no_grad():
                features_tensor = torch.tensor(features, dtype=torch.float32).cuda()
                predicted_global = self.scr_network(features_tensor).cpu().numpy()
            
            # RANSAC估计位姿
            pose, inlier_ratio = ransac_relocalization(
                predicted_global, 
                local_coords,
                num_hypotheses=self.ransac_hypotheses,
                threshold=self.inlier_distance_threshold
            )
            
            frame.pose = pose
            frame.inlier_ratio = inlier_ratio
        else:
            # 第一帧，假设位姿为单位矩阵
            frame.pose = np.eye(4)
            frame.inlier_ratio = 1.0
        
        # 4. 关键帧选择
        if self.select_keyframe(frame):
            frame.is_keyframe = True
            self.keyframes.append(frame)
        
        self.latest_frame = frame
        
        # 5. 地图更新
        if len(self.keyframes) >= 5:  # 至少5个关键帧后才开始更新
            self.update_map()
        
        return frame.pose
    
    def select_keyframe(self, frame):
        """关键帧选择"""
        if len(self.keyframes) == 0:
            return True
        
        last_kf = self.keyframes[-1]
        time_gap = frame.timestamp - last_kf.timestamp
        
        return (time_gap > self.time_gap_threshold) or \
               (frame.inlier_ratio < self.inlier_threshold)
    
    def update_map(self):
        """更新隐式地图"""
        # 采样优化窗口
        window = self.window_sampler.sample_window(
            self.keyframes, 
            self.latest_frame
        )
        
        # 更新网络
        self.scr_network.train()
        update_map(self.scr_network, self.optimizer, window)
```

## 8. 实时性保障

### 8.1 计算成本固定

```python
# 每个优化周期的计算量是固定的:
# 1. 优化窗口: 固定数量的帧 (W_L + W_G + 1)
# 2. 重定位: 固定数量的RANSAC假设 (H=128)
# 3. 地图更新: 固定数量的mini-batch (N_B=4)

# 这确保了:
# - 处理时间可预测
# - 不会因为场景变复杂而变慢
# - 可以保证实时性
```

### 8.2 帧跳过机制

```python
def process_frame_with_skip(self, timestamp, rgb, depth, K):
    """带帧跳过的处理（保证实时性）"""
    
    start_time = time.time()
    
    # 处理当前帧
    pose = self.process_frame(timestamp, rgb, depth, K)
    
    processing_time = time.time() - start_time
    
    # 如果处理时间超过帧间隔，跳过下一帧
    frame_interval = 1.0 / 30  # 假设30 FPS
    if processing_time > frame_interval:
        self.skip_next_frame = True
    
    return pose
```

## 9. 隐式回环检测

aceSLAM如何实现回环检测（不需要额外模块）：

```python
# 当相机回到之前去过的地方时:
# 1. 当前帧的特征被提取
# 2. TriMLP预测这些特征的全局坐标
# 3. 由于网络已经"记住"了这个场景，预测的坐标会与之前的关键帧一致
# 4. RANSAC会找到正确的位姿
# 5. 内点比例会很高

# 关键机制:
# - TriMLP是一个压缩的表示，会将相似的特征映射到相似的3D坐标
# - 当看到相同的场景时，网络会输出相同的坐标
# - 这自然实现了回环检测

# 数学解释:
# 假设场景中有两个位置A和B
# 在A处训练时，网络学会了: f_A → y_A
# 当从B看到A时，特征f_A' ≈ f_A
# 网络会输出: f_A' → y_A' ≈ y_A
# RANSAC会发现: y_A'与y_A一致
# → 正确识别出回环
```

## 10. 对动态场景的鲁棒性

```python
# 动态物体（如人）的处理:
# 1. 动态物体的特征会在不同帧中出现在不同位置
# 2. TriMLP对这些特征的预测会不一致
# 3. RANSAC会将这些特征视为外点
# 4. 内点比例会降低，但位姿估计仍然正确

# 具体过程:
# 帧t: 人在位置P1，特征f_person → 预测坐标y1
# 帧t+1: 人移动到P2，同一特征f_person → 预测坐标y2
# y1 ≠ y2，不一致
# RANSAC采样时，这些点会被排除
# 只有静态背景的点被用来估计位姿
```

## 11. 总结：aceSLAM的核心流程

```
每帧处理流程:
1. 特征提取 (异步)
2. 重定位 (RANSAC)
3. 关键帧选择
4. 优化窗口采样
5. 地图更新 (梯度下降)

关键设计:
- 并行跟踪与建图
- 始终在线的重定位
- 固定计算成本
- 隐式回环检测
- 对动态场景鲁棒
```

继续第四部分（TriMLP训练细节与损失函数设计）？