---
{"dg-publish":true,"permalink":"/计算机科学/C++/C++八股文集锦/"}
---


# C++类

## 默认编写的函数

### 构造函数

默认构造函数`Empty(){}`，因此空类可以如下实例化对象，

```
class Empty{};
Empty e;
Empty e2();
Empty *p_e = new Empty();
Empty es[10];
```

只要定义了构造函数，编译器就不会再提供默认构造函数了，但是一般最好手动再定义一个默认构造函数，以支持默认构造（如数组），

```
class Empty
{
public:
	Empty() = default;
	Empty(int z){}
};
```



### 析构函数

默认析构函数`~Empty(){}`



### 拷贝、赋值拷贝函数

默认拷贝函数为`Empty(const Empty& rhs){...}`，其支持以下拷贝操作，

```
Empty e2(e1);
```

默认赋值拷贝函数`Empty operator=(const Empty& rhs){...}`，其支持以下拷贝操作，

```
Empty e2 = e1;
e1 = e2;
```

对于拷贝和赋值拷贝，可以发现，拷贝函数调用于构造期，在构造期，成员变量得到初始化；但是赋值操作可以在初始化后使用。

**默认拷贝和赋值拷贝都是对内存建立副本，而非引用，然而指针拷贝则是指向同一块内存，一般会导致下列问题**。

```
class Holder
{
public:
    Holder(int size)         // Constructor
    {
      m_data = new int[size];
    }
    ~Holder()                // Destructor
    {
      delete[] m_data;
    }

private:
	int*   m_data;
};

Holder h1(100);
Holder h2(h1);//运行错误，多次释放内存，因为h1和h2的m_data都指向了统一块内存
```

解决的方法就是为含有指针变量的类定义拷贝函数，使它开辟一个新的内存区。

```
Holder(const Holder& other)
{
  m_data = new int[other.m_size];  // (1)
  std::copy(other.m_data, other.m_data + other.m_size, m_data);  // (2)
  m_size = other.m_size;
}
```





### 拷贝赋值中注意点

例如成员变量是引用和常量的时候，通过拷贝创建对象是允许的，但是不能在创建后再拷贝赋值其他对象。

```
class Object
{
    public:
		...
        string& _name;
        const int _value;
};
Object obj0,obj1;
Object obj2(obj0); //正确，_name和_value初始化，_name指向obj0._name
obj2 = obj1; // 错误，引用不能更改指向对象，常量被初始化后不能被更改值。
			// 等价于string& obj2._name = obj1._name
```

请注意，以下对于引用的使用是正确的`a=c`只是会改变b的内容，而不会更改a的指向。

```
string &a = b;
a = c;
```





### 如何调用

调用对象的默认函数时，默认函数又会调用成员变量的默认函数或者是为内置类型赋予初始值（**由内向外**进行初始化）；因此类定义中除了自己写的构造，都会额外定义一个默认构造函数，防止嵌套调用时，产生缺少默认构造的错误。



### 拷贝中的函数调用

```c++
class Copyable {
public:
    Copyable(){}
    Copyable(const Copyable &o) {
        cout << "Copied" << endl;
    }
};
Copyable ReturnRvalue() {
    return Copyable(); //返回一个临时对象
}
void AcceptVal(Copyable a) {

}
void AcceptRef(const Copyable& a) {

}

int main() {
    cout << "pass by value: " << endl;
    AcceptVal(ReturnRvalue()); // 应该调用两次拷贝构造函数
    cout << "pass by reference: " << endl;
    AcceptRef(ReturnRvalue()); //应该只调用一次拷贝构造函数
}
```

上面代码中，执行`AcceptVal(ReturnRvalue())`，在没有编译器优化的情况下，会执行一次构造函数、两次拷贝构造函数。`Copyable()`进行了构造，在函数返回时会通过拷贝构造生成一个临时对象，这个临时对象又传入到`AcceptVal`函数中，通过拷贝构造赋给形参a。







## 多态

### 虚函数

用virtual定义，**定义一个函数为虚函数，不代表函数为不被实现的函数，基类必须提供虚函数的实现，派生类可以缺省实现；定义虚函数的目的在于允许用基类的指针来调用子类这个函数（多态，见《重写》小节）**。

```
virtual void goToDestination(){...} //A、B类都有实现
A *a = new B();
a->goToDestination(); //实际调用B的实现
```



### 纯虚函数

**定义纯虚函数是为了实现一个接口**，规定继承这个类的程序员必须实现这个函数。**含有纯虚函数的类叫做抽象类，抽象类不能被实例化**

```
virtual void chirp() = 0;
```

**对比**：虚函数和纯虚函数都可以支持多态，但是很多情况下，基类本身生成对象是不合情理的，因此仅仅声明虚函数就不够了，必须通过纯虚函数进行规范。



### 抽象类

定义：带有纯虚函数的类为抽象类。

+ 抽象类只能作为基类来使用，其纯虚函数的实现由派生类给出
+ 如果派生类中没有重新定义纯虚函数，而只是继承基类的纯虚函数，则这个派生类仍然还是一个抽象类
+ 抽象类不能定义对象



### 多态的原理（虚函数表）

多态：所谓多态多态，就是通过基类指针或引用调用一个虚函数时有很多个形态，编译器并不确定到底调用的是基类还是某个派生类的函数，直到运行时才被确定。即**动态绑定**。

这种动态绑定通过虚函数表实现：每一个有虚函数的类所实例化的对象在创建时都会建立一个虚函数表，并在对象中以指针指向这个表（这个对象内存占用将多出4个字节（32位系统），用于存放这个指针）。在对象被创建时，虚函数表中的函数都映射为自身的虚函数实现，在之后的程序过程中通过`p_obj=new A();`等方式映射到派生类实现中，在析构时又映射回自身的虚函数实现。

**虚函数表正是能够形成多态的原因**。

虚函数表是编译器生成的，程序运行时被载入内存。如下代码

```
class A{
public:
    int i;
    virtual void func() {}
    virtual void func2() {}
};
class B : public A
{
    int j;
    void func() {}
    void func3(){}
};
```

A实例化的对象指向的虚函数表中有`func()`、`func2()`的地址，当如下创建基类指针时，obj属于指向A对象的指针，其中虚寒表中的func函数动态绑定到`B::func()`，而`func2()`绑定到`A::func2()`；并且obj是无法调用`B::func3()`，因为它是A对象的指针。

```
A* obj=new B();
```

![1-1PS1111S0Q6](https://i.loli.net/2021/03/05/h2JS5VI1Unjoyuf.jpg)

![1-1PS1111SQ58](https://i.loli.net/2021/03/05/2q1EQgiUpISyBKM.jpg)



再考虑下例，尽管`fun2()`是在基类函数中调用的，但是由于在虚函数表中已经将基类的`func2`绑定到了派生类的`func2`，因此最终还是执行了派生类的函数。即，**在非构造函数，非析构函数的成员函数中调用虚函数，是多态!!!**

```
class Base 
{
public:
    void fun1() 
    { 
        fun2(); 
    }
    
    virtual void fun2()  // 虚函数
    { 
        cout << "Base::fun2()" << endl; 
    }
};

class Derived : public Base 
{
public:
    virtual void fun2()  // 虚函数
    { 
        cout << "Derived:fun2()" << endl; 
    }
};

int main() 
{
    Derived d;
    Base * pBase = & d;
    pBase->fun1();
    return 0;
}

```



### 构造函数和析构函数是否有多态

接上文，**在构造函数和析构函数中调用虚函数不是多态**。

`B* obj=new A();delete obj;`这个代码中，程序顺序是：执行完基类的构造函数->执行完派生类的构造函数->调用派生类的析构->调用基类的析构。

可见，执行基类的构造函数时派生类还未创建，**建立的虚函数表中虚函数执行基类自身函数**，因此多态还没形成，此时构造函数中的`func2`将执行基类的实现；析构时，因为执行基类析构时，派生类已经被释放，虚函数表也发生了变化，映射到基类自身函数中，所以也会执行基类的实现。



### 基类的virtual析构函数

我的理解是，析构函数也会作为虚函数放在虚函数表中，obj在被析构时，根据虚函数表调用派生类的析构，即先析构派生对象；当派生对象被析构完成时，虚函数表中的析构函数重新绑定到obj自身实现，实现自己的析构。

```
class A{
public:
	int *p;
	A(){p = (int *)malloc(sizeof(int));}
	virtual void func();
	// ~A(){ delete p;} 仅调用基类的析构,导致派生类中开辟的内存泄漏
	virtual ~A(){ delete p;}//virtul除了调用基类的析构还会调用派生类的析构
}
class B : public A{
public:
	B(){p = (int *)malloc(sizeof(int));}
	virtual void func(){...};
	~B(){ delete p;} 仅调用基类的析构,导致派生类中开辟的内存泄漏
}

B *obj = new A();
delete obj;
```



## 静态成员















## 重写（override）

函数名，参数列表，返回值类型，所有都必须同基类中被重写的函数一致，只有实现不同。

**普通的成员函数是可以被重写的，但是无法形成多态**，而虚函数则正是为多态而设计的；实际上，应避免对non-virtual成员函数进行重写，或者需要重写的函数应当在设计基类的时候就考虑到，并添加virtual关键字，以提示这个函数应当被派生类重写。

关于为何会产生普通函数和virtual的重写结果的区别，主要是因为普通函数重写是**静态绑定**，类似于基类和基类的getName函数绑定，派生类和派生类的getName函数绑定，之后就不再改变；而virtual函数

```
class Bird{
public:
	void getName(){cout << "Bird" << endl;}
    virtual void getNameVirtual(){cout << "Bird" << endl;}
};
class Sparrow : public Bird{
public:
	void getName(){cout << "Sparrow" << endl;}
    virtual void getNameVirtual(){cout << "Sparrow" << endl;}
};


int main()
{
    Sparrow obj;
    obj.getName();

    Sparrow* p_obj1 = new Sparrow();
    p_obj1->getName();//调用Sparrow的getName
    p_obj1->getNameVirtual();//调用Sparrow的getName
    Bird* p_obj2 = new Sparrow();
    p_obj2->getName(); //不是虚函数，会调用基类的getName
    p_obj1->getNameVirtual();//调用Sparrow的getName
}
```







## 重载（overload）





## 相关面试题

+ 构造函数能否重载，析构函数能否重载，为什么？





# C++ 语法



## remove_reference

这是一个模板，可以从引用类型还原出元素类型（通过type成员来获取）。

```c++
remove_reference<int>::type x; // 模板实参可以不是引用：int x
remove_reference<int&>::type x;// int x
```



## decltype

获取参数的类型。

```c++
int x = 0;
decltype(&x) y = &x;
```





## constexpr

常量表达式，也就是在编译期可求值的表达式。

```c++
constexpr  int  Inc( int  i) {
     return  i + 1;
}
 
constexpr  int  a = Inc(1); // ok
constexpr  int  b = Inc(cin.get()); // !error
constexpr  int  c = a * 2 + 1; // ok
```

+ 是一种很强的约束，更好地保证程序的正确语义不被破坏。
+ 编译器可以在编译期对constexpr的代码进行非常大的优化，比如将用到的constexpr表达式都直接替换成最终结果等。
+ 相比宏来说，没有额外的开销，但更安全可靠。



## noexcept

该关键字告诉编译器，函数中不会发生异常,这有利于编译器对程序做更多的优化。

C++中的异常处理是在运行时而不是编译时检测的。为了实现运行时检测，编译器创建额外的代码，然而这会妨碍程序优化。

```c++
constexpr initializer_list() noexcept
      : _M_array(0), _M_len(0) { }
```





## function

用法：function<返回类型<参数1,参数2,...>>

```c++
int f(int a, int b)
{
  return a+b;
}
function<int(int, int)> func(f);
cout << func(1, 2);
```





## define/typedef/using

+ define：将一个标识符定义为一个字符串

```C++
#define A 3
#define func(x,y) (x+y)
#define PR(...) printf(__VA_ARGS__)  // 变参操作符
#define STRING(x) #x#x // 构串操作符
##define CLASS_NAME(name) class##name // 合并操作符
```

+ typedef：定义类型别名

```c++
typedef unsigned int uint_t;
typedef struct StructName
{
} TYPE;
typedef char(*PTRFUN)(int); // 定义函数指针char(*)(int)别名为PTRFUN
```

+ using：定义类型别名（以更直观的形式）；定义模板别名

```c++
using PTRFUN = char(*)(int); // 更直观

template<typename A>
typedef vector<A> my_map; // error

template<typename A>
using my_map = vector<A>; // 正确
```





## static关键字

static作用主要有：控制变量的存储位置、可见性、生命周期。

+ 修饰全局变量（静态全局变量）
  + 原来和现在都存储在数据段（已初始化）或BSS段（未初始化）
  + 静态全局变量只在本文件中可用，不能被其他文件访问（即无法extern）

+ 修饰局部变量（静态局部变量）

  + 原本存储在栈区，现在存储在数据段和BSS段
  + 生命周期得到了延长
  + 但是作用域没有改变

+ 修饰普通函数（静态函数）

  + 和全局变量一样，不能被其他文件访问

+ 修饰类的成员变量

  + 属于类本身，被所有对象共享，比如记录对象创建的个数

  + 不能在类中进行初始化（因为初始化是调用构造函数，然后再调用成员变量的构造），在类外部使用范围解析进行初始化：

    ```
    class Box
    {
       public:
          static int objectCount;
    };
    
    int Box::objectCount = 0;//静态成员变量初始化方法
    
    int main(){...}
    ```

+ 修饰类的成员函数

  + 类函数中由static修饰的函数只能访问其他静态成员（函数、变量）以及默认构造函数（也就是可以在函数内构造本类对象）而无法访问普通函数和变量。主要是因为函数参数中没有this指针，静态资源先于对象创建前便已存在。



## 四种cast转换的区别

由于C转换可以在任意类型之间转换， 编译器不能进行错误检查，所以C++引入了四种cast方法。

+ const_cast：只用于去掉/加上常量的**底层const性质**，并且只能转指针，不能转类型，不要希望通过改转换而对对象写值。

  ```
  const int a = 3;
  int* b = const_cast<int*>(&a);//正确，去除const性质
  *b = 0;//可以编译通过，但是不能改变a的值
  int b= const_cast<int>(a);//错误，只能转指针
  double* b= const_cast<double*>(&a);//错误，不能转类型
  ```

  const_cast常用于有函数重载的上下文中，如下所示：

```c++
// 考虑到传入的参数可能是常量string，所以引用采用const形式
// 问题在于当传入非常量string时，理应返回一个string&
const string& getShorter(const string& str1, const string& str2)
{
    return str1.size() < str2.size() ? str1 :str2;
}
int main()
{
    string a, b; // 有两个非常量string
    const string& res = getShorter(a, b);
    res[0] = 'A'; // 错误，不能更改
    
}
```

利用重载+const_cast可以解决上面的问题：

```c++
// 考虑到传入的参数可能是常量string，所以引用采用const形式
// 问题在于当传入非常量string时，理应返回一个string&
const string& getShorter(const string& str1, const string& str2)
{
    return str1.size() < str2.size() ? str1 :str2;
}

// 用重载+const_cast解决上面问题
string& getShorter(string& str1, string& str2)
{
    auto& res = getShorter(const_cast<const string&>(str1), 
                            const_cast<const string&>(str2)); // 加上底层const
    return const_cast<string&>(res); // 去掉顶层const
}

int main()
{
    string a, b; // 有两个非常量string
    string& res = getShorter(a, b);
    res[0] = 'A'; // 正确    
}
```







+ static_cast:所有隐式转换都可以用static_cast，也可以找回空指针（不安全的）。

  ```
  string d = "ss";
  void* p = &d;
  double* dp = static_cast<double*>(p);//可以运行，但是类型不一致，不应当这么做
  ```

  

+ reinterpret_cast：在底层上重新解释，即能够完成任意指针类型向任意指针类型的转换，即使它们毫无关联。它非常的不安全。

  ```
  int* ip;
  char* pc2 = (char*)ip;//正确C强转可以做底层解释
  char *pc = reinterpret_cast<char*>(ip);//正确，reinterpret_cast与C强转类似，但是只限于指针
  string a(pc);// 编译通过，但是运行出错
  ```



+ dynamic_cast：用于类层次间的向上向下转化，只能转指针或者引用。向上一般都可以转换，向下转换的成败取决于将要转换的类型，是安全的转换。

  ```
  class Base {virtual void func() {} };
  class Derived :public Base {};
  int main()
  {
      Base* p1 = new Derived;
      Base* p2 = new Base;
      Derived* p3 = new Derived;
      Derived* p4 = dynamic_cast<Derived*>(p1);//返回类指针
      Derived* p5 = dynamic_cast<Derived*>(p2);//转换失败返回null指针
      Base* p6 = dynamic_cast<Base*>(p3);//返回类指针
  }
  ```
  
  
  
  



## 指针和引用的区别

**对象**是指一块能存储数据并具有某种类型的内存空间

+ **指针**也是**对象**，有地址&p和存储的值，只是存储的数据是数据的地址；引用没有空间，只是一个被引用对象的别名。
+ 既然指针有内存，所以它的sizeof就是地址长度；而对引用sizeof是被引对象的大小。
+ 指针可以初始化为NULL，但是引用初始化必须绑定到一个对象；
+ 指针可以改指，但是引用初始化后就不能改引；
+ 指针需要解引用才是对象的操作，但是引用本身就是对对象的操作；





## 四种智能指针

手动的new/delete很容易忘记释放或多次释放，造成内存泄漏。

智能指针是一个类，有析构函数，系统会通过调用其析构函数自动释放资源。

+ auto_ptr：自动指针。所谓自动表现在：内部有一个普通的指针，然后在构造的时候获取资源，析构中进行delete；所以外部就不需要主动释放。在拷贝赋值函数中会转移资源，将原auto_ptr对象中的指针赋值为null。缺陷：
  + 不能指向一个数组，因为数组的释放需要delete[]
  + 被赋值拷贝后，只有为源auto_ptr对象重新赋予资源后才能继续使用。
  + 不能用作容器元素，因为容器要求复制或赋值后，两个元素拥有相同的值。
  + 两个指针不能指向同一对象。

```
template<class T>
class AutoPtr
{
public:
	AutoPtr(T* ptr = nullptr)
		: _ptr(ptr)
	{}

	// 拷贝构造：将ap的资源转移到当前对象上
	AutoPtr(AutoPtr<T>& ap)
		: _ptr(ap._ptr)
	{
		ap._ptr = nullptr;
	}

	AutoPtr<T>& operator=(AutoPtr<T>& ap)
	{
		if (this != &ap)
		{
			if (_ptr)
				delete _ptr;

			_ptr = ap._ptr;
			ap._ptr = nullptr;
		}

		return *this;
	}

	~AutoPtr()
	{
		if (_ptr)
		{
			delete _ptr;
			_ptr = nullptr;
		}
	}

	T& operator*()
	{
		return *_ptr;
	}

	T* operator->()
	{
		return _ptr;
	}

	T* Get()
	{
		return _ptr;
	}

	void ReSet(T* ptr)
	{
		if (_ptr)
			delete _ptr;

		_ptr = ptr;
	}
protected:
	T* _ptr;
};


struct A
{
	int a;
	int b;
	int c;
};

void TestAutoPtr1()
{
	AutoPtr<int> ap1(new int);
	*ap1 = 10;

	AutoPtr<A> ap2(new A);
	ap2->a = 1;
	ap2->b = 2;
	ap2->c = 3;
}
```

+ ununique_ptr：独占式指针，和auto_ptr不同在于不允许指针赋值，实现原理是将拷贝，赋值函数的私有化；但是保留移动构造函数，即可以通过move来拷贝赋值（同auto_ptr，源指针对象不能再使用）。抑或者是拷贝来自函数的返回指针对象（右值）。unique_ptr和auto_ptr都希望一个对象只被一个智能指针所拥有，但由于auto_ptr在不小心时候会导致旧指针的非法访问，因此unique_ptr就显式禁止指针复制

```
unique_ptr<string> p1(new string ("auto"));
unique_ptr<string> p2;
p2 = p1; // 报错
```

+ share_ptr：共享指针，对auto_ptr的改进，使用计数机制来表明资源被几个指针共享，该对象和其相关资源会在“最后一个引用被销毁”时候释放。计数机制的实现原理是在类中定义一个int计数指针，在拷贝智能指针时，获取这个指针，并递增计数，由于是指针，所以所有指针对象都公用一个计数内存。

  共享指针不要进行下列操作，会造成多次释放，

  ```
  int *a = new int;
  share_ptr<int> p1(a);
  share_ptr<int> p2(p1);//没问题，计数=2
  share_ptr<int> p3(a);//会多次释放，p3计数1
  ```

  

+ week_ptr:weak_ptr是用来解决shared_ptr相互引用时的死锁问题，**它的构造和析构不会引起引用记数的增加或减少**。

  例如，A类对象a中含有指向B类对象b的共享指针，而B类对象b也含有指向A类对象a的共享指针。这两个指针只有在调用a\b对象的析构函数时才会被析构（类析构调用成员析构）。

  pb、pa作为共享指针，离开函数后被析构；注意此时对象a、b只是一块内存，没有名字，因此其析构函数不会被调用，他们的引用计数都为1。现在，对象a、b内存被释放的唯一机会就是共享指针的析构被调用，然后引用为0，指针释放内存；而指针析构调用的前提是类的析构被调用。显然，这是一个死锁。

```
void fun()
{
    shared_ptr<B> pb(new B());
    shared_ptr<A> pa(new A());
    pb->pa_ = pa;
    pa->pb_ = pb;
}
```

​	解决的方法就是pb->pa_的内部用week_ptr。使得一开始，a的引用为1，b的引用为2；这样出函数作用域时，对象a的引用为0，对象a的析构调用，导致`pa->pb_`的析构也被调用，于是对象b也被析构。



## 前置申明

在上节智能指针死锁代码中，涉及到两个类互相包含的情况（以指针的形式），那么编译时，必须采用**前置申明**才能正常编译。

```c++
struct A;
struct B;

struct A
{
    // shared_ptr<B> ptrb;
    weak_ptr<B> ptrb; // 只要将AB其中一个类的成员定义为week_ptr就可以消除死锁
};

struct B
{
    shared_ptr<A> ptra;
};


int main()
{
    shared_ptr<A> a(new A);
    shared_ptr<B> b(new B);
    a->ptrb = b;
    b->ptra = a;
}
```

如果两个类分别定义在两个文件中，那么其中一个类的头文件中必须取消include，并且采取前置申明，如果该类有cpp文件，则cpp文件中要进行include的。

```cpp
// testB.h
#ifndef _TESTB_H
#define _TESTB_H

#include<memory>
#include "testA.h"

class B
{
public:
    std::shared_ptr<A> ptrb;
};

#endif

```

```cpp
// testA.h
#ifndef _TESTA_H
#define _TESTA_H

#include<memory>
class B;
class A
{
public:
    A();
    std::shared_ptr<B> ptrb;
};

#endif

```

```cpp
// testA.cpp
#include<testA.h>
#include<testB.h>
using namespace std;

A::A()
{
    ptrb = make_shared<B>();
}
```





## lambda表达式

表达式构成：捕获列表、参数、返回类型、函数体

```
//find_if查找第一个长度<=size的字符串
int size = 4;
//返回一个迭代器，指向找到的该元素
//只有lambda才能获取外界参数
auto it = find_if(vec.begin(), vec.end(), [size](const string &s)->bool{
	return s.size() <= size;
});
```

lambda函数捕获默认采用拷贝副本，如果要引用，则需要加&

```
int i = 10;
//lambda函数在创建时，便拷贝了捕获列表参数，因此在调用时使用的值就是创建时候的值。
auto f = [i](){return i;};
i = 0;
cout << "捕获列表拷贝: "<< f() << endl;
//通过引用创建lambda，那么调用时候会去查询原值，这与拷贝不同
auto f2 = [&i](){return i;};
i = 10;
cout << "捕获列表引用: " << f2() << endl;
```



## bind()函数

```
void func(int a1, int a2, int a3);
auto it = bind(func, placeholders::_1, 0, placeholders::_2);
it(1,2);//等价于func(1, 0, 2);
```

bind()中参数排列顺序必须和func函数顺序一致，而占为符与it被调用时参数的顺序有关,placeholders::_1表示在调用时获取第一个参数值。



## const

### 修饰指针变量

```
const int *p = 8; 
int* const p = &a;//左定值，右定向
```



### 修饰函数参数

```
void Cpf(const int a); //a不能被改变
void Cpf(int *const a)；//a指针指向不可被篡改
void Cmf(const Test& _tt)； // const外加引用传递可以省去临时对象的创建，节省时间
```



### 修饰普通函数

const 加在函数前面，**返回的值不能作为左值使用，既不能被赋值，也不能被修改**，



### 修饰类成员函数

const 加在参数表后面是修饰this指针的，即**被修饰成员函数不能改变对象的变量**。所以const **const 不能与 static 关键字同时使用**，因为static成员函数没有this指针。



## 左值和右值

左值和右值本身没有一个明确清晰的定义。

+ lvalue：左值是一个存储了值的内存空间，或者叫数据单元，如变量。
+ rvalue：右值指的是数据本身，没有明确的内存地址，其存在是短暂的，可以是字面常量，或者临时对象。

有的运算符需要左值运算对象，有的运算符需要右值运算对象；有的运算返回左值结果，有的返回右值结果。需要右值的地方可以用左值替代，需要左值的地方不能用右值替代。

赋值运算符要求左操作数是左值，返回结果也是左值。

```
int var;
var = 2;//var是lvalue
int func(){return var;}
func()=2;//错误，func()返回临时对象，不是lvalue
int& func2(){return var;}
func()=2;//正确，func()返回全局变量var的引用，是lvalue
```

取地址符作用于一个左值对象，返回一个指针，该指针是右值。

```
int var1=1, var2=2;
int *p = &(var1=var2+1);//正确，赋值操作返回新的var1且是左值，*p等于3
```

注意，左值不代表能够被赋值，如：

```
const int a;
a = 0;//错误
```



### 正确认识右值



**已命名的右值引用，编译器会认为是个左值**，可以看做一个有内存的变量，只是通过右值引用申明夺取了资源。如下：

```c++
int a;
int&& b = std::move(a); // b是一个左值
int&& func(int&& a)
{
	// 函数形参也是命名了的，所以是左值
	return std::move(a); // 要先将左值a转为右值，才能赋值给临时的右值引用对象，其中没发生拷贝
}
int c;
int&& d = func(std::move(c)); // 函数返回值是右值,赋值给d的过程中未发生拷贝
```



### move的原理
{ #f0e41b}


**move的作用：将左值转为右值，从而在类对象拷贝的时候启用[[计算机科学/C++/C++八股文集锦#^ba17df\|移动语义]]，即调用移动赋值、构造函数，实现资源的移动，避免临时对象的拷贝。**

例如在一个函数中，定义了一个局部变量，这个局部变量最后的使用是通过一个消息发送接口发送出去，那么可以使用move来省去构造环节。

```cpp
AmclNode::publishParticleCloud(const pf_sample_set_t * set)
{
  auto cloud_with_weights_msg = std::make_unique<nav2_msgs::msg::ParticleCloud>();
  cloud_with_weights_msg->header.stamp = this->now();
  cloud_with_weights_msg->header.frame_id = global_frame_id_;
  cloud_with_weights_msg->particles.resize(set->sample_count);
  particle_cloud_pub_->publish(std::move(cloud_with_weights_msg));
}
```

要将一个左值转为右值，可以巧妙地利用函数返回右值的特性，但如何在函数内避免拷贝是个问题，这要通过引用来解决。由于move是模板函数，由于引用折叠，实例化后变量a就可能变为int&类型，因此要使用remove_reference去左值引用，如下：

```c++
int&& func(int& a)
{
	return static_cast<std::remove_reference<int&>::type&&>(a); 
}// 利用函数返回临时对象一般为右值，即可将a转为右值，只要确保函数体内不发生拷贝行为，所以强转为右值引用形式，返回值也要是右值，这样在返回生成临时对象时才不会发生拷贝。
```



### 左值和右值的正真区别

左值也可以避免拷贝，右值也能避免拷贝。它们的区别如下：

通过int&实现的避免拷贝无法传入右值，通过const int&实现的避免拷贝是只读的。

```c++
void func(int&);
void func2(const int&);
```

有时既想要避免拷贝，又能够传入右值，然后又可以在函数内修改该对象，则需要使用右值引用。比如移动赋值拷贝就是希望将源对象内容重置。





## 左值引用和右值引用

左值引用就是常规的引用，

```
int &a = var;//引用要求var是左值
```

右值引用用'&&'表示，用于绑定到一个即将被销毁的临时对象上，实现一个对象资源"移动"到另一个对象上。

```
int &r1 = i*42；//错误，i*42是右值
const int &r2 = i*42;//正确，由于声明了const,i*42可以作为短暂的左值而存在
int &&r3=i*42;//正确，可以将右值引用绑定到右值上
int &&r4 = i;//错误，右值引用不能绑定到左值
```

虽然不能将一个右值引用绑定到一个左值上，但是可以通过move来获取左值上的右值引用，

```
int &&r5 = std::move(i)；//正确
```



const虽然也可以对右值进行引用，但是我们无法更改r2的值，但是右值引用不一样，我们可以更改这个临时对象的值，

```
const int &r2 = i*42;
r2 = 0; // 错误
int &&r4 = 0;
r4 = 1;//正确
```



### 移动语义编写的注意点
{ #ba17df}


右值引用的另一个优点在于可以实现“资源移动”，减少临时对象的拷贝。如下，分别对比了类的普通拷贝函数、普通赋值函数、移动拷贝函数、移动赋值函数。对于普通的拷贝、赋值函数，都需要重新开辟内存，然后把input对象的内存区内容拷贝过来；对于createHolder函数返回的对象，属于右值，即临时的，在赋值后就会被释放，那么为什么省去开辟新内存的步骤，直接把临时对象的资源“偷”过来呢？移动函数就实现了这样的功能。

需要注意，移动函数在偷走源对象的资源后，会对源对象指针赋值nullptr，以避免源对象和新对象都指向同一块内存，在delete时产生多次释放的错误。

我们在编写移动语义时要注意一下事情：

+ **一旦源对象被转移了资源，其中的指针为null，则我们除了重新赋值和销毁以外，不能对其进行使用**。
+ 无论是否采用移动，拷贝和赋值函数中都应该避免自己拷贝自己、自己移动自己的情况。
+ 在移动拷贝和移动拷贝函数中，基本类型都是复制构造，这无法避免，而如果成员是自定义类，那么如果不加move则不会执行移动拷贝和赋值，所以要加move。

+ **在移动赋值和移动拷贝函数的实现中，不应当设计出成员内存拷贝的行为，并且一旦资源完成移动，源对象必须不在指向被移动的资源**。在这样的前提下，移动赋值和移动构造函数中是一定不会分配内存资源的，因此，移动操作不会抛出任何异常，通常会用noexcept告知标准库我们的移动构造函数不会抛出异常。



```
class Holder
{
public:
  Holder(int size)         // Constructor
  {
    cout << "Constructor" << endl;
    m_data = new int[size];
    m_size = size;
  }
  Holder(const Holder& other)
  {
    cout << "copy constructor (lvalue in input)" << endl;
    m_data = new int[other.m_size];  // (1)
    std::copy(other.m_data, other.m_data + other.m_size, m_data);  // (2)
    m_size = other.m_size;
  }

  Holder(Holder&& other)     // <-- rvalue reference in input
  {
    cout << "move constructor (rvalue in input)" << endl;
    m_data = other.m_data;   // (1)
    m_size = other.m_size;
    other.m_data = nullptr;  // (2)
    other.m_size = 0;
  }

  Holder& operator=(const Holder& other) 
  {
    cout << "assignment operator (lvalue in input)" << endl;
    if(this == &other) return *this;  // (1)
    delete[] m_data;  // (2)
    m_data = new int[other.m_size];
    std::copy(other.m_data, other.m_data + other.m_size, m_data);
    m_size = other.m_size;
    return *this;  // (3)
  }

  Holder& operator=(Holder&& other)     // <-- rvalue reference in input  
  {  
    cout << "move assignment operator (rvalue in input)" << endl;
    if (this == &other) return *this;

    delete[] m_data;         // (1)

    m_data = other.m_data;   // (2)
    m_size = other.m_size;

    other.m_data = nullptr;  // (3)
    other.m_size = 0;

    return *this;
  }

  ~Holder()                // Destructor
  {
    delete[] m_data;
  }

private:

  int*   m_data;
  size_t m_size;
};

Holder createHolder(int size)
{
  //  move constructor (rvalue in input) 
  return Holder(size);
}


int main()
{
  Holder h1(1000);                // regular constructor
  cout << "-------------"<< endl;
  Holder h2(h1);                  // copy constructor (lvalue in input)
  cout << "-------------"<< endl;
  Holder h3(createHolder(2000));  // move constructor (rvalue in input)
  cout << "-------------"<< endl;
  h2 = h3;                        // assignment operator (lvalue in input)
  cout << "-------------"<< endl;
  h2 = createHolder(500);         // move assignment operator (rvalue in input)
  cout << "-------------"<< endl;
  //std::move将左值转为右值
  Holder h4(std::move(h1)); 

}
```



### 区别右值引用和移动语义

```c++
B a;
B&& b = std::move(a); // 右值引用，无论B类是否定义移动函数，都是直接转移资源，这种转移是在内存层面直接转移的。
B c(a);// 涉及到类的拷贝赋值，右值引用只是实现移动语义的方法。

int a;
int b = std::move(a); // 会发生内存拷贝，因为没有内置类型没有定义移动赋值函数
int&& c = std::move(a); // 不会发生拷贝，因为这是右值引用
```



## 完美转发

所谓转发，就是通过一个函数将参数继续转交给另一个函数进行处理，原参数可能是右值，可能是左值，如果还能继续保持参数的原有特征，那么它就是完美的。例如一个不完美转发的例子，这种不完美是因为右值引用本身是左值而导致的。

```c++
void func1(int& a){}
void func1(int&& a){}
void func2(int&& a)
{
    func1(a);
}
```

要实现完美转发，需要用模板使得函数既能接收左值和右值（func2不能绑定到左值），然后要用forward来保持左右值性质不变。

```c++
template<typename... Args>
void func2(Args&&... args)
{
    func1(std::forward<Args>(args)...); // 完美转发
}
```



### forward的原理

forward和move的区别在于，move只会返回右值，而forward要根据不同的模板参数返回左值或右值，具体来说函数返回值不一定都是右值，当返回是一个左值引用时，就不是右值，因此我们可以利用模板引用折叠的机制，来进行实现：

```c++
template<typename T>
T&& forward(remove_reference_t<T>& _Arg) // _Arg先是对T去除引用，然后加一个&，所以必然是左值引用，正好符合函数参数是左值这一原则
{
    return static_cast<T&&>(_Arg);// 引用折叠,T为int&，T&&折叠为int&，T为int&&，T&&折叠为int&&
}
```





# C++ 模板



## 函数类型模板

```
template<typename T1, typename T2>
void compare(T1 v1, T1 v2, T2 v3){}
// 自动类型判断
compare(1, 0, 0);// compare<int, int>(int v1, int v2, int v3)
compare(1.0, 0.0, 0);// compare<double, int>(double v1, double v2, int v3)
// compare(1, 0.0, 0) // 错误，v1和v2必须同类型
compare2<int, double>(1.0, 2, 3);// 显式实例化
```



## 类模板

```
template<typename T>
class Object
{
    T func(T v1);
};
template<typename T>//类外部的每个成员函数实现前都要加
T Object<T>::func(T v1)
{
    T temp;
    return temp;//返回必须要是T类型
}
```



## 非类型参数模板

参数表示一个值而非类型。

```
template<int N>
int compare3(double a[N]){}
double temp[4];
compare3<4>(temp);//compare3(double a[4])
```



## 可变参数模板

```
template<typename T, typename... Args>
void foo(T v1, const Args& ... rest)
{
    cout << sizeof...(Args) << endl;// 类型参数数目
    cout << sizeof...(rest) << endl;// 类型参数数目
}
// 隐式
foo(1, 1, 1.0, "s");//输出3,3
// 显式
foo<int, double>(1, 1);//输出1,1
```



## 模板类继承中访问权限问题

模板类继承中，对于父类成员的访问需要加上作用域：

```
template<typename T>
class A
{
public:
    int a;
};

template<typename T>
class B: public A<T>
{
public:
    void func()
    {
        A<T>::a++;
    }
};
```

