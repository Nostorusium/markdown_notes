# C++快捷入门指南Vol.3

**C++ fast Amateur Tutorial Vol.3**

## @TODO:线程Thread

使用thread.join()来等待该线程执行完毕.



## 计时 timing

善用计时来看看实际运行代码需要的时间,判断优化的好坏.

```
#include<iostream>
#include<thread>
#include<chrono>
int main(){
    using namespace std::literals::chrono_literals;
    
    auto start = std::chrono::high_resolution_clock::now();  // get current time
    std::this_thread::sleep_for(1s);
    auto end = std::chrono::high_resolution_clock::now();

    std::chrono::duration<float> duration = end-start;
    std::cout << duration.count() <<"s"<<std::endl;
}
```

using namespace std::literals::chrono_literals提供了s这样的特殊操作符



下面是一个简单的计时器类
```
class Timer{
public:

    std::chrono::time_point<std::chrono::steady_clock> start,end;
    std::chrono::duration<float> duration;

    Timer(){
        // get current time
        start = std::chrono::steady_clock::now();
    }

    ~Timer(){
        end = std::chrono::steady_clock::now();
        duration = start - end;
        float ms = duration.count*1000;
        std::cout<<"Timer took: "<< ms<<" ms\n";
    }
};
```
我们可以在函数体开头用栈实例化这个Timer,当函数结束时它随之自动销毁执行析构函数并给出该函数所用时长.

## sort

sort()接受迭代器与比较函数

```
bool cmp(const type1&a,const type2 &b)
```
比较函数的逻辑是如果 a < b 则返回true;
列表中a,b反映了排序后的先后顺序,如果返回true就表示a在前面

```
std::vector values= {1,2,5,6,7,3,4};
std::sort(values.begin(),values.end(),[](int a,int b){
    if(a==1){
        return false;
    }
    if(b==1){
        return true;
    }
    return a<b;
});
```
这段代码的含义是把1挪到列表最后

## 类型双关type punning

尽管有auto,C++仍是一个强类型语言.
类型是编译器强制的,但我们可以绕过类型系统,把同一块内存的东西用不同type的指针取出.

```
int a = 50;
double value = *(double*)&a;
```

由于添加了四字节不属于自己原本内存的部分,value的值会改变.

```
struct Entity{
    int x,y;
};

int main(){
    Entity e = {1,2};

    int* pos = (int*) &e;
    std::cout<<pos[0]<<pos[1]<<std::endl;

    int y = *(int*)((char*)&e+4);
}
```

通过把e的首地址解释为一个(int*),我们得以用数组方式访问Entity的成员
通过复杂的地址转换也可以,int y可以代表e.y

得益于能够直接操控内存(raw memory),内存里的内容可以进行任意的操作和解释,这也意味这危险和不可读
在实际的应用中别这么写.

## 虚析构函数 Virtual Destructors

当处理多态时,虚析构函数十分重要
假设你有一系列子类与继承,子类有自己的destructor,
而我们有时会遇到以基类指针访问的子类实例,
我们需要虚函数使它能正常地调用子类自己的析构函数

```
Base* poly = new Derived();
delete poly;
```
如果基类没有虚析构函数则该子类将调用基类的destructor
当你允许一个类被继承,你必须把他的析构函数定义为虚函数.

虚函数表在编译时生成,这个表记录下所有虚函数的地址.
该类的对象将包含一个指向虚函数表的指针,由此指针这个对象就能访问到所有的虚函数的地址(可以认为有函数指针)

继承了这个基类的子类也会有自己的虚函数表,它的内容会包含基类定义的所有虚函数.当子类重写了基类的虚函数,那么子类的虚函数表中的对应项会被覆盖为子类的函数,这就实现了多态.
C++11之后推荐使用override来标注重写的方法.当把基类的析构函数定义为虚函数,子类的析构函数也遵循这一原则.

此外,基类的构造函数不能是虚函数.
尽管虚函数表在编译时就已存在,而每个对象所持有的指向虚函数表的指针vptr在实例化时才被初始化.
这造成了一个闭环:
要使用虚函数表就要先实例化,要实例化就需要虚构造函数创建而使用虚函数表.
因此构造函数不能是虚函数.

## 安全性 Safety

> SMART POINTERS ARE GOOD
> being able to use pointer freely!

shared_ptr不是线程安全的.
但通常而言他们会极大地帮助内存释放.

shared_ptr的引用计数本身是安全且无锁的,但是对象的读写不是.
由于shared_ptr有两个数据成员(指针本身与指向引用计数的指针),所以读写操作并不能原子化.

如果要从多个线程读写同一个shared_ptr,需要加锁.

```
shared_ptr<Entity>
                       +----------+
   +-----------+   +-->+  Entity  +<--+  +-----------+
   |    ptr    +---+   +----------+   +--+    ptr    |
   +-----------+                         +-----------+
   | ref_count +---+   +----------+   ?  | ref count |
   +-----------+   +-->+    1     |      +-----------+
                       +----------+
```

由于复制ptr和增加引用计数分为两步,构不成原子操作,所以多线程情景下先后执行顺序造成的不一致无法避免.

## 预编译头文件 Precompiled Headers

预编译头可以来获得一堆头文件,并把他们转换成一种编译格式
它只编译一次,然后以二进制格式存储.

当项目要引入一堆头文件,且头文件的内容不会经常更改,如标准库中的内容
那么使用预编译是很好的策略,这样程序每次编译就不需要再重新编译这些内容.

预编译的内容不能经常修改,不然需要重新编译生成预编译头文件,而预编译头文件比一般的头文件编译更耗时.

```
// std.hpp:
#pragma once

#include<iostream>
#include<vector>
#include<array>
#include<string>
#include<thread>
...
```

这样我们就定义了一个std.hpp,我们以它为单位进行预编译.

```
#include"std.hpp"
int main(){
    std::cout<<"hello!\n;
}
```

最后设置预编译,VSCODE可以采用Cmake来设置,结构如下

```
target_precompile_headers(目标名 可见度控制 头文件)
```

以下列cmake为例

```
# 输出文件名
set(OUTPUT_NAME "test")
# 源文件
set(SRC_FILES
   "${PROJECT_SOURCE_DIR}/src/test.cpp"
)
add_executable(${OUTPUT_NAME} ${SRC_FILES})
# 预编译
target_precompile_headers(${OUTPUT_NAME} PUBLIC 
   ${PROJECT_SOURCE_DIR}/src/preCompile.hpp
)
```

## Dynamic Casting

dynamic_cast用于类的类型转化.
考虑一个继承关系:

```
Entity -> Player
       -> Enemy
```

当给出一个Entity的指针,它也有可能是Player或Enemy.
当我们使用一个Entity指针,但它实际上是Player,却把它转化为Enemy,那么这种强制转换将返回一个空指针.
所以动态转化是常用来判断一个对象是否为给定的类型
我们可以强转Entity为一个Player,如果没有返回NULL那么说明这个指针确实是Player

```
    Player* player = new Player();

    // OK
    Entity* actuallyPlayer = player;
    Entity* actuallyEnemy = new Enemy();
    
    // ERROR
    Player* p1 = actuallyPlayer;
    Player* p2 = actuallyEnemy;
```
把entity转换为继承类可以
但把继承类转化为父类不行.
*Player不能指向一个 *entity因为这个entity有可能实际上是一个Enemy.

```
Player* p = (Player*)actuallyPlayer;
```
我们可以使用强制转换.
这种转换实际上是告诉编译器不要担心,这个entity确实是一个player.
这其实很危险,如果不幸地把一个Enemy强转为一个Player,那么显然会出问题.

所以可以使用动态转换
```
Player* p = dynamic_cast<Player*>(actuallyPlayer);
```
动态转换只能用于有多态的类,需要类有一个虚表,所以Entity内部要拥有vitrual修饰的函数.

```
// OK
Player* p = dynamic_cast<Player*>(actuallyEnemy);
```
那么这条语句并不会报错,它将返回NULL.

这就是动态转化的作用:判断转化是否有效
如果无效,则返回NULL.

这依赖于Run-Time Type Information,RTI,运行时类型信息.
如果我们关闭RTI,那么编译器会报错,动态转化会导致不可预测的结果.
RTI是有开销的因为它需要更多的空间来保存类型信息.
dynamic_casting也是有成本的,如果你需要写高性能高优化的代码,那就少用.

dynamic_casting也提供了一个判断类型的途径:

```
if(dynamic_casting<Player*>(actuallyEnemy) != NULL){
}
```

## @TODO Benchmarking 基准测试

## structeds bindings 结构化绑定

C++17的新特性,让我们更好的处理多个返回值

```
#include<iostream>
#include<tuple>

std::tuple<std::string,int> CreatePerson(){
    return {"T.S.H.R.",24};
}

int main(){
    auto person = CreatePerson();
    std::string& name = std::get<0>(person);
    int age = std::get<1>(person);
}
```

这是一个使用元组(也可以用pair)作为返回值的例子,取出内容非常麻烦,而且可读性非常差.
使用tie来赋值在可读性方面好一些:

```
std::string name;
int age;
std::tie(name,age) = CreatePerson();
```

又或者定义struct,返回一整个结构再去访问.

```
struct Person{
    std::string name;
    int age;
}
```

C++17的特性结构化绑定解决了一系列问题.

```
int main(){
    auto[name,age] = CreatePerson();
    std::cout << name;
}
```
只要在同一个作用域内就可以这样直接访问.

## Optional

Optional用于处理有可能存在也有可能不存在的数据.
比如读一个文件,有可能存在也有可能不存在.

```
#include<iostream>
#include<fstream>

std::string readFile(const std::string& filepath){
    std::ifstream stream(filepath);
    if(stream){
        std::string result;

        //read file here

        stream.close();
        return result;
    }
    return std::string();
}

int main(){
    std::string data = readFile("data.txt");
}
```
在这个例子中我们无法得知是否成功的打开了文件,只能通过返回的结果来判断.
我们可以在函数中加一个bool引用作为参数,如果成功则为true.
但这很麻烦.

C++17中提出的std::optional解决了这个问题
std::optional类型的变量要么是一个T类型的变量,要么是一个表示什么都没有的状态.

```
 std::optional<std::string> readFile(const std::string& filePath){
    if(stream){
        std::string result;
        // ...

        stream.close();
        return result;
    }
    return {};  //空
}

int main(){
    std::optional<std::string> data = readFile("data.text);
    if(data.has_value()){   // if(data) is cleaner
        std::cout<<"File read successfully\n";
    }else{
        std::cout<<"File not opened\n";
    }
}
```

通过optional就可以直接判断.
optional有以下用法

```
data.value();   //返回data本身的值
std::string& string = *data; //操作符重载,支持解引用
data->size();   //用->使用各类方法
```

另一个用法是:

```
std::string value = data.value_or("Not Present");
```
出于习惯,有时会为未成功读取的情况下设一个默认值.
使用value_or,如果成功读取那就用读取值,不然就用我们给出的默认值.

## std::variant 单一变量存放多类型数据 

## std::any 单一变量存放任意类型数据

## std::async

## 让字符串更快

## 单例模式Singleton

## 左右值

## 移动语义 move

## 迭代器Iterator