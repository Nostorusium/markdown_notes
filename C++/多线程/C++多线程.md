# C++多线程

多线程的应用场景:
- 比较耗时的操作,将任务分解拆到不同的线程去
- 需要利用多核CPU处理数据的场景
- 数据流分解,读写分离的设计,解耦合的场景

## 线程基础

### HelloWorld

进程创建后,我们会拥有一个主线程,而子线程的创建需要在main中执行.

```
std::thread th1(func);
```

使用thread创建子进程,并传入一个函数指针作为子进程的内容.
可以把main视作主线程的入口函数,func视为子线程的入口函数.

```
// 命名空间std::this_thread下的get_id()方法得到当前线程的id
void threadFunc(){
    std::cout<<"[Sub Thread]thread main start!"<<std::endl;
    std::cout<<"[Sub Thread]sub thread id: "<<std::this_thread::get_id()<<std::endl;

    // sleep() 表示释放该线程的资源多长时间
    for(int i=0;i<10;i++){
        std::this_thread::sleep_for(std::chrono::seconds(1));
        std::cout<<"[Sub Thread]in thread "<<std::this_thread::get_id()<<std::endl;
    }

    std::cout<<"[Sub Thread]end sub thread "<<std::this_thread::get_id()<<std::endl;
}
```

join()可以阻塞正在运行的线程,让其等待调用join()的线程结束后再执行

```
int main(){
    // 启动子线程 子线程的内容由函数指针传入
    std::thread thread1(threadFunc);

    std::cout<<"[Main Thread]main thread id: "<<std::this_thread::get_id()<<std::endl;
    std::cout<<"[Main Thread]wait sub thread"<<std::endl;

    // join()等待子线程退出,阻塞了主线程
    // 如果主线程提前退出了 那么会出错,thread对象被销毁子线程还在运行
    // 为了防止主线程提前退出 可以使用join阻塞主线程,强制其等待子线程结束
    thread1.join();
    std::cout<<"[Main Thread]sub thread complete"<<std::endl;
    return 0;
}
```

在main中创建子线程.主线程和子线程共享地址空间,如果主线程销毁时子线程还在运行,子线程会一并退出
如果不希望主线程结束时还有子线程在运行,可以使用join()来阻塞主线程使其等待子线程结束.

```
std::thread::join()
```

>对于WINDOWS系统,主线程退出则子线程也退出
> Windows下主进程退出会调用exit直接终止整个进程,从而收回所有的进程资源
> 则线程自然会终止
> 在LINUX下,主线程退出子线程不会退出.此时这个进程会变成僵尸进程.
> 在主线程return前调用phread_exit()可以让主线程退出,子线程继续执行.

### 生命周期与线程分离

```
thread.detach();
```

detach()方法可以使主线程和子线程分离,也可以称之为守护线程.
detach的含义是让子线程与主线程彻底分离,不再与主线程有关,也不关心主线程的状况.

在主线程退出后,detach出去的子线程不一定退出.当所有的资源被释放后,守护线程再访问相关资源则会导致错误.

- 守护线程只访问自己函数的内部变量,不访问外部变量
- 主线程退出时通知守护线程,一并退出

由于detach()的麻烦我们通常不用这种方法,而是使用传统的join().
由于join()会导致主线程阻塞直至子线程退出,我们可以用某种通知手段在主线程退出时通知子线程也一并退出,减少等待的时间.比如使用全局变量作为一个flag.

> detach是相对于join而言的
> join可以理解为"会和",子进程与主进程会和,主进程要等待子进程结束并汇入
> detach则表示分离,子进程无需与主进程会和,当子进程结束则结束,主进程也不等待其结束.

### 传递参数

使用thread创建线程采用值传递.

```
class Entity{
public:
    Entity(){
        cout<<"Entity Constructed\n";
    }
    ~Entity(){
        cout<<"Entity Destructed\n";
    }
    Entity(const Entity& e){
        cout<<"Entity Copied\n";
    }
};
```

我们使用这样一个类来跟踪他的拷贝构造函数.

```
void threadFunc(int x,Entity e){
    cout<<"[SubThread] threadFunc running."<<endl;
}

int main(){
    thread th;
    Entity e;   // constructed
    th = thread(threadFunc,114514,e); // copied
    th.join();
    return 0;
}
```

由于值传递,在这个类作为参数传递的过程中需要多次复制.

```
Entity Constructed
Entity Copied
Entity Copied
Entity Destructed
Entity Copied
[SubThread] threadFunc running.
Entity Destructed
Entity Destructed
```

可以看到Entity一共被拷贝了三次,这是性能的极大浪费.
在线程参数传递的过程中,所有的参数都要复制.
我们要考虑使用引用传递这个类.

### 传递指针与引用

以指针和引用传递参数也有以下问题:
- 传递的空间已经被销毁
- 多线程共享一块访问空间
- 传递的指针的生命周期小于线程

```
// by pointer:
void threadFunc(Entity* e);

// main:
Entity* e = new Entity();
thread th(threadFunc,e);
th.join();
```

如果我们采用指针传递,我们可以得到如下结果:

```
Entity Constructed
[SubThread] threadFunc running.
[SubThread] Entity name: default
```

此时并没有出现销毁.子线程并没有这个传进来的指针的所有权.
一个潜在的问题是:在子线程执行之前,传递进来的指针已经被释放.
同理,使用引用传递也有这样的问题.

```
// by reference
void threadFuncRef(Entity& e){
    cout<<"[SubThread] threadFunc running."<<endl;
    cout<<"[SubThread] Entity name: "<< e.name <<endl;
}
// main:
Entity* e1 = new Entity();
thread th(threadFuncRef,ref(*e1));
th.join();
```

要注意thread创建线程时参数不能识别引用,需要用ref()来额外标注引用.

### 使用类创建线程

```
// class
class myThread{
public:
    string name = "";

    //入口函数
    void threadFunc(){
        cout<<"[myThread] thread name:"<< name <<endl;
    }
};

// main
myThread myth;
thread thread1(&myThread::threadFunc,&myth);
thread1.join();
```

使用类与类方法创建,参数中传递进函数地址和类地址.

### 线程封装基类

由于使用std::thread()非常复杂,我们可以弄一个基类,让
具体线程的启动,停止等都在基类中完成,并继承这个类去实现业务逻辑.
对于子类而言它隐藏了线程的内部结构,使用更简洁.
它的结构可能如下所示:

```
class baseThread{
    ...
};

class myThread : public baseThread{
public:
    void main() override{
        cout << "myThread main" << endl;
    }
}

// main:
mythread myth;
myth.start();
myth.stop();
myth.wait();
```

此时子类只需关注于入口函数等逻辑,而不用操心具体的参数传递,线程释放等

```
class baseThread{
private:
    thread th;                  // 线程对象
    bool is_exit = false;       // 控制退出的状态位
    virtual void main() = 0;    // 纯虚函数,让子类实现.
public:
    virtual void start(){
        is_exit = false;
        th = thread(&baseThread::main,this);    //使用类方法与类对象创建线程
    }


    vitrual void stop(){
        is_exit = true;         // 让退出位为true 可以用于判断何时退出
    }

    virtual void wait(){
        if(th.joinable()){      // 如果没有被join,则join
            th.join();
        }
    }

    bool exitStatus(){          // 返回退出状态
        return is_exit;
    }
};
```

这个基类实现了线程的启动,等待与退出,并定义了纯虚的main留给继承类实现.

```
class myThread : public baseThread{
    string name;

    void main() override{
        cout << "[myThread] main is running!\n" 

        while(exitStatus() == false){
            this_thread::sleep_for(100ms);
            cout << "[myThread] processing...\n"<<flush;
        }
    }
}
```

在基类的基础上,实现了我们自定义的入口函数.
当我们需要定义一个新的线程,只需继承基类即可,无需麻烦的重头使用std::thread();

```
int main(){
    myThread th;
    th.name = "Ruarua";

    th.start();
    th.sleep_for(3s);
    th.stop();
    th.wait();
    return 0;
}
```

把这个写好的基类写进头文件,以后直接继承它即可

>头文件用来声明让当前文件下编译器通过
>他们的定义由链接链入

### Lambda作为入口函数

对于一些简单的逻辑,可以直接使用匿名函数,免去声明和定义.

```
auto lambda = [](int i)->void{
    cout<<i<<endl;
};

thread th(lambda,114514);
th.join();
```

同理,类的成员函数也可以使用lambda

```
class Entity{
public:
    string name = "test test!";

    void start(){
        thread th([this]()->void{
            this->name = "Ruarua";
        });
    }
};
```

>Lambda的捕获列表
>lambda体内部不能直接访问类成员
>在捕获列表中写入需要捕获的内容 默认为值传递

## 线程状态与锁

线程通常分为几个状态:
- 初始化: 该线程正在被创建
- 就绪: 就绪列表中等待CPU调度
- 运行: 正在运行
- 阻塞: 该线程被阻塞挂起
- 退出: 运行结束 等待父线程回收其控制块资源

从初始化到就绪状态需要一些消耗,为此诞生了线程池.

线程与线程之间就要产生同步互斥问题.
竞争状态 Race Condition 指多线程同时读写共享数据.
临界区 Critical Section 指读写共同数据的代码片段.

### 互斥锁 mutex

mutex由C++11提出.

```
#include<mutex>
static mutex mut;

class testThread : public baseThread{
    void main() override{
        mut.lock();
        cout<<"[testThread] test001"<<endl;
        cout<<"[testThread] test002"<<endl;
        cout<<"[testThread] test003"<<endl;
        cout<<"[testThread] test004"<<endl;
        mut.unlock();
    }
};
```

这个例子沿用了之前做好的封装类.可以使用锁括住临界区.
一定要记得释放锁,否则将会造成死锁,所有其他的线程都被这个锁阻塞.
mutex的锁是由操作系统提供的.

```
for(int i=0;i<10;i++){
    if(mut.try_lock() == false){
        myLog::log(this_thread::get_id(),"try lock failed");
        this_thread::sleep_for(100ms);
        continue;
    }

    myLog::log(this_thread::get_id(),"test001");
    myLog::log(this_thread::get_id(),"test002");
    myLog::log(this_thread::get_id(),"test003");
    myLog::log(this_thread::get_id(),"test004");
    mut.unlock();
}
```

try_lock()对锁的申请如果失败,则不会阻塞而继续向下执行.
只要能得到锁,就能保证当前线程能够不被打断地执行完临界区的内容.
要注意try_lock()的开销,由于lock()可以直接阻塞而try_lock()是尝试阻塞,
在循环下可能会导致一直尝试却总是失败从而不断地消耗CPU资源.
所以try_lock()可以配合sleep_for()使用,让对锁的尝试暂停一段时间,放弃一段时间对CPU资源的索取.

### 互斥锁抢占

```
void threadFunc(int i){
    for(;;){
        mut.lock();
        cout<< i << "[in]" << endl;
        this_thread::sleep_for(1s);
        mut.unlock();
    }
}
```

在线程调用unlock之后紧接着又lock了一次,就会有如下问题:

- unlock之后并没有立刻释放锁,其他线程阻塞后又回到当前线程上锁继续执行.
- unlock之后成功释放了锁,而紧跟在循环后面的lock()又立马给当前线程上了锁,其他线程抢占失败.

结果就是该线程的锁一直被它占有,而其他线程一直在等待这个锁而被阻塞.
对于解锁后立马加锁的情况,通常在后面让该线程sleep一小段时间.

```
this_thread::sleep_for(1ms);
```

一是给操作系统一些时间释放掉这个锁,二是留给其他线程一些时间来抢占这个锁
从而防止某个线程一直占有某个锁.
通常而言在解锁和加锁之间会有额外的业务逻辑从而保证锁能被释放和被其他线程应用.
但要额外注意解锁后立马加锁的情况.

### 超时锁 timed_mutex

超时锁可以用来避免长时间死锁.
使用lock造成死锁排查时,调试成本非常大.
一个方案是使用try_lock,每隔一段时间尝试一下,但是需要考虑频繁try消耗CPU资源.
另一个方案就是try_lock_for,可以阻塞给定时间来尝试获得锁,若时间过后仍失败则继续.

```
static timed_mutex tmux;

void threadFunc(int i){
    for(;;){
        if(!tmux.try_lock_for(300ms)){
            myLog::log("try_lock_for","TIME OUT");
            continue;
        }
        cout << i << "[in]" << endl;
        this_thread::sleep_for(2s); //暂缓2s不释放锁 其他线程对锁的尝试将失败
        tmux.unlock();
        this_thread::sleep_for(1ms);
    }
}
```

>何时使用lock和try_lock
>lock()会一直等待锁的释放,线程会被阻塞.
>trylock()在获取锁失败后就立刻返回false,不会阻塞线程.
>lock适用于严格互斥的情景下
>trylock适用于允许获取锁失败的情景
>如在一个Web服务器中处理请求时用锁保证数据一致性
>如果无法立刻获得锁,那么应该迅速返回一个响应,而不让线程一直等待获得这个锁

### 递归锁 recursive_mutex

使同一个线程的同一把锁可以锁多次,避免一些不必要的死锁
当线程再次申请自己的一把锁,计数+1,解锁时计数-1.
当其他线程申请另一个线程已拥有的递归锁则会正常阻塞.

```
static recursive_mutex rmux;

void task1(){
    rmux.lock();
    myLog::log("task1","processing");
    rmux.unlock();
}
void task2(){
    rmux.lock();
    myLog::log("task2","processing");
    rmux.unlock(); 
}
```

假如task1和task2都是需要互斥的任务,则需要加锁包裹其临界区.
此时就适合使用递归锁.

```
void threadRec(){
    for(;;){
        rmux.lock();
        task1();
        myLog::log(this_thread::get_id(),"processing");
        this_thread::sleep_for(2000ms);
        task2();
        rmux.unlock();
        this_thread::sleep_for(1ms);
    }
}
```

当多个线程运行该入口函数时,某一线程获得了互斥锁
在它的函数体内部有3对互斥上锁与开锁,其他线程无法征用它持有的锁.
如果不使用递归锁而使用普通的互斥锁,该线程将死锁在对同一把锁的第二次申请上,因为此时这把锁未来得及释放.
递归锁在保持不同线程对锁的持有的同时,使同一线程内部得以多次申请已持有的锁.
这样就在一定程度上避免了使用普通互斥锁可能发生的死锁.

### 共享锁 shared_mutex

C++14提供 shared_timed_mutex 共享超时互斥锁;
C++17提供 shared_mutex 共享互斥锁

考虑一个同时读写的情景:
在写的时候,其他线程不能读写;
在读的时候,其他线程可读不可写.

共享锁有时也被叫做读写锁,用来解决读者写者问题:
即可以被多个读者拥有,但只能被一个写者拥有,
所以共享锁就相当于读者锁,互斥锁就相当于写者锁;

```
// C++14
shared_timed_mutex stmux;

// 申请共享锁
stmux.lock_shared();
stmux.unlock_shared();
// 申请互斥锁
stmux.lock();
stmux.unlock();
```

对于共享锁,如同10个读者一同在读,允许多个线程共用一个锁.
对于互斥锁,与共享锁相互阻塞,如同读时不能读(无共享锁)也不能写(无其他互斥锁)

## RAII与封装类

### 利用栈释放锁

RAII,Resource Acquisiton Is Initialization.
利用栈的特性实现锁的自动释放

```
class myLock{
private:
    mutex& mux_;    
public:
    myLock(mutex& mux):mux_(mux){
        cout<<"lock"<<endl;
        mux.lock();
    }
    ~myLock(){
        cout<<"unlock"<<endl;
        mux_.unlock();
    }
};
```

其核心要点在于把外部的锁交给类的内部管理,并以栈方式实例化.
当该对象脱离作用于,自动调用其析构函数释放掉这个锁.

### lock_guard

C++11严格实现基于作用域的互斥锁template lock_guard类

```
template<typename _Mutex>
class lock_guard
{
public:
typedef _Mutex mutex_type;

// 不带adopt 且为显式构造
// 传入mutex锁,并上锁
explicit lock_guard(mutex_type& __m) : _M_device(__m)         
{ _M_device.lock(); }

// 带adopt
// 收养一个锁,把一个裸露的锁托管进这个包装类
lock_guard(mutex_type& __m, adopt_lock_t) noexcept : _M_device(__m)
{ } // calling thread owns mutex

//析构函数
~lock_guard()
{ _M_device.unlock(); }

//不允许lock_guard引用转移
lock_guard(const lock_guard&) = delete;
lock_guard& operator=(const lock_guard&) = delete;

private:
mutex_type&  _M_device;
};
```

他的功能与上一节实现的功能差不多
由于是模板类因此支持多种mutex锁的类型
并且多提供了一个参数 adopt_lock_t

```
void testLockGuard(int i){
    {   
        // 现在观察锁,不再看成对的mutex,而看lock_guard所在的单独的大括号
        lock_guard<mutex> lock(gmutex);
        myLog::log(i,"processing");
    }

    for(;;){
        {
            lock_guard<mutex> lock(gmutex);
            myLog::log(i,"processing");
        }
        this_thread::sleep_for(500ms);  //不要放进锁里
    }
}
```

现在只需要在大括号内部即可完成自动地互斥访问,无需手动unlock,交给栈退出时析构来完成.
要注意的是不要把sleep放进互斥区.
因为互斥的本意是让当前线程独占CPU,而sleep的本意是让其他线程占有CPU放权,两者本身是矛盾的.

```
gmutex.lock();
{
    //已经拥有锁
    lock_guard<mutex> lock(gmutex,adopt_lock);
    
}   //结束释放锁
{
    lock_guard<mutex> lock(gmutex);
}
```

adopt,字面意思收养,收养一个已上了锁的mutex
见源码,使用adopt_lock参数与普通初始化几乎没有区别,只是不会额外上锁.
使用adopt_lock参数收养一个锁可以认为是把一个没有被包装的裸露锁托管进lock_guard.
在脱离作用域后这个锁也自动释放.

### unique_lock

上一节的lock_guard类,可见它不允许移动赋值.
如果我们需要可移动的互斥锁,可以使用unique_lock.
并且unique_lock支持手动解锁,而lock_guard只能利用栈解锁.

- 支持adopt_lock 收养锁
- 支持defer_lock 旨在只托管不上锁
- 支持try_to_lock 尝试上锁

```
// 几个构造函数

unique_lock() noexcept
: _M_device(0), _M_owns(false)
{ }

// 没有第二参数的构造
// 直接上锁
explicit unique_lock(mutex_type& __m)
: _M_device(std::__addressof(__m)), _M_owns(false)
{
lock();
_M_owns = true;
}

// 延迟上锁
// 可见own设置为false,不上锁
// 需要用户自行主动上锁
unique_lock(mutex_type& __m, defer_lock_t) noexcept
: _M_device(std::__addressof(__m)), _M_owns(false)
{ }

// 尝试锁 不成功则不占有锁
// 其实就是在包装内部调用相应锁的try_lock()
unique_lock(mutex_type& __m, try_to_lock_t)
: _M_device(std::__addressof(__m)), _M_owns(_M_device->try_lock())
{ }

// adopt 默认已上锁
unique_lock(mutex_type& __m, adopt_lock_t) noexcept
: _M_device(std::__addressof(__m)), _M_owns(true)

// 私有成员为锁的指针device
// bool类型的owns表示该锁是否上锁

private:
    mutex_type*	_M_device;
    bool		_M_owns; // XXX use atomic_bool
```

其析构如下

```
~unique_lock()
{
    if (_M_owns)
    unlock();
}
```

如果锁已经被own,则解锁.
可见不管是try_lock和defer_lock都可以根据锁的情况在析构时保证解锁.
但是adopt_lock默认当前锁已上锁,own被设置为true.
如果当前锁其实没有上锁则会导致析构时再次解锁而错误.

>顺便一提查看STL源码时可以发现其缩进和格式非常混乱
>这纯粹是因为STL作者脑子有病 没有什么特殊的理由

### shared_lock

之前提供的shared_mutex没有经过封装.
于是C++14诞生了他的封装类shared_lock

```
// 主要的构造如下
explicit
shared_lock(mutex_type& __m)
: _M_pm(std::__addressof(__m)), _M_owns(true)
{ __m.lock_shared(); }

shared_lock(mutex_type& __m, defer_lock_t) noexcept
: _M_pm(std::__addressof(__m)), _M_owns(false) { }

shared_lock(mutex_type& __m, try_to_lock_t)
: _M_pm(std::__addressof(__m)), _M_owns(__m.try_lock_shared()) { }

shared_lock(mutex_type& __m, adopt_lock_t)
: _M_pm(std::__addressof(__m)), _M_owns(true) { }

//析构如下
~shared_lock()
{
if (_M_owns)
    _M_pm->unlock_shared();
}

//不能转移
shared_lock(shared_lock const&) = delete;
shared_lock& operator=(shared_lock const&) = delete;
```

shared_lock支持

```
void lock()
{
    _M_lockable();
    _M_pm->lock_shared();
    _M_owns = true;
}

bool try_lock()
{
    _M_lockable();
    return _M_owns = _M_pm->try_lock_shared();
}
```

### scoped_lock

C++17支持scoped_lock.

有时会出现这样的情况:
在一段临界区中使用了两个锁,线程1锁住了锁1,线程2锁住了锁2;
此时线程1申请锁2,线程2申请锁1,就造成了死锁.

```
// C++11
lock(mux1,mux2);

// C++17
scoped_lock lock(mux1,mux2);
```

同时申请两个锁,如果无法同时占有两个锁,则给已占有的解锁,避免了死锁.
当一个线程需要同时占用多个锁,可以使用

### 多线程通信实例

```
class msgServer:public baseThread{
private:
    std::list<std::string> messages;
    std::mutex mux;
    
    // 处理消息
    void main() override;
public:
    // 发送消息
    void sendMsg(std::string msg);
};
```

实现一个简单的线程通信,主线程向子线程发送消息,子线程负责处理并打印.
其中,用来保存接受消息的messages使用std::list保存,为双向链表.

```
void msgServer::main(){
    while(isExit() == false){
        // 为了防止一直为空占用又不停地加锁占用CPU
        // 但是也意味着必须要10ms之后才能处理
        // 在这里锁就不如用信号量
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
        
        // lock
        std::unique_lock<std::mutex> lock(mux);
        if(messages.empty() == true){
            continue;
        }else{
            // 消息处理
            std::cout<<"[Server] received:" << messages.front() << std::endl;
            messages.pop_front();
        }
    }   // unlock
}
```

线程的入口函数要实现处理接收到的消息.
处理消息需要加锁处理,但要注意循环开头可以sleep一段时间防止不停地申请锁.

```
// main.cpp:

msgServer server;
server.start();
for(int i=0;i<10;i++){
    stringstream str;
    str<<"message "<<i+1;

    server.sendMsg(str.str());
    this_thread::sleep_for(500ms);
}
server.stop();
return 0;

// console:
[Server] received:message 1
[Server] received:message 2
[Server] received:message 3
[Server] received:message 4
...
```

只需简单地按照包装类留下的"接口"即可简单的使用其核心功能.

## 条件变量 condition_variable

常用于生产者消费者模型.
- 多个生产者与多个消费者分布在多个线程
- 生产者生产一个产品,通知消费者消费
- 消费者阻塞等待信号,获取信号后消费产品

> 并发有两大需求,一个是互斥,一个是等待
> 互斥用互斥锁就可以解决,如操作系统提供的mutex,或用户态的spinlock.
> 而条件变量用来解决等待需求.
> 考虑生产者消费者队列,如果只有mutex控制互斥,
> 那么消费者发现队列为空时,要么继续查询,要么sleep让渡给生产者一段时间再查.
> 前者会消耗CPU资源,后者会导致消费者线程经常sleep影响性能.
> 当有了条件变量,就可以当队列空时告诉消费者先wait,
> 等待不空时再唤醒.

可见条件变量是作用在锁上的,不能单独存在.
它为锁引入了"事件模式",当条件满足(或接到通知)则让事件发生并打开其控制的锁继续执行.
当不满足条件时应当释放锁,不要占有锁而是让渡给其他线程并继续等待"事件"的发生.

### 生产者与消费者模型.

以读写线程为例,写线程如下

```
// 1. 先获得锁
unique_lock<mutex> lock(mux);

// 2. 加锁后负责写
message.push_back(data);
...

// 3.写完了释放锁
lock.unlock();

// 4.因为刚刚写完,所以告知读者
cv.notify_one(); //通知一个等待信号的线程.
cv.notify_all(); //通知所有的等待信号的线程.
```

读线程如下:

```
// 1.读写线程共享一个mux
unique_lock lock(mux);

// 2.等待来自notify的通知,此时应解锁并阻塞当前线程.直到接受到通知后才重新上锁开始读
cv.wait(lock);

// 3.读
...
```

```
//reader:
// 试图申请锁
unique_lock<mutex> lock(mux);
// 等待信号传递,先释放锁不要占着茅坑不拉屎
// 当写者向读者发送信号表示自己刚写完,则占用锁并继续执行后面的内容
cv.wait(lock);
// 此时认为满足条件并继续执行读
while(msgs.empty() == false){
    cout<<"[reader "<< i <<"] read: "<<msgs.front()<<endl;
    msgs.pop_front();
}

// writer:
// 上锁
unique_lock<mutex> lock(mux);
msgs.push_back(ss.str());
cout<<"[writer]   write message."<<endl;
lock.unlock();
// 解锁
cv.notify_one();
```

其中cv.wait()也可以使用lambda表达式,其源码如下

```
template<typename _Predicate>
void wait(unique_lock<mutex>& __lock, _Predicate __p){
    while (!__p())
        wait(__lock);
}
```

cv.wait(lock,lambda)的第二个参数可以用lambda表达式,用来判断一些条件.
进入lambda表达式判断时上锁,判断true和false.
如果满足条件为true,则取消阻塞继续执行,不管通知的到来.
如果不满足条件为false,则会继续等待通知,释放锁并阻塞.

### 用条件变量控制线程退出

有了条件变量,就可以修改先前msgServer的例子中使用while判断是否为空的逻辑.

```
while(isExit() == false){
    std::this_thread::sleep_for(std::chrono::milliseconds(10));
    
    // lock
    std::unique_lock<std::mutex> lock(mux);

    // 等待为空
    // 在线程准备退出时可能会阻塞在这个wait
    cv.wait(lock,[this]{
        if(isExit() == true)
            return true;
        return this->messages.empty() == false;
    });

    while(messages.empty() == false){
        // 消息处理
        std::cout<<"[Server] received:" << messages.front() << std::endl;
        messages.pop_front();
    }
}   // unlock

void msgServer::sendMsg(std::string msg){
    std::unique_lock<std::mutex> lock(mux);
    messages.push_back(msg);
    lock.unlock();
    cv.notify_one();    // 发出通知
}
```

此处使用了lambda表达式,如果message为空,则继续阻塞;如果message不为空,则进入下面的消息处理阶段.
此时要注意一个阻塞问题:
当线程被阻塞在cv.wait()的同时,线程又执行了退出,
而cv.wait()一直在等待sendMsg给出的信号从而一直阻塞.
所以此时需要修改退出方法向该cv发出信号让他继续

```
// 修改了is_exit为protected
void msgServer::stop(){
    is_exit = true;
    cv.notify_all();
    wait();
}
```


这样就可以保证线程的正常退出.

## 线程异步

当开启一个线程后,不能确定什么时候得到它的结果,此为区别于同步的异步.
启动线程与获得结果在两个接口当中.

可以通过互斥锁和条件变量的方式,通过等待nofity来实现异步.
但面对各种复杂的情景,我们需要一种简单的方法来从线程出获得返回值.

### promise与future

C++11引入了 std::promise 和 std::future 两个模板类,用于实现异步.
std::promise用于在某线程中设置某个值或异常
std::future用于在另一线程中获得这个值或异常.

```
void func(promise<int> p){
    p.set_value(114514);
}

int main(){
    // 1. 创建promise
    promise<int> promise;
    // 2. 由promise构建他的future
    auto future = promise.get_future();
    // 3. 启动线程 以引用传递promise
    thread th(func,move(promise));
    // 4. 通过future获得值
    cout << future.get() <<endl;
    getchar();
    return 0;
}
```

在调用future的get方法时如果promise的值仍没有被设置,则会阻塞当前线程直到被设置.

```
// 默认构造已经完成了对future的赋值
promise()
    : _M_future(std::make_shared<_State>()),
      _M_storage(new _Res_type())
{
}

// std::promise::future()
// 它返回以当前promise内部保存的_M_future构造的future对象
// Retrieving the result
future<_Res> get_future()
{
  return future<_Res>(_M_future);
}
```

>如果线程1需要线程2的数据
>那么线程1需要promise承诺,并把promise传递给线程2表示2承诺向1给数据
>future表示期待着接受被承诺的数据

### packaged_task

packaged_task是一个可调用的目标,包装了一个任务,这个任务可以在另一个线程上运行.他可以捕获任务的返回值/异常,并存储在future中.

以下是一个同步的场景,task仍在主线程执行

```
// 将被包装的task
string myTask(int i){
    cout<<"[test] test pack "<<i<<endl;
    return "test return";
}

int main(){
    // 1.创建task并包装需要执行的任务
    packaged_task<string(int)> task(myTask);
    // 2.构建result对象
    auto result = task.get_future();
    // 3.传递参数执行task
    task(10);
    // 4.获得result
    cout<<"[result] " << result.get() <<endl;

    getchar();
    return 0;
}
```

可见此处只是把一个函数包装成一个task托管给packaged_task.
下面是使用包装好的task启用子线程的例子,参数依然放在第二个位置

```
thread th(move(task),114514);
```

future有wait_for方法,也有如timeout的状态status参数.
wait_for将返回一个future_status.下为源码

```
template<typename _Rep, typename _Period>
future_status
wait_for(const chrono::duration<_Rep, _Period>& __rel){
    // 如果已经future已经收到了返回值,则直接返回状态ready
    if (_M_status._M_load(memory_order_acquire) == _Status::__ready)
        return future_status::ready;
    
    if (_M_is_deferred_future())
        return future_status::deferred;

    // 此处为wait代码 等到了返回ready并同步
    if (_M_status._M_load_when_equal_for(_Status::__ready,
    memory_order_acquire, __rel)){
        _M_complete_async();
        return future_status::ready;
    }
    // 时间内没等到返回timeout
    return future_status::timeout;
}
```

由此我们在get阻塞前写超时判断：

```
// 做10次尝试 每次等100秒 如果没等到歇1ms
for(int i=0;i<10;i++){
    // 见源码 若status已经ready,不会真正等待
    if(result.wait_for(100ms) == future_status::ready){
        break;
    }
    if(result.wait_for(100ms) != future_status::ready){
        this_thread::sleep_for(1ms);
        continue;
    }   
}

if(result.wait_for(100ms) == future_status::timeout){
    cout<< "TIME OUT"<<endl; 
}else{
    cout<<"[result] " << result.get() <<endl;
}
```

对于这种业务一定要加上超时判定,因为我们无法保证另一个线程什么时候会给出结果并返回给future,也无法保证当前线程什么时候才会解开result.get()的阻塞.
如果一直无法得到结果,那么当前线程就可能一只阻塞,如果又恰好这个线程肩负着其他重要任务的同时另一个线程崩了无法给出结果则陷入了死锁.

### async

std::async是一个用于异步执行的模板**函数**,直接返回一个std::future.
他可以根据情况决定是否创建线程.

std::async有3种启动策略
- std::launch::async 直接创建线程 表示异步执行
- std::launch::deferred 任务将在遇到get和wait时同步执行,而不是提前执行等待呼出结果.
- std::launch::async | std::launch::deferred 默认 根据CPU资源自动选择同步异步

```
string testAsync(int value){
    cout<<"[test] thread id:" << this_thread::get_id() <<endl;
    return "test test";
}

int main(){
    cout<<"[main] thread id:"<< this_thread::get_id() <<endl;

    // 1.以推迟方式创建异步线程
    auto future = async(launch::deferred,testAsync,114514);
    // 2.调用get
    cout<<future.get()<<endl;
    getchar();
    return 0;
}
```

此模式下thread id是同一个,并没有创建新的线程,一切都是同步的.
若是使用async模式启动,即异步模式,则会创建线程异步执行.执行日志如下.

```
[main] in thread id:1
[main] create sub thread?
[test] in thread id:2
[test] message returned
[main] begin get
[result] test test
```

### 多线程实现base16编码

将字节拆分成2个4比特位,2^4 = 16
我们可以把这4个比特位映射到16个字符,即base16.

C++17提供的foreach的使用非常简洁即可利用多线程.
但它的性能不会有我们写的好

```
static const char base16Map[] = "0123456789abcdef";

void
base16Encode(const unsigned char*data,int size,unsigned char*out){
    for(int i=0;i<size;i++){
        unsigned char cur = data[i];
        // front back
        // 1234 5678
        // 1234 5678 & 0000 1111 -> 0000 5678
        char back = base16Map[cur>>4]; //高位补0
        char front = base16Map[cur & 0x0f];       //与掩码
        // 原4位扩充8位,容量扩大两倍
        out[i*2] = back;
        out[i*2+1] = front;
    }
}
```

base16的算法如上所示.
利用chrono提供的时间功能来测试效率

```
// 测试效率
{
    std::vector<unsigned char> data;
    data.resize(1024*1024*10);  //数据集大小

    // 捏造数据
    for(int i=0;i<data.size();i++){
        data[i] = i%256;
    }
    // 输出为2倍大小
    std::vector<unsigned char> out;
    out.resize(data.size()*2);
    // 计时start
    auto start = std::chrono::high_resolution_clock::now();

    base16Encode(data.data(),data.size(),out.data());
    // 计时end
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end-start);
    std::cout<<"TIME COST: "<<duration.count()<<"ms"<<std::endl;
}
```

本次测试为28ms.
下面是多线程的实现

```
void base16EncodeThread(const vector<unsigned char>&data,
vector<unsigned char>&out){
    // 数据大小
    int size = data.size();
    // 获得系统支持的线程核心数
    int thread_count = thread::hardware_concurrency();

    // 要能并发,那么数据得先能切片
    // 切片数据,均分 此时丢弃了余数
    int slice_size = size/thread_count;

    //如果数据size太小 那不如单线程
    if(size<thread_count){
        thread_count = 1;
        slice_size = size;
    }
    vector<thread> threads;
    threads.resize(thread_count);

    // 分配任务到各个线程
    // 1234 5678 90ab cdef
    for(int i=0;i<thread_count;i++){
        int offset = i*slice_size;
        int range = slice_size;

        // 最后一个线程 由于计算切片时向下舍入 要补上缺的一点
        if(thread_count > 1 && i == thread_count-1){
            range = slice_size + size % thread_count;
        }
        cout<<"["<<i<<"] "<<offset<<" : "<<range<<endl;
        threads[i] = thread(base16Encode,data.data()+offset,range,out.data());
    }
    //等待所有线程处理结束
    for(auto& thread : threads){
        thread.join();
    }
}
```

本次用时仅12ms

>多线程在性能上的优势更多的是利用了多核CPU的能力
>展现了并发性,打碎了数据相关性

### ForEach

C++17支持.

MinGW不支持execution 算了

## 线程池