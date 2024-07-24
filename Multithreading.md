# C++ Multithreading 多线程剖析

1. 使用 `<thread>` 头引入多线程类（C++11及以后）。
2. 使用 `std::thread t(F&& f, Args&&... args)` 创建子线程 `t`。其中 `f` 是函数指针，`args` 是函数所需的参数列表。
3. 使用 `t.join()` 来等待子线程 `t` 完成。
4. 使用 `t.detach()` 来分离子线程 `t` 。

一个简单的例子是：
```c++
#include <iostream>
#include <thread>

void print_message(std::string msg) {
    std::cout << msg << std::endl;
}

int main() {
    std::thread t(print_message, "Hello World!");  // 创建并启动线程
    t.join();  // 等待线程 t 完成
    return 0;
}
```

---

### 解读标准库代码

`std::thread` 构造函数定义于 `thread.h`，其实现细节如下：
```c++
thread::thread(_Fp&& __f, _Args&&... __args)
{
    typedef unique_ptr<__thread_struct> _TSPtr;
    _TSPtr __tsp(new __thread_struct);
    typedef tuple<_TSPtr, __decay_t<_Fp>, __decay_t<_Args>...> _Gp;
    unique_ptr<_Gp> __p(
            new _Gp(_VSTD::move(__tsp),
                    _VSTD::forward<_Fp>(__f),
                    _VSTD::forward<_Args>(__args)...));
    int __ec = _VSTD::__libcpp_thread_create(&__t_, &__thread_proxy<_Gp>, __p.get());
    if (__ec == 0)
        __p.release();
    else
        __throw_system_error(__ec, "thread constructor failed");
}
```

这里有一些理解的要点：
1. `__thread_struct` 是一个辅助性质的类，而不是一个线程；
2. `_Gp` 包含了期望的线程的全部内容：函数指针（`__f`）和参数列表（`__args`）；
3. `__libcpp_thread_t __t_` 是 `thread` 类的成员，其实际类型就是 `pthread_t`，用于标识一个线程；
4. `__libcpp_thread_create()` 用于真正地创建线程。此方法定义于`__threading_support.h`。其中，`__t_` 是线程的标识符；`&__thread_proxy<_Gp>` 传递一个线程的入口函数，`__p.get()` 从智能指针中获取 `_Gp` 指针给线程入口函数；
5. **此构造函数包装了 `__f` 及其参数 `__args`，实际的系统调用并未使用 `__f` 和 `__args`，而是使用了 `&__thread_proxy<_Gp>` 和 `__p.get()`。** 这更是一种抽象，方便了 `<thread>` 库中其他方法的实现，也让 `<thread>` 的使用者以更直观的方法和更统一的风格来创建和管理线程。

值得注意的是，用 `std::move` 处理 `std::unique_ptr` 可以安全的转移智能指针的所有权，避免潜在的漏洞。

---

下面介绍 `__libcpp_thread_create()` 方法。这是一个封装方法，定义于 `__threading_support`：

```c++
int __libcpp_thread_create(__libcpp_thread_t *__t, void *(*__func)(void *),
                           void *__arg)
{
  return pthread_create(__t, nullptr, __func, __arg);
}
```

其中 `pthread_create` 方法是 `POSIX` 线程库的一部分，已经涉及系统调用，因此将不做过多讨论。其定义于 `pthread.h`，范式为：

```c++
int pthread_create(pthread_t _Nullable * _Nonnull __restrict,
		const pthread_attr_t * _Nullable __restrict,
		void * _Nullable (* _Nonnull)(void * _Nullable),
		void * _Nullable __restrict);
```

---

### 总结

C++11的 `<thread>` 库为使用C++风格管理多线程创造了便捷的环境。我们看到C++ STL提供了足够的封装，使得程序开发者不需要完全了解 `POSIX` 就可以开发多线程程序。

同时，在 `<thread>` 库中，我们看到 `std::unique_ptr` 被用于安全的管理内存，`std::tuple` 被用于灵活地绑定数据，而 `std::move`、`std::forward` 和 `std::decay_t` 尽最大努力防止数据类型在函数传递中出现不可预知的改变。这些STL类为开发稳定的程序提供了不少帮助。
