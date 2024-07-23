# `{}`和`std::initializer_list`

在C++中，编译器将会根据不同的情况将`{}`转换为不同类型。例如，

**1. `{}`可以被解读为初始化数组的列表：**

```cpp
int array[] = {1, 2, 3, 4, 5}; 
```

**2. `{}`还可以被解读为std::initializer_list：**
```cpp
std::vector<int> vec = {1, 2, 3, 4, 5};
// {}内的数据被转换为std::initializer_list，然后调用std::vector(initializer_list<value_type> il)构造函数。
```

事实上，`{}`是C++的核心语法，而`{}`被转换为`std::initializer_list`亦是编译器所为。这一定程度上表明`std::initializer_list`已经接近C++语言的语法核心，即使它只是STL库的一部分。
