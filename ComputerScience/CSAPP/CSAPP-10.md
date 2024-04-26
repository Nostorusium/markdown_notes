# CSAPP章节小结

## 第十章 系统级I/O

考小题 1~2道

- 接口，端口，驱动程序
- 三种文件IO访问与缓冲区
- 文件元数据,共享,重定向(大概率出)

记忆性

22年:
能在Linux使用的IO方式为: Unix IO RobustIO StandardIO都可以
可以用函数 dup2 通过复制文件描述符指针和打开文件表结点引用计数的方式实现输出重定向

### Unix I/O

一个Linux文件,就是一串字节的序列
所有的I/O设备都被模型化为文件,所有的输入输出都被当做对文件的读与写
这种优雅地把设备映射为文件的方式,允许Linux内核引导一个简单,低级的应用接口

```
//成功,返回文件描述符数字(总是当前进程中没有打开的最小描述符) 不然返回-1
//Linux内核创建的每个进程都与一个终端相关联的三个打开的文件开始
//0:stdin 1:stdout 2:stderr
int open(char * filename,int flags,mode_t mode);

//关闭一个已经关闭的文件会造成线程程序灾难
int close(int fd);

//从当前文件位置复制字节到内存为止 然后更新文件位置
//ssize_t是有符号整数
ssize_t read(int fd,void *buf,size_t n);

//从内存复制字节到文件位置
ssize_t wrtie(int fd,const void * buf,size_t n);
```

实例:读一个写一个 用标准输入输出
```
int main(void){
    char c;
    while(Read(STDIN_FILENO,&c,1)!=0)
        write(STDOUT_FILENO,&c,1);
    exit(0);
}
```

### 文件

- 普通文件:包含任意数据 对于内核而言,是文本文件还是二进制文件并无区别
- 目录:包含一组链接,将一个文件名映射到一个文件 .表示文件自身的链接, ..表示父目录的链接
- 套接字(socket)

>Unix内核如何表示打开文件(简答)
每个进程都有一张文件描述符表,
fd是其编号或者索引
1~3为stdin,stdout,stderr
fd4指向打开磁盘文件,fd1指向终端
**描述符表项**指向一个其对应的**文件表的表项**,描述了文件位置等
文件表表项又指向**v-node表**描述了文件的访问,大小,类型等
其中文件表与v-node表被所有进程共享

>共享
不同的描述符指向文件表中的不同表项,但依然可以指向同一个文件(两个文件表表项共享同一个文件)

子进程继承了父进程的打开文件,其描述符表相同,
描述符表所指向的文件表表项与其所指v-node也相同
所以父进程子进程共享打开的文件



### I/O重定向

```
//复制oldfd描述符表项到newfd 并且覆盖
dup2(oldfd,newfd);
```

```
//考虑下方的描述符表
fd0 :-
fd1 :a
fd2 :-
fd3 :-
fd4 :b

//dup(4,2)后:
fd0 :-
fd1 :b
fd2 :-
fd3 :-
fd4 :b
```

### RIO

Robust I/O,是一个封装体,提供了方便健壮高效的I/O

### Standard I/O

C定义了标准I/O库(libc.so),提供了Unix I/O的高级替代
如fopen,fclose fread fwrite fgets fputs fscanf fprintf

Standard I/O Streams 
将一个打开的文件抽象为**流**

### 三种I/O的关系

C中封装了标准I/O与RIO函数
而Unix I/O函数使用系统调用来访问
标准I/O和RIO是基于UnixI/O实现的

函数选择(判断题)
- 标准I/O 使用磁盘文件或终端文件时,大多数C程序员在整个职业生涯中只使用标准I/O
- UnixI/O 信号处理程序(Unix I/O异步信号安全的),和极少数追求性能的情况
- RIO 准备读写网络套接字时 避免在套接字上使用标准I/O