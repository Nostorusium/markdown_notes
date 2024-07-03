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

## 移动语义 move