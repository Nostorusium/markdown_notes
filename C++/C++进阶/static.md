# STATIC

所谓static对象,他的寿命从构造出来一直到程序结束为止。
定义在函数内的static对象被称为local static对象,因为他们对于函数而言是local.
同时,static对象不能跨越翻译单元.

## 类内static与单例模式

EffectiveC++条款04指出,你可以不给对象赋值,但一定要初始化.

考虑一个场景,翻译单元1的static对象使用翻译单元2的static对象,而此时翻译单元2的对象还没有初始化.
此时就会产生灾难性的后果,因为编译器不能看穿翻译单元.这两个对象的初始化有先后顺序.

一个很好的设计是把这些外部的static对象挪到函数内部,作为一个local static对象.
这个手法的基础是,C++能保证local static对象一定能在函数内部首次遇到对象定义式时被初始化.
这样我们能保证获得一个被初始化过的static对象.

单例模式有两种流派，分为饿汉和懒汉模式，其区别在于实例化instance的时机。
所谓懒汉模式,就是先初始化这个instance.
所谓饿汉模式,就是直到执行了getInstance才初始化.

以下为懒汉模式
```
class Game{
private:
    Game(){...}
    Game(Game &&) = delete;

    // 必须类外初始化
    static Game instance;

public:
    static Game& getInstance(){
        return instance;
    }
};

// 在类外初始化静态成员
Game Game::instance = new Game();
```

以下为饿汉模式
```
class Game{
public:
    static Game& getInstance(){
        if(instance == nullptr){
            instance = new Game();
        }
        return instance;
    }
private:
    static Game instance;
    Game(){...}
    Game(const Game&) = delete;
};

//初始化
Game Game::instance = nullptr;
```

上面两个例子的instance都是一个non local static对象,都不能避免这种先后初始化顺序的问题.
因为他们都需要在类外先进行static的初始化.
下面是改进方案,利用local static能自动初始化的特性

```
class Game{
private:
    Game(){...}
    Game(const Game&) = delete;

public:
    Game& getInstance(){
        static Game instance;
        return instance;
    }
}
```

这样,当程序第一次执行getInstance,类的static instance就会被初始化,我们无需在类外额外初始化.