1.运动去畸变
	匀速运动假设，根据last delta去畸变
2.体素下采样
	得到一个比较稀疏的点云，用于ICP
	得到一个比较稠密的点云，用于建图
3.获得sigma
	根据上一轮deviation获得，sigma不小于0.1
4.initial pose猜测
	匀速运动假设，根据last delta
5.ICP
	对于其中一个点，将相邻点作为匹配点，代价为残差乘上一个权重。
		KISS-ICP 用的是 **point-to-plane** ICP，代价不是 `||p' - q||`（两点距离），而是：`残差 r = (p' - q) · n`
		权重：kernel内权重=1, kernel～max correspondence distance,权重不断递减，max correspondence distance外权重=0
	迭代解最小二乘问题，代价最小化。
6.计算deviation
	本轮deviation给下一轮参考
7.更新
	更新deviatino, 更新地图， 更新last delta, 更新last pose	

--- 
防止“退化”：
1. 至少三个不同的法向量（一个法向量约束一个轴的平移，一个轴的旋转，三个就全覆盖了）
2. 不能有周期性，周期变化的水面也会导致在平移上退化，周期变化的球面会导致旋转退化
	
