# 细致分析`std::move`的原理

`std::move`用于高效且安全地转移资源。它将一个对象转移为**将亡右值**，随后，接受右值作为参数的构造函数将把此对象的内容和所有权移交。`std::move`的原型为：
```c++
template< class T >
typename std::remove_reference<T>::type&& move( T&& t ) noexcept;
```
其中`std::remove_reference`是一种类型衰减（decay）模板，给入`T&`或`T&&`类型，其可以给出原始类型`T`。

下面给出一个具体的例子：
```c++
std::vector <int> old_vec = {1,2,3,4,5};
std::vector <int> new_vec = std::move(old_vec);
```

---

### 解读标准库`std::move`
`std::move`方法定义于`move.h`，其实现方法相当简单：
```c++
template <class _Tp>
_LIBCPP_NODISCARD_EXT inline _LIBCPP_HIDE_FROM_ABI _LIBCPP_CONSTEXPR __libcpp_remove_reference_t<_Tp>&&
move(_LIBCPP_LIFETIMEBOUND _Tp&& __t) _NOEXCEPT {
  typedef _LIBCPP_NODEBUG __libcpp_remove_reference_t<_Tp> _Up;
  return static_cast<_Up&&>(__t);
}
```

此外，看看`__libcpp_remove_reference_t`（其实也就是`std::remove_reference`）是如何实现的也十分值得，因为这其中蕴含了C++模版类的精华：
```c++
template <class _Tp> struct _LIBCPP_TEMPLATE_VIS remove_reference        
    {typedef _LIBCPP_NODEBUG _Tp type;};
template <class _Tp> struct _LIBCPP_TEMPLATE_VIS remove_reference<_Tp&>  
    {typedef _LIBCPP_NODEBUG _Tp type;};
template <class _Tp> struct _LIBCPP_TEMPLATE_VIS remove_reference<_Tp&&> 
    {typedef _LIBCPP_NODEBUG _Tp type;};
```
所以，我们用`std::remove_reference<std::vector&>::type`就可以得到一个`std::vector`类型的容器。显然在这里这显得有些繁琐，不过`std::remove_reference`这种语法的通用性是相当高的。

泛型模版`<class _Tp>`的`_Tp`为什么不能是引用？换句话说，为什么`_Tp&`才一定是一种引用？**这是因为，在C++模版化编程中，`template`指定的模版必须是基本类型，即基本数据类型、类类型和指针类型。如果`_Tp`是一个引用类型，编译器会自动去引用。**

---

### 以`std::vector`为例，为什么右值引用可以更完美地转移数据？
`std::vector`的全部内容存储于`vector`头，我们在其中可以找到`vector(vector&& __x)`构造函数。

这个函数的核心是这样实现的：
```c++
this->__begin_ = __x.__begin_;
this->__end_ = __x.__end_;
this->__end_cap() = __x.__end_cap();
__x.__begin_ = __x.__end_ = __x.__end_cap() = nullptr;
```
不难看出，`__begin`、`__end`和`__end_cap()`是`std::vector`类的私有成员。由于`std::vector`内部是顺序存储的，所以只需要保留首尾指针。此外，最后一行的赋值操作展现了右值将亡值的「死亡」过程。此构造函数将使`__x`彻底失效。

---

### 以`std::map`为例，展示复杂的数据结构如何转移所有权？
`std::map`的实现方法是红黑树，因此其数据的转移依赖于树的转移：
```c++
_LIBCPP_INLINE_VISIBILITY
map(map&& __m)
    _NOEXCEPT_(is_nothrow_move_constructible<__base>::value)
    : __tree_(_VSTD::move(__m.__tree_))
    {
    }

```
下面我们来考虑`__tree_`成员如何实现所有权的转移？

在`std::map`类的私有成员中，有两行代码定义了`__tree_`：
```c++
typedef __tree<__value_type, __vc, __allocator_type>   __base;
__base __tree_;
```

我们可以从`__tree`头中找到`__tree`的构造函数。其中的一个构造函数`__tree(__tree&& __t)`正是我们所需要的，其具体实现细节如下：
```c++
template <class _Tp, class _Compare, class _Allocator>
__tree<_Tp, _Compare, _Allocator>::__tree(__tree&& __t)
    _NOEXCEPT_(
        is_nothrow_move_constructible<__node_allocator>::value &&
        is_nothrow_move_constructible<value_compare>::value)
    : __begin_node_(_VSTD::move(__t.__begin_node_)),
      __pair1_(_VSTD::move(__t.__pair1_)),
      __pair3_(_VSTD::move(__t.__pair3_))
{
    if (size() == 0)
        __begin_node() = __end_node();
    else
    {
        __end_node()->__left_->__parent_ = static_cast<__parent_pointer>(__end_node());
        __t.__begin_node() = __t.__end_node();
        __t.__end_node()->__left_ = nullptr;
        __t.size() = 0;
    }
}
```
我们看到，这其中涉及了树结构的转移和原树的销毁。我们将不再做进一步的探讨。