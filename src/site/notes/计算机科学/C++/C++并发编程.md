---
{"dg-publish":true,"permalink":"/计算机科学/C++/C++并发编程/","tags":["Concurrent","Operating-Systems","High-Performance"]}
---




# 并发

## 多核并发

在以前，大多数计算机只有一个处理单元，系统以**任务切换**的方式进行运作，形成“并行的假象”；而今，多核计算机早已普及，一台计算机拥有多个处理单元，可以做到硬件级的真正并行。

如下图所示，运行两个任务，双核系统允许两个任务同时执行，并且没有额外的开销；而在单核系统上，两个任务轮流执行，存在切换任务的开销（灰色部分）。

![计算机科学/C++/assets/C++并发编程/c0be9a56a69e77b4d121eed6643326c0_MD5.png](/img/user/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6/C++/assets/C++%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/c0be9a56a69e77b4d121eed6643326c0_MD5.png)

但即使是多核系统，核心数也是远小于实际需要运行的任务数的，这也就是说，任务切换的方式在多核系统上也有存在的必要性，下图展示了一个双核系统更真实的运作情况。

![计算机科学/C++/assets/C++并发编程/9e18623d3d599ca2d578b34e460f841a_MD5.png](/img/user/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6/C++/assets/C++%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/9e18623d3d599ca2d578b34e460f841a_MD5.png)



## 何时使用并发

使用的并发的原因有二：

+ 功能程序的分离：将功能不同的代码分离开来，可以使程序更易理解和测试，比如将图像处理和GUI程序分离；
+ 追求高性能：合理利用并发可以提高程序性能，并发提升性能有两种方式，
  + 任务并行：每个线程负责处理完整过程的一部分；
  + 数据并行：每个线程使用相同处理流程处理完整数据的一部分；

然而，并不是所有时候都适合使用并发，并发可能产生以下问题：

+ 编写和维护并发代码通常是件痛苦的事情，如果并发所带来的性能提升甚微，不如把这些时间省下来做更有意义的事；
+ 多线程会带来额外开销，有些任务用单线程方式实现性能可能更高；
+ 操作系统上的线程资源有限（每个线程要维护独立堆栈）；
+ 线程数过多可能会导致整体应用性能下降；



# 线程管理

## 通过函数操作符进行构造

使用`std::thread`时，除了传入普通函数创建线程外，也可以传入一个对象，前提是该对象定义了函数操作符。在这个例子中，我们使用多线程同时对一张图片的不同区域作阈值化处理。

```c++
class BinarizationTask
{
public:
    BinarizationTask(cv::Mat img, int threshold):
        img_(img), threshold_(threshold){}
    void operator()(const cv::Range &range) // 函数操作符
    {
        int r0 = range.start, r1 = range.end;
        int cols = img_.cols;
        for (int i = r0; i < r1; i++)
            for (int j = 0; j < cols; j++)
                if (img_.at<uchar>(i, j) > threshold_)
                    img_.at<uchar>(i, j) = 255;
                else
                    img_.at<uchar>(i, j) = 0;
    }
private:
    cv::Mat img_;
    int threshold_;
};

int main()
{
    cv::Mat img = cv::imread("./lena.jpg", 0);
    BinarizationTask task(img, 125);
    std::thread th1(task, cv::Range(0, img.rows / 2));
    std::thread th2(task, cv::Range(img.rows / 2, img.rows));
    th1.join();
    th2.join();
    cv::imshow("bin", img);
    cv::waitKey(0);
}
```



## 在析构函数中汇入

join函数需要考虑的问题是，什么时候写join函数。如果线程汇入点放置的位置不完全正确，可能会由于中途异常抛出而导致线程跳过join()，例如：

```c++
int main()
{
    std::thread t(func);
    do_something_in_current_thread(); // 发生异常，主线程提前终止
    t.join(); // 未按照预期汇入
}
```

避免这种情况的方法之一是使用`try...catch...`捕获，然后在catch中join；另一种方式是使用RAII方式，在构造函数中对线程进行汇入。

```c++
class thread_guard 
{
public:
    explicit thread_guard(std::thread &t):t_(t){}
    ~thread_guard(){
        if (t_.joinable())  t_.join();
    }
private:
    std::thread &t_;
    thread_guard(const thread_guard&);
    thread_guard& operator=(const thread_guard&);
};

int main()
{
    std::thread t(func);
    thread_guard g(t);
    do_something_in_current_thread(); // 发生异常，主线程提前终止，析构g对象，调用join()
    if (t.joinable())  t.join();
}
```

当然，我们不一定要额外创建一个`thread_guard`——如果线程任务函数是一个类的成员函数，那么只要在这个类的析构函数中join即可。



## 分离线程

线程在创建出来后除了可以被汇入（join）之外，也可以执行分离（detach）。分离的线程意味着主线程不再拥有对它的控制权，即使主线程之后结束，分离的线程也可以独立继续运行，C++ Runtime Library保证在分离的线程结束时，相关资源能够得到回收。

**分离线程要确保子线程中的参数必须为对象的副本**，否则主线程结束，子线程使用的部分资源会被释放。

分离线程很有用，比如UNIX守护线程（分离线程）可以在后台监视文件系统、清理缓存、对数据结构进行优化。试想一下在编辑一个Word文档时，在选项卡中点击创建一个新文档窗口的这个过程是如何实现的。一种处理方式是，让每个文档窗口拥有自己的线程，每个线程运行同样的代码，并隔离不同窗口处理的数据。如此，基于当前窗口新建文档时，实际上就是创建了一个分离线程，即使之后关闭了当前窗口，这个新窗口也不会收到影响。



## 传递参数

与`std:bind`的传参机制相同，使用`std::thread`创建线程时，传递参数的过程如下：

+ 向`std::thread`构造函数传参：一般实参会被拷贝至新线程的内存空间。具体拷贝的过程是由调用线程（主线程）在堆上创建并交由子线程管理，在子线程结束时同时被释放。
+ 向线程函数传参：由于`std::thread`对象里一般保存的是参数的副本，为了效率同时兼顾一些只移动类型的对象，所有的副本均被`std::move`到线程函数，即以右值的形式传入。



### 避免隐式转换

第一个过程采用拷贝是必要的，因为调用线程（主线程）很可能先于新线程结束而释放相关对象的内存。但是，这个机制并不能保证内存访问的完全安全，例如悬空指针的情况：

```C++
void func(std::string s){}
int main()
{
    char buffer[]="hello";
    std::thread th1(func, buffer);
    th1.detach(); // 除非使用join()，否则会导致悬空指针
}
```

上述代码中，buffer需要隐式转换为string类型才能传入线程函数func，但是在传参的第一阶段，构造函数拷贝的是buffer指针，而非整个字符串对象，在第一阶段之后main函数就可以结束，并会释放字符串的内存，如果之后才发生隐式转换，便会导致内存的非法访问。

解决方案就是使用显式转换，使得第一阶段能拷贝真实对象的内存，

```c++
void func(std::string s){}
int main()
{
    char buffer[]="hello";
    std::thread th1(func, std::string(buffer));
    th1.detach();
}
```



### 使用std::ref

传参的第二个过程，线程函数接受右值实参，这就使得引用形参不能直接通过编译，

```c++
void func(int &s){} // 左值引用不能绑定到右值
int i = 0;
std::thread th1(func, i); // 产生编译错误
th1.join();
```

这时候为了能通过编译，并且保留引用性质，可以使用`std::ref`将参数转换为引用的形式，这样线程函数就收到局部变量的引用，而非其拷贝副本。

```c++
void func(int &s){}
int i = 0;
std::thread th1(func, std::ref(i); // 编译通过
th1.join();
```

> 分离线程不应当使用引用参数，因为引用的局部变量不会被拷贝为副本，当离开作用域后，变量内存会被释放。



## 转移所有权

类似于标准库中的`std::unique_ptr`、`std::ifstream`等资源占有类型，`std::thread`也是可移动但不可复制的（删除拷贝和赋值函数，重新定义了移动拷贝和移动赋值函数）。

利用转移特性，我们可以实现`thread_guard `的另一个版本，

```c++
class thread_guard 
{
public:
    explicit thread_guard(std::thread t):t_(std::move(t)){}
    ~thread_guard(){
        if (t_.joinable())  t_.join();
    }
private:
    std::thread t_;
    thread_guard(const thread_guard&);
    thread_guard& operator=(const thread_guard&);
};

int main()
{
    thread_guard g(std::thread(func));
}
```

与之前版本的代码类似，不过新版本下新线程作为临时量直接传递到`thread_guard`的构造函数中，而非创建一个独立变量。



# 共享数据



## 使用std::lock_guard

C++标准库为互斥量提供了RAII模板类`std::lock_guard`，在构造时加锁，在析构时自动释放锁，比使用lock和unlock要安全。

```c++
std::mutex lock;
int cnt=0;
void func()
{
    std::lock_guard<std::mutex> guard(lock);
    for (int i = 0; i < 10000; i++) cnt++;
}//析构，释放锁
```



## 谨慎传递保护数据

切勿将受保护数据的指针或引用传递到外部作用域，或者传递到用户函数，这会导致保护失效。

```c++
class data_wrapper
{
private:
    int data=0;
    std::mutex m;
public:
    template<typename Function>
    void process_data(Function func)
    {
        std::lock_guard<std::mutex> l(m);
        func(data);
    }
};

int* unprotected_data;
void user_func1(int& protected_data)
{
    unprotected_data=&protected_data; // 用户获取了受保护数据的指针
}
data_wrapper x;
void user_func2()
{
    x.process_data(user_func1);
    (*unprotected_data)++; // 非线程安全操作
}
```



## 实现并发堆栈

即使确保了实现代码的线程安全，使用接口不一定仍是线程安全的，因为**接口可能存在条件竞争**。这是接口固有的问题，与实现方式无关。

例如我们可以实现一个栈容器，其中一些系列操作pop()、top()、push()、empty()各自都是线程安全的，即多个线程对同一容器执行push操作是安全、多个线程对同一容器执行pop操作（元素数量足够多）是安全的；但是当多个接口混合使用时，会产生竞争，这种竞争需要用户负责消除。

```c++
// 存在接口竞争
stack<int> s;
if (!s.empty()){
    int const value = s.top();
    s.pop();
    do_something(value);
}
```

上述代码一种可能的执行顺序如下图所示，

![计算机科学/C++/assets/C++并发编程/044d59658395947749313f6f308f4be3_MD5.png](/img/user/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6/C++/assets/C++%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/044d59658395947749313f6f308f4be3_MD5.png)

要消除接口间的竞争，方法就是变更接口设计，比如我们可以把pop和top接口合并，让pop直接返回栈顶元素。但这产生了另外一个问题——**异常安全**。

我们假设有一个`stack<vector<int>>`，因为vector是在堆上存储的，如果系统资源不足，内存分配会失败。当pop()函数返回“弹出值”时，会从栈中将这个值删除，然后调用vector的拷贝构造函数传递给外部变量；假如在这个赋值过程中，堆内存空间不足，分配失败，就会抛出异常，弹出的数据将会丢失。完美的程序应当使用`try...catch...`捕获该异常，然后清理出一些不要的内存，再重新进行拷贝赋值，但麻烦在于pop已将该值从堆中删除了。

所以，你能发现C++标准库的栈容器采用的仍是分离设计，在top()函数返回时拷贝构造，如果出现异常，因为数据还在堆中，所以应用仍然有弥补的机会。

无论是单纯的分离还是接口合并，到现在为止都不能兼具异常安全和线程安全。幸运的是，我们还有别的方案，虽然这些方案牺牲了部分性能。

+ 使用引用形参：即在pop函数之前预先构造一个实例用于接收；对象必须支持赋值或拷贝操作；由于需要临时构造一个实例，所以有额外的开销。

+ 使用指针返回类型：返回一个指向弹出元素的指针，而不是直接返回值；指针的拷贝不会产生异常；指针的麻烦在于内存管理，如果使用智能指针，相对来说开销会大。

下面给出了基于C++标准库stack实现的并发堆栈，它既是异常安全的，也是线程安全的。

```c++
struct empty_stack: std::exception
{
    const char* what() const throw()
    {
        return "empty stack";
    }  
};

template<typename T>
class threadsafe_stack
{
public:
    threadsafe_stack(){}
    threadsafe_stack(const threadsafe_stack& other)
    {
        std::lock_guard<std::mutex> lock(m);
        data = other.data;
    }
    threadsafe_stack& operator=(const threadsafe_stack&) = delete;

    void push(T new_value)
    {
        std::lock_guard<std::mutex> lock(m);
        data.push(std::move(new_value));
    }
    void pop(T &value) // 使用引用形参
    {
        std::lock_guard<std::mutex> lock(m);
        if (data.empty()) throw empty_stack();
        value = std::move(data.top()); // 不需要再申请内存了
        data.pop();
    }
    std::shared_ptr<T> pop()
    {
        std::lock_guard<std::mutex> lock(m);
        if (data.empty()) throw empty_stack();
        std::shared_ptr<T> res(std::make_shared<T>(std::move(data.top()))); //即使内存分配失败，也没有执行pop，仍然可以补救
        data.pop();
        return res;
    }
    bool empty() const
    {
        std::lock_guard<std::mutex> lock(m);
        return data.empty();
    }
private:
    std::stack<T> data;
    mutable std::mutex m; // empty()是const函数， 要声明为可变的才能lock
};
```







## 原子加锁

避免死锁问题有两种常用方法：
{ #fe36cc}


+ 固定加锁顺序：总在互斥量B锁住前先锁住A，就不会导致死锁（大多死锁情况其加锁顺序都是相反的）。
+ 原子加锁：要么将多个锁都锁住，要么一个都不锁。

在“哲学家问题”，这两个方法都是有效的，我们假设只有两个哲学家和两把叉子的情况：

虽然问题最初规定每个哲学家先取左叉再取右叉，看似加锁顺序固定，其实不然——由于两个哲学家形成了环路，第一个哲学家导致1号叉先加锁，2号叉后加锁，第二个哲学家则导致2号叉先加锁，1号叉后加锁，形成死锁。如果我们让第二个哲学家先取右叉再取左叉，则加锁顺序一致，避免了死锁。

其次，我们也可以规定每个哲学家只有同时拿到叉子时才能进餐，否则必须归还叉子，这样也能避免死锁，即所谓的原子加锁方法。

相比较，第二种方法更通用一些，这是因为固定加锁顺序在某些情况下难以实现。比如交换两个对象数据的场合，最初代码如下，

```c++
class data_wrapper
{
public:
    data_wrapper(int d):data(d){}
    friend void swapData(data_wrapper &lhs, data_wrapper &rhs)
    {
        if (&lhs == &rhs) return; // 不能对同一个锁重复加锁
        std::lock_guard<std::mutex> lock_a(lhs.m);
        std::lock_guard<std::mutex> lock_b(rhs.m);
        std::swap(lhs.data, rhs.data);        
    }
private:
    int data;
    std::mutex m;
};

data_wrapper d1(0), d2(1);
void test(int order)
{
    order ? swapData(d1, d2) : swapData(d2, d1); // 参数传入的顺序最后会决定加锁的顺序
}
std::thread th1(test, 0);
std::thread th2(test, 1);
```

虽然，在swap函数中，我们的实现尽可能保持固定的加锁顺序，但是最终的顺序取决于用户传入的参数。在这样的情况下，我们换用原子加锁就能避免死锁。

C++为原子加锁提供了`std::lock`和`std::scoped_lock`，前者还需要借助`std::guard_lock`释放锁，后者是C++17特性，可以自己负责锁的释放。

```c++
friend void swapData(data_wrapper &lhs, data_wrapper &rhs)
{
    if (&lhs == &rhs) return;
    std::lock(lhs.m, rhs.m);
    std::lock_guard<std::mutex> lock_a(lhs.m, std::adopt_lock); // std::adopt_lock告知lock_guard只获取锁的拥有权，而不会加锁；释放锁需要由lock_guard析构完成，std::lock不负责释放
    std::lock_guard<std::mutex> lock_b(rhs.m, std::adopt_lock);
    std::swap(lhs.data, rhs.data);
}
```

```c++
friend void swapData(data_wrapper &lhs, data_wrapper &rhs)
{
    if (&lhs == &rhs) return;
    std::scoped_lock guard(lhs.m, rhs.m);
    std::swap(lhs.data, rhs.data);
}
```



## 层次锁

层次锁也是一种固定加锁顺序的方式。一个线程如果获取了一个层次锁实例，那么接下来它只能获取更低层次的锁，即层次锁确保一个线程按照层次递减的顺序进行加锁。

使用层次锁需要对应用进行分层，并且本身需要提供层次检查机制。

一个简单的层次锁实现如下，

```c++
class hierarchical_mutex
{
    std::mutex internal_mutex;
    unsigned long const hierarchy_value;
    // 记录在当前mutex被上锁前的层级
    unsigned long previous_hierarchy_value;
    // thread_local：每个线程有自己的副本
    // satic 独立于所有对象存在
    static thread_local unsigned long this_thread_hierarchy_value;

    void check_for_hierarchy_violation()
    {
        // 一个线程可能使用多个mutex,要锁住当前mutex，要确保层级递减
        if(this_thread_hierarchy_value <= hierarchy_value)
        {
            throw std::logic_error("mutex hierarchy violated");
        }
    }
    void update_hierarchy_value()
    {
        previous_hierarchy_value=this_thread_hierarchy_value;
        this_thread_hierarchy_value=hierarchy_value;
    }
public:
    explicit hierarchical_mutex(unsigned long value):
        hierarchy_value(value),
        previous_hierarchy_value(0)
    {}
    void lock()
    {
        check_for_hierarchy_violation();
        internal_mutex.lock();
        update_hierarchy_value();
    }
    void unlock()
    {
        // 必须按照层级顺序一依次解锁
        if(this_thread_hierarchy_value!=hierarchy_value) 
            throw std::logic_error("mutex hierarchy violated");
        this_thread_hierarchy_value=previous_hierarchy_value;
        internal_mutex.unlock();
    }
    bool try_lock()
    {
        check_for_hierarchy_violation();
        if(!internal_mutex.try_lock())
            return false;
        update_hierarchy_value();
        return true;
    }
};

// 当前线程层级值，初始化为最大值，使得起初任何mutex都可以被锁住
thread_local unsigned long
    hierarchical_mutex::this_thread_hierarchy_value(ULONG_MAX);
```

+ `this_thread_hierarchy_value`：该值表示当前线程的层级，通过`check_for_hierarchy_violation()`它限制了当前线程能够获取的锁的级别。它必须独立于类实例，并且每个线程都应有一个独立的副本，所以你可以看到该变量分别使用了`static`、`thread_local`关键字声明；并且初始化为`ULONG_MAX`，使最开始线程可以获取任何锁资源。
+ `update_hierarchy_value()`：每次加锁成功后，需要更新`this_thread_hierarchy_value`和`previous_hierarchy_value`变量，前者的更新则是确保`lock()`时按照层级递减的顺序加锁；后者的更新是确保`unlock()`时按照加锁的顺序逆向解锁（即层级递增）。

层次锁的测试应用如下，其中线程a是符合层级规则的，线程b不符合。

```c++
hierarchical_mutex high_level_mutex(10000);
hierarchical_mutex low_level_mutex(5000);

int do_low_level_stuff()
{
    return 42;
}


int low_level_func()
{
    std::lock_guard<hierarchical_mutex> lk(low_level_mutex);
    return do_low_level_stuff();
}

void high_level_stuff(int some_param)
{}


void high_level_func()
{
    std::lock_guard<hierarchical_mutex> lk(high_level_mutex);
    high_level_stuff(low_level_func());
}

void thread_a()
{
    high_level_func();
}

hierarchical_mutex other_mutex(100);
void do_other_stuff()
{}


void other_stuff()
{
    high_level_func();
    do_other_stuff();
}

void thread_b()
{
    std::lock_guard<hierarchical_mutex> lk(other_mutex);
    other_stuff();
}
```



## 使用std::unique_lock

`std::unique_lock`类似于`std::lock_guard`，也采用了RAII机制。区别在于前者提供了更多接口，如`lock()`、`unlock`、`try_lock()`等，使用起来更为灵活，当然作为代价，其性能要低于后者。

首先，`std::lock_guard`在定义时加锁并取得互斥量的管理权，而`std::unique_lock`可以接受`std::defer_lock`作为参数，使得定义时取得管理权但不加锁。在定义之后，我们可以通过`lock()`和`unlock()`函数加锁和释放锁，并且它内部会有标记表示锁是否已经释放，然后决定析构时是否执行`unlock()`。

```c++
std::mutex m;
void func()
{
    std::unique_lock lk(m, std::defer_lock);
    lk.lock();
    do_task1();
    lk.unlock();
    do_task2(); // task2可以不加锁执行
    lk.lock();
    do_task3();
}// lk析构时判断当前锁是否已经被用户释放，如果没有则执行unlock
```



## 锁的粒度

> We all know the frustration of waiting in the checkout line in a supermarket with a cart full of groceries only for the person currently being served to suddenly realize that they forgot some cranberry sauce and then leave everybody waiting while they go and find some, or for the cashier to be ready for payment and the customer to only then start rummaging in their bag for their wallet. Everything proceeds much more easily if everybody gets to the checkout with everything they want and with an appropriate method of payment ready.



锁的粒度用来描述这个锁保护着的数据量大小。大多时候，锁的粒度和持锁的时间是紧密相关的，当然也有粒度小但持锁时间长的情况；但总之，锁的粒度会影响持锁时间，时间变长，就会产生更多的线程等待。

例如下例，当读取共享数据和写入共享数据时加锁，而数据处理过程是对数据的副本进行的，完全不需要放在加锁区域之内。

```c++
void get_and_process_data() {
    std::unique_lock<std::mutex> my_lock(the_mutex);
    some_class data_to_process=get_next_data_chunk();
    my_lock.unlock();
    result_type result=process(data_to_process); 
    my_lock.lock();
    write_result(data_to_process,result);
}
```



然而，当我们一味削减不必要的持锁时间时，很可能导致一些竞争隐患。比如在比较操作中，不再对整个比较操作加锁，而仅对比较数的读取过程加锁，最后导致相比较的不同时间点的数值。

```c++
class Y {
private:
    int some_detail; mutable std::mutex m; int get_detail() const {
        std::lock_guard<std::mutex> lock_a(m); 
        return some_detail;
    }
public: 
    Y(int sd):some_detail(sd){}
    friend bool operator==(Y const& lhs, Y const& rhs) {
        if(&lhs==&rhs) return true;
        int const lhs_value=lhs.get_detail(); // 时刻t1获取的lhs
        									  // 另一个线程改变了lhs的值
        int const rhs_value=rhs.get_detail(); // 时刻t2获取的rhs
        return lhs_value==rhs_value;
    } 
};
```

因此，我们应当尽可能选择合适粒度的锁，既保证锁有能力保护数据，又不至于带来无谓的等待。



## 只读数据的初始化

只读数据因为不需要更新，所以是多线程安全的，但数据初始化（延迟初始化）时仍然需要保护。这个问题最常见的应用就是单例模式中实例的创建。

下面给出了单线程下的延迟初始化代码，

```c++
std::shared_ptr<some_resource> resource_ptr; 
void foo() {
    if(!resource_ptr) {
        resource_ptr.reset(new some_resource); 
    } 
    resource_ptr->do_something(); 
}
```

转为多线程代码时，需要保护`reset`操作。我们可以在`foo`函数一开始就加锁，在`do_something()`前释放锁。但是因为初始化的操作可能只进行一次，如果读操作也一直加锁和释放锁的话就很不划算了。

### 双重检查锁方式

一种常见的方式是使用**双重检查锁模式**。

```c++
void foo()
{
    if (!resource_ptr)
    {
        std::lock_guard<std::mutex> lk(resource_mutex);
        if(!resource_ptr)
        {
            resource_ptr.reset(new some_resource);
        }
    }
    resource_ptr->do_something();
}
```

但是，这样的实现方式仍然存在潜在的条件竞争，具体可以参考著名的论文《[C++ and the Perils of Double-Checked Locking](https://www.aristeia.com/Papers/DDJ_Jul_Aug_2004_revised.pdf)》。



### 静态创建方式

另一种方法是使用static声明局部变量，该方式在声明时便完成初始化，所以无法做到延迟初始化。其次，这种方法放在C++11标准之前是存在条件竞争的——多个线程抢着去定义这个变量，C++11之后，该问题便得到了解决。

```c++
class my_class;
my_class& get_my_class_instance() {
    static my_class instance = new my_class; 
    return instance;
}
```



### 使用std::call_once

C++标准提供了`std::call_once`可以解决多线程下延迟初始化的问题，并且使用它比显式使用互斥量消耗的资源更少。当多线程进入`foo`函数时，只会有一个线程执行`init()`，其他线程都要等待这个线程完成数据的初始化操作才能继续运行。

```c++
std::once_flag flag;
void init()
{
    resource_ptr.reset(new some_resource);
}
void foo()
{
	std::call_once(flag, init);
    resource_ptr->do_something();
}
```





## 保护不常更新的数据

有这么些共享变量，线程对它们的访问大多以读取方式进行，偶尔会发生写入操作；为了同步写入，我们使用`std::mutex`来保护数据，但同时使得读取操作的开销增大，多少有点不划算。

C++17标准库提供了`std::shared_mutex`，即共享锁。当有线程拥有共享锁时，尝试获取独占锁会被阻塞，直到所有其他线程放弃锁；当任一线程拥有一个独占锁时，其他线程就无法获得共享锁或者独占锁，直到第一个线程放弃其拥有的锁。

通常，我们使用`std::shared_lock<std::shared_mutex>`获取访问权；而使用`std::lock_guard<std::shared_mutex>`获取写入权。

```c++
class dns_cache
{
    std::map<std::string,dns_entry> entries;
    std::shared_mutex entry_mutex;
public:
    dns_entry find_entry(std::string const& domain)
    {
        std::shared_lock<std::shared_mutex> lk(entry_mutex); // 使用std::shared_lock共享锁
        std::map<std::string,dns_entry>::const_iterator const it=
            entries.find(domain);
        return (it==entries.end())?dns_entry():it->second;
    }
    void update_or_add_entry(std::string const& domain,
                             dns_entry const& dns_details)
    {
        std::lock_guard<std::shared_mutex> lk(entry_mutex); // 使用std::lock_guard独占锁
        entries[domain]=dns_details;
    }
};
```





# 同步操作

> Suppose you’re traveling on an overnight train. One way to ensure you get off at the right station would be to stay awake all night and pay attention to where the train stops. You wouldn’t miss your station, but you’d be tired when you got there. Alternatively, you could look at the timetable to see when the train is supposed to arrive, set your alarm a bit before, and go to sleep. That would be OK; you wouldn’t miss your stop, but if the train got delayed, you’d wake up too early. There’s also the possibility that your alarm clock’s batteries would die, and you’d sleep too long and miss your station. What would be ideal is if you could go to sleep and have somebody or something wake you up when the train gets to your station, whenever that is.



## 条件变量

对于一个线程等待另一个线程的需求，单单使用互斥量作为共享数据标志会浪费CPU资源；我们更希望正在等待的线程能够处于休眠状态，而不是原地自旋；然而睡眠的时间是很难把控的，因此最佳的方案应当是让持有资源的线程完成相关任务后主动唤醒它，这种机制称为**条件变量**。

C++标准库对条件变量提供了两套实现：`std::condition_variable`和`std::condition_variable_any`，前者只能和`std::mutex`搭配，后者还可以和其他互斥量一起工作，所以更为灵活（同时意味着更多的开销）。

比如，我们可以使用条件变量处理队列中的数据等待。生产者不断向队列中添加数据，并负责唤醒消费线程；消费者不断读取数据，并在队列为空时进入等待。

```c++
std::mutex m;
std::queue<int> data_queue;
std::condition_variable data_cond;
void produce()
{
    while(true)
    {
        std::lock_guard<std::mutex> lk(m);
        int data = get_data();
        data_queue.push(data);
        data_cond.notify_one(); // 唤醒消费线程
    }
}

void consume()
{
    while (true)
    {
        std::unique_lock<std::mutex> lk(m); // 不能使用lock_guard
        data_cond.wait(lk, []{return !data_queue.empty();}); // 用lambda表达式作为等待条件
        int data = data_queue.front();
        data_queue.pop();
        lk.unlock();
        process(data);
    }  
}
```

在`wait()`中，如果条件不满足，将解锁互斥量并使线程休眠；当`notify_once()`调用后，消费线程被唤醒，重新获取互斥锁，再次进行条件检查，条件满足时才执行后续代码。由于这一过程中`wait()`需要对互斥量进行锁定和释放，因此必须使用`unique_lock`而不是`lock_guard`。

当有多个线程等待同一事件时，`notify_once()`只会通知其中某一个线程，`std::condition_variable`也提供`notify_all()`接口，用于通知等待的全部线程。



## 实现并发队列

对于上一节的代码，事实上我们可以将同步操作转移到队列中去，实现一个异常安全和线程安全的并发队列，就如之前的并发堆栈一样，我们必须考虑到内存分配可能产生的异常，也要避免接口间的竞争。

以下是一个完整的实现。

```c++
template<typename T>
class threadsafe_queue
{
public:
    threadsafe_queue(){};
    void push(T new_value);

    // 尝试从队列中弹出数据，即使队列为空，也会直接返回（结果为false）
    bool try_pop(T &value);
    std::shared_ptr<T> try_pop();

    // 如果队列为空，将会等待队列有值时才返回
    void wait_and_pop(T &value);
    std::shared_ptr<T> wait_and_pop();

    bool empty() const;

private:
    std::queue<T> data_queue;
    std::condition_variable data_cond;
    mutable std::mutex m;
};

template<typename T>
void threadsafe_queue<T>::push(T new_value)
{
    std::lock_guard<std::mutex> lk(m);
    data_queue.push(std::move(new_value));
    data_cond.notify_one();
}

template<typename T>
bool threadsafe_queue<T>::try_pop(T &value)
{
    std::lock_guard<std::mutex> lk(m);
    if (data_queue.empty()) return false;
    value = std::move(data_queue.front());
    data_queue.pop();
}

template<typename T>
std::shared_ptr<T> threadsafe_queue<T>::try_pop()
{
    std::lock_guard<std::mutex> lk(m);
    if (data_queue.empty()) return std::make_shared<T>();
    std::shared_ptr<T> res(std::make_shared<T>(std::move(data_queue.front())));
    data_queue.pop();
    return res;
}

template<typename T>
void threadsafe_queue<T>::wait_and_pop(T &value)
{
    std::unique_lock<std::mutex> lk(m);
    data_cond.wait(lk, [this]{return !data_queue.empty();});
    value = std::move(data_queue.front());
    data_queue.pop();
}

template<typename T>
std::shared_ptr<T> threadsafe_queue<T>::wait_and_pop()
{
    std::lock_guard<std::mutex> lk(m);
    data_cond.wait(lk, [this]{return !data_queue.empty();});
    std::shared_ptr<T> res(std::make_shared<T>(std::move(data_queue.front())));
    data_queue.pop();
    return res;
}

template<typename T>
bool threadsafe_queue<T>::empty() const
{
    std::lock_guard<std::mutex> lock(m);
    return data_queue.empty();
}
```

利用并发队列，我们可以简化上节中数据生产和消费的代码，

```c++
threadsafe_queue<int> data_queue;
void produce()
{
    while(true)
    {
        int data = get_data();
        data_queue.push(data);
    }
}
void consume()
{
    int data;
    while(true)
    {
        data_queue.wait_and_pop(data);
        process(data);
    }
}
```



## 子任务

有些时候，主线程会临时创建一些子线程用于执行子任务，当任务完成以后，子线程的使命就结束了，并且主线程可能需要获得它的执行结果，C++提供了`std::future`来支持线程返回。



### 异步返回

假设有一个长时间计算的任务，需要得到它的运算结果，但当前并不迫切需要。那么可以启动新的线程来执行这个计算，调用线程在其运算期间可以做其他工作，这些工作完成后，调用线程查询运算状态，并阻塞至结果返回。

由于`std::thread`并不提供直接接收返回值的机制，这里就需要使用`std::async`函数模板，来支持异步任务的启动和返回。`std::async`返回一个`std::future`对象，这个对象持有运算的结果，通过`get()`成员函数查询，如果future对象还未就绪，就会阻塞调用线程。

```c++
double sqrt_wrapper(double x)
{
    return sqrt(x);
}
int main()
{
    std::future<double> fut = std::async(sqrt_wrapper, 4);
    do_other_work(); 
    std::cout << fut.get() << std::endl;
}
```



### 任务抽象

在一些线程池等任务的管理中，可能有大量的子任务被调度运行，而要批量运行参数类型和返回类型都不同的任务函数，就需要对任务函数进行抽象（或者说是泛化）。

`std::packaged_task<>`是一个模板，将future和任务函数绑定，然后在合适的地方再调用。

```c++
double sqrt_wrapper(double x)
{
    return sqrt(x);
}

int main()
{
    std::packaged_task<double(double)> task(sqrt_wrapper);
    std::future<double> fut = task.get_future();
    std::thread th1(std::move(task), 4);
    do_other_work(); 
    std::cout << fut.get() << std::endl;
    th1.join();
}
```



### 异常

当`std::async`的任务函数调用抛出一个异常时，这个异常会存储在`future`中，在之后调用`get()`时被抛出。将函数打包进`std::packaged_task`后再进行调用，同样的事情也会发生。

```c++
double sqrt_throw(double x)
{
    if (x < 0) throw std::out_of_range("x<0");
    return sqrt(x);
}

int main()
{
    std::future<double> fut = std::async(sqrt_throw, -1);
    do_other_work(); 
    std::cout << fut.get() << std::endl;
}
```







### 使用std::promise

`std::promise`可以和一个`std::future`关联，等待线程可以使用`std::future`来阻塞自己，提供数据的线程可以使用`std::promise`来设置数据的值。可以说`promise-future`实现的是一个单工的通信方式（持有promise的为发送者，持有future的为接收者），并且`promise`只能被`get_future`一次，`future`也只能被`get`一次。

```c++
void func1(std::promise<int> &prom_)
{
    std::future<int> fut_ = prom_.get_future();
    do_other_work1();
    std::cout << fut_.get() << std::endl;
}

int main()
{
    std::promise<int> prom;
    std::thread th1(func1, std::ref(prom));
    do_other_work2();
    prom.set_value(10);
    th1.join();
}
```



`std::promise`也可以存储抛出异常到`future`中，这需要我们在`catch`块中调用`set_exception`来填充`future`。

```c++
double sqrt_throw(double x)
{
    if (x < 0) throw std::out_of_range("x<0");
    return sqrt(x);
}
std::promise<double> prom;
void func(double x)
{
    try{
    	prom.set_value(sqrt_throw(x));
    }catch(...){
        prom.set_exception(std::current_exception());
    }   
}

int main()
{
    std::future<double> fut = prom.get_future();
    std::thread th1(func, -1);
    do_other_work();
    std::cout << fut.get() << std::endl;
    th1.join();
}
```



### 多个线程的等待

`std::future`有局限性。很多线程等待时，只有一个线程能够获取结果，当其他线程也想要获取结果时，会导致数据竞争和未定义行为。

`std::shared_future`可以支持多个线程等待同一个事件结果。因为`std::future`是只移动的，所以其所有权可以在不同线程间转移，但不能被多个线程同时持有；而`std::shared_future`是可拷贝的。

`std::shared_future`可从`std::future`对象或者其它$std::shared_future$进行构造，并且只支持右值形式的`std::future`。

```c++
std::promise<int> prom;
std::future<int> fut(prom.get_future());
std::shared_future<int> sfut1(fut.share()); // 通过std::future的share成员函数返回std::share_future对象，再调用移动构造函数
std::shared_future<int> sfut2(std::move(fut)); // 必须以右值形式传入future
assert(!fut.valid());
```

如下例所示，线程th1和th2都将等待主线程填充`std::shared_future`对象，直到`set_value`被调用，这两个线程都会被唤醒，效果类似于`std::condition_variable::notice_all()`。

```c++
std::promise<int> prom;
std::shared_future<int> sfut(prom.get_future());
auto func = [sfut](){
std::cout << sfut.get() << std::endl;
};
std::thread th1(func);
std::thread th2(func);

prom.set_value(1);

th1.join();
th2.join();
```





## 限时等待

### C++ 时钟库

`std::chrono`是C++标准的时钟库，支持三个时钟概念：

+ Durations：时间段；
+ Time Points：时间点；
+ Clocks：时钟类型；



#### 时钟类型

`std::chrono`中常用的时钟有`system_clock`、`steady_clock`、`high_resolution_clock`。其中`high_resolution_clock`具备最小节拍周期（一秒多少个时钟周期），因此具有最高精度；`steady_clock`称为稳定时钟（时钟节拍均匀并且不可修改），能够确保调用`now()`返回的时间一定比前一次大



#### 时间段

`std::chrono::duration`表示时间段，有两个模板参数，`Rep`表示数值类型，如int、double等；`Period`是`std::ratio`类型，表示时间单位。

```c++
 template <class Rep, class Period = ratio<1> > class duration;
```

如下所示，C++已经定义了一些时间段类型。默认的`Period`时间单位为秒，而`std::ratio<3600,1>`就是3600秒；`std::ratio<1,1000>`就是毫秒。

![计算机科学/C++/assets/C++并发编程/0ccb4f5d2a81eff74872d8841c54df5a_MD5.png](/img/user/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6/C++/assets/C++%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/0ccb4f5d2a81eff74872d8841c54df5a_MD5.png)

粗粒度时间单位可以隐式转为细粒度时间，相反则不行，必须借助`std::chrono::duration_cast<>`来显式转换。时间段之间也支持四则运算，并通过`count`成员函数获取单位时间的数量，该数值类型和`Rep`一致。

```c++
std::chrono::minutes m(60);
std::chrono::seconds s(m); // 3600s
// std::chrono::hours h(m); // error
std::chrono::hours h = std::chrono::duration_cast<std::chrono::hours>(m); // 1h
auto d = m - s; // std::chrono::seconds
auto val = d.count(); // int64_t
```



#### 时间点

UNIX系统中，系统时间都是从1970年1月1日开始计时的，时间点（时间戳）就是指从1970年1月1日开始所经过的秒数。（关于UNIX时间戳，有一个著名的[2038年问题](https://baike.baidu.com/item/unix%E6%97%B6%E9%97%B4%E6%88%B3/2078227?fr=aladdin)，非常有趣）

时间点用`std::chrono::time_point<>`表示，第一个参数用来指定使用的时钟，第二个参数表示时间单位（特化的`std::chrono::duration`）。时间点最常被用来获取当前时间。

```c++
auto start = std::chrono::high_resolution_clock::now();
```

`std::chrono::time_point<>`不能够直接查看数值，可以通过`time_since_epoch()`和`system_clock::to_time_t()`来获取具体数值。

```c++
auto start = std::chrono::system_clock::now();
std::cout << std::chrono::system_clock::to_time_t(start) << std::endl; // unit:s
std::cout << start.time_since_epoch().count() << std::endl; // unit: nano
```

时间点之间可以进行相减，结果得到两个时间点的时间差；也可以对一个时间点加上一个时间段，得到一个新的时间点。

```c++
auto start = std::chrono::high_resolution_clock::now();
auto next_day = start + std::chrono::hours(24);
auto stop = std::chrono::high_resolution_clock::now();
std::cout << std::chrono::duration<double, std::ratio<1, 1000000>>(stop-start).count() << " us" << std::endl;
```



### 使用超时

一般的规律是`_until`成员函数实现时间点等待，`_for`成员函数实现时间段等待。

比如`std::future`既可以使用`wait_for`，又可以使用`wait_until`。

```c++
double sqrt_wrapper(double x)
{
    return sqrt(x);
}
int main()
{
    std::future<double> fut = std::async(sqrt_wrapper, 4);
    auto status = fut.wait_for(std::chrono::microseconds(100));
    // auto status = fut.wait_until(std::chrono::steady_clock::now() + std::chrono::microseconds(10));
    if (status == std::future_status::ready)
        std::cout << fut.get() << std::endl;
    else if (status == std::future_status::timeout)
        std::cout << "timeout!" << std::endl;
}
```



条件变量也可以实现超时等待，`wait_for`或者`wait_until`都有两个重载，一个只需要指定超时条件，另一个还可以传入`Predicate`。从返回值角度看，`cv_status`在超时结果上表述直观些，有`timeout`和`no_timeout`两个结果，但是使用该版本意味着我们需要在外部另加`Predicate`的判断逻辑，此时便要注意假唤醒的情况，坚决使用`while`而不是`if`。

```c++
template< class Rep, class Period >
std::cv_status wait_for( std::unique_lock<std::mutex>& lock,
                         const std::chrono::duration<Rep, Period>& rel_time);
template< class Rep, class Period, class Predicate >
bool wait_for( std::unique_lock<std::mutex>& lock,
               const std::chrono::duration<Rep, Period>& rel_time,
               Predicate pred);
```



我们改进之前“条件变量”一节中的生产消费代码，加入超时处理（当超过1s队列仍然为空时，结束线程）。

```c++
std::mutex m;
std::queue<int> data_queue;
std::condition_variable data_cond;
void produce()
{
    int len = 100;
    while(len--)
    {
        std::lock_guard<std::mutex> lk(m);
        int data = get_data();
        data_queue.push(data);
        data_cond.notify_one();
    }
}

void consume()
{
    while (true)
    {
        auto timeout = std::chrono::steady_clock::now() + std::chrono::seconds(1);
        std::unique_lock<std::mutex> lk(m);
        while(data_queue.empty()) // 使用while处理假唤醒
            if (data_cond.wait_until(lk, timeout) == std::cv_status::timeout)
                return; // 超时不再接收数据
        int data = data_queue.front();
        data_queue.pop();
        std::cout << data << std::endl;
        lk.unlock();
        process(data);  
    }  
}
```

假设我们使用了`if`来实现`data_queue.empty()`的判断，在多消费线程的情况下就会产生问题：比如一个生产者线程和两个消费者线程的情况，当第一个消费者线程调用`wait_until()`进入休眠后，生产者线程运行并生产了一个数据，然后唤醒第一个消费者线程，但此时第二个消费者线程先取得锁而运行，导致队列又为空，之后刚被唤醒的那个线程跳出了判断分支将无数据可消费。

然而，使用`while`就意味着这里不适合用`wait_for`，因为如此会使得线程陷入等待循环，永远无法结束。

除了以上这些限时等待的例子，可接受超时的函数还有很多，如下表所示。

![计算机科学/C++/assets/C++并发编程/f2157304852982df0cc2a04d290c12a7_MD5.png](/img/user/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6/C++/assets/C++%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/f2157304852982df0cc2a04d290c12a7_MD5.png)





# 基于锁的并发数据结构



## 栈

```c++
struct empty_stack: std::exception
{
    const char* what() const throw()
    {
        return "empty stack";
    }  
};

template<typename T>
class threadsafe_stack
{
public:
    threadsafe_stack(){}
    threadsafe_stack(const threadsafe_stack& other)
    {
        std::lock_guard<std::mutex> lock(m);
        data = other.data;
    }
    threadsafe_stack& operator=(const threadsafe_stack&) = delete;

    void push(T new_value)
    {
        std::lock_guard<std::mutex> lock(m);
        data.push(std::move(new_value));
    }
    void pop(T &value) // 使用引用形参
    {
        std::lock_guard<std::mutex> lock(m);
        if (data.empty()) throw empty_stack();
        value = std::move(data.top()); // 不需要再申请内存了
        data.pop();
    }
    std::shared_ptr<T> pop()
    {
        std::lock_guard<std::mutex> lock(m);
        if (data.empty()) throw empty_stack();
        std::shared_ptr<T> res(std::make_shared<T>(std::move(data.top()))); //即使内存分配失败，也没有执行pop，仍然可以补救
        data.pop();
        return res;
    }
    bool empty() const
    {
        std::lock_guard<std::mutex> lock(m);
        return data.empty();
    }
private:
    std::stack<T> data;
    mutable std::mutex m; // empty()是const函数， 要声明为可变的才能lock
};
```



## 队列



### 粗粒度



“同步操作”一节曾实现过并发队列，但是仍存在线程安全问题。具体来说如果在`wait_and_pop`函数中抛出了一个异常，例如构造新的`std::shared_ptr<>`对象出错，那么其他线程就会永世长眠。对此，有三种解决方案：

+ 将`data_cond.notify_one()`改为`data_cond.notify_all()`，但是这样会导致多个线程没有意义地醒来再睡去。
+ 有异常抛出时，在`wait_and_pop`捕获并调用`data_cond.notify_one()`。
+ 将`std::shared_ptr<>`对象的创建移到`push`中去，并且队列存储的数据类型改为指针类型。

采用第三种方式的改进如下。

```c++
template<typename T>
class threadsafe_queue
{
public:
    threadsafe_queue(){};
    void push(T new_value);

    // 尝试从队列中弹出数据，即使队列为空，也会直接返回（结果为false）
    bool try_pop(T &value);
    std::shared_ptr<T> try_pop();

    // 如果队列为空，将会等待队列有值时才返回
    void wait_and_pop(T &value);
    std::shared_ptr<T> wait_and_pop();

    bool empty() const;

private:
    std::queue<std::shared_ptr<T>> data_queue;
    std::condition_variable data_cond;
    mutable std::mutex m;
};

template<typename T>
void threadsafe_queue<T>::push(T new_value)
{
    std::shared_ptr<T> data(std::make_shared<T>(std::move(new_value)));
    std::lock_guard<std::mutex> lk(m);
    data_queue.push(data);
    data_cond.notify_one();
}

template<typename T>
bool threadsafe_queue<T>::try_pop(T &value)
{
    std::lock_guard<std::mutex> lk(m);
    if (data_queue.empty()) return false;
    value = std::move(*data_queue.front());
    data_queue.pop();
    return true;
}

template<typename T>
std::shared_ptr<T> threadsafe_queue<T>::try_pop()
{
    std::lock_guard<std::mutex> lk(m);
    if (data_queue.empty()) return std::make_shared<T>();
    std::shared_ptr<T> res = data_queue.front();
    data_queue.pop();
    return res;
}

template<typename T>
void threadsafe_queue<T>::wait_and_pop(T &value)
{
    std::unique_lock<std::mutex> lk(m);
    data_cond.wait(lk, [this]{return !data_queue.empty();});
    value = std::move(*data_queue.front());
    data_queue.pop();
}

template<typename T>
std::shared_ptr<T> threadsafe_queue<T>::wait_and_pop()
{
    std::lock_guard<std::mutex> lk(m);
    data_cond.wait(lk, [this]{return !data_queue.empty();});
    std::shared_ptr<T> res = data_queue.front(); // 直接返回指针对象，避免创建过程抛出异常导致其他线程睡眠
    data_queue.pop();
    return res;
}

template<typename T>
bool threadsafe_queue<T>::empty() const
{
    std::lock_guard<std::mutex> lock(m);
    return data_queue.empty();
}
```



### 细粒度

为了实现细粒度锁，就不能再借助STL中提供的队列了，我们可以通过链表来编写一个简单的队列，初步方案如下图所示，当队列不为空时，头尾指针都指向真实数据节点；当队列为空时，头尾指针都是空指针。

![计算机科学/C++/assets/C++并发编程/039744a168284f92963512166017958a_MD5.png](/img/user/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6/C++/assets/C++%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/039744a168284f92963512166017958a_MD5.png)

```c++
template<typename T>
class queue
{
private:
    struct node
    {
        T data;
        std::unique_ptr<node> next;
        node(T data_):data(std::move(data_)){}
    };
    std::unique_ptr<node> head;
    node* tail = nullptr; // unique_ptr独享被管理对象，不适合作为尾指针

public:
    queue(){}
    queue(const queue&) = delete;
    queue& operator=(const queue&) = delete;

    std::shared_ptr<T> try_pop() // 返回指针，避免接口合并导致的异常
    {
        if (!head) return std::shared_ptr<T>();
        std::shared_ptr<T> data_ptr(std::make_shared<T>(std::move(head->data)));
        std::unique_ptr<node> const old_head(std::move(head));
        head = std::move(old_head->next);
        if (old_head.get() == tail) tail = nullptr; // pop最后一个节点时
        return data_ptr;
    }

    void push(T new_value)
    {
        std::unique_ptr<node> p(new node(std::move(new_value)));
        node* const new_tail = p.get();
        if (tail)
        {
            tail->next = std::move(p);
        }
        else
        {
            head = std::move(p);
        }
        tail = new_tail;
    }

};
```

使用`std::unique_ptr`的好处在于可以自动析构对象，但是这也导致了对象只能被一个unique指针独享的结果，因此`tail`必须使用普通指针，因为`tail`指向的对象也被前一个节点所指向。

`try_pop`函数考虑了接口合并后可能因为拷贝操作而引起的异常，所以采用了返回共享指针的方案。

但是，该方案最大的问题在于`try_pop`和`push`既能访问`tail`也能访问`head`，甚至`old_head->next`和`tail->next`访问的都可以是同一对象。换句话说，没办法通过这个方案实现细粒度锁。



改进的版本采用分离数据的方法，增加了虚拟节点，当队列为空时，头尾指针指向同一个虚拟节点；当队列不空时，头指针指向第一个真实节点，尾指针指向虚拟节点。

```c++
template<typename T>
class threadsafe_queue
{
private:
    struct node
    {
        std::shared_ptr<T> data; // 为了避免pop时因为构造指针对象抛出异常导致其他线程永远睡去
        std::unique_ptr<node> next;
    };
    std::unique_ptr<node> head;
    node* tail = nullptr; // unique_ptr独享被管理对象，不适合作为尾指针
    std::mutex head_mutex;
    std::mutex tail_mutex;

    node* get_tail()
    {
        std::lock_guard<std::mutex> tail_lock(tail_mutex);
        return tail;
    }

    std::unique_ptr<node> pop_head()
    {
        std::lock_guard<std::mutex> head_lock(head_mutex);
        if (head.get() == get_tail()) // 访问tail需要加锁
        {
            return nullptr;
        }
        std::unique_ptr<node> old_head = std::move(head);
        head = std::move(old_head->next); // pop后head指向第一个真实数据节点
        return old_head;
    }

public:
    threadsafe_queue():head(new node), tail(head.get()){} // 队列为空时，head和tail指向一个虚拟节点
    threadsafe_queue(const threadsafe_queue&) = delete;
    threadsafe_queue& operator=(const threadsafe_queue&) = delete;

    std::shared_ptr<T> try_pop() // 返回指针，避免接口合并导致的异常
    {
        std::unique_ptr<node> old_head = pop_head();
        return old_head?old_head->data:std::shared_ptr<T>();
    }

    void push(T new_value)
    {
        std::shared_ptr<T> new_data(std::make_shared<T>(std::move(new_value)));
        std::unique_ptr<node> p(new node); 
        node* const new_tail = p.get();
        std::lock_guard<std::mutex> tail_lock(tail_mutex);
        tail->data = new_data;
        tail->next = std::move(p);
        tail = new_tail; // push后，tail指向一个新的虚拟节点
    }

};
```

首先，就像前文提及的那样，我们将指针对象的构造从``pop`函数中移到`push`中，以避免其他线程永久沉睡的情况。

`push`函数中只有对`tail`的访问，因此只需要加`tail_mutex`锁；`try_pop`函数则有意思一些，先是获取了`head_mutex`，然后再通过`get_tail`获取`tail_mutex`。尾锁保证`get_tail`要么在`push()`之前被调用，返回旧的尾节点，要么就是在`push()`之后调用，返回新的尾节点。

其次，该版本的实现也是异常安全的，并且不会产生死锁问题。

需要注意的是`tail_mutex`应当在`head_mutex`之前被获取，比如下面这个例子就存在缺陷，例如，当线程A执行完`get_tail()`后，一直等待线程B释放`head_mutex`，期间线程C执行了多次pop；当线程A终于取到锁时，`old_tail`所指向的节点可能早就被pop掉了。

```c++
std::unique_ptr<node> pop_head() // 存在缺陷的实现
{
    node* const old_tail = get_tail();
    std::lock_guard<std::mutex> head_lock(head_mutex);
    if (head.get() == old_tail)
    {
        return nullptr;
    }
    std::unique_ptr<node> old_head = std::move(head);
    head = std::move(old_head->next); // pop后head指向第一个真实数据节点
    return old_head;
}
```



接下来的任务就是实现可上锁和等待的线程安全队列了，其实现如下。

```c++
template<typename T>
class threadsafe_queue
{
private:
    struct node
    {
        std::shared_ptr<T> data; // 为了避免pop时因为构造指针对象抛出异常导致其他线程永远睡去
        std::unique_ptr<node> next;
    };
    std::unique_ptr<node> head;
    node* tail = nullptr; // unique_ptr独享被管理对象，不适合作为尾指针
    std::mutex head_mutex;
    std::mutex tail_mutex;
    std::condition_variable data_cond;

    node* get_tail()
    {
        std::lock_guard<std::mutex> tail_lock(tail_mutex);
        return tail;
    }

    // 删除并返回头节点
    std::unique_ptr<node> pop_head()
    {
        std::unique_ptr<node> old_head = std::move(head);
        head = std::move(old_head->next); // pop后head指向第一个真实数据节点
        return old_head;
    }

    // 等待队列有数据并弹出
    std::unique_lock<std::mutex> wait_for_data()
    {
        std::unique_lock<std::mutex> head_lock(head_mutex);
        data_cond.wait(head_lock, [&]{return head.get() == get_tail();});
        return std::move(head_lock);
    }

    std::unique_ptr<node> wait_pop_head()
    {
        std::unique_ptr<std::mutex> head_lock(wait_for_data());
        return pop_head();
    }

    std::unique_ptr<node> wait_pop_head(T& value)
    {
        std::unique_ptr<std::mutex> head_lock(wait_for_data());
        value = std::move(*head->data);
        return pop_head();
    }

public:
    threadsafe_queue():head(new node), tail(head.get()){} // 队列为空时，head和tail指向一个虚拟节点
    threadsafe_queue(const threadsafe_queue&) = delete;
    threadsafe_queue& operator=(const threadsafe_queue&) = delete;

    void push(T new_value);

    bool try_pop(T &value);
    std::shared_ptr<T> try_pop();

    void wait_and_pop(T &value);
    std::shared_ptr<T> wait_and_pop();

    bool empty();
};

template<typename T>
void threadsafe_queue<T>::push(T new_value)
{
    std::shared_ptr<T> new_data(std::make_shared<T>(std::move(new_value)));
    std::unique_ptr<node> p(new node); 
    node* const new_tail = p.get();
    {
        std::lock_guard<std::mutex> tail_lock(tail_mutex);
        tail->data = new_data;
        tail->next = std::move(p);
        tail = new_tail; // push后，tail指向一个新的虚拟节点
    }
    data_cond.notify_one(); // 为什么需要先解锁tail_mutex
}

template<typename T>
std::shared_ptr<T> threadsafe_queue<T>::wait_and_pop()
{
    std::unique_ptr<node> const old_head = wait_pop_head();
    return old_head->data;
}

template<typename T>
void threadsafe_queue<T>::wait_and_pop(T& value)
{
    std::unique_ptr<node> const old_head = wait_pop_head(value);
}

template<typename T>
std::shared_ptr<T> threadsafe_queue<T>::try_pop()
{
    std::lock_guard<std::mutex> head_lock(head_mutex);
    if (head.get() == get_tail())
    {
        return std::shared_ptr<T>();
    }
    return pop_head();
}

template<typename T>
bool threadsafe_queue<T>::try_pop(T& value)
{
    std::lock_guard<std::mutex> head_lock(head_mutex);
    if (head.get() == get_tail())
    {
        return false;
    }
    value = *head->data;
    pop_head();
    return true;  
}

template<typename T>
bool threadsafe_queue<T>::empty()
{
    std::lock_guard<std::mutex> head_lock(head_mutex);
    return head.get() == get_tail();
}
```



## 查询表

基本操作：

+ 添加一对“键值-数据”；
+ 修改指定键值的数据；
+ 删除一对“键值-数据”；
+ 通过给定键值，获取对应的数据；
+ 判断容器是否为空；

+ 建立快照（返回当前状态的map）；

```c++
template<typename Key, typename Value, typename Hash=std::hash<Key>>
class threadsafe_lookup_table
{
private:
    class bucket_type
    {
    private:
        typedef std::pair<Key, Value> bucket_value;
        typedef std::list<bucket_value> bucket_data;
        typedef typename bucket_data::iterator bucket_iterator;
        bucket_iterator find_entry(const Key& key)
        {
            return std::find_if(data_.begin(), data_.end(), 
                [&](const bucket_value& item){
                    return item.first == key;
            });
        }

    public:        
        mutable std::shared_mutex mutex;
        bucket_data data_;
        bucket_type(){}
        Value find(const Key& key, const Value& default_value) 
        {
            std::shared_lock<std::shared_mutex> lock(mutex);
            const bucket_iterator iter = find_entry(key);
            return iter==data_.end()?default_value:iter->second;
        }

        void add_or_update(const Key& key, const Value& value)
        {
            std::unique_lock<std::shared_mutex> lock(mutex);
            const bucket_iterator iter = find_entry(key);
            if (iter == data_.end())
            {
                data_.push_back(bucket_value(key, value));
            }
            else
            {
                iter->second = value;
            }
        }

        void remove(const Key& key)
        {
            std::unique_lock<std::shared_mutex> lock(mutex);
            const bucket_iterator iter = find_entry(key);
            if (iter != data_.end())
                data_.erase(iter);
        }
    };

    Hash hasher_;
    std::vector<bucket_type> buckets;

    bucket_type& get_bucket(const Key& key) 
    {
        const std::size_t bucket_index = hasher_(key)%buckets.size();
        return buckets[bucket_index];
    }

public:
    threadsafe_lookup_table(unsigned num_buckets=19, Hash hasher=Hash()):
        buckets(num_buckets), hasher_(hasher){}
    
    threadsafe_lookup_table(const threadsafe_lookup_table& other)=delete;
    threadsafe_lookup_table& operator=(
        const threadsafe_lookup_table& other)=delete;
    
    Value find(const Key& key, 
        const Value& default_value=Value()) 
    {
        return get_bucket(key).find(key, default_value);
    }

    void add_or_update(const Key& key, const Value& value)
    {
        get_bucket(key).add_or_update(key, value);
    }

    void remove(const Key& key)
    {
        get_bucket(key).remove(key);
    }

    std::map<Key, Value> get_map()
    {
        std::vector<std::shared_lock<std::shared_mutex>> locks;
        for (auto &b : buckets)
            locks.emplace_back(b.mutex);
        std::map<Key, Value> res;
        for (auto &b : buckets)
            for (auto it = b.data_.begin(); 
                it != b.data_.end(); it++)
                res.insert(*it);
        return res;
    }
};
```



## 链表

基本操作：

+ 向链表添加一个元素；
+ 当某个条件满足时，从链表删除某个元素；
+ 当某个条件满足时，从链表查找某个元素；
+ 当某个条件满足时，更新链表中的某个元素；
+ 将容器中链表的每一个元素复制到另一个元素；



```c++
template<typename T>
class threadsafe_list
{
    struct node
    {
        std::shared_mutex m;
        std::shared_ptr<T> data;
        std::unique_ptr<node> next;
        node(){}
        node(const T& value):data(std::make_shared<T>(value)){}

    };
    node head;

public:
    threadsafe_list(){}
    threadsafe_list(const threadsafe_list&) = delete;
    threadsafe_list& operator=(const threadsafe_list&) = delete;

    void push_front(const T& value)
    {
        std::unique_ptr<node> new_node(new node(value));
        std::unique_lock<std::shared_mutex> lk(head.m); // 在某个节点之后插入，要获取该节点的锁
        new_node->next = std::move(head.next);
        head.next = std::move(new_node);
    }

    template<typename Function>
    void for_each(Function f)
    {
        node* cur = &head;
        std::shared_lock<std::shared_mutex> lk(cur->m);
        while(node* const next = cur->next.get())
        {
            std::shared_lock<std::shared_mutex> next_lk(next->m); // 按照链表顺序依次加锁
            lk.unlock();
            f(*next->data); // 传出拷贝值
            cur = next;
            lk = std::move(next_lk); // 转移所有权
        }
    }

    template<typename Predicate>
    std::shared_ptr<T> find_first_if(Predicate p)
    {
        node* cur = &head;
        std::shared_lock<std::shared_mutex> lk(cur->m);
        while(node* const next = cur->next.get())
        {
            std::shared_lock<std::shared_mutex> next_lk(next->m);
            lk.unlock();
            if (p(*next->data))
            {
                return next->data;
            }
            cur = next;
            lk = std::move(next_lk);
        }
        return std::shared_ptr<T>();
    }

    template<typename Predicate>
    void remove_if(Predicate p)
    {
        node* cur = &head;
        std::unique_lock<std::shared_mutex> lk(cur->m);
        while(node* next = cur->next.get())
        {
            std::unique_lock<std::shared_mutex> next_lk(next->m);
            if (p(*next->data))
            {
                std::unique_ptr<node> old_next = std::move(cur->next); // 用于释放内存
                cur->next = std::move(next->next); // 删除节点
                next_lk.unlock(); // 继续持有cur的锁
            }
            else
            {
                lk.unlock(); // 同for_each
                cur = next;
                lk = std::move(next_lk);
            }
        }
    }
    
};
```





# 线程池

需要考虑的设计问题：

+ 可使用的线程数量；
+ 高效的任务分配方式；
+ 是否需要等待一个任务完成；



## 简单实现

构建`thread_pool`对象时，会预先创建一定数量的线程。用户通过`submit`将任务函数添加到任务队列`work_queue`中。`worker_thread`是线程函数，会循环查询队列，如果没有任务则当前线程让出时间片，如果有任务则执行。

简单实现版本只支持没有返回值和输入参数的任务函数。

```c++
class thread_pool
{
private:
    std::atomic_bool done;
    threadsafe_queue<std::function<void()>> work_queue; // 任务队列
    std::vector<std::thread> threads;  // 工作线程

    void worker_thread()
    {
        while(!done)
        {
            std::function<void()> task;
            if (work_queue.try_pop(task)) // 从任务队列取出一个任务执行
            {
                task();
            } 
            else
            {
                std::this_thread::yield();  // 放弃时间片，下次线程运行时可以执行其他任务
            }
        }
    }
public:
    thread_pool():done(false)
    {
        const unsigned thread_amount = std::thread::hardware_concurrency();
        try
        {
            for (int i = 0; i < thread_amount; i++)
            {
                threads.push_back(std::thread(&thread_pool::worker_thread, this));
            }
        }
        catch(...)
        {
            done = true;
            throw;
        }
    }

    ~thread_pool()
    {
        done = true;                        // 线程池对象析构时，如果任务队列非空，也不会继续执行
        for (auto &th : threads)           // 利用析构函数汇入
            if (th.joinable())
                th.join();
    }

    template<typename FunctionType>
    void submit(FunctionType f)
    {
        work_queue.push(std::function<void()>(f));
    }

};
```



## 任务返回值

