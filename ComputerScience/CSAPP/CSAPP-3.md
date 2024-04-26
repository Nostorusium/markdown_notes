# CSAPP章节小结

## 第三章 机器级表示

- **编译运行过程与gcc指令**

  **C文件(text)** -*编译器*-> **汇编文件(text)** -*汇编器*-> **目标程序(binary)** -*链接器*-> **可执行文件(binary)**
  **1. 产生汇编文件.s**
  gcc -S xxx.c
  **2. 产生目标文件.o**
  gcc -C xxx.c
  **3. 反汇编xxx.o**
  objdump -d xxx.o
  **4. 产生可执行文件**
  gcc -o prog

- **数据格式**
  
  Intel传统：以16位即2B为一字
  汇编后缀:
  1B b,byte 字节
  2B w,word 字
  4B l,longword 双字
  8B q,quadword 四字

  |数据类型|长度|汇编后缀|
  |:-|:-|:-|
  char|1B|b
  short|2B|w
  int|4B|l
  long|8B|q
  char*|8B|q

  浮点数：单精度与双精度对应short和long

  |浮点数|长度|汇编后缀|
  |:-|:-|:-|
  float|4B|s
  double|8B|l

- **寄存器**
  |名称|用途|
  |:-|:-|
  %rax|返回值
  %rbx|
  %rcx|arg4
  %rdx|arg3
  %rsi|arg2
  %rdi|arg1
  %rbp|
  %rsp|
  %r8|arg5
  %r9|arg6
  %r10-%r15|

参数位置:
%rdi %rsi %rdx %rcx %r8 %r9

- **操作数指示符**
  访存Imm(rb,ri,s): M[ Imm +R[rb] + R[Ri] * s]
  rb为基址 ri为变址 s为变多少
  访问寄存器 r:R[r]
  访存(r):M[R[r]]

- **指令**
  1. 数据传送
   movb movw movl moq movabsq
   扩展:movz/movs z表示零扩展 s表示符号扩展 加后缀一短一长
  2. 栈
   pushq popq
  3. 算术逻辑运算(此处省略后缀)
   二元:
   add sub imul
   and or xor
   一元:
   inc dec
   neg not
  4. 一元乘除 另一参数在%rax
   imulq mulq idivq divq
   带i表示有符号 不带i表示无符号
  5. leaq
   只读地址不访存
  6. 移位
   S-A/L-L/R

- **条件**
  条件码：CF ZF SF OF carry,zero,sign,overflow

  **比较：**
  cmpb/w/l/q/ A B: 比较B-A为0 正数 负数来修改条件码

  **测试：**
  test用于测试 表示与 A&B
  cmp和test只改变条件码 不改变寄存器内容

  **set**
  后缀:
  对于有符号数 l:less g:greater
  对于无符号数 b:below a:above
  e:equal
  set D,将某个字节根据条件码情况设置为0或1 可视为真或假

  **跳转**
  jmp无条件跳转
  j+后缀 条件跳转
  跳转的寻址是**PC相对寻址** 是PC+偏移
  而PC指向下一条指令的地址
  所以跳转地址=下一条指令的地址+偏移

- **过程**
  参数1~6:%rdi %rsi %rdx %rcx %r8 %r9
  多于6个:放进栈里
  返回值：用%rax传递
  call与ret:通过修改PC值(%rip)控制转移
  call,将返回地址入栈，控制转移到被call的函数
  ret，返回地址出栈，修改%%rip(程序计数器)返回
  callq与retq的后缀q只是为了区分强调x86-64
  局部变量放进栈

- **对齐**
  地址必须是K的倍数 K=2,4,8...
  首地址必须对齐
  进行恰当排序可以节省空间

- **缓冲区溢出**
  C对于数组引用不进行任何越界检查
  考法：
  根据汇编语言写出C语言程序，指出漏洞
  给出攻击方法
  **缓冲区溢出漏洞**：覆盖了返回地址
  防范方法：金丝雀 使用安全的函数 编译加安全选项 栈随机化(一定上)
  简述溢出原理和防范方法：
  **原理：**
  如C对于数组引用不进行任何越界检查，向程序输入缓冲区写入数据时，使用如gets等不限制长度的库函数时
  可能使栈内的缓冲区数据溢出，覆盖栈的内容，如函数的返回地址。函数结束时，从栈中读取返回地址后，
  程序将返回到错误的位置，执行特定的代码，从而达到共计目的。

  **防范：**
  使用限制字符串长度的库函数
  1. 随机化偏移
   多分配一段随机的区域不使用
  2. 限制可执行代码区域
   只允许读和写，不允许执行
  3. 栈的破坏性检查-金丝雀
  在局部缓冲区和栈的状态之间存储一个特殊值作为哨兵值
  随机产生，程序检查该值是否改变，若是则终止程序。
