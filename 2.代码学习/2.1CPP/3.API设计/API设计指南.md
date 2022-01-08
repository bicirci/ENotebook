---
fontsize: 12pt
linkcolor: blue
urlcolor: green
citecolor: cyan
filecolor: magenta
toccolor: red
papersize: A4
documentclass: article
CJKmainfont: 'WenQuanYi Micro Hei Mono'
title: 'API Design'
author: 'Eric Luo'
date: 'Dec 3, 2021'
output:
    pdf_document: {path: ./output/API设计指南.pdf, toc: true, toc_depth: 3, highlight: tango, latex_engine: xelatex, template: ./book.tex}
---  
  
  
#  API设计指南
  
  
  
##  2.优秀API设计法则  
  
  
###  2.1 对问题的抽象  
  
  
* API需要对所解决的问题进行高层的抽象, 并且接口要易于理解, 形成体系.  
  
* UML使用说明  
![API_design-2021-11-18-uml说明](https://cdn.jsdelivr.net/gh/bicirci/PicBed@master/MarkDown/API_design-2021-11-18-uml说明.png )  
  
* 模型的抽象是紧跟需求的, 比如person 可以有多个address和多个telephone number, 那么就需要单独的Address和Telephone Number的抽象类, 包含到person中.  
  
###  2.2 隐藏实现
  
  
隐藏实现是为了将今后可能发生变化的部分隐藏起来, 从而使得对外的接口保持稳定, 变化只发生在内部.  
  
* 尽量减少回调和hook的使用, 这意味着将控制权交给用户, 从而使得今后的接口升级和优化变得更难.  
  
隐藏实现的方法:  
  
1. 隐藏数据成员
所有data member应该设置为private做隐藏, 对数据的读写通过setter/getter实现:
  
* 有利于设置数据时的validation
* 有利于lazy evaluation, 实际读取时才进行数据的计算
* 有利于caching, 比如从文件读取的数据, 读一次后就可cache起来
* 有利于打点
* 有利于模块间的联动触发
* 有利于调试打印
* 有利于多线程的数据安全(读取时可加锁)
* 有利于同模块内其他基于这个数据的接口的稳定性
  
protected也是不安全的, 可以被用户继承而访问. 继承是对抽象的一种妥协!
  
要注意的是 setter 和 getter 在对性能要求极高的场景下, 可能需要进行inline的处理(函数的入栈出栈都有损耗).  
  
2. 隐藏成员函数
  
* 避免将内部数据的指针或引用以非const的形式暴露给用户
* 由于C++编译时需要知道类大小的限制, 导致所有private的函数用户也是能看到的, 且其需要的头文件也都对用户开放. 可以通过Pimpl 的 idiom来进行优化, 后文会详细介绍.  
* 对于Pimpl idiom无法使用的情况, 则尽量将私有函数转化为static函数, 扔到.cpp里. (例如不需要访问该类内其他私有函数/数据的函数, 都可以变成static)
  
###  2.3 最简化实现
  
  
在接口设计时要了解用户的实际需求, 并提供最为简化的接口. 越简化的接口越容易被用户使用.  
  
1. 设计中保持克制
  
* 作为工程师都是想提供更为通用的接口, 从而便于今后的扩展. 但其实要克制这种欲望. 因为向API增加内容的难度要远远小于向API删除内容.但内部还是可以保留通用性, 只是对外的public接口要保持克制.  
  
2. 减少虚函数的使用
  
虚函数类似C中的回调, 是提供用户自定义的一种方式. 某些设计模式可能大量使用了虚函数, 但这些文件不应该作为接口层暴露给用户.
  
* 会造成fragile base class problem. 用户自由定义的虚函数实现, 可能基于base的某一特定点. 对base的这一点的修改可能会造成用户的虚函数override的失效.
  
* 如果不得不使用虚函数, 那么要遵循以下原则:
  
a. 如果api提供了虚函数, destructor也需要设置成virtual, 这样用户才能释放自己添加的资源
  
b. 提供明确的文档指明该虚函数需要实现哪些必要的内部调用
  
c. 不要在constructor 和 destructor内调用虚函数, 这里的调用不会实现多态, 固定调用到base的实现或者报错.
  
3. 简化和易用性的平衡
  
想实现简化和易用性的平衡, 可以通过分离接口来实现. 对于需要自由度高的用户, 提供一层核心api接口. 而对于其他需要开箱即用的用户, 则可以在此之上再封装一层简化接口. 甚至可以编译成不同的库文件, 提供给用户.  
但这里的core api依然要满足minimal的要求.  
  
###  2.4 接口的易用性
  
  
1.如果一个接口有==多个类型相同==的参数, 就要考虑到用户有可能搞混顺序的问题. 可以通过数据封包class, 或者将简单的flag类参数type变为enum来实现.  
  
``` c
result FindString(text, true, false);
result FindString(text, FORWARD, CASE_INSENSITIVE)
```
  
第二种方式明显更不容易出错.
  
```C++
class Date
{
public:
Date(int year, int month, int day);
...
};
  
class Date
{
public:
Date(const Year &y, const Month &m, const Day &d);
...
};
  
class Month
{
public:
explicit Month(int m) : mMonth(m) {}
int GetMonth() const { return mMonth; }
static Month Jan() { return Month(1); }
static Month Feb() { return Month(2); }
static Month Mar() { return Month(3); }
static Month Apr() { return Month(4); }
static Month May() { return Month(5); }
static Month Jun() { return Month(6); }
static Month Jul() { return Month(7); }
static Month Aug() { return Month(8); }
static Month Sep() { return Month(9); }
static Month Oct() { return Month(10); }
static Month Nov() { return Month(11); }
static Month Dec() { return Month(12); }
private:
int mMonth;
};
  
class Day //或Year
{
public:
explicit Day(int d) : mDay(d) {}
int GetDay() const { return mDay; }
private:
int mDay;
};
  
Date birthday(Year(1976), Month::Jul(), Day(7)); //这种调用明显更为直观, 且对格式的限制更为严格, 不会出现13月这种情况
  
```
  
2. 接口的命名需要直观, 减少缩写的使用, 且保持规则的统一. 比如类似的接口, 使用了begin 和 end, 就不要再出现start和finish. read() 和write()如果要读取handle, 那么就都放在第一位, 而不要一个放前面, 一个放后面.
  
3. 通过template实现接口的相似性.
  
``` c++
template <typename T>
class Coord2D
{
public:
Coord2D(T x, T y) : mX(x), mY(y) {};
T GetX() const { return mX; }
T GetY() const { return mY; }
void SetX(T x) { mX x; }
void SetY(T y) { mY y; }
void Add(T dx, T dy) { mX þ dx; mY þ dy; }
void Multiply(T dx, T dy) { mX * dx; mY * dy; }
private:
T mX;
T mY;
};
  
//Coord2D<int>, Coord2D<float>, and Coord2D<double> 都有相同的使用方式
```
  
4. 接口的数据独立性. 单一method所修改的数据,应尽量减少对其他method的影响, 也尽量不要依赖于其他method会修改的数据.  
  
5. 内存稳定性. C++的内存使用常会出现, Null dereferencing(对nullpty进行* 或 ->), double freeing, mixing allocators(malloc 搭配 delete), incorrect array deallocation(对array 应该用delete[] ), memery leaks. 尽量使用智能指针.
RAII也是为了这一点而创造的, 最据代表性的示例就是lock_gurad. 它避免考虑release_lock的操作,从而更具有稳定性.  
  
6. 平台共存. 如果代码依据不同的平台有不同的实现, 不要把用于区分平台的宏暴露给用户, 即接口要统一, 宏要封装在.cpp的代码中.  
  
###  2.5 松耦合
  
  
合格的程序应该是高内聚, 松耦合的. 内聚意味着单一模块高度聚合了需要实现的所有功能, 松耦合则意味着不同模块之间的联系要尽可能的少, 即如果AB是相关连的, 对A的修改会多大程度的影响到B.  
  
1. 一定要尽可能的避免循环依赖dependency cycle的情况. A include B, B include A.
  
2. 在可能的情况下使用forward declaration, 减少对类的内部实现的依赖
  
3. 在提供了一个类的最基础接口后, 后续更上层的接口, 就要开始考虑作为类的成员函数是否合适. 如果可以通过一个全局函数实现, 那就尽量使用全局函数.
  
``` c++
// myobject.h
class MyObject
{
public:
std::string GetName() const;
...
protected:
...
private:
std::string mName;
...
};
void PrintName(const MyObject &obj);
```
  
PrintName可以独立为全局函数, 就尽量独立出来. 更好的做法, 甚至是可以通过namespace或者为PrintName新建一个MyObjectHelper的类
  
4. 如果两个模块之间的联系仅仅只是功能上的调用, 或可复制的数据引用, 那么可以通过拷贝/重复相同代码的方式增加解耦性(代码的简洁和解耦, 优先选择解耦的需求). 即两个模块之间尽量通过参数传递, 而非类的直接依赖.  
  
5. manager class. 当你有多个相类似, 但又不完全相同的类时, 接口层和底层之间最好再添加一层manager class用来进行接口的抽象统一,和关系的解耦.  
![API_design-2021-11-23-2021-11-23_10-09](https://cdn.jsdelivr.net/gh/bicirci/PicBed@master/MarkDown/API_design-2021-11-23-2021-11-23_10-09.png )  
变为  
![API_design-2021-11-23-2021-11-23_10-10](https://cdn.jsdelivr.net/gh/bicirci/PicBed@master/MarkDown/API_design-2021-11-23-2021-11-23_10-10.png )  
  
6. 信号交互的处理 - 回调, Observers 和 Notifications
  
处理交互类的问题, 不管采用哪种方案, 都要先注意以下几点:
a. reentrancy重入.当接口通过回调将处理权交给用户时, 很有可能用户在这个回调中又调用了某些接口. 比如, 当处理一个对象队列时, 每处理一个对象就执行一次回调, 而用户在这个回调中有可能调用增删改查改变了这个对象队列.此时, 就要考虑这种重入情况怎样保持一致性.  
  
b. 应该提供取消注册关系的接口, 让用户可以管理注册的回调的生命周期. 这样当用户对象被删除前, 可以主动切断与接口的联系, 防止接口继续向错误的空地址发信息.  
  
c. 接口应该能让用户明确的了解注册的回调会在何时被调用. Cocoa API提供了willChange 和 didChange接口明确的告诉用户是事件发生前触发回调还是事件发生后.  
  
回调函数: 回调函数的本意是基于依赖关系, 底层不应依赖于上层的代码, 因此需要通过回调的方式, 来获得上层的功能.  
回调函数的现代发展, 则是通过functor和closure等类似的实现, 将数据和功能做打包封装.
  
Observers: 更为面向对象的实现方式, 后文会更详细的说明
  
Notifications: 相比较observer是一种更为解耦的实现. 被调用的对象不是直接在触发者内部保存, 而是通过一个中间层的消息中心, 发送者并不了解任何接收者的信息. 最典型的实现是 signals and slots.
  
###  2.6 接口的稳定
  
  
接口一定要考虑到不同版本之间的兼容性.
  
##  3. 设计模式
  
  
为了设计出高效的API, 许多设计模式是必不可少的, 这里只介绍对C++ 的API封装最为重要的一些模式.  
  
* pimpl idom范式 : 用于隐藏私有数据成员和接口, 将其转移到.cpp文件中.  
* Singleton and Factory Method : Singleton用于全局唯一的对象, Factory Method用于生成derived class.  
* Proxy, Adapter and Facade : 这几种都是对原有接口进行再次封装时用到的方法, 前两种用于对单一class的重新封装,Facade则是对多个class接口的统一封装.  
* Observer : 用于解耦有交互关系的两个接口类.  
  
###  3.1 Pimpl Idiom (Pointer to implementation)
  
  
利用forward declaration 和 一个private 的指针, 实现隐藏数据的目的.  
下面通过一个例子说明. AutoTimer的作用是在销毁时自动打印自身的生命周期.  
使用pimpl idiom 之前
  
``` C++
// autotimer.h
#ifdef _WIN32
#include <windows.h>
#else
#include <sys/time.h>
#endif
#include <string>
class AutoTimer
{
public:
/// Create a new timer object with a human-readable name
explicit AutoTimer(const std::string &name);
/// On destruction, the timer reports how long it was alive
~AutoTimer();
  
private:
// Return how long the object has been alive
double GetElapsed() const;
std::string mName;
#ifdef _WIN32
DWORD mStartTime;
#else
struct timeval mStartTime;
#endif
};
  
```
  
经pimpl idiom重构之后
======>
  
``` c++
// autotimer.h
#include <string>
class AutoTimer
{
public:
explicit AutoTimer(const std::string &name);
~AutoTimer();
private:
class Impl;
Impl *mImpl;
};
  
```
  
``` c++
// autotimer.cpp
#include "autotimer.h"
#include <iostream>
#if _WIN32
#include <windows.h>
#else
#include <sys/time.h>
#endif
class AutoTimer::Impl
{
public:
double GetElapsed() const
{
#ifdef _WIN32
return (GetTickCount() - mStartTime) / 1e3;
#else
struct timeval end_time;
gettimeofday(&end_time, NULL);
double t1 = mStartTime.tv_usec / 1e6 + mStartTime.tv_sec;
double t2 = end_time.tv_usec / 1e6 + end_time.tv_sec;
return t2 - t1;
#endif
}
std::string mName;
#ifdef _WIN32
DWORD mStartTime;
#else
struct timeval mStartTime;
#endif
};
  
AutoTimer::AutoTimer(const std::string &name) :
mImpl(new AutoTimer::Impl())
{
mImpl->mName = name;
#ifdef _WIN32
mImpl->mStartTime = GetTickCount();
#else
gettimeofday(&mImpl->mStartTime, NULL);
#endif
}
AutoTimer::~AutoTimer()
{
std::cout << mImpl->mName << ": took " << mImpl->GetElapsed()
<< " secs" << std::endl;
delete mImpl;
mImpl = NULL;
}
  
```
  
可以看到, 由于头文件中没有了底层结构体的定义, 所以不需要再进行platform-macro的判断以及相关include, 因此简洁了许多. 这种简洁带来的优点, 是远远大于pimpl idiom的额外代码的付出的.  
  
使用pimpl idiom, 有几点需要注意的:
  
1. 需要在构造函数生成pimpl对象, 在析构函数释放pimpl对象.  
2. 尽量将pimpl类在private段进行声明.  除非同一文件的其他类, 或者对外的全局接口必须访问到pimpl类内的成员函数, 就需要在public段进行声明.  
3. pimpl类如果想访问到master类的公共函数, 需要通过额外的指针传递实现.  
4. 如果类内存在private的virtual method, 用户有override的需求, 那么就不能将其移动到pimpl内实现.  
  
####  pimpl类的拷贝
  
  
需要额外注意的是, 由于编译器会自动生成copy constructor和assignment的函数, 而这些默认生成的函数在处理pimpl时只会进行浅拷贝, 这就会造成两个对象使用同一个指针的问题, 从而出现double free的问题.  
  
如果确实没有拷贝的需求,解决方法比较简单, 只要将copy constructor和assignment在private段进行声明即可, 这样就可以阻止编译器自动生成拷贝的实现.  
  
修改后的版本:
  
``` c++
#include <string>
class AutoTimer
{
public:
explicit AutoTimer(const std::string &name);
~AutoTimer();
private:
// Make this object be non-copyable
AutoTimer(const AutoTimer &);
const AutoTimer &operator=(const AutoTimer &);
  
class Impl;
Impl *mImpl;
};
```
  
####  Pimpl和智能指针
  
  
如果想避免手动创建和释放pimpl类,可以借用智能指针. 但要注意的是, 智能指针也没法解决拷贝的问题.  
  
用智能指针封装的pimpl
  
``` c++
#include <boost/scoped_ptr.hpp>
#include <string>
class AutoTimer
{
public:
explicit AutoTimer(const std::string &name);
~AutoTimer();
private:
class Impl;
boost::scoped_ptr<Impl> mImpl;
};
```
  
(由于boost::scoped_ptr是 non-copyable, 所以编译器也不会自动生成copy constructor和assignment)
  
####  Pimpl的优缺点
  
  
优点:
有利于解耦, 以及更快的编译速度(头文件依赖的减少). 还可实现lazy Allocation等操作.
  
缺点:
增加代码的复杂度, 对const的控制不再严格(const method只能限制到当前类内的成员变量不被修改, 而对pimpl指针指向的对象是不会做检查的, 也就是在const method内也可以修改pimpl的数据)
  
###  3.2 Singletons
  
  
因为比较常用, 这里只介绍之前没注意的重点:
  
1. 将constructor 和 copy con/assignment 都放到private段  
2. destructor也最好放在private, 避免用户调用 delete 强制删除  
3. GetInstance() 应该优先返回reference, 因为返回指针, 如果没有设置private的destructor, 用户还是可以通过delete删除指针.  
  
针对Singletons的==多线程使用==, 主要是在初次调用分配内存空间的时候可能存在竞争问题.
解决的思路大致有两类:
  
1. 如果想保持lazy allocation的要求,使用时才分配内存, 那么就必须要通过mutex来进行加固.  
  
``` c++
static Mutex mutex;
Singleton &Singleton::GetInstance()
{
    ScopedLock(&mutex); // unlocks mutex on function exit
    static Singleton instance;
    return instance;
}
```
  
如果想优化每次获取Instance时mutex的损耗, 那么就需要使用Double check的方式, 但这种方式在某些编译器依旧是不保险的.
  
``` c++
static Mutex mutex;
Singleton &Singleton::GetInstance()
{
    static Singleton *instance = NULL;
        if (! instance) // check #1
        {
            ScopedLock(&mutex);
            if (! instance) // check #2
            {
            instance = new Singleton();
            } 
        }
    return *instance;
}
  
```
  
2. 如果可以接受在初始化阶段就先分配好该接口的内存, 那么就简单一些, 可以使用静态分配的方法,避开线程的问题. 或者单独写一个所有接口统一的initialize的接口, 在线程开启前就分配好.  
  
``` c++
static Singleton &foo = Singleton::GetInstance();//因为是全局的, 会在main函数执行前就分配内存, 而这一阶段通常还没有线程开启
  
Singleton &Singleton::GetInstance()
{
    static Singleton instance;
    return instance;
}
```
  
####  关于单例的多态
  
  
单例的多态是希望实现的可以复用代码, 调用GetInstance()获得不同的单例对象.  
由于static的函数是无法override的, 没法直接通过virtual返回不同的子类对象. 所以单例是无法直接通过override来实现多态的, 但可以通过模板来实现, 从而实现可以复用的GetInstance代码, 并且返回的对象又是可以多态的.  
  
``` C++
template <typename T>
class QLogger
{
public:
    static T& GetInstance() 
    {
        static T logger;
        return logger;
    }
protected:
    QLogger() {};
    virtual ~QLogger() {};
  
private:
    QLogger(const QLogger&);
    QLogger& operator =(const QLogger&);
};
  
class name : public QLogger<name>
{
public:
    void interface_method();
private:
    name();
    ~name() override{};
    friend class QLogger<name>;  //注意这里的friend class设置, 模板内生成static对象时, 需要访问到构造函数
};
  
```
  
###  3.3 Dependency Injection依赖注入
  
  
如果你的接口类是依赖一个更为底层的类, 且不对其保留所有权, 那么这通常需要使用指针. 对这一指针的初始化, 如果是在该类的外部进行初始化, 而把指针直接传递进来, 从而避免直接在constructor里进行对象内存的创建, 就称为依赖注入. 这种方法可以提高解耦程度, 并且常和Singleton搭配使用.  
  
不使用依赖注入时:
  
```C++
class MyClass
{
    MyClass() :
        mDatabase(new Database("mydb", "localhost", "user", "pass"))
        {}
private:
    Database *mDatabase;
};
```
  
使用依赖注入
  
```C++
class MyClass
{
    MyClass(Database *db) :
        mDatabase(db)
        {}
private:
    Database *mDatabase;
};
```
  
而在外部通常有一个专门的dependency container的类(类似扫地机用过的XXX_Man类), 用来调用singleton的GetInstance, 并将这个指针cache下来, 分配给各个需要该指针的类MyClass. 当然也可以进行非singleton的类的指针的统一管理和分配.  
  
###  3.4 Monostate 单状态类
  
  
如果对全局唯一这一要求没有特别强烈, 或者某个类没有数据的保存, 仅仅进行接口调用, 那么就可以变成Monostate类, 其实就是如果需要数据, 就放在private区的static或者直接在cpp里定义静态全局变量, 从而可以让多个对象访问相同的数据.  
  
``` c++
// monostate.h
class Monostate
{
public:
    int GetTheAnswer() const { return sAnswer; }
private:
    static int sAnswer;
};
// monostate.cpp
int Monostate::sAnswer = 42;
  
```
  
或者
  
``` c++
// monostate.h
class Monostate
{
public:
    int GetTheAnswer() const;
private:
};
// monostate.cpp
static int sAnswer = 42;
int Monostate::GetTheAnswer() const{return sAnswer;}
  
```
  
###  3.5 Session context
  
  
Singleton的使用并不是必须的, Singleton的本质实际上就是用来保存一份全局的数据. 其实可以将这些数据独立出来单独组成一个session类, 只要控制这个类的生成数量, 也可以达到和singleton的类似效果.  
  
###  3.6 工厂模式
  
  
工厂模式是针对多态继承而发展出的一种 在运行时才确认究竟要创建哪种继承子类的的对象.  
  
重点:
  
1. 继承时的base类的constructor是无法virtual的. 子类的内存占用是基于base类的, 因此编译器在编译时固定需要先调用base类的constructor,再调用子类的constructor, 从而才能确认子类的大小.  如果在子类的constructor的initialize list中没有指明是调用base具体的哪个constructor,编译器就会默认使用default constructor进行base类的construct.  
  
2. 继承时的base类的destructor是需要设置成virtual的. 因为指针的delete默认只会调用base类的destructor, 除非将base类的destructor设置成virtual, 才会先去查找子类的destructor, 再调用base类的destructor(顺序和constructor是相反的)
  
``` c++
class base{
    public:
    ~base(){std::cout << "bdes" << endl;};
};
  
  
class child: public base{
    public:
    ~child(){std::cout << "cdes" << endl; };
};
  
int main() {
    base* r = new child;
    delete r;
    return 0;
}
  
//结果:
// bdes
```
  
``` c++
class base{
    public:
    virtual ~base(){std::cout << "bdes" << endl;};
};
  
  
class child: public base{
    public:
    ~child() override {std::cout << "cdes" << endl; };
};
  
int main() {
    base* r = new child;
    delete r;
    return 0;
}
  
//结果: 
// cdes
// bdes
```
  
工厂模式的实现就是通过一个abstract base class, 和一系列子类, 再加上用于生成子类的factory接口
  
``` c++
// renderer.h
#include <string>
class IRenderer
{
public:
virtual ~IRenderer() {}
virtual bool LoadScene(const std::string &filename) = 0;
virtual void SetViewportSize(int w, int h) = 0;
virtual void SetCameraPosition(double x, double y, double z) = 0;
virtual void SetLookAt(double x, double y, double z) = 0;
virtual void Render() = 0;
};
  
// rendererfactory.h
#include "renderer.h"
#include <string>
class RendererFactory
{
public:
IRenderer *CreateRenderer(const std::string &type);
};
  
// rendererfactory.cpp
#include "rendererfactory.h"
#include "openglrenderer.h"
#include "directxrenderer.h"
#include "mesarenderer.h"
  
IRenderer *RendererFactory::CreateRenderer(
    const std::string &type)
{
    if (type == "opengl")
    return new OpenGLRenderer();
    if (type == "directx")
    return new DirectXRenderer();
    if (type == "mesa")
    return new MesaRenderer();
    return NULL;
    }
  
```
  
上面这种工厂, 只能生成sdk内部已定义好的子类, 如果用户有需求要提供自己的子类时, 可以继续扩展成带回调的更为灵活的方式.  
  
注意工厂类通常是单例或者单一状态的(全static)  
  
```c++
  
// rendererfactory.h
#include "renderer.h"
#include <string>
#include <map>
class RendererFactory
{
public:
    typedef IRenderer *(*CreateCallback)();
  
    static void RegisterRenderer(const std::string &type, CreateCallback cb);
    static void UnregisterRenderer(const std::string &type);
    static IRenderer *CreateRenderer(const std::string &type);
private:
    typedef std::map<std::string, CreateCallback> CallbackMap;
    static CallbackMap mRenderers;
};
  
#include "rendererfactory.h"
// instantiate the static variable in RendererFactory
RendererFactory::CallbackMap RendererFactory::mRenderers;
void RendererFactory::RegisterRenderer(const std::string &type, CreateCallback cb)
{
    mRenderers[type] = cb;  
}
void RendererFactory::UnregisterRenderer(const std::string &type)
{
    mRenderers.erase(type);
}
IRenderer *RendererFactory::CreateRenderer(const std::string &type)
{
    CallbackMap::iterator it = mRenderers.find(type);
    if (it != mRenderers.end())
    {
        // call the creation callback to construct this derived type
        return (it->second)();
    }
    return NULL;
}
  
//用户的实际使用代码
class UserRenderer : public IRenderer
{
public:
    ~UserRenderer() {}
    bool LoadScene(const std::string &filename) { return true; }
    void SetViewportSize(int w, int h) {}
    void SetCameraPosition(double x, double y, double z) {}
    void SetLookAt(double x, double y, double z) {}
    void Render() { std::cout << "User Render" << std::endl; }
    static IRenderer *Create() { return new UserRenderer(); }
};
int main(int, char **)
{
    // register a new renderer
    RendererFactory::RegisterRenderer("user", UserRenderer::Create);
    // create an instance of our new renderer
    IRenderer *r = RendererFactory::CreateRenderer("user");
    r->Render();
    delete r;
    return 0;
}
  
  
```
  
注意实现方式是把回调函数的指针用一个map保存起来了. 用户的回调是通过类内的static函数实现的. 也可以放在外部作为一个helper的全局函数.  
  
还有一点要注意的是, CreateRenderer的实现, 依然可以先预制几种sdk内部定义的类型, 可以在初始化函数中预先注册好.  
  
###  3.7 API的wrapping模式
  
  
常用的三种模式 : Proxy Adapter 和 Facade  
  
####  Proxy模式
  
  
Proxy : 一对一的接口封装. 只做数据中转发, 不进行任何额外的处理. Proxy类可以直接通过指针的形式, 管理旧接口的生命周期.  
  
如果可以直接修改源码, 那么基于旧接口创建一个抽象的基类可能逻辑上更为内聚.  
  
```C++
  
class IOriginal //新增的基类
{
public:
    virtual bool DoSomething(int value) = 0;
};
  
class Original : public IOriginal
{
public:
    bool DoSomething(int value);
};
  
class Proxy : public IOriginal{
public:
    Proxy() : mOrig(new Original())
    {};
    ~Proxy()
    {
        delete mOrig;
    }
  
    bool DoSomething(int value)
    {
        return mOrig->DoSomething(value);
    }
private:
    Proxy(const Proxy &);
    const Proxy &operator =(const Proxy &);
    Original *mOrig;
}
  
```
  
####  Adapter模式
  
  
Adapter : Adapter实际就是Proxy的延伸. Proxy通常不做数据的处理, 仅仅是转发, 而Adapter所处理的往往是需要对接口输入的数据进行一次格式化处理后 , 再转给旧接口. 新旧接口是可兼容但不一致.  
  
实现方法既可以像proxy一让通过内部的指针实现. 另一种方式则是通过private继承, 让新接口获得旧接口的全部功能, 但暴露给用户的全是新的接口实现.  
  
通常SDK中如果使用了第三方的库实现某些功能, 为了保持接口的一致性, 会使用Adapter模式.  
  
另外如果想对C的接口进行C++的封装, 也可以用到Adapter  
  
``` C++
class CppAdapter
{
public:
    CppAdapter()
    {
        mHandle = create_object();
    }
    ~CppAdapter()
    {
        destroy_object(mHandle);
        mHandle = NULL;
    }
    void DoSomething(int value)
    {
        object_do_something(mHandle, value);
    }
private:
    CppAdapter(const CppAdapter &);
    const CppAdapter &operator =(const CppAdapter &);
    Handle *mHandle;
};
```  
  
####  Facade模式
  
  
就是对多个接口类的封装, 形成更高层的抽象接口.  
  
旧接口:  
  
```C++
class Taxi
{
public:
bool BookTaxi(int npeople, time_t pickup_time);
};
  
class Restaurant
{
public:
bool ReserveTable(int npeople, time_t arrival_time);
};
  
......
  
```
  
经过facade后
  
``` C++
class ConciergeFacade
{
public:
    enum ERestaurant {
    RESTAURANT_YES,
    RESTAURANT_NO
    };
    enum ETaxi {
    TAXI_YES,
    TAXI_NO
    };
  
    time_t BookShow(int npeople, ERestaurant addRestaurant, ETaxi addTaxi);
};
```  
  
###  3.8 Observer模式
  
  
Observer需要解决的问题是过度依赖或者循环依赖的情况. 通常情况下A类如果需要调用B类的method, 那么就必须include B.h. 这在大型项目中会增加依赖链条, 一旦B的实现有修改, 那么所有的依赖类都需要重新编译, 耗时较长.  
  
####  MVC model - view - controller模型
  
  
model 是核心业务代码  view 是用于对外展示的服务代码  controller则是对外的接口, 负责调用两者, 更新两者的状态.  
model是稳定的, 不常变化的. view则是经常变化. 所以依赖关系是view依赖model, 而controller两者都依赖.  
![API_design-2021-11-29-2021-11-29_17-45](https://cdn.jsdelivr.net/gh/bicirci/PicBed@master/MarkDown/API_design-2021-11-29-2021-11-29_17-45.png )  
  
整个系统的状态变化的来源: 一是来自用户使用controller进行主动交互, 一是系统自动运行时内部各种数据的变化.  
因此view除了接受controller的消息, 还要监测model的状态变化, 从而作出不同的反应.  
  
各种UI类的工具都是这样实现的, 对一个简单的checkbox, model负责保存on/off state, view通过一个按钮展示当前状态, controller则用于更新这两者中状态.  
  
view依赖model从而可以监测model的状态, model则完全不需要知道view的细节.但是有些情况可能不能单纯的要求view一直监测model的状态, 而是希望model变化时可以通知到view, 这时就用到了Observer模式.  
  
Observer通常是一对多的关系. 使用的术语为subject和observer.  
![API_design-2021-11-29-2021-11-29_18-23](https://cdn.jsdelivr.net/gh/bicirci/PicBed@master/MarkDown/API_design-2021-11-29-2021-11-29_18-23.png )  
  
两个抽象类的实现
  
``` C++
#include <map>
#include <vector>
class IObserver
{
public:
virtual ~IObserver() {}
virtual void Update(int message) = 0;
}
  
class ISubject
{
public:
ISubject();
virtual ~ISubject();
virtual void Subscribe(int message, IObserver *observer);
virtual void Unsubscribe(int message, IObserver *observer);
virtual void Notify(int message);
private:
typedef std::vector<IObserver *> ObserverList;
typedef std::map<int, ObserverList> ObserverMap;
ObserverMap mObservers;
};
  
```
  
注意其中的map格式, 根据message可以筛选
  
####  Observer的 push 和 pull模式
  
  
依据Observer内Update这个函数的实现方式, 可以分成两类. push模式是在消息推送时就附带了所需的消息, 类似QT的实现. 而pull模式则在update时, 只是一个通知信号, observer想知道具体变化后的状态, 还得自己再去查询subject. 一般简单的数据可以用push, 而大块的数据通常都需要pull模式.  
  
##  4. API的设计和思考
  
  
正常的开发流程图:  
![API_design-2021-11-29-2021-11-29_18-54](https://cdn.jsdelivr.net/gh/bicirci/PicBed@master/MarkDown/API_design-2021-11-29-2021-11-29_18-54.png )  
  
对于一项开发任务, 务必要从三个方面明确目标: 1. Business requirements : 怎样推动公司业务发展 2. Functional requirement : 软件功能定义 3. Non-funtional requirement : 面向用户的适用性.  
  
###  4.1 Functional requirement
  
  
对于API开发来说, 设计阶段最重要的就是功能定义. 功能定义一定要切实了解用户需求, 不可按经验猜测. 设计时需要关注以下几点:
  
1. 用户希望用API完成什么任务?
2. 从用户的角度来看, 最佳的任务流程workflow是怎样的?
3. 用户可能使用的inputs, 要同时考虑types和ranges
4. 用户期待的输出, 同时考虑type format和ranges
5. 用户使用哪些文件格式和协议, 哪些需要支持
6. 用户目前已有哪些业务模型?
7. 用户使用了那些技术工具?
  
此外, 还有一些非功能性的需求需要考虑:
  
1. 性能(哪些API会有具体的性能要求)
2. 平台兼容性
3. 安全性
4. 扩展性和灵活性: 在现实使用中, 是否有大量同时调用发生, API能否应对, 是否有扩展性.
5. 易用程度
6. 是否有多线程需求
7. 软件定价
  
关键是要明确软件需要做什么事, 而不是怎么去实现.  
  
在整个开发流程中, Funcitonal requirement是一定会发生变化的. 在每一次变化发生时, 都一定要跟各方明确所增加的成本和时间. 如果跟业务发展方向不符合的需求, 要尽早发现并拒绝.  
  
###  4.2 User Cases
  
  
用户刻画作为一项分析工具, 是用来辅助Functional requiremnet的. Functional requirement 是list of features, 而 user caces 则是站在用户的角度, 理解用户的使用方式. 理解 "用户怎样开车" 而不是 "用户怎样调节活塞进油量来加速"  
  
User case的描述方式通常是 角色 + 动作的组合, 名词+动词开头.  
  
以一个ATM机程序为例:
Step 1. User inserts ATM card.
Step 2. System validates that ATM card is valid for use with the ATM machine.
Step 3. System prompts the user to enter PIN number.
Step 4. User enters PIN number.
Step 5. System checks that the PIN number is correct.
  
所有步骤都是描述行为, 不要带有具体的数据逻辑. 比如 用户输入小于1000元的值, 会弹出XXX页面 这类包含逻辑的描述. 避免沉入细节中.  
  
User case只是从用户角度帮助理解functional requirement, 而不是要为你api的所有接口一一找到对应的case. 描述核心用法和workflow即可.  
  
一份 user case 案例:  
![API_design-2021-11-30-2021-11-30_11-54](https://cdn.jsdelivr.net/gh/bicirci/PicBed@master/MarkDown/API_design-2021-11-30-2021-11-30_11-54.png )  
  
###  4.3 程序设计指南
  
  
在明确了用户需求之后, 可以正式开始系统的设计. 系统设计主要从两个维度进行, 一是Object Hierarchy, 描述的是系统内的结构组成, 另一个是Class Hierarchy, 描述的是某一个组件的具体分类(多态继承分析).  
  
具体区别见图:  
![API_design-2021-11-30-2021-11-30_12-14](https://cdn.jsdelivr.net/gh/bicirci/PicBed@master/MarkDown/API_design-2021-11-30-2021-11-30_12-14.png )  
![API_design-2021-11-30-2021-11-30_12-14_1](https://cdn.jsdelivr.net/gh/bicirci/PicBed@master/MarkDown/API_design-2021-11-30-2021-11-30_12-14_1.png )  
  
这两个维度也就分别对应了实际分析时的两个步骤:  
  
1. Architecture design : 描述顶层架构  
2. Detailed design : 描述每个模块的具体实现. 又包含了Class design和Function design.  
  
###  4.4 Architecture design
  
  
基于对user case的理解, 进行架构设计. 基础的步骤是先从user case中定位出所有的object, 再根据依赖关系, 各objec中之间的关联程度, 慢慢梳理出最顶层的模块, 和包含关系.  
  
将模块间的关系梳理好后, 就开始分层排列这些模块, 最终要实现的是自顶向下的依赖关系, ==一定要避免出现直接的循环依赖==.  
  
最后要针对架构梳理出可读文档, 除了描述架构外还要说明设计意图, 在设计时基于哪些因素做了trade off.  
  
架构设计是个大坑, 要靠时间积累.  
  
###  4.5 Class Design
  
  
Focus on designing 20% of the classes that define 80% of your API’s functionality.  
  
程序设计的二八原则, 要把精力集中在最常使用的20%的接口的设计上.  
  
多态能在C++实现, 是因为C++支持推迟对一个object的格式检查, 称为lat binding 或 dynamic bind, 从而可以在运行时再确定一个object的真正实现细节.  
  
####  4.5.1 Class参考细则
  
  
在class的设计时, 以下列表中系细则可以用作参考:
  
1. 使用inheritance时: 使用继承的类是否是 is-a的关系, 应该使用public inheritance还是private.  
2. 使用composition时: 使用时是否是 has-a的关系.  
3. 使用抽象base类时: 抽象类的定义是否准确, 是否子类有override的需求.  
4. 是否能使用已有的标准pattern来实现.  
5. 初始化和释放. 用户是用new 和 delete来获取object, 还是提供factory接口给用户.是否有特殊的内存需求.  
6. copy操作. 如果类内申请了动态内存, 那么就要对copy constructor 和 assignment做对应的处理.  
7. templates. 如果存在一系列的对象, 就要考虑是否要使用template.  
8. const和explicit的使用.  尽量多使用const , explicit常用于单参数的constructor.  
9. 操作符的override. + -  >> 等是否会用到.  
10. 隐式转换. 是否需要支持对某些格式的隐式转换.  
11. friend的使用. 尽量避免friend的使用, 当成是最终手段.friend会破坏encapsulation.  
12. 是否有性能需求要考虑.  
  
####  4.5.2 Inheritance
  
  
继承是最为常用的手法, 因此也需要额外的重视.  
  
1. 如果一个类不想被继承, 那么destructor就不要设置为virtual.  
2. 避免多于三层的继承关系.  
3. 不要对现有的接口做纯虚函数的修改. 无论是增加还是删除, 都有可能导致用户的原有代码崩溃.  
4. 不要过度设计.  比如一个类在整个系统中只被继承了一次, 那么就完全没有必要使用继承.  
5. 尽量不要使用多重继承, 尤其是菱形结构的多重继承.  
  
####  4.5.3 Liskov法则
  
  
所有的子类需要完整实现基类所要求的功能, 且需要具有一致性.  
比如很常见的矩形和正方形的例子.又一次体现了完善的测试案例的重要性.  
针对正方形的例子, 可以有两种解决思路:  
  
1. Private Inheritance
通过隐藏基类的接口, 可以在编译阶段就阻止对正方形的错误调用, 从而可以避免相同接口不一致的问题, 直接就报错告诉用户.  
这种方案再延伸一步, 如果只是希望使用部分接口, 那么可以用如下的using方式实现:
  
``` c++
class Circle : private Ellipse
{
public:
    Circle();
    explicit Circle(float r);
    // expose public methods of Ellipse
    using Ellipse::GetMajorRadius;
    using Ellipse::GetMinorRadius;
  
    void SetRadius(float r);
    float GetRadius() const;
};
  
```
  
2. Composition  
虽然使用private inheritance可以很便捷的解决liskov问题, 但是更加推荐的做法是修改为composition关系.  
  
``` c++
class Circle
{
public:
Circle();
    explicit Circle(float r);
    void SetRadius(float r);
    float GetRadius() const;
private:
    Ellipse mEllipse;
};
  
void Circle::SetRadius(float r)
{
mEllipse.SetMajorRadius(r);
mEllipse.SetMinorRadius(r);
}
  
```
  
在C++中, 所有的同类继承关系要基于接口的行为是否一致, 而不是逻辑上的一致, 逻辑上的is-a方法只是辅助.  
  
####  4.5.4 开闭法则
  
  
open to extend, closed to modification  
  
但更现实的做法是 closed to 不兼容的接口变化, open to 功能性的扩充.  
就像之前的render的api一样, 全部内部写死的做法一定是不符合开闭法则的.  
  
####  4.5.5 Least Knowledge法则
  
  
成员函数应该只能获取到有限的内容:
  
* other Functions in same class
* Functions on data members
* Functions on parameters
* Functions on local objects
* Limited access to global object
  
需要避免通过一个内部object获取另一个object的操作:
  
```C++
void MyClass::MyFunction()
{
    mObjectA.GetObjectB().DoAction();
}
  
//应该修改为
  
void MyClass::MyFunction()
{
    mObjectA.DoAction();
}
//或者
void MyClass::MyFunction(const ObjectB &objectB)
{
    objectB.DoAction();
}
```  
  
####  4.5.6 Class的命名
  
  
当函数名超过2个单词后, 就要仔细考虑这个名称是否合适了.  命名要尽量依赖使用者的专业术语, 有抽象性的动作含义.  
  
abstract base classes 通常都是形容类的, 可以用Renderable, Observable来命名, 这样在继承时更为直观.  当然也可以在前面加I来指示这是一个Interface(本书常用) IRenderer IObserver
  
避免缩写的使用!
  
最顶层的函数, 最好用namespace包裹, 或者有统一的独特前缀
  
##  4.6 Function Design
  
  
###  4.6.1 Function参考细则
  
  
全局函数
  
* Static versus non-static function.
* Pass arguments by value, reference, or pointer.
* Pass arguments as const or non-const.
* Use of optional arguments with default values.
* Return result by value, reference, or pointer.
* Return result as const or non-const.
* Operator or non-operator function.
* Use of exception specifications
  
成员函数
  
* Virtual versus non-virtual member function.
* Pure virtual versus non-pure virtual member function.
* Const versus non-const member function.
* Public, protected, or private member function.
* Use of the explicit keyword for non-default constructors.
* Friend function versus non-friend function.
* Inline function versus non-inline function
  
###  4.6.2 函数命名
  
  
* 常用的属性操作, 遵守 Get 和 Set的命名前缀
* 回答对错的函数, 应该用 Is, Are, Has 为前缀, 并返回bool的值.  
* 执行类的函数需要用强动词为前缀, 比如Enable Print Save. 如果是全局函数, 最好还加上被操作的对象 FormatString() MakeVector3d()
* 尽量使用正向的判断, IsConnected 而不是 IsUnconnected(), 避免 double negatives
* 底层的函数名要描述所有的操作, 比如图片操作后进行本地保存, SharpenAndSaveImage() 而不要用SharpenImage(), 丢失信息. 如果一个底层函数名过长, 很可能你需要将这个函数进行拆分重构.  
* 尽量成对的设置接口.  OpenWindow 对应的就是 CloseWindow.  
![API_design-2021-12-03-2021-12-03_15-46](https://cdn.jsdelivr.net/gh/bicirci/PicBed@master/MarkDown/API_design-2021-12-03-2021-12-03_15-46.png )  
  
###  4.6.3 函数的参数
  
  
* 选择合适的数据类型
* 选择合适的参数数量, 超过5个参数的函数都是不可接受的.
  减少参数数量可以通过将参数封装成struct实现, 也可以通过在调用函数前通过一系列的set函数, 将所需的参数传递进接口内部.这里还可以使用Named Parameter Idiom (NPI)的技巧.  
  将每个参数设置的接口返回值都设置为对象的引用 (return *this;), 这样就可以实现链式调用.
  
  ``` C++
  QTimer timer = QTimer().setInterval(1000).setSingleShot(true).start();
  ```
  
###  4.6.4 错误的处理
  
  
* 接口需要能返回错误信息给使用者. 并且所有的错误信号要进行明确的文档定义Use a consistent and well-documented error handling mechanism.  
C++下的最优方法依旧是exception. 但是纯C接口则只能通过error_code实现. 如果希望一个函数既返回错误码, 也返回一个处理后的结果值, 可以通过std::tuple<> 和 std::make_tuple<>来实现.  
可以考虑像C库一样, 提供一个全局的glGetError()函数来让用户查看错误信息.  
使用exception的优点是更加简洁, 但一旦使用就意味着所有使用你代码的程序都要具备exception的handle能力. 这也是为什么google禁止了exception的使用.  
Derive your own exceptions from std::exception.  
使用exception 要尽量搭配RAII一起使用. 保证栈展开后的内存不发生泄漏.  
减少全部捕获 catch(...)的使用.  
  