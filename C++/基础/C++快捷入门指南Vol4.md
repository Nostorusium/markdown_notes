# C++快捷入门指南Vol.4

## 单例模式Singleton

Singleton只允许被实例化一次.
单例是组织一堆全局变量的一种方式.

单例模式通常用于某种全局数据集,比如随机数生成器.
我们不希望重复的实例化它,而是一直用下去.

```
class Singleton{

private:
    Singleton(){}
    Singleton(const Singleton&) = delete;
    static Singleton instance;

public:
    static Singleton& get(){
        return instance;
    }
};

// the way you initialize static variable.
// default construction and forms the unique instance.
Singleton Singleton::instance;
```

我们需要把构造函数定义为private,并禁用复制构造函数.
然后提供一个静态访问该类的方法在类外.

>此处补充关于静态成员变量的用法
静态成员属于整个类,而不属于某个对象,不属于某个实例.
而构造函数构造的是对象,或者实例.所以静态成员变量需要在类外使用::初始化
类名::静态成员,这个行为不是在访问某个具体的实例,而是在访问这个类本身的结构,去全局区寻找它的static成员
如果不存在其初始化,则创建第一个对象时所有静态数据都会初始化为0.
使用静态变量和函数就好似是在使用namespace
单例模式完全可以绑定在一个namespace里作为非类代码,但把它定义为类则保留下了类的很多性质

全局区域内,初始化静态的instance成员变量,由于它也是一个Singleton,所以对其的初始化执行了默认构造函数从而得到了一个实例.
此处的关系十分混淆,我们从内存的视角入手：

1. 由于静态变量的存在,全局区中写死了一个static Singleton instance,目前是0
2. 对它的初始化触发了构造函数,这个instance被初始化为一个正经的实例
3. 理论上我们已经可以通过Singleton::instance来访问它了,但由于它被设置为private,所以应使用public的get()函数来获得它.

## 左右值

左右可以理解为等号的左右.
L value有时被描述为Location Value,表示它是有位置的,储存在某个位置,可以被赋值
而R value没有位置,是一个临时的值.

```
int a = getValue(); // ok
getValue() = a;     // error
```

对一个左值赋值是有意义的,因为它有位置.
对一个右值是无意义的,它没有位置.

```
int& getValue(){
    static int value = 10;
    return value;
}

// main:
getValue() = 114514; //ok
```

而如果返回一个左值引用,就可以赋值了.

引用必须是左值,你无法从R值中获得L值引用
这很合理,因为你无法从一个不存在的值获得一个存在的有位置的值.

```
void setValue(int& value);
// main:
int i=10;
setValue(i);    // ok
setValue(10);   // error
```

引用必须是左值.

```
int& a = 10;    // error
int temp = 10;
int& a = temp   // ok
```

但常量引用可以接受左右值

```
const int& a = 10;  //ok
```

之所以这样是为了支持这样使用:

```
void setValue(const int& value);

// main:
setValue(10);   // ok
```

以上我们使用的都是左值引用.我们也可以使用右值引用.
用两个&表示使用右值引用

```
void printName(string&& name);
// main:

string name1 = "rua";
string name2 = "rua";
string name = name1+name2;
printName(name);    // error
printname(name1+name2); //ok
```

name1+name2是一个临时的变量,所以是rvalue.
而name已经完成了运算,是一个lvalue.

通常我们选择重载这个函数,让他同时支持左右值引用.
如果我么能够确定我们接受了一个右值,即临时的变量
那么我们不必担心他的生命周期等等,在程序优化上有了操作空间.

> lvalue是某种有存储空间的变量
> rvalue是暂时的值


## 移动语义 move semantics

移动语义允许我们移动对象,这在C++11之前是不可行的.
使用移动语义可以防止对象在参数传递过程中被赋值多次.
考虑一个String类

```
// noexcept表示不接受异常
// 使用右值引用而不是去构造新的
String(String&& other) noexcept{    
    size = other.size; 
    // 让string的data指针直接指向右值的data地址
    data = other.data;  

    // 而如果other被释放,那么当前data从other.data偷来的东西也一并被释放了
    // 所以让other的data指针清空,不要让它在销毁的时候释放掉data处存储的数据
    other.data = nullptr;
    other.size = 0;
}
```

这样一来other的东西就被转交给了当前string,在传递完自身的数据后自己合理地被释放掉.
此时在使用右值构造函数时,把右侧转交给左侧,这样就实现了浅复制.

```
Entity(const String& name) : name_(name){}
Entity(String&& name) : name_(name){}   //此处仍然有复制
```

此时Entity还是会执行复制构造函数,使用右值来构造左,我们可以把name转化为右值,让此处使用右值构造函数来构造左,避免复制.
```
Entity(String&& name) : name_((String&&)name){}
```

在实际中,通常使用类似std::move(...)来转换.

分析一下性能上的优化:
原先在main中需要先构建String对象,显式或隐式地执行了String的构造函数,这是第一次分配.
随后它被传递给Entity中给Entity的成员赋值,此时调用了String的复制构造函数,这是第二次复制.
通过使用移动语义,我们实现了：
- 避免第一次分配而是直接拿来用
- 避免第二次复制

用一句话简单的总结,就是利用了右值的临时性使得右值传递给左值避免了再分配.
此时任何右值都不会引起原先的复制了.