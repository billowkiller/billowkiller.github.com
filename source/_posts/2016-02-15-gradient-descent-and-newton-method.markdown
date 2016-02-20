---
layout: post
title: "Gradient Descent and Newton Method"
date: 2016-02-15 14:00
comments: true
category: "Machine Learning"
tags: math
---

梯度下降和牛顿法都是最优化算法，二者都是求解无约束优化问题的方法，通过递归地逼近最优值来达到求解值。区别在于梯度下降是一阶收敛，而牛顿法是二阶收敛的，所以牛顿法通常会更快，因为牛顿法是用一个二次曲面去拟合当前所处位置的局部曲面，而梯度下降法是用一个平面去拟合当前的局部曲面。wiki上有张图形象地说明了这个问题：

<img src="https://upload.wikimedia.org/wikipedia/commons/d/da/Newton_optimization_vs_grad_descent.svg" width="200px"/>

下面给出两种方法的具体推导。

<!--more-->

## Gradient Descent 

假设 $f(x)$ 是 $R^n$ 上具有一阶连续偏导数的函数，要求解无约束最优化问题 $min f(x)$.

由于 $f(x)$ 具有一阶连续偏导数，$k$ 次迭代后在 $x^{(k)}$ 附近进行一阶泰勒展开：

$$ f(x) = f(x^{(k)}) + \nabla f(x^{(k)})^T (x - x^{(k)}) $$

第 $k+1$ 次迭代值 $x^{(k+1)} \gets x^{(k)} - \lambda \nabla f(x^{(k)})$, 其中 $-\nabla f(x^{(k)})$是负梯度方向，$\lambda$是步长。

在上述公式中，$\lambda$ 的每次迭代都可以由一维搜索得到结果，这时的梯度搜索方法叫`Exact line search`。它形成的搜索路径很有意思，相邻的搜索路径是正交的，形状是 zig-zagging，如图：

<img src="https://upload.wikimedia.org/wikipedia/commons/d/db/Gradient_ascent_%28contour%29.png" width = "300px"/>

很容易证明：
$$ \varphi(\lambda) = f(x^{(k)}) + \lambda d^{(k)}, d^{(k)} = -\nabla f(x^{(k)}) $$
为求出从 $x^{(k)}$ 出发沿着负梯度方向的极小值，令
$$ \varphi'(\lambda) = \nabla f(x^{(k)} + \lambda d^{(k)})^T d^{(k)} = 0$$
$$-\nabla f(x^{(k+1)})^T -\nabla f(x^{(k)}) = 0 $$
 
上述表明 $d^{(k)}$ 与 $d^{(k+1)}$ 正交，搜索路径是锯齿形状的，当接近极小值点的时候，每次迭代移动的步长很小，这样影响了收敛速度。

大多数的梯度搜索方法代用`inexact line search`，这种方法使用更加的普遍，它不要求每次迭代得到准确的步长值，而是采用估计值。有种搜索方法叫`backtracking line search`，它依赖两个常量：$\alpha, \beta$, 

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-2-15/4737593.jpg" width = "600px"/>

## Newton Method

牛顿法用迭代点的梯度和二阶导数对目标函数进行二次逼近，把二次函数的极小点作为新的迭代点，不断重复此过程，直到找到最优点。

假设 $f(x)$ 是 $R^n$ 上具有二阶连续偏导数的函数，要求解无约束最优化问题 $min f(x)$.

由于 $f(x)$ 具有二阶连续偏导数，$k$ 次迭代后在 $x^{(k)}$ 附近进行二阶泰勒展开：

$$ f(x) = f(x^{(k)}) + \nabla f(x^{(k)})^T (x - x^{(k)}) + 1/2 (x - x^{(k)})^T \nabla^2 f(x^{(k)}) (x - x^{(k)})$$

其中 $\nabla^2 f(x^{(k)})$ 是 $f(x)$ 在 $x^{(k)}$ 处的Hesse矩阵，为了求极值，对二阶泰勒公式求导，得到

$$ \nabla f(x) = \nabla f(x^{(k)}) + \nabla^2 f(x^{(k)})(x - x^{(k)}) = 0 $$

$$ x^{(k+1)} \gets x^{(k)} - \nabla^2 f(x^{(k)})^{-1} \nabla f(x^{(k)}) $$

其中我们假设Hesse矩阵是可逆的，并且对于正定的Hesse矩阵，我们可以确定是迭代方向是下降的，因为 $-\nabla f(x)^T \nabla^2 f(x)^{-1} \nabla f(x) < 0$, $-\nabla^2 f(x)^{-1} \nabla f(x)$就被称为 *Newton step*. 算法如下：

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-2-15/19107554.jpg" width = "600px"/>

### Quasi-Newton Methods

Quasi-Newton即拟牛顿法，这是一种“模拟“的牛顿法，它模拟牛顿法中的搜索方向的生成方式。那么为什么要模拟呢？

在牛顿法中，有如下的缺点：

* 可能出现Hesse矩阵奇异的情形，因此不能确定后继点；
* 即使矩阵非奇异，也未必正定，因而牛顿方向不一定是下降方向
* 需要计算Hesse矩阵的逆矩阵，计算比较复杂。

我们可以使用另外一个n阶矩阵 $G_k$ 来代替，并且需要确保 $G_k$ 的正定。

在牛顿法中，我们令 $\nabla f(x)$ 中 $x$ 为 $x^{(k+1)}$，得到
$$ g_{k+1} - g_k = H_k (x^{(k+1)} - x^{k}), 其中 g_k = \nabla f(x^{(k)}), H_k = \nabla^2 f(x^{(k)})$$

记 $y_k = g_{k+1} - g_k, \delta_k = x^{(k+1)} - x^{k}$, 则有
$$y_k = H_k \delta_k 或者 H_k^{-1} y_k = \delta_k $$

拟牛顿法将 $G_k$ 作为 $H_k^{-1}$ 的近似，要求矩阵 $G_k$ 同样满足，每次迭代都是正定，并且 $G\_{k+1} y_k = \delta\_k$ . 按照拟牛顿条件，每次迭代中可以选择更新矩阵 $G_{k+1} = G_k + \nabla G_k$ . 由此延伸出来三种算法

1. DFP算法使用 $G_k$ 逼近Hesse矩阵的逆矩阵。
2. BFGS算法使用 $B_k$ 逼近Hesse矩阵。
3. Broyden类算法。


