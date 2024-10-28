# QT速通

## 基础使用

### 信号与槽 signal & slot

你点击了一个按钮,就相当于发出了一个信号.
而所谓的槽指的是接下来要执行的一段代码,即所谓的槽函数.
信号和槽是独立的,我们要让他们关联.有很多实现方法.

最基本的方法是在hpp中定义好槽函数并给出实现,这一过程可以使用ui工具里转到槽自动生成.
控件的名字要与SLOT中的槽函数对应.
除此之外还支持使用connect来连接信号与槽,在构造函数中写.

```
// 在构造函数里写
// 链接信号和槽 4个参数为:谁发出信号 发出什么信号 谁来接收信号 怎么处理
connect(ui->cmdEditor,SIGNAL(returnPressed()),this,SLOT(on_butten_commit_clicked()));

// 不用宏也可以用取地址的方式 推荐这么写
connect(ui->butten_cancel,&QPushButton::clicked,this,&Widget::on_butten_cancel_clicked);

// 可以使用Lambda表达式,只需要三个参数
connect(ui->butten_browse,&QPushButton::clicked,[this](){
    QMessageBox::information(this,"message","点击浏览");
});
```

要注意信号和槽是可以重复定义的,你可以定义完全相同的信号与槽
因此要时而注意及时disconnect防止一个信号被两个相同的槽执行.
并且在slot中固定的命名格式会与ui中直接关联成一个信号槽.
此外connect则是手动绑定槽.
要注意如果命名格式符合的同时又绑定了connect会绑定两次.
所以要注意重复connect情况

>更形象的理解信号和槽
所谓信号可以是任何外部事件 比如敲击键盘,点了一下鼠标
QT为我们定义了一系列各种类型的信号,他们在不同的时刻被触发.
比如在UI界面设计了一个button,那么当这个button被点击就会触发一个信号.
我们需要做的是把这个信号与某个处理行为绑定起来,即所谓的槽.
通常而言我们不希望某个槽不停地执行来自各个位置的同一信号,
因此在connect绑定的过程中可以看到参数里强制要求你给定信号的发出者.
这样通过connect,我们就可以实现这样的绑定:
来自XXX所发出的XXX类型的信号,交给XXX槽来处理.

### 事件

窗口的event()用来处理事件 比如键盘按快捷键,鼠标移动等.
QT将系统产生的消息转化为QT事件,QT事件被封装成对象.所有的QT事件都继承自QEvent类.
任意的QObject对象都具备处理QT事件的能力.

### Action

Action表示一个抽象的用户操作,经常和用户组建关联.
通常而言它用于把用户操作和UI进行关联,并且能够在多个组件中重复使用.
你只需要把QAction和槽函数链接,然后绑定到不同的组件就可以简化信号槽的连接管理.
它适合操作重复、逻辑和UI分离的场景。

QAction通常和菜单栏，工具栏结合。

上文的Event是一种更底层的事件,处理的是GUI事件(如鼠标点击,键盘按键等),他代表的是系统或者用户发生了什么.所以Event更适合捕捉用户输入等事件的管理.

总结:QAction适用于高层次的用户操作抽象,QEvent是低层次的事件处理.

## 事件机制 Event  Mechanism

Qt框架的底层是一个事件驱动的系统,事件系统和事件循环是共同构成了QT的核心功能.

事件系统Event Sytstem负责处理各种事件:如鼠标点击,键盘输入,定时器等.他们被发送到应用程序的不同部分,通常是某个窗口部件(Widget)并触发响应动作.

事件循环Event Loop负责不断地检查与分发事件,当程序启动,循环就开始运行并不断地检查新的事件发生,并把他们发送到适当的对象进行处理。当没有事件处理时，事件循环会等待新事件的发生。这两者的紧密协作保证了Qt能够响应用户操作和其他事件源的动作。

QT启动程序后就会进入主事件循环,随后判断事件队列中是否有事件。
当拿到一个事件会判断是否为GUI事件，如果是，则处理GUI的更新和操作。
如果不是，则判断是系统级事件还是主线程定时器事件，并执行相应的处理。
启动子线程后，子线程也进入事件循环，但子线程不处理GUI事件。
当所有线程都结束，关闭程序。

### 事件与事件循环

Event通常是QEvent类的实例，它包含了事件的所有信息：事件类型Type,时间戳Timestamp等。
当发生事件时，一个QEvent对象会被创建并发送到事件循环中。事件是系统和用户交互的基础，使得程序能够以非阻塞的方式响应外界的变化。当事件被插入循环，按照事件发生的顺序，从中取出并分发这些事件到相应的对象进行处理。这个过程是异步的，意味着事件的产生和处理是分离的。参考一下“消息队列”。

每个线程都可以有自己的事件循环。主线程的事件循环主要处理与用户界面相关的事件，工作线程的事件循环则特定与该线程的任务。

### 事件类型和处理

QT定义了一系列的事件类型，表示不同的动作和通知。
- 用户界面事件 UI Event：如QMouseEvent，QKeyEvent等。
- 窗口系统事件 Window System Events：如QShowEvent，QCloseEvent
- 定时器事件 Timer Events：由QTimer触发的事件。
- 网络事件 Network Events
- 自定义事件 Custom Events

在Qt中，事件的处理通常要重写QObject派生类中的事件处理函数。

```
void MyWidget::mousePressEvent(QMouseEvent* event){
    ...
}
```
事件在Qt中可以传播，如果一个子控件不处理某个事件，那么这个事件会传播到父控件。

### 主事件循环

主事件循环位于主线程,特别处理所有与GUI相关的事件，并承担着处理系统级事件的任务。

```
#include <QApplication>
#include <QPushButton>
int main(int argc, char *argv[]) {
    QApplication app(argc, argv); // 初始化应用程序和主事件循环
    QPushButton button("Hello, World!");
    button.show(); // 显示一个按钮
    return app.exec(); // 启动主事件循环
}
```

app.exec()启动主事件循环,loop持续到exit()被调用，随后返回值。

### 事件处理流程

当一个GUI事件发生，流程如下：
1. 事件生成：事件被创建并加入到队列中。
2. 事件循环处理：事件循环中，事件被从队列中取出。
3. 事件分发：事件被分发给QWidget或者QObject派生类的实例。
4. 事件相应：事件处理函数被调用(如mousePressEvent)

我们通常重写某个控件的事件处理函数，当事件被分发给对应实例，则执行自定义的事件处理。

### 系统级事件

系统级事件通常包括：
- 应用程序生命周期事件：如启动，退出。
- 系统通知：如电源情况，系统时间更改。
- 设备时间：如USB设备的插入和拔出。

```
protected:
bool event(QEvent *event) override {
    if (event->type() == QEvent::ApplicationActivate) {
        qDebug() << "应用程序被激活";
    } else if (event->type() == QEvent::ApplicationDeactivate) {
        qDebug() << "应用程序失去焦点";
    }
    return QApplication::event(event);
}
```

大多数系统级事件的处理通常隐藏在QT内部，但我们可以重写特定的事件处理函数来相应。
比如重写比较高级的event。在返回时，不要忘记继续延续原先event的处理。

### 线程与事件循环

每个线程都可以拥有自己的事件循环，但这不是自动启动的。

```
class WorkerThread : public QThread {
    Q_OBJECT
public:
    void run() override {
        // ... 执行任务 ...
        exec(); // 启动事件循环
    }
};
```

你需要使用exec()来启用线程的事件循环。它们尤其适用于需要定期或在特定条件下执行任务的场景。
在线程中的时间处理通常涉及以下几个步骤：
1. 启动事件循环
2. 事件生成与处理
3. 事件响应
4. 线程间通信：Qt的信号和槽机制在这里发挥重要作用。它允许线程间的安全通信。

在这里，信号槽机制排上了用场，比如线程发射一个信号通知主线程某个任务已经完成。

### 线程间的信号槽通信

当一个信号在一个线程中被发射，而对应的槽在另一个线程中，则QT的事件系统会安排这个槽在目标线程中被调用。
也就是说槽函数的执行在接受信号的线程的上下文中，能够保证槽函数“所属”于某个线程。

在多线程应用中，有以下主要的实践方法：
1. 通过继承QThread创建和管理线程。
2. 在线程中处理耗时任务，如计算密集型/IO操作，避免阻塞主线程。
3. 利用信号槽安全地在线程间传递数据。此时也要注意临界区的互斥。
4. 事件循环在线程中的应用：通常用来处理Qtimer等event.


## 元对象系统

### 构成

- QObject为所有需要利用元对象系统的对象提供了一个基类。
- Q_OBJECT宏在类的声明体内部激活meta-object功能，如动态属性，信号和槽。
- MOC(Meta Object Complier)为每个QObject派生类生成代码以支持meta-object功能。

moc工具读取一个源文件，若他发现一个或者多个类的生命中包括Q_OBJECT宏，则会创建额外的C++源文件(moc开头的cpp源文件),其中包含了为每一个类生成的元对象代码。这些产生的文件或被include进源文件，或者与其他源文件链接。MOC的处理过程这个过程可以看作是一种“预处理”，它在标准的C++编译之前发生。

根据核心组件也可以做下列划分：
- MOC
- 信号与槽
- 动态属性：通过元对象系统，QT允许运行时动态地添加，查询，修改对象的属性。

MOC通常是在后台默默工作的，开发者不需要直接与MOC交互。MOC生成的代码与我们手写的代码一起被传递给C++编译器。这确保了Qt的元对象系统与标准C++完美地融合在一起。

### QMetaObject

每个QObject都有一个对应的QMetaObject类，构成一个平行的层次。
QMetaObject类提供了关于QObject派生类的元信息。这包括类名、父类、属性、方法等。通过这些信息，我们可以在运行时动态地与对象交互，而无需在编译时知道对象的确切类型。

```
virtual QObject::metaObject();
```
该方法返回一个QObject对应的metaObject对象。如果一个类的声明包括了Q_OBJECT宏，那么编译器就会生成代码来实现这个类对应的QMetaObject的实例引用。
如果一个类从QObject派生，但没有声明Q_OBJECT宏，那么这个类的metaobject对象不会被生成，这个类声明的signal与slot都无法使用。这个方法将返回其父类的metaobject。这样的后果就是你从这个类实例获得的元数据其实是父类的数据。

### 元数据信息

- 基本信息:

```
struct Q_CORE_EXPORT QMetaObject{
    const char *className() const;
    const QMetaObject *superClass() const;　　　　
    struct {
        // private data
        const QMetaObject *superdata; //父类QMetaObject实例的指针
        const char *stringdata;  //一段字符串内存块，包含MetaObject信息之字符串信息
        const uint *data;  //一段二级制内存块，包含MetaObject信息之二进制信息
        const void *extradata; //额外字段，暂未使用
    };
    ...
};
```

- classinfo:额外的信息，键值对形式。

```
int classInfoOffset() const;
int classInfoCount() const;
int indexOfClassInfo(const char *name) const;
QMetaClassInfo classInfo(int index) const;
```

可以使用宏来添加:

```
Q_CLASSINFO("author", "Sabrina Schweinsteiger")
Q_CLASSINFO("url", "http://ruarua")
```

- constructor：构造方法信息

```
int constructorCount() const;
int indexOfConstructor(const char *constructor) const;
QMetaMethod constructor(int index) const;
```

- enum：类声明中包含的枚举信息

```
int enumeratorOffset() const;
int enumeratorCount() const;
int indexOfEnumerator(const char *name) const;
QMetaEnum enumerator(int index) const;
```

- method：类方法信息，包括property，signal，slot

```
int methodOffset() const;
int methodCount() const;
int indexOfMethod(const char *method) const;
int indexOfSignal(const char *signal) const;
int indexOfSlot(const char *slot) const;
QMetaMethod method(int index) const;
```

- property：类型的属性信息

```
int propertyOffset() const;
int propertyCount() const;
int indexOfProperty(const char *name) const;
QMetaProperty property(int index) const;
```

要注意，对于类内定义的函数，构造函数，枚举，只有加上一些宏才表示你希望为该方法提供meta信息。
比如说Q_ENUMS用来注册宏，Q_INVACABLE用来注册方法。这样避免了meta信息的臃肿。

```
#include <QObject>  
class TestObject : public QObject  {
    Q_OBJECT  
    Q_PROPERTY(QString propertyA  READ getPropertyA WRITE getPropertyA RESET resetPropertyA DESIGNABLE true SCRIPTABLE true STORED true USER false)  
    Q_PROPERTY(QString propertyB  READ getPropertyB WRITE getPropertyB RESET resetPropertyB)  
    Q_CLASSINFO("Author", "TianSuoHaoEr")  
    Q_CLASSINFO("Version", "TestObjectV1.0")  
    Q_ENUMS(TestEnum)  
public:  
    enum TestEnum {  
        EnumValueA,  
        EnumValueB  
    };  
public:  
    TestObject();  
signals:  
    void clicked();  
    void pressed();  
public slots:  
    void onEventA(const QString &);  
    void onEventB(int);  
}
```

在这个类的内部,我们使用宏Q_OBJECT表示启用元对象系统。
此外我们通过宏标注了两个属性，两条额外信息，一个枚举类型。他们的moc文件将完整地包含这些信息。

## 信号与槽的实现

当执行connect，QT会检查发送signal的对象是否包含这个signal。
其方法是查找这个对象的class所对应的元数据对象staticMetaObject所包含的d.stringdata所指的字符串中是否包含这个signal的名字。在这个检查的过程中，需要用到d.data所指的一串整数，用来计算每个具体的字符串的起始地址。同理，用同样的方法检查是否响应对象包含相应的slot。如果这两个检查中任何一个失败了，那么connect函数就失败了，返回false。

当检查结束，就要把发送signal的对象和响应signal的对象关联起来。在QObject的私有数据类中，使用这些数据结构来保存信息：

```
class QObjectPrivate : public QObjectData {  
　　 struct Connection  {  
        QObject *receiver;  
        int method;  

        // 0 == auto, 1 == direct, 2 == queued, 4 == blocking  
        uint connectionType : 3; 
　　　　 QBasicAtomicPointer<int> argumentTypes;  
　　};  
   
   // QList
　　typedef QList<Connection>; ConnectionList;
    
    // connection存入列表
　　QObjectConnectionListVector *connectionLists;  
   
　　struct Sender{ 
　　　　QObject *sender;  
　　　　int signal;  
　　　　int ref;  
　　};

    // sender存入列表
　　QList<Sender> senders;  
};  
```

发送signal的对象中,每一个connection都会创建一个Connection对象，并把它存到Lists里去。  
发送信号的类中，Connection里保存发送信号的对象。响应signal的对象在sender中保存信号的来源。  

要注意,发送方本身维持一个ConnectionLists,每个signal对应了一个list,每个list中包含了一组connection信息.同理,接收方的每个slot对应了一个sender。
当emit signal时,每个signal都会被moc转化为一个与之相对应的成员函数。
```
void sigBtnClicked();  

void ZMytestObj::sigBtnClicked(){
    // 1,0表示第几个signal被发送出
　　QMetaObject::activate(this,&staticMetaObject,1,0);
}
```

执行activate后,从connectionLists中取出与之对应的connection信息,然后调用具体的slot.
因为connection维持了接受者的指针，当信号被activate，就遍历它所维护的所有connection，再由这个指针去执行对应的槽函数。method_index指示执行哪个slot。

## 对象树

QT使用对象树object tree来组织和管理所有QObject类和子类的对象。当创建一个QObject,如果使用了其他对象作为其父对象，那么这个QObject就会被添加到父对象的children列表中。当父对象被销毁，QObject也会被销毁。这个机制非常适合管理GUI对象。

例如，关闭一个对话框，那么对话框中的按钮和标签也被销毁。这正是我们希望的，因为按钮和标签是对话框的子部件。

### 应用

QObject是以对象树的形式组织起来的，这也是为什么你能看到parent指针。
构造QObject时parent指针默认为nullptr。你可以指定这个parent指针。
当父类对象析构，其对象链表中的所有子类对象也会自动被析构。QT保证不会有对象被delete两次。

## 反射机制

C++语言本身不支持反射机制，但QT框架在语言层面之上实现了反射机制。
所谓反射，就是是程序在运行时查询其结构和行为的能力。
在Qt中，反射是通过QMetaObject类实现的。这个类提供了关于对象的元信息，如其属性、方法和信号。可以说QT的反射机制是基于元对象系统的。
例如，我们可以使用QMetaObject来查询对象的所有属性，或者动态地调用一个方法。这种能力使得Qt可以实现一些高级功能，如属性动画和脚本绑定。

任何具备反射能力的语言或者框架，在运行时并不能凭空获取类型信息，更不能凭空创建对象，调用方法。必然是在编译该类型时就产生了对应类型的信息并保存在了可执行文件中，并支持日后载入内存。

QT框架基于QT元对象系统实现了反射机制，QT的反射所需的类型信息均是由QT源对象系统提供。实际上类型信息页存储在元对象系统中，即所谓的元对象数据meta-object-data.这些数据由元对象编译器MOC,meta-object-complier在编译时期产生。

### 应用

在一般的应用程序中，几乎任何语言的反射机制都没有什么实际价值，因为大部分普通应用直接遵循C++语法规则来创建对象和调用方法就可以了。而对于框架开发者，反射机制的价值得以体现。

比如，想实现在XML配置文件中配置类型名称与属性名称，然后在程序运行时软件框架自动化地创建出所配置的类型名称对应的类的对象实例，还要调用对象上的方法，读写属性。

对于框架而言，在他编译时并不确定用户会创建哪一个具体类的对象实例，也无从实例化，只有反射机制才能实现这种功能。

### Q_PROPERTY宏

Q_PROPERTY使属性能够在运行时通过QT的元对象系统进行访问和修改。

```
Q_PROPERTY(Type name READ getter WRITE setter NOTIFY signal)
```

- Type：数据类型
- name：属性名称
- READ getter：读取属性值的函数
- WRITE setter：可选，设置属性值的函数
- NOTIFY signal：可选，当属性值改变时发出的信号

```
class MyObject : public QObject {
    Q_OBJECT
    Q_PROPERTY(int myProperty READ getMyProperty WRITE setMyProperty NOTIFY myPropertyChanged)

public:
    int getMyProperty() const {
        return m_myProperty;
    }
    void setMyProperty(int value) {
        m_myProperty = value;
        emit myPropertyChanged(value);
    }

signals:
    void myPropertyChanged(int newValue);

private:
    int m_myProperty;
};
```

在这个例子中，int类型的属性m_myProperty可以被通过get,set方法读取和设置，并在值改变时发出信号。通过Q_PROPERTY宏，我们得到了一个强大灵活的方式处理类的属性。

```
MyObject obj;
obj.setMyProperty(10); // 使用 setter 方法设置属性
int value = obj.getMyProperty(); // 使用 getter 方法获取属性
obj.setProperty("myProperty", 20); // 使用 setProperty 设置属性
value = obj.property("myProperty").toInt(); // 使用 property 获取属性
```

一旦属性被声明，就可以在外部用标准的QT方法访问和修改这些属性。