---
title: 关于std::thread初始化的一点问题
date: 2021-08-28 19:19:44
tags:
  - C++
  - multi-thread
---
## 问题代码
``` c++
class Message{
public:
    std::string s;
    Message(const std::string&s = ""):s(s){}
};

class MessageQueue{
    std::queue<Message> qu;
    std::mutex mutex;
    std::condition_variable condition;
    size_t cap;
public:
    MessageQueue(size_t cap=100);
    void produce(Message message);
    Message consume();
};

MessageQueue mq;
void produce(MessageQueue mq){
    //code
}
Message consume(MessageQueue mq){
    //code
}
int main()
{
    auto t1 = std::thread(produce, mq);
    auto t2 = std::thread(consume, mq);
    t1.join();
    t2.join();
    return 0;
}
```
## 错误信息
``` c++
/usr/include/c++/8/thread:120:17: error: static assertion failed: 
std::thread arguments must be invocable after conversion to rvalues
```

## 问题分析

首先定位`/usr/include/c++/8/thread`第120行左右

   ``` c++
       template<typename _Callable, typename... _Args,
   	     typename = _Require<__not_same<_Callable>>>
         explicit
         thread(_Callable&& __f, _Args&&... __args)
         {
   	static_assert( __is_invocable<typename decay<_Callable>::type,
   				      typename decay<_Args>::type...>::value,
   	  "std::thread arguments must be invocable after conversion to rvalues"
   	  );
   ```
   decay有很多功能。在这里可以认为将函数引用转为函数指针，将参数的引用类型去掉引用（也就是这里英文说的转为右值）。而`__is_invocable`是判断转换之后的类型是否适配，是否可以使用对应的实参完成到形参的转换。我们发现这里转换以后的形参和实参都是`MessageQueue`，看上去似乎没有问题。但是实际上问题恰好就在这里，因为形参不是引用类型，不管实参是什么类型，都会需要**调用相应的拷贝构造函数**。而由于`MessageQueue`里有成员`std::mutex`和`std::condition_variable`，其拷贝构造函数都被`delete`，所以`MessageQueue`的默认的拷贝构造函数无法被调用，因而断言失败。

## 解决方案

### 解决问题

实际上，从设计上讲，本来就完全不应当让`MessageQueue`调用拷贝构造函数，完全是多余的开销。所以不妨进行像下面这样的修改：

```  c++
MessageQueue mq;
void produce(MessageQueue& mq);
Message consume(MessageQueue& mq);

int main(){
    auto t1 = std::thread(produce, std::ref(mq));
    auto t2 = std::thread(consume, std::ref(mq));
    t1.join();
    t2.join();
    return 0;
}
```

首先显然是直接将`produce`和`consume`的形参改为引用。但仅仅这样的修改是无法通过` __is_invocable`的判断的。因为左值引用无法绑定右值。所以需要使用`std::ref`来做一个修饰，我们可以看源码：

``` c++
  template<typename _Tp>
    inline reference_wrapper<_Tp>
    ref(_Tp& __t) noexcept
    { return reference_wrapper<_Tp>(__t); }
//......
  template<typename _Tp>
    class reference_wrapper
    : public _Reference_wrapper_base<typename remove_cv<_Tp>::type>
    {
      _Tp* _M_data;
    public:
      typedef _Tp type;
      reference_wrapper(_Tp& __indata) noexcept
      : _M_data(std::__addressof(__indata))
      { }
//......
      operator _Tp&() const noexcept
      { return this->get(); }
      _Tp&
      get() const noexcept
      { return *_M_data; }
//......

```

我们发现`ref`只是返回了一个模板类`reference_wrapper<_Tp>`，而这个模板类有点类似引用的底层实现，拥有数据实际存储的指针，**特别值得注意的是**，它实现了对`_Tp&`的重载，使得它可以转换为引用类型，从而绕过了`decay`的影响。

### 另外的办法？

既然形参变成左值引用可以，那右值引用是否也可以呢？大概代码如下：

```c++
MessageQueue mq;
void produce(MessageQueue&& mq);
Message consume(MessageQueue&& mq);

int main(){
    auto t1 = std::thread(produce, mq);
    auto t2 = std::thread(consume, mq);
    t1.join();
    t2.join();
    return 0;
}
```

实际上看起来是比较奇怪的。因为右值引用只能绑定右值，但`mq`并不是右值，这样不应该能够运行。但是实际上真正运行起来，是会出现编译错，但是原因并不是这个，是因为需要获得一个包含函数指针以及参数列表的`std::tuple`实例化对象，所以所有参数都应当有拷贝构造或者移动构造，但这里全被`delete`了。这里仅仅贴一下代码，不进行展开讨论：

```c++
    template<typename _Callable, typename... _Args>
      static _Invoker<__decayed_tuple<_Callable, _Args...>>
      __make_invoker(_Callable&& __callable, _Args&&... __args)
      {
	return { __decayed_tuple<_Callable, _Args...>{
	    std::forward<_Callable>(__callable), std::forward<_Args>(__args)...
	} };
      }
```

## 总结

对于`std::thread`的第一个参数，也就是线程运行的函数本身，它的参数一般情况下没有必要使用右值引用。如果参数是原始类型，应当保证拷贝构造函数或者移动构造函数存在（比较常识，但容易忘掉）。如果是左值引用类型，记得对形参加上`std::ref`即可。

## 其他的收获

在形参与实参传递的过程中，T,T&,T&&类型的功能是不同的（注意，此处的T类型仅指**原始类型**）。

|      | 形参                 | 实参         |
| ---- | -------------------- | ------------ |
| T    | 一定会调用构造函数   | 作为**右值** |
| T&   | 绑定左值引用，无构造 | 作为左值     |
| T&&  | 绑定右值引用，无构造 | 作为**右值** |

其实与普通的定义语句语义是一样的，但我经常搞混，甚至现在才明白T可以被认为是右值。

## 参考

[C++：std::thread arguments must be invocable after conversion to rvalues](https://blog.csdn.net/qq_45858169/article/details/114642760)

[利用std::is_invocable在编译期间判断函数与传递参数是否匹配](https://blog.csdn.net/wangzhicheng1983/article/details/118691854)

