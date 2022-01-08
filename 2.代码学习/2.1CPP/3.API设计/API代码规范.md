  
  
#  API代码规范
  
  
本文是更细节的代码实现规范
  
##  1. Style
  
  
常见的API主要的Style有四大类:Flat C APIs, Object-Oriented C++ APIs, Template-Based APIs, Data-Driven APIs
  
###  1.1 Flat C APIS
  
  
纯C接口通常使用受限制的数据类型:
  
1. Built-in types such as int, float, double, char, and arrays and pointers to these.
2. Custom types created via the typedef and enum keywords.
3. Custom structures declared with the struct or union keywords.
4. Global free functions.
5. Preprocessor directives such as #define.
  
C++对函数的格式检查更为严格, 因此纯C接口也可以尝试用C++编译器编译一下, 检查有没有错误.
Try compiling your C API with a C++ compiler for greater type checking and to ensure that a C++ program can use
your API.
  
但正因为C的API对数据检查更为宽松, 使得C的接口升级更为简单. C++的类即便有很小的变化, 也可能造成类的二进制格式发生改变, 从而需要用户重新编译全部的代码.  
  
####  1.1.1
  
  
先比较一下c++和c的接口形式
  
``` C++
class Stack
{
public:
void Push(int val);
int Pop();
bool IsEmpty() const;
private:
int *mStack;
int mCurSize;
};
```
  
``` C
struct Stack
{
int *mStack;
int mCurSize;
};
void StackPush(Stack *stack, int val);
int StackPop(Stack *stack);
bool StackIsEmpty(const Stack *stack);
  
```
  
主要的区别在于C的接口需要自行保存指针, 而C++将指针做了对象化的转化. 通常做法, 还会将Stack* 再定义为StackPtr的自定义类型.  
  
###  1.2 object-oriented c++ apis
  
  
Object-oriented APIs let you model software in terms of objects instead of actions. They also offer the advantages
of inheritance and encapsulation.
  
OOP的缺点主要在于无法binary compatiblity(接口变化一般都需要再重新编译库文件) 和inheritance的乱用.  
  
###  1.3 Template-based API
  
  
模板元编程
  
模板其实就是用于compile阶段的编程, 并且可以进行类型判断, 从而替代了原有的#define
  
尽量减少define的使用
  
函数重载实际也是一种编译期的编程
  
这种编程方式的最大优点是可以提升运行期的效率, 因此在深度学习领域可能会得到更多的运用. 但其不易维护, 编程难度较大, 也是其未成为主流编程模式的原因.  
  
template<>还可以进行特化, 这样针对某种特殊类型可以有独特的操作
  
``` C++
template <>
void Stack<int>::Push(int val)
{
// integer specific push implementation
}
```
  
而最大的缺点就是代码实现全部在头文件中, 造成逻辑暴露. 书中提供了explicit instantiation的方法改善这类问题, 后问会详细介绍.  
  
此外, 通过template会造成可执行文件的尺寸增长, 如果系统对硬盘或内存十分敏感, 而对运行速度的要求没有那么高, 那么最好还是使用oop的方式.  
  
最后, 就是模板的错误信息往往长的离谱, 不熟悉的开发人员会很头痛.  
  
###  1.4 Data Driven API
  
  
数据驱动的API指的是直接传入数据和控制流参数的接口, 各类命令行指令都是这种类型.  
  
几种API格式的类比:
• func(obj, a, b, c) = flat C-style function
• obj.func(a, b, c) = object-oriented C++ function
• send("func", a, b, c) = data-driven function with parameters
• send("func", dict(arg1=a, arg2=b, arg2=c)) = data-driven function with a dictionary of named arguments (pseudo code)
  
这种接口的优点是可以批量生成任务流写入文本文件中, 比如一个Data Driven的Stack示例:  
  
``` C++
Class Stack
{
public:
    Stack();
    Result Command(const std::string &command, const ArgList &args);
};
  
s = new Stack();
if (s)
{
    s->Command("Push", ArgList().Add("value", 10));
    s->Command("Push", ArgList().Add("value", 3));
    Stack::Result r = s->Command("IsEmpty");
    while (! r.convertToBool())
    {
    s->Command("Pop");
    r = s->Command("IsEmpty");
    }
    delete s;
}
```
  
数据流文本:
  
``` txt
# Input file for data-driven Stack API
Push value:10
Push value:3
Pop
Pop
```
  
只要再配合一个用空格分割文本并生成ArgList结构的程序, 就可以根据不同的文本,动态的进行计算.  
对文本的改动就相当与编程, 不需要再进行重新编译.  
  
Data-driven APIs map well to Web services and other client/server APIs. They also support data-driven testing
techniques.
  
Data Driven的缺点就是运行速度比较慢, 需要针对每一条指令先进行索引的操作. 而且Data Driven的接口无法通过头文件了解到其所能实现的功能, 因此需要比较详细的文档说明.此外, 这类接口的数据格式的检测较为宽松, 需要格外的注意.  
  
###  1.5 数据入参的封装
  
  
由于C++的强类型检查, 没法像python一样, 用单个入口接收不同类型的参数. 因此, 书中提供了一种ArgList的数据封装, 类似Qany的形式, 从而可以实现key-value对的数据封装.  
许多公开库都提供了这种封装. 比如QT的QVariant, boost::any.  
这种封装的内部实现, 大多是利用Union Inheritance void*这几种方式实现的.  
不管内部实现是怎样的, 他们的外部接口大概是如下的格式:  
  
``` C++
class Arg
{
public:
    // constructor, destructor, copy constructor, and assignment
    Arg();
    ~Arg();
    Arg(const Arg&);
    Arg &operator = (const Arg &other);
    // constructors to initialize with a value
    explicit Arg(bool value);
    explicit Arg(int value);
    explicit Arg(double value);
    explicit Arg(const std::string &value);
    // set the arg to be empty/undefined/NULL
    void Clear();
    // change the current value
    void Set(bool value);
    void Set(int value);
    void Set(double value);
    void Set(const std::string &value);
    // test the type of the current value
    bool IsEmpty() const;
    bool ContainsBool() const;
    bool ContainsInt() const;
    bool ContainsDouble() const;
    bool ContainsString() const;
    // can the current value be returned as another type?
    bool CanConvertToBool() const;
    bool CanConvertToInt() const;
    bool CanConvertToDouble() const;
    bool CanConvertToString() const;
    // return the current value as a specific type
    bool ToBool() const;
    int ToInt() const;
    double ToDouble() const;
    std::string ToString() const;
private:
    ...
};
  
//通过ArgList的结构体进行输入参数的整体封装(类似qarray)
  
class ArgList{
public:
    ArgList();
    ArgList& Add(std::string arg, bool value);
    ArgList& Add(std::string arg, int value);
    //......
  
private:
    std::map<std::string, Arg> keeper_;
}
  
s = new Stack();
s->Command("Pop", ArgList().Add("NumberOfElements", 2).Add("Arg1", true));
  
//同样也可创建类似的Result类
```
  
##  2.C++的惯常用法
  
  
###  2.1 namespace
  
  
namespace是为了避免接口命名冲突的问题, 在C接口, 通常是使用前缀的方式实现.
  
>The OpenGL API uses “gl” as a prefix for all public symbols, for example, glBegin(),
glVertex3f(), and GL_BLEND_COLOR (Shreiner, 2004).
  
C++的namespace也可以达到隔离的效果
  
``` C++
using namespace std;
string str("Look, no std::");
  
using std::string;  //这种方式更优, 有接口范围限制
string str("Look, no std::");
```
  
要小心不要把using放到全局的位置.  
  
###  2.2 Constructor 和 Assignment
  
  
Copy Constructor 当以下情况发生时会被调用:  
  
1. 作为函数by value的参数或返回值  
2. 用另一个对象初始化声明+定义新的对象: `MyClasss a = b; MyClass a{b}`  
3. 在exception处理中被thrown或caught(类似1情况)  
  
Assignment Operator  
当对一个已经存在的对象赋值时调用, Assignment Operator有几点要注意:  
  
1. 使用const reference作为right-hand operand  
2. 以Return by reference 的形式返回 *this, 从而支持链式操作  
3. 把left-hand operand的状态先清空, 再赋新值  
4. 记住检查self-assignment, 用this 和 &rhs比较  
  
如果不允许拷贝行为, 那么就要将以上两项声明为private class或使用boost::noncopyable  
  
在类的定义时要遵循Big Three原则, 即  the destructor, the copy constructor, and the assignment operator 不可缺一.(尤其是有指针类成员时, 默认生成的copy行为, 可能会造成double free的问题)  
  
####  2.2.1 管理自动生成的类函数  
  
  
尽量使用 default 和 delete 对默认的类函数(constructor assignement等)进行修饰. 自动生成的行为都是编译器行为, 所以这些修饰词也是在编译阶段生效的.  
  
####  2.2.2 Explicit关键词  
  
  
避免constructor的自动格式转换.  
  
###  2.3 Const 的正确使用  
  
  
####  2.3.1 const method的应用
  
  
使用const修饰method, 意味着
1.该接口不会对用户能感知的数据有任何修改  
2.const对象只能调用const的接口  
  
关于第一点, 关键是想说明某些情况下, const修饰的接口仍旧可以对class内部数据进行修改, 但是该修改应该对用户是不可见的. 但要注意的是, 这种对内部数据的修改,必须是受限的,需要将需要修改的变量声明为mutable.(或者使用前文的pimpl方法)  
  
下面是一个案例, 获取一个对象的容量, 如果该对象没有发生过增减, 那么完全可以使用之前cache的数据进行加速.  
  
``` C++
Class HashTable
{
public:
void Insert(const std::string &str);
int Remove(const std::string &str);
bool Has(const std::string &str) const;
int GetSize() const;
...
private:
mutable int mCachedSize;
mutable bool mSizeIsDirty;
...
};
  
int HashTable::GetSize() const
{
    if (mSizeIsDirty)
    {
        mCachedSize = CalculateSize();
        mSizeIsDirty = false;
    }
    return mCachedSize;
}
  
```
  
####  2.3.2 const修饰接口返回值
  
  
某些情况受格式限制, 可能存在返回reference或者pointer的情况, 如果返回给用户的这些引用是不希望用户修改, 只提供给用户做调用或者拷贝, 那么最好用const修饰返回值.  
  
  
``` c++
const std::string &GetName() const
{
    return mName;
}
```
  
当然最好还是避免这种用法, 直接使用return by value, 因为即便是使用const reference, 用户依然可以强制cast成可修改的状态
  
``` C++
// get a const reference to an internal string
const std::string &const_name = object.GetName(); //调用上文的GetName
// cast away the constness
std::string &name = const_cast<std::string &>(const_name);
// and modify the object’s internal data!
name.clear();
```
  
###  2.4 Template
  
  
Template Parameters : 参数声明
Template Arguments : 实际传入的格式 int  string ......
模板的特殊性在于其是由编译器生成的代码, 因此如果没有主动进行模板的特化, 编译器就会自行判断该在何处进行代码的插入, 并保证只生成一次, 从而不会引发编译错误.  
Implicit Instantiation: 隐性实例化. 即编译器自行决定何时生成模板代码. 要求把模板暴露在头文件中, 从而可以供编译器随时访问. 但这也意味着破坏了接口的封装性.  
Explicit Instantiation: 显性实例化. 主动进行模板的实例化, 要求全局唯一, 可以加速编译和链接速度.  
Explicit specialization: 可以针对模板函数进行多态的实现.
  
``` C++
template <typename T>
class Stack
{
public:
    void Push(T val);
    T Pop();
    bool IsEmpty() const;
private:
    std::vector<T> mStack;
};
  
template <>
void Stack<int>::Push(int val)
{
// integer specific push implementation
}
  
```  
  
Partial Specialization: 针对引用或指针类模板的多态实现
  
``` C++
template <typename T>
class Stack{
   ......//同上 
}
  
template <typename T>
class Stack<T *> {
public:
    void Push(T *val);
T *Pop();
    bool IsEmpty() const;
private:
    std::vector<T *> mStack;
};
  
```  
  
####  2.4.1 隐性实例化
  
  
有些情况, 因为无法确定用户具体会使用哪些数据格式, 可能必须进行隐性实例化. 那么此时最好将声明和实现分别在两个头文件中编写, 从而实现一定程度的解耦.  
  
``` C++
// stack.h
#ifndef STACK_H
#define STACK_H
#include <vector>
template <typename T>
class Stack
{
public:
    void Push(T val);
    T Pop();
    bool IsEmpty() const;
private:
    std::vector<T> mStack;
};
// isolate all implementation details within a separate header
#include "stack_priv.h"
#endif
```
  
``` C++
// stack_priv.h
#ifndef STACK_PRIV_H
#define STACK_PRIV_H
template <typename T>
void Stack<T>::Push(T val)
{
    mStack.push_back(val);
}
template <typename T>
T Stack<T>::Pop()
{
    if (IsEmpty())
    return T();
    T val = mStack.back();
    mStack.pop_back();
    return val;
}
template <typename T>
bool Stack<T>::IsEmpty() const
{
    return mStack.empty();
}
#endif
```
  
还有一种比较少用的export方法, 可以强制编译器在某个cpp文件内寻找模板的实现, 但由于定义的不完善, 在C++11以后, 该用法已经被废除.  
  
  