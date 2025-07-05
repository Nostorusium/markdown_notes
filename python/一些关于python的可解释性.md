#  你需要了解的python

## python对象,引用与赋值

```
// C++
int a = 5;
int* ptr = &a;
// allocate 4 Bytes in the stack
// ptr stores address of a

# python
a = 5
# create integer in heap of value 5 
# a is a reference, refers to address of that int
```

python在堆上分配对象，一切变量名实际上是引用/受限指针。

---

```
// C++
int a = 5;
int b = a;
// copy the value of a to b

# python
a = 5
b = a
# a and b refers to the same object
```

python的赋值实际上是对引用的修改

---

```
# python

a = 5
a = a + 1
# ref a redirect to a new object with value of 6
```

python有可变对象与不可变对象之分。
对于如int,str,tuple的不可变对象，其修改实则创建了新对象。原对象如果没有其他引用会被自动回收。

```
# python

list = [1,2,3]
list.append(4)
```

list一直指向同一个列表对象，该对象内部的数据结构被修改

---

```
// C++
void func(int x){x=114514;}
void func(int&x){x=1919810;}
void func(int*x){x="bruh";}
// pass by value,reference,pointer

# python

def func1(x):
    x = 10
def func2(list):
    list.append(4)
# local variable x does not affect outer
# list is modified
```

python只有一种传递方式：引用值传递。
局部变量x是其所指对象的引用(某种意义上受限的指针)，他作为引用值传递入函数内部作为临时变量。该临时变量的修改不影响外部。
局部变量list使用了方法.append，其引用所指对象发生改变。

---

```
// C++
int a = 5
a = "hello";
// error

# python
a = 5
a = "hello"
// ok
```
C++是静态类型，而python是动态类型。
python中，这个a将重新指向字符串对象。变量名只是标签，类型信息由其所指对象携带。

---

```
// C++
include <vector>
std::vector<int> list = new vector<int>(1);

# python
list = []
```

不同于C++，如果你需要复合类型的数据结构往往需要通过标准库STL链接进源文件。
python中的list,dict,set,tuple是内置类型，直接编译进python解释器，无需额外的import

## 内存管理与垃圾回收

```
// C++
{
int* ptr1 = new int(5);
delete ptr1;

std::shared_ptr<string> ptr2(new std::string("RUA"));
}
```

众所周知C++的垃圾回收非常垃圾，你需要时刻注意内存溢出。
而python中，当对象没有引用所指，垃圾回收器会自动释放它。

## import模块

C++的头文件引入是由预处理器进行的复制粘贴，在编译时静态完成。
而python的模块引入则由python解释器在运行时动态完成。

```
import math
```

当执行import math时，我们认为math是一个模块。随后创建一个模块对象，执行模块中的代码。该模块对象会被缓存进sys.modules中。

---

在python中，模块本身就是一个**对象**。

```
import math

print(type(math))           # <class 'module'>
print(dir(math))            # 查看模块的属性和方法
print(math.__name__)        # 'math'
print(math.__file__)        # 模块文件的路径

# 可以像操作其他对象一样操作模块
math.my_custom_attr = "Hello"  # 给模块添加属性
print(math.my_custom_attr)     # Hello
```

你可以单独引用模块下的可引用对象。

```
// C++
namespace math{
    double sin(double x);
    double cos(double x);
}
using math::sin;
using math::cos;

# python
from math import sin,cos
print(f"sin:{type(sin)}") # <class 'builtin_function_or_method'>
```

from import的本质是从模块对象中提取属性引用到当前命名空间。无论怎样import，python都会完整加载整个模块，但命名空间有所不同。

---

python中的模块，就类似于C++中的namespace+编译单元，是代码组织单位。它会塞一大堆变量，类，等等，就好比把一堆东西组织在同一个namespace中。

区别在于C++的namespace是编译时构造的，而python的module是运行时导入的。

## python类与魔术方法

```
class Person:
    def __init__(self, name, age):  # 构造函数
        self.name = name            # 实例变量
        self.age = age
    
    def introduce(self):            # 方法
        return f"{self.name},{self.age}"
```

python中的类没有显式的访问控制。每个方法的第一个参数都必须是self,作用相当于C++的this指针。

python拥有动态属性，可以随时添加新属性。


```
man = Person("田所浩二",24)
man.city = "下北泽"
```

---

python的魔术方法均有双下划线包裹。

```
class MyClass:
    def __add__(self, other):      # 重载 + 运算符
        return result
    def __eq__(self, other):       # 重载 == 运算符
        return True/False
    def __init__(self, name):      # 构造函数（类似C++构造函数）
        self.name = name
    def __str__(self):             # 转字符串,类似toString
        return f"Person({self.name})"
    def __len__(self):             # 支持len()函数
        return len(self.name)
    def __getitem__(self, key):    # 支持[]索引访问
        return self.name[key]
```

类比C++的重写操作符，你可以给每个类定义这样的特殊方法，使它们实现一定的特殊功能。