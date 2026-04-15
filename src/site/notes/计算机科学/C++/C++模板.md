---
{"dg-publish":true,"permalink":"/计算机科学/C++/C++模板/","tags":["Template"]}
---

# 术语约定

为不至于概念的混淆，本文先对各种术语进行辨析声明。

+ **模板中的参数**、**模板参数**、**模板实参**、**函数参数**、**函数实参**

  模板中的参数={模板参数、函数参数、模板实参、函数实参}

  ```C++
  template<typename 模板参数T>
  void func(T 函数参数t){}
  int i;
  func<模板实参int>(函数实参i);
  ```

+ **参数包**、**模板参数包**、**函数参数包**

  + 参数包：包括模板参数包、函数参数包；
  + 模板参数包：可变数目的模板参数；
  + 函数参数包：可变数目的函数参数；
  
  ```c++
  // Args:模板参数包 rest:函数参数包
  template<typename ...Args>
  void func(Args ...rest);
  ```
  
  
  
+ **具体化**、**实例化**

  + 具体化：为模板参数和函数参数传递实参的过程；
  + 实例化：具体化后从模板生成实例代码的过程；

  

# 模板实例化

## 隐式实例化

当调用（使用）一个函数（类）模板时，如果这种实例还不存在，编译器会生成相应的函数或类代码，这样的方式称为**隐式实例化**。

编译器通常可以根据函数实参来推断模板实参，即所谓的**隐式具体化**；我们也可以指定模板实参（类模板必须人为指定实参类型），即**显式具体化**。

```C++
template<typename T>
void func(T t){}
func(1); // 隐式实例化+隐式具体化:void func<int>(int t)
func<double>(1); // 隐式实例化+显式具体化:void func<double>(double t) 

template<typename T>
class A{};
A<int> a; // 使用类模板必须显式具体化
```

> + 隐式实例化直到使用模板时才会生成相应的函数或类代码，这决定了在错误检查上会有部分“延迟”；
> + 模板的每个实例都形成独立的函数或类，它们之间没有任何的关联。
> + 对于一个隐式实例化的模板类，其成员函数只有在使用时才被实例化。



## 显式实例化

不同于隐式实例化，显式实例化在实例定义时（还未被使用）就生成相应代码，不同文件可以通过`extern`关键字或者直接包含头文件的方式共享一份实例代码（在隐式实例中每个文件都有自己的一份实例），从而减少开销，因此这种方法适合于构建大型的模板库。

```C++
// template.h
template<typename T>
void func(T t){}
template void func(int t); // 定义实例

template<typename T>
class A{};
template class A<int>; // 定义实例

// test1.cpp
#include<template.h>
void test1()
{
    func<int>(0); // 实例化出现在template.h中
    A<int> a;
}

// test2.cpp
#include<template.h>
void test2()
{
    func<int>(0); // 实例化出现在template.h中
    A<int> a;
}
```





# 模板类型别名

我们可以用`typedef`为类模板的一个实例定义类型别名；但是不能用它定义类模板本身的别名。

新标准允许我们使用`using`关键字来为类模板定义别名。

```c++
template<typename T>
class A
{
    typedef A<int> a_int1; // 正确
    typedef A<T> a_t1; // 错误
    using a_int2 = A<int>; // 正确 
    using a_t2 = A<T>; // 正确
};
```



# 模板尾置返回

对于模板函数的返回类型，最简单的做法就是利用模板参数，让用户确定；但在某些情况下，这样做可能会让用户感到“恼火”。

```c++
template<typename It, typename Type>
Type func(It beg, It end)
{
    return *beg;
}
vector<int> v={1,2};
func<vector<int>::iterator, int>(v.begin(), v.end()); // 用户不得不显式指定模板实参
```

改进的一个想法是通过`decltype`关键字来获取`*beg`的元素类型，然后作为函数返回类型。但是，在编译器遇到函数的参数列表之前，beg是不存在的，所以前置返回的方案不再适合，而需要采用**尾置返回**。

```c++
template<typename It>
auto func(It beg, It end)->decltype(*beg)
{
    return *beg;
}
vector<int> v={1,2};
func(v.begin(), v.end()); // 用户表示很舒服，此时返回类型为int&
```

问题还没完，在某些情况下，返回元素的引用可能是不合适的，我们可以用`remove_reference`从引用类型获得元素类型（通过type成员获取）。

```c++
template<typename It>
auto func(It beg, It end)->
    typename remove_reference<decltype(*beg)>::type 
{
    return *beg;
}
vector<int> v={1,2};
func(v.begin(), v.end()); // 返回类型:int
```





# 成员模板

普通类其成员函数可以是函数模板，类模板的成员函数也可以是函数模板。它们的定义和实例方式如下：

```c++
class A
{
public:
    template<typename T>// 普通类成员函数模板
    void func(T t){}
};
A a;
a.func<double>(0); // 实例化
```

```c++
template<typename T1>
class B
{
public:
    template<typename T2>// 模板类成员函数模板
    T2 func(T1 t);
};
template<typename T1> // 类的类型参数
template<typename T2> // 成员函数的类型参数
T2 B<T1>::func(T1 t)
{
    return t;
}
B<double> b;
b.func<int>(0); // 实例化
```



# 模板中的参数



## 模板参数类型

模板参数可以是**类型参数**或者**非类型参数**，前者可以看作类型说明符，后者则表示一个值。

```c++
template<typename T, int N> // 类型参数T, 非类型参数N
void func(T t[N]){}
func1<const char, 4>("abc");// 实例化
```



## 参数推断

在模板具体化时，如果采用隐式的方式，编译器会自动推断模板参数和函数参数的类型。特别是在右值引用中，我们将看到C++中“引用折叠”以及转发的机制。



### 隐式类型转换

调用模板传递函数实参时，编译器通常不是对函数实参进行类型转换，而是生成一个新的模板实例，少数能够自动应用类型转换的只有：

+ const转换：
  + 顶层const无论在形参中还是在实参中，都会被忽略；
  + 可以将一个非const对象传递给一个const的引用（或指针）形参；

```C++
template<typename T> void func1(T);
template<typename T> void func2(const T);
template<typename T> void func3(const T&);
string s1("abc");
const string s2("abc");
func1(s2); // 忽略实参顶层const, void func1<std::string>(std::string)
func2(s1); // 忽略形参顶层const, void func2<std::string>(std::string)
func2(s2); // void func2<std::string>(std::string)
func3(s1); // 非顶层const不忽略，将s1转化为const void func3<std::string>(const std::string &)
```

+ 数组或函数指针转换；

```c++
template<typename T> void func(T, T);
int a[10], b[10];
func(a, b); // void func<int *>(int *, int *)
```



### 左值引用

当模板函数参数为左值引用时，传递的函数实参必须为左值；如果函数参数类型为const型的左值引用，则可以传递给它任何类型的实参，但是函数内部无法修改它。

```C++
template<typename T> void func1(T&);
template<typename T> void func2(const T&);
int a = 0;
const int b = 0;
func1(a); // void func1<int>(int &)
func1(b); // void func1<const int>(const int &)
func1(0); // 错误
func2(a); // void func2<int>(const int &)
func2(b); // void func2<int>(const int &)
func2(0); // void func2<int>(const int &)
```



### 右值引用

我们知道右值引用不能绑定到一个左值，除非使用`std::move`函数。

```C++
int c = 0;
int &&d = c; // 错误
int &&d = std::move(c); //正确
```

> 右值引用本身不一定是右值的，例如上面的引用d就是左值；此外函数参数也都是左值的，即使参数为右值引用。

但是在模板函数参数中，有了例外的规定：

+ 如果传入的参数是**左值**（包括有名字的变量、引用变量等），则 `T` 被推导为该左值类型的**左值引用**，经过引用折叠后 `T&&` 变为左值引用。我们称该机制为“**引用折叠**”。
+ 如果传入的参数是**右值**（如字面量、临时对象等），则 `T` 被推导为该右值类型的**非引用类型**，`T&&` 保持为右值引用。

```C++
template<typename T> void func3(T&&);
int a = 0;
const int b = 0;
int &&d = 0;
int &e = a;
func3(0); // 0属于右值，推导T->int，绑定到右值，void func3<int>(int &&)
func3(d); // d是一个左值（如上文所说），推导T->int&（引用折叠），void func3<int&>(int &)
func3(e); // 绑定到左值，void func3<int&>(int &)
func3(a); // 绑定到左值，void func3<int&>(int &)
func3(b); // 绑定到左值，void func3<const int &>(const int &)
```



上述规则给函数内部代码带来了不小的麻烦：如果模板函数参数是右值引用，实例化之前对函数内部来说，模板参数是引用类型还是非引用类型是无法确定的！

```C++
template<typename T>
void func(T&& val)
{
    T t = val; // 拷贝还是引用
    t = 0; // 只改变t还是既改变t又改变val
    if (t == val){} // 会相等吗
}
```

上面的模板，传入左值实参和右值实参将得到完全不同的行为。可见编写这样的函数代码变得异常困难。

在实际中，右值引用通常用于两种情况：模板转发其实参、模板重载。



### std::move原理

在我们对模板右值引用有所了解之后，不妨来探察一下`std::move`的实现原理。

通过`std::move`我们可以获得一个绑定在左值上的右值引用。由于它本质上能接受任何类型的实参，因此我们不会惊讶于它是一个函数模板。它在标准库中的定义如下，

```C++
template<typename _Tp>
constexpr typename std::remove_reference<_Tp>::type&& move(_Tp&& __t) noexcept
{ 
    return static_cast<typename std::remove_reference<_Tp>::type&&>(__t); 
}
```

其中`constexpr`和`noexcept`都是优化编译的关键字，前者表示编译时可计算，后者承诺函数不会发生异常，运行时不需要异常检查。

`std::move`其实借助了`static_cast`完成了引用的转化（左值引用转右值引用）。没错，事实上我们在任何需要引用转化的情况下都可以用`static_cast`，如果你能够忍受敲类型名称的话。为了能够泛型化，`move`对其进行了包装，唯一需要考虑的问题就是如何确定强转的目标类型。

采用了右值引用的函数参数允许`move`可以接受任意类型的参数，如此我们既可以对右值调用`move`，也可以对左值调用，产生的结果就是函数参数可能被实例化为左值引用也有可能是右值引用——但无论哪种情况，元素的类型总是不变的，可以通过`remove_reference<_Tp>::type`获得，一旦该信息已知，转换的目标类型也就可以确定下来了。



### 转发

现在有一个实现的函数和一个待改进的模板如下，

```c++
void func1(int& a, int&& b){a=b;}
template<typename T1, typename T2>
void func2(T1 t1, T2 t2)
{
    func1(t1, t2);
}
int x = 0, y = 1;
func2(x, y);
```

我们的任务是改进这个模板使调用`func2(x, y)`可以通过编译并且使x等于y。



函数参数的一个重要特点是**左值性**，模板函数也逃不出这个限制。



# 变参模板

## 参数包扩展

我们通过在参数右边放置`...`来扩展参数包，扩展就是将它分解为多个参数，对每个参数应用模式，获得扩展后的列表。为充分说明这个机制，我们编写一个支持多参数打印的函数。

在这份代码中，`const Args&...`是一个扩展，`rest...`也是一个扩展。大多数变参函数模板是递归运行的，我们的打印函数亦是：

```c++
template<typename T>
void print(const T& t) // 版本一
{
    cout << t << endl;
}

template<typename T, typename... Args>
void print(const T& first, const Args&... rest) // 版本二
{
    cout << first << ", ";
    print(rest...);
}
print(0, 0.0, 'c', "s");
```

+ 第一次调用版本二的`print`，通过扩展Args，函数参数列表实例化为：

```
const int &first, const double &rest, const char &rest, const char (&rest)[2]
```

+ 第二次调用版本二的`print`，我们传入了`rest...`，其中第一个参数会被绑定到`first`形参上，即变参部分参数数量减少了，此时函数参数列表实例化为：

```
const double &first, const char &rest, const char (&rest)[2]
```

+ 最后一次调用的是版本一函数，我们需要使用该版本终止递归。



## 参数包大小

我们可以通过`sizeof`获取参数包大小（参数数量）。

```c++
template<typename... Args>
void func(Args... args)
{
    cout << sizeof...(Args) << endl;
    cout << sizeof...(args) << endl;
}
func(1, 2, 3);
```



## 参数包转发





