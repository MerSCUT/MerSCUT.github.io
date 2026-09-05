# 背景

渲染方程描述了表面某一点 $x$ 在方向 $\omega_o$ 上的出射辐射度（Radiance） $L_o$：

$$
L_o(x, \omega_o) = L_e(x, \omega_o) + \int_{\Omega} f_r(x, \omega_i, \omega_o) L_i(x, \omega_i) \cos\theta_i d\omega_i
$$
其中：

- $L_e$ 是自发光项。
- $f_r$ 是双向反射分布函数（BRDF）。
- $L_i$ 是来自入射方向 $\omega_i$ 的辐射度。
- $\cos\theta_i$ 即 $(n \cdot \omega_i)$，是几何衰减项。
- 积分域 $\Omega$ 是点 $x$ 处的半球空间。

求解该方程的最大难点在于**处理包含无限次反弹的间接光照积分项**. 光子映射常用来解决"简介光照"的计算.

参考 :

- [Lightmass源码分析之 经典Photon Mapping算法介绍](https://zhuanlan.zhihu.com/p/73749698)
- [【Lightmass】(三) 核心流程](https://zhuanlan.zhihu.com/p/639449822)



# Light Pass

光子映射的核心思想是**将光源发射出的连续能量**（辐射通量 Radiant Flux，记为 $\Phi$）**离散化为数量有限的光子**（Photons）。

假设光源的总辐射通量为 $\Phi_{total}$，算法从光源发射 $N$ 个光子。每个光子携带的初始能量（通量）为：
$$
\Delta \Phi = \frac{\Phi_{total}}{N}
$$
光子在场景中通过蒙特卡洛随机游走进行传播.

每次撞击表面时，使用**俄罗斯轮盘赌（Russian Roulette）**结合材质的反照率（Albedo）决定光子是**发生反射（漫反射/高光）还是被吸收**

如果发生反射, 光子会更新其方向并记录在当前撞击点 $x$ 的空间数据结构中（UE Lightmass 中通常使用 KD-Tree 的变体）。

<img src="https://pic2.zhimg.com/v2-986071656c46b3f7af3da5229cfe3ed5_1440w.jpg" alt="img" style="zoom: 50%;" />

# Camera Pass

我们需要计算相机视线击中点 $x$ 的出射辐射度 $L_o$。这就需要用第一阶段收集到的光子来近似渲染方程中的积分。

根据 Radiance 的公式简单变形, 代入渲染方程中, 可得 :
$$
L_o(x, \omega_o) \approx \int_{A} f_r(x, \omega_i, \omega_o) \frac{d^2\Phi(x, \omega_i)}{dA}
$$
以离散来估计连续, 假设摄像机射出的射线打到了 点 $x$, 我们在点 $x$ 处寻找最近的 $k$ 个光子（或者在固定半径 $r$ 内寻找所有光子），这些光子分布在一个半径为 $r$ 的圆形区域 $\Delta A = \pi r^2$ 内。

因此，渲染方程的近似离散求和公式变为：
$$
L_o(x, \omega_o) \approx \frac{1}{\pi r^2} \sum_{p=1}^k f_r(x, \omega_p, \omega_o) \Delta \Phi_p
$$
这里：

- $p$ 是搜索到的第 $p$ 个光子。
- $\omega_p$ 是光子 $p$ 的入射方向。
- $\Delta \Phi_p$ 是光子 $p$ 携带的辐射通量。
- $\pi r^2$ 是用来近似概率密度函数的局部面积。

通过这个公式，光子映射成功地将渲染方程复杂的半球积分，转化为对局部空间内光子能量的简单加权求和。

<img src="https://pic1.zhimg.com/v2-8dbf6367d8028512e9f02f0fd07364d4_1440w.jpg" alt="img" style="zoom:50%;" />

# Final Gather

渲染流程中, 如果对 Camera Rays 击中的点 $x$ 直接评估, 渲染质量并不高.

![img](https://pic3.zhimg.com/v2-91e7710eb463007574d6ca1b85fe1938_1440w.jpg)

Final Gather 的做法是 : 

- 在 $x$ (称为 Final Gather Point) 处, 向上半球区域发射大量的光线 (Final  Gather Ray). 
- 如果打到光源, 则直接叠加光源的照度. 
- 若打到漫反射表面, 则根据上面的公式统计间接光的总照度值, 叠加.

FG能比基础的KNN近似估计提供更好的渲染质量，代价是每个FG点发射出许多的采样射线，**其时间成本也同样巨大。**

# 照度缓存

Irradiance Caching的基本假设是：**照度在邻域内的变化是连续而且缓慢的**，所以我们不需要单独为每个点评估照度而是**只需要评估其中的一小部分点**，对于大部分点的照度评估使用已经评估照度的点使用Irradiance **梯度插值**计算出来即可。

有 Irradiance Caching，就不再需要对所有点使用高代价的FG进行照度评估了。





# UE 中的优化

直接使用上述公式 :

- 如果半径 $r$ 太大，会导致光照泄漏（Light Leaking，阴影处变得异常明亮）；

- 如果半径 $r$ 太小，会导致低频噪声（呈现出块状或斑点状，即 Splotchy 伪影）



