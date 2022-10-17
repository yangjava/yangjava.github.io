# Java数据结构和算法

[TOC]

为什么要学习数据结构和算法，这里我举个简单的例子。

　　编程好比是一辆汽车，而数据结构和算法是汽车内部的变速箱。一个开车的人不懂变速箱的原理也是能开车的，同理一个不懂数据结构和算法的人也能编程。但是如果一个开车的人懂变速箱的原理，比如降低速度来获得更大的牵引力，或者通过降低牵引力来获得更快的行驶速度。那么爬坡时使用1档，便可以获得更大的牵引力；下坡时便使用低档限制车的行驶速度。回到编程而言，比如将一个班级的学生名字要临时存储在内存中，你会选择什么数据结构来存储，数组还是ArrayList，或者HashSet，或者别的数据结构。如果不懂数据结构的，可能随便选择一个容器来存储，也能完成所有的功能，但是后期如果随着学生数据量的增多，随便选择的数据结构肯定会存在性能问题，而一个懂数据结构和算法的人，在实际编程中会选择适当的数据结构来解决相应的问题，会极大的提高程序的性能。

# 数据结构

　　**数据结构是计算机存储、组织数据的方式，指相互之间存在一种或多种特定关系的数据元素的集合。**

　　通常情况下，精心选择的数据结构可以带来更高的运行或者存储效率。数据结构往往同高效的检索算法和索引技术有关。

### 数据结构的基本功能

　　**①、如何插入一条新的数据项**

　　**②、如何寻找某一特定的数据项**

　　**③、如何删除某一特定的数据项**

　　**④、如何迭代的访问各个数据项，以便进行显示或其他操作**

### 常用的数据结构

 ![Java常见数据结构](png\Java\Java常见数据结构.png)

　　这几种结构优缺点如下：先有个大概印象，后面会详细讲解！！！

　　**PS：我看下面的评论，对于下面所说的数组，插入块，查找、删除慢的特点有疑问。**

　　　　**这里可能是我没有描述清楚，对于数组，你们所说的查找快，我想只是随机查找快，因为知道数组下标，可以按索引获取任意值。但是你要查找某个特定值，对于无序数组，还是需要遍历整个数组，那么查找效率是O（n），效率是很低的（有序数组按照二分查找算法还是很快的）。**

　　　　**插入快，是在数组尾部进行插入，获取到数组的最后一个索引下标，加1进行赋值就可以了。**

　　　　**删除慢，除开尾部删除，在任意中间或者前面删除，后面的元素都要整体进行平移的，所以也是比较慢的。**

　　**综上所述：对于数组，随机查找快，数组尾部增删快，其余的操作效率都是很低的。**

　　![数据结构的优缺点](png\Java\数据结构的优缺点.png)

## 抽象数据类型（ADT）

在介绍抽象数据类型的时候，我们先看看什么是数据类型，听到这个词，在Java中我们可能首先会想到像 int,double这样的词，这是Java中的基本数据类型，一个数据类型会涉及到两件事：

　　①、拥有特定特征的数据项

　　②、在数据上允许的操作

　　比如Java中的int数据类型，它表示整数，取值范围为：-2147483648~2147483647，还能使用各种操作符，+、-、*、/ 等对其操作。数据类型允许的操作是它本身不可分离的部分，理解类型包括理解什么样的操作可以应用在该类型上。

　　那么当年设计计算机语言的人，为什么会考虑到数据类型？

　　我们先看这样一个例子，比如，大家都需要住房子，也都希望房子越大越好。但显然，没有钱，考虑房子没有意义。于是就出现了各种各样的商品房，有别墅的、复式的、错层的、单间的……甚至只有两平米的胶囊房间。这样做的意义是满足不同人的需要。

　　同样，在计算机中，也存在相同的问题。计算1+1这样的表达式不需要开辟很大的存储空间，不需要适合小数甚至字符运算的内存空间。于是计算机的研究者们就考虑，要对数据进行分类，分出来多种数据类型。比如int，比如float。

　　虽然不同的计算机有不同的硬件系统，但实际上高级语言编写者才不管程序运行在什么计算机上，他们的目的就是为了实现整形数字的运算，比如a+b等。他们才不关心整数在计算机内部是如何表示的，也不管CPU是如何计算的。于是我们就考虑，无论什么计算机、什么语言都会面临类似的整数运算，我们可以考虑将其抽象出来。抽象是抽取出事物具有的普遍性本质，是对事物的一个概括，是一种思考问题的方式。

　　**抽象数据类型（ADT）是指一个数学模型及定义在该模型上的一组操作。**它仅取决于其逻辑特征，而与计算机内部如何表示和实现无关。比如刚才说得整型，各个计算机，不管大型机、小型机、PC、平板电脑甚至智能手机，都有“整型”类型，也需要整形运算，那么整型其实就是一个抽象数据类型。 

　　更广泛一点的，比如我们刚讲解的栈和队列这两种数据结构，我们分别使用了数组和链表来实现，比如栈，对于使用者只需要知道pop()和push()方法或其它方法的存在以及如何使用即可，使用者不需要知道我们是使用的数组或是链表来实现的。

　　ADT的思想可以作为我们设计工具的理念，比如我们需要存储数据，那么就从考虑需要在数据上实现的操作开始，需要存取最后一个数据项吗？还是第一个？还是特定值的项？还是特定位置的项？回答这些问题会引出ADT的定义，只有完整的定义了ADT后，才应该考虑实现的细节。

　　这在我们Java语言中的接口设计理念是想通的。

## 数组(Array)

![数据结构-array](png\Java\数据结构-array.png)

数据结构的鼻祖——数组，可以说数组几乎能表示一切的数据结构，在每一门编程语言中，数组都是重要的数据结构，当然每种语言对数组的实现和处理也不相同，但是本质是都是用来存放数据的的结构，这里我们以Java语言为例，来详细介绍Java语言中数组的用法。

### Java数组介绍

　　在Java中，数组是用来存放同一种数据类型的集合，注意只能存放同一种数据类型(Object类型数组除外)。

　　**①、数组的声明**

　　**第一种方式：**

```
数据类型 []  数组名称 = new 数据类型[数组长度];
```

　　这里 [] 可以放在数组名称的前面，也可以放在数组名称的后面，我们推荐放在数组名称的前面，这样看上去 数据类型 [] 表示的很明显是一个数组类型，而放在数组名称后面，则不是那么直观。

　　**第二种方式：**

```
数据类型 [] 数组名称 = {数组元素1，数组元素2，......}
```

　　这种方式声明数组的同时直接给定了数组的元素，数组的大小由给定的数组元素个数决定。

```
//声明数组1,声明一个长度为3，只能存放int类型的数据
int [] myArray = new int[3];
//声明数组2,声明一个数组元素为 1,2,3的int类型数组
int [] myArray2 = {1,2,3};
```

　　**②、访问数组元素以及给数组元素赋值**

　　数组是存在下标索引的，通过下标可以获取指定位置的元素，数组小标是从0开始的，也就是说下标0对应的就是数组中第1个元素，可以很方便的对数组中的元素进行存取操作。

　　前面数组的声明第二种方式，我们在声明数组的同时，也进行了初始化赋值。

```
//声明数组,声明一个长度为3，只能存放int类型的数据
int [] myArray = new int[3];
//给myArray第一个元素赋值1
myArray[0] = 1;
//访问myArray的第一个元素
System.out.println(myArray[0]);
```

　　上面的myArray 数组，我们只能赋值三个元素，也就是下标从0到2，如果你访问 myArray[3] ，那么会报数组下标越界异常。

　　**③、数组遍历**

　　数组有个 length 属性，是记录数组的长度的，我们可以利用length属性来遍历数组。

```
//声明数组2,声明一个数组元素为 1,2,3的int类型数组
int [] myArray2 = {1,2,3};
for(int i = 0 ; i < myArray2.length ; i++){
    System.out.println(myArray2[i]);
}
```

### 用类封装数组实现数据结构

数据结构必须具有以下基本功能：

　　**①、如何插入一条新的数据项**

　　**②、如何寻找某一特定的数据项**

　　**③、如何删除某一特定的数据项**

　　**④、如何迭代的访问各个数据项，以便进行显示或其他操作**

　　而我们知道了数组的简单用法，现在用类的思想封装一个数组，实现上面的四个基本功能：

　　ps:假设操作人是不会添加重复元素的，这里没有考虑重复元素，如果添加重复元素了，后面的查找，删除，修改等操作只会对第一次出现的元素有效。

```
package com.demo.array;

public class MyArray {
    //定义一个数组
    private int [] intArray;
    //定义数组的实际有效长度
    private int elems;
    //定义数组的最大长度
    private int length;

    //默认构造一个长度为50的数组
    public MyArray(){
        elems = 0;
        length = 50;
        intArray = new int[length];
    }
    //构造函数，初始化一个长度为length 的数组
    public MyArray(int length){
        elems = 0;
        this.length = length;
        intArray = new int[length];
    }

    //获取数组的有效长度
    public int getSize(){
        return elems;
    }

    /**
     * 遍历显示元素
     */
    public void display(){
        for(int i = 0 ; i < elems ; i++){
            System.out.print(intArray[i]+" ");
        }
        System.out.println();
    }

    /**
     * 添加元素
     * @param value,假设操作人是不会添加重复元素的，如果有重复元素对于后面的操作都会有影响。
     * @return添加成功返回true,添加的元素超过范围了返回false
     */
    public boolean add(int value){
        if(elems == length){
            return false;
        }else{
            intArray[elems] = value;
            elems++;
        }
        return true;
    }

    /**
     * 根据下标获取元素
     * @param i
     * @return查找下标值在数组下标有效范围内，返回下标所表示的元素
     * 查找下标超出数组下标有效值，提示访问下标越界
     */
    public int get(int i){
        if(i<0 || i>elems){
            System.out.println("访问下标越界");
        }
        return intArray[i];
    }
    /**
     * 查找元素
     * @param searchValue
     * @return查找的元素如果存在则返回下标值，如果不存在，返回 -1
     */
    public int find(int searchValue){
        int i ;
        for(i = 0 ; i < elems ;i++){
            if(intArray[i] == searchValue){
                break;
            }
        }
        if(i == elems){
            return -1;
        }
        return i;
    }
    /**
     * 删除元素
     * @param value
     * @return如果要删除的值不存在，直接返回 false;否则返回true，删除成功
     */
    public boolean delete(int value){
        int k = find(value);
        if(k == -1){
            return false;
        }else{
            if(k == elems-1){
                elems--;
            }else{
                for(int i = k; i< elems-1 ; i++){
                    intArray[i] = intArray[i+1];

                }
                 elems--;
            }
            return true;
        }
    }
    /**
     * 修改数据
     * @param oldValue原值
     * @param newValue新值
     * @return修改成功返回true，修改失败返回false
     */
    public boolean modify(int oldValue,int newValue){
        int i = find(oldValue);
        if(i == -1){
            System.out.println("需要修改的数据不存在");
            return false;
        }else{
            intArray[i] = newValue;
            return true;
        }
    }

} 
```

**优点**

- 构建非常简单
- 能在 O(1) 的时间里根据数组的下标（index）查询某个元素

**缺点**

- 构建时必须分配一段连续的空间
- 查询某个元素是否存在时需要遍历整个数组，耗费 O(n) 的时间（其中，n 是元素的个数）
- 删除和添加某个元素时，同样需要耗费 O(n) 的时间

**基本操作**

- **insert**：在某个索引处插入元素
- **get**：读取某个索引处的元素
- **delete**：删除某个索引处的元素
- **size**：获取数组的长度

### 分析数组的局限性

　　通过上面的代码，我们发现数组是能完成一个数据结构所有的功能的，而且实现起来也不难，那数据既然能完成所有的工作，我们实际应用中为啥不用它来进行所有的数据存储呢？那肯定是有原因呢。

　　数组的局限性分析：

　　①、插入快，对于无序数组，上面我们实现的数组就是无序的，即元素没有按照从大到小或者某个特定的顺序排列，只是按照插入的顺序排列。无序数组增加一个元素很简单，只需要在数组末尾添加元素即可，但是有序数组却不一定了，它需要在指定的位置插入。

　　②、查找慢，当然如果根据下标来查找是很快的。但是通常我们都是根据元素值来查找，给定一个元素值，对于无序数组，我们需要从数组第一个元素开始遍历，直到找到那个元素。有序数组通过特定的算法查找的速度会比无需数组快，后面我们会讲各种排序算法。

　　③、删除慢，根据元素值删除，我们要先找到该元素所处的位置，然后将元素后面的值整体向前面移动一个位置。也需要比较多的时间。

　　④、数组一旦创建后，大小就固定了，不能动态扩展数组的元素个数。如果初始化你给一个很大的数组大小，那会白白浪费内存空间，如果给小了，后面数据个数增加了又添加不进去了。

　　很显然，数组虽然插入快，但是查找和删除都比较慢，而且扩展性差，所以我们一般不会用数组来存储数据，那有没有什么数据结构插入、查找、删除都很快，而且还能动态扩展存储个数大小呢，答案是有的，但是这是建立在很复杂的算法基础上，后面我们也会详细讲解。

**案例一：翻转字符串“algorithm”**

![翻转字符串algorithm](png/Java/翻转字符串algorithm.gif)

**解法**：用两个指针，一个指向字符串的第一个字符 a，一个指向它的最后一个字符 m，然后互相交换。交换之后，两个指针向中央一步步地靠拢并相互交换字符，直到两个指针相遇。这是一种比较快速和直观的方法。

**案例二：给定两个字符串s和t，编写一个函数来判断t是否是s的字母异位词。**

说明：你可以假设字符串只包含小写字母。

**解题思路**：字母异位词，也就是两个字符串中的相同字符的数量要对应相等。

- 可以利用两个长度都为 26 的字符数组来统计每个字符串中小写字母出现的次数，然后再对比是否相等
- 可以只利用一个长度为 26 的字符数组，将出现在字符串 s 里的字符个数加 1，而出现在字符串 t 里的字符个数减 1，最后判断每个小写字母的个数是否都为 0





# 排序算法

我们实现的数组结构是无序的，也就是纯粹按照插入顺序进行排列，那么如何进行元素排序，本篇博客我们介绍几种简单的排序算法。

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

### 选择排序

选择排序是每一次从待排序的数据元素中选出最小的一个元素，存放在序列的起始位置，直到全部待排序的数据元素排完。

　　分为三步：

　　①、从待排序序列中，找到关键字最小的元素

　　②、如果最小元素不是待排序序列的第一个元素，将其和第一个元素互换

　　③、从余下的 N - 1 个元素中，找出关键字最小的元素，重复(1)、(2)步，直到排序结束

![Java选择排序](png\Java\Java选择排序.png)

代码如下：

```
package com.demo.sort;
 
public class ChoiceSort {
    public static int[] sort(int[] array){
        //总共要经过N-1轮比较
        for(int i = 0 ; i < array.length-1 ; i++){
            int min = i;
            //每轮需要比较的次数
            for(int j = i+1 ; j < array.length ; j++){
                if(array[j]<array[min]){
                    min = j;//记录目前能找到的最小值元素的下标
                }
            }
            //将找到的最小值和i位置所在的值进行交换
            if(i != min){
                int temp = array[i];
                array[i] = array[min];
                array[min] = temp;
            }
            //第 i轮排序的结果为
            System.out.print("第"+(i+1)+"轮排序后的结果为:");
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
     
    public static void main(String[] args){
        int[] array = {4,2,8,9,5,7,6,1,3};
        //未排序数组顺序为
        System.out.println("未排序数组顺序为：");
        display(array);
        System.out.println("-----------------------");
        array = sort(array);
        System.out.println("-----------------------");
        System.out.println("经过选择排序后的数组顺序为：");
        display(array);
    }
}
```

**选择排序性能分析：**

　　选择排序和冒泡排序执行了相同次数的比较：N*（N-1）/2，但是至多只进行了N次交换。

　　当 N 值很大时，比较次数是主要的，所以和冒泡排序一样，用大O表示是O(N2) 时间级别。但是由于选择排序交换的次数少，所以选择排序无疑是比冒泡排序快的。当 N 值较小时，如果交换时间比选择时间大的多，那么选择排序是相当快的。

### 插入排序

直接插入排序基本思想是每一步将一个待排序的记录，插入到前面已经排好序的有序序列中去，直到插完所有元素为止。

　　插入排序还分为直接插入排序、二分插入排序、链表插入排序、希尔排序等等，这里我们只是以直接插入排序讲解，后面讲高级排序的时候会将其他的。

![Java插入排序](png\Java\Java插入排序.png)

**代码如下：**

```
package com.demo.sort;
 
public class InsertSort {
    public static int[] sort(int[] array){
        int j;
        //从下标为1的元素开始选择合适的位置插入，因为下标为0的只有一个元素，默认是有序的
        for(int i = 1 ; i < array.length ; i++){
            int tmp = array[i];//记录要插入的数据
            j = i;
            while(j > 0 && tmp < array[j-1]){//从已经排序的序列最右边的开始比较，找到比其小的数
                array[j] = array[j-1];//向后挪动
                j--;
            }
            array[j] = tmp;//存在比其小的数，插入
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
     
    public static void main(String[] args){
        int[] array = {4,2,8,9,5,7,6,1,3};
        //未排序数组顺序为
        System.out.println("未排序数组顺序为：");
        display(array);
        System.out.println("-----------------------");
        array = sort(array);
        System.out.println("-----------------------");
        System.out.println("经过插入排序后的数组顺序为：");
        display(array);
    }
 
}
```

**插入排序性能分析：**

　　在第一轮排序中，它最多比较一次，第二轮最多比较两次，一次类推，第N轮，最多比较N-1次。因此有 1+2+3+...+N-1 = N*（N-1）/2。

　　假设在每一轮排序发现插入点时，平均只有全体数据项的一半真的进行了比较，我们除以2得到：N*（N-1）/4。用大O表示法大致需要需要 O(N2) 时间级别。

　　复制的次数大致等于比较的次数，但是一次复制与一次交换的时间耗时不同，所以相对于随机数据，插入排序比冒泡快一倍，比选择排序略快。

　　这里需要注意的是，如果要进行逆序排列，那么每次比较和移动都会进行，这时候并不会比冒泡排序快。

上面讲的三种排序，冒泡、选择、插入用大 O 表示法都需要 O(N2) 时间级别。一般不会选择冒泡排序，虽然冒泡排序书写是最简单的，但是平均性能是没有选择排序和插入排序好的。

　　选择排序把交换次数降低到最低，但是比较次数还是挺大的。当数据量小，并且交换数据相对于比较数据更加耗时的情况下，可以应用选择排序。

　　在大多数情况下，假设数据量比较小或基本有序时，插入排序是三种算法中最好的选择。

## **栈**(Stack)

数组更多的是用来进行数据的存储，纯粹用来存储数据的数据结构，我们期望的是插入、删除和查找性能都比较好。对于无序数组，插入快，但是删除和查找都很慢，为了解决这些问题，后面我们会讲解比如二叉树、哈希表的数据结构。

　 现在讲解的数据结构和算法更多是用作程序员的工具，它们作为构思算法的辅助工具，而不是完全的数据存储工具。这些数据结构的生命周期比数据库类型的结构要短得多，在程序执行期间它们才被创建，通常用它们去执行某项特殊的业务，执行完成之后，它们就被销毁。这里的它们就是——栈和队列。

栈是一种**先进后出**（`FILO`，First in last out）或**后进先出**（`LIFO`，Last in first out）的数据结构。

![数据结构-stack](png\Java\数据结构-stack.png)

### 栈的基本概念

**栈**（英语：stack）又称为**堆栈**或**堆叠**，栈作为一种数据结构，是一种只能在一端进行插入和删除操作的特殊线性表。它按照先进后出的原则存储数据，先进入的数据被压入栈底，最后的数据在栈顶，需要读数据的时候从栈顶开始弹出数据（最后一个数据被第一个读出来）。栈具有记忆作用，对栈的插入与删除操作中，不需要改变栈底指针。

　　栈是允许在同一端进行插入和删除操作的特殊线性表。允许进行插入和删除操作的一端称为栈顶(top)，另一端为栈底(bottom)；栈底固定，而栈顶浮动；栈中元素个数为零时称为空栈。插入一般称为进栈（PUSH），删除则称为退栈（POP）。

　　由于堆叠数据结构只允许在一端进行操作，因而按照后进先出（LIFO, Last In First Out）的原理运作。栈也称为后进先出表。

![数据结构-Stack操作](png\Java\数据结构-Stack操作.png)

　　这里以羽毛球筒为例，羽毛球筒就是一个栈，刚开始羽毛球筒是空的，也就是空栈，然后我们一个一个放入羽毛球，也就是一个一个push进栈，当我们需要使用羽毛球的时候，从筒里面拿，也就是pop出栈，但是第一个拿到的羽毛球是我们最后放进去的。



### Java模拟简单的顺序栈实现

```
package com.demo.datastructure;
 
public class MyStack {
    private int[] array;
    private int maxSize;
    private int top;
     
    public MyStack(int size){
        this.maxSize = size;
        array = new int[size];
        top = -1;
    }
     
    //压入数据
    public void push(int value){
        if(top < maxSize-1){
            array[++top] = value;
        }
    }
     
    //弹出栈顶数据
    public int pop(){
        return array[top--];
    }
     
    //访问栈顶数据
    public int peek(){
        return array[top];
    }
     
    //判断栈是否为空
    public boolean isEmpty(){
        return (top == -1);
    }
     
    //判断栈是否满了
    public boolean isFull(){
        return (top == maxSize-1);
    }
     
 
}
```

**测试：**

```
package com.demo.test;
 
import com.demo.datastructure.MyStack;
 
public class MyStackTest {
    public static void main(String[] args) {
        MyStack stack = new MyStack(3);
        stack.push(1);
        stack.push(2);
        stack.push(3);
        System.out.println(stack.peek());
        while(!stack.isEmpty()){
            System.out.println(stack.pop());
        }
         
    }
 
}
```

这个栈是用数组实现的，内部定义了一个数组，一个表示最大容量的值以及一个指向栈顶元素的top变量。构造方法根据参数规定的容量创建一个新栈，push()方法是向栈中压入元素，指向栈顶的变量top加一，使它指向原顶端数据项上面的一个位置，并在这个位置上存储一个数据。pop()方法返回top变量指向的元素，然后将top变量减一，便移除了数据项。要知道 top 变量指向的始终是栈顶的元素。

　　**产生的问题：**

　　**①、上面栈的实现初始化容量之后，后面是不能进行扩容的（虽然栈不是用来存储大量数据的），如果说后期数据量超过初始容量之后怎么办？（\**自动扩容\**）**

　　**②、我们是用数组实现栈，在定义数组类型的时候，也就规定了存储在栈中的数据类型，那么同一个栈能不能存储不同类型的数据呢？（声明为Object）**

　　**③、栈需要初始化容量，而且数组实现的栈元素都是连续存储的，那么能不能不初始化容量呢？（改为由链表实现）**

### **增强功能版栈**

**对于上面出现的问题，第一个能自动扩容，第二个能存储各种不同类型的数据，解决办法如下：（第三个在讲链表的时候在介绍）**

　　**这个模拟的栈在JDK源码中，大家可以参考 Stack 类的实现。**

```
package com.demo.datastructure;
 
import java.util.Arrays;
import java.util.EmptyStackException;
 
public class ArrayStack {
    //存储元素的数组,声明为Object类型能存储任意类型的数据
    private Object[] elementData;
    //指向栈顶的指针
    private int top;
    //栈的总容量
    private int size;
     
     
    //默认构造一个容量为10的栈
    public ArrayStack(){
        this.elementData = new Object[10];
        this.top = -1;
        this.size = 10;
    }
     
    public ArrayStack(int initialCapacity){
        if(initialCapacity < 0){
            throw new IllegalArgumentException("栈初始容量不能小于0: "+initialCapacity);
        }
        this.elementData = new Object[initialCapacity];
        this.top = -1;
        this.size = initialCapacity;
    }
     
     
    //压入元素
    public Object push(Object item){
        //是否需要扩容
        isGrow(top+1);
        elementData[++top] = item;
        return item;
    }
     
    //弹出栈顶元素
    public Object pop(){
        Object obj = peek();
        remove(top);
        return obj;
    }
     
    //获取栈顶元素
    public Object peek(){
        if(top == -1){
            throw new EmptyStackException();
        }
        return elementData[top];
    }
    //判断栈是否为空
    public boolean isEmpty(){
        return (top == -1);
    }
     
    //删除栈顶元素
    public void remove(int top){
        //栈顶元素置为null
        elementData[top] = null;
        this.top--;
    }
     
    /**
     * 是否需要扩容，如果需要，则扩大一倍并返回true，不需要则返回false
     * @param minCapacity
     * @return
     */
    public boolean isGrow(int minCapacity){
        int oldCapacity = size;
        //如果当前元素压入栈之后总容量大于前面定义的容量，则需要扩容
        if(minCapacity >= oldCapacity){
            //定义扩大之后栈的总容量
            int newCapacity = 0;
            //栈容量扩大两倍(左移一位)看是否超过int类型所表示的最大范围
            if((oldCapacity<<1) - Integer.MAX_VALUE >0){
                newCapacity = Integer.MAX_VALUE;
            }else{
                newCapacity = (oldCapacity<<1);//左移一位，相当于*2
            }
            this.size = newCapacity;
            int[] newArray = new int[size];
            elementData = Arrays.copyOf(elementData, size);
            return true;
        }else{
            return false;
        }
    }
     
     
 
}
```

测试：

```
//测试自定义栈类 ArrayStack
//创建容量为3的栈，然后添加4个元素，3个int，1个String.
@Test
public void testArrayStack(){
    ArrayStack stack = new ArrayStack(3);
    stack.push(1);
    //System.out.println(stack.peek());
    stack.push(2);
    stack.push(3);
    stack.push("abc");
    System.out.println(stack.peek());
    stack.pop();
    stack.pop();
    stack.pop();
    System.out.println(stack.peek());
}
```

- **单向链表**：可以利用一个单链表来实现栈的数据结构。而且，因为我们都只针对栈顶元素进行操作，所以借用单链表的头就能让所有栈的操作在 O(1) 的时间内完成。
- **Stack**：是Vector的子类，比Vector多了几个方法

```java
public class Stack<E> extends Vector<E> {
        // 把元素压入栈顶
        public E push(E item) {
            addElement(item);
            return item;
        }
    
        // 弹出栈顶元素
        public synchronized E pop() {
            E obj;
            int len = size();
            obj = peek();
            removeElementAt(len - 1);
            return obj;
        }
    
        // 访问当前栈顶元素，但是不拿走栈顶元素
        public synchronized E peek() {
            int len = size();
            if (len == 0)
                throw new EmptyStackException();
            return elementAt(len - 1);
        }
    
	// 测试堆栈是否为空
        public boolean empty() {
            return size() == 0;
        }
        
        // 返回对象在堆栈中的位置，以1为基数
        public synchronized int search(Object o) {
            int i = lastIndexOf(o);
            if (i >= 0) {
                return size() - i;
            }
            return -1;
        }
}
```

**基本操作**（失败时：add/remove/element为抛异常，offer/poll/peek为返回false或null）

- `E push(E)`：把元素压入栈
- `E pop()`：把栈顶的元素弹出
- `E peek()`：取栈顶元素但不弹出
- `boolean empty()`：堆栈是否为空测试
- `int search(o)`：返回对象在堆栈中的位置，以 1 为基数

### 利用栈实现字符串逆序

我们知道栈是后进先出，我们可以将一个字符串分隔为单个的字符，然后将字符一个一个push()进栈，在一个一个pop()出栈就是逆序显示了。如下：

　　将 字符串“how are you” 反转！！！

　　ps：这里我们是用上面自定的栈来实现的，大家可以将ArrayStack替换为JDK自带的栈类Stack试试

```
//进行字符串反转
@Test
public void testStringReversal(){
    ArrayStack stack = new ArrayStack();
    String str = "how are you";
    char[] cha = str.toCharArray();
    for(char c : cha){
        stack.push(c);
    }
    while(!stack.isEmpty()){
        System.out.print(stack.pop());
    }
}
```

### 利用栈判断分隔符是否匹配

写过xml标签或者html标签的，我们都知道<必须和最近的>进行匹配，[ 也必须和最近的 ] 进行匹配。

　　比如：<abc[123]abc>这是符号相匹配的，如果是 <abc[123>abc] 那就是不匹配的。

　　对于 12<a[b{c}]>，我们分析在栈中的数据：遇到匹配正确的就消除

最后栈中的内容为空则匹配成功，否则匹配失败！！！

```
//分隔符匹配
//遇到左边分隔符了就push进栈，遇到右边分隔符了就pop出栈，看出栈的分隔符是否和这个有分隔符匹配
@Test
public void testMatch(){
    ArrayStack stack = new ArrayStack(3);
    String str = "12<a[b{c}]>";
    char[] cha = str.toCharArray();
    for(char c : cha){
        switch (c) {
        case '{':
        case '[':
        case '<':
            stack.push(c);
            break;
        case '}':
        case ']':
        case '>':
            if(!stack.isEmpty()){
                char ch = stack.pop().toString().toCharArray()[0];
                if(c=='}' && ch != '{'
                    || c==']' && ch != '['
                    || c==')' && ch != '('){
                    System.out.println("Error:"+ch+"-"+c);
                }
            }
            break;
        default:
            break;
        }
    }
}
```

根据栈后进先出的特性，我们实现了单词逆序以及分隔符匹配。所以其实栈是一个概念上的工具，具体能实现什么功能可以由我们去想象。栈通过提供限制性的访问方法push()和pop()，使得程序不容易出错。

在解决某个问题的时候，只要求关心最近一次的操作，并且在操作完成了之后，需要向前查找到更前一次的操作。

**应用场景**

**案例一：判断字符串是否有效**

给定一个只包括 '('，')'，'{'，'}'，'['，']' 的字符串，判断字符串是否有效。有效字符串需满足：

- 左括号必须用相同类型的右括号闭合
- 左括号必须以正确的顺序闭合
- 空字符串可被认为是有效字符串

**解题思路**：利用一个栈，不断地往里压左括号，一旦遇上了一个右括号，我们就把栈顶的左括号弹出来，表示这是一个合法的组合，以此类推，直到最后判断栈里还有没有左括号剩余。

**案例二：每日温度**

根据每日气温列表，请重新生成一个列表，对应位置的输入是你需要再等待多久温度才会升高超过该日的天数。如果之后都不会升高，请在该位置用 0 来代替。

**解题思路**

- 思路 1：最直观的做法就是针对每个温度值向后进行依次搜索，找到比当前温度更高的值，这样的计算复杂度就是 O(n2)。

- 思路 2：可以运用一个堆栈 stack 来快速地知道需要经过多少天就能等到温度升高。从头到尾扫描一遍给定的数组 T，如果当天的温度比堆栈 stack 顶端所记录的那天温度还要高，那么就能得到结果。

　　对于栈的实现，我们稍微分析就知道，数据入栈和出栈的时间复杂度都为O(1)，也就是说栈操作所耗的时间不依赖栈中数据项的个数，因此操作时间很短。而且需要注意的是栈不需要比较和移动操作，我们不要画蛇添足。

## 队列(Queue)

队列。栈是后进先出，而队列刚好相反，是先进先出。

### 队列的基本概念

![数据结构-queue](png\Java\数据结构-queue.png)

队列(Queue)和栈不同，队列的最大特点是**先进先出**（`FIFO`），就好像按顺序排队一样。对于队列的数据来说，我们只允许在队尾查看和添加数据，在队头查看和删除数据。

- **栈**：采用**后进先出**（`LIFO`）
- **队列**：采用 **先进先出**（First in First Out，即`FIFO`）

![数据结构-queue操作](png\Java\数据结构-queue操作.png)

队列（queue）是一种特殊的线性表，特殊之处在于它只允许在表的前端（front）进行删除操作，而在表的后端（rear）进行插入操作，和栈一样，队列是一种操作受限制的线性表。进行插入操作的端称为队尾，进行删除操作的端称为队头。队列中没有元素时，称为空队列。

　　队列的数据元素又称为队列元素。在队列中插入一个队列元素称为入队，从队列中删除一个队列元素称为出队。因为队列只允许在一端插入，在另一端删除，所以只有最早进入队列的元素才能最先从队列中删除，故队列又称为先进先出（FIFO—first in first out）线性表。

　　比如我们去电影院排队买票，第一个进入排队序列的都是第一个买到票离开队列的人，而最后进入排队序列排队的都是最后买到票的。

　　在比如在计算机操作系统中，有各种队列在安静的工作着，比如打印机在打印列队中等待打印。

　　队列分为：

　　①、单向队列（Queue）：只能在一端插入数据，另一端删除数据。

　　②、双向队列（Deque）：每一端都可以进行插入数据和删除数据操作。

　　这里我们还会介绍一种队列——优先级队列，优先级队列是比栈和队列更专用的数据结构，在优先级队列中，数据项按照关键字进行排序，关键字最小（或者最大）的数据项往往在队列的最前面，而数据项在插入的时候都会插入到合适的位置以确保队列的有序。

### Java模拟单向队列实现

在实现之前，我们先看下面几个问题：

　　①、与栈不同的是，队列中的数据不总是从数组的0下标开始的，移除一些队头front的数据后，队头指针会指向一个较高的下标位置，

​		②、我们再设计时，队列中新增一个数据时，队尾的指针rear 会向上移动，也就是向下标大的方向。移除数据项时，队头指针 front 向上移动。那么这样设计好像和现实情况相反，比如排队买电影票，队头的买完票就离开了，然后队伍整体向前移动。在计算机中也可以在队列中删除一个数之后，队列整体向前移动，但是这样做效率很差。我们选择的做法是移动队头和队尾的指针。

　　③、如果向第②步这样移动指针，相信队尾指针很快就移动到数据的最末端了，这时候可能移除过数据，那么队头会有空着的位置，然后新来了一个数据项，由于队尾不能再向上移动了，那该怎么办呢？

为了避免队列不满却不能插入新的数据，我们可以让队尾指针绕回到数组开始的位置，这也称为“循环队列”。

弄懂原理之后，Java实现代码如下：

```
package com.demo.datastructure;
 
public class MyQueue {
    private Object[] queArray;
    //队列总大小
    private int maxSize;
    //前端
    private int front;
    //后端
    private int rear;
    //队列中元素的实际数目
    private int nItems;
     
    public MyQueue(int s){
        maxSize = s;
        queArray = new Object[maxSize];
        front = 0;
        rear = -1;
        nItems = 0;
    }
     
    //队列中新增数据
    public void insert(int value){
        if(isFull()){
            System.out.println("队列已满！！！");
        }else{
            //如果队列尾部指向顶了，那么循环回来，执行队列的第一个元素
            if(rear == maxSize -1){
                rear = -1;
            }
            //队尾指针加1，然后在队尾指针处插入新的数据
            queArray[++rear] = value;
            nItems++;
        }
    }
     
    //移除数据
    public Object remove(){
        Object removeValue = null ;
        if(!isEmpty()){
            removeValue = queArray[front];
            queArray[front] = null;
            front++;
            if(front == maxSize){
                front = 0;
            }
            nItems--;
            return removeValue;
        }
        return removeValue;
    }
     
    //查看对头数据
    public Object peekFront(){
        return queArray[front];
    }
     
     
    //判断队列是否满了
    public boolean isFull(){
        return (nItems == maxSize);
    }
     
    //判断队列是否为空
    public boolean isEmpty(){
        return (nItems ==0);
    }
     
    //返回队列的大小
    public int getSize(){
        return nItems;
    }
     
}
```

测试：

```
package com.demo.test;
 
import com.demo.datastructure.MyQueue;
 
public class MyQueueTest {
    public static void main(String[] args) {
        MyQueue queue = new MyQueue(3);
        queue.insert(1);
        queue.insert(2);
        queue.insert(3);//queArray数组数据为[1,2,3]
         
        System.out.println(queue.peekFront()); //1
        queue.remove();//queArray数组数据为[null,2,3]
        System.out.println(queue.peekFront()); //2
         
        queue.insert(4);//queArray数组数据为[4,2,3]
        queue.insert(5);//队列已满,queArray数组数据为[4,2,3]
    }
 
}
```

**实现方式**

可借助**双链表**来实现队列。双链表的头指针允许在队头查看和删除数据，而双链表的尾指针允许在队尾查看和添加数据。

**基本操作**（失败时：add/remove/element为抛异常，offer/poll/peek为返回false或null）

- `int size()`：获取队列长度
- `boolean add(E)`/`boolean offer(E)`：添加元素到队尾
- `E remove()`/`E poll()`：获取队首元素并从队列中删除
- `E element()`/`E peek()`：获取队首元素但并不从队列中删除

**应用场景**

当需要按照一定的顺序来处理数据，而该数据的数据量在不断地变化的时候，则需要队列来处理。

### 双端队列

​		双端队列就是一个两端都是结尾或者开头的队列， 队列的每一端都可以进行插入数据项和移除数据项，这些方法可以叫做：

　　insertRight()、insertLeft()、removeLeft()、removeRight()

　　如果严格禁止调用insertLeft()和removeLeft()（或禁用右端操作），那么双端队列的功能就和前面讲的栈功能一样。

　　如果严格禁止调用insertLeft()和removeRight(或相反的另一对方法)，那么双端队列的功能就和单向队列一样了。

双端队列和普通队列最大的不同在于，它允许我们在队列的头尾两端都能在 O(1) 的时间内进行数据的查看、添加和删除。

**实现方式**

双端队列（Deque）与队列相似，可以利用一个**双链表实现**双端队列。

**应用场景**

双端队列最常用的地方就是实现一个长度动态变化的窗口或者连续区间，而动态窗口这种数据结构在很多题目里都有运用。



**案例一：滑动窗口最大值**

给定一个数组 nums，有一个大小为 k 的滑动窗口从数组的最左侧移动到数组的最右侧。你只可以看到在滑动窗口 k 内的数字，滑动窗口每次只向右移动一位，就返回当前滑动窗口中的最大值。

**解题思路**

- 思路 1：移动窗口，扫描，获得最大值。假设数组里有 n 个元素，算法复杂度就是 O(n)。这是最直观的做法。

- 思路 2：利用一个双端队列来保存当前窗口中最大那个数在数组里的下标，双端队列新的头就是当前窗口中最大的那个数。通过该下标，可以很快知道新窗口是否仍包含原来那个最大的数。如果不再包含，就把旧的数从双端队列的头删除。

  因为双端队列能让上面的这两种操作都能在 O(1) 的时间里完成，所以整个算法的复杂度能控制在 O(n)。

### 优先级队列

优先级队列（priority queue）是比栈和队列更专用的数据结构，在优先级队列中，数据项按照关键字进行排序，关键字最小（或者最大）的数据项往往在队列的最前面，而数据项在插入的时候都会插入到合适的位置以确保队列的有序。

　　优先级队列 是0个或多个元素的集合，每个元素都有一个优先权，对优先级队列执行的操作有：

　　（1）查找

　　（2）插入一个新元素

　　（3）删除

　　一般情况下，查找操作用来搜索优先权最大的元素，删除操作用来删除该元素 。对于优先权相同的元素，可按先进先出次序处理或按任意优先权进行。

　　这里我们用数组实现优先级队列，这种方法插入比较慢，但是它比较简单，适用于数据量比较小并且不是特别注重插入速度的情况。

　　后面我们会讲解堆，用堆的数据结构来实现优先级队列，可以相当快的插入数据。

　　**数组实现优先级队列，声明为int类型的数组，关键字是数组里面的元素，在插入的时候按照从大到小的顺序排列，也就是越小的元素优先级越高。**

**实现方式**
优先队列的本质是一个二叉堆结构。堆在英文里叫 Binary Heap，它是利用一个数组结构来实现的完全二叉树。换句话说，优先队列的本质是一个数组，数组里的每个元素既有可能是其他元素的父节点，也有可能是其他元素的子节点，而且，每个父节点只能有两个子节点，很像一棵二叉树的结构。

```
package com.demo.datastructure;
 
public class PriorityQue {
    private int maxSize;
    private int[] priQueArray;
    private int nItems;
     
    public PriorityQue(int s){
        maxSize = s;
        priQueArray = new int[maxSize];
        nItems = 0;
    }
     
    //插入数据
    public void insert(int value){
        int j;
        if(nItems == 0){
            priQueArray[nItems++] = value;
        }else{
            j = nItems -1;
            //选择的排序方法是插入排序，按照从大到小的顺序排列，越小的越在队列的顶端
            while(j >=0 && value > priQueArray[j]){
                priQueArray[j+1] = priQueArray[j];
                j--;
            }
            priQueArray[j+1] = value;
            nItems++;
        }
    }
     
    //移除数据,由于是按照大小排序的，所以移除数据我们指针向下移动
    //被移除的地方由于是int类型的，不能设置为null，这里的做法是设置为 -1
    public int remove(){
        int k = nItems -1;
        int value = priQueArray[k];
        priQueArray[k] = -1;//-1表示这个位置的数据被移除了
        nItems--;
        return value;
    }
     
    //查看优先级最高的元素
    public int peekMin(){
        return priQueArray[nItems-1];
    }
     
    //判断是否为空
    public boolean isEmpty(){
        return (nItems == 0);
    }
     
    //判断是否满了
    public boolean isFull(){
        return (nItems == maxSize);
    }
 
}
```

insert() 方法，先检查队列中是否有数据项，如果没有，则直接插入到下标为0的单元里，否则，从数组顶部开始比较，找到比插入值小的位置进行插入，并把 nItems 加1.

　　remove 方法直接获取顶部元素。

　　优先级队列的插入操作需要 O(N)的时间，而删除操作则需要O(1) 的时间，后面会讲解如何通过 堆 来改进插入时间。

**优先队列的性质**

- 数组里的第一个元素 array[0] 拥有最高的优先级别
- 给定一个下标 i，那么对于元素 array[i] 而言：
  - 它的父节点所对应的元素下标是 (i-1)/2
  - 它的左孩子所对应的元素下标是 2×i + 1
  - 它的右孩子所对应的元素下标是 2×i + 2
- 数组里每个元素的优先级别都要高于它两个孩子的优先级别

**应用场景**：从一堆杂乱无章的数据当中按照一定的顺序（或者优先级）逐步地筛选出部分乃至全部的数据。

**基本操作**

**① 向上筛选（sift up/bubble up）**

- 当有新的数据加入到优先队列中，新的数据首先被放置在二叉堆的底部。
- 不断进行向上筛选的操作，即如果发现该数据的优先级别比父节点的优先级别还要高，那么就和父节点的元素相互交换，再接着往上进行比较，直到无法再继续交换为止。

**时间复杂度**：由于二叉堆是一棵完全二叉树，并假设堆的大小为 k，因此整个过程其实就是沿着树的高度往上爬，所以只需要 O(logk) 的时间。

**② 向下筛选（sift down/bubble down）**

- 当堆顶的元素被取出时，要更新堆顶的元素来作为下一次按照优先级顺序被取出的对象，需要将堆底部的元素放置到堆顶，然后不断地对它执行向下筛选的操作。
- 将该元素和它的两个孩子节点对比优先级，如果优先级最高的是其中一个孩子，就将该元素和那个孩子进行交换，然后反复进行下去，直到无法继续交换为止。

**时间复杂度**：整个过程就是沿着树的高度往下爬，所以时间复杂度也是 O(logk)。

**案例一：[前 K 个高频元素](https://leetcode-cn.com/problems/top-k-frequent-elements/)**

给你一个整数数组 `nums` 和一个整数 `k` ，请你返回其中出现频率前 `k` 高的元素。你可以按 **任意顺序** 返回答案。

**最小堆解法**：题目最终需要返回的是前 k 个频率最大的元素，可以想到借助堆这种数据结构，对于 k 频率之后的元素不用再去处理，进一步优化时间复杂度。

我们介绍了队列的三种形式，分别是单向队列、双向队列以及优先级队列。其实大家听名字也可以听得出来他们之间的区别，单向队列遵循先进先出的原则，而且一端只能插入，另一端只能删除。双向队列则两端都可插入和删除，如果限制双向队列的某一段的方法，则可以达到和单向队列同样的功能。最后优先级队列，则是在插入元素的时候进行了优先级别排序，在实际应用中单项队列和优先级队列使用的比较多。后面讲解了堆这种数据结构，我们会用堆来实现优先级队列，改善优先级队列插入元素的时间。

　　通过前面讲的栈以及本篇讲的队列这两种数据结构，我们稍微总结一下：

　　①、栈、队列（单向队列）、优先级队列通常是用来简化某些程序操作的数据结构，而不是主要作为存储数据的。

　　②、在这些数据结构中，只有一个数据项可以被访问。

　　③、栈允许在栈顶压入（插入）数据，在栈顶弹出（移除）数据，但是只能访问最后一个插入的数据项，也就是栈顶元素。

　　④、队列（单向队列）只能在队尾插入数据，对头删除数据，并且只能访问对头的数据。而且队列还可以实现循环队列，它基于数组，数组下标可以从数组末端绕回到数组的开始位置。

　　⑤、优先级队列是有序的插入数据，并且只能访问当前元素中优先级别最大（或最小）的元素。

　　⑥、这些数据结构都能由数组实现，但是可以用别的机制（后面讲的链表、堆等数据结构）实现。

## 链表(Linked List)

​		数组作为数据存储结构有一定的缺陷。在无序数组中，搜索性能差，在有序数组中，插入效率又很低，而且这两种数组的删除效率都很低，并且数组在创建后，其大小是固定了，设置的过大会造成内存的浪费，过小又不能满足数据量的存储。

　　我们将讲解一种新型的数据结构——链表。我们知道数组是一种通用的数据结构，能用来实现栈、队列等很多数据结构。而链表也是一种使用广泛的通用数据结构，它也可以用来作为实现栈、队列等数据结构的基础，基本上除非需要频繁的通过下标来随机访问各个数据，否则很多使用数组的地方都可以用链表来代替。

　　但是我们需要说明的是，链表是不能解决数据存储的所有问题的，它也有它的优点和缺点。本篇博客我们介绍几种常见的链表，分别是单向链表、双端链表、有序链表、双向链表以及有迭代器的链表。并且会讲解一下抽象数据类型（ADT）的思想，如何用 ADT 描述栈和队列，如何用链表代替数组来实现栈和队列。

### 链表（Linked List）

![数据结构-LinkedList](png\Java\数据结构-LinkedList.png)

　　链表通常由一连串节点组成，每个节点包含任意的实例数据（data fields）和一或两个用来指向上一个/或下一个节点的位置的链接（"links"）

​		**链表**（Linked list）是一种常见的基础数据结构，是一种线性表，但是并不会按线性的顺序存储数据，而是在每一个节点里存到下一个节点的指针(Pointer)。

　　使用链表结构可以克服数组链表需要预先知道数据大小的缺点，链表结构可以充分利用计算机内存空间，实现灵活的内存动态管理。但是链表失去了数组随机读取的优点，同时链表由于增加了结点的指针域，空间开销比较大。

**优点**

- 链表能灵活地分配内存空间
- 能在 O(1) 时间内删除或者添加元素

**缺点**

- 不像数组能通过下标迅速读取元素，每次都要从链表头开始一个一个读取
- 查询第 k 个元素需要 O(k) 时间

**基本操作**

- **insertAtEnd**：在链表结尾插入元素
- **insertAtHead**：在链表开头插入元素
- **delete** ：删除链表的指定元素
- **deleteAtHead** ：删除链表第一个元素
- **search**：在链表中查询指定元素
- **isEmpty**：查询链表是否为空

**应用场景**

- 如果要解决的问题里面**需要很多快速查询**，链表可能并不适合
- 如果遇到的问题中，数据的元素个数不确定，而且需要**经常进行数据的添加和删除**，那么链表会比较合适
- 如果数据元素大小确定，**删除和插入的操作并不多**，那么数组可能更适合

### 单向链表（Single-Linked List）

![数据结构-单向链表](png\Java\数据结构-单向链表.png)

单向链表包含两个域：

- **一个数据域**：用于存储数据
- **一个指针域**：用于指向下一个节点（最后一个节点则指向一个空值）：

单链表的遍历方向单一，只能从链头一直遍历到链尾。它的缺点是当要查询某一个节点的前一个节点时，只能再次从头进行遍历查询，因此效率比较低，而双向链表的出现恰好解决了这个问题。单向链表代码如下：

```
public class Node<E> {
   private Eitem;
   private Node<E> next;
}
```

单链表是链表中结构最简单的。一个单链表的节点(Node)分为两个部分，第一个部分(data)保存或者显示关于节点的信息，另一个部分存储下一个节点的地址。最后一个节点存储地址的部分指向空值。

　　单向链表只可向一个方向遍历，一般查找一个节点的时候需要从第一个节点开始每次访问下一个节点，一直访问到需要的位置。而插入一个节点，对于单向链表，我们只提供在链表头插入，只需要将当前插入的节点设置为头节点，next指向原头节点即可。删除一个节点，我们将该节点的上一个节点的next指向该节点的下一个节点。

单向链表的具体实现

```
package com.demo.datastructure;

public class SingleLinkedList {
    private int size;//链表节点的个数
    private Node head;//头节点

    public SingleLinkedList(){
        size = 0;
        head = null;
    }

    //链表的每个节点类
    private class Node{
        private Object data;//每个节点的数据
        private Node next;//每个节点指向下一个节点的连接

        public Node(Object data){
            this.data = data;
        }
    }

    //在链表头添加元素
    public Object addHead(Object obj){
        Node newHead = new Node(obj);
        if(size == 0){
            head = newHead;
        }else{
            newHead.next = head;
            head = newHead;
        }
        size++;
        return obj;
    }

    //在链表头删除元素
    public Object deleteHead(){
        Object obj = head.data;
        head = head.next;
        size--;
        return obj;
    }

    //查找指定元素，找到了返回节点Node，找不到返回null
    public Node find(Object obj){
        Node current = head;
        int tempSize = size;
        while(tempSize > 0){
            if(obj.equals(current.data)){
                return current;
            }else{
                current = current.next;
            }
            tempSize--;
        }
        return null;
    }

    //删除指定的元素，删除成功返回true
    public boolean delete(Object value){
        if(size == 0){
            return false;
        }
        Node current = head;
        Node previous = head;
        while(current.data != value){
            if(current.next == null){
                return false;
            }else{
                previous = current;
                current = current.next;
            }
        }
        //如果删除的节点是第一个节点
        if(current == head){
            head = current.next;
            size--;
        }else{//删除的节点不是第一个节点
            previous.next = current.next;
            size--;
        }
        return true;
    }

    //判断链表是否为空
    public boolean isEmpty(){
        return (size == 0);
    }

    //显示节点信息
    public void display(){
        if(size >0){
            Node node = head;
            int tempSize = size;
            if(tempSize == 1){//当前链表只有一个节点
                System.out.println("["+node.data+"]");
                return;
            }
            while(tempSize>0){
                if(node.equals(head)){
                    System.out.print("["+node.data+"->");
                }else if(node.next == null){
                    System.out.print(node.data+"]");
                }else{
                    System.out.print(node.data+"->");
                }
                node = node.next;
                tempSize--;
            }
            System.out.println();
        }else{//如果链表一个节点都没有，直接打印[]
            System.out.println("[]");
        }

    }

}
```

测试：

```
@Test
public void testSingleLinkedList(){
    SingleLinkedList singleList = new SingleLinkedList();
    singleList.addHead("A");
    singleList.addHead("B");
    singleList.addHead("C");
    singleList.addHead("D");
    //打印当前链表信息
    singleList.display();
    //删除C
    singleList.delete("C");
    singleList.display();
    //查找B
    System.out.println(singleList.find("B"));
}
```

#### 用单向链表实现栈



栈的pop()方法和push()方法，对应于链表的在头部删除元素deleteHead()以及在头部增加元素addHead()。

```
package com.demo.datastructure;

public class StackSingleLink {
    private SingleLinkedList link;

    public StackSingleLink(){
        link = new SingleLinkedList();
    }

    //添加元素
    public void push(Object obj){
        link.addHead(obj);
    }

    //移除栈顶元素
    public Object pop(){
        Object obj = link.deleteHead();
        return obj;
    }

    //判断是否为空
    public boolean isEmpty(){
        return link.isEmpty();
    }

    //打印栈内元素信息
    public void display(){
        link.display();
    }

}
```

### 双端链表

对于单项链表，我们如果想在尾部添加一个节点，那么必须从头部一直遍历到尾部，找到尾节点，然后在尾节点后面插入一个节点。这样操作很麻烦，如果我们在设计链表的时候多个对尾节点的引用，那么会简单很多。

**注意和后面将的双向链表的区别！！！**

双端链表的具体实现

```
package com.demo.link;

public class DoublePointLinkedList {
    private Node head;//头节点
    private Node tail;//尾节点
    private int size;//节点的个数

    private class Node{
        private Object data;
        private Node next;

        public Node(Object data){
            this.data = data;
        }
    }

    public DoublePointLinkedList(){
        size = 0;
        head = null;
        tail = null;
    }

    //链表头新增节点
    public void addHead(Object data){
        Node node = new Node(data);
        if(size == 0){//如果链表为空，那么头节点和尾节点都是该新增节点
            head = node;
            tail = node;
            size++;
        }else{
            node.next = head;
            head = node;
            size++;
        }
    }

    //链表尾新增节点
    public void addTail(Object data){
        Node node = new Node(data);
        if(size == 0){//如果链表为空，那么头节点和尾节点都是该新增节点
            head = node;
            tail = node;
            size++;
        }else{
            tail.next = node;
            tail = node;
            size++;
        }
    }

    //删除头部节点，成功返回true，失败返回false
    public boolean deleteHead(){
        if(size == 0){//当前链表节点数为0
            return false;
        }
        if(head.next == null){//当前链表节点数为1
            head = null;
            tail = null;
        }else{
            head = head.next;
        }
        size--;
        return true;
    }
    //判断是否为空
    public boolean isEmpty(){
        return (size ==0);
    }
    //获得链表的节点个数
    public int getSize(){
        return size;
    }

    //显示节点信息
    public void display(){
        if(size >0){
            Node node = head;
            int tempSize = size;
            if(tempSize == 1){//当前链表只有一个节点
                System.out.println("["+node.data+"]");
                return;
            }
            while(tempSize>0){
                if(node.equals(head)){
                    System.out.print("["+node.data+"->");
                }else if(node.next == null){
                    System.out.print(node.data+"]");
                }else{
                    System.out.print(node.data+"->");
                }
                node = node.next;
                tempSize--;
            }
            System.out.println();
        }else{//如果链表一个节点都没有，直接打印[]
            System.out.println("[]");
        }
    }

}
```

用双端链表实现队列

```
package com.demo.link;

public class QueueLinkedList {

    private DoublePointLinkedList dp;

    public QueueLinkedList(){
        dp = new DoublePointLinkedList();
    }
    public void insert(Object data){
        dp.addTail(data);
    }

    public void delete(){
        dp.deleteHead();
    }

    public boolean isEmpty(){
        return dp.isEmpty();
    }

    public int getSize(){
        return dp.getSize();
    }

    public void display(){
        dp.display();
    }

}
```

### 有序链表

前面的链表实现插入数据都是无序的，在有些应用中需要链表中的数据有序，这称为有序链表。

　　在有序链表中，数据是按照关键值有序排列的。一般在大多数需要使用有序数组的场合也可以使用有序链表。有序链表优于有序数组的地方是插入的速度（因为元素不需要移动），另外链表可以扩展到全部有效的使用内存，而数组只能局限于一个固定的大小中。

```
package com.demo.datastructure;

public class OrderLinkedList {
    private Node head;

    private class Node{
        private int data;
        private Node next;

        public Node(int data){
            this.data = data;
        }
    }

    public OrderLinkedList(){
        head = null;
    }

    //插入节点，并按照从小打到的顺序排列
    public void insert(int value){
        Node node = new Node(value);
        Node pre = null;
        Node current = head;
        while(current != null && value > current.data){
            pre = current;
            current = current.next;
        }
        if(pre == null){
            head = node;
            head.next = current;
        }else{
            pre.next = node;
            node.next = current;
        }
    }

    //删除头节点
    public void deleteHead(){
        head = head.next;
    }

    public void display(){
        Node current = head;
        while(current != null){
            System.out.print(current.data+" ");
            current = current.next;
        }
        System.out.println("");
    }

}
```

在有序链表中插入和删除某一项最多需要O(N)次比较，平均需要O(N/2)次，因为必须沿着链表上一步一步走才能找到正确的插入位置，然而可以最快速度删除最值，因为只需要删除表头即可，如果一个应用需要频繁的存取最小值，且不需要快速的插入，那么有序链表是一个比较好的选择方案。比如优先级队列可以使用有序链表来实现。

####  有序链表和无序数组组合排序

比如有一个无序数组需要排序，前面我们在讲解冒泡排序、选择排序、插入排序这三种简单的排序时，需要的时间级别都是O(N2)。

　　现在我们讲解了有序链表之后，对于一个无序数组，我们先将数组元素取出，一个一个的插入到有序链表中，然后将他们从有序链表中一个一个删除，重新放入数组，那么数组就会排好序了。和插入排序一样，如果插入了N个新数据，那么进行大概N2/4次比较。但是相对于插入排序，每个元素只进行了两次排序，一次从数组到链表，一次从链表到数组，大概需要2*N次移动，而插入排序则需要N2次移动，

　　效率肯定是比前面讲的简单排序要高，但是缺点就是需要开辟差不多两倍的空间，而且数组和链表必须在内存中同时存在，如果有现成的链表可以用，那么这种方法还是挺好的。

### 双向链表

![数据结构-双向链表](png\Java\数据结构-双向链表.png)

我们知道单向链表只能从一个方向遍历，那么双向链表它可以从两个方向遍历。

双向链表的每个节点由三部分组成：

- **prev指针**：指向前置节点
- **item节点**：数据信息
- **next指针**：指向后置节点

双向链表代码如下：

```
public class Node<E> {
   private E item;
   private Node<E> next;
   private Node<E> prev;
}
```

具体代码实现：

```
package com.demo.datastructure;

public class TwoWayLinkedList {
    private Node head;//表示链表头
    private Node tail;//表示链表尾
    private int size;//表示链表的节点个数

    private class Node{
        private Object data;
        private Node next;
        private Node prev;

        public Node(Object data){
            this.data = data;
        }
    }

    public TwoWayLinkedList(){
        size = 0;
        head = null;
        tail = null;
    }

    //在链表头增加节点
    public void addHead(Object value){
        Node newNode = new Node(value);
        if(size == 0){
            head = newNode;
            tail = newNode;
            size++;
        }else{
            head.prev = newNode;
            newNode.next = head;
            head = newNode;
            size++;
        }
    }

    //在链表尾增加节点
    public void addTail(Object value){
        Node newNode = new Node(value);
        if(size == 0){
            head = newNode;
            tail = newNode;
            size++;
        }else{
            newNode.prev = tail;
            tail.next = newNode;
            tail = newNode;
            size++;
        }
    }

    //删除链表头
    public Node deleteHead(){
        Node temp = head;
        if(size != 0){
            head = head.next;
            head.prev = null;
            size--;
        }
        return temp;
    }

    //删除链表尾
    public Node deleteTail(){
        Node temp = tail;
        if(size != 0){
            tail = tail.prev;
            tail.next = null;
            size--;
        }
        return temp;
    }

    //获得链表的节点个数
    public int getSize(){
        return size;
    }
    //判断链表是否为空
    public boolean isEmpty(){
        return (size == 0);
    }

    //显示节点信息
    public void display(){
        if(size >0){
            Node node = head;
            int tempSize = size;
            if(tempSize == 1){//当前链表只有一个节点
                System.out.println("["+node.data+"]");
                return;
            }
            while(tempSize>0){
                if(node.equals(head)){
                    System.out.print("["+node.data+"->");
                }else if(node.next == null){
                    System.out.print(node.data+"]");
                }else{
                    System.out.print(node.data+"->");
                }
                node = node.next;
                tempSize--;
            }
            System.out.println();
        }else{//如果链表一个节点都没有，直接打印[]
            System.out.println("[]");
        }

    }
}
```

我们也可以用双向链表来实现双端队列

我们讲了各种链表，每个链表都包括一个LinikedList对象和许多Node对象，LinkedList对象通常包含头和尾节点的引用，分别指向链表的第一个节点和最后一个节点。而每个节点对象通常包含数据部分data，以及对上一个节点的引用prev和下一个节点的引用next，只有下一个节点的引用称为单向链表，两个都有的称为双向链表。next值为null则说明是链表的结尾，如果想找到某个节点，我们必须从第一个节点开始遍历，不断通过next找到下一个节点，直到找到所需要的。栈和队列都是ADT，可以用数组来实现，也可以用链表实现。

### 循环链表

循环链表又分为单循环链表和双循环链表，也就是将单向链表或双向链表的首尾节点进行连接。

- **单循环链表**

  ![单循环链表](png/Java/数据结构-单循环链表.png)

- **双循环链表**

  ![双循环链表](png/Java/数据结构-双循环链表.png)



### **链表翻转算法**

**① 递归翻转**

```java
/**
 * 链表递归翻转模板
 */
public Node reverseLinkedList(参数0) {
    // Step1：终止条件
    if (终止条件) {
        return;
    }
    
    // Step2：逻辑处理：可能有，也可能没有，具体问题具体分析
    // Step3：递归调用
    Node reverse = reverseLinkedList(参数1);
    // Step4：逻辑处理：可能有，也可能没有，具体问题具体分析
}

/**
 * 链表递归翻转算法
 */
public Node reverseLinkedList(Node head) {
        // Step1：终止条件
        if (head == null || head.next == null) {
        	return head;
        }
    
        // Step2：保存当前节点的下一个结点
        Node next = head.next;
        // Step3：从当前节点的下一个结点开始递归调用
        Node reverse = reverseLinkedList(next);
        // Step4：head挂到next节点的后面就完成了链表的反转
        next.next = head;
        // 这里head相当于变成了尾结点，尾结点都是为空的，否则会构成环
        head.next = null;
        return reverse;
}
```

**② 三指针翻转**

```java
public static Node reverseLinkedList(Node head) {
    // 单链表为空或只有一个节点，直接返回原单链表
    if (head == null || head.getNext() == null) {
        return head;
    }
    
    // 前一个节点指针
    Node preNode = null;
    // 当前节点指针
    Node curNode = head;
    // 下一个节点指针
    Node nextNode = null;
    while (curNode != null) {
        // nextNode 指向下一个节点
        nextNode = curNode.getNext();
        // 将当前节点next域指向前一个节点
        curNode.setNext(preNode);
        // preNode 指针向后移动
        preNode = curNode;
        // curNode指针向后移动
        curNode = nextNode;
    }
    
    return preNode;
}
```

**③ 利用栈翻转**

```java
public Node reverseLinkedList(Node node) {
    Stack<Node> nodeStack = new Stack<>();
    // 存入栈中，模拟递归开始的栈状态
    while (node != null) {
        nodeStack.push(node);
        node = node.getNode();
    }
    
    // 特殊处理第一个栈顶元素：反转前的最后一个元素，因为它位于最后，不需要反转
    Node head = null;
    if ((!nodeStack.isEmpty())) {
        head = nodeStack.pop();
    }
    
    // 排除以后就可以快乐的循环
    while (!nodeStack.isEmpty()) {
        Node tempNode = nodeStack.pop();
        tempNode.getNode().setNode(tempNode);
        tempNode.setNode(null);
    }
    
    return head;
}
```

## 树

我们知道对于有序数组，查找很快，并介绍可以通过二分法查找，但是想要在有序数组中插入一个数据项，就必须先找到插入数据项的位置，然后将所有插入位置后面的数据项全部向后移动一位，来给新数据腾出空间，平均来讲要移动N/2次，这是很费时的。同理，删除数据也是。

　　然后我们介绍了另外一种数据结构——链表，链表的插入和删除很快，我们只需要改变一些引用值就行了，但是查找数据却很慢了，因为不管我们查找什么数据，都需要从链表的第一个数据项开始，遍历到找到所需数据项为止，这个查找也是平均需要比较N/2次。

　　那么我们就希望一种数据结构能同时具备数组查找快的优点以及链表插入和删除快的优点，于是 树 诞生了。

### 树

**树**（tree）是一种抽象数据类型（ADT），用来模拟具有树状结构性质的数据集合。它是由n（n>0）个有限**节点**通过连接它们的**边**组成一个具有层次关系的集合。把它叫做“树”是因为它看起来像一棵倒挂的树，也就是说它是根朝上，而叶朝下的。

​		①、节点：上图的圆圈，比如A,B,C等都是表示节点。节点一般代表一些实体，在java面向对象编程中，节点一般代表对象。

　　②、边：连接节点的线称为边，边表示节点的关联关系。一般从一个节点到另一个节点的**唯一方法**就是沿着一条顺着有边的道路前进。在Java当中通常表示引用。

　　树有很多种，向上面的一个节点有多余两个的子节点的树，称为多路树，后面会讲解2-3-4树和外部存储都是多路树的例子。而每个节点最多只能有两个子节点的一种形式称为二叉树

### 树的常用术语

​		①、**路径**：顺着节点的边从一个节点走到另一个节点，所经过的节点的顺序排列就称为“路径”。

　　②、**根**：树顶端的节点称为根。一棵树只有一个根，如果要把一个节点和边的集合称为树，那么从根到其他任何一个节点都必须有且只有一条路径。A是根节点。

　　③、**父节点**：若一个节点含有子节点，则这个节点称为其子节点的父节点；B是D的父节点。

　　④、**子节点**：一个节点含有的子树的根节点称为该节点的子节点；D是B的子节点。

　　⑤、**兄弟节点**：具有相同父节点的节点互称为兄弟节点；比如上图的D和E就互称为兄弟节点。

　　⑥、**叶节点**：没有子节点的节点称为叶节点，也叫叶子节点，比如上图的H、E、F、G都是叶子节点。

　　⑦、**子树**：每个节点都可以作为子树的根，它和它所有的子节点、子节点的子节点等都包含在子树中。

　　⑧、**节点的层次**：从根开始定义，根为第一层，根的子节点为第二层，以此类推。

　　⑨、**深度**：对于任意节点n,n的深度为从根到n的唯一路径长，根的深度为0；

　　⑩、**高度**：对于任意节点n,n的高度为从n到一片树叶的最长路径长，所有树叶的高度为0；

**树(Tree)**是一个分层的数据结构，由节点和连接节点的边组成，是一种特殊的图，它与图最大的区别是没有循环。树的结构十分直观，而树的很多概念定义都有一个相同的特点：递归。各种树解决的问题以及面临的新问题：

- **二叉查找树(BST)**：解决了排序的基本问题，但是由于无法保证平衡，可能退化为链表
- **平衡二叉树(AVL)**：通过旋转解决了平衡的问题，但是旋转操作效率太低
- **红黑树**：通过舍弃严格的平衡和引入红黑节点，解决了AVL旋转效率过低的问题，但是在磁盘等场景下，树仍然太高，IO次数太多
- **B树**：通过将二叉树改为多路平衡查找树，解决了树过高的问题
- **B+树**：在B树的基础上，将非叶节点改造为不存储数据的纯索引节点，进一步降低了树的高度；此外将叶节点使用指针连接成链表，范围查询更加高效

## 二叉树(Binary Tree)

![数据结构-BinaryTree](png\Java\数据结构-BinaryTree.png)

二叉树：**二叉树是每个节点最多有两个子节点的树**。二叉树的叶子节点有0个字节点，根节点或内部节点有一个或两个子节点。

**存储结构**：

```java
public class TreeNode {
    // 数据域
    private Object data;
    // 左孩子指针
    private TreeNode leftChild;
    // 右孩子指针
    private TreeNode rightChild;
}
```

### 斜树

- **左斜树**：所有结点都只有左子树
- **右斜树**：所有结点都只有右子树

![斜树](png/Java/数据结构-斜树.png)



### 满二叉树

一颗二叉树的所有分支结点都存在左子树和右子树，且所有叶子节点都只存在在最下面一层。

![满二叉树](png/Java/数据结构-满二叉树.png)



### 完全二叉树

若二叉树的深度为k，二叉树的层数从1到k-1层的结点都达到了最大个数，在第k层的所有结点都集中在最左边，这就是完全二叉树。完全二叉树由满二叉树引出，满二叉树一定是完全二叉树，但完全二叉树不一定是满二叉树。

![完全二叉树](png/Java/数据结构-完全二叉树.png)



### 二叉搜索树(Binary Search Tree)

二叉搜索树， 又叫**二叉查找树**，它是一棵空树或是具有下列性质的二叉树：

- **若左子树不空，则左子树上所有结点的值均小于它的根结点的值**
- **若右子树不空，则右子树上所有结点的值均大于它的根结点的值**
- **它的左、右子树也分别为二叉搜索树**

![BinarySearchTree](png/Java/数据结构-BinarySearchTree.png)

**效率总结** 

- **访问/查找**：最好时间复杂度 `O(logN)`，最坏时间复杂度 `O(N)`
- **插入/删除**：最好时间复杂度 `O(logN)`，最坏时间复杂度 `O(N)`

　　如果我们给二叉树加一个额外的条件，就可以得到一种被称作二叉搜索树(binary search tree)的特殊二叉树。

　　二叉搜索树要求：若它的左子树不空，则左子树上所有结点的值均小于它的根结点的值； 若它的右子树不空，则右子树上所有结点的值均大于它的根结点的值； 它的左、右子树也分别为二叉排序树。

二叉搜索树作为一种数据结构，那么它是如何工作的呢？它查找一个节点，插入一个新节点，以及删除一个节点，遍历树等工作效率如何，下面我们来一一介绍。

**二叉树的节点类：**

```
package com.demo.tree;
 
public class Node {
    private Object data;    //节点数据
    private Node leftChild; //左子节点的引用
    private Node rightChild; //右子节点的引用
    //打印节点内容
    public void display(){
        System.out.println(data);
    }
 
}
```

**二叉树的具体方法：**

```
package com.demo.tree;
 
public interface Tree {
    //查找节点
    public Node find(Object key);
    //插入新节点
    public boolean insert(Object key);
    //删除节点
    public boolean delete(Object key);
    //Other Method......
}
```

### 查找节点

查找某个节点，我们必须从根节点开始遍历。

　　①、查找值比当前节点值大，则搜索右子树；

　　②、查找值等于当前节点值，停止搜索（终止条件）；

　　③、查找值小于当前节点值，则搜索左子树；

```
//查找节点
public Node find(int key) {
    Node current = root;
    while(current != null){
        if(current.data > key){//当前值比查找值大，搜索左子树
            current = current.leftChild;
        }else if(current.data < key){//当前值比查找值小，搜索右子树
            current = current.rightChild;
        }else{
            return current;
        }
    }
    return null;//遍历完整个树没找到，返回null
}
```

用变量current来保存当前查找的节点，参数key是要查找的值，刚开始查找将根节点赋值到current。接在在while循环中，将要查找的值和current保存的节点进行对比。如果key小于当前节点，则搜索当前节点的左子节点，如果大于，则搜索右子节点，如果等于，则直接返回节点信息。当整个树遍历完全，即current == null，那么说明没找到查找值，返回null。

　　**树的效率**：查找节点的时间取决于这个节点所在的层数，每一层最多有2n-1个节点，总共N层共有2n-1个节点，那么时间复杂度为O(logN),底数为2。

　　我看评论有对这里的时间复杂度不理解，这里解释一下，O(logN)，N表示的是二叉树节点的总数，而不是层数。

### 插入节点

要插入节点，必须先找到插入的位置。与查找操作相似，由于二叉搜索树的特殊性，待插入的节点也需要从根节点开始进行比较，小于根节点则与根节点左子树比较，反之则与右子树比较，直到左子树为空或右子树为空，则插入到相应为空的位置，在比较的过程中要注意保存父节点的信息 及 待插入的位置是父节点的左子树还是右子树，才能插入到正确的位置。

```
//插入节点
public boolean insert(int data) {
    Node newNode = new Node(data);
    if(root == null){//当前树为空树，没有任何节点
        root = newNode;
        return true;
    }else{
        Node current = root;
        Node parentNode = null;
        while(current != null){
            parentNode = current;
            if(current.data > data){//当前值比插入值大，搜索左子节点
                current = current.leftChild;
                if(current == null){//左子节点为空，直接将新值插入到该节点
                    parentNode.leftChild = newNode;
                    return true;
                }
            }else{
                current = current.rightChild;
                if(current == null){//右子节点为空，直接将新值插入到该节点
                    parentNode.rightChild = newNode;
                    return true;
                }
            }
        }
    }
    return false;
}
```

### 遍历树

遍历树是根据一种特定的顺序访问树的每一个节点。比较常用的有前序遍历，中序遍历和后序遍历。而二叉搜索树最常用的是中序遍历。

　　**①、中序遍历:左子树——》根节点——》右子树**

　　**②、前序遍历:根节点——》左子树——》右子树**

　　**③、后序遍历:左子树——》右子树——》根节点**

```
//中序遍历
public void infixOrder(Node current){
    if(current != null){
        infixOrder(current.leftChild);
        System.out.print(current.data+" ");
        infixOrder(current.rightChild);
    }
}
 
//前序遍历
public void preOrder(Node current){
    if(current != null){
        System.out.print(current.data+" ");
        preOrder(current.leftChild);
        preOrder(current.rightChild);
    }
}
 
//后序遍历
public void postOrder(Node current){
    if(current != null){
        postOrder(current.leftChild);
        postOrder(current.rightChild);
        System.out.print(current.data+" ");
    }
}
```

### 查找最大值和最小值

这没什么好说的，要找最小值，先找根的左节点，然后一直找这个左节点的左节点，直到找到没有左节点的节点，那么这个节点就是最小值。同理要找最大值，一直找根节点的右节点，直到没有右节点，则就是最大值。

```
//找到最大值
public Node findMax(){
    Node current = root;
    Node maxNode = current;
    while(current != null){
        maxNode = current;
        current = current.rightChild;
    }
    return maxNode;
}
//找到最小值
public Node findMin(){
    Node current = root;
    Node minNode = current;
    while(current != null){
        minNode = current;
        current = current.leftChild;
    }
    return minNode;
}
```

### 删除节点

删除节点是二叉搜索树中最复杂的操作，删除的节点有三种情况，前两种比较简单，但是第三种却很复杂。

　　1、该节点是叶节点（没有子节点）

　　2、该节点有一个子节点

　　3、该节点有两个子节点

　　下面我们分别对这三种情况进行讲解。

①、删除没有子节点的节点

　　要删除叶节点，只需要改变该节点的父节点引用该节点的值，即将其引用改为 null 即可。要删除的节点依然存在，但是它已经不是树的一部分了，由于Java语言的垃圾回收机制，我们不需要非得把节点本身删掉，一旦Java意识到程序不在与该节点有关联，就会自动把它清理出存储器。

```
@Override
public boolean delete(int key) {
    Node current = root;
    Node parent = root;
    boolean isLeftChild = false;
    //查找删除值，找不到直接返回false
    while(current.data != key){
        parent = current;
        if(current.data > key){
            isLeftChild = true;
            current = current.leftChild;
        }else{
            isLeftChild = false;
            current = current.rightChild;
        }
        if(current == null){
            return false;
        }
    }
    //如果当前节点没有子节点
    if(current.leftChild == null && current.rightChild == null){
        if(current == root){
            root = null;
        }else if(isLeftChild){
            parent.leftChild = null;
        }else{
            parent.rightChild = null;
        }
        return true;
    }
    return false;
}
```

删除节点，我们要先找到该节点，并记录该节点的父节点。在检查该节点是否有子节点。如果没有子节点，接着检查其是否是根节点，如果是根节点，只需要将其设置为null即可。如果不是根节点，是叶节点，那么断开父节点和其的关系即可。

②、删除有一个子节点的节点

　　删除有一个子节点的节点，我们只需要将其父节点原本指向该节点的引用，改为指向该节点的子节点即可。

```
//当前节点有一个子节点
if(current.leftChild == null && current.rightChild != null){
    if(current == root){
        root = current.rightChild;
    }else if(isLeftChild){
        parent.leftChild = current.rightChild;
    }else{
        parent.rightChild = current.rightChild;
    }
    return true;
}else{
    //current.leftChild != null && current.rightChild == null
    if(current == root){
        root = current.leftChild;
    }else if(isLeftChild){
        parent.leftChild = current.leftChild;
    }else{
        parent.rightChild = current.leftChild;
    }
    return true;
}
```

③、删除有两个子节点的节点

当删除的节点存在两个子节点，那么删除之后，两个子节点的位置我们就没办法处理了。既然处理不了，我们就想到一种办法，用另一个节点来代替被删除的节点，那么用哪一个节点来代替呢？

　　我们知道二叉搜索树中的节点是按照关键字来进行排列的，某个节点的关键字次高节点是它的中序遍历后继节点。用后继节点来代替删除的节点，显然该二叉搜索树还是有序的。（这里用后继节点代替，如果该后继节点自己也有子节点，我们后面讨论。）

那么如何找到删除节点的中序后继节点呢？其实我们稍微分析，这实际上就是要找比删除节点关键值大的节点集合中最小的一个节点，只有这样代替删除节点后才能满足二叉搜索树的特性。

　　后继节点也就是：比删除节点大的最小节点。

　　算法：程序找到删除节点的右节点，(注意这里前提是删除节点存在左右两个子节点，如果不存在则是删除情况的前面两种)，然后转到该右节点的左子节点，依次顺着左子节点找下去，最后一个左子节点即是后继节点；如果该右节点没有左子节点，那么该右节点便是后继节点。

需要确定后继节点没有子节点，如果后继节点存在子节点，那么又要分情况讨论了。

　　①、后继节点是删除节点的右子节点

　　这种情况简单，只需要将后继节点表示的子树移到被删除节点的位置即可！

​		②、后继节点是删除节点的右子节点的左子节点

```
public Node getSuccessor(Node delNode){
    Node successorParent = delNode;
    Node successor = delNode;
    Node current = delNode.rightChild;
    while(current != null){
        successorParent = successor;
        successor = current;
        current = current.leftChild;
    }
    //将后继节点替换删除节点
    if(successor != delNode.rightChild){
        successorParent.leftChild = successor.rightChild;
        successor.rightChild = delNode.rightChild;
    }
     
    return successor;
}
```

④、删除有必要吗？

通过上面的删除分类讨论，我们发现删除其实是挺复杂的，那么其实我们可以不用真正的删除该节点，只需要在Node类中增加一个标识字段isDelete，当该字段为true时，表示该节点已经删除，反正没有删除。那么我们在做比如find()等操作的时候，要先判断isDelete字段是否为true。这样删除的节点并不会改变树的结构。

```
public class Node {
    int data;   //节点数据
    Node leftChild; //左子节点的引用
    Node rightChild; //右子节点的引用
    boolean isDelete;//表示节点是否被删除
}

```

### 二叉树的效率

从前面的大部分对树的操作来看，都需要从根节点到下一层一层的查找。

　　一颗满树，每层节点数大概为2n-1，那么最底层的节点个数比树的其它节点数多1，因此，查找、插入或删除节点的操作大约有一半都需要找到底层的节点，另外四分之一的节点在倒数第二层，依次类推。

　　总共N层共有2n-1个节点，那么时间复杂度为O(logn),底数为2。

　　在有1000000 个数据项的无序数组和链表中，查找数据项平均会比较500000 次，但是在有1000000个节点的二叉树中，只需要20次或更少的比较即可。

　　有序数组可以很快的找到数据项，但是插入数据项的平均需要移动 500000 次数据项，在 1000000 个节点的二叉树中插入数据项需要20次或更少比较，在加上很短的时间来连接数据项。

　　同样，从 1000000 个数据项的数组中删除一个数据项平均需要移动 500000 个数据项，而在 1000000 个节点的二叉树中删除节点只需要20次或更少的次数来找到他，然后在花一点时间来找到它的后继节点，一点时间来断开节点以及连接后继节点。

　　所以，树对所有常用数据结构的操作都有很高的效率。

　　遍历可能不如其他操作快，但是在大型数据库中，遍历是很少使用的操作，它更常用于程序中的辅助算法来解析算术或其它表达式。

### 用数组表示树

用数组表示树，那么节点是存在数组中的，节点在数组中的位置对应于它在树中的位置。下标为 0 的节点是根，下标为 1 的节点是根的左子节点，以此类推，按照从左到右的顺序存储树的每一层。

树中的每个位置，无论是否存在节点，都对应于数组中的一个位置，树中没有节点的在数组中用0或者null表示。

　　假设节点的索引值为index，那么节点的左子节点是 2*index+1，节点的右子节点是 2*index+2，它的父节点是 （index-1）/2。

　　在大多数情况下，使用数组表示树效率是很低的，不满的节点和删除掉的节点都会在数组中留下洞，浪费存储空间。更坏的是，删除节点如果要移动子树的话，子树中的每个节点都要移到数组中新的位置，这是很费时的。

　　不过如果不允许删除操作，数组表示可能会很有用，尤其是因为某种原因要动态的为每个字节分配空间非常耗时。

### 完整的BinaryTree代码

Node

```
package com.demo.tree;
 
public class Node {
    int data;   //节点数据
    Node leftChild; //左子节点的引用
    Node rightChild; //右子节点的引用
    boolean isDelete;//表示节点是否被删除
     
    public Node(int data){
        this.data = data;
    }
    //打印节点内容
    public void display(){
        System.out.println(data);
    }
 
}
```

Tree

```
package com.demo.tree;
 
public interface Tree {
    //查找节点
    public Node find(int key);
    //插入新节点
    public boolean insert(int data);
     
    //中序遍历
    public void infixOrder(Node current);
    //前序遍历
    public void preOrder(Node current);
    //后序遍历
    public void postOrder(Node current);
     
    //查找最大值
    public Node findMax();
    //查找最小值
    public Node findMin();
     
    //删除节点
    public boolean delete(int key);
     
    //Other Method......
}
```

BinaryTree

```
 
public class BinaryTree implements Tree {
    //表示根节点
    private Node root;
 
    //查找节点
    public Node find(int key) {
        Node current = root;
        while(current != null){
            if(current.data > key){//当前值比查找值大，搜索左子树
                current = current.leftChild;
            }else if(current.data < key){//当前值比查找值小，搜索右子树
                current = current.rightChild;
            }else{
                return current;
            }
        }
        return null;//遍历完整个树没找到，返回null
    }
 
    //插入节点
    public boolean insert(int data) {
        Node newNode = new Node(data);
        if(root == null){//当前树为空树，没有任何节点
            root = newNode;
            return true;
        }else{
            Node current = root;
            Node parentNode = null;
            while(current != null){
                parentNode = current;
                if(current.data > data){//当前值比插入值大，搜索左子节点
                    current = current.leftChild;
                    if(current == null){//左子节点为空，直接将新值插入到该节点
                        parentNode.leftChild = newNode;
                        return true;
                    }
                }else{
                    current = current.rightChild;
                    if(current == null){//右子节点为空，直接将新值插入到该节点
                        parentNode.rightChild = newNode;
                        return true;
                    }
                }
            }
        }
        return false;
    }
     
    //中序遍历
    public void infixOrder(Node current){
        if(current != null){
            infixOrder(current.leftChild);
            System.out.print(current.data+" ");
            infixOrder(current.rightChild);
        }
    }
     
    //前序遍历
    public void preOrder(Node current){
        if(current != null){
            System.out.print(current.data+" ");
            infixOrder(current.leftChild);
            infixOrder(current.rightChild);
        }
    }
     
    //后序遍历
    public void postOrder(Node current){
        if(current != null){
            infixOrder(current.leftChild);
            infixOrder(current.rightChild);
            System.out.print(current.data+" ");
        }
    }
    //找到最大值
    public Node findMax(){
        Node current = root;
        Node maxNode = current;
        while(current != null){
            maxNode = current;
            current = current.rightChild;
        }
        return maxNode;
    }
    //找到最小值
    public Node findMin(){
        Node current = root;
        Node minNode = current;
        while(current != null){
            minNode = current;
            current = current.leftChild;
        }
        return minNode;
    }
     
    @Override
    public boolean delete(int key) {
        Node current = root;
        Node parent = root;
        boolean isLeftChild = false;
        //查找删除值，找不到直接返回false
        while(current.data != key){
            parent = current;
            if(current.data > key){
                isLeftChild = true;
                current = current.leftChild;
            }else{
                isLeftChild = false;
                current = current.rightChild;
            }
            if(current == null){
                return false;
            }
        }
        //如果当前节点没有子节点
        if(current.leftChild == null && current.rightChild == null){
            if(current == root){
                root = null;
            }else if(isLeftChild){
                parent.leftChild = null;
            }else{
                parent.rightChild = null;
            }
            return true;
             
            //当前节点有一个子节点，右子节点
        }else if(current.leftChild == null && current.rightChild != null){
            if(current == root){
                root = current.rightChild;
            }else if(isLeftChild){
                parent.leftChild = current.rightChild;
            }else{
                parent.rightChild = current.rightChild;
            }
            return true;
            //当前节点有一个子节点，左子节点
        }else if(current.leftChild != null && current.rightChild == null){
            if(current == root){
                root = current.leftChild;
            }else if(isLeftChild){
                parent.leftChild = current.leftChild;
            }else{
                parent.rightChild = current.leftChild;
            }
            return true;
        }else{
            //当前节点存在两个子节点
            Node successor = getSuccessor(current);
            if(current == root){
                root= successor;
            }else if(isLeftChild){
                parent.leftChild = successor;
            }else{
                parent.rightChild = successor;
            }
            successor.leftChild = current.leftChild;
        }
        return false;
         
    }
 
    public Node getSuccessor(Node delNode){
        Node successorParent = delNode;
        Node successor = delNode;
        Node current = delNode.rightChild;
        while(current != null){
            successorParent = successor;
            successor = current;
            current = current.leftChild;
        }
        //后继节点不是删除节点的右子节点，将后继节点替换删除节点
        if(successor != delNode.rightChild){
            successorParent.leftChild = successor.rightChild;
            successor.rightChild = delNode.rightChild;
        }
         
        return successor;
    }
     
    public static void main(String[] args) {
        BinaryTree bt = new BinaryTree();
        bt.insert(50);
        bt.insert(20);
        bt.insert(80);
        bt.insert(10);
        bt.insert(30);
        bt.insert(60);
        bt.insert(90);
        bt.insert(25);
        bt.insert(85);
        bt.insert(100);
        bt.delete(10);//删除没有子节点的节点
        bt.delete(30);//删除有一个子节点的节点
        bt.delete(80);//删除有两个子节点的节点
        System.out.println(bt.findMax().data);
        System.out.println(bt.findMin().data);
        System.out.println(bt.find(100));
        System.out.println(bt.find(200));
         
    }
 
}
```

### 哈夫曼(Huffman)编码

我们知道计算机里每个字符在没有压缩的文本文件中由一个字节（比如ASCII码）或两个字节（比如Unicode,这个编码在各种语言中通用）表示，在这些方案中，每个字符需要相同的位数。

　　有很多压缩数据的方法，就是减少表示最常用字符的位数量，比如英语中，E是最常用的字母，我们可以只用两位01来表示，2位有四种组合：00、01、10、11，那么我们可以用这四种组合表示四种常用的字符吗？

　　答案是不可以的，因为在编码序列中是没有空格或其他特殊字符存在的，全都是有0和1构成的序列，比如E用01来表示，X用01011000表示，那么在解码的时候就弄不清楚01是表示E还是表示X的起始部分，所以在编码的时候就定下了一个规则：**每个代码都不能是其它代码的前缀。**

　　①、哈夫曼编码　

　　二叉树中有一种特别的树——哈夫曼树（最优二叉树），其通过某种规则（权值）来构造出一哈夫曼二叉树，在这个二叉树中，只有叶子节点才是有效的数据节点（很重要），其他的非叶子节点是为了构造出哈夫曼而引入的！
哈夫曼编码是一个通过哈夫曼树进行的一种编码，一般情况下，以字符：‘0’与‘1’表示。编码的实现过程很简单，只要实现哈夫曼树，通过遍历哈夫曼树，规定向左子树遍历一个节点编码为“0”，向右遍历一个节点编码为“1”，结束条件就是遍历到叶子节点！因为上面说过：哈夫曼树叶子节点才是有效数据节点！

我们用01表示S，用00表示空格后，就不能用01和11表示某个字符了，因为它们是其它字符的前缀。在看三位的组合，分别有000,001,010,100,101,110和111，A是010，I是110，为什么没有其它三位的组合了呢？因为已知是不能用01和11开始的组合了，那么就减少了四种选择，同时011用于U和换行符的开始，111用于E和Y的开始，这样就只剩下2个三位的组合了，同理可以理解为什么只有三个四位的代码可用。

　　所以对于消息：SUSIE SAYS IT IS EASY

　　哈夫曼编码为：100111110110111100100101110100011001100011010001111010101110

　　②、哈夫曼解码

　　如果收到上面的一串哈夫曼编码，怎么解码呢？消息中出现的字符在哈夫曼树中是叶节点，也就是没有子节点，如下图：它们在消息中出现的频率越高，在树中的位置就越高，每个圆圈外面的数字就是频率，非叶节点外面的数字是它子节点数字的和。

　　每个字符都从根开始，如果遇到0，就向左走到下一个节点，如果遇到1，就向右。比如字符A是010，那么先向左，再向右，再向左，就找到了A，其它的依次类推。

树是由边和节点构成，根节点是树最顶端的节点，它没有父节点；二叉树中，最多有两个子节点；某个节点的左子树每个节点都比该节点的关键字值小，右子树的每个节点都比该节点的关键字值大，那么这种树称为二叉搜索树，其查找、插入、删除的时间复杂度都为logN；可以通过前序遍历、中序遍历、后序遍历来遍历树，前序是根节点-左子树-右子树，中序是左子树-根节点-右子树，后序是左子树-右子树-根节点；删除一个节点只需要断开指向它的引用即可；哈夫曼树是二叉树，用于数据压缩算法，最经常出现的字符编码位数最少，很少出现的字符编码位数多一些。

## 平衡二叉树(AVL Tree)

二叉查找树在最差情况下竟然和顺序查找效率相当，这是无法仍受的。事实也证明，当存储数据足够大的时候，树的结构对某些关键字的查找效率影响很大。当然，造成这种情况的主要原因就是BST不够平衡(左右子树高度差太大)。既然如此，那么我们就需要通过一定的算法，将不平衡树改变成平衡树。因此，AVL树就诞生了。

**平衡二叉树全称叫做 `平衡二叉搜索（排序）树`，简称 AVL树**。高度为 `logN`。本质是一颗二叉查找树，AVL树的特性：

- 它是**一棵空树**或**左右两个子树的高度差**的绝对值不超过 `1`
- 左右两个子树也都是一棵平衡二叉树

如下图，根节点左边高度是3，因为左边最多有3条边；右边高度而2，相差1。根节点左边的节点50的左边是1条边，高度为1，右边有两条边，高度为2，相差1。

![AVLTree](png/Java/数据结构-AVLTree.png)

**效率总结** 

- 查找：时间复杂度维持在`O(logN)`，不会出现最差情况
- 插入：插入操作时最多需要 `1` 次旋转，其时间复杂度在`O(logN)`左右
- 删除：删除时代价稍大，执行每个删除操作的时间复杂度需要`O(2logN)`

## 红黑树

那么为了能够以较快的时间O(logN)来搜索一棵树，我们需要保证树总是平衡的（或者大部分是平衡的），也就是说每个节点的左子树节点个数和右子树节点个数尽量相等。红-黑树的就是这样的一棵平衡树，对一个要插入的数据项（删除也是），插入例程要检查会不会破坏树的特征，如果破坏了，程序就会进行纠正，根据需要改变树的结构，从而保持树的平衡。

### 红-黑树的特征

二叉平衡树的严格平衡策略以**牺牲建立查找结构(插入，删除操作)的代**价，换来了稳定的O(logN) 的查找时间复杂度。但是这样做是否值得呢？ 能不能找一种折中策略，即不牺牲太大的建立查找结构的代价，也能保证稳定高效的查找效率呢？ 答案就是：红黑树。

**红黑树是一种含有红、黑结点，并能自平衡的二叉查找树**。高度为 `logN`。其性质如下：

- **每个结点或是红色的，或是黑色的**
- **根节点是黑色的**
- **每个叶子节点（NIL）是黑色的**
- **如果一个节点是红色的，则它的两个子节点都是黑色的（反之不一定）,(也就是从每个叶子到根的所有路径上不能有两个连续的红色节点)；**
- **从根节点到叶节点或空子节点的每条路径，必须包含相同数目的黑色节点（即相同的黑色高度）。**

正是红黑树的这5条性质，使一棵n个结点的红黑树始终保持了logN的高度，从而也就解释了“红黑树的查找、插入、删除的时间复杂度最坏为O(logN)”这一结论成立的原因。

![Red-BlackTree](png/Java/数据结构-Red-BlackTree.jpg)

**效率总结** 

- **查找**：最好情况下时间复杂度为`O(logN)`，但在最坏情况下比`AVL`要差一些，但也远远好于`BST`

- **插入/删除**：改变树的平衡性的概率要远远小于`AVL`（RBT不是高度平衡的）

  因此需要旋转操作的可能性要小，且一旦需要旋转，插入一个结点最多只需旋转`2`次，删除最多只需旋转`3`次(小于`AVL`的删除操作所需要的旋转次数)。虽然变色操作的时间复杂度在`O(logN)`，但是实际上，这种操作由于简单所需要的代价很小

  

　红黑树有如下两个特征：

　　①、节点都有颜色；

　　②、在插入和删除的过程中，要遵循保持这些颜色的不同排列规则。

　　第一个很好理解，在红-黑树中，每个节点的颜色或者是黑色或者是红色的。当然也可以是任意别的两种颜色，这里的颜色用于标记，我们可以在节点类Node中增加一个boolean型变量isRed，以此来表示颜色的信息。

　　第二点，在插入或者删除一个节点时，必须要遵守的规则称为红-黑规则：

　　**1.每个节点不是红色就是黑色的；**

　　**2.根节点总是黑色的；**

　　**3.如果节点是红色的，则它的子节点必须是黑色的（反之不一定）,(也就是从每个叶子到根的所有路径上不能有两个连续的红色节点)；**

　　**4.从根节点到叶节点或空子节点的每条路径，必须包含相同数目的黑色节点（即相同的黑色高度）。**

　　从根节点到叶节点的路径上的黑色节点的数目称为黑色高度，规则 4 另一种表示就是从根到叶节点路径上的黑色高度必须相同。

　　注意：**新插入的节点颜色总是红色的**，这是因为插入一个红色节点比插入一个黑色节点违背红-黑规则的可能性更小，原因是插入黑色节点总会改变黑色高度（违背规则4），但是插入红色节点只有一半的机会会违背规则3（因为父节点是黑色的没事，父节点是红色的就违背规则3）。另外违背规则3比违背规则4要更容易修正。当插入一个新的节点时，可能会破坏这种平衡性，那么红-黑树是如何修正的呢？

### 红-黑树的自我修正

红-黑树主要通过三种方式对平衡进行修正，改变节点颜色、左旋和右旋。

　　①、改变节点颜色

　　![img](https://images2017.cnblogs.com/blog/1120165/201712/1120165-20171220164306537-599617363.png)

　　新插入的节点为15，一般新插入颜色都为红色，那么我们发现直接插入会违反规则3，改为黑色却发现违反规则4。这时候我们将其父节点颜色改为黑色，父节点的兄弟节点颜色也改为黑色。通常其祖父节点50颜色会由黑色变为红色，但是由于50是根节点，所以我们这里不能改变根节点颜色。

　　②、右旋

　　首先要说明的是节点本身是不会旋转的，旋转改变的是节点之间的关系，选择一个节点作为旋转的顶端，如果做一次右旋，这个顶端节点会向下和向右移动到它右子节点的位置，它的左子节点会上移到它原来的位置。右旋的顶端节点必须要有左子节点。

　　![img](https://images2017.cnblogs.com/blog/1120165/201712/1120165-20171220170015303-267904943.gif)

　　③、左旋

　　左旋的顶端节点必须要有右子节点。

　　![img](https://images2017.cnblogs.com/blog/1120165/201712/1120165-20171220170041631-2108296110.gif)

 　**注意**：我们改变颜色也是为了帮助我们判断何时执行什么旋转，而旋转是为了保证树的平衡。光改变节点颜色是不能起到任何作用的，旋转才是关键的操作，在新增节点或者删除节点之后，可能会破坏二叉树的平衡，那么何时执行旋转以及执行什么旋转，这是我们需要重点关注的。

### 左旋和右旋代码

①、节点类

　　节点类和二叉树的节点类差不多，只不过在其基础上增加了一个 boolean 类型的变量来表示节点的颜色。

```
public class RBNode<T extends Comparable<T>> {
    boolean color;//颜色
    T key;//关键值
    RBNode<T> left;//左子节点
    RBNode<T> right;//右子节点
    RBNode<T> parent;//父节点
     
    public RBNode(boolean color,T key,RBNode<T> parent,RBNode<T> left,RBNode<T> right){
        this.color = color;
        this.key = key;
        this.parent = parent;
        this.left = left;
        this.right = right;
    }
     
    //获得节点的关键值
    public T getKey(){
        return key;
    }
    //打印节点的关键值和颜色信息
    public String toString(){
        return ""+key+(this.color == RED ? "R":"B");
    }
}
```

　②、左旋的具体实现

```
/*************对红黑树节点x进行左旋操作 ******************/
/* 
 * 左旋示意图：对节点x进行左旋 
 *     p                       p 
 *    /                       / 
 *   x                       y 
 *  / \                     / \ 
 * lx  y      ----->       x  ry 
 *    / \                 / \ 
 *   ly ry               lx ly 
 * 左旋做了三件事： 
 * 1. 将y的左子节点赋给x的右子节点,并将x赋给y左子节点的父节点(y左子节点非空时) 
 * 2. 将x的父节点p(非空时)赋给y的父节点，同时更新p的子节点为y(左或右) 
 * 3. 将y的左子节点设为x，将x的父节点设为y 
 */
private void leftRotate(RBNode<T> x){
    //1. 将y的左子节点赋给x的右子节点，并将x赋给y左子节点的父节点(y左子节点非空时)
    RBNode<T> y = x.right;
    x.right = y.left;
    if(y.left != null){
        y.left.parent = x;
    }
     
    //2. 将x的父节点p(非空时)赋给y的父节点，同时更新p的子节点为y(左或右)
    y.parent = x.parent;
    if(x.parent == null){
        this.root = y;//如果x的父节点为空(即x为根节点)，则将y设为根节点
    }else{
        if(x == x.parent.left){//如果x是左子节点
            x.parent.left = y;//则也将y设为左子节点 
        }else{
            x.parent.right = y;//否则将y设为右子节点 
        }
    }
     
    //3. 将y的左子节点设为x，将x的父节点设为y
    y.left = x;
    x.parent = y;
}
```

③、右旋的具体实现　　

```
/*************对红黑树节点y进行右旋操作 ******************/ 
/*
 * 左旋示意图：对节点y进行右旋
 *        p                   p
 *       /                   /
 *      y                   x
 *     / \                 / \
 *    x  ry   ----->      lx  y
 *   / \                     / \
 * lx  rx                   rx ry
 * 右旋做了三件事：
 * 1. 将x的右子节点赋给y的左子节点,并将y赋给x右子节点的父节点(x右子节点非空时)
 * 2. 将y的父节点p(非空时)赋给x的父节点，同时更新p的子节点为x(左或右)
 * 3. 将x的右子节点设为y，将y的父节点设为x
 */
private void rightRotate(RBNode<T> y){
    //1. 将y的左子节点赋给x的右子节点，并将x赋给y左子节点的父节点(y左子节点非空时)
    RBNode<T> x = y.left;
    y.left = x.right;
    if(x.right != null){
        x.right.parent = y;
    }
     
    //2. 将x的父节点p(非空时)赋给y的父节点，同时更新p的子节点为y(左或右)
    x.parent = y.parent;
    if(y.parent == null){
        this.root = x;//如果y的父节点为空(即y为根节点)，则旋转后将x设为根节点
    }else{
        if(y == y.parent.left){//如果y是左子节点
            y.parent.left = x;//则将x也设置为左子节点
        }else{
            y.parent.right = x;//否则将x设置为右子节点
        }
    }
     
    //3. 将x的左子节点设为y，将y的父节点设为y
    x.right = y;
    y.parent = x;
}
```

### 插入操作

和二叉树的插入操作一样，都是得先找到插入的位置，然后再将节点插入。先看看插入的前段代码：

```
/*********************** 向红黑树中插入节点 **********************/
public void insert(T key){
    RBNode<T> node = new RBNode<T>(RED, key, null, null, null);
    if(node != null){
        insert(node);
    }
}
public void insert(RBNode<T> node){
    RBNode<T> current = null;//表示最后node的父节点
    RBNode<T> x = this.root;//用来向下搜索
     
    //1.找到插入位置
    while(x != null){
        current = x;
        int cmp = node.key.compareTo(x.key);
        if(cmp < 0){
            x = x.left;
        }else{
            x = x.right;
        }
    }
    node.parent = current;//找到了插入的位置，将当前current作为node的父节点
     
    //2.接下来判断node是左子节点还是右子节点
    if(current != null){
        int cmp = node.key.compareTo(current.key);
        if(cmp < 0){
            current.left = node;
        }else{
            current.right = node;
        }
    }else{
        this.root = node;
    }
     
    //3.利用旋转操作将其修正为一颗红黑树
    insertFixUp(node);
}
```

这与二叉搜索树中实现的思路一样，这里不再赘述，主要看看方法里面最后一步insertFixUp(node)操作。因为插入后可能会导致树的不平衡，insertFixUp(node) 方法里主要是分情况讨论，分析何时变色，何时左旋，何时右旋。我们先从理论上分析具体的情况，然后再看insertFixUp(node) 的具体实现。

　　如果是第一次插入，由于原树为空，所以只会违反红-黑树的规则2，所以只要把根节点涂黑即可；如果插入节点的父节点是黑色的，那不会违背红-黑树的规则，什么也不需要做；但是遇到如下三种情况，我们就要开始变色和旋转了：

　　①、插入节点的父节点和其叔叔节点（祖父节点的另一个子节点）均为红色。

　　②、插入节点的父节点是红色的，叔叔节点是黑色的，且插入节点是其父节点的右子节点。

　　③、插入节点的父节点是红色的，叔叔节点是黑色的，且插入节点是其父节点的左子节点。

　　下面我们挨个分析这三种情况都需要如何操作，然后给出实现代码。

　　在下面的讨论中，使用N,P,G,U表示关联的节点。N(now)表示当前节点，P(parent)表示N的父节点，U(uncle)表示N的叔叔节点，G(grandfather)表示N的祖父节点，也就是P和U的父节点。

　　对于情况1：**插入节点的父节点和其叔叔节点（祖父节点的另一个子节点）均为红色。**此时，肯定存在祖父节点，但是不知道父节点是其左子节点还是右子节点，但是由于对称性，我们只要讨论出一边的情况，另一种情况自然也与之对应。这里考虑父节点是其祖父节点的左子节点的情况，如下左图所示：

　　![img](https://images2017.cnblogs.com/blog/1120165/201712/1120165-20171224131058990-520825689.png)  　　　　　![img](https://images2017.cnblogs.com/blog/1120165/201712/1120165-20171224131107115-1429056985.png)

 

 　对于这种情况，我们要做的操作有：将当前节点(4) 的父节点(5) 和叔叔节点(8) 涂黑，将祖父节点(7)涂红,变成了上有图所示的情况。再将当前节点指向其祖父节点，再次从新的当前节点开始算法（具体看下面的步骤）。这样上右图就变成情况2了。

　　对于情况2：**插入节点的父节点是红色的，叔叔节点是黑色的，且插入节点是其父节点的右子节点**。我们要做的操作有：将当前节点(7)的父节点(2)作为新的节点，以新的当前节点为支点做左旋操作。完成后如左下图所示，这样左下图就变成情况3了。

 　![img](https://images2017.cnblogs.com/blog/1120165/201712/1120165-20171224132524225-1171882786.png)　　　　![img](https://images2017.cnblogs.com/blog/1120165/201712/1120165-20171224132533537-1859473656.png)

 

 　对于情况3：**插入节点的父节点是红色，叔叔节点是黑色，且插入节点是其父节点的左子节点**。我们要做的操作有：将当前节点的父节点(7)涂黑，将祖父节点(11)涂红，在祖父节点为支点做右旋操作。最后把根节点涂黑，整个红-黑树重新恢复了平衡，如右上图所示。至此，插入操作完成！

　　我们可以看出，如果是从情况1开始发生的，必然会走完情况2和3，也就是说这是一整个流程，当然咯，实际中可能不一定会从情况1发生，如果从情况2开始发生，那再走个情况3即可完成调整，如果直接只要调整情况3，那么前两种情况均不需要调整了。故变色和旋转之间的先后关系可以表示为：变色->左旋->右旋。

　　至此，我们完成了全部的插入操作。下面我们看看insertFixUp方法中的具体实现（可以结合上面的分析图，更加利与理解）：

```
private void insertFixUp(RBNode<T> node){
    RBNode<T> parent,gparent;//定义父节点和祖父节点
     
    //需要修正的条件：父节点存在，且父节点的颜色是红色
    while(((parent = parentOf(node)) != null) && isRed(parent)){
        gparent = parentOf(parent);//获得祖父节点
         
        //若父节点是祖父节点的左子节点，下面的else相反
        if(parent == gparent.left){
            RBNode<T> uncle = gparent.right;//获得叔叔节点
             
            //case1:叔叔节点也是红色
            if(uncle != null && isRed(uncle)){
                setBlack(parent);//把父节点和叔叔节点涂黑
                setBlack(gparent);
                setRed(gparent);//把祖父节点涂红
                node = gparent;//把位置放到祖父节点处
                continue;//继续while循环，重新判断
            }
             
            //case2:叔叔节点是黑色，且当前节点是右子节点
            if(node == parent.right){
                leftRotate(parent);//从父节点出左旋
                RBNode<T> tmp = parent;//然后将父节点和自己调换一下，为下面右旋做准备
                parent = node;
                node = tmp;
            }
             
            //case3:叔叔节点是黑色，且当前节点是左子节点
            setBlack(parent);
            setRed(gparent);
            rightRotate(gparent);
        }else{//若父节点是祖父节点的右子节点，与上面的情况完全相反，本质是一样的
            RBNode<T> uncle = gparent.left;
             
            //case1:叔叔节点也是红色的
            if(uncle != null && isRed(uncle)){
                setBlack(parent);
                setBlack(uncle);
                setRed(gparent);
                node = gparent;
                continue;
            }
             
            //case2:叔叔节点是黑色的，且当前节点是左子节点
            if(node == parent.left){
                rightRotate(parent);
                RBNode<T> tmp = parent;
                parent = node;
                node = tmp;
            }
             
            //case3:叔叔节点是黑色的，且当前节点是右子节点
            setBlack(parent);
            setRed(gparent);
            leftRotate(gparent);
        }
    }
    setBlack(root);//将根节点设置为黑色
}
```

### 删除操作

上面探讨完了红-黑树的插入操作，接下来讨论删除，红-黑树的删除和二叉查找树的删除是一样的，只不过删除后多了个平衡的修复而已。我们先来回忆一下二叉搜索树的删除：

　　①、如果待删除的节点没有子节点，那么直接删除即可。

　　②、如果待删除的节点只有一个子节点，那么直接删掉，并用其子节点去顶替它。

　　③、如果待删除的节点有两个子节点，这种情况比较复杂：首先找出它的后继节点，然后处理“后继节点”和“被删除节点的父节点”之间的关系，最后处理“后继节点的子节点”和“被删除节点的子节点”之间的关系。每一步中也会有不同的情况。

　　实际上，删除过程太复杂了，很多情况下会采用在节点类中添加一个删除标记，并不是真正的删除节点。详细的删除我们这里不做讨论。

### 红黑树的效率

红黑树的查找、插入和删除时间复杂度都为O(log2N)，额外的开销是每个节点的存储空间都稍微增加了一点，因为一个存储红黑树节点的颜色变量。插入和删除的时间要增加一个常数因子，因为要进行旋转，平均一次插入大约需要一次旋转，因此插入的时间复杂度还是O(log2N),(时间复杂度的计算要省略常数)，但实际上比普通的二叉树是要慢的。

　　大多数应用中，查找的次数比插入和删除的次数多，所以应用红黑树取代普通的二叉搜索树总体上不会有太多的时间开销。而且红黑树的优点是对于有序数据的操作不会慢到O(N)的时间复杂度。

## B-树(Balance Tree)

![B-tree图解](png/Java/数据结构-B-tree.png)

对于在内存中的查找结构而言，红黑树的效率已经非常好了(实际上很多实际应用还对RBT进行了优化)。但是如果是数据量非常大的查找呢？将这些数据全部放入内存组织成RBT结构显然是不实际的。实际上，像OS中的文件目录存储，数据库中的文件索引结构的存储…. 都不可能在内存中建立查找结构。必须在磁盘中建立好这个结构。
在磁盘中组织查找结构，从任何一个结点指向其他结点都有可能读取一次磁盘数据，再将数据写入内存进行比较。大家都知道，频繁的磁盘IO操作，效率是很低下的(机械运动比电子运动要慢不知道多少)。显而易见，所有的二叉树的查找结构在磁盘中都是低效的。因此，B树很好的解决了这一个问题。



**B树也称B-树、B-Tree，它是一颗多路平衡查找树**。描述一颗B树时需要指定它的阶数，阶数表示了一个结点最多有多少个孩子结点，一般用字母m表示阶数。当m取2时，就是我们常见的二叉搜索树。B树的定义：

- **每个结点最多有m-1个关键字**
- **根结点最少可以只有1个关键字**
- **非根结点至少有Math.ceil(m/2)-1个关键字**
- **每个结点中的关键字都按照从小到大的顺序排列，每个关键字的左子树中的所有关键字都小于它，而右子树中的所有关键字都大于它**
- **所有叶子结点都位于同一层，或者说根结点到每个叶子结点的长度都相同**



**B树的优势**

- **B树的高度远远小于AVL树和红黑树(B树是一颗“矮胖子”)，磁盘IO次数大大减少**
- **对访问局部性原理的利用**。指当一个数据被使用时，其附近的数据有较大概率在短时间内被使用。当访问其中某个数据时，数据库会将该整个节点读到缓存中；当它临近的数据紧接着被访问时，可以直接在缓存中读取，无需进行磁盘IO



**案例分析**

如下图（B树的内部节点可以存放数据，类似ZK的中间节点一样。B树不是每个节点都有足够多的子节点）：

![BalanceTree](png/Java/数据结构-BalanceTree.png)

上图是一颗阶数为4的B树。在实际应用中的B树的阶数m都非常大（通常大于100），所以即使存储大量的数据，B树的高度仍然比较小。每个结点中存储了关键字（key）和关键字对应的数据（data），以及孩子结点的指针。**我们将一个key和其对应的data称为一个记录**。**但为了方便描述，除非特别说明，后续文中就用key来代替（key, value）键值对这个整体**。在数据库中我们将B树（和B+树）作为索引结构，可以加快查询速速，此时B树中的key就表示键，而data表示了这个键对应的条目在硬盘上的逻辑地址。



## B+树(B+Tree)

![B+tree图解](png/Java/数据结构-B+tree图解.png)

**B+树是从B树的变体**。跟B树的不同：

- **内部节点不保存数据，只用于索引**
- **B+树的每个叶子节点之间存在指针相连，而且是单链表**，叶子节点本身依关键字的大小自小而大顺序链接



**案例分析**

如下图，其实B+树上二叉搜索树的扩展，二叉搜索树是每次一分为二，B树是每次一分为多，现代操作系统中，磁盘的存储结构使用的是B+树机制，mysql的innodb引擎的存储方式也是B+树机制：

![B+Tree](png/Java/数据结构-B+Tree.png)

**B+树与B树相比有以下优势**

- **更少的IO次数**：B+树的非叶节点只包含键，而不包含真实数据，因此每个节点存储的记录个数比B数多很多（即阶m更大），因此B+树的高度更低，访问时所需要的IO次数更少。此外，由于每个节点存储的记录数更多，所以对访问局部性原理的利用更好，缓存命中率更高
- **更适于范围查询**：在B树中进行范围查询时，首先找到要查找的下限，然后对B树进行中序遍历，直到找到查找的上限；而B+树的范围查询，只需要对链表进行遍历即可
- **更稳定的查询效率**：B树的查询时间复杂度在1到树高之间(分别对应记录在根节点和叶节点)，而B+树的查询复杂度则稳定为树高，因为所有数据都在叶节点。



**B+树劣势**

由于键会重复出现，因此**会占用更多的空间**。但是与带来的性能优势相比，空间劣势往往可以接受，因此B+树的在数据库中的使用比B树更加广泛。 



## B*树

是B+树的变体，**在B+树的非根和非叶子结点再增加指向兄弟的指针**，且**定义了非叶子结点关键字个数至少为(2/3)×M**，即块的最低使用率为2/3（代替B+树的1/2）：

![Bx树](png/Java/数据结构-Bx树.jpg)



## 2-3-4树

2-3-4树，它是一种多叉树，它的每个节点最多有四个子节点和三个数据项。

### 2-3-4 树介绍

2-3-4树每个节点最多有四个字节点和三个数据项，名字中 2,3,4 的数字含义是指一个节点可能含有的子节点的个数。对于非叶节点有三种可能的情况：

　　①、有一个数据项的节点总是有两个子节点；

　　②、有二个数据项的节点总是有三个子节点；

　　③、有三个数据项的节点总是有四个子节点；

　　简而言之，非叶节点的子节点数总是比它含有的数据项多1。如果子节点个数为L，数据项个数为D，那么：L = D + 1

　　叶节点（上图最下面的一排）是没有子节点的，然而它可能含有一个、两个或三个数据项。空节点是不会存在的。

　　树结构中很重要的一点就是节点之间关键字值大小的关系。在二叉树中，所有关键字值比某个节点值小的节点都在这个节点左子节点为根的子树上；所有关键字值比某个节点值大的节点都在这个节点右子节点为根的子树上。2-3-4 树规则也是一样，并且还加上以下几点：

　　为了方便描述，用从0到2的数字给数据项编号，用0到3的数字给子节点编号，如下图：

　　①、根是child0的子树的所有子节点的关键字值小于key0；

　　②、根是child1的子树的所有子节点的关键字值大于key0并且小于key1；

　　③、根是child2的子树的所有子节点的关键字值大于key1并且小于key2；

　　④、根是child3的子树的所有子节点的关键字值大于key2。

　　简化关系如下图，由于2-3-4树中一般不允许出现重复关键值，所以不用考虑比较关键值相同的情况。

### 搜索2-3-4树

查找特定关键字值的数据项和在二叉树中的搜索类似。从根节点开始搜索，除非查找的关键字值就是根，否则选择关键字值所在的合适范围，转向那个方向，直到找到为止。

　　比如对于下面这幅图，我们需要查找关键字值为 64 的数据项。

　　首先从根节点开始，根节点只有一个数据项50，没有找到，而且因为64比50大，那么转到根节点的子节点child1。60|70|80 也没有找到，而且60<64<70，所以我们还是找该节点的child1,62|64|66，我们发现其第二个数据项正好是64，于是找到了。

### 插入

　新的数据项一般要插在叶节点里，在树的最底层。如果你插入到有子节点的节点里，那么子节点的编号就要发生变化来维持树的结构，因为在2-3-4树中节点的子节点要比数据项多1。

　　插入操作有时比较简单，有时却很复杂。

　　①、当插入没有满数据项的节点时是很简单的，找到合适的位置，只需要把新数据项插入就可以了，插入可能会涉及到在一个节点中移动一个或其他两个数据项，这样在新的数据项插入后关键字值仍保持正确的顺序。如下图：

　　②、如果往下寻找插入位置的途中，节点已经满了，那么插入就变得复杂了。发生这种情况时，节点必须分裂，分裂能保证2-3-4树的平衡。

　　ps：这里讨论的是自顶向下的2-3-4树，因为是在向下找到插入点的路途中节点发生了分裂。把要分裂的数据项设为A,B,C，下面是节点分裂的情况（假设分裂的节点不是根节点）：

　　1、节点分裂

　　一、创建一个新的空节点，它是要分裂节点的兄弟，在要分裂节点的右边；

　　二、数据项C移到新节点中；

　　三、数据项B移到要分裂节点的父节点中；

　　四、数据项A保留在原来的位置；

　　五、最右边的两个子节点从要分裂处断开，连到新节点上。

　　上图描述了节点分裂的例子，另一种描述节点分裂的说法是4-节点变成了两个 2- 节点。节点分裂是把数据向上和向右移动，从而保持了数的平衡。一般插入只需要分裂一个节点，除非插入路径上存在不止一个满节点时，这种情况就需要多重分裂。

　根的分裂

　　如果一开始查找插入节点时就碰到满的根节点，那么插入过程更复杂：

　　①、创建新的根节点，它是要分裂节点的父节点。

　　②、创建第二个新的节点，它是要分裂节点的兄弟节点；

　　③、数据项C移到新的兄弟节点中；

　　④、数据项B移到新的根节点中；

　　⑤、数据项A保留在原来的位置；

　　⑥、要分裂节点最右边的两个子节点断开连接，连到新的兄弟节点中。

　　上图便是根分裂的情况，分裂完成之后，整个树的高度加1。另外一种描述根分裂的方法是说4-节点变成三个2-节点。

　　**注意：插入时，碰到没有满的节点时，要继续向下寻找其子节点进行插入。如果直接插入该节点，那么还要进行子节点的增加，因为在2-3-4树中节点的子节点个数要比数据项多1；如果插入的节点满了，那么就要进行节点分裂。下图是一系列插入过程，有4个节点分裂了，两个是根，两个是叶节点：**

### 完整源码实现

分为节点类Node,表示每个节点的数据项类DataItem,以及最后的2-3-4树类Tree234.class

```
 
public class Tree234 {
    private Node root = new Node() ;
    /*public Tree234(){
        root = new Node();
    }*/
    //查找关键字值
    public int find(long key){
        Node curNode = root;
        int childNumber ;
        while(true){
            if((childNumber = curNode.findItem(key))!=-1){
                return childNumber;
            }else if(curNode.isLeaf()){//节点是叶节点
                return -1;
            }else{
                curNode = getNextChild(curNode,key);
            }
        }
    }
     
    public Node getNextChild(Node theNode,long theValue){
        int j;
        int numItems = theNode.getNumItems();
        for(j = 0 ; j < numItems ; j++){
            if(theValue < theNode.getItem(j).dData){
                return theNode.getChild(j);
            }
        }
        return theNode.getChild(j);
    }
     
    //插入数据项
    public void insert(long dValue){
        Node curNode = root;
        DataItem tempItem = new DataItem(dValue);
        while(true){
            if(curNode.isFull()){//如果节点满数据项了，则分裂节点
                split(curNode);
                curNode = curNode.getParent();
                curNode = getNextChild(curNode, dValue);
            }else if(curNode.isLeaf()){//当前节点是叶节点
                break;
            }else{
                curNode = getNextChild(curNode, dValue);
            }
        }//end while
        curNode.insertItem(tempItem);
    }
     
    public void split(Node thisNode){
        DataItem itemB,itemC;
        Node parent,child2,child3;
        int itemIndex;
        itemC = thisNode.removeItem();
        itemB = thisNode.removeItem();
        child2 = thisNode.disconnectChild(2);
        child3 = thisNode.disconnectChild(3);
        Node newRight = new Node();
        if(thisNode == root){//如果当前节点是根节点，执行根分裂
            root = new Node();
            parent = root;
            root.connectChild(0, thisNode);
        }else{
            parent = thisNode.getParent();
        }
        //处理父节点
        itemIndex = parent.insertItem(itemB);
        int n = parent.getNumItems();
        for(int j = n-1; j > itemIndex ; j--){
            Node temp = parent.disconnectChild(j);
            parent.connectChild(j+1, temp);
        }
        parent.connectChild(itemIndex+1, newRight);
         
        //处理新建的右节点
        newRight.insertItem(itemC);
        newRight.connectChild(0, child2);
        newRight.connectChild(1, child3);
    }
     
    //打印树节点
    public void displayTree(){
        recDisplayTree(root,0,0);
    }
    private void recDisplayTree(Node thisNode,int level,int childNumber){
        System.out.println("levle="+level+" child="+childNumber+" ");
        thisNode.displayNode();
        int numItems = thisNode.getNumItems();
        for(int j = 0; j < numItems+1 ; j++){
            Node nextNode = thisNode.getChild(j);
            if(nextNode != null){
                recDisplayTree(nextNode, level+1, j);
            }else{
                return;
            }
        }
    }
 
    //数据项
    class DataItem{
        public long dData;
        public DataItem(long dData){
            this.dData = dData;
        }
        public void displayItem(){
            System.out.println("/"+dData);
        }
    }
     
    //节点
    class Node{
        private static final int ORDER = 4;
        private int numItems;//表示该节点有多少个数据项
        private Node parent;//父节点
        private Node childArray[] = new Node[ORDER];//存储子节点的数组，最多有4个子节点
        private DataItem itemArray[] = new DataItem[ORDER-1];//存放数据项的数组，一个节点最多有三个数据项
         
        //连接子节点
        public void connectChild(int childNum,Node child){
            childArray[childNum] = child;
            if(child != null){
                child.parent = this;
            }
        }
        //断开与子节点的连接，并返回该子节点
        public Node disconnectChild(int childNum){
            Node tempNode = childArray[childNum];
            childArray[childNum] = null;
            return tempNode;
        }
        //得到节点的某个子节点
        public Node getChild(int childNum){
            return childArray[childNum];
        }
        //得到父节点
        public Node getParent(){
            return parent;
        }
        //判断是否是叶节点
        public boolean isLeaf(){
            return (childArray[0] == null)?true:false;
        }
        //得到节点数据项的个数
        public int getNumItems(){
            return numItems;
        }
        //得到节点的某个数据项
        public DataItem getItem(int index){
            return itemArray[index];
        }
        //判断节点的数据项是否满了（最多3个）
        public boolean isFull(){
            return (numItems == ORDER-1) ? true:false;
        }
         
        //找到数据项在节点中的位置
        public int findItem(long key){
            for(int j = 0 ; j < ORDER-1 ; j++){
                if(itemArray[j]==null){
                    break;
                }else if(itemArray[j].dData == key){
                    return j;
                }
            }
            return -1;
        }
         
        //将数据项插入到节点
        public int insertItem(DataItem newItem){
            numItems++;
            long newKey = newItem.dData;
            for(int j = ORDER-2 ; j >= 0 ; j--){
                if(itemArray[j] == null){//如果为空，继续向前循环
                    continue;
                }else{
                    long itsKey = itemArray[j].dData;//保存节点某个位置的数据项
                    if(newKey < itsKey){//如果比新插入的数据项大
                        itemArray[j+1] = itemArray[j];//将大数据项向后移动一位
                    }else{
                        itemArray[j+1] = newItem;//如果比新插入的数据项小，则直接插入
                        return j+1;
                    }
                }
            }
            //如果都为空，或者都比待插入的数据项大，则将待插入的数据项放在节点第一个位置
            itemArray[0] = newItem;
            return 0;
        }
        //移除节点的数据项
        public DataItem removeItem(){
            DataItem temp = itemArray[numItems-1];
            itemArray[numItems-1] = null;
            numItems--;
            return temp;
        }
        //打印节点的所有数据项
        public void displayNode(){
            for(int j = 0 ; j < numItems ; j++){
                itemArray[j].displayItem();
            }
            System.out.println("/");
        }
    }
 
     
}
```

### 2-3-4树和红黑树

　2-3-4树是多叉树，而红黑树是二叉树，看上去可能完全不同，但是，在某种意义上它们又是完全相同的，一个可以通过应用一些简单的规则变成另一个，而且使他们保持平衡的操作也是一样，数学上称他们为同构。

　　①、对应规则

　　应用如下三条规则可以将2-3-4树转化为红黑树：

　　一、把2-3-4树中的每个2-节点转化为红-黑树的黑色节点。

　　二、把每个3-节点转化为一个子节点和一个父节点，子节点有两个自己的子节点：W和X或X和Y。父节点有另一个子节点：Y或W。哪个节点变成子节点或父节点都无所谓。子节点涂成红色，父节点涂成黑色。

　　三、把每个4-节点转化为一个父节点和两个子节点。第一个子节点有它自己的子节点W和X；第二个子节点拥有子节点Y和Z。和前面一样，子节点涂成红色，父节点涂成黑色。

　　下图是一颗2-3-4树转化成对应的红-黑树。虚线环绕的子树是由3-节点和4-节点变成的。转化后符合红-黑树的规则，根节点为红色，两个红色节点不会相连，每条从根到叶节点的路径上的黑节点个数是一样的。　　

　　②、操作等价

　　不仅红-黑树的结构与2-3-4树对应，而且两种树操作也一样。2-3-4树用节点分裂保持平衡，红-黑树用颜色变换和旋转保持平衡。

　　![img](https://images2018.cnblogs.com/blog/1120165/201712/1120165-20171226204053651-1868267222.png)

　　上图是4-节点分裂。虚线环绕的部分等价于4-节点。颜色变换之后，40,60节点都为黑色的，50节点是红色的。因此，节点 50 和它的父节点70 对于3-节点，如上图虚线所示。

### 2-3-4 树的效率

分析2-3-4树我们可以和红黑树作比较分析。红-黑树的层数（平衡二叉树）大约是log2(N+1)，而2-3-4树每个节点可以最多有4个数据项，如果节点都是满的，那么高度和log4N。因此在所有节点都满的情况下，2-3-4树的高度大致是红-黑树的一半。不过他们不可能都是满的，所以2-3-4树的高度大致在log2(N+1)和log2(N+1)/2。减少2-3-4树的高度可以使它的查找时间比红-黑树的短一些。

　　但是另一方面，每个节点要查看的数据项就多了，这会增加查找时间。因为节点中用线性搜索来查看数据项，使得查找时间的倍数和M成正比，即每个节点数据项的平均数量。总的查找时间和M*log4N成正比。

## 哈希表

Hash表也称散列表，也有直接译作哈希表，Hash表是一种根据关键字值（key - value）而直接进行访问的数据结构。它基于数组，通过把关键字映射到数组的某个下标来加快查找速度，但是又和数组、链表、树等数据结构不同，在这些数据结构中查找某个关键字，通常要遍历整个数据结构，也就是O(N)的时间级，但是对于哈希表来说，只是O(1)的时间级。

　　注意，这里有个重要的问题就是如何把关键字转换为数组的下标，这个转换的函数称为哈希函数（也称散列函数），转换的过程称为哈希化。

![数据结构-hash_table](png\Java\数据结构-hash_table.png)

### 哈希函数的引入

　　大家都用过字典，字典的优点是我们可以通过前面的目录快速定位到所要查找的单词。如果我们想把一本英文字典的每个单词，从 a 到 zyzzyva(这是牛津字典的最后一个单词)，都写入计算机内存，以便快速读写，那么哈希表是个不错的选择。

　　这里我们将范围缩小点，比如想在内存中存储5000个英文单词。我们可能想到每个单词会占用一个数组单元，那么数组的大小是5000，同时可以用数组下标存取单词，这样设想很完美，但是数组下标和单词怎么建立联系呢？

　　首先我们要建立单词和数字（数组下标）的关系：

　　我们知道 ASCII 是一种编码，其中 a 表示97，b表示98，以此类推，一直到122表示z，而每个单词都是由这26个字母组成，我们可以不用 ASCII 编码那么大的数字，自己设计一套类似 ASCII的编码，比如a表示1，b表示2，依次类推，z表示26，那么表示方法我们就知道了。

　　接下来如何把单个字母的数字组合成代表整个单词的数字呢？

　　**①、把数字相加**

　　首先第一种简单的方法就是把单词的每个字母表示的数字相加，得到的和便是数组的下标。

　　比如单词 cats 转换成数字：

　　cats = 3 + 1 + 20 + 19 = 43

　　那么单词 cats 存储在数组中的下标为43，所有的英文单词都可以用这个办法转换成数组下标。但是这个办法真的可行吗？

　　假设我们约定一个单词最多有 10 个字母，那么字典的最后一个单词为 zzzzzzzzzz ，其转换为数字：

　　zzzzzzzzzz = 26*10 = 260

　　那么我们可以得到单词编码的范围是从1-260。很显然，这个范围是不够存储5000个单词的，那么肯定有一个位置存储了多个单词，每个数组的数据项平均要存储192个单词（5000除以260）。

　　对于上面的问题，我们如何解决呢？

　　**第一种方法：**考虑每个数组项包含一个子数组或者一个子链表，这个办法存数据项确实很快，但是如果我们想要从192个单词中查找到其中一个，那么还是很慢。

　　**第二种方法：**为啥要让那么多单词占据同一个数据项呢？也就是说我们没有把单词分的足够开，数组能表示的元素太少，我们需要扩展数组的下标，使其每个位置都只存放一个单词。

　　对于上面的第二种方法，问题产生了，我们如何扩展数组的下标呢？

　　**②、幂的连乘**

　　我们将单词表示的数拆成数列，用适当的 27 的幂乘以这些位数（因为有26个可能的字符，以及空格，一共27个），然后把乘积相加，这样就得出了每个单词独一无二的数字。

　　比如把单词cats 转换为数字：

　　cats = 3*273 + 1*272 + 20*271 + 19*270 = 59049 + 729 + 540 + 19 = 60337

　　这个过程会为每个单词创建一个独一无二的数，但是注意的是我们这里只是计算了 4 个字母组成的单词，如果单词很长，比如最长的10个字母的单词 zzzzzzzzzz，仅仅是279 结果就超出了7000000000000，这个结果是很巨大的，在实际内存中，根本不可能为一个数组分配这么大的空间。

　　所以这个方案的问题就是虽然为每个单词都分配了独一无二的下标，但是只有一小部分存放了单词，很大一部分都是空着的。那么现在就需要一种方法，把数位幂的连乘系统中得到的巨大的整数范围压缩到可接受的数组范围中。

　　对于英语字典，假设只有5000个单词，这里我们选定容量为10000 的数组空间来存放（后面会介绍为啥需要多出一倍的空间）。那么我们就需要将从 0 到超过 7000000000000 的范围，压缩到从0到10000的范围。

　　第一种方法：取余，得到一个数被另一个整数除后的余数。首先我们假设要把从0-199的数字（用largeNumber表示），压缩为从0-9的数字（用smallNumber表示），后者有10个数，所以变量smallRange 的值为10，这个转换的表达式为：

　　smallNumber = largeNumber % smallRange

　　当一个数被 10 整除时，余数一定在0-9之间，这样，我们就把从0-199的数压缩为从0-9的数，压缩率为 20 :1。

　　![img](https://images2017.cnblogs.com/blog/1120165/201801/1120165-20180106221144471-1865375569.png)

 

　　我们也可以用类似的方法把表示单词唯一的数压缩成数组的下标：

　　arrayIndex = largerNumber % smallRange

　　**这也就是哈希函数。它把一个大范围的数字哈希（转化）成一个小范围的数字，这个小范围的数对应着数组的下标。使用哈希函数向数组插入数据后，这个数组就是哈希表。**

### 冲突

把巨大的数字范围压缩到较小的数字范围，那么肯定会有几个不同的单词哈希化到同一个数组下标，即产生了**冲突**。

　　冲突可能会导致哈希化方案无法实施，前面我们说指定的数组范围大小是实际存储数据的两倍，因此可能有一半的空间是空着的，所以，当冲突产生时，一个方法是通过系统的方法找到数组的一个空位，并把这个单词填入，而不再用哈希函数得到数组的下标，这种方法称为开放地址法。比如加入单词 cats 哈希化的结果为5421，但是它的位置已经被单词parsnip占用了，那么我们会考虑将单词 cats 存放在parsnip后面的一个位置 5422 上。

　　另一种方法，前面我们也提到过，就是数组的每个数据项都创建一个子链表或子数组，那么数组内不直接存放单词，当产生冲突时，新的数据项直接存放到这个数组下标表示的链表中，这种方法称为链地址法。

### 开放地址法

开发地址法中，若数据项不能直接存放在由哈希函数所计算出来的数组下标时，就要寻找其他的位置。分别有三种方法：线性探测、二次探测以及再哈希法。

　　①、线性探测

　　在线性探测中，它会线性的查找空白单元。比如如果 5421 是要插入数据的位置，但是它已经被占用了，那么就使用5422，如果5422也被占用了，那么使用5423，以此类推，数组下标依次递增，直到找到空白的位置。这就叫做线性探测，因为它沿着数组下标一步一步顺序的查找空白单元。

　　完整代码：

```
 
public class MyHashTable {
    private DataItem[] hashArray;   //DataItem类，表示每个数据项信息
    private int arraySize;//数组的初始大小
    private int itemNum;//数组实际存储了多少项数据
    private DataItem nonItem;//用于删除数据项
     
    public MyHashTable(int arraySize){
        this.arraySize = arraySize;
        hashArray = new DataItem[arraySize];
        nonItem = new DataItem(-1);//删除的数据项下标为-1
    }
    //判断数组是否存储满了
    public boolean isFull(){
        return (itemNum == arraySize);
    }
     
    //判断数组是否为空
    public boolean isEmpty(){
        return (itemNum == 0);
    }
     
    //打印数组内容
    public void display(){
        System.out.println("Table:");
        for(int j = 0 ; j < arraySize ; j++){
            if(hashArray[j] != null){
                System.out.print(hashArray[j].getKey() + " ");
            }else{
                System.out.print("** ");
            }
        }
    }
    //通过哈希函数转换得到数组下标
    public int hashFunction(int key){
        return key%arraySize;
    }
     
    //插入数据项
    public void insert(DataItem item){
        if(isFull()){
            //扩展哈希表
            System.out.println("哈希表已满，重新哈希化...");
            extendHashTable();
        }
        int key = item.getKey();
        int hashVal = hashFunction(key);
        while(hashArray[hashVal] != null && hashArray[hashVal].getKey() != -1){
            ++hashVal;
            hashVal %= arraySize;
        }
        hashArray[hashVal] = item;
        itemNum++;
    }
    /**
     * 数组有固定的大小，而且不能扩展，所以扩展哈希表只能另外创建一个更大的数组，然后把旧数组中的数据插到新的数组中。
     * 但是哈希表是根据数组大小计算给定数据的位置的，所以这些数据项不能再放在新数组中和老数组相同的位置上。
     * 因此不能直接拷贝，需要按顺序遍历老数组，并使用insert方法向新数组中插入每个数据项。
     * 这个过程叫做重新哈希化。这是一个耗时的过程，但如果数组要进行扩展，这个过程是必须的。
     */
    public void extendHashTable(){
        int num = arraySize;
        itemNum = 0;//重新计数，因为下面要把原来的数据转移到新的扩张的数组中
        arraySize *= 2;//数组大小翻倍
        DataItem[] oldHashArray = hashArray;
        hashArray = new DataItem[arraySize];
        for(int i = 0 ; i < num ; i++){
            insert(oldHashArray[i]);
        }
    }
     
    //删除数据项
    public DataItem delete(int key){
        if(isEmpty()){
            System.out.println("Hash Table is Empty!");
            return null;
        }
        int hashVal = hashFunction(key);
        while(hashArray[hashVal] != null){
            if(hashArray[hashVal].getKey() == key){
                DataItem temp = hashArray[hashVal];
                hashArray[hashVal] = nonItem;//nonItem表示空Item,其key为-1
                itemNum--;
                return temp;
            }
            ++hashVal;
            hashVal %= arraySize;
        }
        return null;
    }
     
    //查找数据项
    public DataItem find(int key){
        int hashVal = hashFunction(key);
        while(hashArray[hashVal] != null){
            if(hashArray[hashVal].getKey() == key){
                return hashArray[hashVal];
            }
            ++hashVal;
            hashVal %= arraySize;
        }
        return null;
    }
     
    public static class DataItem{
        private int iData;
        public DataItem(int iData){
            this.iData = iData;
        }
        public int getKey(){
            return iData;
        }
    }
 
}
```

　　需要注意的是，当哈希表变得太满时，我们需要扩展数组，但是需要注意的是，数据项不能放到新数组中和老数组相同的位置，而是要根据数组大小重新计算插入位置。这是一个比较耗时的过程，所以一般我们要确定数据的范围，给定好数组的大小，而不再扩容。

　　另外，当哈希表变得比较满时，我们每插入一个新的数据，都要频繁的探测插入位置，因为可能很多位置都被前面插入的数据所占用了，这称为聚集。数组填的越满，聚集越可能发生。

　　这就像人群，当某个人在商场晕倒时，人群就会慢慢聚集。最初的人群聚过来是因为看到了那个倒下的人，而后面聚过来的人是因为它们想知道这些人聚在一起看什么。人群聚集的越大，吸引的人就会越多。

　　②、装填因子

　　已填入哈希表的数据项和表长的比率叫做装填因子，比如有10000个单元的哈希表填入了6667 个数据后，其装填因子为 2/3。当装填因子不太大时，聚集分布的比较连贯，而装填因子比较大时，则聚集发生的很大了。

　　我们知道线性探测是一步一步的往后面探测，当装填因子比较大时，会频繁的产生聚集，那么如果我们探测比较大的单元，而不是一步一步的探测呢，这就是下面要讲的二次探测。

　　③、二次探测

 　二测探测是防止聚集产生的一种方式，思想是探测相距较远的单元，而不是和原始位置相邻的单元。

　　线性探测中，如果哈希函数计算的原始下标是x, 线性探测就是x+1, x+2, x+3, 以此类推；而在二次探测中，探测的过程是x+1, x+4, x+9, x+16，以此类推，到原始位置的距离是步数的平方。二次探测虽然消除了原始的聚集问题，但是产生了另一种更细的聚集问题，叫二次聚集：比如讲184，302，420和544依次插入表中，它们的映射都是7，那么302需要以1为步长探测，420需要以4为步长探测， 544需要以9为步长探测。只要有一项其关键字映射到7，就需要更长步长的探测，这个现象叫做二次聚集。二次聚集不是一个严重的问题，但是二次探测不会经常使用，因为还有好的解决方法，比如再哈希法。

　　④、再哈希法

　　为了消除原始聚集和二次聚集，我们使用另外一种方法：再哈希法。

　　我们知道二次聚集的原因是，二测探测的算法产生的探测序列步长总是固定的：1,4，9,16以此类推。那么我们想到的是需要产生一种依赖关键字的探测序列，而不是每个关键字都一样，那么，不同的关键字即使映射到相同的数组下标，也可以使用不同的探测序列。

　　方法是把关键字用不同的哈希函数再做一遍哈希化，用这个结果作为步长。对于指定的关键字，步长在整个探测中是不变的，不过不同的关键字使用不同的步长。

　　第二个哈希函数必须具备如下特点：

　　一、和第一个哈希函数不同

　　二、不能输出0（否则，将没有步长，每次探测都是原地踏步，算法将陷入死循环）。

　　专家们已经发现下面形式的哈希函数工作的非常好：stepSize = constant - key % constant; 其中constant是质数，且小于数组容量。
　　再哈希法要求表的容量是一个质数，假如表长度为15(0-14)，非质数，有一个特定关键字映射到0，步长为5，则探测序列是0,5,10,0,5,10,以此类推一直循环下去。算法只尝试这三个单元，所以不可能找到某些空白单元，最终算法导致崩溃。如果数组容量为13, 质数，探测序列最终会访问所有单元。即0,5,10,2,7,12,4,9,1,6,11,3,一直下去，只要表中有一个空位，就可以探测到它。

　　完整再哈希法代码：

```
 
public class HashDouble {
    private DataItem[] hashArray;   //DataItem类，表示每个数据项信息
    private int arraySize;//数组的初始大小
    private int itemNum;//数组实际存储了多少项数据
    private DataItem nonItem;//用于删除数据项
     
    public HashDouble(){
        this.arraySize = 13;
        hashArray = new DataItem[arraySize];
        nonItem = new DataItem(-1);//删除的数据项下标为-1
    }
    //判断数组是否存储满了
    public boolean isFull(){
        return (itemNum == arraySize);
    }
     
    //判断数组是否为空
    public boolean isEmpty(){
        return (itemNum == 0);
    }
     
    //打印数组内容
    public void display(){
        System.out.println("Table:");
        for(int j = 0 ; j < arraySize ; j++){
            if(hashArray[j] != null){
                System.out.print(hashArray[j].getKey() + " ");
            }else{
                System.out.print("** ");
            }
        }
    }
    //通过哈希函数转换得到数组下标
    public int hashFunction1(int key){
        return key%arraySize;
    }
     
    public int hashFunction2(int key){
        return 5 - key%5;
    }
     
    //插入数据项
    public void insert(DataItem item){
        if(isFull()){
            //扩展哈希表
            System.out.println("哈希表已满，重新哈希化...");
            extendHashTable();
        }
        int key = item.getKey();
        int hashVal = hashFunction1(key);
        int stepSize = hashFunction2(key);//用第二个哈希函数计算探测步数
        while(hashArray[hashVal] != null && hashArray[hashVal].getKey() != -1){
            hashVal += stepSize;
            hashVal %= arraySize;//以指定的步数向后探测
        }
        hashArray[hashVal] = item;
        itemNum++;
    }
 
    /**
     * 数组有固定的大小，而且不能扩展，所以扩展哈希表只能另外创建一个更大的数组，然后把旧数组中的数据插到新的数组中。
     * 但是哈希表是根据数组大小计算给定数据的位置的，所以这些数据项不能再放在新数组中和老数组相同的位置上。
     * 因此不能直接拷贝，需要按顺序遍历老数组，并使用insert方法向新数组中插入每个数据项。
     * 这个过程叫做重新哈希化。这是一个耗时的过程，但如果数组要进行扩展，这个过程是必须的。
     */
    public void extendHashTable(){
        int num = arraySize;
        itemNum = 0;//重新计数，因为下面要把原来的数据转移到新的扩张的数组中
        arraySize *= 2;//数组大小翻倍
        DataItem[] oldHashArray = hashArray;
        hashArray = new DataItem[arraySize];
        for(int i = 0 ; i < num ; i++){
            insert(oldHashArray[i]);
        }
    }
     
    //删除数据项
    public DataItem delete(int key){
        if(isEmpty()){
            System.out.println("Hash Table is Empty!");
            return null;
        }
        int hashVal = hashFunction1(key);
        int stepSize = hashFunction2(key);
        while(hashArray[hashVal] != null){
            if(hashArray[hashVal].getKey() == key){
                DataItem temp = hashArray[hashVal];
                hashArray[hashVal] = nonItem;//nonItem表示空Item,其key为-1
                itemNum--;
                return temp;
            }
            hashVal += stepSize;
            hashVal %= arraySize;
        }
        return null;
    }
     
    //查找数据项
    public DataItem find(int key){
        int hashVal = hashFunction1(key);
        int stepSize = hashFunction2(key);
        while(hashArray[hashVal] != null){
            if(hashArray[hashVal].getKey() == key){
                return hashArray[hashVal];
            }
            hashVal += stepSize;
            hashVal %= arraySize;
        }
        return null;
    }
    public static class DataItem{
        private int iData;
        public DataItem(int iData){
            this.iData = iData;
        }
        public int getKey(){
            return iData;
        }
    }
}
```

### 链地址法

在开放地址法中，通过再哈希法寻找一个空位解决冲突问题，另一个方法是在哈希表每个单元中设置链表（即链地址法），某个数据项的关键字值还是像通常一样映射到哈希表的单元，而数据项本身插入到这个单元的链表中。其他同样映射到这个位置的数据项只需要加到链表中，不需要在原始的数组中寻找空位。

　　有序链表：

```
 
public class SortLink {
    private LinkNode first;
    public SortLink(){
        first = null;
    }
    public boolean isEmpty(){
        return (first == null);
    }
    public void insert(LinkNode node){
        int key = node.getKey();
        LinkNode previous = null;
        LinkNode current = first;
        while(current != null && current.getKey() < key){
            previous = current;
            current = current.next;
        }
        if(previous == null){
            first = node;
        }else{
            previous.next = node;
        }
　　　　　　node.next = curent;
    }
    public void delete(int key){
        LinkNode previous = null;
        LinkNode current = first;
        if(isEmpty()){
            System.out.println("Linked is Empty!!!");
            return;
        }
        while(current != null && current.getKey() != key){
            previous = current;
            current = current.next;
        }
        if(previous == null){
            first = first.next;
        }else{
            previous.next = current.next;
        }
    }
     
    public LinkNode find(int key){
        LinkNode current = first;
        while(current != null && current.getKey() <= key){
            if(current.getKey() == key){
                return current;
            }
                        current = current.next;
        }
        return null;
    }
     
    public void displayLink(){
        System.out.println("Link(First->Last)");
        LinkNode current = first;
        while(current != null){
            current.displayLink();
            current = current.next;
        }
        System.out.println("");
    }
    class LinkNode{
        private int iData;
        public LinkNode next;
        public LinkNode(int iData){
            this.iData = iData;
        }
        public int getKey(){
            return iData;
        }
        public void displayLink(){
            System.out.println(iData + " ");
        }
    }
}
```

链地址法：

```
 
import com.ys.hash.SortLink.LinkNode;
 
public class HashChain {
    private SortLink[] hashArray;//数组中存放链表
    private int arraySize;
    public HashChain(int size){
        arraySize = size;
        hashArray = new SortLink[arraySize];
        //new 出每个空链表初始化数组
        for(int i = 0 ; i < arraySize ; i++){
            hashArray[i] = new SortLink();
        }
    }
     
    public void displayTable(){
        for(int i = 0 ; i < arraySize ; i++){
            System.out.print(i + "：");
            hashArray[i].displayLink();
        }
    }
     
    public int hashFunction(int key){
        return key%arraySize;
    }
     
    public void insert(LinkNode node){
        int key = node.getKey();
        int hashVal = hashFunction(key);
        hashArray[hashVal].insert(node);//直接往链表中添加即可
    }
     
    public LinkNode delete(int key){
        int hashVal = hashFunction(key);
        LinkNode temp = find(key);
        hashArray[hashVal].delete(key);//从链表中找到要删除的数据项，直接删除
        return temp;
    }
     
    public LinkNode find(int key){
        int hashVal = hashFunction(key);
        LinkNode node = hashArray[hashVal].find(key);
        return node;
    }
 
}
```

　链地址法中，装填因子（数据项数和哈希表容量的比值）与开放地址法不同，在链地址法中，需要有N个单元的数组中转入N个或更多的数据项，因此装填因子一般为1，或比1大（有可能某些位置包含的链表中包含两个或两个以上的数据项）。

　　找到初始单元需要O(1)的时间级别，而搜索链表的时间与M成正比，M为链表包含的平均项数，即O(M)的时间级别。

### 桶

另外一种方法类似于链地址法，它是在每个数据项中使用子数组，而不是链表。这样的数组称为桶。

　　这个方法显然不如链表有效，因为桶的容量不好选择，如果容量太小，可能会溢出，如果太大，又造成性能浪费，而链表是动态分配的，不存在此问题。所以一般不使用桶。

哈希表基于数组，类似于key-value的存储形式，关键字值通过哈希函数映射为数组的下标，如果一个关键字哈希化到已占用的数组单元，这种情况称为冲突。用来解决冲突的有两种方法：开放地址法和链地址法。在开发地址法中，把冲突的数据项放在数组的其它位置；在链地址法中，每个单元都包含一个链表，把所有映射到同一数组下标的数据项都插入到这个链表中。

## 堆

优先级队列是一种抽象数据类型（ADT），它提供了删除最大（或最小）关键字值的数据项的方法，插入数据项的方法，优先级队列可以用有序数组来实现，这种实现方式尽管删除最大数据项的时间复杂度为O(1)，但是插入还是需要较长的时间 O(N)，因为每次插入平均需要移动一半的数据项，来保证插入后，数组依旧有序。

　　本篇博客我们介绍另外一种数据结构——堆，注意这里的堆和我们Java语言，C++语言等编程语言在内存中的“堆”是不一样的，这里的堆是一种树，由它实现的优先级队列的插入和删除的时间复杂度都为O(logN)，这样尽管删除的时间变慢了，但是插入的时间快了很多，当速度非常重要，而且有很多插入操作时，可以选择用堆来实现优先级队列。

### 堆的定义

　①、它是完全二叉树，除了树的最后一层节点不需要是满的，其它的每一层从左到右都是满的。注意下面两种情况，第二种最后一层从左到右中间有断隔，那么也是不完全二叉树。

　　②、它通常用数组来实现。

　　这种用数组实现的二叉树，假设节点的索引值为index，那么：

　　节点的左子节点是 2*index+1，

　　节点的右子节点是 2*index+2，

　　节点的父节点是 （index-1）/2。

　　③、堆中的每一个节点的关键字都大于（或等于）这个节点的子节点的关键字。

　　这里要注意堆和前面说的二叉搜索树的区别，二叉搜索树中所有节点的左子节点关键字都小于右子节点关键字，在二叉搜索树中通过一个简单的算法就可以按序遍历节点。但是在堆中，按序遍历节点是很困难的，如上图所示，堆只有沿着从根节点到叶子节点的每一条路径是降序排列的，指定节点的左边节点或者右边节点，以及上层节点或者下层节点由于不在同一条路径上，他们的关键字可能比指定节点大或者小。所以相对于二叉搜索树，堆是弱序的。

### 遍历和查找

前面我们说了，堆是弱序的，所以想要遍历堆是很困难的，基本上，堆是不支持遍历的。

　　对于查找，由于堆的特性，在查找的过程中，没有足够的信息来决定选择通过节点的两个子节点中的哪一个来选择走向下一层，所以也很难在堆中查找到某个关键字。

　　因此，堆这种组织似乎非常接近无序，不过，对于快速的移除最大（或最小）节点，也就是根节点，以及能快速插入新的节点，这两个操作就足够了。

### 移除

　　移除是指删除关键字最大的节点（或最小），也就是根节点。

　　根节点在数组中的索引总是0，即maxNode = heapArray[0];

　　移除根节点之后，那树就空了一个根节点，也就是数组有了一个空的数据单元，这个空单元我们必须填上。

　　第一种方法：将数组所有数据项都向前移动一个单元，这比较费时。

　　第二种方法：

　　　　①、移走根

　　　　②、把最后一个节点移动到根的位置

　　　　③、一直向下筛选这个节点，直到它在一个大于它的节点之下，小于它的节点之上为止。

　　具体步骤如下：

　　图a表示把最后一个节点移到根节点，图b、c、d表示将节点向下筛选到合适的位置，它的合适位置在最底层（有时候可能在中间），图e表示节点在正确位置的情景。

　　注意：向下筛选的时候，将目标节点和其子节点比较，谁大就和谁交换位置。

### 插入

　插入节点也很容易，插入时，选择向上筛选，节点初始时插入到数组最后第一个空着的单元，数组容量大小增一。然后进行向上筛选的算法。

　　注意：向上筛选和向下不同，向上筛选只用和一个父节点进行比较，比父节点小就停止筛选了。

　

### 完整的Java堆代码

　　首先我们要知道用数组表示堆的一些要点。若数组中节点的索引为x，则：

　　节点的左子节点是 2*index+1，

　　节点的右子节点是 2*index+2，

　　节点的父节点是 （index-1）/2。

　　注意："/" 这个符号，应用于整数的算式时，它执行整除，且得到是是向下取整的值。

```
 
public class Heap {
     
    private Node[] heapArray;
    private int maxSize;
    private int currentSize;
     
    public Heap(int mx) {
        maxSize = mx;
        currentSize = 0;
        heapArray = new Node[maxSize];
    }
     
    public boolean isEmpty() {
        return (currentSize == 0)? true : false;
    }
     
    public boolean isFull() {
        return (currentSize == maxSize)? true : false;
    }
     
    public boolean insert(int key) {
        if(isFull()) {
            return false;
        }
        Node newNode = new Node(key);
        heapArray[currentSize] = newNode;
        trickleUp(currentSize++);
        return true;
    }
    //向上调整
    public void trickleUp(int index) {
        int parent = (index - 1) / 2; //父节点的索引
        Node bottom = heapArray[index]; //将新加的尾节点存在bottom中
        while(index > 0 && heapArray[parent].getKey() < bottom.getKey()) {
            heapArray[index] = heapArray[parent];
            index = parent;
            parent = (parent - 1) / 2;
        }
        heapArray[index] = bottom;
    }
     
    public Node remove() {
        Node root = heapArray[0];
        heapArray[0] = heapArray[--currentSize];
        trickleDown(0);
        return root;
    }
    //向下调整
    public void trickleDown(int index) {
        Node top = heapArray[index];
        int largeChildIndex;
        while(index < currentSize/2) { //while node has at least one child
            int leftChildIndex = 2 * index + 1;
            int rightChildIndex = leftChildIndex + 1;
            //find larger child
            if(rightChildIndex < currentSize &&  //rightChild exists?
                    heapArray[leftChildIndex].getKey() < heapArray[rightChildIndex].getKey()) {
                largeChildIndex = rightChildIndex;
            }
            else {
                largeChildIndex = leftChildIndex;
            }
            if(top.getKey() >= heapArray[largeChildIndex].getKey()) {
                break;
            }
            heapArray[index] = heapArray[largeChildIndex];
            index = largeChildIndex;
        }
        heapArray[index] = top;
    }
    //根据索引改变堆中某个数据
    public boolean change(int index, int newValue) {
        if(index < 0 || index >= currentSize) {
            return false;
        }
        int oldValue = heapArray[index].getKey();
        heapArray[index].setKey(newValue);
        if(oldValue < newValue) {
            trickleUp(index);
        }
        else {
            trickleDown(index);
        }
        return true;
    }
     
    public void displayHeap() {
        System.out.println("heapArray(array format): ");
        for(int i = 0; i < currentSize; i++) {
            if(heapArray[i] != null) {
                System.out.print(heapArray[i].getKey() + " ");
            }
            else {
                System.out.print("--");
            }
        }
    }
}
class Node {
    private int iData;
    public Node(int key) {
        iData = key;
    }
     
    public int getKey() {
        return iData;
    }
     
    public void setKey(int key) {
        iData = key;
    }
}
```

## 图(Graph)

图也是计算机程序设计中最常用的数据结构之一，从数学意义上讲，树是图的一种，大家可以对比着学习。

### 图的定义

![数据结构-graph](png\Java\数据结构-graph.png)

**图(graph)**由多个**节点(vertex)**构成，节点之间阔以互相连接组成一个网络。(x, y)表示一条**边(edge)**，它表示节点 x 与 y 相连。边可能会有**权值(weight/cost)**。

#### 邻接：

　　如果两个顶点被同一条边连接，就称这两个顶点是邻接的，如上图 I 和 G 就是邻接的，而 I 和 F 就不是。有时候也将和某个指定顶点邻接的顶点叫做它的邻居，比如顶点 G 的邻居是 I、H、F。

#### 路径：

　　路径是边的序列，比如从顶点B到顶点J的路径为 BAEJ，当然还有别的路径 BCDJ，BACDJ等等。

#### 连通图和非连通图：

　　如果至少有一条路径可以连接起所有的顶点，那么这个图称作连通的；如果假如存在从某个顶点不能到达另外一个顶点，则称为非联通的。

#### 有向图和无向图：

　　如果图中的边没有方向，可以从任意一边到达另一边，则称为无向图；比如双向高速公路，A城市到B城市可以开车从A驶向B，也可以开车从B城市驶向A城市。但是如果只能从A城市驶向B城市的图，那么则称为有向图。

#### 有权图和无权图：

　　图中的边被赋予一个权值，权值是一个数字，它能代表两个顶点间的物理距离，或者从一个顶点到另一个顶点的时间，这种图被称为有权图；反之边没有赋值的则称为无权图。

　　本篇博客我们讨论的是无权无向图。

### 在程序中表示图

　我们知道图是由顶点和边组成，那么在计算机中，怎么来模拟顶点和边？

#### 顶点：

　　在大多数情况下，顶点表示某个真实世界的对象，这个对象必须用数据项来描述。比如在一个飞机航线模拟程序中，顶点表示城市，那么它需要存储城市的名字、海拔高度、地理位置和其它相关信息，因此通常用一个顶点类的对象来表示一个顶点，这里我们仅仅在顶点中存储了一个字母来标识顶点，同时还有一个标志位，用来判断该顶点有没有被访问过（用于后面的搜索）。

```
/**
 * 顶点类
 * @author vae
 */
public class Vertex {
    public char label;
    public boolean wasVisited;
     
    public Vertex(char label){
        this.label = label;
        wasVisited = false;
    }
}
```

　　顶点对象能放在数组中，然后用下标指示，也可以放在链表或其它数据结构中，不论使用什么结构，存储只是为了使用方便，这与边如何连接点是没有关系的。

#### 边：

　　在前面讲解各种树的数据结构时，大多数树都是每个节点包含它的子节点的引用，比如红黑树、二叉树。也有用数组表示树，树组中节点的位置决定了它和其它节点的关系，比如堆就是用数组表示。

　　然而图并不像树，图没有固定的结构，图的每个顶点可以与任意多个顶点相连，为了模拟这种自由形式的组织结构，用如下两种方式表示图：邻接矩阵和邻接表（如果一条边连接两个顶点，那么这两个顶点就是邻接的）

　　**邻接矩阵：**

　　邻接矩阵是一个二维数组，数据项表示两点间是否存在边，如果图中有 N 个顶点，邻接矩阵就是 N*N 的数组。上图用邻接矩阵表示如下：

　　![img](https://images2017.cnblogs.com/blog/1120165/201802/1120165-20180209221722904-1549384558.png)

　　1表示有边，0表示没有边，也可以用布尔变量true和false来表示。顶点与自身相连用 0 表示，所以这个矩阵从左上角到右上角的对角线全是 0 。

　　注意：这个矩阵的上三角是下三角的镜像，两个三角包含了相同的信息，这个冗余信息看似低效，但是在大多数计算机中，创造一个三角形数组比较困难，所以只好接受这个冗余，这也要求在程序处理中，当我们增加一条边时，比如更新邻接矩阵的两部分，而不是一部分。

　　**邻接表：**

　　邻接表是一个链表数组（或者是链表的链表），每个单独的链表表示了有哪些顶点与当前顶点邻接。

　　![img](https://images2017.cnblogs.com/blog/1120165/201802/1120165-20180209222328123-1963427649.png)　　

### 搜索

　　在图中实现最基本的操作之一就是搜索从一个指定顶点可以到达哪些顶点，比如从武汉出发的高铁可以到达哪些城市，一些城市可以直达，一些城市不能直达。现在有一份全国高铁模拟图，要从某个城市（顶点）开始，沿着铁轨（边）移动到其他城市（顶点），有两种方法可以用来搜索图：深度优先搜索（DFS）和广度优先搜索（BFS）。它们最终都会到达所有连通的顶点，深度优先搜索通过栈来实现，而广度优先搜索通过队列来实现，不同的实现机制导致不同的搜索方式。

#### 深度优先搜索（DFS）

　　深度优先搜索算法有如下规则：

　　规则1：如果可能，访问一个邻接的未访问顶点，标记它，并将它放入栈中。

　　规则2：当不能执行规则 1 时，如果栈不为空，就从栈中弹出一个顶点。

　　规则3：如果不能执行规则 1 和规则 2 时，就完成了整个搜索过程。

　　![img](https://images2017.cnblogs.com/blog/1120165/201802/1120165-20180209225344107-1667286888.png)

　　对于上图，应用深度优先搜索如下：假设选取 A 顶点为起始点，并且按照字母优先顺序进行访问，那么应用规则 1 ，接下来访问顶点 B，然后标记它，并将它放入栈中；再次应用规则 1，接下来访问顶点 F，再次应用规则 1，访问顶点 H。我们这时候发现，没有 H 顶点的邻接点了，这时候应用规则 2，从栈中弹出 H，这时候回到了顶点 F，但是我们发现 F 也除了 H 也没有与之邻接且未访问的顶点了，那么再弹出 F，这时候回到顶点 B，同理规则 1 应用不了，应用规则 2，弹出 B，这时候栈中只有顶点 A了，然后 A 还有未访问的邻接点，所有接下来访问顶点 C，但是 C又是这条线的终点，所以从栈中弹出它，再次回到 A，接着访问 D,G,I，最后也回到了 A，然后访问 E，但是最后又回到了顶点 A，这时候我们发现 A没有未访问的邻接点了，所以也把它弹出栈。现在栈中已无顶点，于是应用规则 3，完成了整个搜索过程。

　　深度优先搜索在于能够找到与某一顶点邻接且没有访问过的顶点。这里以邻接矩阵为例，找到顶点所在的行，从第一列开始向后寻找值为1的列；列号是邻接顶点的号码，检查这个顶点是否未访问过，如果是这样，那么这就是要访问的下一个顶点，如果该行没有顶点既等于1（邻接）且又是未访问的，那么与指定点相邻接的顶点就全部访问过了（后面会用算法实现）。

#### 广度优先搜索（BFS）

　　深度优先搜索要尽可能的远离起始点，而广度优先搜索则要尽可能的靠近起始点，它首先访问起始顶点的所有邻接点，然后再访问较远的区域，这种搜索不能用栈实现，而是用队列实现。

　　规则1：访问下一个未访问的邻接点（如果存在），这个顶点必须是当前顶点的邻接点，标记它，并把它插入到队列中。

　　规则2：如果已经没有未访问的邻接点而不能执行规则 1 时，那么从队列列头取出一个顶点（如果存在），并使其成为当前顶点。

　　规则3：如果因为队列为空而不能执行规则 2，则搜索结束。

　　对于上面的图，应用广度优先搜索：以A为起始点，首先访问所有与 A 相邻的顶点，并在访问的同时将其插入队列中，现在已经访问了 A,B,C,D和E。这时队列（从头到尾）包含 BCDE，已经没有未访问的且与顶点 A 邻接的顶点了，所以从队列中取出B，寻找与B邻接的顶点，这时找到F，所以把F插入到队列中。已经没有未访问且与B邻接的顶点了，所以从队列列头取出C，它没有未访问的邻接点。因此取出 D 并访问 G，D也没有未访问的邻接点了，所以取出E，现在队列中有 FG，在取出 F，访问 H，然后取出 G，访问 I，现在队列中有 HI，当取出他们时，发现没有其它为访问的顶点了，这时队列为空，搜索结束。

#### 程序实现

实现深度优先搜索的栈 StackX.class

```
 
public class StackX {
    private final int SIZE = 20;
    private int[] st;
    private int top;
     
    public StackX(){
        st = new int[SIZE];
        top = -1;
    }
     
    public void push(int j){
        st[++top] = j;
    }
     
    public int pop(){
        return st[top--];
    }
     
    public int peek(){
        return st[top];
    }
     
    public boolean isEmpty(){
        return (top == -1);
    }
 
}
```

实现广度优先搜索的队列Queue.class

```
 
public class Queue {
    private final int SIZE = 20;
    private int[] queArray;
    private int front;
    private int rear;
     
    public Queue(){
        queArray = new int[SIZE];
        front = 0;
        rear = -1;
    }
     
    public void insert(int j) {
        if(rear == SIZE-1) {
            rear = -1;
        }
        queArray[++rear] = j;
    }
     
    public int remove() {
        int temp = queArray[front++];
        if(front == SIZE) {
            front = 0;
        }
        return temp;
    }
     
    public boolean isEmpty() {
        return (rear+1 == front || front+SIZE-1 == rear);
    }
}
```

图代码 Graph.class

```
 
public class Graph {
    private final int MAX_VERTS = 20;//表示顶点的个数
    private Vertex vertexList[];//用来存储顶点的数组
    private int adjMat[][];//用邻接矩阵来存储 边,数组元素0表示没有边界，1表示有边界
    private int nVerts;//顶点个数
    private StackX theStack;//用栈实现深度优先搜索
    private Queue queue;//用队列实现广度优先搜索
    /**
     * 顶点类
     * @author vae
     */
    class Vertex {
        public char label;
        public boolean wasVisited;
         
        public Vertex(char label){
            this.label = label;
            wasVisited = false;
        }
    }
     
    public Graph(){
        vertexList = new Vertex[MAX_VERTS];
        adjMat = new int[MAX_VERTS][MAX_VERTS];
        nVerts = 0;//初始化顶点个数为0
        //初始化邻接矩阵所有元素都为0，即所有顶点都没有边
        for(int i = 0; i < MAX_VERTS; i++) {
            for(int j = 0; j < MAX_VERTS; j++) {
                adjMat[i][j] = 0;
            }
        }
        theStack = new StackX();
        queue = new Queue();
    }
     
    //将顶点添加到数组中，是否访问标志置为wasVisited=false(未访问)
    public void addVertex(char lab) {
        vertexList[nVerts++] = new Vertex(lab);
    }
     
    //注意用邻接矩阵表示边，是对称的，两部分都要赋值
    public void addEdge(int start, int end) {
        adjMat[start][end] = 1;
        adjMat[end][start] = 1;
    }
     
    //打印某个顶点表示的值
    public void displayVertex(int v) {
        System.out.print(vertexList[v].label);
    }
    /**深度优先搜索算法:
     * 1、用peek()方法检查栈顶的顶点
     * 2、用getAdjUnvisitedVertex()方法找到当前栈顶点邻接且未被访问的顶点
     * 3、第二步方法返回值不等于-1则找到下一个未访问的邻接顶点，访问这个顶点，并入栈
     *    如果第二步方法返回值等于 -1，则没有找到，出栈
     */
    public void depthFirstSearch() {
        //从第一个顶点开始访问
        vertexList[0].wasVisited = true; //访问之后标记为true
        displayVertex(0);//打印访问的第一个顶点
        theStack.push(0);//将第一个顶点放入栈中
         
        while(!theStack.isEmpty()) {
            //找到栈当前顶点邻接且未被访问的顶点
            int v = getAdjUnvisitedVertex(theStack.peek());
            if(v == -1) {   //如果当前顶点值为-1，则表示没有邻接且未被访问顶点，那么出栈顶点
                theStack.pop();
            }else { //否则访问下一个邻接顶点
                vertexList[v].wasVisited = true;
                displayVertex(v);
                theStack.push(v);
            }
        }
         
        //栈访问完毕，重置所有标记位wasVisited=false
        for(int i = 0; i < nVerts; i++) {
            vertexList[i].wasVisited = false;
        }
    }
     
    //找到与某一顶点邻接且未被访问的顶点
    public int getAdjUnvisitedVertex(int v) {
        for(int i = 0; i < nVerts; i++) {
            //v顶点与i顶点相邻（邻接矩阵值为1）且未被访问 wasVisited==false
            if(adjMat[v][i] == 1 && vertexList[i].wasVisited == false) {
                return i;
            }
        }
        return -1;
    }
     
    /**
     * 广度优先搜索算法：
     * 1、用remove()方法检查栈顶的顶点
     * 2、试图找到这个顶点还未访问的邻节点
     * 3、 如果没有找到，该顶点出列
     * 4、 如果找到这样的顶点，访问这个顶点，并把它放入队列中
     */
    public void breadthFirstSearch(){
        vertexList[0].wasVisited = true;
        displayVertex(0);
        queue.insert(0);
        int v2;
         
        while(!queue.isEmpty()) {
            int v1 = queue.remove();
            while((v2 = getAdjUnvisitedVertex(v1)) != -1) {
                vertexList[v2].wasVisited = true;
                displayVertex(v2);
                queue.insert(v2);
            }
        }
         
        //搜索完毕，初始化，以便于下次搜索
        for(int i = 0; i < nVerts; i++) {
            vertexList[i].wasVisited = false;
        }
    }
     
    public static void main(String[] args) {
        Graph graph = new Graph();
        graph.addVertex('A');
        graph.addVertex('B');
        graph.addVertex('C');
        graph.addVertex('D');
        graph.addVertex('E');
         
        graph.addEdge(0, 1);//AB
        graph.addEdge(1, 2);//BC
        graph.addEdge(0, 3);//AD
        graph.addEdge(3, 4);//DE
         
        System.out.println("深度优先搜索算法 :");
        graph.depthFirstSearch();//ABCDE
         
        System.out.println();
        System.out.println("----------------------");
         
        System.out.println("广度优先搜索算法 :");
        graph.breadthFirstSearch();//ABDCE
    }
}
```

#### 最小生成树

　对于图的操作，还有一个最常用的就是找到最小生成树，最小生成树就是用最少的边连接所有顶点。对于给定的一组顶点，可能有很多种最小生成树，但是最小生成树的边的数量 E 总是比顶点 V 的数量小1，即：

　　V = E + 1

　　这里不用关心边的长度，不是找最短的路径（会在带权图中讲解），而是找最少数量的边，可以基于深度优先搜索和广度优先搜索来实现。

　　比如基于深度优先搜索，我们记录走过的边，就可以创建一个最小生成树。因为DFS 访问所有顶点，但只访问一次，它绝对不会两次访问同一个顶点，但她看到某条边将到达一个已访问的顶点，它就不会走这条边，它从来不遍历那些不可能的边，因此，DFS 算法走过整个图的路径必定是最小生成树。

```
//基于深度优先搜索找到最小生成树
public void mst(){
    vertexList[0].wasVisited = true;
    theStack.push(0);
     
    while(!theStack.isEmpty()){
        int currentVertex = theStack.peek();
        int v = getAdjUnvisitedVertex(currentVertex);
        if(v == -1){
            theStack.pop();
        }else{
            vertexList[v].wasVisited = true;
            theStack.push(v);
             
            displayVertex(currentVertex);
            displayVertex(v);
            System.out.print(" ");
        }
    }
     
    //搜索完毕，初始化，以便于下次搜索
    for(int i = 0; i < nVerts; i++) {
        vertexList[i].wasVisited = false;
    }
}
```

图是由边连接的顶点组成，图可以表示许多真实的世界情况，包括飞机航线、电子线路等等。搜索算法以一种系统的方式访问图中的每个顶点，主要通过深度优先搜索（DFS）和广度优先搜索（BFS），深度优先搜索通过栈来实现，广度优先搜索通过队列来实现。最后需要知道最小生成树是包含连接图中所有顶点所需要的最少数量的边。

## 前缀树（Trie）

假如有一个字典，字典里面有如下词："A"，"to"，"tea"，"ted"，"ten"，"i"，"in"，"inn"，每个单词还能有自己的一些权重值，那么用前缀树来构建这个字典将会是如下的样子：

![前缀树](F:/lemon-guide-main/images/Algorithm/前缀树.png)

**性质**

- 每个节点至少包含两个基本属性

  - children：数组或者集合，罗列出每个分支当中包含的所有字符
  - isEnd：布尔值，表示该节点是否为某字符串的结尾

- 前缀树的根节点是空的

  所谓空，即只利用到这个节点的 children 属性，即只关心在这个字典里，有哪些打头的字符

- 除了根节点，其他所有节点都有可能是单词的结尾，叶子节点一定都是单词的结尾



**实现**

- 创建

  - 遍历一遍输入的字符串，对每个字符串的字符进行遍历
  - 从前缀树的根节点开始，将每个字符加入到节点的 children 字符集当中
  - 如果字符集已经包含了这个字符，则跳过
  - 如果当前字符是字符串的最后一个，则把当前节点的 isEnd 标记为真。

  由上，创建的方法很直观。前缀树真正强大的地方在于，每个节点还能用来保存额外的信息，比如可以用来记录拥有相同前缀的所有字符串。因此，当用户输入某个前缀时，就能在 O(1) 的时间内给出对应的推荐字符串。

- 搜索

  与创建方法类似，从前缀树的根节点出发，逐个匹配输入的前缀字符，如果遇到了就继续往下一层搜索，如果没遇到，就立即返回。

## 线段树（Segment Tree）

假设有一个数组 array[0 … n-1]， 里面有 n 个元素，现在要经常对这个数组做两件事。

- 更新数组元素的数值
- 求数组任意一段区间里元素的总和（或者平均值）



**解法 1：遍历一遍数组**

- 时间复杂度 O(n)。

**解法 2：线段树**

- 线段树，就是一种按照二叉树的形式存储数据的结构，每个节点保存的都是数组里某一段的总和。
- 适用于数据很多，而且需要频繁更新并求和的操作。
- 时间复杂度 O(logn)。



**实现**
如数组是 [1, 3, 5, 7, 9, 11]，那么它的线段树如下。

![线段树](F:/lemon-guide-main/images/Algorithm/线段树.png)

根节点保存的是从下标 0 到下标 5 的所有元素的总和，即 36。左右两个子节点分别保存左右两半元素的总和。按照这样的逻辑不断地切分下去，最终的叶子节点保存的就是每个元素的数值。



## 树状数组（Fenwick Tree）

树状数组（Fenwick Tree / Binary Indexed Tree）。

**举例**：假设有一个数组 array[0 … n-1]， 里面有 n 个元素，现在要经常对这个数组做两件事。

- 更新数组元素的数值
- 求数组前 k 个元素的总和（或者平均值）

 

**解法 1：线段树**

- 线段树能在 O(logn) 的时间里更新和求解前 k 个元素的总和

**解法 2：树状数**

- 该问题只要求求解前 k 个元素的总和，并不要求任意一个区间
- 树状数组可以在 O(logn) 的时间里完成上述的操作
- 相对于线段树的实现，树状数组显得更简单



**树状数组特点**

- 它是利用数组来表示多叉树的结构，在这一点上和优先队列有些类似，只不过，优先队列是用数组来表示完全二叉树，而树状数组是多叉树。
- 树状数组的第一个元素是空节点。
- 如果节点 tree[y] 是 tree[x] 的父节点，那么需要满足条件：y = x - (x & (-x))。

# 算法

　　**算法简单来说就是解决问题的步骤。**

　　在Java中，算法通常都是由类的方法来实现的。前面的数据结构，比如链表为啥插入、删除快，而查找慢，平衡的二叉树插入、删除、查找都快，这都是实现这些数据结构的算法所造成的。后面我们讲的各种排序实现也是算法范畴的重要领域。

### 算法的五个特征

　　①、**有穷性**：对于任意一组合法输入值，在执行又穷步骤之后一定能结束，即：算法中的每个步骤都能在有限时间内完成。

　　②、**确定性**：在每种情况下所应执行的操作，在算法中都有确切的规定，使算法的执行者或阅读者都能明确其含义及如何执行。并且在任何条件下，算法都只有一条执行路径。

　　③、**可行性**：算法中的所有操作都必须足够基本，都可以通过已经实现的基本操作运算有限次实现之。

　　④、**有输入**：作为算法加工对象的量值，通常体现在算法当中的一组变量。有些输入量需要在算法执行的过程中输入，而有的算法表面上可以没有输入，实际上已被嵌入算法之中。

　　⑤、**有输出**：它是一组与“输入”有确定关系的量值，是算法进行信息加工后得到的结果，这种确定关系即为算法功能。

### 算法的设计原则

　　①、**正确性**：首先，算法应当满足以特定的“规则说明”方式给出的需求。其次，对算法是否“正确”的理解可以有以下四个层次：

　　　　　　　　一、程序语法错误。

　　　　　　　　二、程序对于几组输入数据能够得出满足需要的结果。

　　　　　　　　三、程序对于精心选择的、典型、苛刻切带有刁难性的几组输入数据能够得出满足要求的结果。

　　　　　　　　四、程序对于一切合法的输入数据都能得到满足要求的结果。

　　　　　　　　PS：通常以第 三 层意义的正确性作为衡量一个算法是否合格的标准。

　　**②、可读性**：算法为了人的阅读与交流，其次才是计算机执行。因此算法应该易于人的理解；另一方面，晦涩难懂的程序易于隐藏较多的错误而难以调试。

　　**③、健壮性**：当输入的数据非法时，算法应当恰当的做出反应或进行相应处理，而不是产生莫名其妙的输出结果。并且，处理出错的方法不应是中断程序执行，而是应当返回一个表示错误或错误性质的值，以便在更高的抽象层次上进行处理。

　　**④、高效率与低存储量需求**：通常算法效率值得是算法执行时间；存储量是指算法执行过程中所需要的最大存储空间，两者都与问题的规模有关。

　　前面三点 正确性，可读性和健壮性相信都好理解。对于第四点算法的执行效率和存储量，我们知道比较算法的时候，可能会说“A算法比B算法快两倍”之类的话，但实际上这种说法没有任何意义。因为当数据项个数发生变化时，A算法和B算法的效率比例也会发生变化，比如数据项增加了50%，可能A算法比B算法快三倍，但是如果数据项减少了50%，可能A算法和B算法速度一样。所以描述算法的速度必须要和数据项的个数联系起来。也就是“大O”表示法，它是一种算法复杂度的相对表示方式，这里我简单介绍一下，后面会根据具体的算法来描述。

　　相对(relative)：你只能比较相同的事物。你不能把一个做算数乘法的算法和排序整数列表的算法进行比较。但是，比较2个算法所做的算术操作（一个做乘法，一个做加法）将会告诉你一些有意义的东西；

　　表示(representation)：大O(用它最简单的形式)把算法间的比较简化为了一个单一变量。这个变量的选择基于观察或假设。例如，排序算法之间的对比通常是基于比较操作(比较2个结点来决定这2个结点的相对顺序)。这里面就假设了比较操作的计算开销很大。但是，如果比较操作的计算开销不大，而交换操作的计算开销很大，又会怎么样呢？这就改变了先前的比较方式；

### 复杂度

　　复杂度(complexity)：如果排序10,000个元素花费了我1秒，那么排序1百万个元素会花多少时间？在这个例子里，复杂度就是相对其他东西的度量结果。

　　然后我们在说说算法的存储量，包括：

　　程序本身所占空间；

　　输入数据所占空间；

　　辅助变量所占空间；

　　一个算法的效率越高越好，而存储量是越低越好。

![SortAlgorithm](png\Java\SortAlgorithm.png)

**相关概念**

- **稳定**：如果a原本在b前面，而a=b，排序之后a仍然在b的前面
- **不稳定**：如果a原本在b的前面，而a=b，排序之后 a 可能会出现在 b 的后面
- **时间复杂度**：对排序数据的总的操作次数。反映当n变化时，操作次数呈现什么规律
- **空间复杂度：**是指算法在计算机内执行时所需存储空间的度量，它也是数据规模n的函数



**时间复杂度与时间效率**：O(1) < O(log2N) < O(n) < O(N \* log2N) < O(N2) < O(N3) < 2N < 3N < N!

一般来说，前四个效率比较高，中间两个差强人意，后三个比较差（只要N比较大，这个算法就动不了了）。



#### 常数阶

```java
int sum = 0,n = 100; //执行一次  
sum = (1+n)*n/2; //执行一次  
System.out.println (sum); //执行一次
```

上面算法的运行的次数的函数为 f(n)=3，根据推导大 O 阶的规则 1，我们需要将常数 3 改为 1，则这个算法的时间复杂度为 O(1)。如果 sum=(1+n)*n/2 这条语句再执行 10 遍，因为这与问题大小 n 的值并没有关系，所以这个算法的时间复杂度仍旧是 O(1)，我们可以称之为常数阶。



#### 线性阶

线性阶主要要分析循环结构的运行情况，如下所示：

```java
for (int i = 0; i < n; i++) {
    //时间复杂度为O(1)的算法
    ...
}
```

上面算法循环体中的代码执行了n次，因此时间复杂度为O(n)。



#### 对数阶

接着看如下代码：

```java
int number = 1;
while (number < n) {
    number = number * 2;
    //时间复杂度为O(1)的算法
    ...
}
```

可以看出上面的代码，随着 number 每次乘以 2 后，都会越来越接近 n，当 number 不小于 n 时就会退出循环。假设循环的次数为 X，则由 2^x=n 得出 x=log₂n，因此得出这个算法的时间复杂度为 O(logn)。



#### 平方阶

下面的代码是循环嵌套：

```java
for (int i = 0; i < n; i++) {   	
    for(int j = 0; j < n; i++) {     
        //复杂度为O(1)的算法			... 
    }}
```

内层循环的时间复杂度在讲到线性阶时就已经得知是O(n)，现在经过外层循环n次，那么这段算法的时间复杂度则为O(n²)。

## 算法思想

![算法思想](png\Java\算法思想.jpg)

### 分治(Divide and Conquer)

分治算法思想很大程度上是基于递归的，也比较适合用递归来实现。顾名思义，分而治之。一般分为以下三个过程：

- **分解**：将原问题分解成一系列子问题
- **解决**：递归求解各个子问题，若子问题足够小，则直接求解
- **合并**：将子问题的结果合并成原问题

比较经典的应用就是**归并排序 (Merge Sort)** 以及**快速排序 (Quick Sort)** 等。我们来从归并排序理解分治思想，归并排序就是将待排序数组不断二分为规模更小的子问题处理，再将处理好的子问题合并起来。

**基本概念**

  在计算机科学中，分治法是一种很重要的算法。字面上的解释是“分而治之”，就是把一个复杂的问题分成两个或更多的相同或相似的子问题，再把子问题分成更小的子问题……直到最后子问题可以简单的直接求解，原问题的解即子问题的解的合并。这个技巧是很多高效算法的基础，如排序算法(快速排序，归并排序)，傅立叶变换(快速傅立叶变换)……

  任何一个可以用计算机求解的问题所需的计算时间都与其规模有关。问题的规模越小，越容易直接求解，解题所需的计算时间也越少。例如，对于n个元素的排序问题，当n=1时，不需任何计算。n=2时，只要作一次比较即可排好序。n=3时只要作3次比较即可，…。而当n较大时，问题就不那么容易处理了。要想直接解决一个规模较大的问题，有时是相当困难的。

**基本思想及策略**

  分治法的设计思想是：将一个难以直接解决的大问题，分割成一些规模较小的相同问题，以便各个击破，分而治之。

  分治策略是：对于一个规模为n的问题，若该问题可以容易地解决（比如说规模n较小）则直接解决，否则将其分解为k个规模较小的子问题，这些子问题互相独立且与原问题形式相同，递归地解这些子问题，然后将各子问题的解合并得到原问题的解。这种算法设计策略叫做分治法。

  如果原问题可分割成k个子问题，1<k≤n，且这些子问题都可解并可利用这些子问题的解求出原问题的解，那么这种分治法就是可行的。由分治法产生的子问题往往是原问题的较小模式，这就为使用递归技术提供了方便。在这种情况下，反复应用分治手段，可以使子问题与原问题类型一致而其规模却不断缩小，最终使子问题缩小到很容易直接求出其解。这自然导致递归过程的产生。分治与递归像一对孪生兄弟，经常同时应用在算法设计之中，并由此产生许多高效算法。

**分治法适用的情况**

分治法所能解决的问题一般具有以下几个特征：

1) 该问题的规模缩小到一定的程度就可以容易地解决

2) 该问题可以分解为若干个规模较小的相同问题，即该问题具有最优子结构性质。

3) 利用该问题分解出的子问题的解可以合并为该问题的解；

4) 该问题所分解出的各个子问题是相互独立的，即子问题之间不包含公共的子子问题。

第一条特征是绝大多数问题都可以满足的，因为问题的计算复杂性一般是随着问题规模的增加而增加；

**第二条特征是应用分治法的前提**它也是大多数问题可以满足的，此特征反映了递归思想的应用；、

**第三条特征是关键，能否利用分治法完全取决于问题是否具有第三条特征**，如果**具备了第一条和第二条特征，而不具备第三条特征，则可以考虑用贪心法或动态规划法**。

**第四条特征涉及到分治法的效率**，如果各子问题是不独立的则分治法要做许多不必要的工作，重复地解公共的子问题，此时虽然可用分治法，但**一般用动态规划法较好**。

**分治法的基本步骤**

分治法在每一层递归上都有三个步骤：

   step1 分解：将原问题分解为若干个规模较小，相互独立，与原问题形式相同的子问题；

   step2 解决：若子问题规模较小而容易被解决则直接解，否则递归地解各个子问题

   step3 合并：将各个子问题的解合并为原问题的解。

它的一般的算法设计模式如下：

  Divide-and-Conquer(P)

1. if |P|≤n0

2. then return(ADHOC(P))

3. 将P分解为较小的子问题 P1 ,P2 ,...,Pk

4. for i←1 to k

5. do yi ← Divide-and-Conquer(Pi) △ 递归解决Pi

6. T ← MERGE(y1,y2,...,yk) △ 合并子问题

7. return(T)

  其中|P|表示问题P的规模；n0为一阈值，表示当问题P的规模不超过n0时，问题已容易直接解出，不必再继续分解。ADHOC(P)是该分治法中的基本子算法，用于直接解小规模的问题P。因此，当P的规模不超过n0时直接用算法ADHOC(P)求解。算法MERGE(y1,y2,...,yk)是该分治法中的合并子算法，用于将P的子问题P1 ,P2 ,...,Pk的相应的解y1,y2,...,yk合并为P的解。

**分治法的复杂性分析**

  一个分治法将规模为n的问题分成k个规模为n／m的子问题去解。设分解阀值n0=1，且adhoc解规模为1的问题耗费1个单位时间。再设将原问题分解为k个子问题以及用merge将k个子问题的解合并为原问题的解需用f(n)个单位时间。用T(n)表示该分治法解规模为|P|=n的问题所需的计算时间，则有：

 T（n）= k T(n/m)+f(n)

  通过迭代法求得方程的解：

   递归方程及其解只给出n等于m的方幂时T(n)的值，但是如果认为T(n)足够平滑，那么由n等于m的方幂时T(n)的值可以估计T(n)的增长速度。通常假定T(n)是单调上升的，从而当          mi≤n<mi+1时，T(mi)≤T(n)<T(mi+1)。

**可使用分治法求解的一些经典问题**

 （1）二分搜索

（2）大整数乘法

 （3）Strassen矩阵乘法

（4）棋盘覆盖

（5）合并排序

（6）快速排序

（7）线性时间选择

（8）最接近点对问题

（9）循环赛日程表

（10）汉诺塔

**依据分治法设计程序时的思维过程**

   实际上就是类似于数学归纳法，找到解决本问题的求解方程公式，然后根据方程公式设计递归程序。

1、一定是先找到最小问题规模时的求解方法

2、然后考虑随着问题规模增大时的求解方法

3、找到求解的递归函数式后（各种规模或因子），设计递归程序即可。



### 贪心(Greedy)

**贪心算法**是动态规划算法的一个子集，可以更高效解决一部分更特殊的问题。实际上，用贪心算法解决问题的思路，并不总能给出最优解。因为它在每一步的决策中，选择目前最优策略，不考虑全局是不是最优。

**贪心算法+双指针求解**

- 给一个孩子的饼干应当尽量小并且能满足孩子，大的留来满足胃口大的孩子
- 因为胃口小的孩子最容易得到满足，所以优先满足胃口小的孩子需求
- 按照从小到大的顺序使用饼干尝试是否可满足某个孩子
- 当饼干 j >= 胃口 i 时，饼干满足胃口，更新满足的孩子数并移动指针 `i++ j++ res++`
- 当饼干 j < 胃口 i 时，饼干不能满足胃口，需要换大的 `j++`



### 回溯(Backtracking)

使用回溯法进行求解，回溯是一种通过穷举所有可能情况来找到所有解的算法。如果一个候选解最后被发现并不是可行解，回溯算法会舍弃它，并在前面的一些步骤做出一些修改，并重新尝试找到可行解。究其本质，其实就是枚举。

- 如果没有更多的数字需要被输入，说明当前的组合已经产生
- 如果还有数字需要被输入：
  - 遍历下一个数字所对应的所有映射的字母
  - 将当前的字母添加到组合最后，也就是 `str + tmp[r]`



### 动态规划(Dynamic Programming)

虽然动态规划的最终版本 (降维再去维) 大都不是递归，但解题的过程还是离开不递归的。新手可能会觉得动态规划思想接受起来比较难，确实，动态规划求解问题的过程不太符合人类常规的思维方式，我们需要切换成机器思维。使用动态规划思想解题，首先要明确动态规划的三要素。动态规划三要素：

- `重叠子问题`：切换机器思维，自底向上思考
- `最优子结构`：子问题的最优解能够推出原问题的优解
- `状态转移方程`：dp[n] = dp[n-1] + dp[n-2]

## 算法模板

### 递归模板

```java
public void recur(int level, int param) {
    // terminator 
    if (level > MAX_LEVEL) {
        // process result    
        return;
    }

    // process current logic  
    process(level, param);

    // drill down   
    recur(level + 1, newParam);

    // restore current status 
}
```

**List转树形结构**

- **方案一：两层循环实现建树**
- **方案二：使用递归方法建树**

```java
public class TreeNode {
    private String id;
    private String parentId;
    private String name;
    private List<TreeNode> children;

    /**
     * 方案一：两层循环实现建树
     *
     * @param treeNodes 传入的树节点列表
     * @return
     */
    public static List<TreeNode> bulid(List<TreeNode> treeNodes) {
        List<TreeNode> trees = new ArrayList<>();
        for (TreeNode treeNode : treeNodes) {
            if ("0".equals(treeNode.getParentId())) {
                trees.add(treeNode);
            }

            for (TreeNode it : treeNodes) {
                if (it.getParentId() == treeNode.getId()) {
                    if (treeNode.getChildren() == null) {
                        treeNode.setChildren(new ArrayList<TreeNode>());
                    }
                    treeNode.getChildren().add(it);
                }
            }
        }

        return trees;
    }

    /**
     * 方案二：使用递归方法建树
     *
     * @param treeNodes
     * @return
     */
    public static List<TreeNode> buildByRecursive(List<TreeNode> treeNodes) {
        List<TreeNode> trees = new ArrayList<>();
        for (TreeNode treeNode : treeNodes) {
            if ("0".equals(treeNode.getParentId())) {
                trees.add(findChildren(treeNode, treeNodes));
            }
        }

        return trees;
    }

    /**
     * 递归查找子节点
     *
     * @param treeNodes
     * @return
     */
    private static TreeNode findChildren(TreeNode treeNode, List<TreeNode> treeNodes) {
        for (TreeNode it : treeNodes) {
            if (treeNode.getId().equals(it.getParentId())) {
                if (treeNode.getChildren() == null) {
                    treeNode.setChildren(new ArrayList<>());
                }
                treeNode.getChildren().add(findChildren(it, treeNodes));
            }
        }

        return treeNode;
    }
}
```



### 回溯模板



### DFS模板



### BFS模板



## 查找算法

### 顺序查找

就是一个一个依次查找。



### 二分查找

二分查找又叫折半查找，从有序列表的初始候选区`li[0:n]`开始，通过对待查找的值与候选区中间值的比较，可以使候选区减少一半。如果待查值小于候选区中间值，则只需比较中间值左边的元素，减半查找范围。依次类推依次减半。

- 二分查找的前提：**列表有序**
- 二分查找的有点：**查找速度快**
- 二分查找的时间复杂度为：**O(logn)**

![二分查找](png/Java/算法-二分查找.gif)

JAVA代码如下：

```java
/**
 * 执行递归二分查找，返回第一次出现该值的位置
 *
 * @param array     已排序的数组
 * @param start     开始位置，如：0
 * @param end       结束位置，如：array.length-1
 * @param findValue 需要找的值
 * @return 值在数组中的位置，从0开始。找不到返回-1
 */
public static int searchRecursive(int[] array, int start, int end, int findValue) {
        // 如果数组为空，直接返回-1，即查找失败
        if (array == null) {
            return -1;
        }
  
        if (start <= end) {
            // 中间位置
            int middle = (start + end) / 1;
            // 中值
            int middleValue = array[middle];
            if (findValue == middleValue) {
                // 等于中值直接返回
                return middle;
            } else if (findValue < middleValue) {
                // 小于中值时在中值前面找
                return searchRecursive(array, start, middle - 1, findValue);
            } else {
                // 大于中值在中值后面找
                return searchRecursive(array, middle + 1, end, findValue);
            }
        } else {
            // 返回-1，即查找失败
            return -1;
        }
}

/**
 * 循环二分查找，返回第一次出现该值的位置
 *
 * @param array     已排序的数组
 * @param findValue 需要找的值
 * @return 值在数组中的位置，从0开始。找不到返回-1
 */
public static int searchLoop(int[] array, int findValue) {
        // 如果数组为空，直接返回-1，即查找失败
        if (array == null) {
            return -1;
        }

        // 起始位置
        int start = 0;
        // 结束位置
        int end = array.length - 1;
        while (start <= end) {
            // 中间位置
            int middle = (start + end) / 2;
            // 中值
            int middleValue = array[middle];
            if (findValue == middleValue) {
                // 等于中值直接返回
                return middle;
            } else if (findValue < middleValue) {
                // 小于中值时在中值前面找
                end = middle - 1;
            } else {
                // 大于中值在中值后面找
                start = middle + 1;
            }
        }

        // 返回-1，即查找失败
        return -1;
}
```



### 插值查找

插值查找算法类似于二分查找，不同的是插值查找每次从自适应mid处开始查找。将折半查找中的求mid索引的公式，low表示左边索引left，high表示右边索引right，key就是前面我们讲的findVal。

![二分查找mid](png/Java/算法-二分查找mid.png)      **改为**   ![插值查找mid](F:/lemon-guide-main/images/Algorithm/插值查找mid.png)  

**注意事项**

- 对于数据量较大，关键字分布比较均匀的查找表来说，采用插值查找，速度较快
- 关键字分布不均匀的情况下，该方法不一定比折半查找要好

```java
/**
 * 插值查找
 *
 * @param arr 已排序的数组
 * @param left 开始位置，如：0
 * @param right 结束位置，如：array.length-1
 * @param findValue
 * @return
 */
public static int insertValueSearch(int[] arr, int left, int right, int findValue) {
        //注意：findVal < arr[0]  和  findVal > arr[arr.length - 1] 必须需要, 否则我们得到的 mid 可能越界
        if (left > right || findValue < arr[0] || findValue > arr[arr.length - 1]) {
            return -1;
        }

        // 求出mid, 自适应
        int mid = left + (right - left) * (findValue - arr[left]) / (arr[right] - arr[left]);
        int midValue = arr[mid];
        if (findValue > midValue) {
            // 向右递归
            return insertValueSearch(arr, mid + 1, right, findValue);
        } else if (findValue < midValue) {
            // 向左递归
            return insertValueSearch(arr, left, mid - 1, findValue);
        } else {
            return mid;
        }
}
```



### 斐波那契查找

黄金分割点是指把一条线段分割为两部分，使其中一部分与全长之比等于另一部分与这部分之比。取其前三位数字的近似值是0.618。由于按此比例设计的造型十分美丽，因此称为黄金分割，也称为中外比。这是一个神奇的数字，会带来意向不大的效果。斐波那契数列{1, 1,2, 3, 5, 8, 13,21, 34, 55 }发现斐波那契数列的两个相邻数的比例，无限接近黄金分割值0.618。
斐波那契查找原理与前两种相似，仅仅改变了中间结点(mid)的位置，mid不再是中间或插值得到，而是位于黄金分割点附近，即mid=low+F(k-1)-1(F代表斐波那契数列)，如下图所示：

![斐波那契查找](png/Java/算法-斐波那契查找.png)

JAVA代码如下：

```java
/**
 * 因为后面我们mid=low+F(k-1)-1，需要使用到斐波那契数列，因此我们需要先获取到一个斐波那契数列
 * <p>
 * 非递归方法得到一个斐波那契数列
 *
 * @return
 */
private static int[] getFibonacci() {
        int[] fibonacci = new int[20];
        fibonacci[0] = 1;
        fibonacci[1] = 1;
        for (int i = 2; i < fibonacci.length; i++) {
            fibonacci[i] = fibonacci[i - 1] + fibonacci[i - 2];
        }
        return fibonacci;
}

/**
 * 编写斐波那契查找算法
 * <p>
 * 使用非递归的方式编写算法
 *
 * @param arr       数组
 * @param findValue 我们需要查找的关键码(值)
 * @return 返回对应的下标，如果没有-1
 */
public static int fibonacciSearch(int[] arr, int findValue) {
        int low = 0;
        int high = arr.length - 1;
        int k = 0;// 表示斐波那契分割数值的下标
        int mid = 0;// 存放mid值
        int[] fibonacci = getFibonacci();// 获取到斐波那契数列
        // 获取到斐波那契分割数值的下标
        while (high > fibonacci[k] - 1) {
            k++;
        }

        // 因为 fibonacci[k] 值可能大于 arr 的 长度，因此我们需要使用Arrays类，构造一个新的数组
        int[] temp = Arrays.copyOf(arr, fibonacci[k]);
        // 实际上需求使用arr数组最后的数填充 temp
        for (int i = high + 1; i < temp.length; i++) {
            temp[i] = arr[high];
        }

        // 使用while来循环处理，找到我们的数 findValue
        while (low <= high) {
            mid = low + fibonacci[k] - 1;
            if (findValue < temp[mid]) {
                high = mid - 1;
                k--;
            } else if (findValue > temp[mid]) {
                low = mid + 1;
                k++;
            } else {
                return Math.min(mid, high);
            }
        }

        return -1;
}
```



## 搜索算法

### 深度优先搜索(DFS)

深度优先搜索（Depth-First Search / DFS）是一种**优先遍历子节点**而不是回溯的算法。

![深度优先搜索](png/Java/算法-深度优先搜索.jpg)

**DFS解决的是连通性的问题**。即给定两个点，一个是起始点，一个是终点，判断是不是有一条路径能从起点连接到终点。起点和终点，也可以指的是某种起始状态和最终的状态。问题的要求并不在乎路径是长还是短，只在乎有还是没有。



**代码案例**

```java
/**
 * Depth-First Search(DFS)
 * <p>
 * 从根节点出发，沿着左子树方向进行纵向遍历，直到找到叶子节点为止。然后回溯到前一个节点，进行右子树节点的遍历，直到遍历完所有可达节点为止。
 * <p>
 * 数据结构：栈
 * 父节点入栈，父节点出栈，先右子节点入栈，后左子节点入栈。递归遍历全部节点即可
 *
 * @author lry
 */
public class DepthFirstSearch {

    /**
     * 树节点
     *
     * @param <V>
     */
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    public static class TreeNode<V> {
        private V value;
        private List<TreeNode<V>> childList;

        // 二叉树节点支持如下

        public TreeNode<V> getLeft() {
            if (childList == null || childList.isEmpty()) {
                return null;
            }

            return childList.get(0);

        }

        public TreeNode<V> getRight() {
            if (childList == null || childList.isEmpty()) {
                return null;
            }

            return childList.get(1);
        }
    }

    /**
     * 模型：
     * .......A
     * ...../   \
     * ....B     C
     * .../ \   /  \
     * ..D   E F    G
     * ./ \        /  \
     * H  I       J    K
     */
    public static void main(String[] args) {
        TreeNode<String> treeNodeA = new TreeNode<>("A", new ArrayList<>());
        TreeNode<String> treeNodeB = new TreeNode<>("B", new ArrayList<>());
        TreeNode<String> treeNodeC = new TreeNode<>("C", new ArrayList<>());
        TreeNode<String> treeNodeD = new TreeNode<>("D", new ArrayList<>());
        TreeNode<String> treeNodeE = new TreeNode<>("E", new ArrayList<>());
        TreeNode<String> treeNodeF = new TreeNode<>("F", new ArrayList<>());
        TreeNode<String> treeNodeG = new TreeNode<>("G", new ArrayList<>());
        TreeNode<String> treeNodeH = new TreeNode<>("H", new ArrayList<>());
        TreeNode<String> treeNodeI = new TreeNode<>("I", new ArrayList<>());
        TreeNode<String> treeNodeJ = new TreeNode<>("J", new ArrayList<>());
        TreeNode<String> treeNodeK = new TreeNode<>("K", new ArrayList<>());
        // A->B,C
        treeNodeA.getChildList().add(treeNodeB);
        treeNodeA.getChildList().add(treeNodeC);
        // B->D,E
        treeNodeB.getChildList().add(treeNodeD);
        treeNodeB.getChildList().add(treeNodeE);
        // C->F,G
        treeNodeC.getChildList().add(treeNodeF);
        treeNodeC.getChildList().add(treeNodeG);
        // D->H,I
        treeNodeD.getChildList().add(treeNodeH);
        treeNodeD.getChildList().add(treeNodeI);
        // G->J,K
        treeNodeG.getChildList().add(treeNodeJ);
        treeNodeG.getChildList().add(treeNodeK);

        System.out.println("非递归方式");
        dfsNotRecursive(treeNodeA);
        System.out.println();
        System.out.println("前续遍历");
        dfsPreOrderTraversal(treeNodeA, 0);
        System.out.println();
        System.out.println("后续遍历");
        dfsPostOrderTraversal(treeNodeA, 0);
        System.out.println();
        System.out.println("中续遍历");
        dfsInOrderTraversal(treeNodeA, 0);
    }

    /**
     * 非递归方式
     *
     * @param tree
     * @param <V>
     */
    public static <V> void dfsNotRecursive(TreeNode<V> tree) {
        if (tree != null) {
            // 次数之所以用 Map 只是为了保存节点的深度，如果没有这个需求可以改为 Stack<TreeNode<V>>
            Stack<Map<TreeNode<V>, Integer>> stack = new Stack<>();
            Map<TreeNode<V>, Integer> root = new HashMap<>();
            root.put(tree, 0);
            stack.push(root);

            while (!stack.isEmpty()) {
                Map<TreeNode<V>, Integer> item = stack.pop();
                TreeNode<V> node = item.keySet().iterator().next();
                int depth = item.get(node);

                // 打印节点值以及深度
                System.out.print("-->[" + node.getValue().toString() + "," + depth + "]");

                if (node.getChildList() != null && !node.getChildList().isEmpty()) {
                    for (TreeNode<V> treeNode : node.getChildList()) {
                        Map<TreeNode<V>, Integer> map = new HashMap<>();
                        map.put(treeNode, depth + 1);
                        stack.push(map);
                    }
                }
            }
        }
    }

    /**
     * 递归前序遍历方式
     * <p>
     * 前序遍历(Pre-Order Traversal) ：指先访问根，然后访问子树的遍历方式，二叉树则为：根->左->右
     *
     * @param tree
     * @param depth
     * @param <V>
     */
    public static <V> void dfsPreOrderTraversal(TreeNode<V> tree, int depth) {
        if (tree != null) {
            // 打印节点值以及深度
            System.out.print("-->[" + tree.getValue().toString() + "," + depth + "]");

            if (tree.getChildList() != null && !tree.getChildList().isEmpty()) {
                for (TreeNode<V> item : tree.getChildList()) {
                    dfsPreOrderTraversal(item, depth + 1);
                }
            }
        }
    }

    /**
     * 递归后序遍历方式
     * <p>
     * 后序遍历(Post-Order Traversal)：指先访问子树，然后访问根的遍历方式，二叉树则为：左->右->根
     *
     * @param tree
     * @param depth
     * @param <V>
     */
    public static <V> void dfsPostOrderTraversal(TreeNode<V> tree, int depth) {
        if (tree != null) {
            if (tree.getChildList() != null && !tree.getChildList().isEmpty()) {
                for (TreeNode<V> item : tree.getChildList()) {
                    dfsPostOrderTraversal(item, depth + 1);
                }
            }

            // 打印节点值以及深度
            System.out.print("-->[" + tree.getValue().toString() + "," + depth + "]");
        }
    }

    /**
     * 递归中序遍历方式
     * <p>
     * 中序遍历(In-Order Traversal)：指先访问左（右）子树，然后访问根，最后访问右（左）子树的遍历方式，二叉树则为：左->根->右
     *
     * @param tree
     * @param depth
     * @param <V>
     */
    public static <V> void dfsInOrderTraversal(TreeNode<V> tree, int depth) {
        if (tree.getLeft() != null) {
            dfsInOrderTraversal(tree.getLeft(), depth + 1);
        }

        // 打印节点值以及深度
        System.out.print("-->[" + tree.getValue().toString() + "," + depth + "]");

        if (tree.getRight() != null) {
            dfsInOrderTraversal(tree.getRight(), depth + 1);
        }
    }

}
```



### 广度优先搜索(BFS)

广度优先搜索（Breadth-First Search / BFS）是**优先遍历邻居节点**而不是子节点的图遍历算法。

![广度优先搜索](png/Java/算法-广度优先搜索.jpg)

**BFS一般用来解决最短路径的问题**。和深度优先搜索不同，广度优先的搜索是从起始点出发，一层一层地进行，每层当中的点距离起始点的步数都是相同的，当找到了目的地之后就可以立即结束。广度优先的搜索可以同时从起始点和终点开始进行，称之为双端 BFS。这种算法往往可以大大地提高搜索的效率。



**代码案例**

```java
/**
 * Breadth-First Search(BFS)
 * <p>
 * 从根节点出发，在横向遍历二叉树层段节点的基础上纵向遍历二叉树的层次。
 * <p>
 * 数据结构：队列
 * 父节点入队，父节点出队列，先左子节点入队，后右子节点入队。递归遍历全部节点即可
 *
 * @author lry
 */
public class BreadthFirstSearch {

    /**
     * 树节点
     *
     * @param <V>
     */
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    public static class TreeNode<V> {
        private V value;
        private List<TreeNode<V>> childList;
    }

    /**
     * 模型：
     * .......A
     * ...../   \
     * ....B     C
     * .../ \   /  \
     * ..D   E F    G
     * ./ \        /  \
     * H  I       J    K
     */
    public static void main(String[] args) {
        TreeNode<String> treeNodeA = new TreeNode<>("A", new ArrayList<>());
        TreeNode<String> treeNodeB = new TreeNode<>("B", new ArrayList<>());
        TreeNode<String> treeNodeC = new TreeNode<>("C", new ArrayList<>());
        TreeNode<String> treeNodeD = new TreeNode<>("D", new ArrayList<>());
        TreeNode<String> treeNodeE = new TreeNode<>("E", new ArrayList<>());
        TreeNode<String> treeNodeF = new TreeNode<>("F", new ArrayList<>());
        TreeNode<String> treeNodeG = new TreeNode<>("G", new ArrayList<>());
        TreeNode<String> treeNodeH = new TreeNode<>("H", new ArrayList<>());
        TreeNode<String> treeNodeI = new TreeNode<>("I", new ArrayList<>());
        TreeNode<String> treeNodeJ = new TreeNode<>("J", new ArrayList<>());
        TreeNode<String> treeNodeK = new TreeNode<>("K", new ArrayList<>());
        // A->B,C
        treeNodeA.getChildList().add(treeNodeB);
        treeNodeA.getChildList().add(treeNodeC);
        // B->D,E
        treeNodeB.getChildList().add(treeNodeD);
        treeNodeB.getChildList().add(treeNodeE);
        // C->F,G
        treeNodeC.getChildList().add(treeNodeF);
        treeNodeC.getChildList().add(treeNodeG);
        // D->H,I
        treeNodeD.getChildList().add(treeNodeH);
        treeNodeD.getChildList().add(treeNodeI);
        // G->J,K
        treeNodeG.getChildList().add(treeNodeJ);
        treeNodeG.getChildList().add(treeNodeK);

        System.out.println("递归方式");
        bfsRecursive(Arrays.asList(treeNodeA), 0);
        System.out.println();
        System.out.println("非递归方式");
        bfsNotRecursive(treeNodeA);
    }

    /**
     * 递归遍历
     *
     * @param children
     * @param depth
     * @param <V>
     */
    public static <V> void bfsRecursive(List<TreeNode<V>> children, int depth) {
        List<TreeNode<V>> thisChildren, allChildren = new ArrayList<>();
        for (TreeNode<V> child : children) {
            // 打印节点值以及深度
            System.out.print("-->[" + child.getValue().toString() + "," + depth + "]");

            thisChildren = child.getChildList();
            if (thisChildren != null && thisChildren.size() > 0) {
                allChildren.addAll(thisChildren);
            }
        }

        if (allChildren.size() > 0) {
            bfsRecursive(allChildren, depth + 1);
        }
    }

    /**
     * 非递归遍历
     *
     * @param tree
     * @param <V>
     */
    public static <V> void bfsNotRecursive(TreeNode<V> tree) {
        if (tree != null) {
            // 跟上面一样，使用 Map 也只是为了保存树的深度，没这个需要可以不用 Map
            Queue<Map<TreeNode<V>, Integer>> queue = new ArrayDeque<>();
            Map<TreeNode<V>, Integer> root = new HashMap<>();
            root.put(tree, 0);
            queue.offer(root);

            while (!queue.isEmpty()) {
                Map<TreeNode<V>, Integer> itemMap = queue.poll();
                TreeNode<V> node = itemMap.keySet().iterator().next();
                int depth = itemMap.get(node);

                //打印节点值以及深度
                System.out.print("-->[" + node.getValue().toString() + "," + depth + "]");

                if (node.getChildList() != null && !node.getChildList().isEmpty()) {
                    for (TreeNode<V> child : node.getChildList()) {
                        Map<TreeNode<V>, Integer> map = new HashMap<>();
                        map.put(child, depth + 1);
                        queue.offer(map);
                    }
                }
            }
        }
    }

}
```



### 迪杰斯特拉算法(Dijkstra)

**迪杰斯特拉(Dijkstra)算法** 是典型最短路径算法，用于计算一个节点到其他节点的最短路径。它的主要特点是以起始点为中心向外层层扩展(广度优先搜索思想)，直到扩展到终点为止。



**基本思想**

通过Dijkstra计算图G中的最短路径时，需要指定起点s(即从顶点s开始计算)。

此外，引进两个集合S和U。S的作用是记录已求出最短路径的顶点(以及相应的最短路径长度)，而U则是记录还未求出最短路径的顶点(以及该顶点到起点s的距离)。

初始时，S中只有起点s；U中是除s之外的顶点，并且U中顶点的路径是"起点s到该顶点的路径"。然后，从U中找出路径最短的顶点，并将其加入到S中；接着，更新U中的顶点和顶点对应的路径。 然后，再从U中找出路径最短的顶点，并将其加入到S中；接着，更新U中的顶点和顶点对应的路径。 ... 重复该操作，直到遍历完所有顶点。


**操作步骤**

- 初始时，S只包含起点s；U包含除s外的其他顶点，且U中顶点的距离为"起点s到该顶点的距离"[例如，U中顶点v的距离为(s,v)的长度，然后s和v不相邻，则v的距离为∞]
- U中选出"距离最短的顶点k"，并将顶点k加入到S中；同时，从U中移除顶点k
- 更新U中各个顶点到起点s的距离。之所以更新U中顶点的距离，是由于上一步中确定了k是求出最短路径的顶点，从而可以利用k来更新其它顶点的距离；例如，(s,v)的距离可能大于(s,k)+(k,v)的距离
- 重复步骤(2)和(3)，直到遍历完所有顶点



**迪杰斯特拉算法图解**

![img](png/Java/算法-迪杰斯特拉.png)

以上图G4为例，来对迪杰斯特拉进行算法演示(以第4个顶点D为起点)：

![img](png/Java/算法-迪杰斯特拉演示.png)

 **代码案例**

```java
public class Dijkstra {
     // 代表正无穷
    public static final int M = 10000;
    public static String[] names = new String[]{"A", "B", "C", "D", "E", "F", "G",};

    public static void main(String[] args) {
        // 二维数组每一行分别是 A、B、C、D、E 各点到其余点的距离,
        // A -> A 距离为0, 常量M 为正无穷
        int[][] weight1 = {
                {0, 12, M, M, M, 16, 14},
                {12, 0, 10, M, M, 7, M},
                {M, 10, 0, 3, 5, 6, M},
                {M, M, 3, 0, 4, M, M},
                {M, M, 5, 4, 0, 2, 8},
                {16, 7, 6, M, 2, 0, 9},
                {14, M, M, M, 8, 9, 0}
        };

        int start = 0;
        int[] shortPath = dijkstra(weight1, start);
        System.out.println("===============");
        for (int i = 0; i < shortPath.length; i++) {
            System.out.println("从" + names[start] + "出发到" + names[i] + "的最短距离为：" + shortPath[i]);
        }
    }

    /**
     * Dijkstra算法
     *
     * @param weight 图的权重矩阵
     * @param start  起点编号start（从0编号，顶点存在数组中）
     * @return 返回一个int[] 数组，表示从start到它的最短路径长度
     */
    public static int[] dijkstra(int[][] weight, int start) {
        // 顶点个数
        int n = weight.length;
        // 标记当前该顶点的最短路径是否已经求出,1表示已求出
        int[] visited = new int[n];
        // 保存start到其他各点的最短路径
        int[] shortPath = new int[n];

        // 保存start到其他各点最短路径的字符串表示
        String[] path = new String[n];
        for (int i = 0; i < n; i++) {
            path[i] = names[start] + "-->" + names[i];
        }

        // 初始化，第一个顶点已经求出
        shortPath[start] = 0;
        visited[start] = 1;

        // 要加入n-1个顶点
        for (int count = 1; count < n; count++) {
            // 选出一个距离初始顶点start最近的未标记顶点
            int k = -1;
            int dMin = Integer.MAX_VALUE;
            for (int i = 0; i < n; i++) {
                if (visited[i] == 0 && weight[start][i] < dMin) {
                    dMin = weight[start][i];
                    k = i;
                }
            }

            // 将新选出的顶点标记为已求出最短路径，且到start的最短路径就是dmin
            shortPath[k] = dMin;
            visited[k] = 1;

            // 以k为中间点，修正从start到未访问各点的距离
            for (int i = 0; i < n; i++) {
                // 如果 '起始点到当前点距离' + '当前点到某点距离' < '起始点到某点距离', 则更新
                if (visited[i] == 0 && weight[start][k] + weight[k][i] < weight[start][i]) {
                    weight[start][i] = weight[start][k] + weight[k][i];
                    path[i] = path[k] + "-->" + names[i];
                }
            }
        }

        for (int i = 0; i < n; i++) {
            System.out.println("从" + names[start] + "出发到" + names[i] + "的最短路径为：" + path[i]);
        }

        return shortPath;
    }

}
```



### kruskal(克鲁斯卡尔)算法

https://www.cnblogs.com/skywang12345/category/508186.html



## 排序算法

十种常见排序算法可以分为两大类：

- **非线性时间比较类排序**

  通过比较来决定元素间的相对次序，由于其时间复杂度不能突破O(nlogn)，因此称为非线性时间比较类排序。

- **线性时间非比较类排序**

  不通过比较来决定元素间的相对次序，它可以突破基于比较排序的时间下界，以线性时间运行，因此称为线性时间非比较类排序。

交换排序法 
   冒泡排序 |鸡尾酒排序 |奇偶排序 |梳排序 |地精排序(gnome_sort) |Bogo排序|快速排序
选择排序法 
   选择排序 | 堆排序
插入排序法 
   插入排序 | 希尔排序 | 二叉查找树排序 | Library sort | Patience sorting
归并排序法 
   归并排序 | Strand sort
非比较排序法 
   基数排序 | 桶排序 | 计数排序 | 鸽巢排序 | Burstsort | Bead sort
其他 
   拓扑排序 | 排序网络 | Bitonic sorter | Batcher odd-even mergesort | Pancake sorting
低效排序法 
   Bogosort | Stooge sort

![排序算法](png/Java/排序算法.png)

### 冒泡排序（Bubble Sort）

**循环遍历多次每次从前往后把大元素往后调，每次确定一个最大(最小)元素，多次后达到排序序列。**这个算法的名字由来是因为越小的元素会经由交换慢慢“浮”到数列的顶端。 

![冒泡排序](png/Java/冒泡排序.jpg)

![冒泡排序](png/Java/冒泡排序.gif)

**算法步骤**

- 比较相邻的元素。如果第一个比第二个大，就交换它们两个
- 对每一对相邻元素作同样的工作，从开始第一对到结尾的最后一对，这样在最后的元素应该会是最大的数
- 针对所有的元素重复以上的步骤，除了最后一个
- 重复步骤1~3，直到排序完成

**代码实现**

```java
/**
  * 冒泡排序
  * <p>
  * 描述：每轮连续比较相邻的两个数，前数大于后数，则进行替换。每轮完成后，本轮最大值已被移至最后
  *
  * @param arr 待排序数组
  */
public static int[] bubbleSort(int[] arr) {
    for (int i = 0; i < arr.length - 1; i++) {
        for (int j = 0; j < arr.length - 1 - i; j++) {
            // 每次比较2个相邻的数,前一个小于后一个
            if (arr[j] > arr[j + 1]) {
                int tmp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = tmp;
            }
        }
    }
    
    return arr;
}
```

以下是冒泡排序算法复杂度:

| 平均时间复杂度 | 最好情况 | 最坏情况 | 空间复杂度 |
| :------------- | :------- | :------- | :--------- |
| O(n²)          | O(n)     | O(n²)    | O(1)       |

冒泡排序是最容易实现的排序, 最坏的情况是每次都需要交换, 共需遍历并交换将近n²/2次, 时间复杂度为O(n²). 最佳的情况是内循环遍历一次后发现排序是对的, 因此退出循环, 时间复杂度为O(n)。平均来讲, 时间复杂度为O(n²). 由于冒泡排序中只有缓存的temp变量需要内存空间, 因此空间复杂度为常量O(1)。

Tips: 由于冒泡排序只在相邻元素大小不符合要求时才调换他们的位置, 它并不改变相同元素之间的相对顺序, 因此它是稳定的排序算法。

#### 鸡尾酒排序(Cocktail sort)

**鸡尾酒排序**，也就是**定向冒泡排序**, 鸡尾酒搅拌排序, 是冒泡排序的一种变形。此算法与冒泡排序的不同处在于排序时是以双向在序列中进行排序。此算法与冒泡排序的不同处在于**从低到高然后从高到低**，而冒泡排序则仅从低到高去比较序列里的每个元素。他可以得到比冒泡排序稍微好一点的效能。

**算法步骤**

 i. 先对数组从左到右进行升序的冒泡排序；
 ii. 再对数组进行从右到左的降序的冒泡排序；
 iii. 以此类推，持续的、依次的改变冒泡的方向，并不断缩小没有排序的数组范围；

**算法实例**

例：     88   7   79   64   55   98   48   52   4   13
第一趟:   7   79  64   55   88   48   52   4    13  98
第二趟：  4   7   79   64   55   88   48   42   13   98
第三趟：  4   7   64   55   79   48   42   13   88  98
第四趟：  4   7   13   64   55   79   48   42   88  98
第五趟：  4   7   13   55   64   48  42   79   88  98
第六趟：  4   7   13   42   55   64  48   79   88  98
第七趟：  4   7   13   42   55   48  64   79   88  98
第八趟：  4   7   13   42   48   55  64   79   88  98  

鸡尾酒排序的代码如下：

```
public static void coktailSort(int[] arr) {

        for (int i = 0; i < arr.length/2; i++) {
            for (int j = i; j < arr.length-1-i; j++) {
                if(arr[j]>arr[j+1]) {
                    int t=arr[j];
                    arr[j]=arr[j+1];
                    arr[j+1]=t;
                }
            }
            for (int j = arr.length-(1+1)-i; j >i; j--) {
                if(arr[j]<arr[j-1]) {
                    int t=arr[j];
                    arr[j]=arr[j-1];
                    arr[j-1]=t;
                }
            }
        }

    }
```

与冒泡排序不同的地方：

鸡尾酒排序等于是冒泡排序的轻微变形。不同的地方在于从低到高然后从高到低，而冒泡排序则仅从低到高去比较序列里的每个元素。他可以得到比冒泡排序稍微好一点的效能，原因是冒泡排序只从一个方向进行比对(由低到高)，每次循环只移动一个项目。

以序列(2,3,4,5,1)为例，鸡尾酒排序只需要访问一次序列就可以完成排序，但如果使用冒泡排序则需要四次。 但是在乱数序列的状态下，鸡尾酒排序与冒泡排序的效率都很差劲。

#### 地精排序(gnomeSort)

地精排序（侏儒排序）,传说有一个地精在排列一排花盘。他从前至后的排列，如果相邻的两个花盘顺序正确，他向前一步；如果花盘顺序错误，他后退一步，直到所有的花盘的顺序都排列好。(地精排序思路与插入排序和冒泡排序很像, 主要其对代码进行了极简化)

稳定性:稳定

简单，只有一层循环。时间复杂度O(n^2)，最优复杂度O(n),平均时间复杂度O(n^2)

```
public static void gnomeSort(int[] arr) {

        int i=1;
        while(i<arr.length) {

            if(i==0||arr[i]>=arr[i-1]) {
                i++;
            }else {
                int t=arr[i];
                arr[i]=arr[i-1];
                arr[i-1]=t;
                i--;
            }

        }

    }
```

#### 奇偶排序(OddevenSort)

奇偶排序或奇偶换位排序，或砖排序，是一种相对简单的排序算法，最初发明用于有本地互连的并行计算。这是与冒泡排序特点类似的一种比较排序。

该算法中，排序过程分两个阶段，奇交换和偶交换，两种交换都是成对出现。对于奇交换，它总是比较奇数索引以及其相邻的后续元素。对偶交换，总是比较偶数索引和其相邻的后续元素。思路是在数组中重复两趟扫描。第一趟扫描选择所有的数据项对，a[j]和a[j+1]，j是奇数(j=1, 3, 5……)。如果它们的关键字的值次序颠倒，就交换它们。第二趟扫描对所有的偶数数据项进行同样的操作(j=2, 4,6……)。重复进行这样两趟的排序直到数组全部有序。

简单来说就是：奇数列排一趟序,偶数列排一趟序,再奇数排,再偶数排,直到全部有序。

**示例：**

待排数组[6 2 4 1 5 9]  //目标按从小到大排序

第一次比较奇数列,奇数列与它的邻居偶数列比较,如6和2比,4和1比,5和9比

[6 2 4 1 5 9]

交换后变成

[2 6 1 4 5 9]

第二次比较偶数列,即6和1比,5和5比

[2 6 1 4 5 9]

交换后变成

[2 1 6 4 5 9]

第三次又是奇数列,选择的是2,6,5分别与它们的邻居列比较

[2 1 6 4 5 9]

交换后

[1 2 4 6 5 9] 

第四次偶数列

[1 2 4 6 5 9]

一次交换

[1 2 4 5 6 9]

```
/*
 * 奇偶排序
 */
public class OddevenSort {
    public static void main(String[] args) {
        int[] arrayData = { 2, 3, 4, 5, 6, 7, 8, 9, 1 };
        OddevenSortMethod(arrayData);
        for (int integer : arrayData) {
            System.out.print(integer);
            System.out.print(" ");
        }
    }
 
    public static void OddevenSortMethod(int[] arrayData) {
        int temp;
        int length = arrayData.length;
        boolean whetherSorted = true;
 
        while (whetherSorted) {
            whetherSorted = false;
            for (int i = 1; i < length - 1; i++) {
                if (arrayData[i] > arrayData[i + 1]) {
                    temp = arrayData[i];
                    arrayData[i] = arrayData[i + 1];
                    arrayData[i + 1] = temp;
                    whetherSorted = true;
                }
            }
 
            for (int i = 0; i < length - 1; i++) {
                if (arrayData[i] > arrayData[i + 1]) {
                    temp = arrayData[i];
                    arrayData[i] = arrayData[i + 1];
                    arrayData[i + 1] = temp;
                    whetherSorted = true;
                }
            }
        }
    }
}
```

算法复杂度分析

| **排序方法** | **时间复杂度** | **空间复杂度** | **稳定性** | **复杂性** |      |        |
| ------------ | -------------- | -------------- | ---------- | ---------- | ---- | ------ |
| **平均情况** | **最坏情况**   | **最好情况**   |            |            |      |        |
| **奇偶排序** | O(nlog2n)      | O(nlog2n)      | O(n)       | O(1)       | 稳定 | 较简单 |

### 选择排序（Selection Sort）

**选择排序(SelectionSort)**是一种简单直观的排序算法。它的工作原理：首先在未排序序列中找到最小（大）元素，存放到排序序列的起始位置，然后，再从剩余未排序元素中继续寻找最小（大）元素，然后放到已排序序列的末尾。以此类推，直到所有元素均排序完毕。 

![选择排序](png/Java/选择排序.jpg)

![选择排序](png/Java/选择排序.gif)

**算法描述**

n个记录的直接选择排序可经过n-1趟直接选择排序得到有序结果。具体算法描述如下：

- 初始状态：无序区为R[1..n]，有序区为空
- 第i趟排序(i=1,2,3…n-1)开始时，当前有序区和无序区分别为R[1..i-1]和R(i..n）。该趟排序从当前无序区中-选出关键字最小的记录 R[k]，将它与无序区的第1个记录R交换，使R[1..i]和R[i+1..n)分别变为记录个数增加1个的新有序区和记录个数减少1个的新无序区
- n-1趟结束，数组有序化了

**代码实现**

```java
 /**
     * 选择排序
     * <p>
     * 描述：每轮选择出最小值，然后依次放置最前面
     *
     * @param arr 待排序数组
     */
    public static int[] selectSort(int[] arr) {
        for (int i = 0; i < arr.length - 1; i++) {
            // 选最小的记录
            int min = i;
            for (int j = i + 1; j < arr.length; j++) {
                if (arr[min] > arr[j]) {
                    min = j;
                }
            }

            // 内层循环结束后，即找到本轮循环的最小的数以后，再进行交换：交换a[i]和a[min]
            if (min != i) {
                int temp = arr[i];
                arr[i] = arr[min];
                arr[min] = temp;
            }
        }

        return arr;
    }
```

以下是选择排序复杂度:

| 平均时间复杂度 | 最好情况 | 最坏情况 | 空间复杂度 |
| :------------- | :------- | :------- | :--------- |
| O(n²)          | O(n²)    | O(n²)    | O(1)       |

选择排序的时间复杂度：简单选择排序的比较次数与序列的初始排序无关。 假设待排序的序列有 N 个元素，则比较次数永远都是N (N - 1) / 2。而移动次数与序列的初始排序有关。当序列正序时，移动次数最少，为 0。当序列反序时，移动次数最多，为3N (N - 1) / 2。

选择排序的简单和直观名副其实，这也造就了它”出了名的慢性子”，无论是哪种情况，哪怕原数组已排序完成，它也将花费将近n²/2次遍历来确认一遍。即便是这样，它的排序结果也还是不稳定的。 唯一值得高兴的是，它并不耗费额外的内存空间。

### 插入排序（Insertion Sort）

插入排序（InsertionSort）的算法描述是一种简单直观的排序算法。它的工作原理是通过构建有序序列，对于未排序数据，在已排序序列中从后向前扫描，找到相应位置并插入。插入排序由于操作不尽相同，可分为 **直接插入排序、折半插入排序(又称二分插入排序)、链表插入排序、希尔排序** 。

![插入排序](png/Java/插入排序.jpg)

![插入排序](png/Java/插入排序.gif)

**算法描述**

一般来说，插入排序都采用in-place在数组上实现。具体算法描述如下：

- 从第一个元素开始，该元素可以认为已经被排序
- 取出下一个元素，在已经排序的元素序列中从后向前扫描
- 如果该元素（已排序）大于新元素，将该元素移到下一位置
- 重复步骤3，直到找到已排序的元素小于或者等于新元素的位置
- 将新元素插入到该位置后
- 重复步骤2~5

**代码实现**

```java
   /**
     * 直接插入排序
     * <p>
     * 1. 从第一个元素开始，该元素可以认为已经被排序
     * 2. 取出下一个元素，在已经排序的元素序列中从后向前扫描
     * 3. 如果该元素（已排序）大于新元素，将该元素移到下一位置
     * 4. 重复步骤3，直到找到已排序的元素小于或者等于新元素的位置
     * 5. 将新元素插入到该位置后
     * 6. 重复步骤2~5
     *
     * @param arr 待排序数组
     */
    public static int[] insertionSort(int[] arr) {
        for (int i = 1; i < arr.length; i++) {
            // 取出下一个元素，在已经排序的元素序列中从后向前扫描
            int temp = arr[i];
            for (int j = i; j >= 0; j--) {
                if (j > 0 && arr[j - 1] > temp) {
                    // 如果该元素（已排序）大于取出的元素temp，将该元素移到下一位置
                    arr[j] = arr[j - 1];
                } else {
                    // 将新元素插入到该位置后
                    arr[j] = temp;
                    break;
                }
            }
        }

        return arr;
    }

    /**
     * 折半插入排序
     * <p>
     * 往前找合适的插入位置时采用二分查找的方式，即折半插入
     * <p>
     * 交换次数较多的实现
     *
     * @param arr 待排序数组
     */
    public static int[] insertionBinarySort(int[] arr) {
        for (int i = 1; i < arr.length; i++) {
            if (arr[i] < arr[i - 1]) {
                int tmp = arr[i];

                // 记录搜索范围的左边界,右边界
                int low = 0, high = i - 1;
                while (low <= high) {
                    // 记录中间位置Index
                    int mid = (low + high) / 2;
                    // 比较中间位置数据和i处数据大小，以缩小搜索范围
                    if (arr[mid] < tmp) {
                        // 左边指针则一只中间位置+1
                        low = mid + 1;
                    } else {
                        // 右边指针则一只中间位置-1
                        high = mid - 1;
                    }
                }

                // 将low~i处数据整体向后移动1位
                for (int j = i; j > low; j--) {
                    arr[j] = arr[j - 1];
                }
                arr[low] = tmp;
            }
        }

        return arr;
    }
```

插入排序复杂度：

| 平均时间复杂度 | 最好情况 | 最坏情况 | 空间复杂度 |
| :------------- | :------- | :------- | :--------- |
| O(n²)          | O(n)     | O(n²)    | O(1)       |

Tips：由于直接插入排序每次只移动一个元素的位， 并不会改变值相同的元素之间的排序， 因此它是一种稳定排序。插入排序在实现上，通常采用in-place排序（即只需用到O(1)的额外空间的排序），因而在从后向前扫描过程中，需要反复把已排序元素逐步向后挪位，为最新元素提供插入空间。



### 希尔排序（Shell Sort）

1959年Shell发明，第一个突破O(n2)的排序算法。希尔排序(Shell Sort)是插入排序的一种，是针对直接插入排序算法的改进，是将整个无序列分割成若干小的子序列分别进行插入排序，希尔排序并不稳定。它与插入排序的不同之处在于，它会优先比较距离较远的元素。希尔排序又叫**缩小增量排序**。



![希尔排序](png/Java/希尔排序.gif)

**算法原理**

先将整个待排序的记录序列分割成为若干子序列分别进行直接插入排序，具体算法描述：

- 选择一个增量序列t1，t2，…，tk，其中ti>tj，tk=1
- 按增量序列个数k，对序列进行k 趟排序
- 每趟排序，根据对应的增量ti，将待排序列分割成若干长度为m 的子序列，分别对各子表进行直接插入排序。仅增量因子为1 时，整个序列作为一个表来处理，表长度即为整个序列的长度

**算法实例**

假如有初始数据：25  11  45  26  12  78。

　　1、第一轮排序，将该数组分成 6/2=3 个数组序列，第1个数据和第4个数据为一对，第2个数据和第5个数据为一对，第3个数据和第6个数据为一对，每对数据进行比较排序，排序后顺序为：[25, 11, 45, 26, 12, 78]。

　　2、第二轮排序 ，将上轮排序后的数组分成6/4=1个数组序列，此时逐个对数据比较，按照插入排序对该数组进行排序，排序后的顺序为：[11, 12, 25, 26, 45, 78]。

　　对于插入排序而言，如果原数组是基本有序的，那排序效率就可大大提高。另外，对于数量较小的序列使用直接插入排序，会因需要移动的数据量少，其效率也会提高。因此，希尔排序具有较高的执行效率。

　　希尔排序并不稳定，O(1)的额外空间，时间复杂度为O(N*(logN)^2)。

**代码实现**

```
 /**
     * 希尔排序
     * <p>
     * 1. 选择一个增量序列t1，t2，…，tk，其中ti>tj，tk=1；（一般初次取数组半长，之后每次再减半，直到增量为1）
     * 2. 按增量序列个数k，对序列进行k 趟排序；
     * 3. 每趟排序，根据对应的增量ti，将待排序列分割成若干长度为m 的子序列，分别对各子表进行直接插入排序。
     * 仅增量因子为1 时，整个序列作为一个表来处理，表长度即为整个序列的长度。
     *
     * @param arr 待排序数组
     */
    public static int[] shellSort(int[] arr) {
        int gap = arr.length / 2;

        // 不断缩小gap，直到1为止
        for (; gap > 0; gap /= 2) {
            // 使用当前gap进行组内插入排序
            for (int j = 0; (j + gap) < arr.length; j++) {
                for (int k = 0; (k + gap) < arr.length; k += gap) {
                    if (arr[k] > arr[k + gap]) {
                        int temp = arr[k + gap];
                        arr[k + gap] = arr[k];
                        arr[k] = temp;
                    }
                }
            }
        }

        return arr;
    }


// 其他实现shellSort

public void shellSort(int[] arr) {
        // i表示希尔排序中的第n/2+1个元素（或者n/4+1）
        // j表示希尔排序中从0到n/2的元素（n/4）
        // r表示希尔排序中n/2+1或者n/4+1的值
        int i, j, r, tmp;
        // 划组排序
        for(r = arr.length / 2; r >= 1; r = r / 2) {
            for(i = r; i < arr.length; i++) {
                tmp = arr[i];
                j = i - r;
                // 一轮排序
                while(j >= 0 && tmp < arr[j]) {
                    arr[j+r] = arr[j];
                    j -= r;
                }
                arr[j+r] = tmp;
            }
         
        }
        return arr；
    }
```

以下是希尔排序复杂度:

| 平均时间复杂度 | 最好情况   | 最坏情况   | 空间复杂度 |
| :------------- | :--------- | :--------- | :--------- |
| O(nlog2 n)     | O(nlog2 n) | O(nlog2 n) | O(1)       |

Tips：希尔排序的核心在于间隔序列的设定。既可以提前设定好间隔序列，也可以动态的定义间隔序列。动态定义间隔序列的算法是《算法（第4版）》的合著者Robert Sedgewick提出的。　

### 归并排序（Merging Sort）

归并排序，算林届的新秀，引领着分治法的潮流。归并排序法（Merging Sort）是将两个（或两个以上）有序表合并成一个新的有序表，即把待排序序列分为若干个子序列，每个子序列是有序的，然后再把有序子序列**合并**为整体有序序列。

这种思想是一种算法设计的思想，很多问题都可以采用这种方式解决。映射到编程领域，其实就是递归的思想。因此在归并排序的算法中，将会出现递归调用。

**应用场景**：内存少的时候使用，可以进行**并行计算**的时候使用。

**算法步骤**

- 选择**相邻**两个数组成一个有序序列
- 选择相邻的两个有序序列组成一个有序序列
- 重复第二步，直到全部组成一个**有序**序列

![归并排序](png/Java/归并排序.jpg)

![归并排序](png/Java/归并排序.gif)

**算法描述**

**a.递归法**（假设序列共有n个元素）

①. 将序列每相邻两个数字进行归并操作，形成 floor(n/2)个序列，排序后每个序列包含两个元素；
②. 将上述序列再次归并，形成 floor(n/4)个序列，每个序列包含四个元素；
③. 重复步骤②，直到所有元素排序完毕。

**b.迭代法**

①. 申请空间，使其大小为两个已经排序序列之和，该空间用来存放合并后的序列
②. 设定两个指针，最初位置分别为两个已经排序序列的起始位置
③. 比较两个指针所指向的元素，选择相对小的元素放入到合并空间，并移动指针到下一位置
④. 重复步骤③直到某一指针到达序列尾
⑤. 将另一序列剩下的所有元素直接复制到合并序列尾

**代码实现**

```java
/**
     * 归并排序（递归）
     * <p>
     * ①. 将序列每相邻两个数字进行归并操作，形成 floor(n/2)个序列，排序后每个序列包含两个元素；
     * ②. 将上述序列再次归并，形成 floor(n/4)个序列，每个序列包含四个元素；
     * ③. 重复步骤②，直到所有元素排序完毕。
     *
     * @param arr 待排序数组
     */
    public static int[] mergeSort(int[] arr) {
        return mergeSort(arr, 0, arr.length - 1);
    }

    private static int[] mergeSort(int[] arr, int low, int high) {
        int center = (high + low) / 2;
        if (low < high) {
            // 递归，直到low==high，也就是数组已不能再分了，
            mergeSort(arr, low, center);
            mergeSort(arr, center + 1, high);

            // 当数组不能再分，开始归并排序
            mergeSort(arr, low, center, high);
        }

        return arr;
    }

    private static void mergeSort(int[] a, int low, int mid, int high) {
        int[] temp = new int[high - low + 1];
        int i = low, j = mid + 1, k = 0;

        // 把较小的数先移到新数组中
        while (i <= mid && j <= high) {
            if (a[i] < a[j]) {
                temp[k++] = a[i++];
            } else {
                temp[k++] = a[j++];
            }
        }

        // 把左边剩余的数移入数组
        while (i <= mid) {
            temp[k++] = a[i++];
        }

        // 把右边边剩余的数移入数组
        while (j <= high) {
            temp[k++] = a[j++];
        }

        // 把新数组中的数覆盖nums数组
        for (int x = 0; x < temp.length; x++) {
            a[x + low] = temp[x];
        }
    }
```

以下是归并排序算法复杂度:

| 平均时间复杂度 | 最好情况  | 最坏情况  | 空间复杂度 |
| :------------- | :-------- | :-------- | :--------- |
| O(nlog₂n)      | O(nlog₂n) | O(nlog₂n) | O(n)       |

从效率上看，归并排序可算是排序算法中的”佼佼者”. 假设数组长度为n，那么拆分数组共需logn，, 又每步都是一个普通的合并子数组的过程， 时间复杂度为O(n)， 故其综合时间复杂度为O(nlogn)。另一方面， 归并排序多次递归过程中拆分的子数组需要保存在内存空间， 其空间复杂度为O(n)。

Tips：和选择排序一样，归并排序的性能不受输入数据的影响，但表现比选择排序好的多，因为始终都是`O(nlogn）`的时间复杂度。代价是需要额外的内存空间。



### 快速排序（Quick Sort）

快速排序（Quicksort）是对冒泡排序的一种改进，借用了分治的思想，由C. A. R. Hoare在1962年提出。基本思想是通过一趟排序将待排记录分隔成独立的两部分，其中一部分记录的关键字均比另一部分的关键字小，则可分别对这两部分记录继续进行排序，以达到整个序列有序。

![快速排序](png/Java/快速排序.gif)

**算法描述**

快速排序使用分治法来把一个串（list）分为两个子串（sub-lists）。具体算法描述如下：

- 从数列中挑出一个元素，称为 “基准”（pivot）
- 重新排序数列，所有元素比基准值小的摆放在基准前面，所有元素比基准值大的摆在基准的后面（相同的数可以到任一边）。在这个分区退出之后，该基准就处于数列的中间位置。这个称为分区（partition）操作
- 递归地（recursive）把小于基准值元素的子数列和大于基准值元素的子数列排序

**算法实例**

场景：对 6 1 2 7 9 3 4 5 10 8 这 10 个数进行排序
思路：
先找一个基准数（一个用来参照的数），为了方便，我们选最左边的 6，希望将 >6 的放到 6 的右边，<6 的放到 6 左边。
如：3 1 2 5 4 6 9 7 10 8
先假设需要将 6 挪到的位置为 k，k 左边的数 <6，右边的数 >6

- 我们先从初始数列“6 1 2 7 9 3 4 5 10 8 ”的两端开始“探测 ”，先从右边往左找一个 <6 的数，再从左往右找一个 >6 的数，然后交换。我们用变量 i 和变量 j 指向序列的最左边和最右边。刚开始时最左边 i=0 指向 6，最右边 j=9 指向 8 ；
- 现在设置的基准数是最左边的数，所以序列先右往左移动（j–），当找到一个 <6 的数（5）就停下来。
  接着序列从左往右移动（i++），直到找到一个 >6 的数又停下来（7）；
- 两者交换，结果：6 1 2 5 9 3 4 7 10 8；
- j 的位置继续向左移动（友情提示：每次都必须先从 j 的位置出发），发现 4 满足要求，接着 i++ 发现 9 满足要求，交换后的结果：6 1 2 5 4 3 9 7 10 8；
- j 的位置继续向左移动（友情提示：每次都必须先从 j 的位置出发），发现 4 满足要求，接着 i++ 发现 9 满足要求，交换后的结果：6 1 2 5 4 3 9 7 10 8；
- 目前 j 指向的值为 9，i 指向的值为 4，j-- 发现 3 符合要求，接着 i++ 发现 i=j，说明这一轮移动结束啦。现在将基准数 6 和 3 进行交换，结果：3 1 2 5 4 6 9 7 10 8；现在 6 左边的数都是 <6 的，而右边的数都是 >6 的，但游戏还没结束
- 我们将 6 左边的数拿出来先：3 1 2 5 4，这次以 3 为基准数进行调整，使得 3 左边的数 ❤️，右边的数 >3，根据之前的模拟，这次的结果：2 1 3 5 4
- 再将 2 1 抠出来重新整理，得到的结果： 1 2
- 剩下右边的序列：9 7 10 8 也是这样来搞，最终的结果： 1 2 3 4 5 6 7 8 9 10

**代码实现**

```java
/**
     * 快速排序（递归）
     * <p>
     * ①. 从数列中挑出一个元素，称为"基准"（pivot）。
     * ②. 重新排序数列，所有比基准值小的元素摆放在基准前面，所有比基准值大的元素摆在基准后面（相同的数可以到任一边）。在这个分区结束之后，该基准就处于数列的中间位置。这个称为分区（partition）操作。
     * ③. 递归地（recursively）把小于基准值元素的子数列和大于基准值元素的子数列排序。
     *
     * @param arr 待排序数组
     */
    public static int[] quickSort(int[] arr) {
        return quickSort(arr, 0, arr.length - 1);
    }

    private static int[] quickSort(int[] arr, int low, int high) {
        if (arr.length <= 0 || low >= high) {
            return arr;
        }

        int left = low;
        int right = high;

        // 挖坑1：保存基准的值
        int temp = arr[left];
        while (left < right) {
            // 坑2：从后向前找到比基准小的元素，插入到基准位置坑1中
            while (left < right && arr[right] >= temp) {
                right--;
            }
            arr[left] = arr[right];
            // 坑3：从前往后找到比基准大的元素，放到刚才挖的坑2中
            while (left < right && arr[left] <= temp) {
                left++;
            }
            arr[right] = arr[left];
        }
        // 基准值填补到坑3中，准备分治递归快排
        arr[left] = temp;
        quickSort(arr, low, left - 1);
        quickSort(arr, left + 1, high);

        return arr;
    }

    /**
     * 快速排序（非递归）
     * <p>
     * ①. 从数列中挑出一个元素，称为"基准"（pivot）。
     * ②. 重新排序数列，所有比基准值小的元素摆放在基准前面，所有比基准值大的元素摆在基准后面（相同的数可以到任一边）。在这个分区结束之后，该基准就处于数列的中间位置。这个称为分区（partition）操作。
     * ③. 把分区之后两个区间的边界（low和high）压入栈保存，并循环①、②步骤
     *
     * @param arr 待排序数组
     */
    public static int[] quickSortByStack(int[] arr) {
        Stack<Integer> stack = new Stack<>();

        // 初始状态的左右指针入栈
        stack.push(0);
        stack.push(arr.length - 1);
        while (!stack.isEmpty()) {
            // 出栈进行划分
            int high = stack.pop();
            int low = stack.pop();

            int pivotIdx = partition(arr, low, high);

            // 保存中间变量
            if (pivotIdx > low) {
                stack.push(low);
                stack.push(pivotIdx - 1);
            }
            if (pivotIdx < high && pivotIdx >= 0) {
                stack.push(pivotIdx + 1);
                stack.push(high);
            }
        }

        return arr;
    }

    private static int partition(int[] arr, int low, int high) {
        if (arr.length <= 0) return -1;
        if (low >= high) return -1;
        int l = low;
        int r = high;

        // 挖坑1：保存基准的值
        int pivot = arr[l];
        while (l < r) {
            // 坑2：从后向前找到比基准小的元素，插入到基准位置坑1中
            while (l < r && arr[r] >= pivot) {
                r--;
            }
            arr[l] = arr[r];
            // 坑3：从前往后找到比基准大的元素，放到刚才挖的坑2中
            while (l < r && arr[l] <= pivot) {
                l++;
            }
            arr[r] = arr[l];
        }

        // 基准值填补到坑3中，准备分治递归快排
        arr[l] = pivot;
        return l;
    }
```

以下是快速排序算法复杂度:

| 平均时间复杂度 | 最好情况  | 最坏情况 | 空间复杂度             |
| :------------- | :-------- | :------- | :--------------------- |
| O(nlog₂n)      | O(nlog₂n) | O(n²)    | O(1)（原地分区递归版） |

快速排序排序效率非常高。 虽然它运行最糟糕时将达到O(n²)的时间复杂度, 但通常平均来看, 它的时间复杂为O(nlogn), 比同样为O(nlogn)时间复杂度的归并排序还要快. 快速排序似乎更偏爱乱序的数列, 越是乱序的数列, 它相比其他排序而言, 相对效率更高。

Tips: 同选择排序相似, 快速排序每次交换的元素都有可能不是相邻的, 因此它有可能打破原来值为相同的元素之间的顺序. 因此, 快速排序并不稳定。



### 基数排序（Radix Sort）

基数排序是按照低位先排序，然后收集；再按照高位排序，然后再收集；依次类推，直到最高位。有时候有些属性是有优先级顺序的，先按低优先级排序，再按高优先级排序。最后的次序就是高优先级高的在前，高优先级相同的低优先级高的在前。

![基数排序](png/Java/基数排序.gif)

**算法描述**

- 取得数组中的最大数，并取得位数
- arr为原始数组，从最低位开始取每个位组成radix数组
- 对radix进行计数排序（利用计数排序适用于小范围数的特点）

**代码实现**

```java
/**
 * 基数排序（LSD 从低位开始）
 * <p>
 * 基数排序适用于：
 * (1)数据范围较小，建议在小于1000
 * (2)每个数值都要大于等于0
 * <p>
 * ①. 取得数组中的最大数，并取得位数；
 * ②. arr为原始数组，从最低位开始取每个位组成radix数组；
 * ③. 对radix进行计数排序（利用计数排序适用于小范围数的特点）；
 *
 * @param arr 待排序数组
 */
public static int[] radixSort(int[] arr) {
    // 取得数组中的最大数，并取得位数
    int max = 0;
    for (int item : arr) {
        if (max < item) {
            max = item;
        }
    }
    int maxDigit = 1;
    while (max / 10 > 0) {
        maxDigit++;
        max = max / 10;
    }

    // 申请一个桶空间
    int[][] buckets = new int[10][arr.length];
    int base = 10;

    // 从低位到高位，对每一位遍历，将所有元素分配到桶中
    for (int i = 0; i < maxDigit; i++) {
        // 存储各个桶中存储元素的数量
        int[] bktLen = new int[10];

        // 分配：将所有元素分配到桶中
        for (int value : arr) {
            int whichBucket = (value % base) / (base / 10);
            buckets[whichBucket][bktLen[whichBucket]] = value;
            bktLen[whichBucket]++;
        }

        // 收集：将不同桶里数据挨个捞出来,为下一轮高位排序做准备,由于靠近桶底的元素排名靠前,因此从桶底先捞
        int k = 0;
        for (int b = 0; b < buckets.length; b++) {
            for (int p = 0; p < bktLen[b]; p++) {
                arr[k++] = buckets[b][p];
            }
        }

        base *= 10;
    }

    return arr;
}
```

以下是基数排序算法复杂度，其中k为最大数的位数：

| 平均时间复杂度 | 最好情况   | 最坏情况   | 空间复杂度 |
| :------------- | :--------- | :--------- | :--------- |
| O(d*(n+r))     | O(d*(n+r)) | O(d*(n+r)) | O(n+r)     |

其中，**d 为位数，r 为基数，n 为原数组个数**。在基数排序中，因为没有比较操作，所以在复杂上，最好的情况与最坏的情况在时间上是一致的，均为 `O(d*(n + r))`。基数排序更适合用于对时间, 字符串等这些**整体权值未知的数据**进行排序。

Tips: 基数排序不改变相同元素之间的相对顺序，因此它是稳定的排序算法。



### 堆排序（Heap Sort）

对于堆排序，首先是建立在堆的基础上，堆是一棵完全二叉树，还要先认识下大根堆和小根堆，完全二叉树中所有节点均大于(或小于)它的孩子节点，所以这里就分为两种情况：

- 如果所有节点**「大于」**孩子节点值，那么这个堆叫做**「大根堆」**，堆的最大值在根节点
- 如果所有节点**「小于」**孩子节点值，那么这个堆叫做**「小根堆」**，堆的最小值在根节点

堆排序（Heapsort）是指利用堆这种数据结构所设计的一种排序算法。堆排序的过程就是将待排序的序列构造成一个堆，选出堆中最大的移走，再把剩余的元素调整成堆，找出最大的再移走，重复直至有序。

![大根堆-小根堆](png/Java/大根堆-小根堆.jpg)

**算法描述**

- 将初始待排序关键字序列(R1,R2….Rn)构建成大顶堆，此堆为初始的无序区
- 将堆顶元素R[1]与最后一个元素R[n]交换，此时得到新的无序区(R1,R2,……Rn-1)和新的有序区(Rn),且满足R[1,2…n-1]<=R[n]
- 由于交换后新的堆顶R[1]可能违反堆的性质，因此需要对当前无序区(R1,R2,……Rn-1)调整为新堆，然后再次将R[1]与无序区最后一个元素交换，得到新的无序区(R1,R2….Rn-2)和新的有序区(Rn-1,Rn)。不断重复此过程直到有序区的元素个数为n-1，则整个排序过程完成

**动图演示**

![堆排序](png/Java//堆排序.gif)

**代码实现**

```java
/**
     * 堆排序算法
     *
     * @param arr 待排序数组
     */
    public static int[] heapSort(int[] arr) {
        // 将无序序列构建成一个堆，根据升序降序需求选择大顶堆或小顶堆
        for (int i = arr.length / 2 - 1; i >= 0; i--) {
            adjustHeap(arr, i, arr.length);
        }

        // 将最大的节点放在堆尾，然后从根节点重新调整
        for (int j = arr.length - 1; j > 0; j--) {
            // 交换
            int temp = arr[j];
            arr[j] = arr[0];
            arr[0] = temp;

            // 完成将以i对应的非叶子结点的树调整成大顶堆
            adjustHeap(arr, 0, j);
        }

        return arr;
    }

    /**
     * 功能： 完成将以i对应的非叶子结点的树调整成大顶堆
     *
     * @param arr    待排序数组
     * @param i      表示非叶子结点在数组中索引
     * @param length 表示对多少个元素继续调整， length 是在逐渐的减少
     */
    private static void adjustHeap(int[] arr, int i, int length) {
        // 先取出当前元素的值，保存在临时变量
        int temp = arr[i];
        //开始调整。说明：1. k = i * 2 + 1 k 是 i结点的左子结点
        for (int k = i * 2 + 1; k < length; k = k * 2 + 1) {
            // 说明左子结点的值小于右子结点的值
            if (k + 1 < length && arr[k] < arr[k + 1]) {
                // k 指向右子结点
                k++;
            }

            // 如果子结点大于父结点
            if (arr[k] > temp) {
                // 把较大的值赋给当前结点
                arr[i] = arr[k];
                // i 指向 k,继续循环比较
                i = k;
            } else {
                break;
            }
        }

        //当for 循环结束后，我们已经将以i 为父结点的树的最大值，放在了 最顶(局部)。将temp值放到调整后的位置
        arr[i] = temp;
    }
```

| 平均时间复杂度 | 最好情况       | 最坏情况       | 空间复杂度 |
| :------------- | :------------- | :------------- | :--------- |
| O(n \log_{2}n) | O(n \log_{2}n) | O(n \log_{2}n) | O(1)       |

Tips: **由于堆排序中初始化堆的过程比较次数较多, 因此它不太适用于小序列.** 同时由于多次任意下标相互交换位置, 相同元素之间原本相对的顺序被破坏了, 因此, 它是不稳定的排序.



### 计数排序（Counting Sort）

计数排序不是基于比较的排序算法，其核心在于将输入的数据值转化为键存储在额外开辟的数组空间中。 作为一种线性时间复杂度的排序，计数排序要求输入的数据必须是有确定范围的整数。

**算法描述**

- 找出待排序的数组中最大和最小的元素
- 统计数组中每个值为i的元素出现的次数，存入数组C的第i项
- 对所有的计数累加（从C中的第一个元素开始，每一项和前一项相加）
- 反向填充目标数组：将每个元素i放在新数组的第C(i)项，每放一个元素就将C(i)减去1

**动图演示**

![计数排序](png/Java/计数排序.gif)

**代码实现**

```java
/**
     * 计数排序算法
     *
     * @param arr 待排序数组
     */
    public static int[] countingSort(int[] arr) {
        // 得到数列的最大值与最小值，并算出差值d
        int max = arr[0];
        int min = arr[0];
        for (int i = 1; i < arr.length; i++) {
            if (arr[i] > max) {
                max = arr[i];
            }
            if (arr[i] < min) {
                min = arr[i];
            }
        }
        int d = max - min;

        // 创建统计数组并计算统计对应元素个数
        int[] countArray = new int[d + 1];
        for (int value : arr) {
            countArray[value - min]++;
        }

        // 统计数组变形，后面的元素等于前面的元素之和
        int sum = 0;
        for (int i = 0; i < countArray.length; i++) {
            sum += countArray[i];
            countArray[i] = sum;
        }

        // 倒序遍历原始数组，从统计数组找到正确位置，输出到结果数组
        int[] sortedArray = new int[arr.length];
        for (int i = arr.length - 1; i >= 0; i--) {
            sortedArray[countArray[arr[i] - min] - 1] = arr[i];
            countArray[arr[i] - min]--;
        }

        return sortedArray;
    }

```

**算法分析**

计数排序是一个稳定的排序算法。当输入的元素是 n 个 0到 k 之间的整数时，时间复杂度是O(n+k)，空间复杂度也是O(n+k)，其排序速度快于任何比较排序算法。当k不是很大并且序列比较集中时，计数排序是一个很有效的排序算法。



### 桶排序（Bucket Sort）

桶排序是计数排序的升级版。它利用了函数的映射关系，高效与否的关键就在于这个映射函数的确定。桶排序 (Bucket sort)的工作的原理：假设输入数据服从均匀分布，将数据分到有限数量的桶里，每个桶再分别排序（有可能再使用别的排序算法或是以递归方式继续使用桶排序进行排）。

![桶排序](png/Java/桶排序.png)

![BucketSort](png/Java/BucketSort.gif)

**算法描述**

- 设置一个定量的数组当作空桶
- 遍历输入数据，并且把数据一个一个放到对应的桶里去
- 对每个不是空的桶进行排序
- 从不是空的桶里把排好序的数据拼接起来

**代码实现**

```java
/**
     * 桶排序算法
     *
     * @param arr 待排序数组
     */
    public static int[] bucketSort(int[] arr) {
        // 计算最大值与最小值
        int max = Integer.MIN_VALUE;
        int min = Integer.MAX_VALUE;
        for (int value : arr) {
            max = Math.max(max, value);
            min = Math.min(min, value);
        }

        // 计算桶的数量
        int bucketNum = (max - min) / arr.length + 1;
        List<List<Integer>> bucketArr = new ArrayList<>(bucketNum);
        for (int i = 0; i < bucketNum; i++) {
            bucketArr.add(new ArrayList<>());
        }

        // 将每个元素放入桶
        for (int value : arr) {
            int num = (value - min) / (arr.length);
            bucketArr.get(num).add(value);
        }

        // 对每个桶进行排序
        for (List<Integer> integers : bucketArr) {
            Collections.sort(integers);
        }

        // 将桶中的元素赋值到原序列
        int index = 0;
        for (List<Integer> integers : bucketArr) {
            for (Integer integer : integers) {
                arr[index++] = integer;
            }
        }

        return arr;
    }
```

**算法分析**

桶排序最好情况下使用线性时间O(n)，桶排序的时间复杂度，取决与对各个桶之间数据进行排序的时间复杂度，因为其它部分的时间复杂度都为O(n)。很显然，桶划分的越小，各个桶之间的数据越少，排序所用的时间也会越少。但相应的空间消耗就会增大。 



# Case

## 双指针

### 删除排序数组中的重复项

给你一个有序数组 nums ，请你 原地 删除重复出现的元素，使每个元素 只出现一次 ，返回删除后数组的新长度。不要使用额外的数组空间，你必须在 原地 修改输入数组 并在使用 O(1) 额外空间的条件下完成。

```java
/**
 * 双指针解决
 */
public int removeDuplicates(int[] A) {
        // 边界条件判断
        if (A == null || A.length == 0){
            return 0;
	}
    
        int left = 0;
        for (int right = 1; right < A.length; right++){
            // 如果左指针和右指针指向的值一样，说明有重复的，
            // 这个时候，左指针不动，右指针继续往右移。如果他俩
            // 指向的值不一样就把右指针指向的值往前挪
            if (A[left] != A[right]){
                A[++left] = A[right];
            }
	}
        return ++left;
}
```



### 移动零

给定一个数组 `nums`，编写一个函数将所有 `0` 移动到数组的末尾，同时保持非零元素的相对顺序。

**示例：**输入: [0,1,0,3,12]，输出: [1,3,12,0,0]

可以参照双指针的思路解决，指针j是一直往后移动的，如果指向的值不等于0才对他进行操作。而i统计的是前面0的个数，我们可以把j-i看做另一个指针，它是指向前面第一个0的位置，然后我们让j指向的值和j-i指向的值交换


```java
public void moveZeroes(int[] nums) {
    int i = 0;// 统计前面0的个数
    for (int j = 0; j < nums.length; j++) {
        if (nums[j] == 0) {//如果当前数字是0就不操作
            i++;
        } else if (i != 0) {
            //否则，把当前数字放到最前面那个0的位置，然后再把
            //当前位置设为0
            nums[j - i] = nums[j];
            nums[j] = 0;
        }
    }
}
```



## 反转

### 旋转数组

给定一个数组，将数组中的元素向右移动 `k` 个位置，其中 `k` 是非负数。

先反转全部数组，在反转前k个，最后在反转剩余的，如下所示：

![旋转数组](png/Java/旋转数组.png)


```java
public void rotate(int[] nums, int k) {
    int length = nums.length;
    k %= length;
    reverse(nums, 0, length - 1);//先反转全部的元素
    reverse(nums, 0, k - 1);//在反转前k个元素
    reverse(nums, k, length - 1);//接着反转剩余的
}

//把数组中从[start，end]之间的元素两两交换,也就是反转
public void reverse(int[] nums, int start, int end) {
    while (start < end) {
        int temp = nums[start];
        nums[start++] = nums[end];
        nums[end--] = temp;
    }
}
```



## 栈

### 判断字符串括号是否合法

字符串中只有字符'('和')'。合法字符串需要括号可以配对。如：()，()()，(())是合法的。)(，()(，(()是非法的。

**① 栈**

![判断字符串括号是否合法](png/Java/判断字符串括号是否合法.gif)

```java
/**
 * 判断字符串括号是否合法
 *
 * @param s
 * @return
 */
public boolean isValid(String s) {
        // 当字符串本来就是空的时候，我们可以快速返回true
        if (s == null || s.length() == 0) {
            return true;
        }

        // 当字符串长度为奇数的时候，不可能是一个有效的合法字符串
        if (s.length() % 2 == 1) {
            return false;
        }

        // 消除法的主要核心逻辑:
        Stack<Character> t = new Stack<>();
        for (int i = 0; i < s.length(); i++) {
            // 取出字符
            char c = s.charAt(i);
            if (c == '(') {
                // 如果是'('，那么压栈
                t.push(c);
            } else if (c == ')') {
                // 如果是')'，那么就尝试弹栈
                if (t.empty()) {
                    // 如果弹栈失败，那么返回false
                    return false;
                }
                t.pop();
            }
        }

        return t.empty();
}
```

**复杂度分析**：每个字符只入栈一次，出栈一次，所以时间复杂度为 O(N)，而空间复杂度为 O(N)，因为最差情况下可能会把整个字符串都入栈。



**② 栈深度扩展**

如果仔细观察，你会发现，栈中存放的元素是一样的。全部都是左括号'('，除此之外，再也没有别的元素，优化方法如下。**栈中元素都相同时，实际上没有必要使用栈，只需要记录栈中元素个数。** 我们可以通过画图来解决这个问题，如下图所示：

![leftBraceNumber加减](png/Java/leftBraceNumber加减.gif)

```java
/**
 * 判断字符串括号是否合法
 *
 * @param s
 * @return
 */
public boolean isValid(String s) {
        // 当字符串本来就是空的时候，我们可以快速返回true
        if (s == null || s.length() == 0) {
            return true;
        }

        // 当字符串长度为奇数的时候，不可能是一个有效的合法字符串
        if (s.length() % 2 == 1) {
            return false;
        }

        // 消除法的主要核心逻辑:
        int leftBraceNumber = 0;
        for (int i = 0; i < s.length(); i++) {
            // 取出字符
            char c = s.charAt(i);
            if (c == '(') {
                // 如果是'('，那么压栈
                leftBraceNumber++;
            } else if (c == ')') {
                // 如果是')'，那么就尝试弹栈
                if (leftBraceNumber <= 0) {
                    // 如果弹栈失败，那么返回false
                    return false;
                }
                --leftBraceNumber;
            }
        }

        return leftBraceNumber == 0;
}
```

**复杂度分析**：每个字符只入栈一次，出栈一次，所以时间复杂度为 O(N)，而空间复杂度为 O(1)，因为我们已经只用一个变量来记录栈中的内容。



**③ 栈广度扩展**

接下来再来看看如何进行广度扩展。观察题目可以发现，栈中只存放了一个维度的信息：左括号'('和右括号')'。如果栈中的内容变得更加丰富一点，就可以得到下面这道扩展题。给定一个只包括 '('，')'，'{'，'}'，'['，']' 的字符串，判断字符串是否有效。有效字符串需满足：

- 左括号必须用相同类型的右括号闭合
- 左括号必须以正确的顺序闭合
- 注意空字符串可被认为是有效字符串

```java
/**
 * 判断字符串括号是否合法
 *
 * @param s
 * @return
 */
public boolean isValid(String s) {
        if (s == null || s.length() == 0) {
            return true;
        }
        if (s.length() % 2 == 1) {
            return false;
        }

        Stack<Character> t = new Stack<>();
        for (int i = 0; i < s.length(); i++) {
            char c = s.charAt(i);
            if (c == '{' || c == '(' || c == '[') {
                t.push(c);
            } else if (c == '}') {
                if (t.empty() || t.peek() != '{') {
                    return false;
                }
                t.pop();
            } else if (c == ']') {
                if (t.empty() || t.peek() != '[') {
                    return false;
                }
                t.pop();
            } else if (c == ')') {
                if (t.empty() || t.peek() != '(') {
                    return false;
                }
                t.pop();
            } else {
                return false;
            }
        }

        return t.empty();
}
```



### 大鱼吃小鱼

在水中有许多鱼，可以认为这些鱼停放在 x 轴上。再给定两个数组 Size，Dir，Size[i] 表示第 i 条鱼的大小，Dir[i] 表示鱼的方向 （0 表示向左游，1 表示向右游）。这两个数组分别表示鱼的大小和游动的方向，并且两个数组的长度相等。鱼的行为符合以下几个条件:

- 所有的鱼都同时开始游动，每次按照鱼的方向，都游动一个单位距离
- 当方向相对时，大鱼会吃掉小鱼；
- 鱼的大小都不一样。

输入：Size = [4, 3, 2, 1, 5], Dir = [1, 1, 1, 1, 0]，输出：1。请完成以下接口来计算还剩下几条鱼？

![大鱼吃小鱼](png/Java/大鱼吃小鱼.gif)

```java
/**
 * 大鱼吃小鱼
 *
 * @param fishSize
 * @param fishDirection
 * @return
 */
public int solution(int[] fishSize, int[] fishDirection) {
        // 首先拿到鱼的数量: 如果鱼的数量小于等于1，那么直接返回鱼的数量
        final int fishNumber = fishSize.length;
        if (fishNumber <= 1) {
            return fishNumber;
        }

        // 0表示鱼向左游
        final int left = 0;
        // 1表示鱼向右游
        final int right = 1;
        Stack<Integer> t = new Stack<>();
        for (int i = 0; i < fishNumber; i++) {
            // 当前鱼的情况：1，游动的方向；2，大小
            final int curFishDirection = fishDirection[i];
            final int curFishSize = fishSize[i];
            // 当前的鱼是否被栈中的鱼吃掉了
            boolean hasEat = false;
            // 如果栈中还有鱼，并且栈中鱼向右，当前的鱼向左游，那么就会有相遇的可能性
            while (!t.empty() && fishDirection[t.peek()] == right && curFishDirection == left) {
                // 如果栈顶的鱼比较大，那么把新来的吃掉
                if (fishSize[t.peek()] > curFishSize) {
                    hasEat = true;
                    break;
                }
                // 如果栈中的鱼较小，那么会把栈中的鱼吃掉，栈中的鱼被消除，所以需要弹栈。
                t.pop();
            }
            // 如果新来的鱼，没有被吃掉，那么压入栈中。
            if (!hasEat) {
                t.push(i);
            }
        }

        return t.size();
}
```



### 找出数组中右边比我小的元素

一个整数数组 A，找到每个元素：右边第一个比我小的下标位置，没有则用 -1 表示。输入：[5, 2]，输出：[1, -1]。

```java
/**
 * 找出数组中右边比我小的元素
 *
 * @param A
 * @return
 */
public static int[] findRightSmall(int[] A) {
        // 结果数组
        int[] ans = new int[A.length];
        // 注意，栈中的元素记录的是下标
        Stack<Integer> t = new Stack<>();
        for (int i = 0; i < A.length; i++) {
            final int x = A[i];
            // 每个元素都向左遍历栈中的元素完成消除动作
            while (!t.empty() && A[t.peek()] > x) {
                // 消除的时候，记录一下被谁消除了
                ans[t.peek()] = i;
                // 消除时候，值更大的需要从栈中消失
                t.pop();
            }
            // 剩下的入栈
            t.push(i);
        }
        // 栈中剩下的元素，由于没有人能消除他们，因此，只能将结果设置为-1。
        while (!t.empty()) {
            ans[t.peek()] = -1;
            t.pop();
        }
        return ans;
}
```



### 字典序最小的 k 个数的子序列

给定一个正整数数组和 k，要求依次取出 k 个数，输出其中数组的一个子序列，需要满足：1. **长度为 k**；2.**字典序最小**。

输入：nums = [3,5,2,6], k = 2，输出：[2,6]

**解释**：在所有可能的解：{[3,5], [3,2], [3,6], [5,2], [5,6], [2,6]} 中，[2,6] 字典序最小。所谓字典序就是，给定两个数组：x = [x1,x2,x3,x4]，y = [y1,y2,y3,y4]，如果 0 ≤ p < i，xp == yp 且 xi < yi，那么我们认为 x 的字典序小于 y。

```java
/**
 * 字典序最小的 k 个数的子序列
 *
 * @param nums
 * @param k
 * @return
 */
public int[] findSmallSeq(int[] nums, int k) {
        int[] ans = new int[k];
        Stack<Integer> s = new Stack();
        // 这里生成单调栈
        for (int i = 0; i < nums.length; i++) {
            final int x = nums[i];
            final int left = nums.length - i;
            // 注意我们想要提取出k个数，所以注意控制扔掉的数的个数
            while (!s.empty() && (s.size() + left > k) && s.peek() > x) {
                s.pop();
            }
            s.push(x);
        }
        // 如果递增栈里面的数太多，那么我们只需要取出前k个就可以了。
        // 多余的栈中的元素需要扔掉。
        while (s.size() > k) {
            s.pop();
        }
        // 把k个元素取出来，注意这里取的顺序!
        for (int i = k - 1; i >= 0; i--) {
            ans[i] = s.peek();
            s.pop();
        }
        return ans;
}
```



## FIFO队列

### 二叉树的层次遍历（两种方法）

从上到下按层打印二叉树，同一层结点按从左到右的顺序打印，每一层打印到一行。输入：

![二叉树的层次遍历](png/Java/二叉树的层次遍历.png)

输出：[[3], [9, 8], [6, 7]]

```java
// 二叉树结点的定义
public class TreeNode {
      // 树结点中的元素值
      int val = 0;
      // 二叉树结点的左子结点
      TreeNode left = null;
      // 二叉树结点的右子结点
      TreeNode right = null;
}
```

**Queue表示FIFO队列解法**：

![Queue表示FIFO队列解法](png/Java/Queue表示FIFO队列解法.gif)

```java
public List<List<Integer>> levelOrder(TreeNode root) {
        // 生成FIFO队列
        Queue<TreeNode> Q = new LinkedList<>();
        // 如果结点不为空，那么加入FIFO队列
        if (root != null) {
               Q.offer(root);
        }

        // ans用于保存层次遍历的结果
        List<List<Integer>> ans = new LinkedList<>();
        // 开始利用FIFO队列进行层次遍历
        while (Q.size() > 0) {
                // 取出当前层里面元素的个数
                final int qSize = Q.size();
                // 当前层的结果存放于tmp链表中
                List<Integer> tmp = new LinkedList<>();
                // 遍历当前层的每个结点
                for (int i = 0; i < qSize; i++) {
                    // 当前层前面的结点先出队
                    TreeNode cur = Q.poll();
                    // 把结果存放当于当前层中
                    tmp.add(cur.val);
                    // 把下一层的结点入队，注意入队时需要非空才可以入队。
                    if (cur.left != null) {
                        Q.offer(cur.left);
                    }
                    if (cur.right != null) {
                        Q.offer(cur.right);
                    }
                }

                // 把当前层的结果放到返回值里面。
                ans.add(tmp);
        }

        return ans;
}
```

**ArrayList表示FIFO队列解法**：

![ArrayList表示FIFO队列解法](png/Java/ArrayList表示FIFO队列解法.gif)

```java
public List<List<Integer>> levelOrder(TreeNode root) {
        List<List<Integer>> ans = new ArrayList<>();
        // 初始化当前层结点
        List<TreeNode> curLevel = new ArrayList<>();
        // 注意：需要root不空的时候才加到里面。
        if (root != null) {
                curLevel.add(root);
        }
        while (curLevel.size() > 0) {
                // 准备用来存放下一层的结点
                List<TreeNode> nextLevel = new ArrayList<>();
                // 用来存放当前层的结果
                List<Integer> curResult = new ArrayList<>();
                // 遍历当前层的每个结点
                for (TreeNode cur : curLevel) {
                    // 把当前层的值存放到当前结果里面
                    curResult.add(cur.val);
                    // 生成下一层
                    if (cur.left != null) {
                        nextLevel.add(cur.left);
                    }
                    if (cur.right != null) {
                        nextLevel.add(cur.right);
                    }
                }
                // 注意这里的更迭!滚动前进
                curLevel = nextLevel;
                // 把当前层的值放到结果里面
                ans.add(curResult);
        }
        return ans;
}
```



### 循环队列

设计一个可以容纳 k 个元素的循环队列。需要实现以下接口：

```java
public class MyCircularQueue {
        // 参数k表示这个循环队列最多只能容纳k个元素
        public MyCircularQueue(int k){}
        // 将value放到队列中, 成功返回true
        public boolean enQueue(int value){return false;}
        // 删除队首元素，成功返回true
        public boolean deQueue(){return false;}
        // 得到队首元素，如果为空，返回-1
        public int Front(){return 0;}
        // 得到队尾元素，如果队列为空，返回-1
        public int Rear(){return 0;}
        // 看一下循环队列是否为空
        public boolean isEmpty(){return false;}
        // 看一下循环队列是否已放满k个元素
        public boolean isFull(){return false;}
}
```

循环队列的重点在于**循环使用固定空间**，难点在于**控制好 front/rear 两个首尾指示器**。

**方法一：k个元素空间**

只使用 k 个元素的空间，三个变量 front, rear, used 来控制循环队列。标记 k = 6 时，循环队列的三种情况，如下图所示：

![循环队列k个元素空间](png/Java/循环队列k个元素空间.png)

```java
public class MyCircularQueue {
        // 已经使用的元素个数
        private int used = 0;
        // 第一个元素所在位置
        private int front = 0;
        // rear是enQueue可在存放的位置，注意开闭原则， [front, rear)
        private int rear = 0;
        // 循环队列最多可以存放的元素个数
        private int capacity = 0;
        // 循环队列的存储空间
        private int[] a = null;

        public MyCircularQueue(int k) {
            // 初始化循环队列
            capacity = k;
            a = new int[capacity];
        }

        public boolean enQueue(int value) {
            // 如果已经放满了
            if (isFull()) {
                return false;
            }

            // 如果没有放满，那么a[rear]用来存放新进来的元素
            a[rear] = value;
            // rear注意取模
            rear = (rear + 1) % capacity;
            // 已经使用的空间
            used++;
            // 存放成功!
            return true;
        }

        public boolean deQueue() {
            // 如果是一个空队列，当然不能出队
            if (isEmpty()) {
                return false;
            }

            // 第一个元素取出
            int ret = a[front];
            // 注意取模
            front = (front + 1) % capacity;
            // 已经存放的元素减减
            used--;
            // 取出元素成功
            return true;
        }

        public int front() {
            // 如果为空，不能取出队首元素
            if (isEmpty()) {
                return -1;
            }

            // 取出队首元素
            return a[front];
        }

        public int rear() {
            // 如果为空，不能取出队尾元素
            if (isEmpty()) {
                return -1;
            }

            // 注意：这里不能使用rear - 1
            // 需要取模
            int tail = (rear - 1 + capacity) % capacity;
            return a[tail];
        }

        // 队列是否为空
        public boolean isEmpty() {
            return used == 0;
        }

        // 队列是否满了
        public boolean isFull() {
            return used == capacity;
        }
}
```

**复杂度分析**：入队操作与出队操作都是 O(1)。

**方法二：k+1个元素空间**

方法 1 利用 used 变量对满队列和空队列进行了区分。实际上，这种区分方式还有另外一种办法，使用 k+1 个元素的空间，两个变量 front, rear 来控制循环队列的使用。具体如下：

- 在申请数组空间的时候，申请 k + 1 个空间
- 在放满循环队列的时候，必须要保证 rear 与 front 之间有空隙

如下图（此时 k = 5）所示：

![循环队列k+1个元素空间](png/Java/循环队列k+1个元素空间.png)

```java
public class MyCircularQueue {
        // 队列的头部元素所在位置
        private int front = 0;
        // 队列的尾巴，注意我们采用的是前开后闭原则， [front, rear)
        private int rear = 0;
        private int[] a = null;
        private int capacity = 0;

        public MyCircularQueue(int k) {
            // 初始化队列，注意此时队列中元素个数为
            // k + 1
            capacity = k + 1;
            a = new int[k + 1];
        }

        public boolean enQueue(int value) {
            // 如果已经满了，无法入队
            if (isFull()) {
                return false;
            }
            // 把元素放到rear位置
            a[rear] = value;
            // rear向后移动
            rear = (rear + 1) % capacity;
            return true;
        }

        public boolean deQueue() {
            // 如果为空，无法出队
            if (isEmpty()) {
                return false;
            }
            // 出队之后，front要向前移
            front = (front + 1) % capacity;
            return true;
        }

        public int front() {
            // 如果能取出第一个元素，取a[front];
            return isEmpty() ? -1 : a[front];
        }

        public int rear() {
            // 由于我们使用的是前开后闭原则
            // [front, rear)
            // 所以在取最后一个元素时，应该是
            // (rear - 1 + capacity) % capacity;
            int tail = (rear - 1 + capacity) % capacity;
            return isEmpty() ? -1 : a[tail];
        }

        public boolean isEmpty() {
            // 队列是否为空
            return front == rear;
        }

        public boolean isFull() {
            // rear与front之间至少有一个空格
            // 当rear指向这个最后的一个空格时，
            // 队列就已经放满了!
            return (rear + 1) % capacity == front;
        }
}
```

**复杂度分析**：入队与出队操作都是 O(1)。



## 单调队列

单调队列属于**双端队列**的一种。双端队列与 FIFO 队列的区别在于：

- FIFO 队列只能从尾部添加元素，首部弹出元素
- 双端队列可以从首尾两端 push/pop 元素

### 滑动窗口的最大值

给定一个数组和滑动窗口的大小，请找出所有滑动窗口里的最大值。输入：nums = [1,3,-1,-3,5,3], k = 3，输出：[3,3,5,5]。

![滑动窗口的最大值](png/Java/滑动窗口的最大值.png)

```java
public class Solution {

    // 单调队列使用双端队列来实现
    private ArrayDeque<Integer> Q = new ArrayDeque<>();

    // 入队的时候，last方向入队，但是入队的时候
    // 需要保证整个队列的数值是单调的
    // (在这个题里面我们需要是递减的)
    // 并且需要注意，这里是Q.getLast() < val
    // 如果写成Q.getLast() <= val就变成了严格单调递增
    private void push(int val) {
        while (!Q.isEmpty() && Q.getLast() < val) {
            Q.removeLast();
        }
        // 将元素入队
        Q.addLast(val);
    }

    // 出队的时候，要相等的时候才会出队
    private void pop(int val) {
        if (!Q.isEmpty() && Q.getFirst() == val) {
            Q.removeFirst();
        }
    }

    public int[] maxSlidingWindow(int[] nums, int k) {
        List<Integer> ans = new ArrayList<>();
        for (int i = 0; i < nums.length; i++) {
            push(nums[i]);
            // 如果队列中的元素还少于k个
            // 那么这个时候，还不能去取最大值
            if (i < k - 1) {
                continue;
            }
            // 队首元素就是最大值
            ans.add(Q.getFirst());
            // 尝试去移除元素
            pop(nums[i - k + 1]);
        }
        // 将ans转换成为数组返回!
        return ans.stream().mapToInt(Integer::valueOf).toArray();
    }

}
```

**复杂度分析**：每个元素都只入队一次，出队一次，每次入队与出队都是 O(1) 的复杂度，因此整个算法的复杂度为 O(n)。



### 捡金币游戏

给定一个数组 A[]，每个位置 i 放置了金币 A[i]，小明从 A[0] 出发。当小明走到 A[i] 的时候，下一步他可以选择 A[i+1, i+k]（当然，不能超出数组边界）。每个位置一旦被选择，将会把那个位置的金币收走（如果为负数，就要交出金币）。请问，最多能收集多少金币？

输入：[1,-1,-100,-1000,100,3], k = 2，输出：4。

**解释**：从 A[0] = 1 出发，收获金币 1。下一步走往 A[2] = -100, 收获金币 -100。再下一步走到 A[4] = 100，收获金币 100，最后走到 A[5] = 3，收获金币 3。最多收获 1 - 100 + 100 + 3 = 4。没有比这个更好的走法了。

```java
public class Solution {
    public int maxResult(int[] A, int k) {
        // 处理掉各种边界条件!
        if (A == null || A.length == 0 || k <= 0) {
            return 0;
        }
        final int N = A.length;
        // 每个位置可以收集到的金币数目
        int[] get = new int[N];
        // 单调队列，这里并不是严格递减
        ArrayDeque<Integer> Q = new ArrayDeque<Integer>();
        for (int i = 0; i < N; i++) {
            // 在取最大值之前，需要保证单调队列中都是有效值。
            // 也就是都在区间里面的值
            // 当要求get[i]的时候，
            // 单调队列中应该是只能保存[i-k, i-1]这个范围
            if (i - k > 0) {
                if (!Q.isEmpty() && Q.getFirst() == get[i - k - 1]) {
                    Q.removeFirst();
                }
            }
            // 从单调队列中取得较大值
            int old = Q.isEmpty() ? 0 : Q.getFirst();
            get[i] = old + A[i];
            // 入队的时候，采用单调队列入队
            while (!Q.isEmpty() && Q.getLast() < get[i]) {
                Q.removeLast();
            }
            Q.addLast(get[i]);
        }
        return get[N - 1];
    }
    
}
```

**复杂度分析**：每个元素只入队一次，出队一次，每次入队与出队复杂度都是 O(n)。因此，时间复杂度为 O(n)，空间复杂度为 O(n)。

# 参考资料

参考书籍：《Java数据结构和算法》

参考资料：动画排序http://www.donghuasuanfa.com/sort

十大经典排序算法动画与解析

算法思想：https://www.cnblogs.com/steven_oyj/archive/2010/05/22/1741378.html