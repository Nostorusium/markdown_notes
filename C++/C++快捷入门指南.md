# C++快捷入门指南

C++ Amatur Tutorial

## 一些基础语法

### 预处理与头文件与宏

预处理实际上只是展开宏文件
头文件一般作为函数声明,他会在预处理中被展开为具体的声明
而他的具体实现通常在某个.cpp中,它应当能在链接过程中被链入主程序.
```
hello.hpp:
void print_helloworld();

hello.cpp:
void print_helloworld(){
    std::cout << "helloworld\n" <<std::endl;
}
```

而如#define PI 3.14 则是宏,macro
它会在预处理阶段把全文中的PI都换成3.14 即宏本质上是文本替换

### 未初始化的变量

变量不初始化的值会根据不同架构,不同编译器而有所不同,x86和ARM会有不同的结果 但总之哪个都不是你想要的

在JAVA中不初始化,编译器会直接报错

### sizeof

sizeof是运算符 而不是函数

### char

建议把char当做一个整数来看,它的初始化和int没有区别
C++11标准里对于中文字符可以用char16_t,char32_t
用u'字'和U'字' 赋值

### bool

C++里bool是数据类型,长度为一个字节
C++里认为它是一个值,可以和int转化
true==1,false==0
而如JAVA,bool和int是不能转化的
对bool赋值,非0即1
```
bool B = (-1 != 0); //B为1,true
```

在C中没有boolean类型
```
typedef char bool;
#define true 1
#define false 0
```
在C99标准后stdbool.h头文件引入了布尔值

### size_t

表示内存大小,或元素个数
计算机的内存越来越大,以前用u32能表示4GB,再想更长就不够了
后来引入了size_t,无符号,它是sizeof的返回类型
64位系统则64位,32位系统则32位.
不同的系统里int,long长度不一样,
所以引入了<cstdint>:
int8_t,int16_t,uint32_t等等,从名字就可以看出长度,方便用

### 浮点数

由于浮点数的精度问题,甚至会出现这样的情况:
```
float f1 = 2.34E+10f;
float f2 = f1 + 10;
// (f1 == f2)为true
```
这是因为10这个增量相比于原先的数太小了,没加上去,舍入后仍是原先值.
而且最好别用==判断浮点数相等
因为浮点数都是近似值而不是精确值,用:
fabs(f1-f2) < FLT_EPSILON,
取绝对值,FLT_EPSILON是一个足够小的数.

### auto

auto是灵活的
auto修饰的变量 数据类型取决于赋值时
C++中未初始化的auto变量会报错
```
auto a = 2;
a = 2.3;
//则 a == 2,会强转int
```

### a++和++a的性能

在**原始**的编译器中,
a++会先保存a的值再给a+1最后返回
++a则会a直接自增1然后返回
在现代编译器中没有区别

### 三元运算符

```
factor = isPositive ? 1:-1;
```
使用三元运算符时机器指令里可能存在跳转,效率低下
如果要扣这个效率可以改写为
```
factor = (isPositive)*2-1;
```

## 数组与内存性能

### array

C++中用变量作为数组长度是可以的:
```
int array[num];
```
C由于编译器比较垃圾所以不行;
C++同样无法用数组赋值数组,这意味着指针地址覆盖
```
array2 = array1; //error!
```
想要值覆盖那就逐一赋值.
C++同样没有边界检查,array也不是对象
但没有边界检查也意味着效率高

### 数组风格String

数组风格的字符串同C,以\0结尾
strlen(string)的长度统计会截止到第一个\0前
```
char* strcpy(char* dest,const char* src);
```
如果dest长度小于src,则会直接覆盖
```
char * strncpy(char* dest,const char* src,size_t count); //safer
```
字符串连接:
```
char* strcat(char* dest,const char* src);
```
字符串比较:
```
char* strcmp(const char* lhs,const char* rhs);
```
返回第一位不匹配的差值,如'a'-'b'
返回0说明相等

### 内存

对齐地址为2^n,通常为4B对齐
这意味着要填充空字节

```
// C-like allocation
// this is bad
// this code will pass the compiler
// but this will actually allocate 4*4=16 bytes.
int * p = (int*) malloc (3);
```

分配的内存需要及时free,若是指向动态分配内存的指针丢失了或者没有free程序就结束了,称内存泄漏
得益于操作系统的抽象我们所言的堆区是对于某个进程而言的,它通常由链表形式组成
好消息是程序结束时操作系统会自动释放该内存的所有地址空间(逻辑上)

```
for(int i=0;i<1024;i++){
    p = (int*) malloc (1024*114514*1919810);
}
printf("end\n");
```
得益于某些操作系统的优化,上面这段代码可能不会在中途因内存不足而崩溃.
参考xv6的延迟堆分配

### C++ style allocation

```
int * p1 = new int;     //default,do nothing
int * p2 = new int();   //initialized to 0
int * p3 = new int(5);  //initialized to 5
int * p4 = new int{};   //C++11
int * p5 = new int{5};  //C++11

Student * sp1 = new Student[16];
Student * sp2 = new Student{"TiansuoHaoer",24};

int * ptr = new int[16];    //new一个数组
int * ptr2 = new int[16](); //C++11
int * ptr3 = new int[16]{}; //C++11
int * ptr3 = new int[16]{1,2,3};    //C++11
```
**new操作符**来申请内存,比malloc功能更强.
不需要做类型转换,不需要强转
**delete**来释放内存

```
delete p1;
//以下细微区别
delete ptr; //释放数组,call第一个元素的destructor
delete []ptr2; //释放数组,call所有元素的destructor
```
对于int数组,delete 带不带[]没有区别
而如像:结构数组,sp1则有区别
所以建议对于数组都加方括号

### 函数

一个好的习惯:
同一类的函数放在同一个cpp,并在hpp中声明
```
#ifndef
...
#endif
```
这是为了防止重复include,如果if no defination才去声明 以#endif作为结尾标记.

### 参数传递

C++为值传递,一切参数皆为拷贝
在使用结构作为参数时,意味着要拷贝一份这个结构并占用较大的内存

## C++语法

### 引用references

C++有而C没有,引用即别名
```
int num = 0;
int & num_ref = num; //num_ref为num的别名
num_ref = 10;        //num和num_ref是一个东西 赋值为10
```
引用在声明时必须初始化,否则编译不通过.
引用只是别名,而指针是地址
引用比起指针更加安全

在JAVA中没有指针,所有非基本数据类型都是引用类型,使用引用代替了指针保障内存的安全.

使用引用的好处：当传参为一个较大的结构
```
char* getName(struct Student stu);      //传入结构 内存占用较大
char* getName(struct Student* stu);     //使用指针 但是危险
char* getName(struct Student & stu);    //使用引用操作的是数据本身 没有复制一份 效率更高
char* getName(const struct Sutdent &stu) //使用const修饰 防止被修改
```

```
void increase(int& value){
    value++;
}

int main(){
    int a = 1;
    increase(a);
    std::cout<< a << std::endl;
    return 0;
}
```

### inline减少函数调用

对于被频繁调用,而内容又不多的函数,考虑使用inline函数减少函数调用的次数 这是针对编译器而言的,他们在程序上没有区别
```
inline float max_function(float a,float b);
```
inline只会建议编译器做内联 而不一定
如果函数又长又复杂则不会 因为内联是以空间为代价节省开销
到底会不会执行内联 取决于编译器 他会自动的取消不值得的inline
建议inline不要出现在函数的声明中,用户没有必要也不应该知道函数是否需要内联

对比**宏**:
```
#define MAX_MACRO(a,b) (a)>(b)? (a):(b)
//一定记得加括号 保证运算优先级
```
宏是简单粗暴的替换 同样没有函数调用的代价
但根据宏的简单替换的定义:
```
MAX_MACRO(num1++,num2++)
//  (a++)>(b++)? (a++):(b++)
//  将会加两次
```
使用宏会很复杂,容易出错,很危险

### 参数默认值

```
int multi(int x,int y,int z=0);
```
关于"参数"argument和parameter的区别:
0是一个argument,而z是一个parameter
即argument是**实参**,parameter是**形参**
设置默认值有一个要求:默认参数只能在后面

```
//有顺序要求,从上到下一次定义了默认参数z=0,y=0
int multi(int x,int y,int z);
int multi(int x,int y,int z=0);
int multi(int x,int y=0,int z);

//如果颠倒顺序,会认为z未定义
int multi(int x,int y,int z);
int multi(int x,int y=0,int z);
int multi(int x,int y,int z=0);

//也不能重复定义 只能设一次 先z 再y 最后x,从尾部开始设
int multi(int x,int y,int z);
int multi(int x,int y,int z=0);
int multi(int x,int y=0,int z=0);
```

这是出于**规范需求**,考虑这样的情况:
```
multi(1,,2);
```
这样看着非常不好看,容易造成混乱
```
multi(1,,);
```
则是可以接受的

对于参数很长时,可以使用默认参数.

### 重载,overload

重载就是多态
要求名字相同,返回类型相同而参数不同
对于名字和参数都相同而返回值不同的函数,认为是两个函数,且重名报错

### 函数模板 templates

当某个函数的几个重载非常相似大差不差,考虑使用函数模板
模板建议放进头文件
模板的提出是为了解决泛用性

```
//定义模板
template<typename T>
T sum(T x,T y){
    cout <<"the input type is" << typeid(T).name() <<endl;
    return x+y;
}

//他的显式实例化，instantiation
//<>里放T的类型
template double sum<double>(double,double);
template char sum<>(char,char);
template int sum(int,int);
```
举例:
```
#include<iostream>

template<typename T>
T sum(T x,T y){
    std::cout<<"input type is "<<typeid(T).name()<<std::endl;
    return x+y;
}

template double sum<double>(double,double);

int main(){
    auto value = sum(4.1,5.2);
    std::cout<<value<<std::endl;
    return 0;
}
```
关于隐式实例化:当没有找到显式的实例化,编译器会根据使用的类型猜测T的类型
即隐式实例化
```
sum<int>(2.2f,3.0f);    //根据指定的int实现,强转为int
sum(2.2f,3.0f);         //根据float参数实现
```

一些问题:
```
struct Point{
    int x;
    int y;
};
//不能执行x+y
sum<Point>(point1,point2); //error
```
结构不能满足模板定义的运算符+,所以无法做这样的运算.
那么就需要特例化:Specialization
```
template<>
Point sum<Point>(Point pt1,Point pt2){
    Point pt;
    pt.x = pt1.x+pt2.x;
    pt.y = pt1.y+pt2.y;
    return pt;
}
```
但这里完全可以写一个Point的相加方法.
使用模板的好处是当Point经过了多次迭代后不用怎么修改

### 函数指针与引用

```
//如何理解这句话:
    int (* func_ptr) (int x,int y);
//  ^      ^              ^
//  |      |              |
//  |      |   explicates the function with arguments (int x,int y)
//  | the name of this function pointer
// return type
```

括号不能删,如果删了就会变成:int* func_ptr(int x,int y);
它会变成函数定义.换言之,只要把函数定义加上括号就是它的函数指针形式

使用如下:
```
//写法是等价的
func_ptr = func1;
func_ptr = &func2;
```
实例:
```
cout << sum_ptr(3,4) <<endl;
cout << (*mulpt_ptr)(3,4) <<endl;

//应用案例:把函数作为参数传入
//comp作为排序的标准传入 通过这个指针来调用函数
void qsort(void * ptr,size_count,size_t size, int(*comp)(const void*,const void*));
```

基于指针的危险性,所以我们提出了引用:
```
int (& func_ptr) (int x,int y);
```
引用定义后就要初始化,且之后不能转向别的变量(const的)

### 关于指针和引用的对比

据C++之父所称引入引用只是因为使用指针有时很丑 于是进行操作符重载

```
void func1(const complex* x,const complex* y){
    return *x+*y;
}

//操作符重载后非常好看
void func2(const complex& x,const complex& y){
    return x+y;
}
```


## 组织策略

### 程序组织

- 头文件:声明函数
- 源代码1:保存与结构有关的代码
- 源代码2:保存需要调用结构相关函数的代码

不要在头文件里定义函数和声明变量,如果同一程序的两个不同源代码文件里同时使用了这个头文件,那么预处理后将会重复.

头文件:
- 函数原型
- #define和const定义的常量
- 结构声明
- 类声明
- 模板声明
- 内联函数
在使用include时候,""意味着从当前工作目录查找,而<>意味着从标准头文件位置查找.

为了防止头文件重复引用导致的重复定义,使用:
```
#ifndef
...
#endif
```
### static

一句话总结:静态变量具有**全局变量的生命周期**却**只能作用于自己的作用域**
或者认为是:作用域受限的全局变量
在链接阶段,链接器不会试图在外部寻找static的符号定义

- **类内static变量**
  为本类的所有对象共享,不占用对象的内存而在所有对象之外开辟内存,存储在全局存储区
  使用场景:每一个实例都相同的变量,如:所有存款对象的利息
  比起使用全局变量,static没有进入全局的命名空间所以不会产生命名冲突
  而且static变量可以使用private修饰从而实现信息隐藏
  ```
  //这样访问
  classname::variable = 114514;
  ```
  适用于把某些需要所有对象都共享的信息设置为static

- **类内static函数**
  在类外面的函数定义不能指定关键字static
  静态成员函数只能访问静态成员变量/函数
  它属于类本身,而不作用于对象,也不能访问属于类对象的非静态变量/函数
  因为它没有实例,自然也没法访问非静态变量
  ```
  //这样使用
  classname::function();
  ```

- **static全局变量**
  函数内部的自动变量存放在栈区,new出来的在堆区,全局变量在**全局数据区**(如.data,.bss)
  而static使全局变量仅作用于当前文件(限制了作用域)
  
- **static静态局部变量**
  使该局部变量不在栈区分配而在全局数据区,使得该局部变量不会因为函数结束而销毁
  static的作用于为局部作用域:作用于定义它的函数或语句块

- **static静态函数**
  只能被当前文件使用的函数,因为作用于的问题不会和其他文件中的同名函数冲突

### 命名空间

命名空间只能在全局范围内定义
```
namespace A{
    int a = 100;
}
namespace B{
    int b = 200;
    namespace C{
        int c = 300;
    }
}
int fn(){
    return A::a + B::b + B::C::c;
}
```
命名空间是开放的,可以随时把新的成员加入命名空间:
```
namespace space{
    int a = 1;
}
namespace space{ // OK
    int b = 2;
}
```

我们要保证不要在头文件与其cpp里using任何东西,尤其是namespace
不然include进来的时候回很容易发生命名冲突,有条件的话所有引入的符号都定义在自己的namespace里
使用using语句的头文件被外部使用很可能给适用方带来麻烦


## 面向对象


### class与struct的区别

struct在C++中唯一的存在理由是为了保持对C的兼容
如果你希望所有成员变量都是public,那就是用struct.
class的好处是提供了如private,protected之类关键字方便封装
对于何时使用class/struct,没有正确答案,取决于代码风格

### class

较好的组织方法是把类的声明存入.hpp 类方法只写声明,在.cpp里实现

不定义构造函数默认会构造一个空的构造函数
如果在constructor申请了堆区空间那么在destructor去释放他们是非常自然的

### 继承和多态

inheritance & polymorphism
```
class Player : public Entity{
public:
    const char* Name;
    void printName(){
        std::cout<<Name<<std::endl;
    }
}
```
父类中private不会被继承,如果想继承的同时又保持其不被外部类访问应使用protected

### 虚函数

```
class Entity{
public:
    string printname(){
        cout << "entity\n" << endl
    }
};

class Player : public Entity{
    string printname(){
        count << "player\n" << endl;
    }
};

void printName(Entity entity){
    entity.printname();
}

int main(){

    //what will happen?

    Player player;
    printName(player);
}
```
尽管传入了player对象,可是由于printName接受的参数是entity,所以他会转去执行entity的printname方法.
为了能够识别这种问题,引入虚函数,使用virtual修饰
```
class Entity{
public:
    virtual string printname(){
        cout << "entity\n" << endl
    }
};

class Player : public Entity{
    //C++11支持使用override标注重写的方法
    string printname() override{
        count << "player\n" << endl;
    }
};

void printName(Entity* entity){
    entity->printname();
}

int main(){
    Player p = new Player();
    printName(p);
}
```

虚函数是用虚标vtable实现的,需要额外的内存存储虚表,所以是有代价的
每次调用虚函数都需要遍历虚表来找到最终执行的函数,所以也有性能花费

通常而言在最顶层的虚函数加上virtual,而它的子类在覆写后统一加上override,就不写virtual了
对于不想让子类覆写的情况,使用final来表示不允许覆写
final关键字也可以修饰类,表示这个类不能被继承

如果子类没有重写父类的方法,那就用父类的

### interface

```
// = 0 会把方法设置为纯虚函数
vitrual std::string getname() = 0;
```
纯虚函数只能在子类中实现
这意味着这个类无法实例化,你必须提供一个实现了这个函数的子类

你只能实例化一个实现了所有纯虚函数的类(每个都要有实现)

接口本质上还是一个class,但我们习惯称呼他为interface
它只有纯虚函数
在JAVA中确实有关键字interface来代替class,但C++没有

### 可见性Visibility

public,protected,public

```
class Entity{
private:    
    int X;
    int Y;
    void Print(){}
public:
    Entity(){
        X=0;
        Print();    // in-class is ok
    }

}
```
对于private的X,他的可见性仅限于这个class类的块中
只有Entity类和它的友元能访问,它的继承也不行
```
int main(){
    Entity e;
    e.X = 1; //error
}
```
main已经超出了它的可见域.
在类内使用是可以的

protected在private的基础上把可见域拓展到了继承类
```
class Entity{
protected:
    Print(){}
}

class Player : public Entity{
    Player(){
        Print();    //ok
    }
}
```

可见性与性能无关,不会影响性能
为什么要提出可见域,为什么不把所有设为public:

private告诉你: 你不应该从其他类访问它!你只能使用这个类公开的方法来使用它!
> 移动一个UI的坐标
> 你不应该直接UI.x = 114;Ui.y=514;
> 而应该去使用UI.SetPosition(x,y);
> 因为你并不知道它的具体实现,直接改X Y很有可能无法达到想要的效果甚至造成破坏
> 比如SetPosition里就多了ReFrash(); 而直接赋值无法刷新

### 创建对象与new

C++可以选择对象创建的区域(堆/栈)
有着各自的生存周期.

栈区对象存在着自己的作用域,当作用域结束后栈会弹出并释放
堆区对象在创建后就会一直存在直到你释放

```
int main(){
// create on stack    
    Entity entity1;             //default constructor
    Entity entity2("argument");
    Entity entity3 = Entity("argument");
}
```
这是C++中初始化对象最快,最受管制的方式
但它创建的对象无法再作用域外存活
```
void Function(){
    Entity e = Entity("name");
}
```
一旦函数抵达了下面的大括号,这个栈区对象就会被释放
此外如果对象太大了或者有太多个对象,考虑在堆区创建
因为栈通常很小

```
Entity* entity = new Entity();
```
在堆区分配使用new关键字,他会返回一个指针
和JAVA与C#有点像:
在JAVA中所有东西都是在堆上分配的
而在C#中所有的类都是在堆上分配的

习惯了JAVA和C#的人通常会在C++里乱new
但实际上在堆区分配比在栈区分配的开销更大
所以应该在合适的时候在栈区分配
并且C++没有垃圾回收,申请的堆区需要自己回收
```
//一个new,一个delete!
delete entity;
```

在堆区分配的好处在于
你可以自由控制它的生存周期,并且适用于体积较大的对象

### new的性能

不管是new什么东西,他都意味着在内存中寻找一段连续的有足够大小的空闲块.
这意味着在堆上分配的开销比栈更大
new返回一个void类型的指针,意味着指向一个纯地址
new只是一个**操作符**,它的背后具体依赖于C++库
但通常而言他会调用底层的C函数malloc()
所以相当于:
```
// you should not allocate memory in C++ like this
Entity* p = (Entity*)malloc(sizeof(Entity));
```
但是
```
Entity* p = new Entity();
```
同时调用了Entity的构造函数

## 数组

对于原生数组,你无法直接知道数组的大小
```
int array[5];
```
这意味着在栈上分配一个5*sizeof(int)

此时array作为一个长度为5的int数组,其大小是已知的,通过sizeof即可获得
那么想要知道它的长度只需：
```
int length = sizeof(array) / sizeof(int);
```
这种用法仅限于在栈上分配的数组
见如下代码
```
int* p = new int[5];
```
如果在堆上分配只能获取到其指针p,而sizeof(p)只能得到一个int*指针的大小
编译器只能知道p是一个指向int的指针而不知道其具体内容,也无从判断长度

在某些语言中提供了array->size(),但C++不行,想要长度就别用原生数组
C++11中提供了std::array,使用头文件#include<array>
它支持边界检查,也有记录的大小
```
std::array<int,5> aArray;
```
则可以直接使用aArray.size()
但这也意味着额外的开销.
使用std数组比原生数组安全,但原生数组更快

## String

### 定义

最常见定义字符串的方法
```
cosnt char name[100] = "TianSuo\0HaoEr";
```
也可以用指针定义:
```
//  C++11开始这种定义方式前必须加const
//  加不加这种方式定义的字符串都是不能改变的
const char* name = "TianSuoHaoEr";
```
之所以不能改变是因为不同于前者在栈上分配了一定大小的数组
这种指针仅仅指向一个某个固定分配的内存块,你无法延长或缩短这一块.
实际上const char* name只是在栈上分配了一个指针,它指向.rodata段的一串常量字符串,不能修改
因此只能用const char*定义.

想要知道字符串大小的通用方法是找\0.
对于数组方式定义的字符串,如果一个字符一个字符定义,C++编译器只会认它是一个字符数组,而不视其为一个字符串,直到有一个\0作为结尾.
如果强行以字符串形式读一个没有\0结尾的字符数组,C++标准库会插入栈守卫之类的东西保证你总能读到一个\0而不是一直读个没完,但这意味原本的信息后面会多出一大堆无效垃圾字符 直到读到\0作为兜底.

### 双引号

```
std::string name = "Ruarua";
```
如果你把鼠标放在这个字符串上,会受到提示:const char.
这也是为什么需要const char* 来定义一个字符串
即在C++中使用双引号来定义一个字符串或多个单词时实际上就在定义const char数组而不是char数组
使用 const char* 定义的字符串无法更改
但没有const修饰的数组形式定义与string类定义的字符串都支持修改.

```
// error:
std::string name = "Tiansuo" + "Haoer";
```
既然双引号里是const char数组,自然不能相加两个const char数组
想要字符串拼接如下:
```
// good:
std::string name = "Tiannsuo";
name = name + "Haoer";
```
这样就是把一个指针加到了字符串name上

### 传参

关于传参:

```
void printString(std::string string){
    std::cout<<string<<std::endl;
}
```
由于值传递,这实际上传递了一个string在堆上的副本
由于printString仅仅是只读 完全没有必要在堆上分配这一多余的空间
复制字符串实际上很慢,而且字符串操作很常见,所以有必要用常量引用传递它
```
void printString(const std::string& string){
    std::cout<<string<<std::endl;
}
```
使用const来保证你不会误操作修改它.

## const

const是一个假的关键字,不对生成代码有任何作用,仅仅是对程序员的一种约束
const只是一种承诺,你也可以设法绕过这种承诺

### 绕过const限制

```
// how to cheat from the restraint of const

const int MAX = 114514;
int* ptr = (int*)&MAX;
*ptr = 1919810;

std::cout << MAX << std::endl;
std::cout << &MAX << ": " << *(&MAX) << std::endl;
std::cout << ptr << ": " << *ptr << std::endl;
```
利用指针来绕过限制
```
results:
114514              // value stock in stack
0x61fe14: 1919810
0x61fe14: 1919810
```

### 修饰变量

注意修饰指针时 以下是两种不同的使用
```
// ptr -> const int
const int * ptr;
// const ptr -> int
int* const ptr;
```

### 修饰类方法

类方法后面的const保证只读性

```
public:
int get_x() const{
    return x;}
```

这保证了get方法不会修改任何类的属性值
这也意味着如set方法的需要修改属性值的方法不应该用const修饰
const应该修饰明确不修改值的方法
下面是一个const套娃的例子

```
const int* const GetX() const{
    return x;
}
```
结合一下前文提到的const用法,这个方法的翻译是:返回一个常量指针,指针指向一个int常量,方法内部不允许修改类属性值.

### 修饰参数

考虑下列情况

```
class Entity{
private:
    int x,y;
public
    int getX(){
        return x;
    }
}

// error
void printEntity(const Entity& e){
    cout<<e.getX()<<endl;
}
```
在传递参数时使用const,意味着这个Entity不能够被修改.
而此时使用e.getX()将会报错,因为编译器无法保证getX()方法不会修改Entity.
正确的做法是在类方法中加入const修饰来提供这样的保证.

由于重载有时可以见到两个方法:

```
int getX() const;
int getX();
```

编译器会根据参数是否被const修饰而选择对应的方法
如果没有修改类或者不应该修改类时,把方法标记为const是良好的习惯,以防止有人使用常量引用或类似情况使用你的方法

### mutable

有些情况下,你既需要把类方法标记为const,又需要同时修改某个变量,需要mutable修饰需要修改的变量.
被const修饰的方法中,允许修改被mutable修饰的变量.

一个例子:

```
class Entity{
private:
    int x,y;
    mutable int debug_count = 0;
public:
    int getX() const{
        debug_count++;
        return x;
    }
}
```
mutable与const一起使用时最常见的使用方法
另一种是在lambda表达式中使用
lambda类似一个一次性的小函数,可以写出来并赋给一个变量 可以选择用值或者引用传递
[]里的=表示值传递,&表示引用传递

```
int value = 114514;
auto f = [=]() mutable{
    std::cout << value << std::endl;
};
f();
```

在lambda中使用mutable非常罕见,lambda的具体使用日后再说