当访问到逻辑上分配的内存时,并没有为其分配实际的物理页面,其页表也标记为未分配。
因此发生发生缺页中断，执行xv6的trap.
完善了缺页处理程序


fork出的子进程是父进程的副本 本来应该复制父进程的内存到子进程
父子进程共享只读页面
PTE中保留位定义一个标志为 COW
fork时禁用PTE_W 标记为COW
因此当有进程试图写，触发异常
此时复制一页共享的物理页过去，并允许写权限
此时子进程就真正独立出去了 也允许对他单独写了
此时也要同时清除他的COW标记

当懒复制页的引用为1 即表示只有他自己在用
相当于原地TP 把COW标记下了再把可写划回来即可

新分配的物理内存只对子进程可见 所以此时才把PTE设置为可写。

需要引用计数 一个物理页可能被多个进程引用 只有当以后一个引用消失后才回收该物理页