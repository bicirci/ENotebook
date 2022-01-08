  
  
#  GTest 使用指南
  
  
- [GTest 使用指南](#gtest-使用指南 )
  - [1.概述](#1概述 )
  - [2.测试案例编写的基本原则](#2测试案例编写的基本原则 )
  - [3.基础语法](#3基础语法 )
    - [3.1 测试集和测试案例](#31-测试集和测试案例 )
    - [3.2 常用的判断宏](#32-常用的判断宏 )
    - [3.3 预装数据的测试集](#33-预装数据的测试集 )
  - [4.运行测试集](#4运行测试集 )
  
##  1.概述
  
  
本文旨在说明googletest的常用语法，和使用时的惯常用法。一个完整项目的测试应该在不同的尺度上采用不同的测试方法。googletest主要是用于单元测试阶段，主要针对的是独立的函数的功能验证，以及对部分流程进行性能对比。  
学会使用单元测试工具有助于培养良好的编程习惯，一个便于测试的函数，往往也是与其他功能相解耦的,也更有利于后期测试时的bug定位。  
  
##  2.测试案例编写的基本原则
  
  
1. 一个测试案例只关注单一功能模块  
2. 测试案例的启动前的环境配置尽量简单  
3. 简单易读  
4. 测试案例之间应该相互独立，并且是可重复，可复现的  
  
##  3.基础语法
  
  
###  3.1 测试集和测试案例
  
  
测试案例的格式和普通函数差别不大，是最基础的测试单元。 每个测试案例都从属于某个测试集，一个测试集可包含任意数量的测试案例。
  
``` ditaa (cmd, run_on_save=true, align="center", args=["-E"], hide=true)  
                                +--------+ +---------+
                                |  cYEL  | |  cYEL   |
                                |  测试集名称 | |  测试案例名称 |
                                |        | |         |
                                +----+---+ +----+----+
                                     |          |
                                     v          v
                         +------------------------------------+
                         |                 cBLU               |
           +------------>|   TEST(TriangleTest,isEquilateral){|
           |             |   Triangle tri{2,2,2};             |
           |             |   EXPECT_TRUE(tri.isEquilateral());|
           |             |   }                                |
           |             +------------------------------------+
           |                      ^
           |                      |
+----------+----------+    /------+--------------------------\
|        cYEL         |    |  cYEL  EXPECT 和 ASSERT 两大类宏函数   |
|  TEST宏定义了一个测试案例及其名称 |    |        对需要测试的变量进行正误判断           |
|                     |    +/-------------------------------\+
+---------------------+    ||cA0A ASSERT 大类判断失败时，会终止测试程序继续运行||
                           |+-------------------------------+|
                           ||cGRE EXPECT 大类判断失败时，程序能继续运行    ||
                           |\-------------------------------/|
                           \---------------------------------/
  
```
  
###  3.2 常用的判断宏
  
  
**注意：以下宏语句中的ASSERT都可以用EXPECT替换**
  
| 判断类型 | 宏语句 |  
| --- | --- |  
| 数值比较| ASSERT_EQ(expected, actual) / ASSERT_NE(val1, val2)ASSERT_LT(val1, val2) / ASSERT_LE(val1, val2)ASSERT_GT(val1, val2) / ASSERT_GE(val1, val2) |
| 浮点数比较| ASSERT_FLOAT_EQ(expected, actual)/ASSERT_DOUBLE_EQ(expected, actual)/ASSERT_NEAR(val1, val2, abs_error) |
| 字符串比较| ASSERT_STREQ(expected_str, actual_str) / ASSERT_STRNE(str1, str2)ASSERT_STRCASEEQ(expected_str, actual_str) / ASSERT_STRCASENE(str1, str2) |
| 异常判断| ASSERT_THROW(statement, exception_type)/ASSERT_ANY_THROW(statement)/ASSERT_NO_THROW(statement) |
| bool判读| ASSERT_TRUE(statement) |
  
###  3.3 预装数据的测试集
  
  
当一个测试集内的所有测试案例使用相同的初始化环境（setUp&tearDown相同）时，可以使用预装数据的测试集Test Fixtures。
预装数据的测试集需要先定义规范的数据类，从而实现数据的预装。
  
```C++
class StackTest: public ::testing::test{  //StackTest为测试集名称
  protected:
    void SetUp() override {   
      s1.push(1);             //每个测试案例开始前
      s2.push(2);             //都会新生成一个StackTest数据对象
      s2.push(3);             //并调用该SetUp初始化数据
    }
  
    void TearDown()  override {}  //每个测试案例结束时都会调用该TearDown
  
    MyStack<int> s1;
    MyStack<int> s2;
}
  
//预装数据的测试集在使用时需要使用TEST_F宏声明
    此处的测试集名称需要和上面定义的StackTest类保持一致
TEST_F(StackTest, popOfOneIsEmpty){ 
    s1.pop();               //可以直接使用类中protected部分的成员数据
    EXPECT_EQ(0, s1.size());
}
  
TEST_F(StackTest, popOfTwoIsNotEmpty){ 
    s2.pop();               
    EXPECT_NE(0, s2.size());
    EXPECT_NE(0, s1.size()); //因为每个测试案例都会重新初始化数据，所以s1重新变回包含1个数据的状态
}
  
```
  
##  4.运行测试集
  
  
TEST() 和 TEST_F() 会隐式的将测试集注册到googletest框架内，不需要额外的列出要运行哪些测试集。  
当编写好测试集后，可以在main函数调用RUN_ALL_TEST()来运行所有的测试集。如果所有的测试都通过，该接口返回0,否则返回1。  
但googletest为了简化使用，在编译的静态库libgtest_main.a中已经包含了一个规范的main函数，开发者需要做的仅仅是编写好测试集的cpp源代码，在编译时链接gtest_main的库文件就可以了。  
只有在特殊情况下，开发者可能需要自己编写main函数（比如需要读取命令行参数）。那么此时就务必要保证调用RUN_ALL_TEST()。  
  
* 以下是官方提供的一个模板，请注意结尾部分的main函数写法。  
**注意**： `::testing::InitGoogleTest(&argc, argv)` 是为了处理googletest提供的默认命令行选项，该函数会自动将googletest支持的配置选项摘除，只剩下用户需要使用的命令行参数。
  
```C++
#include "this/package/foo.h"
  
#include "gtest/gtest.h"
  
namespace my {
namespace project {
namespace {
  
// The fixture for testing class Foo.
class FooTest : public ::testing::Test {
 protected:
  // You can remove any or all of the following functions if their bodies would
  // be empty.
  
  FooTest() {
     // You can do set-up work for each test here.
  }
  
  ~FooTest() override {
     // You can do clean-up work that doesn't throw exceptions here.
  }
  
  // If the constructor and destructor are not enough for setting up
  // and cleaning up each test, you can define the following methods:
  
  void SetUp() override {
     // Code here will be called immediately after the constructor (right
     // before each test).
  }
  
  void TearDown() override {
     // Code here will be called immediately after each test (right
     // before the destructor).
  }
  
  // Class members declared here can be used by all tests in the test suite
  // for Foo.
};
  
// Tests that the Foo::Bar() method does Abc.
TEST_F(FooTest, MethodBarDoesAbc) {
  const std::string input_filepath = "this/package/testdata/myinputfile.dat";
  const std::string output_filepath = "this/package/testdata/myoutputfile.dat";
  Foo f;
  EXPECT_EQ(f.Bar(input_filepath, output_filepath), 0);
}
  
// Tests that Foo does Xyz.
TEST_F(FooTest, DoesXyz) {
  // Exercises the Xyz feature of Foo.
}
  
}  // namespace
}  // namespace project
}  // namespace my
  
int main(int argc, char **argv) {
  ::testing::InitGoogleTest(&argc, argv);
  return RUN_ALL_TESTS();
}
  
```
  