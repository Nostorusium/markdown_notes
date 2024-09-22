# bind&function

std::bind可以将可调用对象与其参数一起绑定，绑定的结果可用std::function进行保存

## std::function

function是一个通用函数包装器，用来封装可调用的对象，如普通函数，lambda，函数对象，成员函数等等。他的核心是让你存储和使用不同类型的函数，而不关心他们的实际类型。

```
void myFunction() {
    std::cout << "This is a regular function!" << std::endl;
}
int main() {
    std::function<void()> func = myFunction;
    func();
    return 0;
}
```

std::function提供了类型擦除功能，它允许你将不同类型的可调用对象统一处理。
类型擦除隐藏了具体类型的细节，而只暴露他的调用接口(接受参数，返回类型)，不管你传入lambda，还是普通函数，函数对象，bind都可以通过function统一处理。

## std::bind

bind用来将函数和参数绑定在一起，生成一个新的可调用对象。这个对象可以被延迟执行，相当于存储起来了。
std::bind经常和std::function搭配使用。

```
void printMessage(const std::string& message) {
    std::cout << message << std::endl;
}
int main(){
    std::function<void()> func = std::bind(&MyClass::printMessage, &obj, "Hello, World!");
    return 0;
}

```

## bind与function的结合

我们通常使用function来抽象出一个可以执行的对象，比如各种函数。

```
std::queue<std::function<void()>> tasks;
```

假如我们有一个线程池维护了一个任务队列，task被抽象成一个可以执行的对象，比如一个函数。
这里我们使用了void()，表示无返回值，无参数值。
但如果我们想要传递一个有参数的函数，就不符合这个要求了。于是这个时候bind排上了用场。

```
threadpool_->AddTask(std::bind(&WebServer::OnRead_, this, client));
// WebServer::OnRead_是返回void的
```

你可以使用bind来绑定你想要传入的函数参数，它们共同组成了一个新的执行对象，此时他只是一个返回void可调用对象，并没有参数。

通过function和bind的简单结合，你可以无需function所管理的函数对象所接受的参数不同。
再得益于function的类型擦除功能，它支持接受各类可调用对象，它们就抽象出了一个非常通用的概念：返回特定类型的任意可调用对象，这提供了极大地灵活性。你可以将任意类型，任意参数的函数，lambda，仿函数等一律抽象为通用的function。