# STL从入门到放弃

## STL结构

官方地、不用人话地说，STL可以大致分成六个部分
- 容器
- 算法
- 迭代器
- 分配器
- 仿函数

迭代器用于迭代容器内容.
其中,算法通过迭代器来操控容器,算法不考虑容器里面存储的类型,只关心迭代器的加加减减.
仿函数就是使一个类的使用看上去像一个函数(比如for_each),它可以指导算法
分配器是一个边缘化的东西,只和容器挂钩.


## 复习一下std::vector

vector的大小是24,因为它本质上是三个指针,分别指向起始地址,结束地址,?

我们可以往std里加一个运算符重载来支持打印vector

```
#pragma once
#include <iostream>
#include <vector>
namespace std{

template<class T>
ostream& operator<<(ostream &os,vector<T> const &v){
    os<<'{';
    auto it = v.begin();
    if(it!=v.end()){
        os<<*it;
        for(++it;it!=v.end(),it++){
            os<<','<<*it;
        }
    }
    os << '}';
    return os;
}

} //namespace std
```

vector::resize(int n)
vector::resize(int n,T value)
会调整大小,其中延长的部分会默认置0或其他表示默认的值,或者使用第二个参数指定
缩短的部分将直接被截掉.

vector::clear()
clear是十分暴力的，等同于resize(0),变成一个空数组.

vector::size() 获得大小
vector::push_back在末尾追加一个元素
vector::pop_back则取出一个元素,但它返回值是void,并不会"pop"出
vector::back() 可以获得末尾元素的值,和pop_back组合实现"pop"效果
vector::front() 等价于v[0]
vector::data() 返回首地址指针，等价于&v[0] 首地址+长度即可完全获悉一个vector

vector本身是符合RAII的,在退出作用域时释放内存.在释放后对着同一个位置访问会导致segment fault.
如果希望在作用域外扔保持data对数组的弱引用有效,可以std::move把vector对象移动到外面.
vector在移动时指针不会失效

但要注意到resize过大时,不得不重新调整在堆中的位置,这也会导致起始地址的改变
那么过去.data得到的指针/迭代器对象都会失效.同理,push_back也会.
但是resize缩小并不会重新分配,而且再次扩容回来不会重新分配,而是会保留,放置你移来移去
这里涉及到capacity和size的分离,其中capacity表示实际能存储的总容量,他是首尾地址之间的区域
而size指实际有元素的数量
假如resize从514缩短到114 那么resize所修改的是size为114,而capacity会一直保留为514

考虑一个push_back的场景,当容量不够需要resize扩容时,实际上尽管size只+1,但capacity会扩容更多
比如1.5倍,2倍,不同编译器实现不同.gcc里会扩容100%,即2倍
而如果你resize后的尺寸还是很大,比如超过了原先尺寸的2倍,那就没有这个效果了.
换句话说,resize(n)的逻辑是扩容为max(n,capacity*2)
push_back的扩容逻辑同理.

vector::capacity()返回实际已分配的大小

vector::reserve(int n) 预留一定的容量(capacity),免得老是分配.
reserve只会扩容,不能缩短.
vector::shrink_to_fit(),让capacity就等于size.这也会造成首地址的移动.