# Linux下的一些文件/内存操作

read,write不再赘述

## stat

```
int stat(const char *pathname, struct stat *statbuf);

struct stat{
   mode_t    st_mode;        /* 文件类型和权限 */
   off_t     st_size;        /* 文件大小，字节数*/
};
```

是syscall
stat用于取得指定文件的属性，并将其存储在指定的stat里。

## mmap

是syscall
mmap,内核内存管理中的常胜syscall:memory map 伟大无需多言

```
void* mmap(void* start,size_t length,int prot,int flags,int fd,off_t offset);
```

将一个文件或其他对象映射到内存。
start为起始地址,length表示映射多长。
prot为protect，表示期望的内存保护标志，如PROT_READ表示希望页内容可以被读取。
flags指定映射对象的类型，是否可以共享。MAP_PRIVATE建立一个写入时拷贝的私有映射，写入内存不影响原文件。
fd一般是open返回的文件描述符。
offset表示被映射内容的起点(偏移量)。

munmap与它做相反的操作，释放内存。

## iovec

```
struct iovec {
    void      *iov_base;      /* starting address of buffer */
    size_t    iov_len;        /* size of buffer */
};
```

io vector，定义了一个向量元素，通常作为一个多元素的数组。
使用例：

```
iovec iovector[2];

iovector[0].iov_base = ...
iovector[0].iov_len = ...
iovector[1].iov_base = ...

...

readv(fd,iovector,2);
```

## readv/writev

将文件中的数据读入缓冲数组/将缓冲数组里的数据写入文件。
有时也称聚集读/写


```
ssize_t readv(int filedes, const struct iovec *iov, int iovcnt);
ssize_t writev(int filedes, const struct iovec *iov, int iovcnt);
```

filedes表示文件描述符，iov指向一个iovec数组，iovcnt表示结构体个数。
若成功返回所读/缩写的字节数，出错返回-1.
readv/writev以顺序iov[0]，iov[1]...iov[iovcnt-1]的顺序聚集读写数据。

## fcntl

```
int fcntl(int fd, int cmd);
int fcntl(int fd, int cmd, long arg);         
int fcntl(int fd, int cmd, struct flock *lock);
```

file control，提供对当前文件(fd)的控制。
比如复制描述符，设置描述符标记(F_GETFD,F_SETFD)，设置文件状态标记(F_GETFL,F_SETFL)，设置异步IO所有权，设置锁。

cmd为具体控制命令。如F_DUPFD表示复制这个描述符，返回一个最小的不小于arg的可用描述符，都指向对某对象的引用。