# std::thread与pthread

>https://www.cnblogs.com/vivian187/p/15958851.html
>https://www.cnblogs.com/wangzming/p/7845589.html
>选谁？https://www.jianshu.com/p/1ce08bb031f

## 对比

std::thread由C++11提供，头文件<thread>
而pthread只支持Linux系统，头文件<pthread.h>
pthread源自一个C接口，不提供任何RAII，也更难用，更易出错。

std::thread相对简单易用，而且跨平台。pthread只支持POSIX系统。
并且提供了更多高级功能，比如future，也更加C++。
而操作线程和mutex的api较少。