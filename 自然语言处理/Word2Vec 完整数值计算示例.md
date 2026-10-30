
这篇笔记不展开概念，只把 **Word2Vec 从初始词向量到梯度计算、参数更新、重新计算预测概率**完整算一遍，所有数字前后保持一致。

使用 Skip-gram + Softmax，词汇表只有 3 个词：

$$
V=\{\text{I},\ \text{love},\ \text{deep}\}
$$

训练样本：中心词 `love`，正确的上下文词 `deep`。

学习率：

$$
\alpha=0.1
$$

## 1. 初始化词向量

中心词 `love` 的输入向量：

$$
v_{\text{love}}=
\begin{bmatrix}
0.5\\
0.3
\end{bmatrix}
$$

三个词的输出向量：

$$
u_I=\begin{bmatrix}0.2\\0.1\end{bmatrix}
$$

$$
u_{\text{love}}=\begin{bmatrix}0.4\\0.2\end{bmatrix}
$$

$$
u_{\text{deep}}=\begin{bmatrix}0.2\\0.6\end{bmatrix}
$$

## 2. 计算三个候选词的点积

**候选词 I：**

$$
z_I=u_I^Tv_{\text{love}}
$$

$$
=0.2\times0.5+0.1\times0.3
$$

$$
\boxed{z_I=0.13}
$$

**候选词 love：**

$$
z_{\text{love}}=0.4\times0.5+0.2\times0.3
$$

$$
\boxed{z_{\text{love}}=0.26}
$$

**候选词 deep：**

$$
z_{\text{deep}}=0.2\times0.5+0.6\times0.3
$$

$$
\boxed{z_{\text{deep}}=0.28}
$$

因此，三个分数为：

$$
z=[0.13,\ 0.26,\ 0.28]
$$

## 3. 计算 Softmax 概率

公式：

$$
P_i=\frac{e^{z_i}}{\sum_j e^{z_j}}
$$

先计算指数：

$$
e^{0.13}\approx1.138828
$$

$$
e^{0.26}\approx1.296930
$$

$$
e^{0.28}\approx1.323130
$$

分母：

$$
S=1.138828+1.296930+1.323130
$$

$$
S\approx3.758888
$$

得到：

$$
P(I\mid love)=\frac{1.138828}{3.758888}
\approx0.302969
$$

$$
P(love\mid love)=\frac{1.296930}{3.758888}
\approx0.345030
$$

$$
P(deep\mid love)=\frac{1.323130}{3.758888}
\approx0.352000
$$

所以：

$$
\boxed{P=[0.302969,\ 0.345030,\ 0.352000]}
$$

## 4. 计算损失

正确答案是 `deep`，因此：

$$
L=-\ln P(deep\mid love)
$$

代入：

$$
L=-\ln(0.352000)
$$

$$
\boxed{L\approx1.044123}
$$

## 5. 计算三个分数的梯度

Softmax 配合交叉熵损失时：

$$
\frac{\partial L}{\partial z_i}=P_i-y_i
$$

正确答案 `deep` 的标签为 1，另外两个词的标签为 0。

因此：

$$
\frac{\partial L}{\partial z_I}
=0.302969-0
=0.302969
$$

$$
\frac{\partial L}{\partial z_{\text{love}}}
=0.345030-0
=0.345030
$$

$$
\frac{\partial L}{\partial z_{\text{deep}}}
=0.352000-1
=-0.648000
$$

所以：

$$
\boxed{
\frac{\partial L}{\partial z}
=
[0.302969,\ 0.345030,\ -0.648000]
}
$$

## 6. 更新三个输出向量

对于每个候选词：

$$
\frac{\partial L}{\partial u_i}
=(P_i-y_i)v_{\text{love}}
$$

### 更新 $u_I$

梯度：

$$
\frac{\partial L}{\partial u_I}
=
0.302969
\begin{bmatrix}
0.5\\0.3
\end{bmatrix}
=
\begin{bmatrix}
0.151485\\
0.090891
\end{bmatrix}
$$

更新：

$$
u_I^{new}
=
\begin{bmatrix}0.2\\0.1\end{bmatrix}
-
0.1
\begin{bmatrix}0.151485\\0.090891\end{bmatrix}
$$

$$
\boxed{
u_I^{new}=
\begin{bmatrix}
0.184852\\
0.090911
\end{bmatrix}
}
$$

### 更新 $u_{\text{love}}$

梯度：

$$
\frac{\partial L}{\partial u_{\text{love}}}
=
0.345030
\begin{bmatrix}0.5\\0.3\end{bmatrix}
=
\begin{bmatrix}0.172515\\0.103509\end{bmatrix}
$$

更新：

$$
u_{\text{love}}^{new}
=
\begin{bmatrix}0.4\\0.2\end{bmatrix}
-
0.1
\begin{bmatrix}0.172515\\0.103509\end{bmatrix}
$$

$$
\boxed{
u_{\text{love}}^{new}
=
\begin{bmatrix}
0.382748\\
0.189649
\end{bmatrix}
}
$$

### 更新 $u_{\text{deep}}$

梯度：

$$
\frac{\partial L}{\partial u_{\text{deep}}}
=
-0.648000
\begin{bmatrix}0.5\\0.3\end{bmatrix}
$$

$$
=
\begin{bmatrix}
-0.324000\\
-0.194400
\end{bmatrix}
$$

更新：

$$
u_{\text{deep}}^{new}
=
\begin{bmatrix}0.2\\0.6\end{bmatrix}
-
0.1
\begin{bmatrix}-0.324000\\-0.194400\end{bmatrix}
$$

$$
\boxed{
u_{\text{deep}}^{new}
=
\begin{bmatrix}
0.232400\\
0.619440
\end{bmatrix}
}
$$

## 7. 更新中心词的输入向量

中心词向量也需要更新。

梯度公式：

$$
\frac{\partial L}{\partial v_{\text{love}}}
=
\sum_i(P_i-y_i)u_i
$$

这里使用的是**更新前的输出向量**。

代入：

$$
\frac{\partial L}{\partial v_{\text{love}}}
=
0.302969
\begin{bmatrix}0.2\\0.1\end{bmatrix}
+
0.345030
\begin{bmatrix}0.4\\0.2\end{bmatrix}
-
0.648000
\begin{bmatrix}0.2\\0.6\end{bmatrix}
$$

分别计算两个维度：

第一维：

$$
0.302969(0.2)+0.345030(0.4)-0.648000(0.2)
$$

$$
\approx0.060594+0.138012-0.129600
$$

$$
\approx0.069006
$$

第二维：

$$
0.302969(0.1)+0.345030(0.2)-0.648000(0.6)
$$

$$
\approx0.030297+0.069006-0.388800
$$

$$
\approx-0.289497
$$

所以：

$$
\frac{\partial L}{\partial v_{\text{love}}}
\approx
\begin{bmatrix}
0.069006\\
-0.289497
\end{bmatrix}
$$

更新：

$$
v_{\text{love}}^{new}
=
\begin{bmatrix}0.5\\0.3\end{bmatrix}
-
0.1
\begin{bmatrix}0.069006\\-0.289497\end{bmatrix}
$$

最终得到：

$$
\boxed{
v_{\text{love}}^{new}
=
\begin{bmatrix}
0.493099\\
0.328950
\end{bmatrix}
}
$$

## 8. 更新后，重新计算预测结果

现在使用更新后的所有向量，重新计算点积。

$$
z_I^{new}
=(u_I^{new})^Tv_{\text{love}}^{new}
\approx0.121055
$$

$$
z_{\text{love}}^{new}
=(u_{\text{love}}^{new})^Tv_{\text{love}}^{new}
\approx0.251118
$$

$$
z_{\text{deep}}^{new}
=(u_{\text{deep}}^{new})^Tv_{\text{love}}^{new}
\approx0.318361
$$

因此：

$$
z^{new}\approx[0.121055,\ 0.251118,\ 0.318361]
$$

重新计算 Softmax：

$$
\boxed{
P^{new}\approx[0.297884,\ 0.339260,\ 0.362857]
}
$$

正确答案 `deep` 的概率从：

$$
0.352000\longrightarrow0.362857
$$

损失从：

$$
1.044123\longrightarrow1.013747
$$

**这一次更新之后，正确答案的预测概率提高了，损失也降低了。**

---

## 最后，把这次完整运算汇总起来

| 步骤 | 计算结果 |
|---|---|
| 初始点积 | $[0.13,\ 0.26,\ 0.28]$ |
| 初始 Softmax | $[0.302969,\ 0.345030,\ 0.352000]$ |
| 初始损失 | $1.044123$ |
| 计算梯度 | $[0.302969,\ 0.345030,\ -0.648000]$ |
| 更新三个输出向量 | 得到新的 $u_I,\ u_{\text{love}},\ u_{\text{deep}}$ |
| 更新中心词向量 | $v_{\text{love}}=[0.493099,\ 0.328950]$ |
| 更新后 Softmax | $[0.297884,\ 0.339260,\ 0.362857]$ |
| 更新后损失 | $1.013747$ |

以上是**标准 Skip-gram + 完整 Softmax + 交叉熵 + 一次梯度下降更新**的数值示例。计算过程中，更新所有参数时都使用更新前的梯度，最后再统一计算新的预测概率。
