# inline关键字

inline关键字的发明之初的本意是给优化器当指示,表示可以某个函数内敛替换而不用调用.但实际上你会发现这种指示并不是强制性的,到底要不要做内敛优化由编译器说了算,即便加了inline也可能不内联,不加inline也有可能内联.

因此C++17之后,关键字inline对于函数的定义已经由"优先内敛"变成了"容许多次定义",并让这个含义拓展到了变量.

考虑一个这样的foo.cpp

```
// foo.cpp:
foo(){
    i=i+1;
}
```

## Case1:全局变量

```
// foo.hpp:
int i;
foo();
```

```
// main.cpp:
#include"foo.hpp"

int i;
int main(){
    cout << i;
    foo();
    cout << i;
}
```

考虑这样引用头文件,显然根据同一定义原则,头文件中的的i定义与main中的i定义重复.

## Case2:extern变量

```
// foo.hpp:
int i;
foo();
```

```
// main.cpp:
#include"foo.hpp"

extern int i;
int main(){
    cout << i;
    foo();
    cout << i;
}
```

使用extern不会认为是定义变量,而认为是声明,不会导致重复定义.
main中的i会引用foo.hpp中的i.

同理可以反过来,让头文件中的i使用extern,这样就会以main中的i为准,引用main中的i.

## Case3:static变量

static关键字会让被声明的变量局限于当前翻译单元,可以认为被当前翻译单元私有.
这会导致头文件中的i和main中的i各自独立.
虽然不会导致重复定义,但也没法共享.

## inline

编译器对于多个inline变量不会报重复定义错误,而是选择一个作为真正的定义.
inline变量具有static变量的优点:不会报重复定义错误 又具有全局变量的优点:可以跨翻译单元共享

```
// foo.hpp:
inline int i = 114514;
foo();
```

```
// main.cpp:
#include"foo.hpp"

inline int i = 1919810;
int main(){
    cout << i;
    foo();
    cout << i;
}
```

此时,到底选用哪个inline i则是随机的,这通常被认为是一种未定义行为,编译器只能按心情行事

## 具体使用

```
// main.cpp:
#include "foo.h"

int main(){
    cout << func(i);
}
```

```
// foo.hpp
inline i = 114514;
int func(int x){
    return x*1919810;
}
void foo();
```

```
// foo.cpp
#include"foo.hpp"
foo(){
    i=i+1;
}
```

考虑一个这样的结构,在foo.hpp中定义函数.
这会导致在foo.cpp中出现一次函数定义,main.cpp中又出现一次函数定义,导致重复定义.

要注意到#pragma once和#ifndef #define并不能帮助避免这里的重复定义.这些预处理指令只能保证在同一单元内不会出现重复定义,而无法保证整体链接后的文件状态.

此时使用inline来定义头文件里的函数即可.通常我们会在在头文件中使用inline,保证经过链接后,同一个头文件里的定义经过链接后重复出现的定义都是inline的,那么编译器不管怎么选都将选定同一个定义.

类的成员是默认inline的

## 总结

inline可以让多个翻译单元共享一个变量/函数体