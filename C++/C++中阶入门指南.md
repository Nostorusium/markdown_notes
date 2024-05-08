# C++中阶入门指南

## 初始化成员列表

**Member Initializer Lists**

```
class Entity{
private:
    std::string name;       // here runs the default constructor of string
    int scroe;
public:
    Entity():name("Unknown"),score(114514){}
};
```
在constructor后面加冒号来初始化成员变量
等价于:

```
Entity(string name,int score){
    this->name = name;
    this->score = score;
}
```

要注意的是初始化成员列表的顺序必须与变量成员的定义顺序一致.
使用初始化成员列表某种意义上是一种**代码风格**,尤其是当成员变量过多时
将会很方便

```
Entity()
    : name("tiansuo"),sex(0),old(24),score(114514),tele(1919810)
{
}
```

除此之外还有一个**功能**上的区别
对于如string类或自定义类对象作为类成员
在初始化的过程中会执行默认的构造函数

```
// constructor
Entity(){
    name = "unknown";
}
```

执行Entity构造函数为name赋值前,定义这一string时就已经执行了string的无参构造函数；
随后重新有参构造取代了第一次的的string

```
Entity() : name("TiansuoHaoer"){
    ...
}
```

而使用初始化成员变量则不会,只会执行一次String的构造函数.
使用列表初始化性能上的优势是因为:
使用构造函数初始化的本质是赋值操作,在赋值操作中会产生临时对象,临时对象的构造函数和析构函数造成了性能损耗,而初始化列表可以避免临时对象.

以下为使用初始化列表的场景
1. const成员变量只能用初始化列表来初始化,它无法在定义时赋值,又因为const也不能在构造函数里赋值
2. 初始化的数据成员是对象,在使用构造函数赋值时需要创建临时对象额外执行它的构造函数造成性能损耗
3. 用于引用类型的初始化 引用类型必须有初始值

## 实例化

```
Entity entity;
```
这一小段简单的代码就已经完成了实例化,执行默认构造函数.

```
void Function(){
    int a=2;
    Entity entity = Entity(114514);
}
```

这两个成员在抵达下面的大括号就会被自动释放.
它们在栈上分配,有着自动的生命周期和作用域,作用域即它所在的大括号

```
int main(){
    {
        Entity entity(114514);
        std::cout << entity.getValue() << std::endl;
    }
    // here
}
```

当程序执行脱离了这个括号,则这个entity就不再存在,被从栈中自动释放.
由于在堆上分配更费时间,且需要手动释放内存,所以不要滥用new.
smart pointer智能指针在后面介绍

new是一个操作符,如加减乘除一样是一个operator,这意味着你可以重载并改变他的行为.
new是一个函数,返回一个void指针,表示没有类型的纯地址.new的具体实现依赖于标准库的实现
通常而言new会涉及malloc的使用


## 显式隐式构造

```
public:
    Entity(const std::string& name)
        :name(name),age(-1){}
    Entity(int age)
        :name("unknown"),age(age){}
```
以下为显式构造:
```
int main(){
    Entity e1("田所浩二");
    Entity e2(24);
}
```
隐式构造如下:
```
Entity e1 = "田所浩二";
Entity e2 = 24;
```
在这个过程中,等号右面会自动隐式的构造出一个Entity对象
下例同理
```
void printEntity(const Entity& e){
    std::cout << e.age << std::endl;
}
int main(){
    printEntity(24);
    printEntity("name");  //error
    printEntity(std::string("田所浩二"));
}
```

编译器认为,参数24可以被转换成一个Entity,因为这个参数符合其构造函数的参数格式.
同理,提供string类型的参数进去时也可以进行隐式转换
关键字explicit会强制构造函数不进行隐式转换

```
explicit Entity(int age):age(age){}
```

在写低级封装时也许会用到来避免偶然转换和导致性能问题与bug

## 操作符重载 operator overloading

操作符重载在JAVA中不支持,在C#中得到了部分支持
总的来说应当尽可能少的使用操作符重载,只在非常有意义的时候使用
如果其他人需要查看你的操作符定义,类或结构定义时可能会看不懂
例如: 在一个数学相关的类里相加两个数学对象,重载+操作符是很自然的

```
int main(){
    vector pos(0.0f,1.0f);
    vector speed(0.5f,1.5f);
    vector powerup(1.1f,1.1f);

    // without overloading
    vector result = position.add(speed.multiply(powerup));
}
```
在JAVA中,这是唯一的写法.C++的操作符重载支持我们这么写:

```
vector result = position + speed*powerup;
```

在类中定义操作符重载:

```
vector operator+(const vector& other) const{
    return vector(x+other.x,y+other.y);
}
vector operator*(const vector& other) const{
    return vector(x*other.x,y*other.y);
}
```

同理,在io流输出时操作符<<也需要根据类进行重载
操作符<<接受两个参数,一个是ostream(在左),一个是要输出的参数(在右)
下面重载它使其支持vector的输出

```
std::ostream& operator<<(std::ostream& stream,const vector& other){
    stream << other.x << "," << other.y;
    return stream;
}
```
这里是针对输出流的函数重载,让<<操作符调用这个函数
函数内部的stream由于已经实现了<<的重载所以直接用stream << other 就可以
最后return stream作为新的输出流.
操作符重载其实就是函数重载,我们为原有的操作符新增了新的参数类型使其实现我们想要的自定义功能

在JAVA中常用.equals来判断相等,我们也可以重载==操作符
```
bool operator==(const vector& other) const{
    if(x==other.x && y==other.y){
        return 1;
    }
    return 0;
}
```

注意到<<重载不能放在类内,因为cout并不是类的成员,需要放在全局空间定义
且在头文件里声明的方式会导致重复定义.

## 对象的生存周期

```
int* createArray(){
    int array[50];
    return array;
}
```

这是一个经典的错误,它返回一个在栈上建立的数组指针,而这个数组在离开作用于后就被释放.
这个指针所指毫无意义.如何更好利用栈上的变量?
栈上变量的好处是它有自动化的释放,无需手动delete.
作用域指针本质上是一个类,一个指针的包装器
在构造时在堆上分配指针,然后在析构时删除指针,new与delete被自动化了.

```
int main(){
    {
        Entity* e = new Entity();
    }
}

```
这是一个堆上分配的指针,但我们希望在它跳出作用域时自动删除它
std标准库提供了unique_ptr,作用域指针,下列一个自行实现的作用域指针

```
class scopedPtr{
private:
    Entity* ptr;
public:
    scopedPtr(Entity* ptr) : ptr(ptr){}
    ~scopedPtr(){delete ptr}
};
int main(){
    {
        // be aware this is not "new scopedPtr"
        // p is created on stack area!
        scopedPtr p = new Entity();
    }
}
```

当new Entity()返回Entity的指针,它作为scopedPtr的隐式构造参数构造了scopedPtr;
而又因为scopedPtr的对象是在栈上分配的(没有new它自己),它离开这个括号后就被释放并执行析构函数
析构函数中用来释放这个地址对应的Entity实例对象

这个例子实现了smart_ptr与unique_ptr的最基本功能.
现在我们就拥有了自动构造自动析构,离开作用域后自动销毁的指针

## 智能指针

智能指针意味着当你new后不用自己delete释放
智能指针是对原始指针的包装.

### unique_ptr

unique_ptr即作用于指针,当指针超出自身作用于自动销毁
之所以叫unique_ptr是因为它应当是unique的,你不能复制它
如果复制了将会得到两个指向同内存的unique_ptr,当其中一个释放则另一个也被一同释放
它应当是对你真正想要的指针的唯一引用

使用头文件#include<memory>
unique_ptr的构造函数是explicit的,不要用等号来隐式构造

```
int main(){

    // this is wrong
    // std::unique_ptr<Entity> entity = new Entity();  
    std::unique_ptr<Entity> e(new Entity());
    e->print();
    ...
}
```

另一种更好的方法是:

```
std::unique_ptr<Entity> entity = std::make_unique<Entity>();
```

这是出于异常安全考虑的,当构造函数碰巧抛出异常时更安全,保证你得到一个空指针而不会导致内存泄漏.
这就是最简单的智能指针,只是一个栈分配对象,甚至没有开销.而如果你试图把它传入一个函数或者在另一个类保存这个指针将会报错
unique_ptr的copy构造与copy复制操作被删除了,来防止你自己给自己挖坑.

### shared_ptr

如果想要共享,可以使用shared_ptr,其实现取决于编译器与标准库
但总的来说它使用的都是叫做**引用计数**的东西,一个跟踪统计这个指针有多少引用的方法.
一旦它计数为0,则销毁.

```
                 +-------------+    some block
shared_ptr1 +--->|             |  +------------+
                 |             |  |            |
shared_ptr2 +--->| same address|  | count == 3 |
                 |             |  |            |
shared_ptr3 +--->|             |  +------------+
                 +-------------+
```

```
std::shared_ptr<Entity> sharedEntity2 = std::make_shared<Entity>(); //better
std::shared_ptr<Entity> sharedEntity2(new Entity());
```

shared_ptr会额外分配一块**控制块**内存来保存引用计数.
在shared_ptr中使用make构造的好处在于,如果先new Entity再传递给shared_ptr的构造函数
先后一共分配了两次,先分配Entity再分配控制块
而make_shared将两者结合起来了,效率更高
可见shared_ptr因为引入的计数系统是有开销的

### weak_ptr

weak_ptr经常与shared_ptr一起使用
当shared_ptr被赋值给一个weak_ptr,它不会增加引用计数
这意味着这个weak_ptr不会占有Entity的所有权,它不参与对Entity的何时释放的判断
比如在某个关于Entity的排序函数,传入一个weak_ptr,你并不实际关心Entity对象是否还活着
如果从weak_ptr判断这个底层对象已经似了,那就不排序
此时这个weak_ptr不会影响它的正常释放,Entity对象不会因被占据而延迟释放

换言之,weak_ptr不影响资源的生命周期

```
int main(){
    {
        std::weak_ptr<Entity> e0;
        {
            std::shared_ptr<Entity> sharedEntity = std::make_shared<Entity>();
            e0 = sharedEntity;  //  shared_ptr still count as 1
        }
    }
}
```

### 总结

由于unique_ptr开销最小,尽可能多用.
当指针需要共享时,使用shared_ptr和weak_ptr

## 友元

在类内声明的友元函数意味着允许这个类外函数使用该类的private成员

## 复制与性能分析

在传递参数时由于值传参其效率较低,考虑使用引用.
如果不涉及修改,使用引用时应用const修饰防止原内容被意外修改.
总结: 使用const引用来传递你的对象;

### C++的默认复制

类结构如下
```
class myString{
private:
    char* buffer;
    unsigned int size;
};
```

```
int main(){
    myString str1 = "this is a string";
    myString str2 = str1;
}
```

此处的复制发生了什么?
str1的类成员buffer和size被复制到一个新的内存地址,即str2的位置.
这称作浅拷贝shallow copy.
此时str1与str2的buffer指向同一个位置,当生命周期结束调用析构函数时,这一位置被delete两次从而报错.

要让str2拥有自己的buffer需要深拷贝deep copy,深度复制要完整地复制对象而不是复制值.一个简单的方法是专门写个方法,但这很低级且麻烦.

### 复制构造函数

copy constructor是为了str2调用的构造函数.
当你为一个变量赋值:
```
int a = b;
```
左右两侧的类型相同,此时调用的是一个复制构造函数
C++自动提供了一个复制构造函数,即刚才默认的浅复制
```
myString(const myString& other);
```
它只是复制另一个对象的内存,做浅复制

如果我们不希望复制可以这样写:
```
myString(const myString& other) = delete;
```
unique_ptr也是这样做的来保证它不会被复制.
为了深度复制,可以重写这个复制构造函数:
```
myString(const myString& other){
    size = other.size;
    buffer = new char[size+1];
    // dest,src,size
    memcpy(buffer,other.buffer,size+1);
}
```
这样我们就自定义了复制行为.
myString str2 = str1实际上是隐式构造,以str1为参数构造str2.

## 箭头操作符

箭头操作符在90%的情况下都是作为快速解引用使用,但有些情况下重载它将是很好的:

这是一个作用域指针的结构:

```
class ScopedPtr{
private:
    Entity* ptr;
public:
    ScopedPtr(Entity* entity) : ptr(entity){}
    ~ScopedPtr{
        delete ptr;
    }
}
```

下面将是一个重载->运算符的好场景

```
int main(){
    ScopedPtr entity = new Entity();
    (entity.ptr)->printEntity(); // private class member,can not be done
}
```

因为作用域指针的包装,无法如同使用一般指针一样直接使用->
所以做如下重载:
```
public:
// (LeftObj->) gives the Entity ptr
Entity* operator->(){
    return ptr;
}
```
现在就可以方便的使用->访问Entity了:
```
int main(){
    ScopedPtr entity = new Entity();
    entity->printEntity(); // private class member,can not be done
}
```

### 利用->得到偏移量

```
struct Vector{
    float x,y,z;
}
int main(){
    int offset = (int) &((Vector*)nullptr) ->z;
    std::cout<<offset<<std::endl;
}
```
x,y,z将给出0,4,8,对应了他们在结构中的偏移量.
nullptr的地址值为0,先转化为Vector类型指针,此时Vector->x的寻址方式是在基址上+4;
利用&取指得到地址,为0+0,0+4,0+8;最后把地址转换为int赋给offset

```
          |<----  *(0+4) ---->|
          |                   |
(int) &   ((Vector*)nullptr)->y
          |                |
          |<------ 0 ----->|
```

在处理字节流时经常需要得到偏移量,这将很有用

## std::vector

C++提供了很多std标准模板
在实际的工程中我们需要重写很多,因为std模板库中的不是很快,最后通常需要创建自己的容器库.

### 基本使用

vector这一名字很有迷惑性,他其实就是动态数组array.
考虑到在计算机中vector,向量本身就是数组,也可以接受.

```
struct Vertex{
    float x,y,z;
}
int main(){

    // put type inside the <>
    std::vector<Vertex> vertices;

    // push_back meaning add
    vertices.push_back({1,2,3});
    vertices.push_back({4,5,6});

    // .size() is ok
    int size = vertices.size();
    
    for(int i=0;i<size;i++){
        std::cout<< vertices[i] <<std::endl;
    }

    // advanced for loop
    for(Vertex v : vertices){
        std::cout<< v <<std::endl;
    }

    // erace a single vertex,hard to explain
    vertices.erase(vertices.begin()+1);

    // clear the vector
    vertices.clear();
}
```
在<>里面填入类型,表示这个vector里面装的内容.
不同于JAVA,C++中可以往<>添加原语类型
在C++里填入<int>是完全可以的,而JAVA只能<Integer>
考虑到vector的复杂程度,确保传参时使用引用传参

### Vector Optimizing

如果Vector被装满了,需要重新找一个足够大的位置重新分配,把原先的内容复制过来并加上新加入的元素.如果我们需要不断地重新分配就会导致效率低下.
这是一个方向的优化策略,优化复制

```
struct Vertex{
    float x,y,z;
    Vertex(float x,float y,float z)
        : x(x),y(y),z(z){}
    Vertex(const Vertex& vertex)
        : x(vertex.x),y(vertex.y),z(vertex.z){
        std::cout << "copied" << std::endl;
    }
}

int main(){
    std::vector<Vertex> vertices;

    // copied twice
    vertices.push_back({1,1,4});
    vertices.push_back({5,1,4});

    // copied 4 times
    vertices.push_back(Vertex(1,1,4));
    vertices.push_back(Vertex(5,1,4));
}
```
后一种方法每次push_back都要复制2次.

1. Vertex先在main的栈中被构造,随后被复制到vector里去.如果能在合适的地方构造它就可以省去复制这个过程.push_back基于复制的,而emplace_back是基于构造的

2. vertices的capacity随着push_back不断增长,当超过了容量就需要重新找地方重新分配.如果能提前获知所需要的容量就可以省去这个步骤.

```
int main(){
    std::vector<Vertex> vertices;
    vertices.reserve(2);
    vertices.push_back(Vertex(1,1,4));
    vertices.push_back(Vertex(5,1,4));
}
```

使用reserve来为vector指定一个大小,表示直接找2个Vertex大小的内存用以储备.
这样运行后只会复制两次,我们成功省去了vector长度增长后重新分配空间的过程
注意这不同于resize或者在构造函数中写:
```
std::vector<Vertex> vertices(3);    // wrong
```
这段代码的含义是构造3个顶点对象,而不是分配3个Vertex大小的内存空间.
目前我们只希望vector有足够的内存容纳vertex作为储备. 

```
int main(){
    std::vector<Vertex> vertices;
    vertices.reserve(2);
    vertices.emplace_back(1,1,4);
    vertices.emplace_back(5,1,4);
}
```

更进一步,改用emplace_back直接在vector中构造对象而不是复制过去.
此时执行,Vertex一次也不会被复制.

## @TODO C++ Library 

库里通常分为两部分:include 与 library.
include目录存放头文件,library目录存放二进制文件.
静态链接通常更快,因为编译器或者链接器实际上可以执行某种优化.
.dll为动态链接库 .lib为静态链接库


## 模板 Templates

从JAVA或者C#看template可能认为它是一种**泛型**,但远比泛型强大.

template允许你根据用途来编译,让编译器基于一些规则为你编写代码
编写模板实际上就像创造一个蓝图.

一个使用模板的场景是编写一类功能相近而参数不同的函数
如print出int,float,string.
如果使用重载需要写很多重复代码

```
template<typename T>
void print(T value){
    std::cout<<value<<std::endl;
}
int main(){
    print(114514);          //implicit
    print(1919810);
    print<float>(114.514);  //explicit
}
```
在尖括号里给typename取一个名字T,用T代表一类类型.
尖括号内<>写入的是模板参数,可以是typename,class
在使用时可以隐式或显式的在<>内写入类型.

要注意template只是一个模板,在调用时才被真正的创建.
如果从来都没有调用过,则这个函数根本都不存在,这也意味着模板内部的语法错误不会被识别,
直到编译器自动构建它的内容.

模板也可以用作类,参数也可以不止一个.
```
template<typename T,int N>
class Array{
private:
    T array[N];
public:
    T getsize() const{
        return N;
    }
};
```
这个例子里数组的长度由运行时决定.

```
int main(){
    Array<int,10> array;
    Array<std::string,10> stringList;
    std::cout<<array.getsize()<<std::endl;;
}
```
int和10作为模板参数传入并作为T与N
继而创建了长度为10的int数组

在写例如日志系统,材料系统的时候,模板将非常有用.

## 宏 Macro

宏在预处理器处理后只做纯文本替换

```
#include<iostream>
#define WAIT std::cin.get()
int main(){
    WAIT;
}
```

不要用宏这样做,因为别人看到WAIT不知道它代表什么

```
#define LOG(x) std::cout<<x<<std::endl
```

宏里允许有这样的参数.

考虑这样一个场景: 你开发的应用里有Log日志系统,用于在debug时使用,而你不希望在Release版本里包括日志
所以可以使用宏来实现:

```
#ifdef MY_DEBUG
#define LOG(x) std::cout<<x<<std::endl
#else
#define LOG(x)
#endif
```

在编译debug时,让预处理器定义一个MY_DEBUG,这样宏就会正确的使用LOG(x)
在编译release时不做这个定义,那么就会使用第二个LOG(x),其没有任何作用,并且连带着;一同清空
具体对预处理器的操作可以使用cmake;
定义也可以有具体值:MY_DEBUG == 1;

```
#ifdef MY_DEBUG == 1
#define LOG(x) std::cout<<x<<std::endl
#elif defined(MY_RELEASE)
#define LOG(x)
#endif
```

定义宏时可以使用\反斜杠来换一行,因为宏定义是按行的,用回车会被认为结束定义

```
#define MAIN int main()\
{\
    std::cout<<x<<std::endl;\
}
```

## auto

auto的出现让C++变成一个弱类型的语言,你可以到处乱用auto.
总的来说auto的使用也取决于代码风格.
下面是一个使用auto的场景

```
std::string getName1(){
    return "Rua1";
}
char* getName2(){
    return "Rua2";
}
```

尽管都是返回一个字符串但他们的类型不同;

```
int main(){
    auto name1 = getName1();
    auto name2 = getName2();
}
```

使用auto的好处之一就是当你改变了api,改变了返回类型,变量不必跟着改变,auto会自动识别.
但其坏处就是这种改动会破坏依赖其类型的功能:
比如当返回类型从String类变成char*,那么name.size()便不再有效.使用auto也会破坏一些可读性.
在这种情况下就最好不要使用auto.

而下面是一个适合使用auto的场景:
下面的例子中使用了迭代器,日后介绍

```
int main(){
    std::vector<std::string> strings;
    strings.push_back("name1");
    strings.push_back("name2");
    for(std::vector<std::string>::iterator it = strings.begin(); it!=strings.end();it++){
        std::cout<< *it <<std::endl;
    }
}
```

在定义迭代器it时可以注意到写了一大长串,这个时候用auto代替将是极好的

```
for(auto it = strings.begin(); it!=strings.end();it++){
        std::cout<< *it <<std::endl;
}
```

用auto代替一大长串类型定义将是极好的
而对于一般类型例如int,string使用auto只会降低他的可读性.不要在这些简单类型上用auto

从C++11之后,auto可以用于跟踪返回类型
```
auto getName() -> char*{
    return "ruarua";
}
``` 

## std数组

```
#include<array>
int main(){
    std::array<int,5>;
}
```
其使用与C风格的array一模一样,但它提供了.size(),并且支持越界检查,越界的语句不会有任何作用.
其性能和原生数组差不多.
尽管.size()会返回其长度,但实际上std数组并不实际存储数组长度,size()会把泛型中的"5"直接返回,所以并没有占据额外的空间.

当你想使用一个函数打印数组时,若使用普通数组则把数组长度作为参数写入;
而使用std数组,但仍需要在类型中的<>写入其长度.在这方面std数组没有优势;
但std数组仍有很大的优势,它支持迭代器,支持sort排序,同时维持了size.
所以尽可能的使用std数组,它的性能与原生数组相差无几

## 函数指针

函数指针非常raw style,允许把函数赋值给变量或者作为函数的参数.

```
void HelloWorld(){
    std::cout<<"Hello World!"<<std::endl;
}

int main(){
    auto function = HelloWorld;     // a function pointer
    auto function = &HelloWorld;    // the same

    function();                     // call HelloWorld
}
```

函数只是一堆CPU指令,以某种方式存储在二进制文件中.
此时它的类型是void(*functionName)()
void表示它的返回值
此处第一个括号里*号后面需要一个名字;
第二个括号表示所指函数需要的参数
之所以有第一个括号是因为如果不加第一个括号他会被误认为是void* name()

```
void(*function)() = HelloWorld;
```

由于写起来比较麻烦所以多数情况下会使用auto或typedef
```
int main(){
    typedef void(*functionPtr)();   // use "functionPtr" to represent void(*name)()
    functionPtr function = HelloWorld;
}
```

当含有参数:

```
void HelloWorld(int a){
    std::cout<<"Hello World! with value:"<< a <<std::endl;
}
int main(){
    typedef void(*functionPtr)(int);
    functionPtr function = HelloWorld;
    function(114514);
    function(1919810);
}
```

一个使用函数指针的场景:

```
void printValue(int value){
    std::cout<<value<<std::endl;
}

void forEach(const std::vector<int>* values,void(*func)(int)){
    for(int value:values){
        func(value);
    }
}

int main(){
    std::vector<int> values = {1,1,4,5,1,4};
    ForEach(values,PrintValue);
}
```