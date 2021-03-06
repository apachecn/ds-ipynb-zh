# 5.3 数据表示和可视化

机器学习关于将模型拟合到数据；出于这个原因，我们首先讨论如何表示数据以便计算机理解。 除此之外，我们将基于上一节中的`matplotlib`示例构建，并展示如何可视化数据的一些示例。

## sklearn 中的数据

scikit-learn 中的数据（极少数例外）被假定存储为形状为`[n_samples, n_features]`的二维数组。许多算法也接受形状相同的`scipy.sparse`矩阵。

+   `n_samples`：样本数量：每个样本是要处理（例如分类）的项目。样本可以是文档，图片，声音，视频，天文对象，数据库中的行或 CSV 文件，或者你可以使用的一组固定数量的特征描述的任何内容。
+   `n_features`：特征或不同形状的数量，可用于以定量方式描述每个项目。特征通常是实值，但在某些情况下可以是布尔值或离散值。

必须事先固定特征的数量。然而，它可以是非常高的维度（例如数百万个特征），对于给定的样本，它们中的大多数是“零”。这是`scipy.sparse`矩阵可能有用的情况，因为它们比 NumPy 数组更具内存效率。

我们从上一节（或 Jupyter 笔记本）中回顾，我们将样本（数据点或实例）表示为数据数组中的行，并将相应的特征（“维度”）存储为列。

## 简单示例：鸢尾花数据集

作为简单数据集的一个例子，我们将看一下 scikit-learn 存储的鸢尾花数据。 数据包括三种不同鸢尾花的测量值。 在这个特定的数据集中有三种不同的鸢尾花，如下图所示：

| 物种 | 图像 |
| --- | --- |
| 山鸢尾 | ![](../img/iris_setosa.jpg) |
| 杂色鸢尾 | ![](../img/iris_versicolor.jpg) |
| 弗吉尼亚鸢尾 | ![](../img/iris_virginica.jpg) |

简单问题：

让我们假设我们有兴趣对新观测值进行分类; 我们想分别预测未知的花是 Iris-Setosa，Iris-Versicolor 还是 Iris-Virginica。 根据我们在上一节中讨论的内容，我们将如何构建这样的数据集？

记住：我们需要一个大小为`[n_samples x n_features]`的二维数组。

+   `n_samples`指代什么？
+   `n_features`可能指代什么？

请记住，每个样本必须有固定数量的特征，并且对于每个样本，特征编号`j`必须是同一种数量。

### 在 sklearn 中加载鸢尾花数据集

对于将来使用机器学习算法实验，我们建议你收藏 UCI 机器学习仓库，该仓库托管许多常用的数据集，这些数据集对于机器学习算法的基准测试非常有用 - 这是机器学习实践者和研究人员非常流行的资源。 方便的是，其中一些数据集已经包含在 scikit-learn 中，因此我们可以跳过下载，读取，解析和清理这些文本/ CSV 文件的繁琐部分。你可以在[这里](http://scikit-learn.org/stable/datasets/#toy-datasets)找到 scikit-learn 中可用数据集的列表。

如，scikit-learn 拥有这些鸢尾花物种的非常简单的数据集。 数据包括以下内容：

鸢尾花数据集中的特征：

+   萼片长度，厘米
+   萼片宽度，厘米
+   花瓣长度，厘米
+   花瓣宽度，厘米

要预测的目标类别：

+   山鸢尾
+   杂色鸢尾
+   弗吉尼亚鸢尾

![](../img/petal_sepal.jpg)

（图片来源：[“Petal-sepal”](https://commons.wikimedia.org/wiki/File:Petal-sepal.jpg#/media/File:Petal-sepal.jpg)。通过 Wikimedia Commons 在 CC BY-SA 3.0 下获得许可）

scikit-learn 自带了鸢尾花 CSV 文件的副本以及辅助函数，用于将其加载到`numpy`数组中：


```py
from sklearn.datasets import load_iris
iris = load_iris()
```

生成的数据集是一个`Bunch`对象：你可以使用方法`keys()`查看可用的内容：

```py
iris.keys()
```

每个花样本的特征都存储在数据集的`data`属性中：

```py
n_samples, n_features = iris.data.shape
print('Number of samples:', n_samples)
print('Number of features:', n_features)
# 第一个样本（第一朵花）的萼片长度，萼片宽度，花瓣长度和花瓣宽度
print(iris.data[0])
```

每个样本的类别信息存储在数据集的`target`属性中：

```py
print(iris.data.shape)
print(iris.target.shape)

print(iris.target)

import numpy as np

np.bincount(iris.target)
```

使用 NumPy 的`bincount`函数（上图），我们可以看到类别在这个数据集中均匀分布 - 每个物种有 50 朵花，其中：

+   类 0：山鸢尾
+   类 1：杂色鸢尾
+   类 2：弗吉尼亚鸢尾

这些类名存储在最后一个属性中，即`target_names`：

```py
print(iris.target_names)
```

这个数据是四维的，但我们可以使用简单的直方图或散点图一次可视化一个或两个维度。 再次，我们将从启用`matplotlib`内联模式开始：

```py
%matplotlib inline

import matplotlib.pyplot as plt
x_index = 3

for label in range(len(iris.target_names)):
    plt.hist(iris.data[iris.target==label, x_index], 
             label=iris.target_names[label],
             alpha=0.5)

plt.xlabel(iris.feature_names[x_index])
plt.legend(loc='upper right')
plt.show()

x_index = 3
y_index = 0

for label in range(len(iris.target_names)):
    plt.scatter(iris.data[iris.target==label, x_index], 
                iris.data[iris.target==label, y_index],
                label=iris.target_names[label])

plt.xlabel(iris.feature_names[x_index])
plt.ylabel(iris.feature_names[y_index])
plt.legend(loc='upper left')
plt.show()
```

> 练习
> 
> +    在上面的脚本中，更改`x_index`和`y_index`，找到两个参数的组合，最大限度地将这三个类分开。
> +    本练习是降维的预习，我们稍后会看到。

## 旁注：散点图矩阵

分析人员使用的常用工具称为散点图矩阵，而不是一次查看一个绘图。

散点图矩阵显示数据集中所有特征之间的散点图，以及显示每个特征分布的直方图。

```py
import pandas as pd
    
iris_df = pd.DataFrame(iris.data, columns=iris.feature_names)
pd.plotting.scatter_matrix(iris_df, c=iris.target, figsize=(8, 8));
```

## 其它可用的数据

[Scikit-learn 提供了大量用于测试学习算法的数据集。](http://scikit-learn.org/stable/datasets/#dataset-loading-utilities) 它们有三种形式：

+   打包数据：这些小数据集与 scikit-learn 安装打包在一起，可以使用`sklearn.datasets.load_ *`中的工具下载
+   可下载数据：这些较大的数据集可供下载，scikit-learn 包含简化此过程的工具。 这些工具可以在`sklearn.datasets.fetch_ *`中找到
+   生成的数据：有几个数据集是基于随机种子从模型生成的。 这些可以在`sklearn.datasets.make_ *`中找到

你可以使用 IPython 的制表符补全功能探索可用的数据集加载器，提取器和生成器。 从`sklearn`导入`datasets`子模块后，键入：

```
datasets.load_<TAB>
```

或者：

```
datasets.fetch_<TAB>
```

或者：

```
datasets.make_<TAB>
```

来查看可用函数列表。

```py
from sklearn import datasets
```

请注意：许多这些数据集非常庞大，可能需要很长时间才能下载！

如果你在 IPython 笔记本中开始下载并且想要将其删除，则可以使用 ipython 的“内核中断”功能，该功能可在菜单中使用或使用快捷键`Ctrl-m i`。

你可以按`Ctrl-m h`获取所有 ipython 键盘快捷键的列表。

## 加载数字数据

现在我们来看看另一个数据集，我们必须更多考虑如何表示数据。 我们可以采用与上述类似的方式探索数据：

```py
from sklearn.datasets import load_digits
digits = load_digits()

digits.keys()

n_samples, n_features = digits.data.shape
print((n_samples, n_features))

print(digits.data[0])
print(digits.target)
```

这里的目标只是数据所代表的数字。 数据是长度为 64 的数组......但这些数据意味着什么？

实际上有个线索，我们有两个版本的数据数组：数据和图像。 我们来看看它们：

```py
print(digits.data.shape)
print(digits.images.shape)
```


通过简单的形状改变，我们可以看到它们是相关的：

```py
import numpy as np
print(np.all(digits.images.reshape((1797, 64)) == digits.data))
```

让我们可视化数据。 它比我们上面使用的简单散点图更复杂，但我们可以很快地完成它。

```py
# 建立图形
fig = plt.figure(figsize=(6, 6))  # figure size in inches
fig.subplots_adjust(left=0, right=1, bottom=0, top=1, hspace=0.05, wspace=0.05)

# 绘制数字：每个图像是 8x8 像素
for i in range(64):
    ax = fig.add_subplot(8, 8, i + 1, xticks=[], yticks=[])
    ax.imshow(digits.images[i], cmap=plt.cm.binary, interpolation='nearest')
    
    # 用目标值标记图像
    ax.text(0, 7, str(digits.target[i]))
```

我们现在看到这些特征的含义。 每个特征是实数值，表示手写数字的 8×8 图像中的像素的暗度。

即使每个样本具有固有的二维数据，数据矩阵也将该 2D 数据展平为单个向量，该向量可以包含在数据矩阵的一行中。

> 练习：处理人脸数据集
> 
> 这里，我们将花点时间亲自探索数据集。 稍后我们将使用 Olivetti faces 数据集。 花点时间获取数据（大约 1.4MB），并可视化人脸。 你可以复制用于可视化上述数字的代码，并为此数据进行修改。

```py
from sklearn.datasets import fetch_olivetti_faces
# 获取人脸数据
# 使用上面的脚本绘制人脸图像数据。
# 提示：plt.cm.bone 是用于这个数据的很好的颜色表
```

答案：

```py
# %load solutions/03A_faces_plot.py
```
