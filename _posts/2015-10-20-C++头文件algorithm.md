---
layout: post
categories: [C++]
description: none
keywords: C++
---
# C++头文件algorithm
#include< algorithm >是C++的标准模版库（STL）中最重要的头文件之一，提供了大量基于迭代器的非成员模板函数。

## 常用的库函数
max(),min()和swap()

- max(x,y)　　//返回两个元素中值最大的元素
- min(x,y)　　//返回两个元素中值最小的元素
- swap(x,y)　　//用来交换x和y的值
```
/*
使用algorithm头文件，需要在头文件下加一行using namespace std;” 
*/ 
 
//常用函数max(), min(), abs()
//swap()
//reverse()
//next_permutation()
//fill()
// sort()
//lower_bound和upper_bound()
 
 
/*max(), min(), abs()的使用*/
#include<stdio.h>
#include<algorithm>
#include<iostream>
using namespace std;
 
int main()
{
    int x =1, y =-2;
    cout <<max(x,y)<< " "<< min(x,y)<<endl;
    cout << abs(x)<<" "<<abs(y)<<endl;
 
return 0;   
} 
```

### reverse()
反转排序指定范围中的元素，reverse(a,b) 可以将数组指针在[a,b)之间的元素或容器的迭代器在[a,ib)范围内的元素进行反转。
```
 /*对整形数组逆转*/
#include<algorithm>
#include<iostream>
 
using namespace std;
 
int main()
{
    int a[10]= {10, 11, 12, 13, 14, 15}; 
    reverse(a, a+4);//a[0]~a[3]反转
    for(int i=0; i<6; i++){
        cout<<a[i]<<" ";
    } 
    return 0;
}
 
 
/*对容器中的元素（例如string字符串）进行反转，结果也一样*/
 
 
#include<iostream>
#include<string>
#include<algorithm>
 
using namespace std;
 
int main()
{
    string str = "abcdefghi";
    reverse(str.begin()+2, str.begin()+6);//对a[2]~a[5]逆转*左闭右开* 
    for( int i=0; i < str.length(); i++){
        cout<<str[i]<<" ";
    }   
    return 0;
}
```

### fill()
fill() 可以把数组或容器中的某一段区间赋为某个相同的值。
```
/*
fill()可以把数组或容器中的某一段区间赋为某个相同的值。和memset不同，这里的赋值可以是
数组类型对应范围中的任意值。示例如下： 
*/ 
 
#include<iostream>
#include<string>
#include<algorithm>
 
using namespace std;
 
int main()
{
    int a[5] = {1,2,3,4,5};
    fill( a, a+5, 233);//将a[0]~a[4]赋值3为233 
    for(int i=0;i<5;i++){
        cout<<a[i]<<endl;
    }       
    return 0;
}
```

### sort()
排序函数，默认为递增排序。
如果需要递减排序，需要增加一个比较函数：