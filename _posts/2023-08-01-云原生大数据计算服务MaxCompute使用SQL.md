---
layout: post
categories: [Action]
description: none
keywords: Action
---
# 云原生大数据计算服务 MaxCompute使用SQL
MaxCompute SQL采用的是类似于SQL的语法。它的语法是标准语法ANSI SQL92的一个子集，并有自己的扩展。

## 关系运算符
- A=B  A或B为NULL，返回NULL。
- A<>B A或B为NULL，返回NULL。
- A IS NULL A为NULL，返回TRUE，否则返回FALSE。
```
SELECT * FROM user WHERE user_id = '0001'; 
SELECT * FROM user WHERE user_name <> 'maggie'; 
SELECT * FROM user WHERE age > '50'; 
SELECT * FROM user WHERE birth_day >= '1980-01-01 00:00:00'; 
SELECT * FROM user WHERE is_female is null; 
SELECT * FROM user WHERE is_female is not null; 
SELECT * FROM user WHERE user_id in (0001,0010); 
SELECT * FROM user WHERE user_name like 'M%';
```
由于DOUBLE类型存在一定的精度差，因此，不建议您直接使用等于号（=）对两个DOUBLE类型的数据进行比较。您可以将两个DOUBLE类型数据相减，然后取绝对值进行判断。当绝对值足够小时，认为两个DOUBLE数值相等，示例如下。
```
ABS(0.9999999999 - 1.0000000000) < 0.000000001
 -- 0.9999999999和1.0000000000为10位精度，而0.000000001为9位精度。
 -- 此时可以认为0.9999999999和1.0000000000相等。
```

## Lambda函数
Lambda是一种匿名函数，不需要命名，可以作为参数传递给其他函数或方法。

获取数组列的平方值
```
SELECT numbers,transform(numbers, n -> n * n) as squared_numbers
FROM (
    VALUES (ARRAY(1, 2)),(ARRAY(3, 4)),(ARRAY(5, 6, 7))
) AS t(numbers);
```

计算线性函数值
```
SELECT xvalues, a, b,
       transform(xvalues, x -> a * x + b) as linear_function_values
FROM (
    VALUES (ARRAY(1, 2), 10, 5),(ARRAY(3, 4), 4, 2)
) AS t(xvalues, a, b);
```


