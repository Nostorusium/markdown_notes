# C++从入门到放弃

## hello,world!

```
#include<iostream>
int main() {
    using namespace std;
    cout << "hello";
    cout << endl;
    cout << "world";
    return 0;
}
```
### 再识头文件

头文件.h文件,与C中的用法一样,由#include完成
C++与C的编译过程都包括了**预处理阶段**
预处理阶段负责展开宏文件
那么在**展开宏文件**时,我们在展开什么?
头文件的提出是为了解决**函数声明问题**:
即专门整一个文件,放在头部事先声明,这样后文的函数定义就可以随意摆放了
头文件的本质就是一个**文本文件**,没有任何特殊之处,#include的功能就是原封不动的把其内容贴进来.同理他也可以定义常量,变量与各种数据结构.

不同于**库文件**,头文件只负责声明,不负责实现。编译器先通过头文件再去访问库文件。

C的传统是用.h后缀表示头文件C++中可以继续沿用C的.h头文件
而C++的头文件没有扩展名.h
由于部分C头文件被转化为C++版本,去掉.h 加上前缀c表示来自C语言
如C++版本的**math.h**为**cmath**

旧式的C++风格也曾有.h拓展名,但不能使用namespacec
而对于新式的头文件,可以使用namespace

### 名称空间

假如两个库里使用了同样的方法名该怎么办?
原先的一个办法是在所有的函数,常量名前面带上自己的姓名当前缀,但这很麻烦
因此提出名称空间namespace
```
using namespace std;
```
using是一个编译指令,我们在此处先接受这个指令,日后再考虑他

这种**特性**允许作者把产品封装在一个叫做名称空间的单元中,来区分相同名称的方法
```
//例如
Microsoft::func("test");
Konami::func("fuck konami");
```

按照这种方式,C++编译器的标准组件都被放在名称空间std中
仅当头文件没有扩展名h时才如此.
这意味着iostream中定义的cout实际上是std::cout,endl实际上是std::endl
我们可以省略using,然而这意味着在使用时需要显式给出std:count<<"Hello,world!";

using namespace std使得std名称空间中所有名称都可用,这是一种偷懒的做法,在大型项目中可能存在潜在的问题.
更好的方法是只使所需的名称可用
```
using std::cout;
using std::endl;
using std::cin;
```

### 输入输出

```
cout << "hello,world!";
```
<<符号表示把这个字符串发给cout
该符号指出了信息流动的路径
从概念上来看,输出是一个**流**
即从程序流出的一系列字符,cout对象表示这种流,其属性在iostream中定义
cout的对象属性包括一个插入运算符<< 将右侧信息插入到流中.

>在C中,<<是移位运算符,在C++中重载,使同一个运算符根据上下文有不同的含义
>C本身也有一些重载的情况,如&既可以表示取地址也可以表示按位与 *即使乘法又是解引用

1. 控制符endl
   endl是一个特殊得到C++符号,表示重启一行
   诸如endl对于cout有特殊含义的特殊符号叫**控制符**(manipulator)
   在头文件iostream定义,名称空间为std
2. 换行符\n
   这是在C中的旧式方法
   生成空行: cout <<"\n";与cout<< endl;作用相同
   一个差别是endl确保程序继续运行前刷新输出,立即显示在屏幕上
   而\n不能提供这样的保证,这意味着某些系统中空行可能在输入信息后才出现

### 格式化与C++风格

有些如FORTRAN语言是面向行的,即每条语句占一行
对于这些语言回车的作用是将语句分开
而在C++中分号表示了语句的末尾,回车的作用和空格和制表符相同
可以把一条语句放在几行上,也可以一行写好几个语句 他们都是合法的写法
但是在C与C++中,空格回车制表符不能放在元素与字符串中间

多数程序员遵循以下风格:
- 一行一条语句
- 花括号各占一行
- 语句相对于花括号缩进
- 函数名称紧跟的圆括号没有空白

### C++语句

C和Pascal中,所有的变量声明通常位于函数的开始位置,这样就不用到处找变量类型
但在C++中通常的做法是在第一次使用前声明

C与C++允许连续赋值:
```
int a;int b;int c;
a=b=c=114514;
```
赋值从右向左

1. cout
   在C中printf("%s %d","114514",num114514)需要指出%s与%d
   cout的智能之处在于插入运算符会根据其后的数据类型调整行为cout << num114514;
2. cin
   ```
   int num;
   cout << "ruarua" << endl;
   cin >> num;
   cout << num;
   ```
   信息从cin流流入num
3. cout拼接
   C++允许这样的拼接
   ```
   cout << "the value is" << elder << "years old" << endl;
   ```
   但强烈建议写成:
   ```
   cout << "the value is"
        << elder
        << "years old"
        << endl;
   ```

### 类

现在看来cout是一个ostream对象
ostream类描述了ostream对象表示的数据和可执行的操作
cin则是istream类对象,他们都在iostream中定义

从技术上说istream类与ostream类没有被内置到C++编译器中,而是语言标准所指定.
程序员甚至可以对他们进行修改,但这不是一个好主意.

### 用户定义函数

就像使用库函数前需要提供原型一样,函数使用前需要提供函数头

```
void helloworld(void);

int main()
{
    using namespace std;
    helloworld();
    cout << "func ends" << endl;
    return 0;
}

void helloworld() //不写void也会默认是不接受参数的隐式声明
{
    cout << "Hello,world!" << endl;
}
```

### 多函数程序中的using
```
using namespace std;
void helloworld();

int main() {
	helloworld();
	cout << "func ends" << endl;
	return 0;
}

void helloworld()
{
	cout << "hello,world!" << endl;
}
```
把using namespace std放在外面,且位于两个函数的前面,可以让所有函数都应用std名称空间
现在通行的理念是,是让需要访问名称空间std的函数访问它是更好的选择
如果只有某个函数使用std,那就没有必要让别的函数也能够访问名称空间std

## 处理数据

### 取名

以**两个下划线打头**的,或**下划线加大写字母打头**的名称全部保留给实现使用
**一个下划线开头**的名称保留给实现,做全局标识符

命名方案:
- my_onions C风格使用**下划线分割**
- myEyeTooth Pascal**驼峰风格**
- nMyWeight **前缀风格** n表示整数
  - intMyWeight 多了几个字母的前缀风格
  - sAString s表示字符串;p为指针;b为布尔值

### 整数字面值

默认情况下cout以十进制显示整数
如果要以八进制和十六进制输出:
```
using namespace std;

int main()
{
    int num = 114514;
    cout << "num=" << num << endl;
    cout << hex;    //含义是改变cout显示整数的方式
    cout << "num=0x" << num << endl;   
    cout << oct << "num=0b" << num << endl; //也可以拼接
    cout << dec;    //还能变回去
    cout << "num=" << num << endl;
}
```
cout中默认输出int,加后缀表示其他类型
```
cout << 114514ul << 1919810uLL ;
//大小写与顺序不限,l建议大写L防止与1看混
//u,unsigned;l,long;
```

### 字符char

编程语言通过字母的数值解码来存储字母
因此char实际上是另一种整型,1B就可以表示所有的基本符号
C++将字符表示为整数,不必使用笨重的转换函数在ASCII码与字符之间转换

cin输入char类型变量时给整数也被认为是字符
cout输出char类型变量时以字符输出

cout.put()表示输出一个字符,用来代替<<
这种特性与历史有关,在C++的Release2.0版本之前cout把字符变量视为字符,字符常量显示为数字
而C++的早期版本与C一样把字符常量也存储为int类型,即'M'对应的编码77被存储在4B单元里 而char一般占1B
```
cout << '$';        //将输出ASCII码
cout.put('$');      //将输出$
```
在Release2.0后C++将字符常量存储为char类型,不再有int类型了
```
char char_value;
cin >> char_value;
int int_value = char_value;
cout << "input char is " << char_value << " and the ASCII code is " << int_value << endl;
```