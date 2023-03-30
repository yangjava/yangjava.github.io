---
layout: post
categories: [Python,Pandas]
description: none
keywords: Python,Pandas
---
# Matplotlib数据可视化
在Python的世界里，除了Web应用开发以外，涉及2D绘图或一些简单的3D绘图，大多数程序员会首选Matplotlib库。

## 快速入门
Matplotlib是Python比较底层的可视化库，同时又和科学计算库、数据分析库、机器学习库等高度集成，例如NumPy、SciPy、Pandas、Scikti-learn等。Matplotlib拥有丰富的图表资源，风格接近MATLAB，简单易用，其输出图片的质量可以达到出版级别。

Matplotlib具有很高的可定制性，因此基于Matplotlib又衍生出了seaborn等可视化模块或新的封装。毫无疑问，Matplotlib已经成为Python生态圈中应用最广泛的绘图库。

Matplotlib中的概念比较复杂，对初学者来说可能会比较困难。 在Matplotlib中，绘图前需要先创建一个画布（figure），然后在这个画布上可以画一幅或多幅图，每一幅图都是一个子图（axes）。

子图是Matplotlib中非常重要的类，所有的线条、矩形、文字、图像都是在子图上呈现的。子图在画布上可以存在一个或多个。子图上除了绘制线条、矩形、文字、图像等元素外，还可以设置x轴的标注（x axis label）和y轴的标注（y axis label）、刻度（tick）、子图中的网格（grid）以及图例（legend）等。

Matplotlib是Python数据可视化工具包。IPython为Matplotlib专门提供了特殊的交互模式。如果要在IPython控制台使用Matplotlib，可以使用ipython--matplotlib命令来启动IPython控制台程序。

如果要在IPython notebook里使用Matplotlib，则在notebook的开始位置插入%matplotlib inline魔术命令即可。IPython的Matplotlib模式有两个优点，一是提供了非阻塞的画图操作，二是不需要显式地调用show（）方法来显示画出来的图片。

Matplotlib下的pyplot子包提供了面向对象的画图程序接口。几乎所有的画图函数都与MATLAB类似，连参数都类似。在实际开发工作中，有时候甚至可以访问MATLAB的官方文档cn.mathworks.com/help/matlab来查询画图的接口和参数，这些参数可以直接在pyplot下的画图函数里使用。使用pyplot的习惯性写法是：
```shell
from matplotlitb import pyplot as plt
```
在机器学习领域中，我们经常需要把数据可视化，以便观察数据的模式。此外，在对算法性能进行评估时，也需要把模型相关的数据可视化，才能观察出模型里需要改进的地方。

## 画布
在Matplotlib中figure被称为画布。画布是一个顶级的容器（container），里面可以放置其他低阶容器，如子图（axes）。 figure对应现实中的画布，可以设置它的大小和颜色等属性。
```shell
from matplotlib import pyplot as plt # 导入模块
fig = plt.figure() # 创建画布
fig.set_size_inches(5, 3) # 设置画布大小，单位是英寸
fig.set_facecolor('gray') # 设置画布颜色
fig.show() # 显示画布
```
以交互方式执行上面的代码会弹出一个灰色背景的绘图窗口，这就是通过代码定义的画布。这个窗口下方提供了放大、拖曳、保存文件等操作按钮。

### 子图与子图布局
在画布上可以添加一个或多个子图（axes）。子图是具有数据空间的绘图区域，在子图上可以绘制各种图形。
在画布中添加子图有两种方式，第1种方式的代码如下。
```shell
from matplotlib import pyplot as plt
plt.rcParams['font.sans-serif'] = ['FangSong'] # 设置字体以便正确显示中文
fig = plt.figure()
ax1 = fig.add_axes([0.1, 0.1, 0.8, 0.8]) # 添加子图1
ax2 = fig.add_axes([0.5, 0.5, 0.4, 0.2]) # 添加子图2
ax1.set_title("子图1") # 设置子图1的标题
# Text(0.5, 1.0, '子图1')
ax2.set_title("子图2") # 设置子图2的标题
# Text(0.5, 1.0, '子图2')
ax1.set_facecolor('#E0E0E0') # 设置子图1的背景色
ax2.set_facecolor('#F0F0F0') # 设置子图2的背景色
fig.show()
```
使用add_axes( )函数添加子图时，需要传入一个四元组参数，用于指定子图左下角（坐标原点）在画布上距离左边、底边的距离分别与画布宽度和高度的比例，以及子图的宽度和高度分别占画布宽度和高度的比例。
使用add_axes( )函数添加子图可以实现子图叠加的效果。

常用的添加子图的第2种方式是使用add_subplot( )函数，此方式可以批量生成子图，且各个子图在画布上以网格方式排版，其代码如下。
```shell
from matplotlib import pyplot as plt
plt.rcParams['font.sans-serif'] = ['FangSong'] # 设置字体以便正确显示中文
fig = plt.figure()
fig.suptitle("add_subplot方法示例") # 画布标题
# Text(0.5, 0.98, 'add_subplot方法示例')
ax1 = fig.add_subplot(221)
ax1.set_title("2行2列中的第1个子图")
# Text(0.5, 1.0, '2行2列中的第1个子图')
ax2 = fig.add_subplot(222)
ax2.set_title("2行2列中的第2个子图")
# Text(0.5, 1.0, '2行2列中的第2个子图')
ax3 = fig.add_subplot(223)
ax3.set_title("2行2列中的第3个子图")
# Text(0.5, 1.0, '2行2列中的第3个子图')
ax3 = fig.add_subplot(224)
ax3.set_title("2行2列中的第4个子图")
# Text(0.5, 1.0, '2行2列中的第4个子图')
fig.show()
```
使用add_subplot( )函数添加子图需要传入子图网格的行数、列数，以及当前生成的子图的序号，序号从1开始，优先从水平方向上进行排序。
例如，画布上2行2列共4个子图，当前添加的子图位于第2行第2列，序号为4，传入的参数既可以写成add_subplot(2, 2, 4)，也可以写成add_subplot(224)。

### 坐标轴与刻度的名称
axis是坐标轴，tick是刻度，二者的等级相同。axis和tick相互配合可以设置坐标轴的值域范围、刻度显示、刻度的字符和刻度字符格式化，从而精确地控制刻度位置和标签。
```shell
import numpy as np
from matplotlib import pyplot as plt
from matplotlib.ticker import MultipleLocator, FormatStrFormatter
plt.rcParams['font.sans-serif'] = ['FangSong'] # 设置字体以便正确显示中文
plt.rcParams['axes.unicode_minus'] = False # 正确显示连字符
x = np.linspace(-10,10,50)
y = np.power(x, 2)
fig = plt.figure()
axes = fig.add_axes([0.2, 0.2, 0.7, 0.7])
axes.set_title("设置坐标轴的刻度和名称", fontdict={'size':16}) # 设置子图标题
# Text(0.5, 1.0, '设置坐标轴的刻度和名称')
axes.plot(x, y) # 绘制连线
# 
xmajorLocator = MultipleLocator(4)
xmajorFormatter = FormatStrFormatter('%1.1f')
axes.xaxis.set_major_locator(xmajorLocator) # 设置刻度的疏密
axes.xaxis.set_major_formatter(xmajorFormatter) # 设置刻度字符格式
axes.set_xlabel('x', fontdict={'size':14}) # 设置x轴标注
# Text(0.5, 0, 'x')
axes.set_ylabel('$y=x^2$', fontdict={'size':14}) # 设置y轴标注
# Text(0, 0.5, '$y=x^2$')
fig.show()
```
上述代码中设置y轴名称时，使用了LaTex语法。Matplotlib很好地支持了LaTex语法，可以在坐标轴的名称、画布标题和子图标题、图例等元素中使用LaTex语法加入数学公式。

### 图例和文本标注
数据可视化是用图表来展示数据本质的含义，而图例和文本标注可以帮助我们更加深刻地理解图表所要表达的意义和想要传递的信息。
```shell
import numpy as np
from matplotlib import pyplot as plt
plt.rcParams['font.sans-serif'] = ['FangSong'] # 设置字体以便正确显示中文
plt.rcParams['axes.unicode_minus'] = False # 正确显示连字符
x = np.linspace(-4, 4, 200)
y1 = np.power(10, x)
y2 = np.power(np.e, x)
y3 = np.power(2, x)
fig = plt.figure()
axes = fig.add_axes([0.1, 0.1, 0.8, 0.8])
axes.plot(x, y1, 'r', ls='-', linewidth=2, label='$10^x$')
axes.plot(x, y2, 'b', ls='--', linewidth=2, label='$e^x$')
axes.plot(x, y3, 'g', ls=':', linewidth=2, label='$2^x$')
axes.axis([-4, 4, -0.5, 8]) # 设置坐标轴范围
[-4, 4, -0.5, 8]
axes.text(1, 7.5, r'$10^x$', fontsize=16) # 添加文本标注
# Text(1, 7.5, '$10^x$')
axes.text(2.2, 7.5, r'$e^x$', fontsize=16) # 添加文本标注
# Text(2.2, 7.5, '$e^x$')
axes.text(3.2, 7.5, r'$2^x$', fontsize=16) # 添加文本标注
# Text(3.2, 7.5, '$2^x$')
axes.legend(loc='upper left') # 生成图例
fig.show()
```
在上述代码中，使用plot( )函数绘制曲线时，设置了颜色、线型、线宽，并使用了label参数。生成图例时，Matplotlib会自动将各条曲线的颜色、线型、线宽和label添加到图例中。

## 显示和保存
使用画布的show( )函数可以显示画布，并且只能显示一次。事实上，画布及画布上的子图依然存在，只是因为show( )函数调用tkinter这个GUI库时做了限制。在任何情况下都可以使用画布的savefig( )函数将画布保存为文件，只需要向savefig( )函数传递一个文件名即可。如果需要，还可以指定生成图像文件的分辨率。
```shell
import numpy as np
from matplotlib import pyplot as plt
plt.rcParams['font.sans-serif'] = ['FangSong'] # 设置字体以便正确显示中文
plt.rcParams['axes.unicode_minus'] = False # 正确显示连字符
fig = plt.figure()
fig.set_size_inches(5, 3) # 设置画布的宽为5英寸、高为3英寸
axes = fig.add_axes([0.1, 0.1, 0.8, 0.8])
x = np.random.rand(50) # 随机生成散点x坐标
y = np.random.rand(50) # 随机生成散点y坐标
color = 2 * np.pi * np.random.rand(50) # 随机生成散点颜色
axes.scatter(x, y, c=color, alpha=0.5, cmap='jet') # 绘制散点图
# 
axes.set_title("散点图")
# Text(0.5, 1.0, '散点图')
fig.savefig('scatter.png', dpi=300) # 保存为文件，同时设置分辨率为300dot/in
```
通常savefig( )函数会根据文件名选择保存的图像格式，也可以使用format参数指定图像格式。

### 两种使用风格
为了保持概念的清晰，前面的小节中每个绘图示例在绘图前都是先生成一块画布，并在画布上添加一个或多个子图，调用的plot( )或show( )等函数都是画布或子图的方法。使用Matplotlib时，推荐使用这种面向对象的使用风格。

除此之外，Matplotlib还提供一种类MATLAB风格的API，网络上大部分的绘图示例都是这种风格的。这种使用方式没有画布和子图的概念，或默认有一块画布，并且画布上已经存在一个子图，只需调用pyplot的函数即可绘图和显示。
```shell
import numpy as np
from matplotlib import pyplot as plt
x = np.linspace(0,10,100)
y = np.exp(-x)
plt.plot(x, y) # 调用pyplot的plot()函数
# 
plt.show() # 调用pyplot的show()函数
```
类MATLAB风格的使用方式不需要创建画布，也不用添加子图，使用起来非常简单。如果需要绘制多个子图，pyplot提供了subplot( )函数来实现类似添加子图的功能。
```shell
import numpy as np
from matplotlib import pyplot as plt
x = np.linspace(0,10,100)
y1 = np.exp(-x)
y2 = np.sin(x)
plt.subplot(121) # 现在开始在1行2列的第1个位置绘图
# 
plt.plot(x,y1)
# 
plt.subplot(122) # 现在开始在1行2列的第2个位置绘图
#
plt.plot(x,y2)
plt.show()
```
上面的代码绘制了1行2列共两个子图

很多情况下，这两种使用风格没有严格的区分，经常混合使用。例如，下面的代码以面向对象的使用风格创建了两块画布，最后以类MATLAB风格的使用方式显示画布，结果是同时弹出两个画布窗口。
```shell
import numpy as np
from matplotlib import pyplot as plt
f1 = plt.figure() # 以面向对象的使用风格创建画布
f2 = plt.figure() # 以面向对象的使用风格创建画布
ax1 = f1.add_axes([0.1, 0.1, 0.8, 0.8]) # 以面向对象的使用风格添加子图
ax2 = f2.add_axes([0.1, 0.1, 0.8, 0.8]) # 以面向对象的使用风格添加子图
x = np.arange(5)
ax1.plot(x) # 以面向对象的使用风格绘图
ax2.plot(x) # 以面向对象的使用风格绘图
plt.show() # 以类MATLAB风格的使用方式显示画布
```
请仔细体会两种使用风格的差异。类MATLAB风格的使用方式代码简洁，面向对象的使用风格提供了更多的格式和界面的控制方法，可以更精准地定制可视化结果。

## 丰富多样的图形
Matplotlib是一个2D绘图库，只需要几行代码就可以生成多种高质量的图形。通过执行以下代码导入必要的模块。
```shell
import numpy as np
from matplotlib import pyplot as plt
plt.rcParams['font.sans-serif'] = ['FangSong'] # 设置字体以便正确显示中文
plt.rcParams['axes.unicode_minus'] = False # 正确显示连字符
```

### 曲线图
使用axes子图对象的plot( )函数可以绘制曲线图。严格来说，plot( )函数绘制的是折线图而非曲线图，因为该函数只是将顺序相邻的点以直线连接，并未做任何平滑处理。
该函数有两种调用方式，函数原型及主要参数说明如下。
```shell
plot([x], y, [fmt], **kwargs)
plot([x], y, [fmt], [x2], y2, [fmt2], ..., **kwargs)
```
- x、y：长度相同的一维数组，表示离散点的x坐标和y坐标。如果省略x，则以y的索引序号代替。
- fmt：格式字符串，用来快速设置颜色、线型等属性，例如，“ro”代表红圈。所有参数都可以通过关键字参数进行设置。
- 关键字参数color：简写为c，字符串类型，用于设置颜色。
- 关键字参数linestyle：简写为ls，浮点数类型，用于设置线型。
- 关键字参数linewidth：简写为lw，浮点数类型，用于设置线宽。
- 关键字参数marker：字符串类型，用于设置点的样式。
- 关键字参数markersize：简写为ms，浮点数类型，用于设置点的大小。
- 关键字参数markerfacecolor：简写为mfc，字符串类型，用于设置点的颜色。
- 关键字参数markeredgecolor：简写为mec，字符串类型，用于设置点的边缘颜色。
- 关键字参数label：字符串类型，用于设置在图例中显示的曲线名称。

以下代码演示了plot( )函数及其参数的用法。
```shell
import numpy as np
from matplotlib import pyplot as plt
plt.rcParams['font.sans-serif'] = ['FangSong'] # 设置字体以便正确显示中文
plt.rcParams['axes.unicode_minus'] = False # 正确显示连字符

x = np.linspace(-4, 4, 100)
y1 = np.power(np.e, x)
y2 = np.sin(x)*10 + 30
fig = plt.figure()
axes = fig.add_axes([0.1, 0.1, 0.8, 0.8])
axes.plot(x, y1, c='green', label='$e^x$', ls='-.', alpha=0.6, lw=2)
axes.plot(x, y2, c='m', label='$10*sin(x)+10$', ls=':', alpha=1, lw=3)
axes.legend()
plt.show()
```
除了plot( )函数，semilogx( )、semilogy( )和loglog( )函数也可以用来绘制曲线图，只不过它们分别绘制x轴为对数坐标、y轴为对数坐标或两个坐标轴都为对数坐标的数据。
```shell
import numpy as np
from matplotlib import pyplot as plt
plt.rcParams['font.sans-serif'] = ['FangSong'] # 设置字体以便正确显示中文
plt.rcParams['axes.unicode_minus'] = False # 正确显示连字符

x = np.arange(100)
y = np.exp(x)
fig = plt.figure()
ax1 = fig.add_subplot(221)
ax2 = fig.add_subplot(222)
ax3 = fig.add_subplot(223)
ax4 = fig.add_subplot(224)
ax1.set_title("plot")
# Text(0.5, 1.0, 'plot')
ax1.plot(x,y,c='r')

ax2.set_title("semilogx")
# Text(0.5, 1.0, 'semilogx')
ax2.semilogx(x,y,c='g')
ax3.set_title("semilogy")
# Text(0.5, 1.0, 'semilogy')
ax3.semilogy(x,y,c='m')
ax4.set_title("loglog")
# Text(0.5, 1.0, 'loglog')
ax4.loglog(x,y,c='k')
plt.show()
```
对同一组数据分别使用plot( )、semilogx( )、semilogy( )和loglog( )这4个函数绘制

## 散点图
散点图是指离散数据点在直角坐标系平面上的分布图。离散的数据点之间是相互独立的，没有顺序关系。
使用子图的scatter( )函数可以绘制散点图，该函数原型及主要参数说明如下。
```shell
scatter(x, y, s=None, c=None, marker=None, cmap=None, alpha=None, **kwargs)
```
- x、y：长度相同的一维数组，表示离散点的x坐标和y坐标。
- s：标量或数组，表示离散点的大小。
- c：颜色或颜色数组，表示离散点的颜色；如果是数值数组，则由映射调色板映射为离散点的颜色。
- cmap：用于指定映射调色板，将分层Z值映射到颜色。pyplot. colormaps( )函数返回可用的映射调色板名称。如果同时给出colors参数和cmap参数，则会引发错误。
- alpha：[0,1]之间的浮点数，表示离散点的透明度。
- 关键字参数edgecolors：用于设置离散点的边缘颜色。默认为无边缘色。

以下代码演示了scatter( )函数及其参数的用法。本例随机生成了50个符合标准正态分布的点。
```shell
import numpy as np
from matplotlib import pyplot as plt
plt.rcParams['font.sans-serif'] = ['FangSong'] # 设置字体以便正确显示中文
plt.rcParams['axes.unicode_minus'] = False # 正确显示连字符

x = np.random.randn(50) # 随机生成50个符合标准正态分布的点（x坐标）
y = np.random.randn(50) # 随机生成50个符合标准正态分布的点（y坐标）
color = 10 * np.random.rand(50) # 随机数，用于映射颜色
area = np.square(30*np.random.rand(50)) # 随机数表示点的面积
fig = plt.figure()
ax = fig.add_axes([0.1, 0.1, 0.8, 0.8])
ax.scatter(x, y, c=color, s=area, cmap='hsv', marker='o', edgecolor='r', alpha=0.5)
plt.show()
```

### 等值线图
等值线图也被称为等高线图，是一种在二维平面上显示三维表面的方法。 Matplotlib API提供了两个等值线图的绘制方法：
contour( )函数用于绘制带轮廓线的等值线图，contourf( )函数用于绘制带填充色的等值线图。这两个函数的原型及主要参数说明如下。
```shell
contour([X, Y,] Z, [levels], **kwargs)
contourf([X, Y,] Z, [levels], **kwargs)
```
- Z是二维数组。X和Y是与Z形状相同的二维数组；或X和Y都是一维的，且X的长度等于Z的列数，Y的长度等于Z的行数。如果X和Y没有给出，则假定它们是整数索引。
- levels：表示Z值分级数量。如果是整数n，则绘制n+1条等高线；如果是数组且值是递增的，则在指定的Z值上绘制等高线。
- 关键字参数colors：用于指定层次（轮廓线条或轮廓区域）的颜色。
- 关键字参数cmap：用于指定映射调色板，将分层Z值映射到颜色。如果同时给出colors参数和cmap参数，则会引发错误。

以下代码演示了使用contour( )函数和contourf( )函数绘制等值线图的方法，同时也演示了为子图添加ColorBar（颜色棒或色卡）的方法。
```shell
import numpy as np
from matplotlib import pyplot as plt
plt.rcParams['font.sans-serif'] = ['FangSong'] # 设置字体以便正确显示中文
plt.rcParams['axes.unicode_minus'] = False # 正确显示连字符

y, x = np.mgrid[-3:3:600j, -4:4:800j]
z = (1-y**5+x**5)*np.exp(-x**2-y**2)
fig = plt.figure()
ax1 = fig.add_subplot(121)
ax1.set_title('无填充的等值线图')

c1 = ax1.contour(x, y, z, levels=8, cmap='jet') # 无填充的等值线
ax1.clabel(c1, inline=1, fontsize=12) # 为等值线标注值

ax2 = fig.add_subplot(122)
ax2.set_title('有填充的等值线图')

c2 = ax2.contourf(x, y, z, levels=8, cmap='jet') # 有填充的等值线
fig.colorbar(c1, ax=ax1) # 添加ColorBar

fig.colorbar(c2, ax=ax2) # 添加ColorBar

plt.show()
```
请注意代码中为不同等值线标注数值的方法，以及为子图添加ColorBar的方法。

### 矢量合成图
矢量合成图以箭头的形式绘制矢量，因此也被称为箭头图。矢量合成图以箭头的方向表示矢量的方向，以箭头的长度表示矢量的大小。
Matplotlib API使用quiver( )函数绘制矢量合成图，该函数有多种调用方法，函数原型及主要参数说明如下。
```shell
quiver(U, V, **kw)
quiver(U, V, C, **kw)
quiver(X, Y, U, V, **kw)
quiver(X, Y, U, V, C, **kw)
```
- U、V：一维或二维数组。U表示矢量的水平分量，V表示矢量的垂直分量。
- X和Y：一维或二维数组，表示矢量箭头的位置。如果没有给出X和Y，则根据U和V的维数生成一个统一的整数网格。
- C：一维或二维数组，由映射调色板映射箭头颜色，不支持显示颜色。如果想直接设置颜色，需要使用关键字参数color。
- 关键字参数color：颜色或颜色数组，用于设置箭头颜色。
- 关键字参数cmap：用于指定映射调色板，将C参数的值映射成颜色。

下面的代码生成了一个二维网格，网格上每个点对应一个矢量。用每个点的x坐标表示该点对应矢量的水平分量U，用每个点的y坐标表示该点对应矢量的垂直分量V，将每个分量的长度映射为颜色。
```shell
import numpy as np
from matplotlib import pyplot as plt
plt.rcParams['font.sans-serif'] = ['FangSong'] # 设置字体以便正确显示中文
plt.rcParams['axes.unicode_minus'] = False # 正确显示连字符

y, x = np.mgrid[-3:3:12j, -5:5:20j]
p = np.hypot(x,y)
fig = plt.figure()
ax = fig.add_axes([0.1, 0.1, 0.8, 0.8])
ax.quiver(x, y, x, y, p, cmap='hsv')
plt.show()
```
很容易就可以想象出来，每个矢量都会指向远离原点的方向。

## 直方图
直方图是数值数据分布的精确图形表示。绘制直方图前需要将数值范围分割成等长的间隔，然后计算每个间隔中有多少个数值。
Matplotlib API使用hist( )函数绘制直方图。该函数有很多参数，可以灵活定制输出样式。该函数原型及主要参数说明如下。
```shell
hist(x, bins=None, range=None, histtype='bar', orientation='vertical',
align='mid', rwidth=None, color=None, label=None, stacked=False, **kwargs)
```
- x：输入数据，是一个单独的数组或一组不需要具有相同长度的数组。
- bins：分段方式。如果是整数，表示分段数量；如果是从小到大的一维数组，则表示分割点，也可以是“auto”“fd”“doane”“scott”“stone”“rice”“sturges”“sqrt”等选项之一。
- range：一个二元组，表示统计范围。默认是x的值域范围。
- histtype：设置直方图的形状，有“bar”“barstacked”“step”“stepfilled”等4个选项。
- align：设置水平对齐方式，有“left”“right”“mid”等3个选项。
- rwidth：设置每一个数据条的相对宽度。
- orientation：设置直方图的方向，有“horizontal”和“vertical”两个选项。
- color：表示颜色或颜色数组，设置颜色。
- alpha：[0,1]之间的浮点数，设置透明度。
- stackedbool：设置是否允许堆叠。

以下代码使用hist( )函数分别绘制了堆叠样式的、无填充色的以及横向的直方图。
```shell
import numpy as np
from matplotlib import pyplot as plt
plt.rcParams['font.sans-serif'] = ['FangSong'] # 设置字体以便正确显示中文
plt.rcParams['axes.unicode_minus'] = False # 正确显示连字符

x1 = np.random.randn(1000)
x2 = np.random.randn(800)
c = ['r', 'b']
fig = plt.figure()
ax1 = fig.add_subplot(131)
ax1.set_title('两个数据集堆叠')
ax1.hist([x1,x2], bins=8, histtype='bar', color=c, stacked=True)

ax2 = fig.add_subplot(132)
ax2.set_title('单个数据集，无填充色')

ax2.hist(x1, bins=8, histtype='step')

ax3 = fig.add_subplot(133)
ax3.set_title('横向直方图')

ax3.hist([x1,x2], bins=8, histtype='bar', color=c, orientation='horizontal')

plt.show()
```

## 饼图
饼图以二维或三维形式显示每一个数值相对于全部数值之和的大小。Matplotlib API使用pie( )函数绘制饼图。该函数原型及主要参数说明如下。
```shell
pie(x, explode=None, labels=None, colors=None, autopct=None, shadow=False,
labeldistance=1.1, startangle=None, center=(0, 0))
```
- x：一维数组，表示输入数据。
- explode：与x等长的一维数组或为None，用于设置饼图各部分偏离中心点的距离相对于半径的比例。
- labels：与x等长的一维字符串数组或为None，用于设置饼图各部分的标签。
- colors：与x等长的一维颜色数组或为None，用于设置饼图各部分的颜色。
- autopct：一个函数或格式化字符串，用于标记饼图各部分的数值。
- shadow：一个布尔型参数，用于设置饼图是否显示阴影。
- labeldistance：一个浮点数，用于设置标签位置；如果为None，则可以使用legend( )函数显示图例。
- startangle：表示从x轴逆时针旋转饼图的起始角度，默认为0。
- center：一个二元组，用于设置饼图中心。

以下代码演示了使用pie( )函数绘制饼图的一般性用法。
```shell
import numpy as np
from matplotlib import pyplot as plt
plt.rcParams['font.sans-serif'] = ['FangSong'] # 设置字体以便正确显示中文
plt.rcParams['axes.unicode_minus'] = False # 正确显示连字符

x = np.array([10, 10, 5, 70, 2, 10])
labels = ['娱乐', '育儿', '饮食', '房贷', '交通', '其他']
explode = (0, 0, 0, 0.1, 0 ,0)
fig = plt.figure()
ax = fig.add_axes([0.1, 0.1, 0.8, 0.8])
ax.pie(x, explode=explode, labels=labels, autopct='%1.1f%%', startangle=150)
plt.show()
```

## 箱线图
箱线图一般用来表现数据的分布状况，如分布范围的上下限、上下四分位值、中位数等，也可以用箱线图来反映数据的异常情况，如离群值（异常值）等。
Matplotlib API使用boxplot( )函数绘制箱线图。该函数原型及主要参数说明如下。
```shell
boxplot(x, vert=None, whis=None, labels=None)
```
- x：表示输入数据，既可以是一维数组，也可以是二维数组。二维数组的每一列为一个数据集。
- vert：一个布尔型参数，用于设置箱线图是否是垂直方向。
- whis：一个浮点数或浮点数二元组，用于设置上下限的位置，超出上下限的值被视为异常值。
- labels：与数据集数量等长的一维字符串数组或为None，用于设置数据集的标签。

以下代码演示了使用boxplot( )函数绘制箱线图的一般性用法。
```shell
import numpy as np
from matplotlib import pyplot as plt
plt.rcParams['font.sans-serif'] = ['FangSong'] # 设置字体以便正确显示中文
plt.rcParams['axes.unicode_minus'] = False # 正确显示连字符

x = np.random.randn(500,4)
labels = list('ABCD')
fig = plt.figure()
ax1 = fig.add_subplot(121)
ax1.boxplot(x, labels=labels)
ax2 = fig.add_subplot(122)
ax2.boxplot(x, labels=labels, vert=False)
plt.show()
```
根据4个标准正态分布的数据集绘制的垂直和水平两种箱线图。

### 绘制图像
无论是灰度图像数据，还是RGB或RGBA的彩色图像数据，都可以使用Matplotlib API提供的imshow( )函数绘制成图像。
对于二维的非图像数据，也可以映射为灰度或伪彩色图像，因此imshow( )函数也常被用来绘制热力图。该函数原型及主要参数说明如下。
```shell
imshow(X, cmap=None, aspect=None, alpha=None, origin=None)
```
- X：表示数据，既可以是二维数组（灰度图像数据或数值数据），也可以是三维数组（RGB或RGBA彩色图像数据）。
- cmap：用于指定映射调色板，将标量数据映射为颜色。对于RGB（A）彩色图像数据，此参数将被忽略。
- aspect：用于控制轴的纵横比，有“equal”和“auto”两个选项。默认为“equal”，即宽高比为1。
- alpha：0～1的浮点数，或是一个和X的结构相同的数组，表示像素的透明度。
- origin：用于设置首行数据显示在上方还是下方，有“upper”和”lower”两个选项，默认为“upper”。

以下代码生成了一幅渐变的RGBA彩色图像，每个像素的透明度随机生成，然后使用imshow( )函数绘制成图。
```shell
import numpy as np
from matplotlib import pyplot as plt
plt.rcParams['font.sans-serif'] = ['FangSong'] # 设置字体以便正确显示中文
plt.rcParams['axes.unicode_minus'] = False # 正确显示连字符

r = np.tile(np.linspace(128,255, 10, dtype=np.uint8), (20,1)).T # 红色通道
g = np.tile(np.linspace(128,255, 20, dtype=np.uint8), (10,1)) # 绿色通道
b = np.ones((10,20), dtype=np.uint8)*96 # 蓝色通道
a = np.random.randint(100, 256, (10,20), dtype=np.uint8) # 透明通道
v = np.dstack((r,g,b,a)) # 合成RGBA彩色图像数据
fig = plt.figure()
ax = fig.add_axes([0.1, 0.1, 0.8, 0.8])
ax.imshow(v)
plt.show()
```
纵轴上的数值（对应数组的行）越大越靠近横轴，像素宽高自动保持等比例。

## 极坐标绘图
极坐标系不同于笛卡儿坐标系，极坐标系中的点是由角度和距离来表示的。有些曲线，如椭圆或螺线，适合用极坐标方程来表示。如果使用普通的绘图函数，就需要先把极坐标方程转为笛卡儿直角坐标方程。
幸好Matplotlib API提供了一些极坐标绘图函数，从而可以直接使用极坐标方式绘图。这些函数的原型如下。
```shell
polar(theta, r, **kwargs)
thetagrids(angles, labels=None, fmt=None, **kwargs)
rgrids(radii, labels=None, angle=22.5, fmt=None, **kwargs)
```
polar( ) 函数相当于绘制曲线的plot( )函数，只是将x和y坐标替换成了极坐标的角度theta和半径r，其他参数大致相同。thetagrids( ) 函数用于标注极坐标角度，rgrids( ) 函数用于标注极坐标半径。
以下代码以极坐标方式绘制了3条曲线，并标注了8个方位。
```shell
import numpy as np
from matplotlib import pyplot as plt
plt.rcParams['font.sans-serif'] = ['FangSong'] # 设置字体以便正确显示中文
plt.rcParams['axes.unicode_minus'] = False # 正确显示连字符

theta = np.linspace(0, 2*np.pi, 500)
r1 = 1.4*np.cos(5*theta)
r2 = 1.8*np.cos(4*theta)
r3 = theta/5
angles = range(0, 360, 45)
tlabels = ('东','东北','北','西北','西','西南','南','东南')
plt.polar(theta, r1, c='r')
plt.polar(theta, r2, lw=2)
plt.polar(theta, r3, ls='--', lw=3)
plt.thetagrids(angles, tlabels)
plt.rgrids(np.arange(-1.5, 2, 0.5))
plt.show()
```

## 风格和样式
Matplotlib是一个“博大精深”的绘图库，除了基本的绘图函数，还支持大量的格式控制和风格设置函数。前面几节重点讲解如何绘制一些简单的图形，如果想精确地控制图像输出，绘制出需要的图形样式，还需要学习一些技巧。

### 画布设置
绘图前需要先创建画布，最方便的生成方法是实例化pyplot模块的figure类。如果想要精准定义画布，就必须掌握设置画布的主要参数。

需要注意的是，使用savefig( )函数保存图像时，默认facecolor和edgecolor为白色，而不是画布当前的facecolor和edgecolor设置。如果需要修改保存这两个参数的当前设置，则需要重新设置savefig( )函数的这两个参数。

### 子图布局
使用画布的add_subplot( )函数可以在画布上添加多个子图，实现简单的网格布局。Matplotlib还提供了一个专用的子图布局函数GridSpec( )，可以实现任意复杂的布局需求。
以下代码演示了GridSpec( )函数的使用方法。
```shell
from matplotlib import pyplot as plt
import matplotlib.gridspec as gspec
fig = plt.figure()
gs = gspec.GridSpec(3, 4, width_ratios=[1,1,2,1], height_ratios=[1,1,1])
ax1 = fig.add_subplot(gs[0, :-1])
ax2 = fig.add_subplot(gs[:-1, -1])
ax3 = fig.add_subplot(gs[1:, :2])
ax4 = fig.add_subplot(gs[1, 2])
ax5 = fig.add_subplot(gs[-1, 2:])
plt.show()
```

### 颜色
Matplotlib支持灰度、彩色、带透明通道的彩色等多种颜色的表示方法，归纳起来有以下4种。
- RGB或RGBA浮点型元组，值域范围在[0, 1]。例如，(0.1, 0.2, 0.5)和(0.1, 0.2, 0.5,0.3)都是合法的颜色表示。
- RGB或RGBA十六进制字符串，例如，“#0F0F0F”和“#0F0F0F0F”都是合法的颜色表示。
- 使用[0, 1]的浮点值的字符串表示灰度，例如，“0.5”表示中灰。
- 使用颜色名称或名称缩写的字符串。
```
蓝色：'b' (blue)。
绿色：'g' (green)。
红色：'r' (red)。
墨绿：'c' (cyan)。
洋红：'m' (magenta)。
黄色：'y' (yellow)。
黑色：'k'(black)。
白色：'w'(white)。
```

### 线条和点的样式
使用plot( )函数或对数函数绘制线条时，可以使用linestyle或ls参数设置线条样式。

使用plot( )函数或对数函数绘制线条时，使用marker参数可以将数据点用特殊的形状标识出来。

下面的代码演示了常用的线条和点的样式
```shell
import numpy as np
from matplotlib import pyplot as plt
x = np.linspace(0, 2*np.pi, 15)
y = np.sin(x)
plt.plot(x, y, ls='-', marker='o', label="ls='-', marker='o'")

plt.plot(x, y+0.1, ls='--', marker='x', label="ls='--', marker='x'")

plt.plot(x, y+0.2, ls=':', marker='v', label="ls=':', marker='v'")

plt.plot(x, y+0.3, ls='-.', marker='*', label="ls='-.', marker='*'")

plt.plot(x, y+0.4, ls='none', marker='D', label="ls='none', marker='D'")

plt.legend()

plt.show()
```

### 坐标轴
除了画布和子图，坐标轴（axis）也是一种容器。 Matplotlib提供了定制坐标轴的功能，使用该功能既可以轻松设置坐标轴的范围、反转坐标轴，也可以实现双x轴或双y轴的显示效果。

#### 设置坐标轴范围
使用axes.set_xlim(left, right)和axes.set_ylim(bottom, top)函数设置x轴与y轴的显示范围。函数参数分别是能够显示的最小值和最大值，如果最大值或最小值为None，则表示只限制坐标轴一端的值域范围。
```shell
import numpy as np
from matplotlib import pyplot as plt
x = np.linspace(0, 2*np.pi, 100)
y = np.sin(x)
fig = plt.figure()
ax1 = fig.add_subplot(121)
ax1.plot(x, y, c='r')
ax1.set_ylim(-0.8, 0.8)
ax2 = fig.add_subplot(122)
ax2.plot(x, y, c='g')
ax2.set_xlim(-np.pi, 3*np.pi)
plt.show()
```

#### 反转坐标轴
使用axes.invert_xaxis( )函数和axes.invert_yaxis( )函数可分别反转x轴和y轴。这两个函数均不需要任何参数。

下面的例子使用图像绘制函数imshow( )来演示反转轴。imshow( )函数通过origin参数实现y轴反转，这里将其固定为“lower”。
```shell
import numpy as np
from matplotlib import pyplot as plt
plt.rcParams['font.sans-serif'] = ['FangSong']
plt.rcParams['axes.unicode_minus'] = False
y, x = np.mgrid[-2:2:200j, -3:3:300j]
z = np.exp(-x**2-y**2) - np.exp(-(x-1)**2-(y-1)**2)
fig = plt.figure()
ax1 = fig.add_subplot(221)
ax2 = fig.add_subplot(222)
ax3 = fig.add_subplot(223)
ax4 = fig.add_subplot(224)
ax1.imshow(z, cmap='jet', origin='lower')
ax2.imshow(z, cmap='jet', origin='lower')
ax3.imshow(z, cmap='jet', origin='lower')
ax4.imshow(z, cmap='jet', origin='lower')
ax2.invert_xaxis()
ax3.invert_yaxis()
ax4.invert_xaxis()
ax4.invert_yaxis()
ax1.set_title("正常的x轴、y轴")
# Text(0.5, 1.0, '正常的x轴、y轴')
ax2.set_title("反转x轴")
# Text(0.5, 1.0, '反转x轴')
ax3.set_title("反转y轴")
# Text(0.5, 1.0, '反转y轴')
ax4.set_title("反转x轴、y轴")
# Text(0.5, 1.0, '反转x轴、y轴')
plt.show()
```

#### 显示双轴
Matplotlib支持在一个子图上显示两个x轴或两个y轴。使用axes.twinx( )函数可显示双x轴，使用axes.twiny( )函数可显示双y轴。

以下代码演示了使用axes.twiny( )函数显示双y轴
```shell
import numpy as np
from matplotlib import pyplot as plt
plt.rcParams['font.sans-serif'] = ['FangSong']
plt.rcParams['axes.unicode_minus'] = False

x = np.linspace(-2*np.pi, 2*np.pi, 200)
y1 = np.square(x)
y2 = np.cos(x)
fig = plt.figure()
ax = fig.add_axes([0.1, 0.1, 0.8, 0.8])
ax_twinx = ax.twinx()
ax.plot(x, y1, c='r')
ax_twinx.plot(x, y2, c='g', ls='-.')
plt.show()
```

### 刻度
刻度（tick）是Matplotlib的4个容器（其他3个容器分别是画布、子图和坐标轴）中最复杂的一个，也是绘图时定制需求最多的一个。常用的刻度操作有设置主副刻度、设置刻度显示密度、设置刻度文本样式、设置刻度文本内容等。

#### 设置主副刻度
主副刻度常用于日期时间轴，如主刻度显示年份，副刻度显示月份。非线性的对数轴往往也需要显示副刻度。子图提供了4个函数来设置x轴和y轴的主副刻度。
- ax.xaxis.set_major_locator(locator)：用于设置x轴主刻度。
- ax.xaxis.set_minor_locator(locator)：用于设置x轴副刻度。
- ax.yaxis.set_major_locator(locator)：用于设置y轴主刻度。
- ax.yaxis.set_minor_locator(locator)：用于设置y轴副刻度。

函数的locator参数实例有两种，分别是来自ticker和dates两个子模块中有关刻度的子类实例。下面的代码演示了在x轴上设置日期时间相关的主副刻度。
```shell
import numpy as np
from matplotlib import pyplot as plt
import matplotlib.dates as mdates
x = np.arange('2019-01', '2019-06', dtype='datetime64[D]')
y = np.random.rand(x.shape[0])
fig = plt.figure()
ax = fig.add_axes([0.1, 0.1, 0.8, 0.8])
ax.plot(x, y, c='g')
ax.xaxis.set_major_locator(mdates.MonthLocator())
ax.xaxis.set_major_formatter(mdates.DateFormatter('\n%Y-%m-%d'))
ax.xaxis.set_minor_locator(mdates.DayLocator(bymonthday=(1,11,21)))
ax.xaxis.set_minor_formatter(mdates.DateFormatter('%d'))
plt.show()
```

#### 设置刻度显示密度
Matplotlib的ticker子模块包含的Locator类是所有刻度类的基类，负责根据数据的范围自动调整视觉间隔以及刻度位置的选择。MultipleLocator是Locator的派生类，能够控制刻度的疏密。
```shell
import numpy as np
from matplotlib import pyplot as plt
from matplotlib.ticker import MultipleLocator
plt.rcParams['font.sans-serif'] = ['FangSong']
plt.rcParams['axes.unicode_minus'] = False
x = np.linspace(0, 4*np.pi, 500)
fig = plt.figure()
ax1 = fig.add_subplot(121)
ax1.plot(x, np.sin(x), c='m')
ax1.yaxis.set_major_locator(MultipleLocator(0.3))
ax2 = fig.add_subplot(122)
ax2.plot(x, np.sin(x), c='m')
ax2.xaxis.set_major_locator(MultipleLocator(3))
ax2.xaxis.set_minor_locator(MultipleLocator(0.6))
ax1.set_title('x轴刻度自动调整，y轴设置刻度间隔0.3')
ax2.set_title('x轴设置主刻度间隔3副刻度间隔0.6，y轴刻度自动调整')
plt.show()
```

#### 设置刻度文本样式
设置刻度文本的颜色、字体、字号或旋转文本等样式，需要使用axes.get_xticklabels( )或axes.get_yticklabels( )函数获取x轴或y轴的文本对象列表。
文本对象中包含设置文本大小、颜色、旋转角度的函数，使用对应函数即可完成设置。
```shell
import numpy as np
from matplotlib import pyplot as plt
import matplotlib.dates as mdates
x = np.arange('2019-01', '2020-01', dtype='datetime64[D]')
y = np.random.rand(x.shape[0])
fig = plt.figure()
ax = fig.add_axes([0.1, 0.3, 0.8, 0.6])
ax.plot(x, y, c='g')
ax.xaxis.set_major_locator(mdates.MonthLocator())
ax.xaxis.set_major_formatter(mdates.DateFormatter('%Y/%m/%d'))
for lobj in ax.get_xticklabels():
    lobj.set_rotation(35)
    lobj.set_size(12)
    lobj.set_color('blue')

plt.show()
```

### 设置刻度文本内容
在有些应用场景中，需要将x轴或x轴刻度上的文字设置为更有标识意义的内容。使用set_xticklabels( )和set_yticklabels( )函数可以替换刻度文本，不过只适用于所有可能的取值都已经显示在坐标轴上的情况。
例如，x轴对应的是列表[0,1,2,3]，共4个取值，显示的刻度也是4个，此时可以使用['一季度','二季度','三季度','四季度']替换对应的数值。

Matplotlib提供了强大的刻度文本格式化功能，ticker.Formatter作为基类派生出了多种形式的格式化类，FuncFormatter就是其中之一。使用FuncFormatter可以更加灵活地设置刻度文本内容。
```shell
import numpy as np
from matplotlib import pyplot as plt
from matplotlib.ticker import FuncFormatter
plt.rcParams['font.sans-serif'] = ['FangSong'] # 设置字体以便正确显示中文
plt.rcParams['axes.unicode_minus'] = False # 正确显示连字符
x = np.linspace(0, 10, 200)
y = np.square(x)
def func_x(x, pos):
    return '%0.2f秒'%x

def func_y(y, pos):
    return '%0.2f°C'%y

formatter_x = FuncFormatter(func_x)
formatter_y = FuncFormatter(func_y)
fig = plt.figure()
ax = fig.add_axes([0.1, 0.1, 0.8, 0.8])
ax.plot(x, y, c='r')
ax.xaxis.set_major_formatter(formatter_x)
ax.yaxis.set_major_formatter(formatter_y)
plt.show()
```

### 文本
Matplotlib默认配置中的首选字体为西文字体，如果不做修改，使用中文字体时就会出现问题。
修改默认配置是一个解决方法，更普遍的做法是在绘图前，先设置默认的字体为系统可用的一种中文字体，其代码如下。
```shell
from matplotlib import pyplot as plt
plt.rcParams['font.sans-serif'] = ['FangSong'] # 设置字体以便正确显示中文
plt.rcParams['axes.unicode_minus'] = False # 正确显示连字符
```
如果一个画布上有多个子图，有一个统一的标题是非常有必要的。使用figure.suptitle( )函数可以设置画布标题。
该函数的主要参数中：text参数为标题文本，支持LaTex语法；其他默认参数用于设置标题样式。该函数原型如下。
```shell
suptitle(text, x=0.5, y=0.98, ha='center', va='top', size=None, weight=None)
```
画布上的每个子图可以使用axes. set_title( )函数单独设置子图标题。该函数的主要参数包括：label参数为标题文本，支持LaTex语法；
fontdict参数是一个设置标题字体、字号、颜色等属性的字典；loc参数用于设置标题水平对齐方式，默认居中对齐；
pad参数用于设置标题距离子图顶部的距离。
```shell
axes.set_title(label, fontdict=None, loc='cneter', pad=6.0)
```
使用axes. set_xlabel( )和axes.set_ylabel( )函数可以分别设置x轴和y轴的名称。这两个函数的使用几乎完全一致，主要参数包括：
label参数为坐标轴名称，支持LaTex语法；fontdict参数是一个设置坐标轴名称的字体、字号、颜色等属性的字典；
labelpad参数用于设置坐标轴名称距离子图边框的距离，其函数原型如下。
```shell
axes.set_ xlabel(label, fontdict=None, labelpad=None)
axes.set_ ylabel(label, fontdict=None, labelpad=None)
```
axes.text( )函数可以在子图中的任意位置上绘制文本；figure.figtext( )函数可以在画布的任意位置上绘制函数。axes.annotate( )函数可以在子图中的任意位置显示文字。
与axes.text( )函数不同的是，axes.annotate( )函数除了可以设置显示字体的位置外，还可以设置一个箭头指向的位置。
```shell
import numpy as np
from matplotlib import pyplot as plt
plt.rcParams['font.sans-serif'] = ['FangSong']
plt.rcParams['axes.unicode_minus'] = False

fig = plt.figure()
fig.suptitle('画布标题')
ax = fig.add_axes([0.1, 0.2, 0.8, 0.6])
ax.set_title('子图标题')
ax.set_xlabel('x轴标注文本', fontdict={'fontsize':14})
ax.set_ylabel('y轴标注文本', fontdict={'fontsize':14})
ax.text(3, 8, '边框颜色', style='italic', bbox={'facecolor':'red', 'alpha':0.5, 'pad':10})
ax.text(2, 6, r'LaTex: $E=mc^2$', fontsize=15)
ax.text(0.95, 0.01, '坐标内绿色文字', verticalalignment='bottom',
horizontalalignment='right', transform=ax.transAxes, color='green', fontsize=15)
ax.plot([2], [1], 'o')
ax.annotate('annotate文字', xy=(2, 1), xytext=(3, 4),
arrowprops={'facecolor':'black', 'shrink':0.05})
ax.set_xlim(0,10)
(0, 10)
ax.set_ylim(0,10)
(0, 10)
plt.show()
```

### 图例
如果在一个子图上绘制多个图形元素，通常需要添加图例以便区分不同的图形元素。
在Matpoltlib中，使用axes.legend( )函数来生成图例。

### 网格设置
网格是坐标刻度的辅助观察线，可以显示主要刻度、次要刻度或两者都显示的网格，同时可以设置网格线条的颜色样式等。Matplotlib使用axes.grid( )函数设置子图中的网格标尺。

## Matplotlib扩展
虽然Matplotlib主要用于绘图，并且主要是绘制二维的图形，但是它也有一些不同的扩展，使得我们可以在地理图上绘图，可以把Excel和3D（三维）图表结合起来。在Matplotlib的生态圈里，这些扩展被称为工具包（toolkits）。工具包是针对某个主题或领域（如3D绘图）的特定函数的集合。
比较流行的工具包有Basemap、GTK工具、Excel工具、Natgrid、AxesGrid和mplot3d。

### 使用Basemap绘制地图
Basemap是Matplotlib的扩展工具包之一，用来在地图上绘制各种图形。Basemap在气象、地理、地球物理等学科中有着广泛的应用。
Basemap可以直接使用pip命令安装，安装命令会自动下载安装匹配的版本。
```shell
pip install basemap
```
使用Basemap绘制带有地图的图像前，要先创建一个Basemap实例，而实例化Basemap需要传入一个重要的参数，就是子图。
传入子图后即可使用这个Basemap实例进行绘图。Basemap常用的绘图函数有以下几种。
- Basemap.drawcoastlines( )：绘制海岸线。线宽参数linewidth默认为1.0，线型参数linestyle默认为solid，颜色参数color默认为k（黑色）。
- Basemap.drawcounties( )：绘制国界线。线宽参数linewidth默认为0.5，线型参数linestyle默认为solid，颜色参数color默认为k（黑色）。
- Basemap.drawrivers( )：绘制主要河流。线宽参数linewidth默认为0.5，线型参数linestyle默认为solid，颜色参数color默认为k（黑色）。
- Basemap.drawparallels( )：绘制纬度线。线宽参数linewidth默认为1.0，线型参数dashes默认为[1,1]，颜色参数color默认为k（黑色），文本颜色参数textcolor默认为k（黑色）。
- Basemap.drawmeridians( )：绘制经度线。线宽参数linewidth默认为1.0，线型参数dashes默认为[1,1]，颜色参数color默认为k（黑色），文本颜色参数textcolor默认为k（黑色）。
- Basemap.scatter( )：在地图上用标记绘制点。参数x和y表示点的位置。如果latlon关键字设置为True，则x、y表示经度和纬度（以度为单位）；如果latlon为False（默认值），则x和y被假定为映射投影坐标。该函数支持的关键字参数包括cmap、alpha、marker、edgecolors等，和axes.scatter( )函数完全一致。

## 3D绘图工具包
3D绘图工具包mplot3d已经成为Matplotlib的标准工具包，被包含在mpl_toolkits模块内，无须另外进行安装。
导入mplot3d工具包，即可开始绘制三维图形。mplot3d常用的绘图函数有以下几种。
- Axes3D.plot(xs, ys, *args, **kwargs)：绘制2D或3D曲线，这取决于是否给出zs参数。参数xs、ys和zs是一组连续点的坐标集，函数按照点的顺序用直线连接所有点。该函数支持的关键字参数包括color（c）、alpha等，和axes.plot( )函数完全一致。
- Axes3D.scatter(xs, ys, zs=0, s=20, c=None, *args, **kwargs)：绘制3D散点图。参数xs、ys和zs是一组散列点的坐标集。参数s表示点的大小，既可以是标量，也可以是和点的数量匹配的数组。参数c既可以是单个颜色名称，也可以是和点的数量匹配的颜色数组或数值数组；若是数值数组，则需要cmap参数指定的颜色映射调色板将其映射为点的颜色。该函数支持的关键字参数包括cmap、alpha、marker、edgecolors等，和axes.scatter( )函数完全一致。
- Axes3D.plot_surface(X, Y, Z, *args, **kwargs)：绘制曲面。参数X和Y是二维数组，对应二维平面网格的x和y坐标集。参数Z是二维平面网格上各个点的值，同时Z值可以被由cmap参数指定的颜色映射调色板映射为点的颜色。颜色也可以使用参数color或facecolors指定，前者指定曲面颜色，后者指定每个点的颜色。
- Axes3D.quiver(X, Y, Z, U, V, W, **kwargs)：绘制矢量合成图，以箭头方向表示矢量方向，以箭头长度表示矢量大小。参数X、Y、Z表示矢量箭头的位置坐标数据集，参数U、V、W表示矢量在x、y、z这3个坐标轴方向上的分量。

以下代码以绘制曲面为例，演示了3D绘图工具包的一般性使用方式。
```shell
import numpy as np
from matplotlib import pyplot as plt
plt.rcParams['font.sans-serif'] = ['FangSong']
plt.rcParams['axes.unicode_minus'] = False

import numpy as np
from matplotlib import pyplot as plt
from mpl_toolkits.mplot3d import Axes3D
y, x = np.mgrid[-5:5:100j, -5:5:100j]
r = np.hypot(x, y)
z = np.sin(r)
fig = plt.figure()
ax = fig.add_axes([0.1, 0.1, 0.8, 0.8], projection='3d')
surf = ax.plot_surface(x, y, z, cmap='hsv', linewidth=0, antialiased=False)
fig.colorbar(surf, shrink=0.5, aspect=5)
plt.show()
```
