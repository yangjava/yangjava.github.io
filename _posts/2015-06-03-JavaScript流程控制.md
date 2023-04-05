---
layout: post
categories: JavaScript
description: none
keywords: JavaScript
---
# JavaScript流程控制
JavaScript有其自身的表达式和算术运算符。

- JavaScript的表达式与运算符，主要内容包括赋值表达式与运算符、算术表达式与运算符、比较运算符以及运算符优先级等。

## JavaScript语句的格式
JavaScript语句通常是以分号作为结束符的，但是如前所述，分号并不是必需的。如果处理JavaScript的应用程序判断语句是完整的，并且每行都以换行符结尾，那么就可以忽略分号：
```
var bValue = true
var sValue = "this is also true"
```
如果在同一行中包含多个语句，那么就必须使用分号来结束不同的语句：
```
var bValue = true; var sValue = "this is also true"
```

## 赋值语句
JavaScript中最常见的语句是赋值语句。赋值语句是由左边的变量、紧跟着的赋值运算符（=）以及右边的数值组成的。

右边的表达式可以是数值，或者是变量和字面量以及任意个运算符所组成的表达式，例如：
```shell
nValue = 35.00;
nValue = nValue + 35.00;
```
语句右边的表达式也可以是函数调用：
```
nValue = someFunction();
```
多条赋值语句可以放在同一行，当然每条赋值语句之间要用分号隔开，例如：
```
var firstName = 'Shelley'; var lastName = 'Powers';
```
还可以一次性将相同的数值赋给多个变量：
```
var firstName = lastName = middleName = "";
```
然而，如果用逗号将变量隔开，如下面的代码所示，那么将会给第二个变量赋值，而第一个变量将是undefined：
```
var nValue1,nValue2 = 3;　　// nValue1是undefined
```
也可以通过逗号隔开多个变量的赋值，例如：
```
var nValue1=3, nValue2=4, nValue3=5;
```
为了提高代码可读性，并且避免可能的错误，最好的选择是通过分号隔开每个赋值语句。

## 算术运算符
数值计算常常需要使用算术运算符，它也可以用于给变量赋值，或者作为函数或方法的参数。例如，下面的代码将把两个变量的值相加，并将结果赋给第三个变量：
```
var theResult = varValue1 + varValue2;
```
该操作是二元算术表达式的示例，算术运算符的两边分别是一个操作数，并产生一个新结果。下面给出更为复杂的示例，例如，任意个算术运算符，以及字面量值和变量的任意组合：
```
nValue = nValue + 30.00 /　2 - nValue2 * 3;
```
操作数是在执行算术运算时所需要的数值。

表达式中用到的运算符如下所示，这些运算符经常在数学类和在线计算器等地方可以看到：
- +　加法运算
- −　减法运算
- *　乘法运算
- /　除法运算
- %　取余运算




















