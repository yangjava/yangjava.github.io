---
layout: post
categories: [C++]
description: none
keywords: C++
---
# C++编程1STL_string容器

## string概念
string是STL的字符串类型，通常用来表示字符串。而在使用string之前，字符串通常是用char*表示的。string与char*都可以用来表示字符串，那么二者有什么区别。

string和char*的比较:

string是一个类, char*是一个指向字符的指针。
string封装了char*，管理这个字符串，是一个char*型的容器。
string不用考虑内存释放和越界。
string管理char*所分配的内存。每一次string的复制，取值都由string类负责维护，不用担心复制越界和取值越界等。
string提供了一系列的字符串操作函数
查找find，拷贝copy，删除erase，替换replace，插入insert
```
 //string 转 char*
 string str_1="string";
 const char* cstr_1=str.c_str();
 //char* 转 string
 char* cstr_2="char";
 string str_2(cstr);
```

## string的构造函数
默认构造函数：string(); //构造一个空的字符串string s1。
构造函数：string(const string &str); //构造一个与str一样的string。如string s1(s2)。
带参数的构造函数：
string(const char *s); //用字符串s初始化
string(int n,char c); //用n个字符c初始化
```
     string s1; //调用无参构造
     string s2(10, 'a');
     string s3("abcdefg");
     string s4(s3); //拷贝构造
 
     cout << s1 << endl;
     cout << s2 << endl;
     cout << s3 << endl;
     cout << s4 << endl;
 /*
 结果：
 
 aaaaaaaaaa
 abcdefg
 abcdefg
 */
```

## string的存取字符操作
string类的字符操作：

const char &operator[] (int n) const; //通过[]方式取字符
const char &at(int n) const; //通过at方法获取字符
char &operator[] (int n);
char &at(int n);
operator[]和at()均返回当前字符串中第n个字符，但二者是有区别的。

主要区别在于at()在越界时会抛出异常，[]在刚好越界时会返回(char)0，再继续越界时，编译器直接出错。如果你的程序希望可以通过try,catch捕获异常，建议采用at()。
```
     string s1 = "abcdefg";
 
     //重载[]操作符
     for (int i = 0; i < s1.size(); i++) {
         cout << s1[i] << " ";
     }
     cout << endl;
 
     //at成员函数
     for (int i = 0; i < s1.size(); i++) {
         cout << s1.at(i) << " ";
     }
     cout << endl;
 
     //区别：[]方式 如果访问越界，直接挂了
     //at方式 访问越界 抛异常out_of_range
 
     try {
         //cout << s1[100] << endl;
         cout << s1.at(100) << endl;
     }
     catch (...) {
         cout << "越界!" << endl;
     }
 /*
 结果：
 a b c d e f g
 a b c d e f g
 越界!
 */
```
从string取得const char*的操作
const char *c_str() const; //返回一个以'\0'结尾的字符串的首地址
```
 //string 转 char*
 string str_1="string";
 const char *cstr_1=str.c_str();
```
把string拷贝到char*指向的内存空间的操作
int copy(char *s, int n, int pos=0) const;

把当前串中以pos开始的n个字符拷贝到以s为起始位置的字符数组中，返回实际拷贝的数目。注意要保证s所指向的空间足够大以容纳当前字符串，不然会越界。
```
     string s1 = "abcdefg";
     int n = 5;pose
     int pose = 3;
 
     char* s2 = (char*)malloc(n*sizeof(char));
     if(s1.copy(s2, n, pose))
         cout << s2 ;
     else cout<<"error"<<endl;
 /*
 结果：
 def
 */
```
string的长度
int length() const; //返回当前字符串的长度。长度不包括字符串结尾的'\0'。

bool empty() const; //当前字符串是否为空
```
     string s1 = "abcdefg";
     string s2 = "";
 
     int s1_length=s1.length();  
     bool s1_empty = s1.empty();
     int s2_length = s2.length();  
     bool s2_empty = s2.empty();
     
     cout << s1_length << "     " << s1_empty << endl;
     cout << s2_length << "     " << s2_empty << endl;
 /*
 结果：
 7     0
 0     1
 */
```

string的赋值
string &operator=(const string &s);//把字符串s赋给当前的字符串

string &assign(const char *s); //把字符串s赋给当前的字符串

string &assign(const char *s, int n); //把字符串s的前n个字符赋给当前的字符串

string &assign(const string &s); //把字符串s赋给当前字符串

string &assign(int n,char c); //用n个字符c赋给当前字符串

string &assign(const string &s,int start, int n); //把字符串s中从start开始的n个字符赋给当前字符串
```
     string s1;
     string s2("appp");
     s1 = "abcdef";
     cout << s1 << endl;
     s1 = s2;
     cout << s1 << endl;
     s1 = 'a';
     cout << s1 << endl;
 
     //成员方法assign
     s1.assign("jkl");
     cout << s1 << endl;
 /*
 结果：
 abcdef
 appp
 a
 jkl
 */
```
string字符串连接
string &operator+=(const string &s); //把字符串s连接到当前字符串结尾

string &operator+=(const char *s);//把字符串s连接到当前字符串结尾

string &append(const char *s); //把字符串s连接到当前字符串结尾

string &append(const char *s,int n); //把字符串s的前n个字符连接到当前字符串结尾

string &append(const string &s); //同operator+=()

string &append(const string &s,int pos, int n);//把字符串s中从pos开始的n个字符连接到当前字符串结尾

string &append(int n, char c); //在当前字符串结尾添加n个字符c
```
     string s = "abcd";
     string s2 = "1111";
     s += "abcd";
     s += s2;
     cout << s << endl;
 
     string s3 = "2222";
     s2.append(s3);
     cout << s2 << endl;
 
     string s4 = s2 + s3;
     cout << s4 << endl;
 /*
 结果：
 abcdabcd1111
 11112222
 111122222222
 */
```
string的比较
int compare(const string &s) const; //与字符串s比较

int compare(const char *s) const; //与字符串s比较

compare函数在>时返回 1，<时返回 -1，==时返回 0。比较区分大小写，比较时参考字典顺序，排越前面的越小。大写的A比小写的a小。
```
     string s1 = "abcd";
     string s2 = "abce";
     if (s1.compare(s2)==0) {
         cout << "字符串相等!" << endl;
     }
     else {
         cout << "字符串不相等!" << endl;
     }
```
string的子串
string substr(int pos=0, int n=npos) const; //返回由pos开始的n个字符组成的子字符串
```
     string s = "abcdefg";
     string mysubstr = s.substr(1, 3);
     cout << mysubstr << endl;
 /*
 结果：
 bcd
 */
```
string的查找和替换
查找
int find(char c,int pos=0) const; //从pos开始查找字符c在当前字符串的位置

int find(const char *s, int pos=0) const; //从pos开始查找字符串s在当前字符串的位置

int find(const string &s, int pos=0) const; //从pos开始查找字符串s在当前字符串中的位置

//find函数如果查找不到，就返回-1

int rfind(char c, int pos=npos) const; //从pos开始从后向前查找字符c在当前字符串中的位置

int rfind(const char *s, int pos=npos) const;

int rfind(const string &s, int pos=npos) const;

//rfind是反向查找的意思，如果查找不到， 返回-1

替换
string &replace(int pos, int n, const char *s);//删除从pos开始的n个字符，然后在pos处插入串s

string &replace(int pos, int n, const string &s); //删除从pos开始的n个字符，然后在pos处插入串s

void swap(string &s2); //交换当前字符串与s2的值
```
     // 字符串的查找和替换
     string s1 = "wbm hello wbm 111 wbm 222 wbm 333";
     size_t index = s1.find("wbm", 0);
     cout << "index: " << index;  
 
     //求itcast出现的次数
     size_t offindex = s1.find("wbm", 0);
     while (offindex != string::npos){
 
          cout << "在下标index: " << offindex << "找到wbm\n";
          offindex = offindex + 1;
          offindex = s1.find("wbm", offindex);
     } 
 
     //替换 
     string s2 = "wbm hello wbm 111 wbm 222 wbm 333";
     s2.replace(0, 3, "wbm");
     cout << s2 << endl; 
 
     //求itcast出现的次数
     offindex = s2.find("wbm", 0);
     while (offindex != string::npos){
 
          cout << "在下标index: " << offindex << "找到wbm\n";
          s2.replace(offindex, 3, "WBM");
          offindex = offindex + 1;
          offindex = s1.find("wbm", offindex);
     }
     cout << "替换以后的s2：" << s2 << endl; 
 /*
 结果：
 index: 0在下标index: 0找到wbm
 在下标index: 10找到wbm
 在下标index: 18找到wbm
 在下标index: 26找到wbm
 wbm hello wbm 111 wbm 222 wbm 333
 在下标index: 0找到wbm
 在下标index: 10找到wbm
 在下标index: 18找到wbm
 在下标index: 26找到wbm
 替换以后的s2：WBM hello WBM 111 WBM 222 WBM 333
 */
```
String的区间删除和插入
string &insert(int pos, const char *s);

string &insert(int pos, const string &s);//在pos位置插入字符串s

string &insert(int pos, int n, char c); //在pos位置 插入n个字符c

string &erase(int pos=0, int n=npos); //删除pos开始的n个字符，返回修改后的字符串
```
     string s = "abcdefg";
     s.insert(3, "111");
     cout << s << endl;
 
     s.erase(0, 2);
     cout << s << endl;
 /*
 结果：
 abc111defg
 c111defg
 */
```