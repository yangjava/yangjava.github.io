---
layout: post
categories: [C++]
description: none
keywords: C++
---
# C++编程 STL_vector容器

## Vector容器简介
vector是将元素置于一个动态数组中加以管理的容器。

vector可以随机存取元素（支持索引值直接存取， 用[]操作符或at()方法）。

vector尾部添加或移除元素非常快速，但是在中部或头部插入元素或移除元素比较费时。

## vector对象的默认构造
vector采用模板类实现，vector对象的默认构造形式：vector<T> vecT;
```
vector<int> vecInt;     //一个存放int的vector容器。
 
 vector<float> vecFloat;    //一个存放float的vector容器。
 
 vector<string> vecString;   //一个存放string的vector容器。
 
 ...                  //尖括号内还可以设置指针类型或自定义类型。
 
 Class CA{};
 
 vector<CA*> vecpCA;      //用于存放CA对象的指针的vector容器。
 
 vector<CA> vecCA;       //用于存放CA对象的vector容器。由于容器元素的存放是按值复制的方式进行的，所以此时CA必须提供CA的拷贝构造函数，以保证CA对象间拷贝正常。
```

## vector对象的带参数构造
vector(beg,end); //构造函数将[beg, end)区间中的元素拷贝给本身。注意该区间是左闭右开的区间。

vector(n,elem); //构造函数将n个elem拷贝给本身。

vector(const vector &vec); //拷贝构造函数
```
 void printVector(vector<int>& v) {
     for (vector<int>::iterator it = v.begin(); it != v.end(); it++) {
         cout << *it << " ";
     }
     cout << endl;
 }
```

```
     vector<int> vl;//默认构造
 
     int arr[] = { 10, 20, 30, 40 };
     vector<int> v2(arr, arr + sizeof(arr) / sizeof(int));
     vector<int> v3(v2.begin(), v2.begin()+3);
     vector<int> v4(v3);
     vector<int> v5(4,5);
 
     printVector(v2);
     printVector(v3);
     printVector(v4);
     printVector(v5);
 /*
 结果：
 10 20 30 40
 10 20 30
 10 20 30
 5 5 5 5
 */
```
vector的赋值
vector.assign(beg,end); //将[beg, end)区间中的数据拷贝赋值给本身。注意该区间是左闭右开的区间。

vector.assign(n,elem); //将n个elem拷贝赋值给本身。

vector& operator=(const vector &vec); //重载等号操作符

vector.swap(vec); // 将vec与本身的元素互换。
```
     int arr[] = { 10, 20, 30, 40 };
     vector<int> v1(arr, arr + sizeof(arr) / sizeof(int));//默认构造
 
     //成员方法
     vector<int> v2;
     v2.assign(v1.begin(), v1.end());
 
     //重载=
     vector<int> v3;
     v3 = v2;
 
     int arr1[] = { 100, 200, 300, 400 };
     vector<int> v4(arr1, arr1 + sizeof(arr) / sizeof(int));//默认构造
 
     printVector(v1);
     printVector(v2);
     printVector(v3);
     printVector(v4);
 
     cout << "------------------" << endl;
 
     v4.swap(v1);
     printVector(v1);
     printVector(v2);
     printVector(v3);
     printVector(v4);
 /*
 结果：
 10 20 30 40
 10 20 30 40
 10 20 30 40
 100 200 300 400
 ------------------
 100 200 300 400
 10 20 30 40
 10 20 30 40
 10 20 30 40
 */
```
vector的大小
vector.size(); //返回容器中元素的个数

vector.empty(); //判断容器是否为空

vector.resize(num); //重新指定容器的长度为num，若容器变长，则以默认值填充新位置。如果容器变短，则末尾超出容器长度的元素被删除。

vector.resize(num, elem); //重新指定容器的长度为num，若容器变长，则以elem值填充新位置。如果容器变短，则末尾超出容器长度的元素被删除。

capacity();//容器的容量

reserve(int len);//容器预留len个元素长度，预留位置不初始化，元素不可访问。
```
     int arr1[] = { 100, 200, 300, 400 };
     vector<int> v4(arr1, arr1 + sizeof(arr1) / sizeof(int));//默认构造
 
     cout << "size：" << v4.size() << endl;
     if (v4.empty()) {
         cout << "空！" << endl;
     }
     else {
         cout << "不空！" << endl;
     }
 
     printVector(v4);
     v4.resize(2);
     printVector(v4);
     v4.resize(6);
     printVector(v4);
     v4.resize(8, 1);
     printVector(v4);
 
     for (int i = 0; i < 10000; i++) {
         v4.push_back(i);
     }
     cout << "size：" << v4.size() << endl;  
     cout << "容量:" << v4.capacity() << endl;//容量不一定等于size
 /*
 结果：
 size：4
 不空！
 100 200 300 400
 100 200
 100 200 0 0 0 0
 100 200 0 0 0 0 1 1
 size：10008
 容量:12138
 */
```
vector末尾的添加移除操作
vector.push_back(); //在容器尾部加入一个元素

vector.pop_back();; //移除容器中最后一个元素
```
     int arr1[] = { 100, 200, 300, 400 };
     vector<int> v1(arr1, arr1 + sizeof(arr1) / sizeof(int));//默认构造
 
     v1.push_back(500);
     v1.push_back(600);
     v1.push_back(700);
     v1.pop_back();
     printVector(v1);
 /*
 结果：
 100 200 300 400 500 600100 200 300 400 500 600
 */

```
vector的数据存取
vector.at(int idx); //返回索引idx所指的数据，如果idx越界，抛出out_of_range异常。

vector[int idx];//返回索引idx所指的数据，越界时，运行直接报错

vector.front();//返回容器中第一个数据元素

vector.back();//返回容器中最后一个数据元素
```
     int arr1[] = { 100, 200, 300, 400 };
     vector<int> v4(arr1, arr1 + sizeof(arr1) / sizeof(int));//默认构造
 
     //区别: at抛异常 []不抛异常
     for (int i = 0; i < v4.size(); i++) {
         cout << v4[i] << " ";
     }
     cout << endl;
 
     for (int i = 0; i < v4.size(); i++) {
         cout << v4.at(i) << " ";
     }
     cout << endl;
 
     cout << "front:" << v4.front() << endl;
     cout << "back:" << v4.back() << endl;
 /*
 结果：
 100 200 300 400
 100 200 300 400
 front:100
 back:400
 */
```
vector的插入和删除
vector.insert(pos,elem); //在pos位置插入一个elem元素的拷贝，返回新数据的位置。

vector.insert(pos,n,elem); //在pos位置插入n个elem数据，无返回值。

vector.insert(pos,beg,end); //在pos位置插入[beg,end)区间的数据，无返回值

vector.clear(); //移除容器的所有数据

vector.erase(beg,end); //删除[beg,end)区间的数据，返回下一个数据的位置。

vector.erase(pos); //删除pos位置的数据，返回下一个数据的位置。
```
     vector<int> v;
     v.push_back(10);
     v.push_back(20);
     //头插法
     v.insert(v.begin(), 30);
     v.insert(v.end(), 40);
 ​
     v.insert(v.begin() + 2, 100); //vector支持随机访问
 ​
     //支持数组下标，一般都支持随机访问
     //迭代器可以直接+2 +3 -2 -5操作
     printVector(v);
 ​
     //删除
     v.erase(v.begin());
     printVector(v);
     v.erase(v.begin() + 1, v.end());
     printVector(v);
     v.clear();
     cout << "size:" << v.size() << endl;
 /*
 结果：
 30 10 100 20 40
 10 100 20 40
 10
 size:0
 */
```
巧用swap，收缩内存空间
vector<T>(x).swap(x); //其中，x 指当前要操作的容器，T 为该容器存储元素的类型。
```
	//vector添加元素 他会自动增长 你删除元素时候，不会自动减少

	vector<int> v;
	for (int i = 0; i < 100000; i++) {
		v.push_back(i);
	}

	cout << "size:" << v.size() << endl;
	cout << "capacity:" << v.capacity() << endl;

	v.resize(10);
	cout << "--------------" << endl;
	cout << "size:" << v.size() << endl;
	cout << "capacity:" << v.capacity() << endl;

	//收缩空间
	vector<int>(v).swap(v);

	cout << "--------------" << endl;
	cout << "size:" << v.size() << endl;
	cout << "capacity:" << v.capacity() << endl;
/*
结果：
size:100000
capacity:138255
--------------
size:10
capacity:138255
--------------
size:10
capacity:10
*/
```
reserve 预留空间
vector的reserve增加了vector的capacity，但是它的size没有改变！而resize改变了vector的capacity同时也增加了它的size！

原因如下：

reserve是容器预留空间，但在空间内不真正创建元素对象，所以在没有添加新的对象之前，不能引用容器内的元素。加入新的元素时，要调用push_back()/insert()函数。
resize是改变容器的大小，且在创建对象，因此，调用这个函数之后，就可以引用容器内的对象了，因此当加入新的元素时，用operator[]操作符，或者用迭代器来引用元素对象。此时再调用push_back()函数，是加在这个新的空间后面的。
```
     int num = 0;
     int* address = NULL;
 ​
     vector<int> v;
     v.reserve(100000);
     for (int i = 0; i < 100000; i++) {
         v.push_back(i);
         if (address != &(v[0])) {
             address = &(v[0]);
             num++;
         }
     }
 ​
     cout << "num:" << num << endl;//申请num次空间
 /*
 结果：
 num:1
 */
```
如果一个vector使用默认的capacity，那么在push_back操作的时候，会根据添加元素的数量，动态的自动分配空间，2^n递增；如果声明vector的时候，显式的使用capacity(size_type n)来指定vector的容量，那么在push_back的过程中（元素数量不超过n），vector不会自动分配空间，提高程序效率。