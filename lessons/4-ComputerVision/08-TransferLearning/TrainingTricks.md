# 深度学习训练技巧

随着神经网络变得更深，它们的训练过程变得越来越具有挑战性。其中一个主要问题是所谓的[消失梯度](https://en.wikipedia.org/wiki/Vanishing_gradient_problem)或 [爆炸梯度](https://deepai.org/machine-learning-glossary-and-terms/exploding-gradient-problem#:~:text=Exploding%20gradients%20are%20a%20problem,updates%20are%20small%20and%20controlled.)。[这篇文章](https://towardsdatascience.com/the-vanishing-exploding-gradient-problem-in-deep-neural-networks-191358470c11)在这些问题上提供了很好的介绍。

为了使训练深度网络更加高效，可以使用一些技巧。

## 保持值在合理的范围内

为了使数值计算更加稳定，我们希望确保神经网络中的所有值在合理的范围内，通常为[-1..1]或[0..1]。这并不是一个非常严格的要求，但浮点计算的性质使得不同数量级的值无法准确地一起操作。例如，如果我们将10<sup>-10</sup>和10<sup>10</sup>相加，我们可能会得到10<sup>10</sup>，因为较小的值会被"转换"为与较大值相同的数量级，从而丢失尾数。大多数激活函数在[-1..1]范围内具有非线性性，因此将所有输入数据缩放到[-1..1]或[0..1]间隔是有道理的。

## 初始权重初始化

理想情况下，我们希望在通过网络层之后的值在同一范围内。因此，重要的是以一种能够保持值分布的方式来初始化权重。

正态分布**N(0,1)**并不是一个好主意，因为如果我们有*n*个输入，输出的标准差将为*n*，值很可能跳出[0..1]间隔。

通常使用以下初始化方法：* 均匀分布 - `uniform`
* **N(0,1/n)** - 高斯分布
* **N(0,1/&radic;n_in)** 保证了均值为零、标准差为1的输入的均值/标准差保持不变
* **N(0,&radic;2/(n_in+n_out))** - 所谓的Xavier初始化(`glorot`)，它有助于在前向和反向传播过程中保持信号在范围内。

## 批量归一化

即使进行了适当的权重初始化，权重在训练过程中可能会变得非常大或非常小，并且会导致信号超出适当的范围。我们可以使用一种**归一化**技术将信号恢复正常。虽然有几种归一化技术（权重归一化，层归一化），但最常用的是批量归一化。

**批量归一化**的思想是考虑整个小批量的所有值，并根据这些值进行归一化（即减去均值，除以标准差）。它是作为网络层实现的，在应用权重之后但在激活函数之前进行归一化。结果是我们很可能会看到更高的最终准确度和更快的训练速度。以下是关于批归一化的[原始论文](https://arxiv.org/pdf/1502.03167.pdf)，[维基百科的解释](https://en.wikipedia.org/wiki/Batch_normalization)，以及[一篇很好的入门博客文章](https://towardsdatascience.com/batch-normalization-in-3-levels-of-understanding-14c2da90a338)（俄文版在[这里](https://habrahabr.ru/post/309302/)）。

## Dropout

**Dropout** 是一种有趣的技术，它在训练过程中随机删除了一定比例的神经元。它也被实现为一个具有一个参数的层（要删除的神经元的百分比，通常为10%-50%），并且在训练过程中，它将输入向量的随机元素置零，然后将其传递给下一层。

虽然这听起来可能很奇怪，但你可以在[`Dropout.ipynb`](Dropout.ipynb)笔记本中看到dropout在训练MNIST数字分类器时的效果。它可以加快训练速度，并在较少的训练周期内实现更高的准确性。

这种效果可以用多种方式解释：* 它可以被视为模型的一个随机冲击因素，使得优化结果不再陷入局部最小值
* 它可以被视为“隐式模型平均”，因为我们可以说在dropout过程中我们训练了稍微不同的模型

> *有些人说，当一个喝醉了的人试图学习一些东西时，与一个清醒的人相比，他第二天会更好地记住这些内容，因为具有一些功能障碍的神经元更能适应理解的含义。我们从未亲身验证过这是否为真。*

## 防止过拟合

深度学习中非常重要的一个方面就是能够防止[过拟合](../../3-NeuralNetworks/05-Frameworks/Overfitting.md)。虽然使用非常强大的神经网络模型可能很诱人，但我们应该始终在模型参数数量和训练样本数量之间保持平衡。

> 确保你理解我们以前介绍过的[过拟合](../../3-神经网络/05-框架/Overfitting.md)的概念！

有几种方法可以防止过拟合：

* 提前停止 -- 持续监测验证集上的错误，并在验证错误开始增加时停止训练。
* 显式权重衰减/正则化 -- 在损失函数中增加额外的惩罚，对于权重的绝对值较高的情况，可以防止模型得到非常不稳定的结果。
* 模型平均化 -- 训练多个模型，然后对结果进行平均。这有助于最小化方差。
* 丢弃（隐式模型平均）

## 优化器/训练算法

训练的另一个重要方面是选择好的训练算法。虽然经典的**梯度下降**是一个合理的选择，但有时可能速度太慢，或导致其他问题。

在深度学习中，我们使用**随机梯度下降**（SGD），它是对小批量数据应用梯度下降算法，这些数据是随机从训练集中选择的。通过以下公式调整权重：

w<sup>t+1</sup> = w<sup>t</sup> - &eta;&nabla;&lagran;

### 动量

在**动量随机梯度下降**中，我们保留了先前步骤中一部分梯度的信息。这类似于我们带着惯性朝某个方向移动，然后在不同的方向上受到冲击时，我们的轨迹不会立即改变，而是保持一部分原始运动的一种现象。在这里，我们引入另一个向量v来表示*速度*：
* v<sup>t+1</sup> = γv<sup>t</sup> - η∇L
* w<sup>t+1</sup> = w<sup>t</sup>+v<sup>t+1</sup>

这里的参数 γ 表示我们在多大程度上考虑惯性：γ=0 对应经典的SGD，γ=1 是一个纯运动方程。

### Adam, Adagrad, 等等。

由于在每一层中，我们通过一些矩阵W<sub>i</sub>来乘以信号，这取决于 ||W<sub>i</sub>||，梯度可能会逐渐减小并接近0，或者无限增加。这是梯度爆炸/消失问题的本质。解决这个问题的一个方法是在方程中只使用梯度的方向，而忽略绝对值，即

w^(t+1) = w^t - η (∇𝓁/||∇𝓁||), 其中 ||∇𝓁|| = √∑(∇𝓁)²

这个算法被称为**Adagrad**。另外两个使用相同思想的算法是**RMSProp**和**Adam**。

> **Adam** 被认为是很多应用中非常高效的算法，所以如果你不确定要使用哪个算法，就用 Adam。

### 梯度裁剪

梯度裁剪是上述思想的一种扩展。当 ||∇𝓁|| ≤ θ 时，我们考虑权重优化中的原始梯度，而当 ||∇𝓁|| > θ 时，我们将梯度除以其范数。其中θ是一个参数，在大多数情况下，我们可以取θ=1或θ=10。

### 学习率衰减

训练成功往往取决于学习率参数η。可以合理地假设η的较大值会导致更快的训练，这是我们通常在训练初期所希望的，而较小的η值则允许我们对网络进行微调。因此，在大多数情况下，我们希望在训练过程中减小η。

可以通过在每个训练周期后将η乘以某个数字（例如0.98），或使用更复杂的学习率调度方式来实现。

## 不同的网络架构

选择适合你的问题的网络架构可能会很棘手。通常，我们选择已经被证明对我们的特定任务（或类似任务）有效的架构。这里有一个[好的概述](https://www.topbots.com/a-brief-history-of-neural-network-architectures/)关于计算机视觉中的神经网络架构。

> 选择的架构对于我们拥有的训练样本数量来说足够强大至关重要。选择过于强大的模型可能会导致[过拟合](../../3-NeuralNetworks/05-Frameworks/Overfitting.md)

另一种好的方法是使用能自动调整所需复杂性的架构。在一定程度上，**ResNet**架构和**Inception**架构是自适应的。[更多关于计算机视觉架构的信息](../07-ConvNets/CNN_Architectures.md)