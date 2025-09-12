# PBR 学习基本笔记

## 1. 反射方程与渲染方程简单介绍

**辐射率 (辐亮度)** ：
$$
\mathrm{L} = \frac{\mathrm{d}\Phi}{\mathrm{d}\omega \cdot \mathrm{d}A \cdot \cos\theta}
$$
**BRDF** ：
$$
f(\omega_i, \omega_o) = \frac{\mathrm{d}L_o}{\mathrm{d}E_i}
$$
这里的 $E$ 是辐照率，对它取每个立体角上的微元，可借助辐射率的公式代换为：
$$
\mathrm{L} = \frac{\mathrm{d}\Phi}{\mathrm{d}\omega \cdot \mathrm{d}A \cdot \cos\theta} = \frac{\mathrm{d}E}{\mathrm{d}\omega \cdot \cos\theta} ~~~~~~~~ (1) \\
\\
由~(1)~显然有: \mathrm{d}E = L \cdot \mathrm{d}\omega \cdot \cos \theta
$$
因此 **BRDF** 又可以表示为：
$$
f(\omega_i, \omega_o) = \frac{\mathrm{d}L_o}{L_i \cdot \mathrm{d}\omega \cdot \cos \theta}
$$
因此如果已知 $L_i$ ，我们可以用如下方式求解出 $L_o$：
$$
L_o = \int\limits_\Omega{f(\omega_i, \omega_o) \cdot L_i \cdot \cos \theta}~\mathrm{d}\omega_i
$$
并且由于 $\theta$ 是入射光线和反射平面法线的夹角，因此如果二者皆为单位向量，可以使用点积将其替换：
$$
L_o = \int\limits_\Omega{f(\omega_i, \omega_o) \cdot L_i \cdot (n \cdot \omega_i)}~\mathrm{d}\omega_i
$$
更正式的写法，需要确定具体入射出射点 $p$ 的位置,，以及此时无限小立体角 $\omega$ 可以看作方向向量：
$$
L_o(p,\omega_o) = \int\limits_\Omega{f(p,\omega_i, \omega_o) \cdot L_i(p,\omega_i) \cdot (n \cdot \omega_i)}~\mathrm{d}\omega_i
$$
上式子即为反射方程，而真正的渲染方程则还需要在上述基础上加上一个自发光项 ( $e$ 即 $Emission$ )：
$$
L_o(p,\omega_o) = L_e(p,\omega_o) + \int\limits_\Omega{f(p,\omega_i, \omega_o) \cdot L_i(p,\omega_i) \cdot (n \cdot \omega_i)}~\mathrm{d}\omega_i
$$

## 2. BRDF 介绍

因此实际上渲染的时候，使用上述公式计算即可，许多具体的量例如 $L_i, n, w_i$ 均以确定，因此实际渲染的时候效果，很大程度上取决于 **BRDF** 的式子。

**BRDF** 有很多不同模拟表面光照的算法，但是通常使用的是 **Cook-Torrance BRDF**，形式如下：
$$
f_r = k_df_{lambert} + k_sf_{cook-torrance}
$$
### 2.1 Lambertian Diffuse 介绍与推导

可以看到公式分为了漫反射和镜面反射两个部分，左边 $k_d$ 是入射光被折射的比例，而 $f_{lambert}$ 表示 **Lambertian Diffuse**。

**Lambertian** 材质是指那种理想的完全漫反射表面，从任何观察方向看去，它反射的亮度都是一致的，这样的话由反射方程可以得到如下式子：
$$
L_o(\omega_o) = \int\limits_\Omega{f(\omega_i, \omega_o) \cdot L_i(\omega_i) \cdot (n \cdot \omega_i)}~\mathrm{d}\omega_i
$$

由于假设各个观察方向看去，反射的亮度都一致，那么说明 **BRDF** 是常数，与方向无关，我们现在将其记作 $f_c$。

这里先介绍一下 **定向半球反射率**，它的概念很简单：从某一个方向打过来的光，经过表面反射以后，往半球上所有方向发射出去的总能量比例。所以我们现在能知道它的公式大概是：
$$
\mathrm{R}(l) = \frac{反射到整个半球的能量}{入射总能量}
$$
那么使用物理公式描述那就是：
$$
\mathrm{R}(l) = \frac{1}{\mathrm{E}_i(l)}\int_{v\in\Omega}{\mathrm{L}_o(v)(n \cdot v)} \mathrm{d}v
$$
看过 `Real-Time Rendering 4th - Chapter 9 Physically Based Shading` 肯定发觉这个式子和书上写的有些许不同，下面推导一下，其实本质是相同的：
$$
由~\mathrm{BRDF}~定义: f(l,v) = \frac{\mathrm{dL}_o(v)}{\mathrm{dE}_i(l)} ~ 则有: \\
\\
\mathrm{R}(l) = \frac{1}{\mathrm{E}_i(l)}\int_{v\in\Omega}{\mathrm{L}_o(v)(n \cdot v)} \mathrm{d}v = \int_{v\in\Omega}\frac{\mathrm{L}_o(v)}{\mathrm{E}_i(l)} (n \cdot v) \mathrm{d}v = \int_{v\in\Omega} f(l,v) (n \cdot v) \mathrm{d}v
$$
也许还会疑惑为什么计算反射到整个半球的能量为什么是：
$$
\int_{v\in\Omega}{\mathrm{L}_o(v)(n \cdot v)} \mathrm{d}v
$$
而不是：
$$
\int_{v\in\Omega} f(l,v) \cdot {\mathrm{L}_o(v)(n \cdot v)} \mathrm{d}v ~~~~~~~~ (\times)
$$
这就属于典型的没有认真学习了 `(没错就是我自己)`，注意我们计算的时候使用的已经是出射的辐射率 $\mathrm{L}_o$ 而非 $\mathrm{L}_i$，如果我们使用 $\mathrm{L}_i$ 计算才需要使用 **BRDF** 然后积分将其转化成 $\mathrm{L}_o$ , 而且这还只是一个方向的出射辐射率，还得再计算一遍整个半球形的。

回到一开始的 **Lambertian Diffuse** 问题，由于假设各个观察方向看去，反射的亮度都一致，那么说明 **BRDF** 是常数，与方向无关，且我们已经记作了 $f_c$ 。那么对于上式的定向半球反射率就有：
$$
\mathrm{R}(l) = \int_{v\in\Omega} f_c \cdot (n \cdot v) \mathrm{d}v = f_c \cdot \int_{v\in\Omega}(n \cdot v) \mathrm{d}v = f_c \cdot \int_0^\pi\mathrm{d}\varphi \int_0^{\frac{\pi}{2}} \cos\theta \mathrm{d}\theta = \pi \cdot f_c
$$
因此最后我们可以得到：
$$
f_c = \frac{\mathrm{R}(l)}{\pi}
$$
同时根据能量守恒，我们一定有 $\mathrm{R}(l) \in [0,1]$ ，同时我们会将 **Lambertian Diffuse** 下的这个值：

记作 $c_{diff}$ `漫反射颜色 (diffuse)`  或是 $\rho$ `反照率 (albedo)` ，因此最后有：
$$
f_{lambert} = \frac{c}{\pi}
$$

### 2.2 Cook-Torrance 镜面反射部分 介绍与推导及其细节

目前主流的 **Microfacet Cook-Torrance BRDF** 模型的镜面反射部分公式为：
$$
f_r(\omega_i,\omega_o) = \frac{\mathrm{DFG}}{4(\omega_i\cdot n)(\omega_o \cdot n)}
$$
该公式的 $\mathrm{D, F, G}$ 各自都分别是一个函数，下面逐个介绍：

#### 2.2.1 法线分布函数

法线分布函数刻画了微表面朝向法线 $\mathbf{h}$ 的概率密度分布。

> 为什么是 $\mathbf{h}$ ？我们知道 $\mathbf {h}$ 计算式如下：
> $$
> \mathrm{h} = \frac{\mathrm{I} + v}{|\mathrm{I}+v|}
> $$
> 那么根据平行四边形运算法则，其实 $\mathbf{h}$ 是指向 $\mathrm{I}$ 和 $v$ 的中间，因此 $\mathbf{h} $ 我们也称为半程向量，我们知道镜面反射最理想的情况肯定是入射角和出射角完全相等。因此实际上我们使用 $\mathbf{h}$ 作为微表面法线是在刻画每个微表面上对与视线来说反射最强的部分。

我们为了刻画单位立体角内微表面所占宏表面的密度，可以使用如下公式：
$$
\mathrm{D}(h) = \frac{\mathrm{d}A_h}{\mathrm{d}A\mathrm{d}\omega_h}
$$
其中, $\mathrm{d}\omega_h$ 是以 $h$ 为中心的微分立体角，$\mathrm{d}A_h$ 是宏表面内所有法线所对应的微表面面积之和，$\mathrm{d}A$ 是宏表面面积。

下面来研究有关 $\mathrm{D}(h)$ 的一些性质：归一化

![image-20250428101209902](C:\Users\14518\AppData\Roaming\Typora\typora-user-images\image-20250428101209902.png)

上图很好的展示了如下关系： 
$$
\int D(h)d\omega_h dA \cdot(h\cdot v) = \int \mathrm{d}A_h \cdot (h\cdot v) = \mathrm{d}A \cdot (n \cdot v)
$$
 化简可得：
$$
\int_{h\in\Theta} D(h)(h\cdot v)\mathrm{d}\omega_h = n \cdot v
$$
如果此时我们的视线 $v$ 与 $n$ 相同的话，带入上式则可得：
$$
\int_{h\in\Theta} D(h)(h\cdot n)\mathrm{d}\omega_h = n \cdot n = 1 ~~\Rightarrow~~ \int_{h\in\Theta} D(h)(h\cdot n)\mathrm{d}\omega_h = 1
$$
**推导一个简单的 NDF** ：

> 来自于 [BRDF中的法线分布函数（Normal Distribution Function，NDF），几何函数（Geometry Function）与公式推导 - 知乎](https://zhuanlan.zhihu.com/p/564814632)

最直觉的想法应该是当宏表面法线 $n$ 与微表面 $h$ 最接近时， $D(h)$ 期望取得最大值，$n$ 与 $h$ 越偏离说明越粗糙。

我们可以用 $n \cdot h$ 可以这两个向量之间的偏离程度，同时我们通过一个外部可控的变量来宏观调整粗糙度，记作 $\alpha$ 。

1. 当 $\alpha$ 较小时，$D(h)$ 曲线会比较平缓，因此当我们代入不同的 $h$ 时，不同方向偏移的 $h$ 数量会体现的差不多，因此此时是粗糙的。

2. 当 $\alpha$ 很大时，$D(h)$ 变化及其陡峭，代入不同的 $h$，越靠近 $n$ 的微表面法线越多，偏移的法线几乎没有，因此此时我们说是光滑的。其实这里跟准确应该说 $\alpha$ 只是一个控制粗糙程度的因子。

因此可以将 $D(h)$ 暂时定义为 $D(h) = (h \cdot n)^\alpha$ ，接下来研究该 $D(h)$ 是否满足法线分布函数应有的性质：

最明显的一条性质必须满足，我们发现：
$$
\int_{h\in\Omega} (h\cdot n)^\alpha \cdot (h \cdot n) \mathrm{d}\omega_h
$$
这个值并不是恒等于 1 的，它会随着 $\alpha$ 的变化而改变，这样并不满足能量守恒的原则，所以可以尝试增加一个系数来尝试将最后的结果归一化，该系数暂时记作 $k$ 。

于是我们有如下等式：
$$
\int_{h\in\Omega} k \cdot (h\cdot n)^\alpha \cdot (h \cdot n) \mathrm{d}\omega_h = 1
$$
根据球面坐标系下，$n$ 指向 $z$ 轴，因此 $\cos\theta = h \cdot n$，同时代换立体角的微元有 $\mathrm{d}w_h = \sin\theta~\mathrm{d}\theta~\mathrm{d}\varphi$ ，如果我们在整个球面上积分，可以得到如下等式：
$$
k \int_0^{2\pi}\mathrm{d}\varphi \int_0^{\frac{\pi}{2}}\cos^{\alpha+1}\theta \sin\theta \mathrm{d}\theta
$$
显然存在一个换微分：
$$
k\int_0^{2\pi}\mathrm{d}\varphi \int_0^{\frac{\pi}{2}}\cos^{\alpha+1}\theta \sin\theta \mathrm{d}\theta = -2k\pi \cdot \int_0^{\frac{\pi}{2}}\cos^{\alpha+1}\theta~\mathrm{d}{\cos\theta} = -2k\pi \cdot \left.{\frac{\cos^{\alpha+2}\theta}{\alpha+2}}\right|^{\frac{\pi}{2}}_{0} = \frac{2k\pi}{\alpha+2}
$$
最后由等式可得：
$$
\frac{2k\pi}{\alpha+2} = 1 ~~~\Rightarrow ~~~ k = \frac{\alpha+2}{2\pi}
$$
因此最终的 **NDF** 函数如下：
$$
\mathrm{D}(h) = \frac{\alpha + 2}{2\pi} \cdot (\mathrm{h \cdot n})^{\alpha}
$$
实际上上式即为 **Blinn-Phong** 法线分布函数，并且我们通常会使用一个范围在 $[0,1]$ 的 $r$ 值来调整粗糙度，$r$ 和 $\alpha$ 之间存在如下映射关系：
$$
\alpha = 2\cdot (\frac{1}{r^2} - 1)
$$
取倒数并且向下位移 1，实际上就是将范围映射到 $r:[0,1] \Rightarrow \alpha:[+\infty,0]$ 这里区间写反表示映射也是反的，$0$ 对应 $+\infty$，$1$ 对应 $0$ ，$r$ 取平方使得映射关系更加陡峭，乘二的目的猜测是使得最后化简的式子更加简洁：
$$
\mathrm{D}(h) = \frac{\alpha + 2}{2\pi} \cdot (\mathrm{h \cdot n})^{\alpha} = \frac{1}{\pi r} \cdot (h \cdot n)^{\frac{2}{r^2}-2}
$$

#### 2.2.2 几何函数

法线分布函数无差别的考虑了所有微平面法线 $h$，但实际情况当微表面凹凸不平时，视线入射时，某些凹陷的地方并不会被人眼观察到，某些区域也无法被光线所照射到，微表面会存在一个自阴影的属性，因此我们需要使用几何函数来对修正这个结果。在不同文献中，**几何函数** 存在大量别名，常见的有：**Specular G**, **遮蔽函数**, **阴影函数** 等等。

#### 2.2.3 菲涅尔方程



