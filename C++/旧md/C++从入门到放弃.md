# C++从入门到放弃

## Chapter 1

(前略)

### 字符串

只有以 \0 结尾才被认为是字符串,否则认为是字符数组
要计算字符串所需的最短数组需要加上结尾的空字符
处理字符串的函数根据\0的位置进行处理
```
char string[] = "哼 啊啊啊啊啊";
```
编译器会自动计算长度
"this is a string"会返回这串字符串的地址,所以:
```
char aChar = "C";
```
试图把一个地址赋给aChar,是不合法的.

strlen()计算字符串长度,不包括空字符,在头文件cstring(string.h)定义

```
char string1[10];
char string2[10];
cout << "enter your name:\n";
cin >> string1;
cout << "enter something else:\n";
cin >> string2;
cout << string1 << string2;
```
这段程序的输出将是:
```
enter your name:
田所 浩二
enter something else:
田所浩二
```
cin以**空格,制表符,换行符**来确定字符串的结束
田所存入string1,浩二存入string2.
所以在进行第二个输入前,第二个cin就读入了缓冲区中剩下的浩二
且另一个问题是容易造成数组溢出
为了防止缓冲区越界,需要使用cin的比较高级特性,以后介绍

每次读取一个单词通常不是最好的选择,我们需要采用面向行的方法
istream中的类(如cin)提供了一些面向行的类成员函数
getline()
get()

getline()读取一行,以回车键输入的换行符确定结尾
可使用cin.getline();
有两个参数,arg1为存入数组名称,arg2为需要读取的字符数,包括空字符.如果arg2为10,则最多读9个字符
所以做以下改进:
```
char string1[10];
char string2[10];
cout << "enter your name:\n";
cin.getline(string1,10);
cout << "enter something else:\n";
cin.getline(string2,10);
cout << string1 << string2;
```
getline会自动把换行符替换成\0存入字符串数组.
而get并不读取换行符,他会被留在输入队列中.
```
cin.get(string1,10);
cin.get(string2,10);
cin.get(string3,10);
```
第一个get会读入田所 浩二,不读入换行符
第二个get则会上来就读到换行符,并认为输入结束,后续的get将卡死在这里.
但有一个变体,不给任何参数的get会读取下一个字符
```
cin.get(string1,10);
cin.get(); //读换行符
cin.get(string2,10);
```
cin.get(name,ARSize);返回一个ccin对象
这允许了拼接调用:
```
cin.get(string1,10).get(); //先读,顺带处理换行符
```

get()没有getline()方便,但可以更仔细的检查一行输入:
当数组被填满或一行输入结束,输入停止,此时检查下一个输入
如果为换行符,说明一行确实输入完毕了
如果不是,为待输入的字符,说明数组被填满了.

当get读取了一个空行,getline输出数组满了,会设置失效位关闭后续的输入.
可以使用cin.clear()来恢复
后续会进一步探讨

### string类

和C比起来神中神的天下无敌的C++ String类!
头文件: string
命名空间: std::string

- 可被简单声明为变量
- 可以用C风格初始化string
- 可以用cin输入到string对象内
- 可以用cout输出string对象
- 可用数组表示法单独访问string中的字符
- 允许用string赋值string
- 允许用+拼接两个string;

在string类出现之前,我们使用C库中的函数完成初始化,赋值等操作
cstring(string.n)提供:
strcpy(target,from);  //copy
strcat(target,from);  //append 
而string类只需:
string2 = string1;
string3 = string1 + string2;

C库提供了strncat和strncpy来指出目标数组最大允许长度的第三个参数,但这进一步增加了复杂性
string类有自动调节大小的功能,可以一定程度上避免溢出.

在使用string读入一行时句法有所不同:
getline(cin,string); 此时无需指定大小参数
此处的getline不是cin.getline(arg1,arg2) 不是istream的类方法
在string提出之前,istream没有考虑到设计string类型的类方法
而即便如此
```
cin >> string;
cout << string;
```
依然是可行的,后续介绍友元函数的技术

### 结构

```
struct student{
    char name[10];
    int old;
};
```
创建结构变量:
```
struct student tianSuoHaoEr;
student stu2;

tianSuoHaoEr.name = "田所浩二";
tianSuoHaoEr.old = 24;
```
和C不同,C++允许省略struct关键字

C++提倡使用外部结构声明,即把结构声明放在函数外面
可以这样初始化:
```
//实际上不推荐使用using namespace 会造成命名空间污染
//using namespace std;

using std::string;

struct human
{
    string name;
    string carrer;
    int old;
}name1,name2; //此处可以直接定义结构变量

int main()
{
    human tianSuoHaoEr =
    {
        "田所浩二",
        "学生",
        24
    };
    return 0;
}

```
如果不进行初始化,每个成员会被设置为0.

结构数组:
```
student stu[114514];
stu[0] = {
    "田所浩二",
    "学生",
    24
};
或者
student stu[2] = {
    {"田所浩二","学生",24},
    {"野兽先辈","学生",24}
};
```

### 共用体union

```
union wtf{
    int iVal;
    long lVal;
    double dVal;
}
```
共用体通常用于节省内存空间。
共用体一次只能存储一个值,它的长度为最大成员的长度
在这个例子下wtf时而为int,时而为long,时而为double.
当数据项使用两种或更多格式(但不会同时使用)时可以使用union节省空间
如有的商品ID为int,有的ID为字符串

如果不取名,称匿名共用体

### 枚举enum

提供了一种创建符号常量的方式代替const
```
enum spectrum{red,orange,yellow};
```
默认第一个枚举值red为0,以此类推
可以显式地指定整数值覆盖默认值
```
enum color{red=0,bule=1};
```
可以声明变量:
```
color aColor;
color = 114514;     //非法,114514整形不是枚举量
color = red;        //合法,只能用枚举量赋值
color++;            //非法,枚举量没有算术运算
int num = red;      //合法
int num2 = 1 + red;   //red强转为int类型
```

### 指针与自由空间

>OOP强调在运行阶段进行决策,而不是编译时决策.
考虑创建一个数组,
在C++中声明数组必须指定数组的长度,长度在编译阶段就确定好了
而OOP希望将这样的决策推延到运行阶段进行,根据需求确定长度
C++采用的方法是使用关键字new请求正确数量的内存以及使用指针来跟踪新分配的内存

声明指针有两种风格:
```
int *ptr;
int* ptr;
```
前者强调ptr的类型是int,后者强调ptr是一个int类型的指针
两边空格都是可选的,int*ptr;也是合法的

使用指针是危险的,尤其是解引用:
```
int* ptr;
*ptr = 114514;
```
ptr没有被初始化,它将指向未知的地址,对它的解引用是极其危险的
这提供了使用指针的**原则**:**解引用前必须初始化**

尽管计算机把地址当成整数来处理,但指针不是整形
```
int* ptr;
ptr = 0xB8000000;           //invalid
ptr = (int *) 0xB8000000;   //valid
```
这会导致类型不匹配错误,需要强转

#### new

在C中使用malloc()分配内存
C++仍然可以,但有更好的办法:new
```
int* ptr = new int;
```
new运算符根据类型确定需要多少字节的内存,找到这样的内存并返回其地址
但这个int值没有名字,我们只能通过指针访问该int
为一个数据对象获得并指定分配内存的通用格式如下:
```
typeName * pointer_name = new typeName;
```
new在堆区分配内存,而常规变量在栈区

如果没有足够的内存分配,new通常会引发异常
在较老的实现中,new将返回0,C++中值为0的指针称为**空指针**(null pointer)
C++确保空指针不会指向有效的数据,因此通常用来表示运算符或函数失效

#### delete

```
int* ptr = new int;
...
delete ptr;
```
一定要配对地使用new与delete,否则将引发内存泄漏(memory leak),
即被分配的内存无法再使用.
且不要用delete释放声明变量获得的内存,只用来释放new分配的内存.
delete空指针是安全的