# constexpr

注意到constexpr是 const expression,常量表达式的缩写.
其含义是值不会改变而且编译期间能得到结果的表达式.

## 使用

```
int array[sum(1,2)]; //error
```

由于array的大小必须是常量,这样的返回值会报错.
因为编译器不能保证这个sum(1,2)能返回一个常量.

```
constexpr int add(const int& a, const int&b){
    return a+b;
}
```

此时可以使用constexpr修饰.这意味着它将返回一个编译期就能确定的值
编译器会把constexpr视为内联函数,若能在编译期求出值,则用结果值替换掉函数调用

## 约束

constexpr是一种很强的约束,保证程序的确定语义不会破坏.
其所引用的对象必须在编译期就决定地址.