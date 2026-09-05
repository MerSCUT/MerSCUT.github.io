# MIS 多重重要性采样

对于一般的渲染方程, 经典的 NEE 架构将它们分为两部分 :
$$
L_o = \int_{\Omega}f_r(w_i,w_o)L_{i}(w_i)\cos\theta_i\mathrm dw_i = \int_{\Omega}f_r(w_i,w_o)(L_{dir}(w_i) + L_{indir}(w_i))\cos\theta_i\mathrm dw_i
\\ =\int_{\Omega_1}f_r(w_i, w_o)L_{dir}(w_i)\cos\theta_i\mathrm dw_i + \int_{\Omega_2}f_r(w_i, w_o)L_{indir}(w_i)\cos\theta_i\mathrm dw_i
$$
$\Omega_1$ 是那些**光源直接照射**的半球区域, 此时根据定义有 $L_{indir} = 0$. $\Omega_2$ 反之. 

对于 $\Omega_1$, 我们有上述两种采样方法. 对光源 or 对 BRDF. 此时我们就可以在 $\Omega_1$ 上使用 MIS 了.

- 注 : 在经典架构中, MIS 是仅仅对直接光照而言的. 因为使用这种方法的前提是, **同一个积分域上有两种不同的采样策略.** 在间接光照区域只能采样方向. 但其思想可以推广到其他方向.

接下来我们也只看这一部分直接光照. 下文中令 $f = f_r \cdot \cos\theta_i$.
$$
L_o = \int_{\Omega_1}f(w_i, w_o)L_i(w_i)\mathrm dw_i
$$
其 Monte Carlo Estimator 为
$$
L_o = \frac1N\sum_{j=1}^N\frac{f(w_{i,j}, w_o)L_i(w_{i,j})}{p(w_{i,j})}
$$
其中,  $w_i$ 的 PDF 为 $p(w_i)$. 

根据**重要性采样 IS**, 理想的概率密度函数 $p(w_i)$ 应当正比于 $f(w_i, w_o)L_i(w_i)$.

为了得到这一点, 需要令
$$
p(w_i) = \frac{f(w_i, w_o)L_i(w_i)}C
$$
根据概率密度的全积分为 $1$, 得
$$
C = \int f(w_i,w_o)L_i(w_i)
$$
这本身就是我们要求的结果. 所以我们不可能在应用中提前找到 $p(w_i)$ 准确正比于被积函数的形式.

我们之前的采样方案都退而求其次, 针对 $f$ 和 $L_i$ 分别进行采样.

1. 根据 $f$ 采样 : 即 **BRDF** 采样. 
   1. 例如在 Diffuse 材质中, 我们令 $p(w_i) = \cos\theta_i$, 得到**余弦加权重要性采样**. 
   2. 缺点 : 如果**光源很小, 物体表面粗糙**, 我们很难追踪出一条光线打中光源, 且容易丢失更多光源信息.
2. 根据 $L_i$ 采样 : 即 按光源采样 Lighting Sampling / NEE.
   1. 即生成的样本方向**强行指向光源**中的某一点. 这对**小而强的光源**极其有效
   2. 缺点 : 若**光源很大, 物体材质光滑**, 对应的 BRDF 值 $f(w_i)$ 可能接近 $0$. 浪费计算资源.



MIS 就是将这两种采样策略的估计结果进行**线性加权**.

MIS Estimator :
$$
F = \sum_{i=1}^{n_1}w_1(x_{i,1})\frac{f(x_{i,1})}{n_1p_1(x_{i,1})} + \sum_{i=1}^{n_2}w_2(x_{i,2})\frac{f(x_{i,2})}{n_2p_2(x_{i,2})}
$$
其中 $p_1, p_2$ 分别是两种 IS 策略, $w_1, w_2$ 是由 $p_1, p_2$ 决定的权重. 要使得上式仍然无偏, 只需要满足 :

1. $w_1 + w_2 = 1$.
2. $p_i(x) = 0$ 时, $w_i(x) = 0$ .

而权重该怎么取 ? 有几种方案

1. **Balanced Heuristic 平衡启发式** : 
   $$
   w_i(x) = \frac{n_ip_i(x)}{\sum_{k}n_kp_k(x)}
   $$

2. **Power Heuristic 幂启发式** : 常取 $\beta = 2$.
   $$
   w_i(x) = \frac{(n_ip_i(x))^\beta}{\sum_{k}(n_kp_k(x))^\beta}
   $$

----

按照上述理论, 计算一条没有打到光源的射线需要考虑 : 

- 直接光照 : Light Sampling / NEE 策略
- 直接光照 : BRDF 策略.
- 间接光照 : BRDF 策略.

这与路径追踪的初衷相悖 : 一条射线出射了两条光线, 会导致光线数量指数爆炸.

