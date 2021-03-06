# 5.21 无监督学习：非线性降维

## 流形学习

PCA 的一个弱点是它无法检测到非线性特征。 已经开发了一组称为流形学习的算法，来解决这个缺陷。流形学习中使用的规范数据集是 S 曲线：

```py
from sklearn.datasets import make_s_curve
X, y = make_s_curve(n_samples=1000)

from mpl_toolkits.mplot3d import Axes3D
ax = plt.axes(projection='3d')

ax.scatter3D(X[:, 0], X[:, 1], X[:, 2], c=y)
ax.view_init(10, -60);
```

这是一个嵌入三维的二维数据集，但它以某种方式嵌入，PCA 无法发现底层数据方向：

```py
from sklearn.decomposition import PCA
X_pca = PCA(n_components=2).fit_transform(X)
plt.scatter(X_pca[:, 0], X_pca[:, 1], c=y);
```

然而，`sklearn.manifold`子模块中可用的流形学习算法能够还原底层的二维流形：

```py
from sklearn.manifold import Isomap

iso = Isomap(n_neighbors=15, n_components=2)
X_iso = iso.fit_transform(X)
plt.scatter(X_iso[:, 0], X_iso[:, 1], c=y);
```

## 数字数据上的流形学习

我们可以将流形学习技术应用于更高维度的数据集，例如我们之前看到的数字数据：

```py
from sklearn.datasets import load_digits
digits = load_digits()

fig, axes = plt.subplots(2, 5, figsize=(10, 5),
                         subplot_kw={'xticks':(), 'yticks': ()})
for ax, img in zip(axes.ravel(), digits.images):
    ax.imshow(img, interpolation="none", cmap="gray")
```

我们可以使用线性技术（例如 PCA）可视化数据集。 我们看到这已经提供了一些数据的直觉：

```py
# 构建 PCA 模型
pca = PCA(n_components=2)
pca.fit(digits.data)
# 将数字数据转换为前两个主成分
digits_pca = pca.transform(digits.data)
colors = ["#476A2A", "#7851B8", "#BD3430", "#4A2D4E", "#875525",
          "#A83683", "#4E655E", "#853541", "#3A3120","#535D8E"]
plt.figure(figsize=(10, 10))
plt.xlim(digits_pca[:, 0].min(), digits_pca[:, 0].max() + 1)
plt.ylim(digits_pca[:, 1].min(), digits_pca[:, 1].max() + 1)
for i in range(len(digits.data)):
    # 实际上将数字绘制为文本而不是使用散点图
    plt.text(digits_pca[i, 0], digits_pca[i, 1], str(digits.target[i]),
             color = colors[digits.target[i]],
             fontdict={'weight': 'bold', 'size': 9})
plt.xlabel("first principal component")
plt.ylabel("second principal component");
```

但是，使用更强大的非线性技术可以提供更好的可视化效果。 在这里，我们使用 t-SNE 流形学习方法：

```py
from sklearn.manifold import TSNE
tsne = TSNE(random_state=42)
# 使用 fit_transform 而不是 fit，因为 TSNE 没有 fit 方法
digits_tsne = tsne.fit_transform(digits.data)

plt.figure(figsize=(10, 10))
plt.xlim(digits_tsne[:, 0].min(), digits_tsne[:, 0].max() + 1)
plt.ylim(digits_tsne[:, 1].min(), digits_tsne[:, 1].max() + 1)
for i in range(len(digits.data)):
    # 实际上将数字绘制为文本而不是使用散点图
    plt.text(digits_tsne[i, 0], digits_tsne[i, 1], str(digits.target[i]),
             color = colors[digits.target[i]],
             fontdict={'weight': 'bold', 'size': 9})
```

t-SNE 比其他流形学习算法运行时间更长，但结果非常惊人。 请记住，此算法纯粹是无监督的，并且不知道类标签。 它仍然能够很好地分离类别（尽管类 4 和 类 9 已被分成多个分组）。

> 练习
> 
> 将 isomap 应用于数字数据集的结果与 PCA 和 t-SNE 的结果进行比较。 你认为哪个结果看起来最好？
鉴于 t-SNE 很好地将类别分开，人们可能会试图将这个处理过程用于分类。 尝试在使用 t-SNE 转换的数字数据上，训练 K 最近邻分类器，并与没有任何转换的数据集上的准确性比较。

```py
# %load solutions/21A_isomap_digits.py

# %load solutions/21B_tsne_classification.py
```
