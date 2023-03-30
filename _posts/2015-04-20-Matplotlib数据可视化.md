---
layout: post
categories: [Python,Pandas]
description: none
keywords: Python,Pandas
---
# Matplotlib数据可视化
Matplotlib是Python数据可视化工具包。IPython为Matplotlib专门提供了特殊的交互模式。如果要在IPython控制台使用Matplotlib，可以使用ipython--matplotlib命令来启动IPython控制台程序；如果要在IPython notebook里使用Matplotlib，则在notebook的开始位置插入%matplotlib inline魔术命令即可。IPython的Matplotlib模式有两个优点，一是提供了非阻塞的画图操作，二是不需要显式地调用show（）方法来显示画出来的图片。

Matplotlib下的pyplot子包提供了面向对象的画图程序接口。几乎所有的画图函数都与MATLAB类似，连参数都类似。在实际开发工作中，有时候甚至可以访问MATLAB的官方文档cn.mathworks.com/help/matlab来查询画图的接口和参数，这些参数可以直接在pyplot下的画图函数里使用。使用pyplot的习惯性写法是：
```shell
from matplotlitb import pyplot as plt
```
在机器学习领域中，我们经常需要把数据可视化，以便观察数据的模式。此外，在对算法性能进行评估时，也需要把模型相关的数据可视化，才能观察出模型里需要改进的地方。


## 图形样式
通常使用IPython notebook的Matplotlib模式来画图，这样画出来的图片会直接显示在网页上。要记得在notebook的最上面写上魔术命令%matplotlib inline。
使用Matplotlib的默认样式在一个坐标轴上画出正弦和余弦曲线：
```shell
%matplotlib inline
from matplotlib import pyplot as plt
import numpy as np
 
x = np.linspace(-np.pi, np.pi, 200)
C, S = np.cos(x), np.sin(x)
plt.plot(x, C)                       # 画出余弦曲线
plt.plot(x, S)                       # 画出正弦曲线
plt.show()
```
把正余弦曲线的线条画粗，并且定制合适的颜色：
```shell
# 画出余弦曲线，并设置线条颜色，宽度，样式
plt.plot(X, C, color="blue", linewidth=2.0, linestyle="-")
# 画出正弦曲线，并设置线条颜色，宽度，样式
plt.plot(X, S, color="red", linewidth=2.0, linestyle="-")
```
设置坐标轴的长度：
```shell
plt.xlim(X.min() * 1.1, X.max() * 1.1)
plt.ylim(C.min() * 1.1, C.max() * 1.1)
```
重新设置坐标轴的刻度。X轴的刻度使用自定义的标签，标签的文本使用了LaTeX来显示圆周率符号π。
```shell
# 设置坐标轴的刻度和标签
plt.xticks((-np.pi, -np.pi/2, np.pi/2, np.pi),
          (r'$-\pi$', r'$-\pi/2$', r'$+\pi/2$', r'$+\pi$'))
plt.yticks([-1, -0.5, 0, 0.5, 1])
```
把左侧图片中的4个方向的坐标轴改为两个方向的交叉坐标轴。方法是通过设置颜色为透明色，把上方和右侧的坐标边线隐藏起来。然后移动左侧和下方的坐标边线到原点（0，0）的位置。
```shell
# 坐标轴总共有4个连线，我们通过设置透明色隐藏上方和右方的边线
# 通过 set_position() 移动左侧和下侧的边线
# 通过 set_ticks_position() 设置坐标轴的刻度线的显示位置
ax = plt.gca()                # gca 代表当前坐标轴，即 'get current axis'
ax.spines['right'].set_color('none')             # 隐藏坐标轴
ax.spines['top'].set_color('none')
ax.xaxis.set_ticks_position('bottom')            # 设置刻度显示位置
ax.spines['bottom'].set_position(('data',0))     # 设置下方坐标轴位置
ax.yaxis.set_ticks_position('left')
ax.spines['left'].set_position(('data',0))       # 设置左侧坐标轴位置
```
在图片的左上角添加一个铭牌，用来标识图片中正弦曲线和余弦曲线。
```shell
plt.legend(loc='upper left')
```
在图片中标识出不但把这个公式画到图片上，还在余弦曲线上标识出这个点，同时用虚线画出这个点所对应的X轴的坐标。
```shell
t = 2 * np.pi / 3
# 画出 cos(t) 所在的点在 X 轴上的位置，即使用虚线画出 (t, 0) -> (t, cos(t)) 线段
plt.plot([t, t], [0, np.cos(t)], color='blue', linewidth=1.5, linestyle="--")
# 画出标示的坐标点，即在 (t, cos(t))处画一个大小为50的蓝色点
plt.scatter([t, ], [np.cos(t), ], 50, color='blue')
# 画出标示点的值，即 cos(t) 的值
plt.annotate(r'$cos(\frac{2\pi}{3})=-\frac{1}{2}$',
             xy=(t, np.cos(t)), xycoords='data',
             xytext=(-90, -50), textcoords='offset points', fontsize=16,
             arrowprops=dict(arrowstyle="->", connectionstyle="arc3,rad=.2"))
```
其中，plt.annotate（）函数的功能是在图片上画出标示文本，其文本内容也是使用LaTex公式书写。这个函数参数众多，具体可参阅官方的API说明文档。使用相同的方法，可以在正弦曲线上也标示出一个点。
定制坐标轴上的刻度标签的字体，同时为了避免正余弦曲线覆盖掉刻度标识，在刻度标签上添加一个半透明的方框作为背景。
```shell
# 设置坐标刻度的字体大小，添加半透明背景
for label in ax.get_xticklabels() + ax.get_yticklabels():
    label.set_fontsize(16)
    label.set_bbox(dict(facecolor='white', edgecolor='None', 
        alpha=0.65))
```

## 图形对象
在Matplotlib里，一个图形（figure）是指图片的全部可视区域，可以使用plt.figure（）来创建。在一个图形里，可以包含多个子图（subplot），可以使用plt.subplot（）来创建子图。子图按照网格形状排列显示在图形里，可以在每个子图上单独作画。坐标轴（Axes）和子图类似，唯一不同的是，坐标轴可以在图形上任意摆放，而不需要按照网格排列，这样显示起来更灵活，可以使用plt.axes（）来创建坐标轴。

当我们使用默认配置进行作画时，Matplotlib调用plt.gca（）函数来获取当前的坐标轴，并在当前坐标轴上作画。plt.gca（）函数调用plt.gcf（）函数来获取当前图形对象，如果当前不存在图形对象，则会调用plt.figure（）函数创建一个图形对象。

plt.figure（）函数有以下几个常用的参数。
- num：图形对象的标识符，可以是数字或字符串。当num所指定的图形存在时，直接返回这个图形的引用，如果不存在，则创建一个以这个num为标识符的新图形。最后把当前作画的图形切换到这个图形上。
- figsize：以英寸为单位的图形大小（width，height），是一个元组。
- dpi：指定图形的质量，每英寸多少个点。

下面的代码创建了两个图形，一个是'sin'，并且把正弦曲线画在这个图形上。然后创建了另外一个名称是'cos'的图形，并把余弦曲线画在这个图形上。接着切换到之前创建的'sin'图形上，把余弦图片也画在这个图形上。
```shell
%matplotlib inline
from matplotlib import pyplot as plt
import numpy as np
 
X = np.linspace(-np.pi, np.pi, 200, endpoint=True)
C, S = np.cos(X), np.sin(X)
 
plt.figure(num='sin', figsize=(16, 4))             # 创建 'sin' 图形
plt.plot(X, S)                                     # 把正弦画在这个图形上
plt.figure(num='cos', figsize=(16, 4))             # 创建 'cos' 图形
plt.plot(X, C)                   # 把余弦画在这个图形上
plt.figure(num='sin')            # 切换到 'sin' 图形上
plt.plot(X, C)                   # 在原来的基础上，把余弦曲线也画在这个图形上
print plt.figure(num='sin').number
print plt.figure(num='cos').number
```
不同的图形可以单独保存为一个图片文件，但子图是指一个图形里分成几个区域，在不同的区域里单独作画，所有的子图最终都保存在一个文件里。plt.subplot（）函数的关键参数是一个包含3个元素的元组，分别代表子图的行、列以及当前激活的子图序号。比如plt.subplot（2，2，1）表示把图表对象分成两行两列，激活第一个子图来作画。我们看一个网格状的子图的例子：
```shell
%matplotlib inline
from matplotlib import pyplot as plt
 
plt.figure(figsize=(18, 4))
plt.subplot(2, 2, 1)
plt.xticks(())
plt.yticks(())
plt.text(0.5, 0.5, 'subplot(2,2,1)', ha='center', va='center',
        size=20, alpha=.5)
 
plt.subplot(2, 2, 2)
plt.xticks(())
plt.yticks(())
plt.text(0.5, 0.5, 'subplot(2,2,2)', ha='center', va='center',
        size=20, alpha=.5)
 
plt.subplot(2, 2, 3)
plt.xticks(())
plt.yticks(())
plt.text(0.5, 0.5, 'subplot(2,2,3)', ha='center', va='center',
        size=20, alpha=.5)
 
plt.subplot(2, 2, 4)
plt.xticks(())
plt.yticks(())
plt.text(0.5, 0.5, 'subplot(2,2,4)', ha='center', va='center',
        size=20, alpha=.5)
 
plt.tight_layout()
plt.show()
```
更复杂的子图布局，可以使用gridspec来实现，其优点是可以指定某个子图横跨多个列或多个行。
```shell
%matplotlib inline
from matplotlib import pyplot as plt
import matplotlib.gridspec as gridspec
 
plt.figure(figsize=(18, 4))
G = gridspec.GridSpec(3, 3)
 
axes_1 = plt.subplot(G[0, :])        # 占用第一行，所有的列
plt.xticks(())
plt.yticks(())
plt.text(0.5, 0.5, 'Axes 1', ha='center', va='center', size=24, alpha=.5)
 
axes_2 = plt.subplot(G[1:, 0])       # 占用第二行开始之后的所有行，第一列
plt.xticks(())
plt.yticks(())
plt.text(0.5, 0.5, 'Axes 2', ha='center', va='center', size=24, alpha=.5)
 
axes_3 = plt.subplot(G[1:, -1])      # 占用第二行开始之后的所有行，最后一列
plt.xticks(())
plt.yticks(())
plt.text(0.5, 0.5, 'Axes 3', ha='center', va='center', size=24, alpha=.5)
 
axes_4 = plt.subplot(G[1, -2])       # 占用第二行，倒数第二列
plt.xticks(())
plt.yticks(())
plt.text(0.5, 0.5, 'Axes 4', ha='center', va='center', size=24, alpha=.5)
 
axes_5 = plt.subplot(G[-1, -2])      # 占用倒数第一行，倒数第二列
plt.xticks(())
plt.yticks(())
plt.text(0.5, 0.5, 'Axes 5', ha='center', va='center', size=24, alpha=.5)
 
plt.tight_layout()
plt.show()
```
坐标轴使用plt.axes（）来创建，它用一个矩形来给坐标轴定位，矩形使用[left，bottom，width，height]来表达。其数据为图形对象对应坐标轴长度的百分比。
```shell
%matplotlib inline
from matplotlib import pyplot as plt
 
plt.figure(figsize=(18, 4))
 
plt.axes([.1, .1, .8, .8])
plt.xticks(())
plt.yticks(())
plt.text(.2, .5, 'axes([0.1, 0.1, .8, .8])', ha='center', va='center',
        size=20, alpha=.5)
 
plt.axes([.5, .5, .3, .3])
plt.xticks(())
plt.yticks(())
plt.text(.5, .5, 'axes([.5, .5, .3, .3])', ha='center', va='center',
        size=16, alpha=.5)
 
plt.show()
```
一个优美而恰当的坐标刻度对理解数据异常重要，Matplotlib内置提供了以下几个坐标刻度。
- NullLocater：不显示坐标刻度标签，只显示坐标刻度。
- MultipleLocator：以固定的步长显示多个坐标标签。
- FixedLocator：以列表形式显示固定的坐标标签。
- IndexLocator：以offset为启始位置，每隔base步长就画一个坐标标签。
- LinearLocator：把坐标轴的长度均分为numticks个数，显示坐标标签。
- LogLocator：以对数为步长显示刻度标签。
- MaxNLocator：从提供的刻度标签列表里，显示出最大不超过nbins个数的标签。
- AutoLocator：自动显示刻度标签。

除了内置标签外，我们也可以继承matplotlib.ticker.Locator类来实现自定义样式的刻度标签。

通过下面的代码把内置坐标刻度全部画出来，可以直观地观察到内置坐标刻度的样式。
```shell
%matplotlib inline
from matplotlib import pyplot as plt
import numpy as np
 
def tickline():
    plt.xlim(0, 10), plt.ylim(-1, 1), plt.yticks([])
    ax = plt.gca()
    ax.spines['right'].set_color('none')
    ax.spines['left'].set_color('none')
    ax.spines['top'].set_color('none')
    ax.xaxis.set_ticks_position('bottom')
    ax.spines['bottom'].set_position(('data',0))
    ax.yaxis.set_ticks_position('none')
    ax.xaxis.set_minor_locator(plt.MultipleLocator(0.1))
    # 设置刻度标签的文本字体大小
    for label in ax.get_xticklabels() + ax.get_yticklabels():
        label.set_fontsize(16)
    ax.plot(np.arange(11), np.zeros(11))
    return ax
 
locators = [
                'plt.NullLocator()',
                'plt.MultipleLocator(base=1.0)',
                'plt.FixedLocator(locs=[0, 2, 8, 9, 10])',
                'plt.IndexLocator(base=3, offset=1)',
                'plt.LinearLocator(numticks=5)',
                'plt.LogLocator(base=2, subs=[1.0])',
                'plt.MaxNLocator(nbins=3, steps=[1, 3, 5, 7, 9, 10])',
                'plt.AutoLocator()',
            ]
 
n_locators = len(locators)
 
# 计算图形对象的大小
size = 1024, 60 * n_locators
dpi = 72.0
figsize = size[0] / float(dpi), size[1] / float(dpi)
fig = plt.figure(figsize=figsize, dpi=dpi)
fig.patch.set_alpha(0)
 
 
for i, locator in enumerate(locators):
    plt.subplot(n_locators, 1, i + 1)
    ax = tickline()
    ax.xaxis.set_major_locator(eval(locator))      # 使用 eval 表达式：eval is evil
    plt.text(5, 0.3, locator[3:], ha='center', size=16)
 
plt.subplots_adjust(bottom=.01, top=.99, left=.01, right=.99)
plt.show()
```

## 画图操作