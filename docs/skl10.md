# 5.10 案例学习：泰坦尼克幸存者

## 特征提取

在这里，我们将讨论一个重要的机器学习：从数据中提取定量特征。 到本节结束时，你将

+   了解如何从现实世界数据中提取特征。
+   请参阅从文本数据中提取数值特征的示例

此外，我们将介绍 scikit-learn 中的几个基本工具，可用于完成上述任务。

## 特征是什么？

## 数值特征

回想一下 scikit-learn 中的数据应该是二维数组，大小为`n_samples×n_features`。

以前，我们查看了鸢尾花数据集，它有 150 个样本和 4 个特征。

```py
from sklearn.datasets import load_iris

iris = load_iris()
print(iris.data.shape)
```

这些特征是：

+   萼片长度，厘米
+   萼片宽度，厘米
+   花瓣长度，厘米
+   花瓣宽度，厘米

诸如此类的数值特征非常简单：每个样本都包含对应特征的浮点数列表。

## 类别特征

如果你有类别特征怎么办？ 例如，假设每个鸢尾花的颜色数据为：

```
color in [red, blue, purple]
```

> 译者注：这是个不恰当的例子，因为在计算机看来，颜色是离散的数值特征，拥有 RGB 三个分量。

你可能想为这些特征分配数字，即红色为 1，蓝色为 2，紫色为 3，但总的来说这是一个坏主意。 估计器倾向于假设，数值特征具有某些连续尺度，因此，例如，1 和 2 比 1 和 3 更相似，并且这通常不是类别特征的情况。

实际上，上面的例子是“类别”特征的子类别，即“标称”特征。 标称特征并不意味着有序，而“序数”特征是确实暗示顺序的分类特征。 序数特征的一个例子是 T 恤尺寸，例如`XL> L> M> S`。

将标称特征解析为防止分类算法断言顺序的格式的一种解决方法，是所谓的单热编码表示。 在这里，我们为每个类别提供自己的维度。

因此，在这种情况下，丰富的鸢尾花特征集是：

+   萼片长度，厘米
+   萼片宽度，厘米
+   花瓣长度，厘米
+   花瓣宽度，厘米
+   颜色为紫色（1.0 或 0.0）
+   颜色为红色（1.0 或 0.0）
+   颜色为蓝色（1.0 或 0.0）

请注意，使用许多这些类别特征可能会产生更好表示为稀疏矩阵的数据，我们将在下面的文本分类示例中看到。

## 使用`DictVectorizer`编码分类特征

当要编码的源数据有一个`dicts`列表，其中值是类别或数值的字符串名称时，你可以使用`DictVectorizer`类计算类别特征的布尔扩展，同时保持数值特征不受影响：

```py
measurements = [
    {'city': 'Dubai', 'temperature': 33.},
    {'city': 'London', 'temperature': 12.},
    {'city': 'San Francisco', 'temperature': 18.},
]

from sklearn.feature_extraction import DictVectorizer

vec = DictVectorizer()
vec

vec.fit_transform(measurements).toarray()

vec.get_feature_names()
```

## 衍生特征

另一个常见的特征类型是衍生特征，其中一些预处理步骤应用于数据来生成以某种方式提供更多信息的特征。 派生特征可以基于特征提取和降维（例如 PCA 或流形学习），可以是特征的线性或非线性组合（例如在多项式回归中），或者可以是特征的一些更复杂的变换。

## 组合数值和类别特征

作为如何使用分类和数字数据的一个例子，我们将为 HMS 泰坦尼克号的乘客进行生存预测。

我们将使用泰坦尼克号（`titanic3.xls`）[这里](http://biostat.mc.vanderbilt.edu/wiki/pub/Main/DataSets/titanic3.xls)的版本。 我们将`.xls`转换为`.csv`以便操作，但是数据保持不变。

我们需要读取（`titanic3.csv`）文件中的所有行，空出第一行的键，找到我们的标签（幸存或死亡）和数据（人的属性）。 让我们看看键和一些相应的示例行。

```py
import os
import pandas as pd

titanic = pd.read_csv(os.path.join('datasets', 'titanic3.csv'))
print(titanic.columns)
```

以下是键及其含义的广泛描述：

```
pclass          乘客等级
                (1 = 1st; 2 = 2nd; 3 = 3rd)
survival        是否幸存
                (0 = No; 1 = Yes)
name            名称
sex             性别
age             年龄
sibsp           船上的兄弟姐妹/配偶的数量
parch           父母/孩子数量
ticket          票号
fare            乘客票价
cabin           舱位
embarked        登船港口
                (C = Cherbourg; Q = Queenstown; S = Southampton)
boat            救生艇
body            身份证号
home.dest       家/目的地
```

一般来说，`name`, `sex`, `cabin`, `embarked`, `boat`, `body`, `homedest`可能是类别特征的候选，而其余似乎是数组特征。 我们还可以查看数据集中的前几行以便更好地理解：

```py
titanic.head()
```

我们显然希望丢弃`boat`和`body`列，以便将任何人分类为幸存者和非幸存者，因为它们已经包含此信息。 `name`对每个人（可能）是唯一的，也是非信息性的。 首次尝试中，我们将使用`pclass`，`sibsp`，`parch`，`fare`和`embarked`作为我们的特征：

```py
labels = titanic.survived.values
features = titanic[['pclass', 'sex', 'age', 'sibsp', 'parch', 'fare', 'embarked']]

features.head()
```

数据现在仅包含有用的特征，但它们不是机器学习算法可以理解的格式。 我们需要将字符串`male`和`female`转换为表示性别的二元变量，类似于`embarked`。 我们可以使用`pandas get_dummies`函数来实现：

```py
pd.get_dummies(features).head()
```

这个转换成功编码了字符串列。 但是，有人可能会认为`pclass`也是一个类别变量。 我们可以使用`columns`参数显式列出要编码的列，并包含`pclass`：

```py
features_dummies = pd.get_dummies(features, columns=['pclass', 'sex', 'embarked'])
features_dummies.head(n=16)

data = features_dummies.values

import numpy as np
np.isnan(data).any()
```

完成了所有困难的数据加载工作，对这些数据应用分类器变得简单明了。 建立最简单的模型，我们希望使用`DummyClassifier`看到最简单的得分。

```py
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import Imputer


train_data, test_data, train_labels, test_labels = train_test_split(
    data, labels, random_state=0)

imp = Imputer()
imp.fit(train_data)
train_data_finite = imp.transform(train_data)
test_data_finite = imp.transform(test_data)

np.isnan(train_data_finite).any()

from sklearn.dummy import DummyClassifier

clf = DummyClassifier('most_frequent')
clf.fit(train_data_finite, train_labels)
print("Prediction accuracy: %f"
      % clf.score(test_data_finite, test_labels))
```

> 练习
> 
> 尝试使用`LogisticRegression`和`RandomForestClassifier`而不是`DummyClassifier`执行上述分类
> 选择不同的特征子集会有帮助吗？

```py
# %load solutions/10_titanic.py
```
