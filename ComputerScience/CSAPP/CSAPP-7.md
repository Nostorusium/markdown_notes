# CSAPP章节小结

## 第七章 链接

### GNU:GNU is Not Unix

GNU是一个历史悠久的大型项目,旨在开发一个开源的类Unix操作系统
GNU工具链逐渐被发明出来,其中包括了编译器,汇编器,调试器等
其中最著名的是**GCC**:GNU Compiler Collection
但始终没有自己的内核,直到Linux被发明
Linux内核与GNU工具链紧密结合,日后也称GNU/Linux

GNU中的G,gnu,是一种动物:牛羚,角马,也是其吉祥物

### 编译器驱动程序

```
# 召唤gcc,以main.c和sum.c为目标生成prog
# -o用于指示输出文件的名字
# -Og是optimize for debugging,旨在提供不破坏调试能力的优化
# 使用更高级的优化会导致严重破坏可读性进而难以调试
# 直接使用gcc会直接生成可执行目标文件
gcc -Og -o prog main.c sum.c

# 按照步骤生成如下: 预处理->编译->汇编->链接

# cpp,C Pre-Processor 预处理
cpp main.c -o main.i

# cc1,C Compiler part1 编译
cc1 main.i -o main.s

# as,Assembler 汇编
as main.s -o main.o

# ld,linker 链接 d是啥我也不知道
ld -o prog main.o sum.o

# Linux下执行prog
./prog
```

```
如果使用gcc命令也可以

gcc -E :预处理：PreprocEss
gcc -S :编译:compile to asSembly
gcc -c :汇编:compile

进行链接时:
gcc main.o sum.o -o prog
```

### ELF可重定位文件格式

ELF:Executable and Linkable Format 现代Linux和Unix使用的格式

```
ELF头      :描述了生成该文件的系统的字长,字节顺序(大小端),与其他信息
.text      :已编译的机器代码,指令,函数等
.rodata    :read only data
.data      :初始化的全局/静态C变量;局部变量保存在栈,不存在于.data与.bss
.bss       :未初始化的全局/静态变量。与被初始化为0的全局/静态变量
.symtab    :符号表,定义和引用的函数/全局变量信息
.rel.text  :relocate 一个.text节中位置的列表 链接时修改这些位置
.rel.data  :被引用或定义的所有全局变量的重定位信息
.debug     :调试符号表 只有gcc -g时才存在这张表
.line      :C源程序的行号与.text节中机器指令之间的映射 -g时才存在
.strtab    :字符串表,包含了.symtab与.debug节中的符号表.
节头部表    :不同节的位置与大小,数量 目标文件的每个节有固定大小的条目
```

bss:Block Storage Start,块存储开始,始于IBM汇编语言
一个更好的解释是:Better Save Space,更好地节省空间
所以初始化为0也算入.bss

.bss与.data存**强符号**
**弱符号**(未定义的全局变量)在符号表中记录为**伪节COMMON**


### 符号与符号表

每个可重定位文件都可以视为一个目标模块
在链接的过程中,需要进行符号解析.
每个**符号**对应一个**函数**,**全局变量**或者**static变量**,
符号解析的目的是把每个符号引用与其符号定义关联起来。
>static意味着该函数或变量只能被该文件调用,这意味着它不会被其他文件调用


有三种不同的符号:
- 由模块m所定义,并可被其他模块引用的**全局符号**.对应**非静态的**C函数与全局变量
- 模块m所引用的,被其他模块定义的符号,称**外部符号**.对应其他模块中定义的**非静态的**C函数与全局变量
- 只被模块m定义和引用的**局部符号** 对应带static的C函数与全局变量

.symtab中的符号表不包含**本地非静态程序变量**(局部变量),这些符号在栈中被管理
除非他被static修饰,被static修饰意味着它的生命周期贯穿程序始终

而带有static的本地变量不在栈中管理,编译器在.data与.bss中为它们分配空间,并加入符号表

### 符号表条目

```
int f(){             //在符号表里 加入.text
    static int x=0;  //static 在符号表里,已初始化变量,但是0,加入.bss
    return x;
}
int g(){            //在符号表里 加入.text
    static int x=1; //static 在符号表里,已初始化变量,不是0,加入.data
    return x;
}
```
可以用x.1表示f中的定义,x.2表示g中的定义

符号表**条目**结构如下:
```
typedef struct{
    int name;       //字符串表中的字符偏移,指向某串字符串
    char type:4,    //类型,是函数还是数据
         binding:4; //说明本地或全局
    char reserved;  //保留,unused
    short section;  //该符号被分配到哪个节
    long value;     //节内偏移
    long size;      //大小
}ELF64_Symbol;
```
在可重定位目标文件中,在符号表项的section字段有三个特殊的伪节。
这些节在节头部表中没有条目。
ASB: absolute,绝对的,不应该被重定位的符号
UNDEF: 未定义的,在本模块中引用,却在其他模块定义的符号
COMMON: 一般的 表示没有被分配位置的未初始化数据目标
可执行文件中没有这些伪节(已重定位处理)

现代gcc按以下规则分配:
COMMON伪节 :未初始化的**全局变量**
.bss :未初始化的静态变量,以及初始化为0的全局/静态变量(better saving space)
未初始化的全局变量被认为是**弱符号**,所以放进COMMON

```
Num:            Value  Size    Type   Bind    Vis      Ndx  Name
  8: 0000000000000000    24    FUNC   GLOBAL  DEFAULT    1  main
  9: 0000000000000000     8    OBJECT GLOBAL  DEFAULT    3  array
 10: 0000000000000000     0    NOTYPE GLOBAL  DEFAULT  UND  sum

```
Ndx为section索引
全局符号main(函数) 位于.text节中偏移0处 大小24字节
全局符号array(数据)位于.data节中偏移0处 大小8字节
未定义符号sum(无类型) 位于伪节UND(undefined)

### 符号解析

每个引用与符号表中的一个符号定义关联起来.
对于**局部符号**(自己定义自己引用),静态局部变量:编译器只需创建本地链接器符号,保证它们的名字唯一

而对于**全局符号**的引用解析会很棘手:
当编译器遇到不在当前模块定义的符号,就假设该符号由其他模块定义(外部符号),
生成链接器符号表条目,交给链接器.
在链接中,如果任何模块中找不到这个被引用符号的定义,就报错.

#### 多重定义的全局符号

如果多个模块定义了同名的全局符号?

- 认为:**函数**,**已初始化的全局变量**是**强符号**
- 认为:未初始化的全局变量是**弱符号**

采用以下规则:
1. 不允许多个同名的**强符号**(函数与已赋值全局变量)
2. 强符号与弱符号**重名**,选择**强符号**
3. 多个**弱符号同名**,在弱符号中**任选**一个


```
/*foo1.c*/
int main(void){
    return 0;
}

/*bar1.c*/
int main(){
    return 0;
}
```
在两个模块中main都是强符号,这会报错:强符号被重复定义

```
/*foo2.c*/
int x =15213;
int main(){
    return 0;
}

/*bar2.c*/
int x =15213;
void f(){}
```
两个模块中已初始化的x为强符号,重名,报错

```
/*foo3.c*/
int x =15213;
int main(){
    return 0;
}

/*bar3.c*/
int x;
void f(){
    x=15212;
}
```
此时,foo3.c中的x是强符号,而bar3.c中的x为弱符号
选择强符号:x = 15213;
在链接后bar3.c的f()将会把x=15213改成15212.
```
/*foo5.c*/
#include<stdio.h>
void f(void);
int y = 15212;
int x = 15213;
int main(void){
    f();
    printf("x= 0x%x y=0x%x \n",x,y);
    return 0;
}

/*bar5.c*/
double x;
void f(){
    x = -0.0;
}
```
当一强一弱,又遇到类型不同: int 4B,double 8B
链接时,选择强符号int x = 15213;
内存如下:
```
address  :type  data
0x601020 :int   x
0x601024 :int   y
```
此时,bar5.c中的f()将x赋值双精度的-0.0
大小8B
这将覆盖整个x与y所在的地址

所以在编译时,遇到未定义的全局符号(**弱符号**),我们无法预测链接器使用多重定义中的哪一个,所以把它分配进**伪节COMMON**,留给链接器决定采用哪一个弱符号
如果遇到**强符号**,我们有足够的自信将其**分配为.data与.bss** 如果其他模块中重复定义了强符号直接报错即可,如果其他模块中重复定义的是弱符号,则取强符号.

### 静态库链接

静态库:把所有目标模块打包,用作链接器的输入

- 如果把所有标准函数打包成一个可重定向目标模块:浪费内存资源
- 如果拆分成不同的可重定向目标模块:需要显式指定链接谁:麻烦

所以:把相关的目标模块打包在一起
只复制被程序引用的目标模块到可执行文件

>简答题 什么是静态库 优点是什么 P475
编译时把相关的目标模块打包成一个单独的文件用作链接器的输入,称这个文件为静态库
使用时,通过把相关函数编译成独立模块并封装在一个单独的静态库文件,在命令行上指定单独的文件名来使用这些在库中定义的函数,链接器只需要复制被引用的目标模块,减少了可执行文件的大小.同时应用程序员只需要包含较少库文件的名字


Linux下静态库以存档(archive)格式存放,后缀.a
如C标准库:libc.a 数学库:libm.a
实际上C编译器驱动程序总是传送libc.a给链接器,可以不显式给出

使用:
```
gcc -c main.c -o main.o
gcc -static -o prog main.o ./libvector.a
//也可以:
gcc -static -o prog main.o -L. -lvector
// -static指建立完全链接的可执行文件,可直接加载到内存运行,无需进一步链接
// -L表示在当前目录下找链接库
```
链接器运行时,认定main.c引用了libvector.a中的addvec.o定义的addvec符号
于是复制addvec.o到可执行文件,没有被引用的则不管

### 链接器的命令解析

链接器从左到右按照命令行上的顺序扫描可重定位目标文件与archive文件
>gcc [file] [archive]

定义集合E(executive),U(undefined),D(defined)
E存放我们认定链接在一起的目标模块
U表示未被定义的符号集合
D表示已被定义的符号集合
一开始,E,U,D都为空

1. 若读入一个输入文件(.o),则加入E,解析其符号定义与引用,并修改U与D
2. 若读入一个存档文件(.a),则在U中匹配未解析的符号,找到匹配的符号,加入库中对应的目标文件(.o)到E,并修改U与D
3. 从左到右依次解析,若符号全都匹配成功,此时U为空,合并E中的目标文件,输出可执行文件
4. 若非空,说明有引用的符号未找到其定义,链接不全,报错

考虑:
```
gcc -static ./libvector.a main.c
```
先读到静态库,此时E,U,D均为空,没有任何库成员加入目标文件集合.
读入main.c后解析结束,报错

这意味着使用gcc命令链接库时,应把存档文件放在目标文件后
且如果库与库之间**不独立**,也要对库的先后顺序进行排序.
```
gcc foo.c libx.a liby.a libz.a
#调用顺序关系: libx -> liby -> libz
读入x.a的同时,多了对y的引用,需要进一步解析
一直到z完成所有的解析
```
才能保证所有的符号都解析完成

如果存在相互调用:libx -> liby, liby->libx
则必须重复出现:
```
gcc foo.c libx.a liby.a libx.a
```

```
p.o->x.a->y.a
y.a->x.a->p.o
//最小命令:最后不加p.o 因为p.o是目标文件 上面的顺序是对于链接库的顺序而言的
gcc p.o x.a y.a x.a
```

### 重定位

重定位,即合并输入模块,为每个符号分配运行地址:

- 重定位节和符号定义:合并相同类型的节,将运行时的内存地址赋给新的聚合节,也赋给输入模块定义的每个符号
- 重定位节中的符号引用:修改对符号的引用使其指向正确的运行地址 这个步骤需要的数据结构是**重定位条目**

#### 重定位条目

对于符号定义,我们通过合并,就可以赋给他们运行地址:ADDR(s)
而对于引用,我们并不知道引用具体指向哪,所以提出了重定位条目
对于目标位置未知的引用,编译器生成一个重定位条目,告诉链接器把目标文件合并成可执行文件时如何修改这个引用
代码的重定位条目位于.rel.text
已初始化数据的重定位条目位于.rel.data

我们的目的是在链接时借助重定位条目把不知道具体位置的引用翻译成某个实际运行地址.
对于函数,我们将给出其跳转地址
对于变量,我们将给出其所在位置


```
typedef struct{
    long offset;     //节内偏移
    long type:32,    //重定位类型
         symbol:32;  //符号表index
    long addend;     //一个有符号常数,一些类型的重定位需要使用它做偏移调整
}Eld64_Rela;
```
我们只关心两种基本的重定位类型:
- R_X86_64_PC32 :使用32位的PC相对地址引用:距离PC的偏移量
- R_X86_64_32   :使用32位绝对地址引用 绝对寻址

算法伪代码如下:
```
foreach section s{
    foreach relocation entry r{
        refptr = s + r.offset //重定位引用地址

        //PC相对地址引用
        if(r.type == R_X86_64_PC32){
            refaddr = ADDR(s) + r.offset; //运行时地址
            *refptr = (unsigned) (ADDR(r.symbol) + r.addend - refaddr);
        }
        //绝对地址引用
        if(r.type == R_X86_64_32){
            *refptr = (unsigned) (ADDR(r.symbol) + r.addend);
        }
    }
}

```
PC相对地址引用:
**符号所在地址**与**对符号的引用所在地址**差了多少,并加上一个**调整值**
当执行到对符号的引用,查看重定位条目,计算出相对值.
此时,原PC值压入栈,更新PC值为PC + (*refptr) 就可以找到符号所在地址.
当目标函数结束后ret,原PC值出栈
CPU继续执行对该函数引用的下一条指令

对于非函数符号

#### 重定位引用

```
/*main.c*/
int sum(int *a,int n);      //函数,未初始化,弱符号
int array[2] = {1,2}        //全局变量,已初始化,强符号
int main(){                 //局部符号main
    int val = sum(array,2);
    return val;
}

/*sum.c*/
int sum(int *a,int n){      //函数,已初始化,强符号
    int i,s=0;
    for(i=0;i<n;i++)
        s += a[i];
    return s;
}
```
以下是main.o的反汇编代码
```
1   0000000000000000 <main>:
2   0 : 48 83 ec 08         sub      $0x8,%rsp
3   4 : be 02 00 00 00      mov      $0x2,%esi
4   9 : bf 00 00 00 00      mov      $0x0,%edi
5   a :                 R_X86_62_32 array            Relocation entry
6   e : e8 00 00 00 00      callq    13<main+0x13>   sum()
7   f :                 R_X86_62_PC32 sum-0x4        Relocation entry
8   13: 48 83 c4 08         add      $0x8,%rsp
9   17: c3                  retq
```

- 重定位PC相对引用:sum
- 重定位绝对引用:array

#### PC相对地址引用

PC相对地址引用需要PC值的参与

call指令开始于节偏移0xe的位置
其中,操作码为0xe8,后面则跟着占位符: 00 00 00 00
这些占位符代表了相对于PC偏移的量,目前还没有计算出

其中sum的重定位条目:
```
r.offset = 0xf  //位于偏移量0xf处
r.symbol = sum
r.type   = R_X86_62_PC32
r.addend = -4
```

```
假设:
ADDR(s) = ADDR(.text) = 0x4004d0       //节运行地址  ,该引用所在节的位置
ADDR(r.symbol) = ADDR(sum) = 0x4004e8  //符号运行地址,即sum()的位置
//可求引用地址: 在节内
refaddr = ADDR(s) + r.offset
        = 0x4004d0 + 0xf
        = 0x4004df
//根据算法得到这个值,PC相对引用,所以是PC加上该值
*refptr = (unsigned) (ADDR(r.symbol) + r.addend - refaddr)
        = (unsigned) (0x4004e8       + (-4)     - 0x4004df )
        = (unsigned) 0x5
```
我们得到可执行文件:
```
4004de: e8 05 00 00 00     callq 4004e8 <sum>   sum()
//call存放在0x4004de
//CPU执行call时,PC值为0x4004e3(call的下一条指令,0xx4004de+0x5)
//此时函数调用,PC压入栈,PC+0x5得到4004e8,修改PC值
//该值指向sum()的第一条指令 当sum()抵达ret,则原PC值出栈,执行call的下一条指令
```

#### 绝对地址引用

绝对地址非常简单,它直接给出了其位置
array的重定位条目:
```
r.offset = 0xa
r.symbol = array
r.type   = R_X86_62_32
r.addend = 0
```
我们可以计算array的引用地址:
```
ADDR(r.symbol) = ADDR(array) = 0x601018

*refptr = (unsigned) (ADDR(r.symbol) + r.addend)
        = (unsigned) (0x601018       + 0       )
        = (unsigned) (0x601018)
```
那么经过重定位,我们可以得到:
```
.text:
4004d9: bf 18 10 60 00     mov $0x601018,%edi    %edi = &array

.data:
0000000000601018 <array>:
601018: 01 00 00 00 02 00 00 00
```

至此,各个节中所有对其它模块的符号引用都被重定位到了某个具体的内存位置.


### 可执行目标文件

C程序最终被转化为一个二进制文件,包含了加载程序到内存并运行它所需的所有信息.

```
//代码段,只读
ELF头      
段头部表     :将连续的节映射到内存段
.init       :定义了一个小函数_init,程序的初始化代码会使用它
.text      
.rodata

//数据段,读/写
.data      
.bss

//可执行文件已完全链接,rel段消失

//不加载到内存的符号表与调试信息
.symtab
.debug
.line
.strtab
节头部表    :描述目标文件的节
```
其中程序头部表(program header table)描述了连续的片(chunk)到连续内存段的映射关系
它描述了各段的读写权限,起始位置,偏移,大小,等等;也包括对齐要求,这种对齐是一种优化.

在Linux shell下
```
linux> ./prog
```
prog不是任何shell内置命令,shell会认为prog是一个可执行文件,调用称为加载器(loader)的操作系统代码来运行它
每个Linux程序运行时都有一个内存映像 P485
加载器运行时,在程序头部表的引导下将片复制到代码段与数据段
然后跳转到程序的入口点,即_start的地址

### 动态链接共享库

静态库的一个问题是需要定期维护与更新,
程序员需要了解该库的最新更新情况,并显式地将他们的程序与更新了的库重新链接
另一个问题是,考虑几乎每个C程序都会使用的标准I/O,每个程序都会把这些函数的代码复制到每个运行进程的文本段,这造成了内存空间的极大浪费.

于是提出了**共享库**(shared library),允许在运行和加载时,可以加载到任意的内存位置,并与内存里的程序链接.称之为**动态链接**(dynamic linking),由**动态链接**器执行
微软的操作系统大量地使用了共享库,即**DLL**(动态链接库)

共享库是一个**目标模块**,也称共享目标(shared object),**.so**后缀
```
gcc -shared -fpic -o libvector.so addvec.c multvec.c
```
-shared指示生成一个共享的目标文件
-fpic指示生成与位置无关代码

随后我们可以将其链接
```
gcc -o prog main2.c libvector.so
```
此时我们得到的prog是**部分链接**的可执行目标文件
这个文件的形式可以使它运行时与libvector.so链接
此时,**没有**任何libvector.so的代码和数据节被复制到prog中,
链接器只复制了一些重定位,符号表信息,使得prog可在运行时解析libvector.so中的引用

### 从应用程序中加载和链接共享库

上一节属于加载时链接,而如何在运行时链接:
Linux为动态链接器提供了简单的接口 允许应用程序在运行时加载和链接共享库
```
#include<dlfcn.h>

void *dlopen(const char*filename,int flag);

void *dlsym(void *handle,char *symbol);

int dlclose(void *handle)
```
dlsym:指向前面已经打开了的共享库,输入一个symbol名字,返回符号的地址
dlclose:卸载共享库


### 位置无关代码 非重点

### 库打桩 非重点

给定一个需要打桩的目标函数,创造一个包装函数.他的原型和目标函数一样
使用某种特殊的打桩机制,欺骗系统调用包装函数,然后再调用目标函数,再将目标函数得到返回值传递回调用者

可以发生在编译/链接/运行时.

