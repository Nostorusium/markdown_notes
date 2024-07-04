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

## RAII

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

### lock_guard类

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

如上一节的lock_guard类,可见它不允许移动赋值.
如果我们需要可移动的互斥锁,可以使用unique_lock.
并且unique_lock支持手动解锁,而lock_guard只能利用栈解锁.

- 支持adopt_lock
- 支持defer_lock 延后拥有锁,不加锁,出栈区不释放
- 支持try_to_lock