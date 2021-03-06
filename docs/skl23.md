# 5.23 核外学习 - 用于语义分析的大规模文本分类

## 可扩展性问题

`sklearn.feature_extraction.text.CountVectorizer`和`sklearn.feature_extraction.text.TfidfVectorizer`类受到许多可伸缩性问题的困扰，这些问题都源于`vocabulary_`属性（Python 字典）的内部使用，它用于将 unicode 字符串特征名称映射为整数特征索引。

主要的可扩展性问题是：

+   文本向量化程序的内存使用情况：所有特征的字符串表示形式都加载到内存中
+   文本特征提取的并行化问题：`vocabulary_`是一个共享状态：复杂的同步和开销
+   不可能进行在线或核外/流式学习：`vocabulary_`需要从数据中学习：在遍历一次整个数据集之前无法知道其大小

为了更好地理解这个问题，让我们看一下`vocabulary_`属性的工作原理。 在`fit`的时候，语料库的标记由整数索引唯一标识，并且该映射存储在词汇表中：

```py
from sklearn.feature_extraction.text import CountVectorizer

vectorizer = CountVectorizer(min_df=1)

vectorizer.fit([
    "The cat sat on the mat.",
])
vectorizer.vocabulary_
```

在`transform`的时候，使用词汇表来构建出现矩阵：

```py
X = vectorizer.transform([
    "The cat sat on the mat.",
    "This cat is a nice cat.",
]).toarray()

print(len(vectorizer.vocabulary_))
print(vectorizer.get_feature_names())
print(X)
```

让我们用稍大的语料库重新拟合：

```py
vectorizer = CountVectorizer(min_df=1)

vectorizer.fit([
    "The cat sat on the mat.",
    "The quick brown fox jumps over the lazy dog.",
])
vectorizer.vocabulary_
```

`vocabulary_`随着训练语料库的大小而（以对数方式）增长。 请注意，我们无法在 2 个文本文档上并行构建词汇表，因为它们共享一些单词，因此需要某种共享数据结构或同步障碍，这对于设定来说很复杂，特别是如果我们想要将处理过程分发给集群的时候。

有了这个新的词汇表，输出空间的维度现在变大了：

```py
X = vectorizer.transform([
    "The cat sat on the mat.",
    "This cat is a nice cat.",
]).toarray()

print(len(vectorizer.vocabulary_))
print(vectorizer.get_feature_names())
print(X)
```

## IMDB 电影数据集

为了说明基于词汇的向量化器的可扩展性问题，让我们为经典文本分类任务加载更真实的数据集：文本文档的情感分析。目标是从[互联网电影数据库](http://www.imdb.com/)（IMDb）中区分出积极的电影评论。

在接下来的章节中，使用了 Maas 等人收集的来自 IMDb 的电影评论的[大型子集](http://ai.stanford.edu/~amaas/data/sentiment/)。

> A. L. Maas, R. E. Daly, P. T. Pham, D. Huang, A. Y. Ng, and C. Potts. Learning Word Vectors for Sentiment Analysis. In the proceedings of the 49th Annual Meeting of the Association for Computational Linguistics: Human Language Technologies, pages 142–150, Portland, Oregon, USA, June 2011. Association for Computational Linguistics.

该数据集包含 50,000 个电影评论，分为 25,000 个培训样本和 25,000 个测试样本。评论标记为负面（neg）或正面（pos）。此外，正面意味着电影在 IMDb 上收到`> 6`星；负面意味着电影收到`<5`星。

假设`../fetch_data.py`脚本成功运行，以下文件应该可用：

```py
import os

train_path = os.path.join('datasets', 'IMDb', 'aclImdb', 'train')
test_path = os.path.join('datasets', 'IMDb', 'aclImdb', 'test')
```

现在，让我们通过 scikit-learn 的`load_files`函数，将它们加载到我们的活动会话中：

```py
from sklearn.datasets import load_files

train = load_files(container_path=(train_path),
                   categories=['pos', 'neg'])

test = load_files(container_path=(test_path),
                  categories=['pos', 'neg'])
```

> 注
> 
> 由于电影数据集由 50,000 个单独的文本文件组成，因此执行上面的代码片段可能需要约 20 秒或更长时间。

`load_files`函数将数据集加载到`sklearn.datasets.base.Bunch`对象中，这些对象是 Python 字典：

```py
train.keys()
```

特别是，我们只对`data`和`target`数组感兴趣。

```py
import numpy as np

for label, data in zip(('TRAINING', 'TEST'), (train, test)):
    print('\n\n%s' % label)
    print('Number of documents:', len(data['data']))
    print('\n1st document:\n', data['data'][0])
    print('\n1st label:', data['target'][0])
    print('\nClass names:', data['target_names'])
    print('Class count:', 
          np.unique(data['target']), ' -> ',
          np.bincount(data['target']))
```

正如我们在上面所看到的，`target`数组由整数 0 和 1 组成，其中 0 代表负面，1 代表正面。

## 哈希技巧

回忆一下，使用基于词汇表的向量化器的词袋表示：

![](../img/bag_of_words.svg)

要解决基于词汇表的向量化器的局限性，可以使用散列技巧。 我们可以使用散列函数和模运算，而不是在 Python 字典中构建和存储特征名称到特征索引的显式映射：

对于哈希技巧的原始论文的更多信息和参考，请见[以下网站](http://www.hunch.net/~jl/projects/hash_reps/index.html)，以及特定于语言的描述请见[这里](http://blog.someben.com/2013/01/hashing-lang/)。

```py
from sklearn.utils.murmurhash import murmurhash3_bytes_u32

# encode for python 3 compatibility
for word in "the cat sat on the mat".encode("utf-8").split():
    print("{0} => {1}".format(
        word, murmurhash3_bytes_u32(word, 0) % 2 ** 20))
```

这种映射完全是无状态的，并且输出空间的维度预先明确固定（这里我们使用`2 ** 20`的模，这意味着大约 1M 的维度）。 这使得有可能解决基于词汇表的向量化器的局限性，既可用于并行化，也可用于在线/核外学习。

`HashingVectorizer`类是`CountVectorizer`（或`use_idf=False`的`TfidfVectorizer`类）的替代品，它在内部使用 murmurhash 哈希函数：

```py
from sklearn.feature_extraction.text import HashingVectorizer

h_vectorizer = HashingVectorizer(encoding='latin-1')
h_vectorizer
```

它共享相同的“预处理器”，“分词器”和“分析器”基础结构：

```py
analyzer = h_vectorizer.build_analyzer()
analyzer('This is a test sentence.')
```

我们可以将数据集向量化为`scipy`稀疏矩阵，就像我们使用`CountVectorizer`或`TfidfVectorizer`一样，除了我们可以直接调用`transform`方法：没有必要拟合，因为`HashingVectorizer`是无状态变换器：

```py
docs_train, y_train = train['data'], train['target']
docs_valid, y_valid = test['data'][:12500], test['target'][:12500]
docs_test, y_test = test['data'][12500:], test['target'][12500:]
```

默认情况下，输出的维度事先固定为`n_features = 2 ** 20`（接近 1M 个特征），来最大限度地减少大多数分类问题的碰撞率，同时具有合理大小的线性模型（`coef_`属性中的 1M 权重）：

```py
h_vectorizer.transform(docs_train)
```

现在，让我们将`HashingVectorizer`的计算效率与`CountVectorizer`进行比较：

```py
h_vec = HashingVectorizer(encoding='latin-1')
%timeit -n 1 -r 3 h_vec.fit(docs_train, y_train)

count_vec =  CountVectorizer(encoding='latin-1')
%timeit -n 1 -r 3 count_vec.fit(docs_train, y_train)
```

我们可以看到，在这种情况下，`HashingVectorizer`比`Countvectorizer`快得多。

最后，让我们在 IMDb 训练子集上训练一个`LogisticRegression`分类器：

```py
from sklearn.linear_model import LogisticRegression
from sklearn.pipeline import Pipeline

h_pipeline = Pipeline([
    ('vec', HashingVectorizer(encoding='latin-1')),
    ('clf', LogisticRegression(random_state=1)),
])

h_pipeline.fit(docs_train, y_train)

print('Train accuracy', h_pipeline.score(docs_train, y_train))
print('Validation accuracy', h_pipeline.score(docs_valid, y_valid))

import gc

del count_vec
del h_pipeline

gc.collect()
```

## 核外学习

核外学习是在不放不进内存或 RAM 的数据集上训练机器学习模型的任务。 这需要以下条件：

具有固定输出维度的特征提取层
提前知道所有类别的列表（在这种情况下，我们只有正面和负面的评论）
支持增量学习的机器学习算法（scikit-learn 中的`partial_fit`方法）。

在以下部分中，我们将建立一个简单的批量训练函数来迭代地训练`SGDClassifier`。

但首先，让我们将文件名加载到 Python 列表中：

```py
train_path = os.path.join('datasets', 'IMDb', 'aclImdb', 'train')
train_pos = os.path.join(train_path, 'pos')
train_neg = os.path.join(train_path, 'neg')

fnames = [os.path.join(train_pos, f) for f in os.listdir(train_pos)] +\
         [os.path.join(train_neg, f) for f in os.listdir(train_neg)]

fnames[:3]
```

接下来，让我们创建目标标签数组：

```py
y_train = np.zeros((len(fnames), ), dtype=int)
y_train[:12500] = 1
np.bincount(y_train)
```

现在，我们实现`batch_train`函数，如下所示：

```py
from sklearn.base import clone

def batch_train(clf, fnames, labels, iterations=25, batchsize=1000, random_seed=1):
    vec = HashingVectorizer(encoding='latin-1')
    idx = np.arange(labels.shape[0])
    c_clf = clone(clf)
    rng = np.random.RandomState(seed=random_seed)
    
    for i in range(iterations):
        rnd_idx = rng.choice(idx, size=batchsize)
        documents = []
        for i in rnd_idx:
            with open(fnames[i], 'r', encoding='latin-1') as f:
                documents.append(f.read())
        X_batch = vec.transform(documents)
        batch_labels = labels[rnd_idx]
        c_clf.partial_fit(X=X_batch, 
                          y=batch_labels, 
                          classes=[0, 1])
      
    return c_clf
```

请注意，我们没有像上一节中那样使用`LogisticRegression`，但我们将使用具有 logistic 成本函数的`SGDClassifier`。 `SGD`代表随机梯度下降，这是一种优化算法，它逐样本迭代地优化权重系数，这允许我们一块一块地将数据馈送给分类器。

我们训练`SGDClassifier`；使用`batch_train`函数的默认设置，它将在`25 * 1000 = 25000`个文档上训练分类器。 （根据你的机器，这可能需要`>2`分钟）

```py
from sklearn.linear_model import SGDClassifier

sgd = SGDClassifier(loss='log', random_state=1, max_iter=1000)

sgd = batch_train(clf=sgd,
                  fnames=fnames,
                  labels=y_train)
```

最后，让我们评估一下它的表现：

```py
vec = HashingVectorizer(encoding='latin-1')
sgd.score(vec.transform(docs_test), y_test)
```

## 哈希向量化器的限制

使用Hashing Vectorizer可以实现流式和并行文本分类，但也可能会引入一些问题：

+   碰撞会在数据中引入太多噪声并降低预测质量，
+   `HashingVectorizer`不提供“反向文档频率”重新加权（缺少`use_idf=True`选项）。
+   没有反转映射，和从特征索引中查找特征名称的简单方法。
+   可以通过增加`n_features`参数来控制冲突问题。

可以通过在向量化器的输出上附加`TfidfTransformer`实例来重新引入 IDF 加权。然而，用于特征重新加权的`idf_`统计量的计算，需要在能够开始训练分类器之前，额外遍历训练集至少一次：这打破了在线学习方案。

缺少逆映射（`TfidfVectorizer`的`get_feature_names()`方法）更难以解决。这将需要扩展`HashingVectorizer`类来添加“跟踪”模式，来记录最重要特征的映射，来提供统计调试信息。

在调试特征提取问题的同时，建议在数据集的小型子集上使用`TfidfVectorizer(use_idf=False)`，来模拟具有`get_feature_names()`方法且没有冲突问题的`HashingVectorizer()`实例。

> 练习
> 
> 在我们上面的`batch_train`函数的实现中，我们在每次迭代中随机抽取`k`个训练样本作为批量，这可以被视为带放回的随机子采样。 你可以修改`batch_train`函数，使它无放回地迭代文档，即它在每次迭代中使用每个文档一次。

```py
# %load solutions/23_batchtrain.py
```
