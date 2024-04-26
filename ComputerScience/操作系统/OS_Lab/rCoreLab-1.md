# 第一章:应用程序与基本执行环境

## gdb的使用

```
$ gdb main1
$ r             //或run,运行程序
$ disassemble    //反汇编
$ exit           //退出
```

## 除以0

```
//run:
Program received signal SIGFPE, Arithmetic exception.

//disassemble:
Dump of assembler code for function main:
   0x0000555555555149 <+0>:     endbr64
   0x000055555555514d <+4>:     push   %rbp
   0x000055555555514e <+5>:     mov    %rsp,%rbp
   0x0000555555555151 <+8>:     mov    $0x1,%eax
   0x0000555555555156 <+13>:    mov    $0x0,%ecx
   0x000055555555515b <+18>:    cltd
=> 0x000055555555515c <+19>:    idiv   %ecx
   0x000055555555515e <+21>:    mov    %eax,%esi
   0x0000555555555160 <+23>:    lea    0xe9d(%rip),%rax        # 0x555555556004
   0x0000555555555167 <+30>:    mov    %rax,%rdi
   0x000055555555516a <+33>:    mov    $0x0,%eax
   0x000055555555516f <+38>:    call   0x555555555050 <printf@plt>
   0x0000555555555174 <+43>:    mov    $0x0,%eax
   0x0000555555555179 <+48>:    pop    %rbp
   0x000055555555517a <+49>:    ret
End of assembler dump.
```

程序在执行idiv时发生除以0异常
对应信号SIGFPE,对应行为是终止

值得注意的是异常与信号处理是与架构相关的
该Linux运行在x86架构下,/0对应浮点异常
而在RISC-V架构下除以0不是异常,而有确定的结果

## Rust使用

>以使用 cargo new 创建项目。
可以使用 cargo build 构建项目。
可以使用 cargo run 一步构建并运行项目。
可以使用 cargo check 在不生成二进制文件的情况下构建项目来检查错误。
有别于将构建结果放在与源码相同的目录，Cargo 会将其放到 target/debug 目录。

```
$ cargo new os --bin
```
使用cargo建立新rust项目
--bin表示创建一个可执行程序项目而不是函数库项目

```
tree os
os
├── Cargo.toml  //项目配置,如作者信息联系方式,库依赖等
└── src         //源代码
    └── main.rs //helloworld!
```
进入os目录下
```
$ cargo run //构建并运行
```

应用程序调用了标准库
标准库则需要系统调用实现
打印Hello,world时得到println!宏由Rust标准库std提供

## 目标平台与目标三元组

预处理阶段,实际上是把**源代码的宏展开**
编译器在进行编译与链接时需要知道程序在哪个平台上运行
- 如果用户态基于的内核会导致系统调用接口不同
- 底层硬件不同 对硬件资源的访问方式也有差异

某些编译器支持同一份源代码无需修改即可编译到不同的目标平台上运行,这种情况下源代码是**跨平台**的
而对于另一些编译器,他们本身已经预设好了固定的目标平台

Rust编译器通过**目标三元组(Target Triplet)**来描述程序运行的目标平台

```
$ rustc --version --verbose
rustc 1.76.0-nightly (f5dc2653f 2023-11-25)
binary: rustc
commit-hash: f5dc2653fdd8b5d177b2ccbd84057954340a89fc
commit-date: 2023-11-25
host: x86_64-unknown-linux-gnu
release: 1.76.0-nightly
LLVM version: 17.0.5
```
host给出了默认的目标平台是x86_64-unknown-linux-gnu
这意味着CPU架构是x86,厂商unknown,操作系统是linux.

我们希望他能在另一个硬件平台上运行hello,world
并把CPU架构从x86_64换成RISC-V
```
$ rustc --print target-list | grep riscv
riscv32gc-unknown-linux-gnu
riscv32gc-unknown-linux-musl
riscv32i-unknown-none-elf
riscv32im-unknown-none-elf
riscv32imac-esp-espidf
riscv32imac-unknown-none-elf
riscv32imac-unknown-xous-elf
riscv32imc-esp-espidf
riscv32imc-unknown-none-elf
riscv64-linux-android
riscv64gc-unknown-freebsd
riscv64gc-unknown-fuchsia
riscv64gc-unknown-hermit
riscv64gc-unknown-linux-gnu
riscv64gc-unknown-linux-musl
riscv64gc-unknown-netbsd
=> riscv64gc-unknown-none-elf
riscv64gc-unknown-openbsd
riscv64imac-unknown-none-elf
```
命令给出了当前Rust编译器支持的RISCV目标平台
我们选择riscv64gc-unknown-none-elf
表示CPU架构是riscv64gc,厂商Unknown,操作系统none
elf表示没有标准的运行时库,意味着没有任何系统调用的封装支持
这样的平台称之为**裸机平台**（bare-metal),其上运行的软件没有传统的操作系统支持

Rust语言标准库需要操作系统的支持,如果我们期望运行一个裸机上的操作系统,就不能直接使用Rust标准库

幸运的是,Rust有一个对标准库裁剪后的核心库core
core库不需要任何操作系统支持,但功能也比较受限,但包含了Rust相当一部分的核心机制,可以满足我们的大部分功能需求.

Rust是一种面向系统(包括操作系统)开发的语言,这意味着在Rust语言省泰中有很多第三方库也不依赖标准库std而仅仅依赖核心库core.

对他们的使用可以很大程度上减轻我们的变成负担.
我们现在需要将对于标准库std的引用换成核心库core

## 移除标准库依赖

本章的目标是构建一个内核最小执行环境使其能在risv64gc架构裸机上运行,我们称其"三叶虫"OS

为了移除对Rust std标准库的依赖,我们需要添加能够支持应用的裸机级别的**库操作系统**Library OS,LibOS

>LibOS以函数库的形式存在,为应用程序提供操作系统的基本功能.
它源于MIT在1996年对Exokernel外核的OS结构研究,该架构把传统得到单体内核分为两部分
一部分以库操作系统LibOS与应用程序紧密耦合来实现传统的OS抽象,并进行面向应用的剪裁与优化
另一部分即外核,仅专注于最基本的安全复用物理硬件机制上,为LibOS提供基本的硬件访问服务.
这样的设计思路可以针对应用程序的特征定制LibOS来达到高性能的目的
由于定制一个LibOS的工作量大且难以重复使用,人力成本因素导致了它不太被工业界认可

### 移除println!宏

```
$ rustup target add riscv64gc-unknown-none-elf  //为rustc添加一个新target
```

在os目录下新建.cargo/config
```
[build]
target = "riscv64gc-unknown-none-elf"
```
这会对Cargo工具在os目录下的行为进行调整:
现在会默认使用riscv64gc作为目标平台而不是原先默认的x86
这种编译器运行的开发平台(x86)与目标平台(riscv)不同的情况称**交叉编译**

添加 #![no_std] 告诉编译器不要使用std
此时运行cargo run,将会提示
```
cannot find macro `println` in this scope
```
这是因为我们移除了std依赖
现在把println注释掉,

重新构建运行
```
#[panic_handler]` function required, but not found
```

panic!宏用于处理无法恢复的致命错误(panic)
Rust编译器在编译程序时从安全性 考虑需要有panic!宏的具体实现
std提供了它的具体实现,大致功能是打印出错位置和原因并杀死应用
而核心库core中只有一个panic!宏的空壳,里面没有任何内容
我们需要先实现一个简陋的panic处理函数才能让LibOS的编译通过

### panic函数

创建一个新的子模块lang_item.rs来实现panic函数并通过#[panic_handler]通知编译器用panic来对接panic!宏
```
lang_item.rs:

use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo)->!{
   loop{}
}
```
并在main.rs中添加:
```
#![no_std]
mod lang_items;
...
```

其中,#[panic_handler]是一种编译指导属性,它标记了core中panic!宏要对接的函数
它所标注的函数需要具有fn(&PanicInfo) -> !函数签名
函数可以通过PanicInfo数据结构获取错误相关信息
这样Rust编译器就可以把core库中的panic!宏定义与#[panic_handler]指向的panic函数合并在一起
使得no_std程序具有类似std库对应致命错误的功能

### Rust模块化编程

把一个项目划分为多个子模块是显而易见的道理
每个通过Cargo工具创建的Rust项目都是一个模块,根据项目类型不同,模块的**根**所在位置也不同
- --bin参数表示可执行Rust项目,根为src/main.rs
- --lib参数表示创建一个Rust库项目 根为src/lib.rs
在根文件中声明所有可能会用到的子模块
如在main.rs中通过mod lang_items声明了子模块lang_items
他的实现为lang_item.rs
当子模块比较复杂,也可以放在src同目录下与子模块同名的子目录之下
Rust默认模块的内容私有,private,除非被显式地声明为pub的内容才会对其他模块公开
通过使用绝对路径/相对路径来引用其他模块或当前模块的内容
如 use core::panic::PanicInfo

```
//lang_items.rs
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo)->!{
    loop{}
}

//main.rs
#![no_std]
mod lang_items;

fn main() {
        //println!("Hello, world!");
}
```
上述为两个模块
根模块引用了子模块lang_items
items引用了core核心库中panic/PanicInfo
PanicInfo传入panic函数 其函数内容为原地loop

### 移除main函数

重新cargo build
```
error: requires `start` lang_item
error: could not compile `os` (bin "os") due to previous error
```
我们缺少一个名为start的语义项
在执行应用程序之前,标准库和第三方库作为应用程序的执行环境需要做一些初始化工作,之后才跳转到入口点main函数.
start代表了标准库std在执行应用程序之前需要进行的初始化工作
由于禁用了std,编译器自然找不到它的实现

这里提供一个最简单的解决方案:压根不让编译器使用这项功能.
在main.rs中加入#![no_main] 告诉编译器我们没有一般意义上的main函数,并将原先的main函数删除.

在没有main函数的情况下编译器也就不需要做所谓的初始化工作了
```
#![no_main]
mod lang_items;

//fn main() {
//        println!("Hello, world!");
//}
```
重新cargo build,编译成功。

我们成功地脱离了标准库,通过了编译器的检验
但现在什么功能也没有


```
$ rust-readobj -h target/riscv64gc-unknown-none-elf/debug/os

File: target/riscv64gc-unknown-none-elf/debug/os
Format: elf64-littleriscv
Arch: riscv64
AddressSize: 64bit
LoadName: <Not found>
ElfHeader {
  Ident {
    Magic: (7F 45 4C 46)
    Class: 64-bit (0x2)
    DataEncoding: LittleEndian (0x1)
    FileVersion: 1
    OS/ABI: SystemV (0x0)
    ABIVersion: 0
    Unused: (00 00 00 00 00 00 00)
  }
  Type: Executable (0x2)
  Machine: EM_RISCV (0xF3)
  Version: 1
  Entry: 0x0
  ProgramHeaderOffset: 0x40
  SectionHeaderOffset: 0x1B70
  Flags [ (0x5)
    EF_RISCV_FLOAT_ABI_DOUBLE (0x4)
    EF_RISCV_RVC (0x1)
  ]
  HeaderSize: 64
  ProgramHeaderEntrySize: 56
  ProgramHeaderCount: 4
  SectionHeaderEntrySize: 64
  SectionHeaderCount: 14
  StringTableSectionIndex: 12
}
```
对其进行分析可以看到它的入口地址Entry是0
经验告诉我们0一般对应NULL与空指针 所以为0的入口地址无法对应任何指令
如果对其进行反汇编可以看到没有生成汇编代码
这个二进制程序虽然合法 但它是一个空程序
因为它没有进行任何有意义的工作,甚至由于移除了main而没有传统意义上的入口点.
因此Rust编译器会生成一个空程序

## 内核的第一条指令

### Qemu模拟器

以下是os/Makefile中启动Qemu的部分
```
$ qemu-system-riscv64 \
    -machine virt \
    -nographic \
    -bios ../bootloader/rustsbi-qemu.bin \
    -device loader,file=target/riscv64gc-unknown-none-elf/release/os.bin,addr=0x80200000
```
- --machine virt 表示将模拟的计算机设置名为virt
- -nographic 表示无图形页面
- -bios设置了bootloader的位置
- -device loader表示在qemu开机之前将宿主机上的一个文件加载到qemu的物理内存的指定位置

下面是另一个写法
```
QEMU_ARGS := -machine virt \
    -nographic \
    -bios $(BOOTLOADER) \
    -device loader,file=$(KERNEL_BIN),addr=$(KERNEL_ENTRY_PA)
```
下面是常量内容
```
# Building
TARGET := riscv64gc-unknown-none-elf
MODE := release
KERNEL_ELF := target/$(TARGET)/$(MODE)/os
KERNEL_BIN := $(KERNEL_ELF).bin

# BOARD
BOARD := qemu
SBI ?= rustsbi
BOOTLOADER := ../bootloader/$(SBI)-$(BOARD).bin

# KERNEL ENTRY
KERNEL_ENTRY_PA := 0x80200000
```

Qemu的启动流程分为三个阶段
在Qemu模拟的virt中,物理内存的起始位置为0x80000000,默认大小为128MB
如果使用默认大小则其物理内存地址范围是[0x80000000,0x88000000)
Qemu在开始执行任何指令之前,首先把两个文件加载到物理内存中,
即把bootloader/rustsbi-qemu.bin加载到0x80000000位置,
同时把内核镜像os.bin加载到**KERNEL ENTRY**,0x80200000

### 加电启动流程

计算机加电启动流程可以分为若干个阶段,每个阶段由一层软件或固件负责,然后跳转到下一层的入口地址
Qemu为了模拟计算机加电启动的流程分为三个阶段:
- 阶段1,Qemu内的一小段汇编
  - 将必要的文件就在如Qemu物理内存
  - QemuCPU的PC初始化为0x1000,对应Qemu实际执行的第一条指令位置
  - 执行几条指令后跳转到0x80000000,进入第二阶段
- 阶段2,bootloader负责
  - 我们需要保证负责该阶段的bootloader位于0x80000000
  - bootloader负责进行一些初始化工作,并进入下一阶段的入口处
  - 对于不同的bootloader而言,下一阶段的入口不一定相同,RustSBI则约定为固定的0x80200000
  - RustSBI,RISC-V Supervisor Binary Interface 由于RISC-V的硬件与其他指令集不同,可以通过SBI的方式让内核感知并操控硬件
- 阶段3,内核镜像负责
  - 控制权转移给操作系统

### 内核的第一条指令

```
# os/src/entry.asm
  .section .text.entry
  .globl _start
_start:
  li x1, 100
```

.section .text.entry的含义是将后面的内容放在一个名为text.entry的段中
所有的代码被保存在.text段,而这里命名为.text.entry来区分其他.text的目的是确保这一段能被放置在比任何其它代码段更低的地址,这样才能最先被执行

.globl _start的含义是生成了全局符号_start
该符号指向紧跟在其后面的内容
li为LoadImmediate
表示将立即数100载入寄存器x1

随后,将其嵌入main
```
// os/src/main.rs
#![no_std]
#![no_main]

mod lang_items;

use core::arch::global_asm;
global_asm!(include_str!("entry.asm"));
```
使用核心库global_asm!宏与include_str!宏将同目录下的entry.asm转化为字符串嵌入代码

### 链接脚本

链接器的默认行为不符合我们的要求,修改Cargo的配置文件来应用我们自己的链接脚本
在.cargo/config里增加:
```
 [target.riscv64gc-unknown-none-elf]
 rustflags = [
     "-Clink-arg=-Tsrc/linker.ld", "-Cforce-frame-pointers=yes"
 ]
```
我们的链接脚本 os/src/linker.ld如下
```
OUTPUT_ARCH(riscv)            //设置目标平台为riscv
ENTRY(_start)                 //把程序的入口点定为我们先前定义的全局符号_start
BASE_ADDRESS = 0x80200000;    //定义一个基址 内核应该在的位置

SECTIONS                      //以下体现了链接中对段的合并
{                             //.表示为当前地址 我们可以对.赋值来调整接下来的段放哪,也可以给一些全局符号赋值为.
    . = BASE_ADDRESS;         //改变.的基址
    skernel = .;              /start

    stext = .;                //start
    .text : {                 //冒号前表示最终生成的可执行文件中的段名
        *(.text.entry)        //大括号内表示按照顺序将所有输入目标文件的哪些段,这里先.entry放在.text段的最开头 这样所有的段都从BASE_ADDRESS开始放置
        *(.text .text.*)      //放入这个段中 每行格式为<finename>(sectionname)
    }                         //*为通配符 这一段表示合并*.text.entry和*.text.其他为.text段

    . = ALIGN(4K);            //4K对齐算好当前地址
    etext = .;                //end 每个段都有两个全局符号给出开始和结束地址
    srodata = .;              //接下来轮到.rodata段,如此类推直到所有段合并
    .rodata : {
        *(.rodata .rodata.*)
        *(.srodata .srodata.*)
    }

    . = ALIGN(4K);
    erodata = .;
    sdata = .;
    .data : {
        *(.data .data.*)
        *(.sdata .sdata.*)
    }

    . = ALIGN(4K);
    edata = .;
    .bss : {
        *(.bss.stack)
        sbss = .;
        *(.bss .bss.*)
        *(.sbss .sbss.*)
    }

    . = ALIGN(4K);
    ebss = .;
    ekernel = .;

    /DISCARD/ : {
        *(.eh_frame)
    }
}
```

重新构建, 以release模式生成可执行文件 位置在/release/os
```
$ cargo build --release
```

可以用file文件查看
```
~/workfield/ch1/os/target/riscv64gc-unknown-none-elf/release$ file os
os: ELF 64-bit LSB executable, UCB RISC-V, RVC, double-float ABI, version 1 (SYSV), statically linked, not stripped
```
可以用file文件查看

### 手动加载内核可执行文件

我们得到的ELF os文件符合我们对内存布局的需求,但我们不能将其直接提交给Qemu
因为在代码段与数据段之外还夹着多余的一些元数据,Qemu无法处理这些多余的元数据
我们需要丢弃内核可执行文件中得到元数据来得到内核镜像os.bin:
```
$ rust-objcopy --strip-all target/riscv64gc-unknown-none-elf/release/os -O binary target/riscv64gc-unknown-none-elf/release/os.bin
```
我们可以使用stat来检查这两个文件的区别
```
$ stat os
Size: 5272            Blocks: 16         IO Block: 4096   regular file
$ stat os.bin
Size: 4               Blocks: 8          IO Block: 4096   regular file
```
内核镜像的大小只有4B,因为它里面质保函我们在entry.asm写的一条指令 li x1,100
一般情况下RISC-V架构的一条指令位宽为4B,而内核可执行文件包含了两部分元数据,用于帮助我们更灵活地加载和使用可执行文件,比如在加载时完成重定位,动态链接.
但是由于Qemu的加载功能过于简单,也无法动态链接,我们只能将其丢弃再交给Qemu.
我们可以理解为我们手动帮Qemu完成了可执行文件的加载

### GDB调试

在两个终端里分别执行下面两个命令
```
qemu-system-riscv64 \
    -machine virt \
    -nographic \
    -bios ../bootloader/rustsbi-qemu.bin \
    -device loader,file=target/riscv64gc-unknown-none-elf/release/os.bin,addr=0x80200000 \
    -s -S

riscv64-unknown-elf-gdb \
    -ex 'file target/riscv64gc-unknown-none-elf/release/os' \
    -ex 'set arch riscv:rv64' \
    -ex 'target remote localhost:1234'
```
第一个用来启动qemu,他的参数同之前在makefile中见到的一样。也可以图方便写个makefile
参数-s使Qemu监听本地TCP端口1234,等待GDB客户端链接
--S使Qemu在收到GDB请求后再开始运行。如果想直接运行Qemu则删掉-s -S
下面的命令用于启动GDB客户端,连接到Qemu输出如下

```
[GDB output]
0x0000000000001000 in ?? ()
```
PC为0x1000,Qemu处于启动的第一阶段
```
$ x/10i $pc
=> 0x1000:      auipc   t0,0x0
   0x1004:      addi    a2,t0,40
   0x1008:      csrr    a0,mhartid
   0x100c:      ld      a1,32(t0)
   0x1010:      ld      t0,24(t0)
   0x1014:      jr      t0
   0x1018:      unimp
   0x101a:      0x8000
   0x101c:      unimp
   0x101e:      unimp
```
x/10i的含义是在当前PC开始,反汇编10条指令。
可以看到Qemu固件只包含了几条指令,后面全是数据。
数据为0会被反汇编成unimp指令。
0x101a处的0x8000则是跳转到下一阶段的关键。
我们可以使用si命令单步调试,我们会发现
在执行jr t0跳转之前,寄存器t0的值就会设置为0x80000000

基于同样的道理我们可以反汇编RustSBI最初的几条指令
但RustSBI超出了知识范围,我们并不深入。我们检查一下控制权能否被转交给内核。

```
$ (gdb) b *0x80200000
Breakpoint 1 at 0x80200000
```
b,breakpoint,设置断点。在0x80200000,第三阶段的入口处设置断点
然后执行c,continue命令
```
$ (gdb) c
Continuing.

Breakpoint 1, 0x0000000080200000 in ?? ()
```
此时我们停在了0x80200000,检查一下内核的第一条指令是否被正确执行。
```
$ (gdb) x/5i $pc
=> 0x80200000:  li      ra,100
   0x80200004:  unimp
   0x80200006:  unimp
   0x80200008:  unimp
   0x8020000a:  unimp
```
这是我们在entry.asm中编写的第一条指令
ra是寄存器x1的别名

## 为内核支持函数调用

内核的第一条指令是在汇编entry.asm中手写的。
为了使用Rust来实现我们的内核，我们需要将控制权转交给Rust编写的内核入口函数。我们需要先手写若干汇编代码进行一些初始化操作。我们首先需要设置栈空间,这样内核就可以支持函数调用，随后我么就可以调用Rust编写的内核入口函数了。

### 函数调用与RISC-V寄存器

在函数调用,控制传递给被调用函数的时候,需要保存上下文。
在RISCV架构中有两条指令符合这样的特征:
```
jal rd,imm[20:1]          rd <- pc + 4
                          pc <- pc + imm
jalr rd,(imm[11:0])rs     rd <- pc + 4
                          pc <- rs + imm
```
ret使用ra寄存器保存返回地址,我们要保证他在函数执行的过程中全程不变
考虑到复杂的多层嵌套的函数调用,可能有很多寄存器的值不能改变。
在控制流转移前后需要保持不变的寄存器集合称**函数调用上下文**

由于CPU的寄存器有限,我们需要在物理内存的一个区域保存上下文,在执行结束后再在同样的区域读取恢复上下文。

函数调用上下文中的寄存器分为如下两类:
- 被调用者寄存器 被调用的函数可能会覆盖 而由被调用的函数来保存
- 调用者保存寄存器 被调用的函数可能会覆盖 而由发起调用的函数保存

其过程如下
- 调用函数：先保存**调用者保存寄存器**后再调用 返回后恢复
- 被调用函数：先保存**被调用者保存寄存器**然后再执行 结束后恢复

无论是被调用函数还是调用函数都需要因为调用行为而保存恢复寄存器的汇编代码。可以称之为**开场**(Prologue)与**结尾**(Epilogue),他们由**编译器自动插入**。

在RISC-V架构上的调用规范
- 寄存器a0~a7 调用者保存  用来传递参数,a0与a1还可以保存返回值
- 寄存器t0~t6 调用者保存 临时寄存器 可以随意使用
- 寄存器s0~s11  被调用者保存 临时寄存器 被调用函数保存后才能使用
- zero(x0) 恒为0
- ra(x1)  被调用者保存
- sp(x2)  被调用者保存 栈指针
- fp(s0)  被调用者保存 栈帧指针(frame pointer)
- gp(x3) tp(x4) 程序运行期间不变 不必放在在函数调用上下文

sp与fp共同决定了当前栈中的一块栈帧的位置与大小

### 分配并使用启动栈

在entry.asm中分配启动栈空间,并将控制权在转交给Rust入口之前就将栈指针sp设置为栈顶。
RISC-V中栈与x86一样从高到低增长。

```
  1         .section .text.entry
  2         .globl _start
  3 _start:
  4         la sp,boot_stack_top            //修改了sp栈指针为top
  5         call rust_main                  //把控制权转交给rust_main,在main.rs中实现
  6
  7         .section .bss.stack             //这里先放在.bss段
  8         .globl boot_stack_lower_bound
  9 boot__stack_lower_bound:                //能增长到的下限
 10         .space 4096*16
 11         .globl boot_stack_top
 12 boot_stack_top :                        //栈顶
```
在10行预留了4096*16B,64KB的空间作为栈空间
我们用boot_stack_top来表示栈顶的位置
用boot_stack_lower_bound来标识栈能增长到的下限
他们都设置为全局符号供其他目标文件使用
第7行可以看到这块空间被放在.bss.stack段中.链接后汇聚到.bss段
.bss一般存放未定义,初始化为0的数据,而栈不需要使用前初始化为0,因为函数调用时内容会被覆盖。我们尝试将其放在.data段中但未能成功,所以才将其放在.bss
链接脚本中,sbss与ebss分别分别指向.bss段除了.bss.stack以外的起始与终止地址,在使用这一部分数据之前需要将他们初始化为0.这个过程在下一节中进行。

我们通过call rust_main将控制权转交给Rust代码
该入口点在main.rs中实现。
```
mod lang_items;

//fn main() {
//        println!("Hello, world!");
//}

use core::arch::global_asm;
global_asm!(include_str!("entry.asm"));

#[no_mangle]
pub fn rust_main() -> !{
    clear_bss();            //清零.bss然后循环
    loop{}
}

//内核初始化的一个重要工作就是确保.bss清零
//此处进行对.bss段的清零 我们需要保证被分配到.bss段的全局变量为0
//由于控制权已被转交给Rust,我们可以使用Rust来实现这一功能
fn clear_bss(){
    extern "C"{
        fn sbss();
        fn ebss();
    }
    (sbss as usize..ebss as usize).for_each(|a|{
        unsafe{(a as *mut u8).write_volatile(0)}
    });
}
```
其中#[no_mangle]用于避免编译器对他的名字进行混淆,防止链接时找不到该符号。在函数clear_bss中我们会尝试从其他地方找到全局符号sbss和ebss,他们由链接脚本linker.ld给出,并分别指出.bss段的起始和终止地址。
```
.bss : {
    *(.bss.stack)
    sbss = .;
    *(.bss .bss.*)
    *(.sbss .sbss.*)
}
ebss = .;
```
extern "C" 可以引用一个外部的C函数接口,这里引用位置转化为usize获取他的地址,来得到.bss两端的地址,随后将两者之间的.bss内容归零.
此处使用了Rust的迭代器与闭包的语法,在很多情况下他们可以提高开发效率。也可以改写为等价的for循环。
>在C语言中,把一个地址转化为一个裸指针并修改其值是司空见惯的。
而Rust认为对于裸指针的解引用(Dereference)是一种unsafe的行为,所以要包裹在unsafe块中
Rust进行了更多的语义约束来保证安全性。使用unsafe块来告知编译器不要对他进行完整的约束检查。当代码不能够正常运行时我们往往最先检查unsafe块，因为他没有受到编译器的保护。

## SBI

本节为三叶虫OS的最后一步,基于RustSBI提供的服务在屏幕上打印Hello world与关机
RustSBI介于底层硬件与内核之间,是我们的OS的底层执行环境 它除了为上层应用进行初始化以外,还要为上层应用提供服务

当内核发起请求,计算机转由RustSBI控制来响应,待请求处理完毕后控制权交还给内核.
我们所编写的OS内核处于Supervisor特权级,而RustSBI处于Machine特权级(最高的特权级)
类似RustSBI这样运行在Machine特权级的软件成为**Supervisor Execution Environment**(SEE),即Supervisor执行环境
两层之间的接口被称为**Supervisor Binary Interface**(SBI),即Supervisor二进制接口.
SBI spec标准由RISC-V开源社区维护,规定了SBI接口层要包含的功能,RustSBI按照它实现了需要支持的大部分功能.
不过RustSBI并不是SBI标准的唯一一种实现.

那么如何才能调用RustSBI提供的服务呢?
首先不能函数调用,因为RustSBI是构建后的可执行文件,内核无从得知RustSBI中的符号和地址.
然而RustSBI开源社区的sbi_rt封装了SBI服务的接口,我们可以直接使用.

### 引入sbi_rt依赖

```
// os/Cargo.toml
[dependencies]
sbi-rt = { version = "0.0.2", features = ["legacy"] }
```
带上legacy的feature,因为我们需要用到的串口读写属于SBI的遗留接口
我们将内核与RustSBI通信的相关功能在子模块sbi中实现,并在main.rs中加入mod sbi
在os/src/sbi.rs中直接调用sbi_rt提供的接口来输出字符,同时定义关机功能
```
pub fn console_putchar(c: usize) {
    #[allow(deprecated)]
    sbi_rt::legacy::console_putchar(c);
}
pub fn shutdown(failure: bool) -> ! {
    use sbi_rt::{system_reset, NoReason, Shutdown, SystemFailure};
    if !failure {
        system_reset(Shutdown, NoReason);
    } else {
        system_reset(Shutdown, SystemFailure);
    }
    unreachable!()
}
```

### 格式化输出

console_putchar的功能过于受限,如果想打印一行 Hello world需要进行多次调用
我们尝试自己编写基于console_putchar 的println!宏

在console.rs中实现

```
use crate::sbi::console_putchar;
use core::fmt::{self, Write};       // core::fmt::Write是一个trait 包含一个实现println!宏的方法
                                    //Write trait提供了write_fmt方法
struct Stdout;                      //该结构体不包含任何字段,被称为类单元结构体

impl Write for Stdout {             //为Stdout应用Write trait,来定义接口 write_str方法必须实现
    fn write_str(&mut self, s: &str) -> fmt::Result {
        for c in s.chars() {
            console_putchar(c as usize);
        }
        Ok(())
    }
}

//在为Stdout实现了Write trait需要的write_str方法后,就可以调用他提供的write_fmt方法,进而实现print

pub fn print(args: fmt::Arguments) {
    Stdout.write_fmt(args).unwrap();
}

//下为宏定义
#[macro_export]
macro_rules! print {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!($fmt $(, $($arg)+)?));
    }
}

#[macro_export]
macro_rules! println {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!(concat!($fmt, "\n") $(, $($arg)+)?));
    }
}
```

### Hello,world!

不要忘了重新build并切除元数据
```
cargo build --release
rust-objcopy --strip-all target/riscv64gc-unknown-none-elf/release/os -O binary target/riscv64gc-unknown-none-elf/release/os.bin
```
可喜可贺 可喜可贺
```
[rustsbi] RustSBI version 0.3.1, adapting to RISC-V SBI v1.0.0
.______       __    __      _______.___________.  _______..______   __
|   _  \     |  |  |  |    /       |           | /       ||   _  \ |  |
|  |_)  |    |  |  |  |   |   (----`---|  |----`|   (----`|  |_)  ||  |
|      /     |  |  |  |    \   \       |  |      \   \    |   _  < |  |
|  |\  \----.|  `--'  |.----)   |      |  |  .----)   |   |  |_)  ||  |
| _| `._____| \______/ |_______/       |__|  |_______/    |______/ |__|
[rustsbi] Implementation     : RustSBI-QEMU Version 0.2.0-alpha.2
[rustsbi] Platform Name      : riscv-virtio,qemu
[rustsbi] Platform SMP       : 1
[rustsbi] Platform Memory    : 0x80000000..0x88000000
[rustsbi] Boot HART          : 0
[rustsbi] Device Tree Region : 0x87000000..0x87000ef2
[rustsbi] Firmware Address   : 0x80000000
[rustsbi] Supervisor Address : 0x80200000
[rustsbi] pmp01: 0x00000000..0x80000000 (-wr)
[rustsbi] pmp02: 0x80000000..0x80200000 (---)
[rustsbi] pmp03: 0x80200000..0x88000000 (xwr)
[rustsbi] pmp04: 0x88000000..0x00000000 (-wr)
Hello,world!
[kernel] Panicked at src/main.rs:17 Shutdown machine!

```

## 章节小节

我们实现了一个可以在裸机上运行的Hello,world程序
为了实现它,我们已做了以下工作:
1. 配置Rust编译器的目标三元组,修改了默认的目标平台
2. 移除了Rust程序的标准库依赖,转而使用核心库core,以便在裸机上运行
3. 由于核心库下没有panic宏的具体实现,我们实现了一个简陋的panic处理函数让程序通过编译
4. 程序执行之前需要先跳转到start进行一些初始化工作后再跳转到入口点main,很遗憾他们由标准库完成 我们需要通知编译器程序没有入口点main,这样也就跳过了start
5. 我们会得到一个空程序,入口地址Entry是0,NULL,空指针,反汇编没有生成任何汇编代码,连main也没有,但它仍然是一个合法的二进制程序
6. 我们明晰了Qemu的启动流程,与各阶段对应的起始物理地址 第一阶段由QEMU完成 第二阶段由bootloader完成 我们只关心跳转到第三阶段的内核
7. 我们编写了一个简单的汇编代码,并把它嵌入main中
8. 为了让运行地址符合Qemu的要求,我们需要自定义链接脚本让内核位于正确的内存地址,并配置好cargo的config文件
9. 链接得到的ELF文件仍然有一些多余的元数据来帮助我们灵活的加载和使用可执行文件,Qemu过于简单的加载功能无法处理他们,我们需要丢弃他们
10. 我们配置了GDB调试环境 @TODO:makefile里写好GDB启动内容 并正确检查到内核入口地址成功载入内核
11. 我们期望用Rust来编写内核而不是使用汇编,我们需要把控制权交给Rust,首先在汇编中启动栈空间
12. 在汇编中设置好跳转到Rust程序与栈指针 随后控制权就正常地转交到由Rust编写的程序中 至此我们已经可以用Rust来编写内核了
13. 内核初始化的一项重要步骤便是归零.bss段,在main.rs下rust_main定义 
14. 引入SBI接口,使用RustSBI提供的服务来实现print 最终输出hello,world