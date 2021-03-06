*注：课程笔记地址：

[CS231n]: http://cs231n.github.io/convolutional-networks/

# 卷积神经网络（**CNNs / ConvNets**）



[TOC]

卷积神经网络和上一章讲的常规神经网络非常相似：它们都是由神经元组成，神经元中有具有学习能力的权重和偏差。每个神经元都得到一些输入数据，进行内积（dot product）运算后再进行非线性运算。整个网络依旧是一个可导的评分函数：该函数的输入是原始的图像像素，输出是不同类别的评分。在最后一层（往往是全连接层），网络依旧有一个损失函数（比如SVM或Softmax），并且在神经网络中我们实现的各种技巧和要点依旧适用于卷积神经网络。

那么有哪些地方变化了呢？卷积神经网络的结构基于一个假设，即输入数据是图像，基于该假设，我们就向结构中添加了一些特有的性质。这些特有属性使得前向传播函数实现起来更高效，并且大幅度降低了网络中参数的数量。

## **结构概述**

***回顾：常规神经网络***

在上一章中，神经网络的输入是一个向量，然后在一系列的 ***隐层*** 中对它做变换。每个隐层都是由若干的神经元组成，每个神经元都与前一层中的所有神经元连接。但是在一个隐层中，神经元相互独立不进行任何连接。最后的全连接层被称为“输出层”，在分类问题中，它输出的值被看做是不同类别的评分值。

***常规神经网络对于大尺寸图像效果不尽人意***。在CIFAR-10中，图像的尺寸是32x32x3（宽高均为32像素，3个颜色通道），对应的常规神经网络的第一个隐层中，每一个单独的全连接神经元就有32x32x3=3072个权重。（*参考ppt 5-26，5-27*）

![1.1](./image/1_1.png)

这个数量看起来还可以接受，但是很显然这个全连接的结构不适用于更大尺寸的图像。举例说来，一个尺寸为200x200x3的图像，会让神经元包含200x200x3=120,000个权重值。而网络中肯定不止一个神经元，那么参数的量就会快速增加！显而易见，这种全连接方式效率低下，大量的参数也很快会导致网络过拟合。

***神经元的三维排列***。卷积神经网络针对输入全部是图像的情况，将结构调整得更加合理，获得了不小的优势。与常规神经网络不同，卷积神经网络的各层中的神经元是3维排列的：**宽度**、**高度**和**深度**（这里的**深度**指的是输入数据的第三个维度，而不是整个网络的深度，整个网络的深度指的是网络的层数）。举个例子，CIFAR-10中的图像是作为卷积神经网络的输入，该数据体的维度是32x32x3（宽度为32，高度为32，深度为3）。

![1-2](./image/1-2.png)

卷积神经网络中，卷积层中的神经元只与前一层中的一小块区域连接，而不是采用全连接的方式。卷积层中有一个滤波器(filter)，通过该滤波器(filter)会滑动全部覆盖输入图像，并与输入图像对应区域进行点积（dot products）操作，下图所示：

![1-3](./image/1-3.png)

该滤波器（filter）一般选择 3×3 或 5×5大小，深度与输入相同。下图中，输入图像深度为3，则filter的深度也为3.

![1-4](./image/1-4.png)

在卷积层中，我们将上图所示，5×5×3的filter记作$w$,一次点积（dot products）会输出一个数，该数是filter与输入图像对应区域的3D数据对应乘积相加，结果再加上一个偏置而得出：

![1-5](./image/1-5.png)

一个32×32×3的输入图像经过点积（dot products）运算后，会输出一个28 ×28的激活图（下图所示），该激活图会作为下一层的输入。

![1-6](./image/1-6.png)

上图是只有一个filter，得到一个激活图，若有两个filter则会输出两个独立的激活图：

![1-7](./image/1-7.png)

比如说，一个卷积层中有6个5×5的filter，我们将得到6个独立的激活图，这6个独立的激活图会沿着深度方向存储在一起，这样我们就得到了一副28×28×6的“新的图片”：

![1-8](./image/1-8.png)

对于用来分类CIFAR-10中图像的卷积网络，其最后的输出层的维度是1x1x10，这是因为在卷积神经网络结构的最后部分将会把全尺寸的图像压缩为包含分类评分的一个向量，向量是在深度方向排列的。

![1-3](./image/1-3.jpg)

左边是一个3层的神经网络。右边是一个卷积神经网络，图例中网络将它的神经元都排列成3个维度（宽、高和深度）。卷积神经网络每一层都将3D输入图像转变为神经元的3D激活图（参考ppt 5-30）。在这个例子中，红色的输入层装的是图像，所以它的宽度和高度就是图像的宽度和高度，它的深度是3（代表了R、G、B 3种颜色通道）。

------

> 卷积神经网络是由层组成的。每一层都有一个简单的API：用一些含或者不含参数的可导的函数，将输入的3D数据变换为3D的输出数据。

------

## **用来构建卷积网络的各种层**

一个简单的卷积神经网络是由各种层按照一定顺序排列组成，网络中的每个层通过可微函数将激活数据从一个层传递到另一个层。卷积神经网络主要由三种类型的层构成：**卷积层**，**池化（Pooling）层**和**全连接层**（全连接层和常规神经网络中的相同）。通过将这些层安一定顺序叠加起来，就可以构建一个完整的卷积神经网络。

*网络结构例子：*

一个用于CIFAR-10图像数据分类的卷积神经网络结构可以是[输入层-卷积层-ReLU层-汇聚层-全连接层]。

![1-9](./image/1-9.png)

细节如下：

- 输入[32x32x3]存有图像的原始像素值，本例中图像宽高均为32，有3个颜色通道。
- 卷积层中，神经元与输入层中的一个局部区域相连，每个神经元都计算自己与输入层相连的小区域与自己权重的点积(dot products)。卷积层会计算所有神经元的输出。如果我们使用10个滤波器（filter,也叫作核），得到的输出数据体的维度就是[32x32x10]。
- ReLU层将会逐个元素地进行激活函数操作，比如使用以0为阈值的![max(0,x)](https://www.zhihu.com/equation?tex=max%280%2Cx%29)作为激活函数。该层对数据尺寸没有改变，还是[32x32x10]。
- 汇聚层在在空间维度（宽度和高度）上进行降采样（downsampling）操作，数据尺寸变为[16x16x10]。
- 全连接层将会计算分类评分，数据尺寸变为[1x1x10]，其中10个数字对应的就是CIFAR-10中10个类别的分类评分值。正如其名，全连接层与常规神经网络一样，其中每个神经元都与前一层中所有神经元相连接。

由此看来，卷积神经网络一层一层地将图像从原始像素值变换成最终的分类评分值。其中有的层含有参数，有的没有。具体说来，卷积层和全连接层（CONV/FC）对输入执行变换操作的时候，不仅会用到激活函数，还会用到很多参数（神经元的突触权值和偏差）。而ReLU层和汇聚层则是进行一个固定不变的函数操作。卷积层和全连接层中的参数会随着梯度下降被训练，这样卷积神经网络计算出的分类评分就能和训练集中的每个图像的标签吻合了。

**小结**：

- 简单案例中卷积神经网络的结构，就是一系列的层将输入数据变换为输出数据（比如分类评分）。
- 卷积神经网络结构中有几种不同类型的层（目前最流行的有卷积层、全连接层、ReLU层和汇聚层）。
- 每个层的输入是3D数据，然后使用一个可导的函数将其变换为3D的输出数据。
- 有的层有参数，有的没有（卷积层和全连接层有，ReLU层和汇聚层没有）。
- 有的层有额外的超参数，有的没有（卷积层、全连接层和汇聚层有，ReLU层没有）。

![1-10](./image/1-10.png)

​                                                           一个卷积神经网络的激活输出例子

左边的输入层存有原始图像像素，右边的输出层存有类别分类评分。在处理流程中的每个激活数据体是铺成一列来展示的。因为对3D数据作图比较困难，我们就把每个数据体切成层，然后铺成一列显示。最后一层装的是针对不同类别的分类得分，这里只显示了得分最高的5个评分值和对应的类别。完整的[网页演示](https://link.zhihu.com/?target=http%3A//cs231n.stanford.edu/)在我们的课程主页。本例中的结构是一个小的VGG网络，VGG网络后面会有讨论。

### 卷积层

