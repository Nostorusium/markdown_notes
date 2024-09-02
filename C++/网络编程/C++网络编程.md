# C++网络编程

## 重识一些概念

- 局域网
  局域网并不意味着范围小,只要你想,几个跨洲际的国家也能组成局域网.
  局域网所强调的是区域内的各类计算机与设备所组成的私有网络.
- 广域网
  广域网才是印象中的所谓"外网"或"公网",是链接不同地区局域网/城域网计算机的远程公共网络.
  路由器所组成了一个小的局域网,路由器上连着的网线则真正连接到了外网.
- IP
  IP只是一组数,用来标识一个计算机.
  在局域网内部你通常会见到192.168.x.x的私有地址.
  如果路由器连着网线或者PC主机上插着网线,那么你应当能够连接到外网.
  即便没插网线,在这个小的局域网内部你可以仍能ping通内部的私有ip.
- 端口
  端口用来标识一个进程,并不是所有进程都需要端口,如果一个进程压根不涉及网络功能他也没必要申请一个端口.
  由于端口需要唯一标识一个进程,所以端口不能复用.

### 重识分层模型

- TCP/IP四层模型
  ISO的上三层被合并成TCP/IP的应用层
  传输层仍是传输层
  网络层则变成了TCP/IP的网络互联层
  数据链路层和物理层则合并成了TCP/IP的网络接口层

在网络通信时,程序员只负责应用层数据处理.
在应用层的数据本身也可以使用一些协议封装(可标准也可不标准)后再下放,也可以不封装.
下层的操作一些都通过socket这个API完成,socket的背后是操作系统.

### socket接口

socket的单词含义是插座
socket由加利福尼亚大学伯克利分校的研究所研发,其目的是将TCP/IP协议的相关软件移植到UNIX系统.
开发者开发的接口让应用程序能简单地调用它来进行通信,该接口的不断完善形成了socket套接字.
Linux系统里采用了Socket套接字,并被广泛使用,现在已成为事实上的标准.

### 字节序

在源向目的发送数据后,数据存储在内存的某个位置,此时大小端就会对这个数据做出不同的解释.
因此我们还要定义字节序,定义好是大端还是小端.

- 小端: 低对低,高对高
- 大端: 低对高,高对低

C/C++提供了大小端转化函数

```
uint16_t htons(uint16_t hostshort);
uint32_t htonl(uint32_t hostlong);
uint16_t ntohs(uint16_t netshort);
uint32_t ntohl(uint32_t netshort);
```

其中h表示host主机 n表示net网络
s和l表示short和long(16位或32位)
host to net即从主机字节序转化为网络字节序。

## TCP协议

TCP的特点：面向连接全双工，安全，流式传输。
流式传输的特点是，发送端和接收端处理数据的速度可以不一致。
一旦连接建立，两者之间就好比存在一个管道，如果管道内部满了，那么发送端会等一会儿再继续发送。

三次握手：
1. SYN 客户端想要和服务器建立连接。
2. SYN+ACK 服务器响应，表示同意。
3. ACK 客户端知道服务器同意，做好准备，即将传输数据。
即：我要传输 -> 允许传输 -> 即将开始 的顺序。

所谓四次挥手,其实是各自挥两次,谁先发起都无所谓.
一次挥手关闭一侧的连接,两侧完全关闭.
即：我要走了 -> 行，关闭一侧，我这边还有剩余数据 -> 数据交接完毕，准备关闭 -> 完成数据交接，关闭连接。

挥手不管是谁先发起都无所谓。在HTTP这样的C-S模式的应用层架构，可能会指定具体的交互顺序（比如客户端先发起），但这对于传输层的TCP协议本身这是无所谓的。

### TCP-socket通信流程

```
接收端：
socket()->bind()->listen()->accept()->recv()->send()->recv()->close()

发送端：
socket()->connect()->send()->recv()->close()
```

对于担任服务器的一端，经过bind，listen和accept后，就进入了阻塞，开始等待连接到来。
此时发送端发起connect后，服务端接受连接，TCP连接就正式建立。

### 通信流程

服务端的通信流程
```
int lfd = socket(); // l means listen
bind();
listen();
int cfd = accept(); // c means connection

read();  //recv();
wrtie(); // send();

close();
```

1. lfd = socket()
   socket()会返回一个文件描述符。这是因为socket起源于unix,所以socket和文件io的操作类似。
   linux系统也继承了这一点：打开文件->读写文件->关闭文件。所以网络通信也叫做网络I/O。
   此处获得的fd用于监听。用于通信的fd另有其人。
2. bind
   将监听得到的fd和本地的端口进行绑定。端口对应了一个进程，所以绑定socket和具体的进程。
3. listen
   设置监听，监听的是客户端的连接。
4. accept()
   等待并接受客户端的连接请求。建立新的连接，得到一个新的文件描述符。没有就阻塞。
   在服务端fd分为两类，分别用于监听和通信。此描述符用于通信。
   客户端和服务端的通信是基于这个通信fd的。
5. 读写数据
6. close();
   断开连接，关闭套接字。这里挥手两次。客户端会再挥手两次，然后彻底关闭连接。
   谁先调用close,谁先进行四次挥手的前两次。

客户端的通信流程
```
int cfd = socket(); // 用于通信
connect();          // 客户端要主动的连接服务器，则需要知道ip和端口号。

read();
write();

close();
```

客户端的通信比较简单。
1. 获得用于通信的fd
2. 连接
3. 读写
4. 关闭

### 通信函数


1. socket
```
int socket(int domain, int type, int protocol);

socket() creates an endpoint for communication and returns a file descriptor that refers to that endpoint.
The domain argument specifies a communication domain; this selects the protocol family which will be used
  for communication.  These families are defined in <sys/socket.h>.
On success, a file descriptor for the new socket is returned.  On error, -1 is returned, 
  and errno is set appropriately.
```

2. bind

```
int bind(int sockfd, const struct sockaddr *addr,socklen_t addrlen);

When a socket is created with socket(2), it exists in a name space (address family) but has no ad‐
  dress assigned to it.  bind() assigns the address specified by addr to the socket referred  to  by
  the  file  descriptor  sockfd.   addrlen  specifies  the  size, in bytes, of the address structure
  pointed to by addr.  Traditionally, this operation is called “assigning a name to a socket”.
```

要注意到此处的 *addr 只接受大端。第三个为 *addr所指内存的大小(sizeof)
sockaddr结构体如下：

```
// ipv4
struct sockaddr{
  sa_family_t sa_familty; //地址族协议,ipv4
  char sa_data[14]; //端口2字节 + IP地址4字节 + 填充8字节
};

struct in_addr{
  in_addr_t s_addr;  // typedef uint32_t类型 一个数
}

struct sockaddr_in{
  sa_family_t sin_family; //地址族协议
  in_port_t sin_port;     // 端口 2字节 大端
  srtuct in_addr sin_addr;  // IP地址 4字节 大端
  // 填充 保证和sockaddr内存大小一致
  unsigned char sin_zero[sizeof(struct socketaddr)-sizeof(sin_family)-sizeof(inport_t)-sizeof(struct in_addr)];
};
```

这sockaddr和sockaddr_in结构体内存大小完全相同。
由于sockaddr写入端口ip不方便，通常使用sockaddr_in进行初始化，然后强转成sockaddr。

3. listen

```
int listen(int sockfd, int backlog);

listen() marks  the  socket  referred to by sockfd as a passive socket, that is, as a socket that
  will be used to accept incoming connection requests using accept(2).

The  backlog  argument  defines  the  maximum length to which the queue of pending connections for
  sockfd may grow.  If a connection request arrives when the queue is full, the client  may  receive
  an  error  with  an indication of ECONNREFUSED or, if the underlying protocol supports retransmis‐
  sion, the request may be ignored so that a later reattempt at connection succeeds.
```

backlog所表示的队列最大值是128。该值表示一次性能接受多少个连接请求。

4. accept

```
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);

The accept() system call is used with connection-based socket types (SOCK_STREAM, SOCK_SEQPACKET).
  It extracts the first connection request on the queue of pending  connections  for  the  listening
  socket,  sockfd,  creates  a  new connected socket, and returns a new file descriptor referring to
  that socket.  The newly created socket is not in the listening state.  The original socket  sockfd
  is unaffected by this call.
The addrlen argument is a value-result argument: the caller must initialize it to contain the size
  (in  bytes)  of the structure pointed to by addr; on return it will contain the actual size of the
  peer address.
```

accpet从挂起连接队列中取出头一个，创建一个新的连接了的socket并返回他的fd(通信fd)。
新创建的socket不在监听状态。原来的socket不受影响。参数addr和addrlen可以指定为空。
该函数是阻塞函数。每调用一次只能和一个客户端进行连接。如果需要和多个客户端连接，写个循环。

1. connect

connect为客户端发起连接请求。

```
int connect(int sockfd, const struct sockaddr *addr,socklen_t addrlen);

The connect() system call connects the socket referred to by the file descriptor sockfd to the ad‐
  dress specified by addr.  The addrlen argument specifies the size of addr.

If the socket sockfd is of type SOCK_DGRAM, then addr is the address to which datagrams  are  sent
  by  default,  and  the  only  address from which datagrams are received.  If the socket is of type
  SOCK_STREAM or SOCK_SEQPACKET, this call attempts to make a connection to the socket that is bound
  to the address specified by addr.

If  the  connection or binding succeeds, zero is returned.  On error, -1 is returned, and errno is
  set appropriately.
```

要注意客户端的端口是随机分配的，因为服务器不会主动向客户端连接，也无需知道客户端进程的端口。
但是你也可以在创建socket的时候指定一个固定的。

6. 读写

接收数据
```
ssize_t read(int fd, void *buf, size_t count);
ssize_t recv(int sockfd, void *buf, size_t len, int flags);

read() attempts to read up to count bytes from file descriptor fd into the buffer starting at buf.

On success, the number of bytes read is returned (zero indicates end of file)
On error, -1 is returned,
```

recv比read多一个参数，flags用于接受一些属性。一般不使用，指定为0。
前三个函数含义完全一样。如果返回值为0，意味着连接可能被断开了。如果没有断开，接收数据的函数会阻塞等待数据到达。


发送数据
```
ssize_t send(int fd, const void *buf, size_t len);
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
```

buf为所传入的数据，len指定传入多少字节。(比如一个1919810大小的buf,而只传输前114514有效数据。)
返回值：实际发送的字节数，和参数len相等。若为-1表示失败。

7. 关闭套接字

```
int close(int fd);
close() closes a file descriptor, so that it no longer refers to any file and may be reused.
close() returns zero on success.  On error, -1 is returned, and errno is set appropriately.
```

至此你可以发现TCP通信中使用的函数和文件操作没有什么太大的差别。对于一个socket，我们就是在当做IO操作，视其为一个fd。

### Windows下的socket

由于都遵循同一套协议，原理没有任何区别，只是一些参数，接口不一样。

>【并发网络通信-套接字通信(C/C++ 多线程)】 https://www.bilibili.com/video/BV1F64y1U7A2/?p=15&share_source=copy_web&vd_source=47d5771b3b6d76d52e2289338a07af52


### 文件描述符

服务器通常会持有监听和通信两类fd，而客户端只持有通信fd。
这是因为客户端没有监听的需求。服务器通常持有1个fd专门用于监听，若干fd用于通信。
在服务端，前三步socket() -> bind() -> listen() 都是针对这个监听fd而言的。
直到accept()返回一个通信fd.

每一个socket fd在内核里都对应两块内存：读缓冲区和写缓冲区。
客户端向服务端发起连接请求，客户端cfd的写缓冲区向服务端lfd的读缓冲区写入请求。
此时这个lfd的读缓冲区作为一个连接队列，等待accept从中取出一个连接。
取出一个连接后，得到一个用于通信的新fd，一切通信基于这个新的cfd进行，而不是旧的lfd。

此时，客户端和服务端双方进行交互的socket都是cfd。当一方使用write/send，首先数据会存储在写缓冲区。
当内核检测到写缓冲区有数据，那么就把写缓冲区的数据发送到对侧的读缓冲区。
当对侧调用read/recv，则把数据从读缓冲区拿出。
这个缓冲区就好比是你家门口的邮箱，送信都要通过这个邮箱。发送数据要先把数据放邮箱里，传送过去后，接受数据要从邮箱里往外拿。

accept是阻塞的，它所检测的是监听lfd的连接队列(读缓冲区)。如果没数据则阻塞，直到新的连接进来。