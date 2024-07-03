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

一些关于线程的用法见注释


```
int main(){
    // 启动子线程 子线程的内容由函数指针传入
    std::thread thread1(threadFunc);

    std::cout<<"[Main Thread]main thread id: "<<std::this_thread::get_id()<<std::endl;
    std::cout<<"[Main Thread]wait sub thread"<<std::endl;

    // join()表示等待子线程退出,阻塞了主线程
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