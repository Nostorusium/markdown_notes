# XV6速通

以简要的语言描述几个Lab的内容,准备一下细节
然后包装一下

## 项目介绍

本项目基于一个RISC-V架构的操作系统内核,在原有xv6的基础上进行一些功能性增添与实现
1. 增添了一些**系统调用**用于调试,系统调用追踪trace,函数调用追踪backtrace
   此处应该说明他们的实现,以及如何实现一个**通用的系统调用**,即描述xv6内核架构里如何添加系统调用
2. 为一些系统调用的中断保存上下文并在返回时恢复
   此处应说明用户态进入内核态trap后的过程,以及寄存器上的一些细节
   要说明RISC-V架构下这些工作需要系统程序实现
   以及trap后寄存器多的变化,是如何实现的
3. 应该能描述出xv6内核的基本架构,回头画个图理一理
   比如进程里包含了哪些结构:trapframe之类的
   

## Lab2-系统调用

添加一个系统调用,实现trace()

- trace mask,其中mask是掩码,其值应当为(1 << 系统调用号)
- 如果掩码中设置了某个系统调用的编号,则在每个系统调用即将返回的时候打印出一行
- 行应该包括进程id,系统调用名称与返回值

- 系统调用原型放入user/user.h
- 存根放入user/usys.pl  它会展开一段汇编代码使用RISC-V指令ecall切换到内核态
- 系统调用号 kernel/syscall.h
- kernel/sysproc.c,在proc结构中添加新的变量
- 修改fork()的实现,让追踪的掩码从父进程复制到子进程
- 修改syscall.c的syscall()函数来打印跟踪输出

### 过程

1. kernel/syscall.h
   - 给这个系统调用分配一个编号 #define SYS_trace 22
2. user/usys.pl
   - 在user/usys.pl加入entry("trace")
   - 它的作用是展开一段汇编,把系统调用号存入寄存器a7,并ecall进入内核
   - ecall进入内核后会跳转到kernel/syscall.c中的syscall,执行系统调用
   - num = p->trapframe->a7 读出a7中的系统调用号num
   - p->trapframe->a0 = syscalls[num]() 把返回值保存在a0寄存器
   - syscalls[num]()函数中的第num就是对应的系统调用命令
3. 把trace系统调用加入函数指针数组*syscalls[]里
   - [SYS_trace] sys_trace,
   - 此处的sys_trace还没有具体实现,但先extern uint64 sys_trace(void)声明一下
4. kernel/proc.h
   - 在proc结构体中加入变量mask,这样每个进程都有了自己的mask,他是我们要追踪的系统调用
5. kernel/sysproc.c
   - 在这里给出sys_trace的具体实现
   - 取出a0寄存器的返回值给int mask
   - myproc()->mask = mask; 并把这个值赋给当前进程的mask
   - RISCV的规范是返回值放到a0
6. kernel/syscall.c
   - 我们可以在syscall_names[]中给SYS_TRACE定义一个名
   - 修改通用系统调用syscall(void)
   - 如果1<<num & p->mask为真,意味着我们追踪的调用和当前进程的mask一样则输出
7. kernel/proc.c
   修改fork(),令np->mask = p->mask,子进程继承父进程的mask

### 总结

用户侧
1. atoi把argv[1] 即给定的mask转化为整数,作为参数传入trace
2. 阅读汇编我们发现它根据SYS_NAME得到当前调用的编号
3. 汇编代码将这一编号传入a7,执行ecall,控制转移

业务逻辑
1. 读入一个mask掩码,由它确认跟踪的调用
2. 每个进程要维持一个表示追踪的mask,它由trace 后紧跟的参数给出
3. 每当一个调用号传入sys_call,就拿出当前进程的mask与它比对
4. 如果比对成功,表示这是需要追踪的调用

通用sys_call方法
1. 得到当前进程a7内容,即所传递的调用号
2. 执行相应的函数与判断是否成功
3. 若成功,返回值存入a0

具体实现
1. 修改proc的结构,加入mask,并让子进程复制父进程的mask
2. 定义sys_trace函数,argint函数把参数拿出来交给当前进程的mask
3. 为了通用性,修改sys_call通用函数,在这里做判断决定当前调用是否是追踪的调用

## Lab3-页表

### Lab3.1

给定一个page table,递归的打印PTE和子页表

```
void vmprint(pagetable_t pagetable,int depth){
  for(int i=0;i<512;i++){
    pte_t pte = pagetable[i];

    //表示当前PTE有效
    if((pte & PTE_V)){

      for(int j=1;j<=depth-1;j++){
        printf(".. ");
      }
      // <index>:pte <pte address> pa <pte physical address>
      printf("..%d:pte %p pa %p\n",i,pte,PTE2PA(pte));

      //表示当前PTE指向子级页表
      if((pte & (PTE_R|PTE_W|PTE_X))==0){
        uint64 child = PTE2PA(pte);
        vmprint((pagetable_t)child,depth+1);
      }
    }
  }
}
```

并在exec.c返回argc之前调用;
```
  if(p->pid == 1){
    vmprint(p->pagetable,1);
  }
```
打印pid为1的进程的页表

### Lab3.2

## Lab4-Trap

### Lab4.2 backtrace

实现回溯,反向跟踪函数的调用列表
为了实现它,编译器需要生成机器码,让调用链上的每一个函数在对应的栈上维护一个栈帧;
栈帧应由返回地址和一个指向调用方栈帧的指针组成
寄存器s0包含一个指向当前栈帧的指针(实际上指向保存的返回地址+8)
我们希望使用这些帧指针来遍历栈,并在每个栈帧中打印保存的返回地址

```
--------------
局部变量        <-- fp
--------------
返回地址        <-- fp-8  
--------------
前一个fp的地址   <-- fp-16
--------------
保存的寄存器     
--------------
局部变量         <-- sp
--------------
```

1. 先从s0得到fp指针 那么就可以得到fp-8与fp-16从而获得返回地址与上一fp地址
2. 不断回溯并打印返回地址ra,直到发现fp和pre_fp在同一页,已回溯完
3. 打印出最后一个ra,退出

### Lab4.3 Alarm

实现两个系统调用 sigalarm(int,void(*)()) sigreturn()
这两个系统调用实现alarm,检测用户程序使用CPU的时长并发出提醒
sigreturn用于alarm处理函数执行后恢复中断前的状态
sigalarm的第一个参数用来设置时钟中断个数 当到达该个数时调用指定函数;第二个参数设置了alarm的处理函数

检测时间以时钟中断为单位,统计用户进程进行了几次中断,就可以获知用户进程到达了警报的CPU时长从而给出提醒
比如如果时钟中断为50ms 那就每50ms进行一次中断
时钟中断主要用于计时

#### test0

过程:
1. 给proc新增相关字段
2. 调用alarm系统调用后,先去设置proc与alarm相关的字段
3. 修改trap,使时钟中断时tick自增,并在触发alarm时修改epc为处理函数入口


细节如下
1. 在proc.h进程结构体中添加字段:interval ticks handler
   表示触发alarm需要的中断数量,统计时间中断的次数和alarm处理函数的地址
2. sysproc实现调用函数
   ```
   uint64 sys_sigalarm(void){
   int interval;
   uint64 f_addr;
   struct proc *p = myproc();

   argint(0,&interval);
   argaddr(1,&f_addr);

   p->interval = interval;  //为当前进程p初始化字段
   p->handler = f_addr;
   p->ticks = 0;  

   return 0;
   }
   uint64 sys_sigreturn(void){
    return 0;
   }
   ```
3. kernel/trap.c usertrap中的时间中断处理函数
   ```
   void usertrap(){
    ...
    if(which_dev == 2){
      p->ticks++;
      if(p->ticks == p->interval && p->interval > 0){
        p->trapframe->ecp = p->handler; //当到达指定数量,就将epc寄存器值改为处理函数地址
      }else{
        yield();
      }
    }
   }
   ```

#### test1/2/3

test0中的epc被覆写了,原先的ecp没有保存
我们应当在中断时保存恰当的寄存器值
在proc中加入pre_trapframe字段来保存,并给他分配物理页
修改usertrap,在触发中断时把ra,sp,s0,epc寄存器的值保存,
在alarm处理函数后进入sigreturn,它要恢复保存的寄存器

但这样的写法有很多弊端:
1. 代码并不简洁
2. 只保存了部分寄存器值

于是我们可以选择直接保存与恢复整个陷阱帧trapframe

### 总结

至此我们回顾了trap,异常的机制与保存上下文
第一个Lab中,对函数的追踪让我们看到调用顺序:
trap->syscall->sysproc
用户态先trap进入内核态,然后执行syscall和对应的点sysproc

第二个Lab,我们见到了进程p所维持的trapframe结构的作用:保存寄存器信息
并且我们走完了中断过程中保存状态信息的全流程

RISC_V CPU有一组控制寄存器,在执行trap时进行以下操作
1. 清除SIE(一个位)来禁用中断
2. 复制pc到sepc,当trap发生RISC-V把PC保存在这里
3. 将当前模式(用户态或特权态)保存在sstatus的SPP位
4. 在scause,这里应是一个数字,描述了trap的原因
5. 模式转换为特权态
6. stvec,在这里写下trap处理程序的地址,把他复制到PC并跳转
7. 从新的PC开始执行

在此期间CPU不会切换内核页表,也不保存除PC以外的任何寄存器
内核软件必须执行这些任务

现在我们回顾一下系统调用
用户代码将exec的参数放在a0与a1,系统调用号放在a7,这个号码与syscall中的相关结构项所匹配;
随后ecall进入内核,执行uservec,usertrap,再执行syscall.

syscall从trapframe的a7中获得调用号,查找相应函数,然后调用实现函数

## Lab5-lazy allocation

优化:虚拟的增长进程大小而不直接分配物理页
让缺页中断去分配

## Lab6-Copy on Write

在RISC-V中 PTE有一位RSW保留给用户使用,用来标记cow页