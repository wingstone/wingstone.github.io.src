---
title: 'Monte Carlo Integration'
date: 2020-09-06 14:35:52
tags: [图形学,蒙特卡洛积分]
published: true
hideInList: false
feature: 
isTop: false
---

蒙特卡洛积分的简短介绍，以及PBRT中常用的函数分布；

## 概率部分定义

### 期望定义

$$
E_p[f(x)] = \int_D {f(x)p(x)} \,{\rm d}x.
$$

### 方差定义

$$
V[f(x)] = E[(f(x)-E[f(x)])^2]
$$

## 蒙特卡洛积分

### 一维均匀分布下的积分

欲求
$$
F_N = \int_a^b {f(x)} \,{\rm d}x.
$$
的积分；

假如我们有一个均匀随机分布的变量$X_i\in[a,b]$，那么蒙特卡洛积分器的形式为：
$$
F_N = \frac{b-a}{N}\sum_{i=1}^N f(X_i).
$$

证明为：
$$
E[F_N] = E[\frac{b-a}{N}\sum_{i=1}^N f(X_i)].
$$
$$
E[F_N] = \frac{b-a}{N}\sum_{i=1}^N E[f(X_i)].
$$
$$
E[F_N] = \frac{b-a}{N}\sum_{i=1}^N \int_a^b {f(x)p(x)} \,{\rm d}x.
$$
$$
E[F_N] = \frac{1}{N}\sum_{i=1}^N \int_a^b {f(x)} \,{\rm d}x.
$$
$$
E[F_N] = \int_a^b {f(x)} \,{\rm d}x.
$$

### 一维任意分布下的积分

实际上由于$p(x) = \frac{1}{b-a}$，上式是通用积分下的特殊形式，即：
$$
F_N = \frac{1}{N}\sum_{i=1}^N \frac{f(X_i)}{p(X_i)}.
$$
此即MC积分的重要本质形式，只要得到p(x)分布下的采样点，然后将采样点的f(x)求出，带入上式计算即可；采样点越多，越接近与期望值；

## MC积分的方差分析

MC积分的方差，实际上就代表积分的收敛速度；

$$
\sigma^2[F_N] = \sigma^2[\frac{1}{N}\sum_{i=1}^N \frac{f(X_i)}{p(X_i)}]
$$
$$
\sigma^2[F_N] = \frac{1}{N^2}\sum_{i=1}^N \sigma^2[\frac{f(X_i)}{p(X_i)}]
$$
令$Y = \frac{f(X)}{p(X)}$，则
$$
\sigma^2[F_N] = \frac{1}{N^2}\sum_{i=1}^N \sigma^2[Y_i]
$$
$$
\sigma^2[F_N] = \frac{1}{N^2}(N \sigma^2[Y])
$$
$$
\sigma^2[F_N] = \frac{1}{N}\sigma^2[Y]
$$
$$
\sigma[F_N] = \frac{1}{\sqrt{N}}\sigma[Y]
$$
可知，方差的大小与采样个数有关，且与概率密度分布函数p(x)与f(x)的比例有关；

p(x)与f(x)越接近，$\sigma[Y]$越小，收敛越快越稳定。最理想的情况为p(x)=f(x)，则$Y=1$并且$\sigma[Y]=0$；

### 优点

我们说蒙特卡洛积分是维度无关的，这是因为我们的方差只和N和Y有关，只要使用合适的方法使得Y本身的方差得当。

第二，蒙特卡罗积分很简单。只需要两个基本操作，即抽样和求期望。这鼓励实现中使用面向对象的黑盒接口，使得蒙特卡罗程序的设计具有很大的灵活性。

第三，蒙特卡罗方法是通用的，这是由于它是基于随机采样。就算是不是自然域,一个无限维空间（用数值求积法很难处理），但用蒙特卡罗处理它很简单。

## 已知p(x)，生成符合概率分布的采样

对于很对BRDF函数，其中的D项就是一个概率密度分布函数，如何根据此函数来获取相应的采样方向，即为这里所探讨的；

### 逆推法

1. 先计算PDF的CDF；
2. 然后再计算其反函数，即$CDF^-1(\xi)$由CDF得到原来的变量x;
3. 然后在0-1范围内进行$\xi$的均匀分布采样；
4. 由反函数计算出X即可为符合PDF的采样；

### 舍选法

1. 在更大的范围内进行符合PDF的采样；
2. 然后去除不在采样范围内的采样点；
3. 比较常见的是圆盘均匀采样；由方形采样来获取；

### Metropolis Sampling

略

## 概率密度分布变换

考虑已知一随机变量X的密度函数为$p(x)$，现在令$Y = T(x)$，现在我们要尝试求出Y的密度函数$p(y)$。可推导出:
$$
p(y) = \frac{p(x)}{|J_T(x)|}
$$
这里$|J_T(x)|$为函数T的雅克比行列式；

针对实际情况：

### 极坐标系

极坐标的变换为：

$$
x = rcos\theta\\
y = rsin\theta
$$

所得概率密度转换函数为：

$$
f(r,\theta) = rf(x,y)
$$

### 球坐标系

球坐标的变换为：

$$
x = rsin\theta cos\phi\\
y = rsin\theta sin\phi\\
z = rcon\theta
$$

所得概率密度转换函数为：

$$
f(r,\theta, \phi) = r^2sin\theta f(x,y,z)
$$

与立体角之间的概率密度转换为：
$$
f(\theta, \phi) = sin\theta f(\omega)
$$

## 二维随机采样

多维密度函数采样可以表示为一维密度的乘积，如：
$$
p(x,y) = p_x(x)p_y(y)
$$

但对于一些不可分离的密度函数，要通过条件概率密度来采样；条件密度函数$p(y|x)$是给定某一特定x的情况下y的密度函数：
$$
p(x,y) = \frac{p(x,y)}{p(x)}
$$

从联合分布中进行二维采样的基本思想是：首先计算临界概率密度函数，分离出一个特定的变量，然后使用标准一维技术从该密度中抽取样本。一旦有了样本，就可以计算给定该值的条件密度函数并使用同样的标准的一维采样技术从该分布中获取样本。

### 随机单位半球面采样

在单位半球面上均匀采样时, 每个立体角上都是等可能的. 由此得关于立体角的概率密度$f(\omega)$是常数, 可求出其为$\frac{1}{2\pi}$；

由前面得到的结论可知$f(\theta,\phi)=\frac{sin\theta}{2\pi}$

先来计算$\theta$, 得到$\theta$的边缘概率密度为:

$$
f(\theta) = \int_0^{2\pi}{f(\theta, \phi)}d\phi = \int_0^{2\pi}{\frac{sin\theta}{2\pi}}d\phi = sin\theta
$$

再得到 $\phi$ 的条件概率密度为:

$$f(\phi|\theta)=\frac{f(\theta, \phi)}{f(\theta)} = \frac{1}{2\pi}$$

$\phi$ 的概率密度在 $\theta$ 确定时是固定的, 这和我们的直觉是相同的. 接下来来计算相应的累积概率分布函数:

$$
F(\theta) = \int_0^{\theta}{sint}dt = 1-cos\theta
$$
$$
F(\phi|\theta) = \int_0^{\phi}{\frac{1}{2\pi}}dt = \frac{\phi}{2\pi}
$$

求相应的反函数, 并将 $1-\xi$ 替换为 $\xi$ , 得到:

$$
x = sin\theta cos\phi = cos(2\pi\xi_2)\sqrt{1-\xi_1^2}\\
y = sin\theta sin\phi = sin(2\pi\xi_2)\sqrt{1-\xi_1^2}\\
z = con\theta = \xi_1
$$

### 随机单位球面采样

$$
x = sin\theta cos\phi = cos(2\pi\xi_2)\sqrt{\xi_1(1-\xi_1)}\\
y = sin\theta sin\phi = sin(2\pi\xi_2)\sqrt{\xi_1(1-\xi_1)}\\
z = con\theta = 1-2\xi_1
$$

### 随机单位圆盘采样

$$
r = \sqrt{\xi_1}\\
\theta = 2\pi\xi_2
$$

### 单位半球面余弦权重采样

余弦权重采样在于，光照方程中有一个 $Dot(L,N)$ 项，使用余弦采样可以得到更小的方差。体积角下的概率密度为 $p(\omega) \approx cos\theta$ 则：

$$
 \int_{H^2}{p(\omega)} = 1
$$
$$
 \int_{0}^{2\pi}\int_{0}^{pi/2}{ccos\theta sin\theta}d\phi = 1
$$
$$
 c = \frac{1}{\pi}
$$
$$
f(\theta, \phi) = \frac{cos\theta sin\theta}{\pi}
$$

采样方法可以依旧使用上面条件密度函数的方法来计算采样；另一种方法为Malley方法，该方法先在单位圆上随机采样，然后将单位圆上的点投影到半球面上，来得到半球面上的点。

## 俄罗斯轮盘赌

为了应用俄罗斯轮盘，我们在每次要投射一条新的路径的时候，设定一个概率 $q$ ，该路径有 $q$ 的概率被终止，有 $1-q$ 的概率被继续追踪，设 $F$ 为已经计算出的蒙特卡洛估计值，那么新的变量值为：

$$
F_N = \begin{cases} \frac{F-qc}{1-q}, &{\xi > q} \\
c, &{otherwise} \end{cases}
$$
一般来说 $c=0$，之所以要将 $F$ 除以 $(1-q)$ 是为了保证该估计的期望和原始估计的期望是相等的:
$$
E[F_N]= (1-q)\frac{E[F]-qc}{1-q}+qc=E[F]
$$
我们一般把q设为路径上基于BRDF的权重，或基于basecolor的权重，权重越小则路径追踪越容易被终止。

**俄罗斯轮盘不会减少估计的方差，但是可以提升估计的效率，可以叫停一些贡献很小的路径。**

## 分割(划分机制)

根据 $F$ 的组成成分，可将F划分为几个子项的和：
$$
F = F_1+F_2+...+F_N
$$
比如，将光的反射划分为直接光照以及间接光照；通过对每一项积分，来获取最终的结果；
