---
layout: post
categories: Algorithm
description: none
keywords: Algorithm
---
# 冒泡算法（(Bubble Sort)）

劝君莫惜金缕衣，劝君惜取少年时。花开堪折直须折，莫待无花空折枝。。——杜秋娘《金缕衣》

### 冒泡排序

这个名词的由来很好理解，一般河水中的冒泡，水底刚冒出来的时候是比较小的，随着慢慢向水面浮起会逐渐增大，这物理规律我不作过多解释，大家只需要了解即可。

冒泡算法的运作规律如下：

①、比较相邻的元素。如果第一个比第二个大，就交换他们两个。

②、对每一对相邻元素作同样的工作，从开始第一对到结尾的最后一对。这步做完后，最后的元素会是最大的数（也就是第一波冒泡完成）。

③、针对所有的元素重复以上的步骤，除了最后一个。

④、持续每次对越来越少的元素重复上面的步骤，直到没有任何一对数字需要比较。

![Java冒泡排序](png\Java\Java冒泡排序.png)

**代码如下：**

```
package com.demo.sort;
 
public class BubbleSort {
    public static int[] sort(int[] array){
        //这里for循环表示总共需要比较多少轮
        for(int i = 1 ; i < array.length; i++){
            //设定一个标记，若为true，则表示此次循环没有进行交换，也就是待排序列已经有序，排序已经完成。
            boolean flag = true;
            //这里for循环表示每轮比较参与的元素下标
            //对当前无序区间array[0......length-i]进行排序
            //j的范围很关键，这个范围是在逐步缩小的,因为每轮比较都会将最大的放在右边
            for(int j = 0 ; j < array.length-i ; j++){
                if(array[j]>array[j+1]){
                    int temp = array[j];
                    array[j] = array[j+1];
                    array[j+1] = temp;
                    flag = false;
                }
            }
            if(flag){
                break;
            }
            //第 i轮排序的结果为
            System.out.print("第"+i+"轮排序后的结果为:");
            display(array);
             
        }
        return array;
         
    }
     
    //遍历显示数组
    public static void display(int[] array){
        for(int i = 0 ; i < array.length ; i++){
            System.out.print(array[i]+" ");
        }
        System.out.println();
    }
     
    public static void main(String[] args) {
        int[] array = {4,2,8,9,5,7,6,1,3};
        //未排序数组顺序为
        System.out.println("未排序数组顺序为：");
        display(array);
        System.out.println("-----------------------");
        array = sort(array);
        System.out.println("-----------------------");
        System.out.println("经过冒泡排序后的数组顺序为：");
        display(array);
    }
 
}
```

本来应该是 8 轮排序的，这里我们只进行了 7 轮排序，因为第 7 轮排序之后已经是有序数组了。

**冒泡排序解释：**

冒泡排序是由两个for循环构成，第一个for循环的变量 i 表示总共需要多少轮比较，第二个for循环的变量 j 表示每轮参与比较的元素下标【0,1，......，length-i】，因为每轮比较都会出现一个最大值放在最右边，所以每轮比较后的元素个数都会少一个，这也是为什么 j 的范围是逐渐减小的。相信大家理解之后快速写出一个冒泡排序并不难。

**冒泡排序性能分析：**

假设参与比较的数组元素个数为 N，则第一轮排序有 N-1 次比较，第二轮有 N-2 次，如此类推，这种序列的求和公式为：

（N-1）+（N-2）+...+1 = N*（N-1）/2

当 N 的值很大时，算法比较次数约为 N2/2次比较，忽略减1。

假设数据是随机的，那么每次比较可能要交换位置，可能不会交换，假设概率为50%，那么交换次数为 N2/4。不过如果是最坏的情况，初始数据是逆序的，那么每次比较都要交换位置。

交换和比较次数都和N2 成正比。由于常数不算大 O 表示法中，忽略 2 和 4，那么冒泡排序运行都需要 O(N2) 时间级别。

其实无论何时，只要看见一个循环嵌套在另一个循环中，我们都可以怀疑这个算法的运行时间为 O(N2)级，外层循环执行 N 次，内层循环对每一次外层循环都执行N次（或者几分之N次）。这就意味着大约需要执行N2次某个基本操作。