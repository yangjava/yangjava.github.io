---
layout: post
categories: [Python,scikitlearn]
description: none
keywords: Python
---
# scikit-learn

## 什么是机器学习
机器学习是近年来的一大热门话题，然而其历史要倒推到半个多世纪之前。1959年Arthur Samuel给机器学习的定义是：`Field of study that gives computers the ability to learn without being explicitly programmed` 即让计算机在没有被显式编程的情况下，具备自我学习的能力。

Tom M.Mitchell在操作层面给出了更直观的定义：`A computer program is said to learn from experience E with respect to some class of tasks T and performance measure P，if its performance at tasks in T，as measured by P，improves with experience E.`

翻译过来用大白话来说就是：针对某件事情，计算机会从经验中学习，并且越做越好。从机器学习领域的先驱和“大牛”们的定义来看，我们可以自己总结出对机器学习的理解：机器学习是一个计算机程序，针对某个特定的任务，从经验中学习，并且越做越好。

从这个理解上，我们可以得出以下针对机器学习最重要的内容。
- 数据：经验最终要转换为计算机能理解的数据，这样计算机才能从经验中学习。谁掌握的数据量大、质量高，谁就占据了机器学习和人工智能领域最有利的资本。用人类来类比，数据就像我们的教育环境，一个人要变得聪明，一个很重要的方面是能享受到优质的教育。所以，从这个意义来讲，就能理解类似Google这种互联网公司开发出来的机器学习程序性能为什么那么好了，因为他们能获取到海量的数据。
- 模型：即算法，是本书要介绍的主要内容。有了数据之后，可以设计一个模型，让数据作为输入来训练这个模型。经过训练的模型，最终就成了机器学习的核心，使得模型成为了能产生决策的中枢。一个经过良好训练的模型，当输入一个新事件时，会做出适当的反应，产生优质的输出。

## 机器学习的分类
机器学习可以分成以下两类。

有监督学习（Supervised learning）通过大量已知的输入和输出相配对的数据，让计算机从中学习出规律，从而能针对一个新的输入做出合理的输出预测。比如，我们有大量不同特征（面积、地理位置、朝向、开发商等）的房子的价格数据，通过学习这些数据，预测一个已知特征的房子价格，这种称为回归学习（Regression learning），即输出结果是一个具体的数值，它的预测模型是一个连续的函数。再比如我们有大量的邮件，每个邮件都已经标记是否是垃圾邮件。通过学习这些已标记的邮件数据，最后得出一个模型，这个模型对新的邮件，能准确地判断出该邮件是否是垃圾邮件，这种称为分类学习（Classfication learning），即输出结果是离散的，即要么输出1表示是垃圾邮件，要么输出0表示不是垃圾邮件。

无监督学习（Unsupervised learning）通过学习大量的无标记的数据，去分析出数据本身的内在特点和结构。比如，我们有大量的用户购物的历史记录信息，从数据中去分析用户的不同类别。针对这个问题，我们最终能划分几个类别？每个类别有哪些特点？我们事先是不知道的。这个称为聚类（Clustering）。这里需要特别注意和有监督学习里的分类的区别，分类问题是我们已经知道了有哪几种类别；而聚类问题，是我们在分析数据之前其实是不知道有哪些类别的。即分类问题是在已知答案里选择一个，而聚类问题的答案是未知的，需要利用算法从数据里挖掘出数据的特点和结构。

这两种机器学习类别的最大区别是，有监督学习的训练数据里有已知的结果来“监督”；而无监督学习的训练数据里没有结果“监督”，不知道到底能分析出什么样的结果。

## 机器学习应用开发的典型步骤
假设，我们要开发一个房价评估系统，系统的目标是对一个已知特征的房子价格进行评估预测。建立这样一个系统需要包含以下几个步骤。

### 数据采集和标记
我们需要大量不同特征的房子和所对应的价格信息，可以直接从房产评估中心获取房子的相关信息，如房子的面积、地理位置、朝向、价格等。另外还有一些信息房产评估中心不一定有，比如房子所在地的学校情况，这一特征往往会影响房子的价格，这个时候就需要通过其他途径收集这些数据，这些数据叫做训练样本，或数据集。房子的面积、地理位置等称为特征。在数据采集阶段，需要收集尽量多的特征。特征越全，数据越多，训练出来的模型才会越准确。

通过这个过程也可以感受到数据采集的成本可能是很高的。人们常说石油是黑色的“黄金”，在人工智能时代，数据成了透明的“石油”，这也说明为什么蚂蚁金服估值这么高了。蚂蚁金服有海量的用户交易数据，据此他们可以计算出用户的信用指标，称为芝麻信用，根据芝麻信用给你一定的预支额，这就是一家新的信用卡公司了。而这还只是单单一个点的价值，真正的价值在于互联网金融。

在房价评估系统这个例子里，我们的房子价格信息是从房产评估中心获得的，这一数据可能不准确。有时为了避税，房子的评估价格会比房子的真实交易价格低很多。这时，就需要采集房子的实际成交价格，这一过程称为数据标记。标记可以是人工标记，比如逐个从房产中介那打听房子的实际成交价格；也可以是自动标记，比如通过分析数据，找出房产评估中心给的房子评估价格和真实成交价格的匹配关系，然后直接算出来。数据标记对有监督的学习方法是必须的。比如，针对垃圾邮件过滤系统，我们的训练样例必须包含这个邮件是否为垃圾邮件的标记数据。

### 数据清洗
假设我们采集到的数据里，关于房子面积，有按平方米计算的，也有按平方英尺计算的，这时需要对面积单位进行统一。这个过程称为数据清洗。数据清洗还包括去掉重复的数据及噪声数据，让数据具备结构化特征，以方便作为机器学习算法的输入。

### 特征选择
假设我们采集到了100个房子的特征，通过逐个分析这些特征，最终选择了30个特征作为输入。这个过程称为特征选择。特征选择的方法之一是人工选择方法，即对逐个特征进行人员分析，然后选择合适的特征集合。另外一个方法是通过模型来自动完成。

### 模型选择
房价评估系统是属于有监督学习的回归学习类型，我们可以选择最简单的线性方程来模拟。选择哪个模型，和问题领域、数据量大小、训练时长、模型的准确度等多方面有关。

### 模型训练和测试
把数据集分成训练数据集和测试数据集，一般按照8：2或7：3来划分，然后用训练数据集来训练模型。训练出参数后再使用测试数据集来测试模型的准确度。为什么要单独分出一个测试数据集来做测试呢？答案是必须确保测试的准确性，即模型的准确性是要用它“没见过”的数据来测试，而不能用那些用来训练这个模型的数据来测试。理论上更合理的数据集划分方案是分成3个，此外还要再加一个交叉验证数据集。

### 模型性能评估和优化
模型出来后，我们需要对机器学习的算法模型进行性能评估。性能评估包括很多方面，具体如下。 

训练时长是指需要花多长时间来训练这个模型。对一些海量数据的机器学习应用，可能需要1个月甚至更长的时间来训练一个模型，这个时候算法的训练性能就变得很重要了。

另外，还需要判断数据集是否足够多，一般而言，对于复杂特征的系统，训练数据集越大越好。然后还需要判断模型的准确性，即对一个新的数据能否准确地进行预测。最后需要判断模型是否能满足应用场景的性能要求，如果不能满足要求，就需要优化，然后继续对模型进行训练和评估，或者更换为其他模型。

### 模型使用
训练出来的模型可以把参数保存起来，下次使用时直接加载即可。一般来讲，模型训练需要的计算量是很大的，也需要较长的时间来训练，这是因为一个好的模型参数，需要对大型数据集进行训练后才能得到。而真正使用模型时，其计算量是比较少的，一般是直接把新样本作为输入，然后调用模型即可得出预测结果。

在实际工程应用领域，由于机器学习算法模型只有固定的几种，而数据采集、标记、清洗、特征选择等往往和具体的应用场景相关，机器学习工程应用领域的工程师打交道更多的反而是这些内容。

## 机器学习理论基础
## 过拟合和欠拟合
过拟合是指模型能很好地拟合训练样本，但对新数据的预测准确性很差。欠拟合是指模型不能很好地拟合训练样本，且对新数据的预测准确性也不好。

## scikit-learn简介
scikit-learn是一个开源的Python语言机器学习工具包，它涵盖了几乎所有主流机器学习算法的实现，并且提供了一致的调用接口。它基于Numpy和scipy等Python数值计算库，提供了高效的算法实现。

Scikit-learn是基于NumPy和SciPy的众多开源项目中的一个，由数据科学家David Cournapeau在2007年发起，需要NumPy和SciPy等其他模块的支持，是Python中专门针对机器学习应用而发展起来的一个模块。

总结起来，scikit-learn工具包有以下几个优点。
- 文档齐全：官方文档齐全，更新及时。
- 接口易用：针对所有的算法提供了一致的接口调用规则，不管是KNN、K-Mean还是PCA。
- 算法全面：涵盖主流机器学习任务的算法，包括回归算法、分类算法、聚类分析、数据降维处理等。

当然，scikit-learn不支持分布式计算，不适合用来处理超大型数据。但这并不影响scikit-learn作为一个优秀的机器学习工具库这个事实。许多知名的公司，包括Evernote和Spotify都使用scikit-learn来开发他们的机器学习应用。

## Scikit-learn概览
Scikit-learn的基本功能可以分为6大部分：分类、回归、聚类、数据降维、模型选择和数据预处理。
Scikit-learn提供了很多子模块，大致可以分为两类：一类是和机器学习的算法模型相关的子模块，另一类是数据预处理和模型评估相关的子模块。因为这个分类并不严格，所以下面列出的子模块并没有遵循这个分类。
- linear_model：线性模型子模块，包括最小二乘法、逻辑回归、随机梯度下降SGD等。
- cluster：聚类子模块，包括k-means聚类、DBSCAN、谱聚类、层次聚类、高斯混合等。
- neighbors：近邻算法子模块，包括最近邻算法、最近邻分类、最近邻回归等。
- discriminant_analysis：线性和二次判别分析子模块。
- kernel_ridge：内核岭回归子模块。
- svm：支持向量机子模块。
- gaussian_process：高斯过程子模块，包括高斯过程回归（GPR）、高斯过程分类（GPC）。
- cross_decomposition：交叉分解子模块，包括偏最小二乘法、典型相关分析等。
- naive_bayes：朴素贝叶斯子模块。
- tree：决策树子模块。
- ensemble：集成子模块，包括Bagging、Boosting、随机森林等。
- multiclass：多类和多标签分类算法子模块。
- feature_selection：特征选择算法子模块，包括方差阈值、单变量、递归性特征消除等。
- semi_supervised：半监督学习子模块。
- isotonic：等式回归子模块。
- calibration：概率校正子模块。
- neural_network：神经网络子模块，包括多层感知器（MLP）、限制玻尔兹曼机等。
- mixture：混合模型算法子模块，包括高斯混合和变分贝叶斯高斯混合。
- manifold：流形学习子模块。
- decomposition：成分分析子模块。
- datasets：加载和获取数据集的子模块。
- utils：工具函数子模块。
- preprocessing：预处理数据子模块，包括缩放、中心化、归一化、离散化、二值化、非线性变换、类别特征编码、缺值补全等。
- model_selection：模型选择子模块，包括数据集处理、交叉验证、网格搜索、验证曲线、学习曲线等。
- metrics：模型评估指标子模块，包括各种打分、表现指标、pairwise指标、距离计算、混淆矩阵、分类报告、精确度分数等。
- feature_extraction：特征提取子模块。
- pipeline：链式评估器（管道）和特征联合（FeatureUnion）。
- impute：缺失值插补的转换器子模块，包括SimpleImputer、IterativeImputer、MissingIndicator等。
- random_projection：随机投影子模块。
- kernel_approximation：内核近似子模块。
- inspection：模型检查子模块。
- compose：使用转换器构建复合模型的元评估器子模块。
- covariance：协方差估计子模块。
- dummy：Dummy评估器子模块。

## 数据集
在传统的软件开发中，程序员主要关注的对象是代码而非数据，但在机器学习中，程序员却需要拿出更多的精力来关注数据。在训练机器学习模型时，数据的质量和数量都会影响训练结果的准确性和有效性。因此，无论是学习还是实际应用机器学习模型，前提都是要有足够多且足够好的数据集。

Scikit-learn的数据集子模块datasets提供了两类数据集：一类是模块内置的小型数据集，这类数据集有助于理解和演示机器学习模型或算法，但由于数据规模较小，无法代表真实世界的机器学习任务；另一类是需要从外部数据源下载的数据集，这类数据集规模都比较大，对于研究机器学习来说更有实用价值。
前者使用loaders加载数据，函数名以load开头，后者使用fetchers加载数据，函数名以fetch开头，详细函数如下。
- datasets.load_boston([return_X_y])：加载波士顿房价数据集（回归）。
- datasets.load_breast_cancer([return_X_y])：加载威斯康星州乳腺癌数据集（分类）。
- datasets.load_diabetes([return_X_y])：加载糖尿病数据集（回归）。
- datasets.load_digits([n_class, return_X_y])：加载数字数据集（分类）。
- datasets.load_iris([return_X_y])：加载鸢尾花数据集（分类）。
- datasets.load_linnerud([return_X_y])：加载体能训练数据集（多元回归）。
- datasets.load_wine([return_X_y])：加载葡萄酒数据集（分类）。
- datasets.fetch_20newsgroups([data_home, …])：加载新闻文本分类数据集。
- datasets.fetch_20newsgroups_vectorized([…])：加载新闻文本向量化数据集（分类）。
- datasets.fetch_california_housing([…])：加载加利福尼亚住房数据集（回归）。
- datasets.fetch_covtype([data_home, …])：加载森林植被数据集（分类）。
- datasets.fetch_kddcup99([subset, data_home, …])：加载网络入侵检测数据集（分类）。
- datasets.fetch_lfw_pairs([subset, …])：加载人脸（成对）数据集。
- datasets.fetch_lfw_people([data_home, …])：加载人脸（带标签）数据集（分类）。
- datasets.fetch_olivetti_faces([data_home, …])：加载Olivetti人脸数据集（分类）。
- datasets.fetch_rcv1([data_home, subset, …])：加载路透社英文新闻文本分类数据集（分类）。
- datasets.fetch_species_distributions([…])：加载物种分布数据集。
我们以加载成对的人脸数据集为例，来看一看这个人脸数据集到底提供了什么数据。如果是第一次加载，函数fetch_lfw_pairs( )运行会花费较长的时间（因网络环境而不同）。
```shell
>>> from sklearn import datasets as dss
>>> lfwp = dss.fetch_lfw_pairs()
>>> lfwp.keys() # 数据集带有若干子集
dict_keys(['data', 'pairs', 'target', 'target_names', 'DESCR'])
>>> lfwp.data.shape, lfwp.data.dtype # data子集有2200个样本
((2200, 5828), dtype('float32'))
>>> lfwp.pairs.shape, lfwp.pairs.dtype # pairs子集有2200个样本，每个样本有两张图片
((2200, 2, 62, 47), dtype('float32'))
>>> lfwp.target_names # 有两个标签：不是同一个人、是同一个人
array(['Different persons', 'Same person'], dtype='<U17')
>>> lfwp.target.shape, lfwp.target.dtype # 2200个样本的标签，表示样本是否是同一个人
((2200,), dtype('int32'))
>>> import matplotlib.pyplot as plt
>>> plt.subplot(121)
<matplotlib.axes._subplots.AxesSubplot object at 0x00000161A91A33C8>
>>> plt.imshow(lfwp.pairs[0,0], cmap=plt.cm.gray)
<matplotlib.image.AxesImage object at 0x00000161AC5F8908>
>>> plt.subplot(122)
<matplotlib.axes._subplots.AxesSubplot object at 0x00000161AC6054C8>
>>> plt.imshow(lfwp.pairs[0,1], cmap=plt.cm.gray)
<matplotlib.image.AxesImage object at 0x00000161AC605E88>
>>> plt.show()
```
Scikit-learn提供的其他数据集基本上都类似人脸数据集的形式。有了这些数据集，并了解各个子集的意义、数据结构、数据类型，我们就可以非常方便地学习分类、聚类、回归、降维等基础的机器学习模型了。

## 样本生成器
除了内置数据集，Scikit-learn还提供了各种随机样本的生成器，可以创建样本数量和复杂度均可控制的模拟数据集，用这些数据集来测试或验证各种模型的参数优化。这些样本生成器都有相似的API，如参数n_samples表示样本数量，参数n_features表示特征维数，参数noise表示噪声标准差等。

样本生成器种类较多，下面的代码演示了其中三种样本生成器的用法。
- make_blobs( )常用于创建多类数据集，对于中心和各簇的标准偏差提供了更好的控制，可用于演示聚类。
- make_circles( )和make_moon( )生成二维分类数据集时可以帮助确定算法（如质心聚类或线性分类），包括可以选择性加入高斯噪声。
- 它们可以很容易地实现可视化。make_circles( )生成高斯数据，带有球面决策边界以用于二进制分类，而make_moon( )生成两个交叉的半圆。

```shell
>>> from sklearn import datasets as dss
>>> import matplotlib.pyplot as plt
>>> plt.rcParams['font.sans-serif'] = ['FangSong']
>>> plt.rcParams['axes.unicode_minus'] = False
>>> plt.subplot(131)
<matplotlib.axes._subplots.AxesSubplot object at 0x000001BD080A7CC8>
>>> plt.title('make_blobs()')
Text(0.5, 1.0, 'make_blobs()')
>>> X, y = dss.make_blobs(n_samples=100)
>>> plt.plot(X[:,0], X[:,1], 'o')
[<matplotlib.lines.Line2D object at 0x000001BD0BFFB1C8>]
>>> plt.subplot(132)
<matplotlib.axes._subplots.AxesSubplot object at 0x000001BD0D7B16C8>
>>> plt.title('make_circles()')
Text(0.5, 1.0, 'make_circles()')
>>> X, y = dss.make_circles(n_samples=100, noise=0.05, factor=0.5)
>>> plt.plot(X[:,0], X[:,1], 'o')
[<matplotlib.lines.Line2D object at 0x000001BD0D973648>]
>>> plt.subplot(133)
<matplotlib.axes._subplots.AxesSubplot object at 0x000001BD0D973EC8>
>>> plt.title('make_moons()')
Text(0.5, 1.0, 'make_moons()')
>>> X, y = dss.make_moons(n_samples=100, noise=0.05)
>>> plt.plot(X[:,0], X[:,1], 'o')
[<matplotlib.lines.Line2D object at 0x000001BD0C140D08>]
>>> plt.show()
```

## 加载其他数据集
尽管Scikit-learn的数据集子模块datasets提供了一些样本生成器，但我们仍然希望获得更加真实的数据。在datasets子模块提供的众多函数中，fetch_openml( )函数正是用来从openml.org下载数据集的。openml.org是一个机器学习数据和实验的数据仓库，它允许每个人上传开放的数据集。在openml.org这个数据仓库中，每个数据集都有名字和版本，data_id是数据集的唯一标识。fetch_openml( )函数通过名称或数据集的id从openml.org获取数据集。

以下代码从openml.org下载了名为kropt的数据集。这是一个和国际象棋相关的数据集，提供了28056个样本，根据白王、白车和黑王的位置可以对棋类游戏进行分类。从openml.org下载的数据集和Scikit-learn提供的其他数据集保持着相似的数据结构。
```shell
>>> from sklearn import datasets as dss
>>> chess = dss.fetch_openml(name='kropt')
>>> print(chess.DESCR)
Classify a chess game based on the position of the white king, the white rook and the
black king.

Downloaded from openml.org.
>>> chess.keys()
dict_keys(['data', 'target', 'frame', 'feature_names', 'target_names', 'DESCR',
'details', 'categories', 'url'])
>>> chess.data.shape, chess.data.dtype
((28056, 6), dtype('float64'))
```

## 数据预处理
机器学习的本质是从数据集中发现数据内在的特征，而数据的内在特征往往被样本的规格、分布范围等外在特征所掩盖。数据预处理正是为了最大限度地帮助机器学习模型或算法找到数据内在特征所做的一系列操作，主要是无量纲化。无量纲化是将不同规格的数据转换到同一规格，或不同分布的数据转换到某个特定分布。标准化、归一化等数据预处理属于无量纲化。除了无量纲化，数据预处理还包括特征编码、缺失值补全等。

### 标准化
假定样本集是二维平面上的若干个点，横坐标x分布于区间[0,100]内，纵坐标y分布于区间[0,1]内。显然，样本集的x特征列和y特征列的动态范围相差巨大，对于机器学习模型（如k-近邻或k-means聚类）的影响也会有显著差别。标准化处理正是为了避免某一个动态范围过大的特征列对计算结果造成影响，同时还可以提升模型精度。标准化的实质是对样本集的每个特征列减去该特征列均值进行中心化，再除以标准差进行缩放。

Scikit-learn的预处理子模块preprocessing提供了一个快速标准化函数scale( )，使用该函数可以直接返回标准化后的数据集，其代码如下。
```shell
>>> import numpy as np
>>> from sklearn import preprocessing as pp
>>> d = np.array([[ 1., -5.,  8.], [ 2.,  -3.,  0.], [ 0.,  -1., 1.]])
>>> d
array([[ 1., -5.,  8.],
       [ 2., -3.,  0.],
       [ 0., -1.,  1.]])
>>> d_scaled = pp.scale(d) # 对数据集d做标准化
>>> d_scaled
array([[ 0.        , -1.22474487,  1.40487872],
       [ 1.22474487,  0.        , -0.84292723],
       [-1.22474487,  1.22474487, -0.56195149]])
>>> d_scaled.mean(axis=0) # 标准化以后的数据集，各特征列的均值为0
array([0., 0., 0.])
>>> d_scaled.std(axis=0) # 标准化以后的数据集，各特征列的标准差为1
array([1., 1., 1.])
```
预处理子模块preprocessing还提供了一个实用类StandardScaler，它保存了训练集上各特征列的均值和标准差，以便以后在测试集上应用相同的变换。此外，实用类StandardScaler还可以通过with_mean和with_std参数指定是否中心化和是否按标准差缩放，其代码如下。
```shell
>>> import numpy as np
>>> from sklearn import preprocessing as pp
>>> X_train = np.array([[ 1., -5.,  8.], [ 2.,  -3.,  0.], [ 0.,  -1., 1.]])
>>> scaler = pp.StandardScaler().fit(X_train)
>>> scaler
StandardScaler(copy=True, with_mean=True, with_std=True)
>>> scaler.mean_ # 训练集各特征列的均值
array([ 1., -3.,  3.])
>>> scaler.scale_ # 训练集各特征列的标准差
array([0.81649658, 1.63299316, 3.55902608])
>>> scaler.transform(X_train) # 标准化训练集
array([[ 0.        , -1.22474487,  1.40487872],
       [ 1.22474487,  0.        , -0.84292723],
       [-1.22474487,  1.22474487, -0.56195149]])
>>> X_test = [[-1., 1., 0.]] # 使用训练集的缩放标准来标准化测试集
>>> scaler.transform(X_test)
array([[-2.44948974,  2.44948974, -0.84292723]])
```
### 归一化
标准化是用特征列的均值进行中心化，用标准差进行缩放。如果用数据集各个特征列的最小值进行中心化后，再按极差（最大值－最小值）进行缩放，即数据减去特征列的最小值，并且会被收敛到区间[0,1]内，这个过程就叫作数据归一化。需要说明的是，有很多人把这个操作称为“将特征缩放至特定范围内”，而把“归一化”这个名字交给了正则化。

预处理子模块preprocessing提供MinMaxScaler类来实现归一化功能。MinMaxScaler类有一个重要参数feature_range，该参数用于设置数据压缩的范围，默认是[0,1]。
```shell
>>> import numpy as np
>>> from sklearn import preprocessing as pp
>>> X_train = np.array([[ 1., -5.,  8.], [ 2.,  -3.,  0.], [ 0.,  -1., 1.]])
>>> scaler = pp.MinMaxScaler().fit(X_train) # 默认数据压缩范围为[0,1]
>>> scaler
MinMaxScaler(copy=True, feature_range=(0, 1))
>>> scaler.transform(X_train)
array([[0.5  , 0.   , 1.   ],
       [1.   , 0.5  , 0.   ],
       [0.   , 1.   , 0.125]])
>>> scaler = pp.MinMaxScaler(feature_range=(-2, 2)) # 设置数据压缩范围为[-2,2]
>>> scaler = scaler.fit(X_train)
>>> scaler.transform(X_train)
array([[ 0. , -2. ,  2. ],
       [ 2. ,  0. , -2. ],
       [-2. ,  2. , -1.5]])
```
因为归一化对异常值非常敏感，所以大多数机器学习算法会选择标准化来进行特征缩放。在主成分分析（Principal Component Analysis，PCA）、聚类、逻辑回归、支持向量机、神经网络等算法中，标准化往往是最好的选择。归一化在不涉及距离度量、梯度、协方差计算，以及数据需要被压缩到特定区间时被广泛使用，如数字图像处理中量化像素强度时，都会使用归一化将数据压缩在区间[0,1]内。

### 正则化
归一化是对数据集的特征列的操作，而正则化是将每个数据样本的范数单位化，是对数据集的行操作。如果打算使用点积等运算来量化样本之间的相似度，那么正则化将非常有用。

预处理子模块preprocessing提供了一个快速正则化函数normalize( )，使用该函数可以直接返回正则化后的数据集。normalize( )函数使用参数norm指定I1范式或I2范式，默认使用I2范式。I1范式可以理解为单个样本各元素的绝对值之和为1；I2范式可理解为单个样本各元素的平方和的算术根为1，相当于样本向量的模（长度）。
```shell
>>> import numpy as np
>>> from sklearn import preprocessing as pp
>>> X_train = np.array([[ 1., -5.,  8.], [ 2.,  -3.,  0.], [ 0.,  -1., 1.]])
>>> pp.normalize(X_train) # 使用I2范式正则化，每行的范数为1
array([[ 0.10540926, -0.52704628,  0.84327404],
       [ 0.5547002 , -0.83205029,  0.        ],
       [ 0.        , -0.70710678,  0.70710678]])
>>> pp.normalize(X_train, norm='I1') # 使用I1范式正则化，每行的范数为1
array([[ 0.07142857, -0.35714286,  0.57142857],
       [ 0.4       , -0.6       ,  0.        ],
       [ 0.        , -0.5       ,  0.5       ]])
```

### 离散化
离散化（Discretization）是将连续特征划分为离散特征值，典型的应用是灰度图像的二值化。如果使用等宽的区间对连续特征离散化，则被称为K-bins离散化。预处理子模块preprocessing提供了Binarizer类和KbinsDiscretizer类来进行离散化，前者用于二值化，后者用于K-bins离散化。
```shell
>>> import numpy as np
>>> from sklearn import preprocessing as pp
>>> X = np.array([[-2,5,11],[7,-1,9],[4,3,7]])
>>> X
array([[-2,  5, 11],
       [ 7, -1,  9],
       [ 4,  3,  7]])
>>> bina = pp.Binarizer(threshold=5) # 指定二值化阈值为5
>>> bina.transform(X)
array([[0, 0, 1],
       [1, 0, 1],
       [0, 0, 1]])
>>> est = pp.KBinsDiscretizer(n_bins=[2, 2, 3], encode='ordinal').fit(X)
>>> est.transform(X) # 三个特征列离散化为2段、2段、3段
array([[0., 1., 2.],
       [1., 0., 1.],
       [1., 1., 0.]])
```

### 特征编码
机器学习模型只能处理数值数据，但实际应用中有很多数据不是数值型的，如程序员的性别以及他们喜欢使用的浏览器、编辑器等，都不是数值型的，而是标称型的（categorical）。这些原始的数据特征在传入模型前需要编码成整数。

预处理子模块preprocessing提供OrdinalEncoder类，用来把n个标称型特征转换为0到n-1的整数编码。下面的代码中，x的每一个样本有三个特征项：性别、习惯使用的浏览器和编辑器。
```shell
>>> import numpy as np
>>> from sklearn import preprocessing as pp
>>> X = [['male', 'firefox', 'vscode'],
    ['female', 'chrome', 'vim'],
    ['male', 'safari', 'pycharme']]
>>> oenc = pp.OrdinalEncoder().fit(X)
>>> oenc
OrdinalEncoder(categories='auto', dtype=<class 'numpy.float64'>)
>>> oenc.transform(X)
array([[1., 1., 2.],
       [0., 0., 1.],
       [1., 2., 0.]])
```
然而，这样的整数特征带来了一个新的问题：各种浏览器、编辑器之间原本是无序的，现在却因为整数的连续性产生了2比1更接近3的关系。

另一种将标称型特征转换为能够在Scikit-learn模型中使用的编码是one-of-K，又称为独热码，由OneHotEncoder类实现。该类把每一个具有n个标称型的特征变换为长度为n的二进制特征向量，只有一位是1，其余都是0。
```shell
>>> import numpy as np
>>> from sklearn import preprocessing as pp
>>> X = [['male', 'firefox', 'vscode'],
    ['female', 'chrome', 'vim'],
    ['male', 'safari', 'pycharme']]
>>> ohenc = pp.OneHotEncoder().fit(X)
>>> ohenc.transform(X).toarray()
array([[0., 1., 0., 1., 0., 0., 0., 1.],
       [1., 0., 1., 0., 0., 0., 1., 0.],
       [0., 1., 0., 0., 1., 1., 0., 0.]])
```
上述代码中，性别有两个标称项，占用两位；浏览器和编辑器各有三个标称项，分别占用三位。编码之后的每一个样本有8列。

### 缺失值补全
虽然Scikit-learn的标准化和归一化可以自动忽略数据集中的无效值（numpy.nan），但这不意味着Scikit-learn的模型或算法可以接受有缺失的数据集。大多数的模型或算法都默认数组中的元素是数值，所有的元素都是有意义的。如果使用的数据集不完整，一个简单的策略就是舍弃整行或整列包含缺失值的数据，但这会浪费部分有效数据。更好的策略是从已有的数据推断出缺失值，也就是所谓的缺失值补全。

缺失值补全可以基于一个特征列的非缺失值来插补该特征列中的缺失值，也就是单变量插补。Scikit-learn的缺失值插补子模块impute提供SimpleImputer类用于实现基于一个特征列的插补。
```shell
>>> import numpy as np
>>> from sklearn.impute import SimpleImputer
>>> X = np.array([[3, 2], [np.nan, 3], [4, 6], [8, 4]]) # 首列有缺失
>>> X
array([[ 3.,  2.],
       [nan,  3.],
       [ 4.,  6.],
       [ 8.,  4.]])
>>> simp = SimpleImputer().fit(X) # 默认插补均值，当前均值为5
>>> simp.transform(X)
array([[3., 2.],
       [5., 3.],
       [4., 6.],
       [8., 4.]])
>>> simp = SimpleImputer(strategy='median').fit(X) # 中位数插补，当前中位数为4
>>> simp.transform(X)
array([[3., 2.],
       [4., 3.],
       [4., 6.],
       [8., 4.]])
>>> simp = SimpleImputer(strategy='constant', fill_value=0).fit(X) # 插补0
>>> simp.transform(X)
array([[3., 2.],
       [0., 3.],
       [4., 6.],
       [8., 4.]])
```

## 分类
通过训练已知分类的数据集，从中可以发现分类规则，并以此预测新数据的所属类别，这被称为分类算法。按照类别标签的多少，分类算法可分为二分类算法和多分类算法。分类算法属于监督学习。Scikit-learn中最常用的分类算法包括支持向量机（SVM）、最近邻、朴素贝叶斯、随机森林、决策树等。尽管Scikit-learn也支持多层感知器（MLP），不过Scikit-learn本身既不支持深度学习，也不支持GPU加速，因此MLP不适用于大规模数据应用。如果想借助GPU提高运行速度，建议使用Tensorflow、Keras、PyTorch等深度学习框架。

## 回归
回归是指研究一组随机变量（输入变量）和另一组变量（输出变量）之间关系的统计分析方法。如果输入变量的值发生变化时，输出变量随之改变，则可以使用回归算法预测输入变量和输出变量之间的关系。回归用于预测与给定对象相关联的连续值属性，而分类用于预测与给定对象相关联的离散属性，这是区分分类和回归问题的重要标志。和分类问题一样，回归问题也属于监督学习的一类。回归问题按照输入变量的个数，可以分为一元回归和多元回归；按照输入变量与输出变量之间的关系，可以分为线性回归和非线性回归。

评价一个分类模型的性能相对容易，因为一个样本属于某个类别是一个是非判断问题，分类模型对某个样本的分类只有正确和错误两种结果。但是评价一个回归模型的性能就要复杂得多。例如用回归预测某地区房价，其结果并无正确与错误之分，只能用偏离实际价格的程度来评价这个结果。

## 聚类
聚类是指自动识别具有相似属性的给定对象，并将具有相似属性的对象合并为同一个集合。聚类属于无监督学习的范畴，最常见的应用场景包括顾客细分和试验结果分组。根据聚类思想的不同，分为多种聚类算法，如基于质心的、基于密度的、基于分层的聚类等。Scikit-learn中最常用的聚类算法包括k均值聚类、谱聚类、均值漂移聚类、层次聚类、基于密度的空间聚类等。

## 成分分解与降维
收集数据时，我们总是不想漏掉任何一个可能会影响结果的变量，这将导致数据集变得异常庞大。而实际训练模型时，我们又总想剔除那些无效的或对结果影响甚微的变量，因为更多的变量意味着更大的计算量，这将导致模型训练慢得令人无法忍受。如何才能从众多数据中剔除无效的或对结果影响甚微的变量，以降低计算复杂度，提高模型训练效率呢？答案就是降维。机器学习领域中的降维是指采用某种映射方法，将原本高维度空间中的数据点映射到低维度的空间中。

## 模型评估与参数调优
了解了Scikit-learn在分类、回归、聚类、降维等领域的各种模型后，在实际应用中，我们还会面对许多新的问题：在众多可用模型中，应该选择哪一个？如何评价一个或多个模型的优劣？如何调整参数使一个模型具有更好的性能？这一类问题属于模型评估和参数调优，Scikit-learn在模型选择子模块model_selection中提供了若干解决方案。

在Scikit-learn体系内有以下三种方法可以用来评估模型的预测质量。在讨论分类、回归、聚类、降维等算法时，我们已经多次使用过这三种方式。
- 使用估计器（Estimator）的score( )方法。在Scikit-learn中，估计器是一个重要的角色，分类器和回归器都属估计器，是机器学习算法的实现。score( )方法返回的是估计器得分。
- 使用包括交叉验证在内的各种评估工具，如模型选择子模块model_selection中的cross_val_score和GridSearchCV等。
- 使用模型评估指标子模块metrics提供的针对特定目的评估预测误差的指标函数，包括分类指标函数、回归指标函数和聚类指标函数等。

# 参考资料
scikit-learn机器学习




















