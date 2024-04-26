# 第二章:批处理系统

本章从两个目标出发:保障系统安全和多应用支持

- 构造包含内核和多个应用程序的单一执行程序
- 通过批处理执行多个程序的自动加载与运行 
- 特权级机制实现对OS自身的保护
- 实现特权级的跨越
- 支持跨特权级的系统调用功能

在早期人们仍使用打孔纸带操控计算机时,系统管理员需要在房间各处跑来跑去,或者等待打印机输出的时间里,计算机都没有在工作,我们希望计算机能够不间断的工作且专注于计算任务本身,**批处理系统**(Batch System)应运而生:在资源允许的情况下自动安排程序的执行,称批处理作业.这个名词源于20世纪60年代的大型机时代,核心思想是把多个程序打包到一起输入计算机,一个程序结束后计算机就自动加载下一个程序到内存并开始执行

当软件有了**代替操作员的管理和操作能力**后,便开始形成**真正意义上的操作系统**了.

而应用程序难免会出现错误,我们希望一个程序的执行错误不会影响其他应用程序,OS和整个系统.这就需要OS能够终止出错的应用程序,转而运行下一个应用程序.这种保护计算机系统不收出错程序破坏的机制称**特权级**(Privilege)机制.他让应用程序运行在用户态,而OS运行在内核态,且实现用户态与内核态的隔离,这需要软硬件的共同努力.

本章主要涉及和实现建立支持批处理系统的**邓氏鱼OS**
本章的目标是让邓氏鱼OS能够感知多个应用程序的存在,并一个接着一个地运行这些应用程序,实现批处理

图详见rCore官方文档第二章

Qemu把包含多个app的列表和BatchOS的image镜像加载到内存中,RustSBI(bootloader)完成硬件初始化后就跳转到邓氏鱼BatchOS的起始位置.
BatchOS首先进行正常运行前的初始化工作:清理bss段和建立栈空间
随后通过AppManager内核模块从app列表依次加载各个app到内存在用户态执行.
app的执行过程中会通过系统调用的方式得到邓氏鱼BatchOS提供的OS服务,如输出字符串

## 导读

我们需要对OS和应用程序进行修改,也需要对应用程序的编译生成过程进行修改.
首先改进应用程序,让他们在用户态执行并能发出系统调用
具体而言编写多个应用小程序,修改编译应用所欲的linker.ld来调整程序的内存布局,让OS能够把应用加载到指定内存地址并顺利启动

在应用程序的运行过程中,OS要支持应用程序的输出,退出功能,这需要实现跨特权级的系统调用接口,以及sys_write sys_exit等具体的系统调用功能
具体设计实现上涉及到内敛汇编的编写,以及应用于OS内核之间系统调用的参数传递的约定
为了让应用程序在邓氏鱼OS实现之前就能在Linux for RISC-V 64上进行运行测试,我们采用了Linux on RISC-V64的系统调用参数约定,这样写完应用后就可以通过qemu-riscv64模拟器进行测试

写完了应用程序后,还需实现批处理.这里首先把相对松散的应用程序执行代码和OS执行代码连接在一起,便于qemu-system-riscv64一次性加载两者到内存,并让OS能找到应用程序的位置.
为了把两者连到一起,需要对生成的应用程序进行改造:
首先把应用程序执行文件从ELF变成Binary格式(通过rust-objcopy)
二进制文件通过编译器辅助脚本变为link_app.S,汇编文件的一部分,并生成辅助信息便于OS能找到应用的位置
编译器会把OS源码与link_app.S合在一起,编译出OS+Binary应用的ELF执行文件,并进一步转变为Binary格式

## 特权级机制

实现特权级机制的根本原因是应用程序运行的安全性不可充分信任
上一章中我们实现的LibOS库操作系统与应用紧密连接在一起,构成一个整体,两者通过编译器形成一个单一的执行程序来执行,即便是应用程序本身的问题也会让OS收到连累.

所以提出特权级机制:
让相对安全可靠的OS运行在一个硬件保护的安全执行环境中,不受应用程序的破坏
而让应用程序运行在另一个无法破坏操作系统的受限执行环境中.
对于应用程序而言这样的限制主要在两个方面:
- 应用程序不能访问任意的地址空间
- 应用程序不能执行某些可能破坏计算机系统的指令

我们还要确保应用程序能得到操作系统的服务,两者应有交互的手段
为了实现这样的机制,需要进行软硬件协同设计.一个比较简洁的方法是处理器设置两个不同安全等级的执行环境
且明确指出可能破坏计算机系统的内核态特权级指令子集,规定他们只能在内核态的执行环境中执行
处理器在执行指令前要进行特权级安全检查,如果在用户态执行环境中执行这些内核态特权级指令就产生异常

采用传统的函数调用方式(call,ret)会绕过硬件的特权级保护检查,为了解决这个问题,RISC-V提供了新的机器指令:
**执行态环境调用指令**(Execution Environment Call,ecall)和**执行环境返回**(Execution Environment Return,eret)

(e,exectution; s,supervisor; m,machine )
- ecall 具有用户态到内核态的执行环境切换能力的函数调用指令
- sret 具有内核态到用户态的执行环境切换能力的函数返回指令,s特指了supervisor
- eret 代表了一类执行环境的返回指令

### RISC-V特权级架构

RISC-V架构中定义了4种特权级:

|级别|编码|名称|
|:-|:-|:-|
0|00|用户/应用模式 U,User/Application
1|01|监督模式 S,Supervisor
2|10|虚拟监督模式 H,Hypervisor
3|11|机器模式 M,Machine

级别的数值越大特权级越高,掌控硬件的能力越强
M处在最高的特权级，而U处于最低的特权级.在CPU硬件层面,**M模式必须存在**,其它模式可以不存在。

到目前为止H的特权规范还没完全制定好,所以并不涉及.
除了M模式以外设下的特权级可以根据跑在CPU上的应用的实际需求进行调整:
- 简单的嵌入式应用:M
- 带有一定保护能力的嵌入式系统需要实现M与U
- 复杂的多任务系统需要实现M/S/U

在上一章中我们实现了简单的支持单个裸机应用的库级别的三叶虫OS,它和应用程序全程运行在S模式下,应用程序很容易破坏没有任何保护的执行环境.预编译的bootloader,RustSBI是运行在更底层的M模式下的软件,是OS内核的执行环境.

执行环境的另一种功能是对上层软件的监控管理,当上层软件执行出现了一些异常或特殊情况会需要用到执行环境中他提供的功能,因此需要暂停 上层软件的执行,转而运行执行环境的代码.这个过程也往往伴随着CPU的**特权级切换**.即处理**异常**

执行环境调用ecall,有个更通俗的名字——**系统调用**

### RISC-V的特权指令

与特权级无关的一般的指令和通用寄存器 x0 ~ x31 在任何特权级都可以执行
而每个特权级都对应一些特殊指令和控制状态寄存器(CSR Control and Status Register)来控制该特权级的某些行为并描述其状态。
在RISC-V中会有两类属于高特权级S模式的特权指令：
- 本身属于高特权级的指令,如sret表示从S返回U
- 访问了S特权级下才能访问的寄存器或内存,如表示S模式下表示系统状态的CSR sstatus

## 应用程序

user目录下的结构如下
```
src/
├── bin
│   ├── 00hello_world.rs
│   ├── 01store_fault.rs
│   ├── 02power.rs
│   ├── 03priv_inst.rs
│   └── 04priv_csr.rs
├── console.rs
├── lang_items.rs
├── lib.rs
├── linker.ld
└── syscall.rs
```
lib.rs作为库文件,console,lang_items,syscall作为其子模块.
bin目录下存放的是应用程序
```
#[macro_use]
extern crate user_lib;
```
他们引入了外部库user_lib,即lib.rs的别名(在Cargo.toml中对其名字进行了设置)
lib.rs将作为bin目录下源程序所依赖的用户库,可以类比我们提供了标准库
```
#[no_mangle]
#[link_section = ".text.entry"]
pub extern "C" fn _start() -> ! {
    clear_bss();
    exit(main());
    panic!("unreachable after sys_exit!");
}
```
我们在此处定义用户库的入口点_start
```
#[link_section = ".text.entry"]
```
意味着_start这段代码编译后的汇编代码将放入一个名为.text.entry的代码段中,以便后续链接时我们调整他的位置使其作为用户库的入口
接下来和第一章一样手动清零.bss段,然后调用main函数得到一个i32的返回值,调用用户库提供的exit接口退出应用程序,并将main函数的返回值告知批处理系统. exit由syscall.rs子模块提供实现 此处的main所调用的是应用程序中的main

在lib.rs中还有一个main:
```
#[linkage = "weak"]
#[no_mangle]
fn main() -> i32 {
    panic!("Cannot find main!");
}
```
可以看到我们将其函数符号标志为弱符号,这样链接时链接器会使用bin目录下的应用主逻辑作为main.即便bin目录下的应用程序找不到任何main,那么此处的main也可以保证链接后编译能够通过

### 内存布局

我们依然需要自定义链接脚本.
- 程序起始地址设置为0x80400000
- _start所在的.text.entry放在整个程序的开头,即跳转到0x80400000后就进入用户库入口点进行初始化,然后跳转到应用程序的主逻辑
- 提供.bss段的起始和终止地址,方便clear_bss使用
```
OUTPUT_ARCH(riscv)
ENTRY(_start)       //入口点设为全局符号_start,在lib.rs中所定义

BASE_ADDRESS = 0x80400000;

SECTIONS
{
    . = BASE_ADDRESS;
    .text : {
        *(.text.entry)      //保证_start位于最开头
        *(.text .text.*)
    }
    .rodata : {
        *(.rodata .rodata.*)
        *(.srodata .srodata.*)
    }
    .data : {
        *(.data .data.*)
        *(.sdata .sdata.*)
    }
    .bss : {
        start_bss = .;
        *(.bss .bss.*)
        *(.sbss .sbss.*)
        end_bss = .;
    }
    /DISCARD/ : {
        *(.eh_frame)
        *(.debug*)
    }
}
```

### 系统调用syscall

子模块syscall中,应用程序 通过ecall调用批处理系统提供的接口,trap陷入S模式执行服务代码
现在我们不关心底层的批处理系统如何提供应用程序所需的功能
本章中,应用程序和批处理系统之间约定以下两个系统调用:
```
/// 功能：将内存中缓冲区中的数据写入文件。
/// 参数：`fd` 表示待写入文件的文件描述符；
///      `buf` 表示内存中缓冲区的起始地址；
///      `len` 表示内存中缓冲区的长度。
/// 返回值：返回成功写入的长度。
/// syscall ID：64
fn sys_write(fd: usize, buf: *const u8, len: usize) -> isize;

/// 功能：退出应用程序并将返回值告知批处理系统。
/// 参数：`exit_code` 表示应用程序的返回值。
/// 返回值：该系统调用不应该返回。
/// syscall ID：93
fn sys_exit(exit_code: usize) -> !;
```

系统调用是汇编指令级的二进制接口,在实际调用时我们要按照RISC-V调用规范,在合适的寄存器中放置系统调用的参数,再执行ecall来触发trap
在trap回到U模式后,会从ecall的下一条指令继续执行,同时我们能够按照调用规范在合适的寄存器中读取返回值

>RISC-V 寄存器编号和别名
RISC-V 寄存器编号从 0~31 ，表示为 x0~x31 。 其中：
x10~x17 : 对应 a0~a7
x1 ：对应 ra

我们约定寄存器a0~a6保存系统调用的参数,a0保存系统调用的返回值.
有些许不同的是a7用来传递syscall ID
由于这些内容超出了Rust的表达能力,所以我们需要内嵌汇编代码来完成寄存器级别的操作
```
fn syscall(id: usize, args: [usize; 3]) -> isize {
    let mut ret: isize;
    unsafe {
        asm!(
            "ecall",
            inlateout("x10") args[0] => ret,    //args[0]为返回值存入a0
            in("x11") args[1],                  //args[1]为其它参数存入a1
            in("x12") args[2],                  //args[2]为其它参数存入a2
            in("x17") id                        //systemid存入a7
        );
    }
    ret
}
```
至此,所有的系统调用都被我们封装成了syscall,只要传入syscall ID和恰当的3个参数
所以sys_write与sys_exit的实现只需要将syscall进行包装:
```
const SYSCALL_WRITE: usize = 64;
const SYSCALL_EXIT: usize = 93;

pub fn sys_write(fd: usize, buffer: &[u8]) -> isize {
    syscall(SYSCALL_WRITE, [fd, buffer.as_ptr() as usize, buffer.len()])
}

pub fn sys_exit(xstate: i32) -> isize {
    syscall(SYSCALL_EXIT, [xstate as usize, 0, 0])
}
```
这两个系统调用在用户库lib.rs中进行进一步封装:
```
use syscall::*;

pub fn write(fd: usize, buf: &[u8]) -> isize {
    sys_write(fd, buf)
}
pub fn exit(exit_code: i32) -> isize {
    sys_exit(exit_code)
}
```
在console子模块中,我们把Stdout::write_str改为基于write的实现
```
// user/src/console.rs

use super::write;

const STDOUT: usize = 1;

impl Write for Stdout {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        write(STDOUT, s.as_bytes());
        Ok(())
    }
}
```

### 编译并执行

在build后,使用objcopy将生成的ELF文件删除所有的ELF header和符号得到.bin后缀的纯二进制镜像文件.
他们将被链接进内核并在合适的时机加载到内存

至此我们仅仅实现了应用程序,还没有实现批处理操作系统.
我们需要使用qemu-riscv64来模拟

这是应用程序03,他将在用户态下执行特权指令sret
```
// user/src/bin/03priv_inst.rs
use core::arch::asm;
#[no_mangle]
fn main() -> i32 {
    println!("Try to execute privileged instruction in U Mode");
    println!("Kernel should kill this application!");
    unsafe {
        asm!("sret");
    }
    0
}
```
这将会造成出错Illegal instruction
```
$ qemu-riscv64 ./03priv_inst
Try to execute privileged instruction in U Mode
Kernel should kill this application!
Illegal instruction (core dumped)
```
这说明riscv64得到特权级机制确实在起作用

这是00号程序的内容,输出Hello,world,它应当能正确地执行
```

#[no_mangle]
fn main() -> i32 {
    println!("Hello, world!");
    0
}
```
```
$ qemu-riscv64 ./00hello_world
Hello, world!
```
至此,用户库和应用程序都开发完成了.
尽管我们还没有实现OS,但通过遵循RISCV规范使用ecall,我们仍能通过qemu模拟来正常地运行他们

## 批处理OS的实现

在 os/src/main.rs 中能够找到这样一行：
```
global_asm!(include_str!("link_app.S"));
```
此处嵌入了一端汇编代码link_app.S
```
# os/src/link_app.S

    .align 3
    .section .data
    .global _num_app
_num_app:
    .quad 5
    .quad app_0_start
    .quad app_1_start
    .quad app_2_start
    .quad app_3_start
    .quad app_4_start
    .quad app_4_end

    .section .data
    .global app_0_start
    .global app_0_end
app_0_start:
    .incbin "../user/target/riscv64gc-unknown-none-elf/release/00hello_world.bin"
app_0_end:

    .section .data
    .global app_1_start
    .global app_1_end
app_1_start:
    .incbin "../user/target/riscv64gc-unknown-none-elf/release/01store_fault.bin"
app_1_end:

    .section .data
    .global app_2_start
    .global app_2_end
app_2_start:
    .incbin "../user/target/riscv64gc-unknown-none-elf/release/02power.bin"
app_2_end:

    .section .data
    .global app_3_start
    .global app_3_end
app_3_start:
    .incbin "../user/target/riscv64gc-unknown-none-elf/release/03priv_inst.bin"
app_3_end:

    .section .data
    .global app_4_start
    .global app_4_end
app_4_start:
    .incbin "../user/target/riscv64gc-unknown-none-elf/release/04priv_csr.bin"
app_4_end:
```
可以看到5个数据段分别插入了5个应用程序的二进制镜像,并且各有一对全局符号来描述开始与结束为止
符号_num_app相当于一个数组,第一个元素表示应用程序的数量,后面则按照顺序放置各个应用程序的起始地址
最后一个元素放置最后一个应用程序的结束位置,这样每个应用程序的位置都能从数组中相邻两个元素中得知
这个文件是在构建时由脚本os/build.rs控制生成的 此处不深入

### 找到并加载应用程序二进制码

应用管理器AppManager是邓氏鱼OS的核心组件,在batch子模块中实现
它的主要功能是
- 保存应用和各自的位置信息,以及当前执行到第几个应用
- 根据应用程序信息初始化应用所需的内存空间并加载执行

```
struct AppManager {
    num_app: usize,
    current_app: usize,
    app_start: [usize; MAX_APP_NUM + 1],    //各应用的起始地址
}
```
此处的内容涉及到大量Rust相关细节,我们在此不深入关注Rust的使用,而把精力放在OS逻辑上.
```
lazy_static! {
    static ref APP_MANAGER: UPSafeCell<AppManager> = unsafe { UPSafeCell::new({
        extern "C" { fn _num_app(); }
        let num_app_ptr = _num_app as usize as *const usize;
        let num_app = num_app_ptr.read_volatile();
        let mut app_start: [usize; MAX_APP_NUM + 1] = [0; MAX_APP_NUM + 1];
        let app_start_raw: &[usize] = core::slice::from_raw_parts(
            num_app_ptr.add(1), num_app + 1
        );
        app_start[..=num_app].copy_from_slice(app_start_raw);
        AppManager {
            num_app,
            current_app: 0,
            app_start,
        }
    })};
}
```
初始化的逻辑很简单 extern"C"找到link_app.S中提供的符号 _num_app,并从这里开始解析出应用数量以及各个应用的起始地址。这里我们使用了外部库 lazy_static 提供的 lazy_static! 宏

>lazy_static! 宏提供了全局变量的运行时初始化功能。一般情况下，全局变量必须在编译期设置一个初始值，但是有些全局变量依赖于运行期间才能得到的数据作为初始值。这导致这些全局变量需要在运行时发生变化，即需要重新设置初始值之后才能使用。如果我们手动实现的话有诸多不便之处，比如需要把这种全局变量声明为 static mut 并衍生出很多 unsafe 代码 。这种情况下我们可以使用 lazy_static! 宏来帮助我们解决这个问题。

AppManager的几个方法都很简单
```
pub fn print_app_info(&self) {
        println!("[kernel] num_app = {}", self.num_app);
        for i in 0..self.num_app {
            println!(
                "[kernel] app_{} [{:#x}, {:#x})",
                i,
                self.app_start[i],
                self.app_start[i + 1]
            );
        }
    }

pub fn get_current_app(&self) -> usize {
        self.current_app
    }

pub fn move_to_next_app(&mut self) {
    self.current_app += 1;
}
```

比较复杂的是load_app方法
```
unsafe fn load_app(&self, app_id: usize) {
    if app_id >= self.num_app {
        panic!("All applications completed!");
    }
    println!("[kernel] Loading app_{}", app_id);

    // clear app area
    core::slice::from_raw_parts_mut(
        APP_BASE_ADDRESS as *mut u8,
        APP_SIZE_LIMIT
    ).fill(0);

    let app_src = core::slice::from_raw_parts(
        self.app_start[app_id] as *const u8,
        self.app_start[app_id + 1] - self.app_start[app_id]
    );
    let app_dst = core::slice::from_raw_parts_mut(
        APP_BASE_ADDRESS as *mut u8,
        app_src.len()
    );
    app_dst.copy_from_slice(app_src);
    // memory fence about fetching the instruction memory
    asm!("fence.i");
}
```
这个方法负责将参数app_id对应的应用程序的二进制镜像加载到内存0x80400000,也就是我们上一节中约定好的地址
我们首先将一块内存清空，然后找到待加载应用二进制镜像的位置，并将它复制到正确的位置。
它本质上是把数据从一块内存复制到另一块内存,从批处理OS的角度来看，是将操作系统数据段的一部分数据(应用程序)复制到了一个可以执行代码的内存区域。

一般而言CPU认为程序的代码段不会发生变化,i-cache是只读缓存
而在我们的批处理OS下程序的代码段发生了变化,i-cache指令缓存中可能会有与内存不一致的内容
插入的汇编指令fence.i的作用是取指屏蔽,保证CPU访问的应用代码是最新的而不是i-cache中过时的内容

batch子模块对外暴露出如下接口:
init:调用print_app_info时第一次使用全局变量APP_MANAGER
run_next_app:加载运行下一个应用程序