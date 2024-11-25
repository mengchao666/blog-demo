---
title: gtest
tags: [C语言]
categories: [C语言]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-25 22:48:13
topic: c
description:
cover:
banner:
references:
---
## 一、构建gtest

执行如下命令后，就会在系统目录下生成对应的头文件和静态库，可以直接在代码中引用了。

```c
git clone git@github.com:google/googletest.git 
cd googletest
mkdir build
cd build
cmake .. 
make
sudo make install
```

一个小例子 mySrc.h

```c
#ifndef UNTITLED_MYSRC_H
#define UNTITLED_MYSRC_H
int Foo(int a,int b);
#endif //UNTITLED_MYSRC_H
```

mySrc.cpp

```c
int Foo(int a, int b) {    
	if (0 == a || 0 == b)  throw "don't do that";
	int c = a % b;     
	if (0 == c) { return b; }     
	return Foo(b, c); }`
```

test.cpp

```c
`#include "mySrc.h"
#include "gtest/gtest.h"
TEST(FooTest, HandleNoneZeroInput) 
{     
	EXPECT_EQ(2, Foo(4, 10));     
	EXPECT_EQ(6, Foo(30, 18)); 
}  
int main(int argc, char *argv[]) 
{     
	testing::InitGoogleTest(&argc, argv);    
	return RUN_ALL_TESTS(); 
}
```

CMakeLists.txt

```c
cmake_minimum_required(VERSION 3.20)
project(untitled)  
set(CMAKE_CXX_STANDARD 11)  
include_directories(/usr/local/include)  
add_executable(untitled mySrc.cpp test.cpp)  
FIND_LIBRARY(gtest libgtest.a /usr/local/lib)  
target_link_libraries (untitled ${gtest})
```

输出信息：

```c
[==========] Running 1 test from 1 test suite. 
[----------] Global test environment set-up. 
[----------] 1 test from FooTest 
[ RUN      ] FooTest.HandleNoneZeroInput
[       OK ] FooTest.HandleNoneZeroInput (0 ms) 
[----------] 1 test from FooTest (0 ms total)  
[----------] Global test environment tear-down [==========] 1 test from 1 test suite ran. (0 ms total) [  PASSED  ] 1 test.
```

**上面的是通过事先编译好gtest的方式进行使用，其实也可以在项目的目录下，直接把googletest放进去的方式进行使用**

## 二、assertion

在gtest中，是通过断言（assertion）来判断代码实现的功能是否符合预期。断言的结果分为success、non-fatal failture和fatal failture。

根据断言失败的种类，gtest提供了两种断言函数：

success：即断言成功，程序的行为符合预期，程序继续向下允许。

non-fatal failure：即断言失败，但是程序没有直接crash，而是继续向下运行。

gtest提供了宏函数EXPECT_XXX(expected, actual)：如果condition(expected, actual)返回false，则EXPECT_XXX产生的就是non-fatal failure错误，并显示相关错误。

fatal failure：断言失败，程序直接crash，后续的测试案例不会被运行。

gtest提供了宏函数ASSERT_XXX(expected, actual)。

在写单元测试时，更加倾向于使用EXPECT_XXX，因为ASSERT_XXX是直接crash退出的，可能会导致一些内存、文件资源没有释放，因此可能会引入一些bug。

具体的EXPECT_XXX、ASSERT_XXX函数及其判断条件，如下两个表。

表1 一元比较

| ASSERT                  | EXPECT                  | Verifies           |
| ------------------------- | ------------------------- | -------------------- |
| ASSERT_TRUE(condition); | EXPECT_TRUE(condition); | condition is true  |
| ASSERT_FALSE(condition) | EXPECT_FALSE(condition) | condition is false |

表2 二元比较

| ASSERT                 | EXPECT                 | Condition    |
| ------------------------ | ------------------------ | -------------- |
| ASSERT_EQ(val1, val2); | EXPECT_EQ(val1, val2); | val1 == val2 |
| ASSERT_NE(val1, val2); | EXPECT_NE(val1, val2); | val1 != val2 |
| ASSERT_LT(val1, val2); | EXPECT_LT(val1, val2); | val1 < val2  |
| ASSERT_LE(val1, val2); | EXPECT_LE(val1, val2); | val1 <= val2 |
| ASSERT_GT(val1, val2); | EXPECT_GT(val1, val2); | val1 > val2  |
| ASSERT_GE(val1, val2); | EXPECT_GE(val1, val2); | val1 >= val2 |

## 三、Quick Start

下面以EXPECT_XXX为例子，快速开始使用gtest吧。

对于EXPECT_XXX，无论条件是否满足，都会继续向下运行，但是如果条件不满足，在报错的地方会显示：

没有通过的那个EXPECT_XXX函数位置； EXPECT_XXX第一个参数的值，即期待值 EXPECT_XXX第二个参数的值，即实际值 如下demo：

```c
// in gtest_demo_1.cc
#include <gtest/gtest.h>
int add(int lhs, int rhs) 
{ 
	return lhs + rhs; 
}  
int main(int argc, char const *argv[]) 
{      
	EXPECT_EQ(add(1,1), 2); // PASS     EXPECT_EQ(add(1,1), 1) << "FAILED: EXPECT: 2, but given 1";; // FAILDED          return 0; 
}
```

编译执行后输出如下：

```c
`$ ./gtest_demo_1 
/Users/self_study/Cpp/OpenSource/demo/gtest_demo_1.cc:9: Failure Expected equality of these values:   
	add(1,1)     
	Which is: 2                
# 期待的值   1                            
# 给定的值
FAILED: EXPECT: 2, but given 1 # 自己添加的提示信息
```

可能你注意到了，在EXPECT_EQ(add(1,1), 1)后有个<<，这是因为gtest允许添加自定义的描述信息，当这个语句测试未通过时就会显示，比如上面的"FAILED: EXPECT: 2, but given 1"。

这个<<和std::ostream接受的类型一致，即可以接受std::ostream可以接受的类型。

## 四、TEST

下面以googletest/samples中的sample1_unittest.cc中的demo为例，介绍如何更好地组织测试案例。

一个简单计算阶乘函数Factorial实现如下：

```c
int Factorial(int n) {   
	int result = 1;   
	for (int i = 1; i <= n; i++) {     
		result *= i;   
	}    
	return result; 
}
```

怎么使用gtest来测试这个函数的行为？

按照上面的quick start可知，这个时候就可以使用EXPECT_EQ宏来判断：

```c
EXPECT_EQ(1, Factorial(-5)); // 测试计算负数的阶乘   
EXPECT_EQ(1, Factorial(0));   // 测试计算0的阶乘   
EXPECT_EQ(6, Factorial(3));   // 测试计算正数的阶乘
```

但是当测试案例规模变大，不好组织。

因此，为了更好的组织test cases，比如针对Factorial函数，输入是负数的cases为一组，输入是0的case为一组，正数cases为一组。gtest提供了一个宏TEST(TestSuiteName, TestName)，用于组织不同场景的cases，这个功能在gtest中称为test suite。

用法如下：

```c
// 下面三个 TEST 都是属于同一个 test suite，即 FactorialTest// 正数为一组TEST(FactorialTest, Negative) {   
	EXPECT_EQ(1, Factorial(-5));   
	EXPECT_EQ(1, Factorial(-1));   
	EXPECT_GT(Factorial(-10), 0); 
} 
// 0
TEST(FactorialTest, Zero) {   
	EXPECT_EQ(1, Factorial(0)); 
} 
// 负数为一组
TEST(FactorialTest, Positive) {   
	EXPECT_EQ(1, Factorial(1));   
	EXPECT_EQ(2, Factorial(2));   
	EXPECT_EQ(6, Factorial(3));   
	EXPECT_EQ(40320, Factorial(8)); 
}
```

问题来了，怎么运行这些TEST？

在sample1_unittest.cc的main函数中，添加RUN_ALL_TESTS函数即可。

```c
int main(int argc, char **argv) {   
	printf("Running main() from %s\n", __FILE__);   
	testing::InitGoogleTest(&argc, argv);   
	return RUN_ALL_TESTS();  
}
```

在build/bin路径下，执行对应的可执行文件，输出如下：

```c
$./sample1_unittest  Running main() from /Users/self_study/Cpp/OpenSource/demo/include/googletest/googletest/samples/sample1_unittest.cc [==========] Running 6 tests from 2 test suites. # 在 sample1_unittest.cc 中有两个 test suites [----------] Global test environment set-up.      # 第一个 test suite，即上面的 FactorialTest [----------] 3 tests from FactorialTest     # 3 组 [ RUN      ] FactorialTest.Negative         # Negative 组输出 [       OK ] FactorialTest.Negative (0 ms)  # OK 表示 Negative 组全部测试通过 [ RUN      ] FactorialTest.Zero             # Zero组输出  [       OK ] FactorialTest.Zero (0 ms)     [ RUN      ] FactorialTest.Positive         # Positive组输出 [       OK ] FactorialTest.Positive (0 ms)    
[----------] 3 tests from FactorialTest (0 ms total) #sample1_unitest 另一个测试案例的输出 ...  [----------] Global test environment tear-down   [==========] 6 tests from 2 test suites ran. (0 ms total)  [  PASSED  ] 6 tests.              # 全部测试结果：PASS表示全部通过  

下面稍微修改下sample1_unittest.cc中的代码，来产生一个错误：  
TEST(FactorialTest, Negative) {   
	EXPECT_EQ(10, Factorial(-5));  // 正确的应该是  EXPECT_EQ(1, Factorial(-5));   
	// ... 
}
```

重新编译，运行结果如下：

```c
$ ./sample1_unittest  Running main() from /Users/self_study/Cpp/OpenSource/demo/include/googletest/googletest/samples/sample1_unittest.cc [==========] Running 6 tests from 2 test suites. [----------] Global test environment set-up. [----------] 3 tests from FactorialTest [ RUN      ] FactorialTest.Negative          # 开始运行上面修改的那个组 /Users/self_study/Cpp/OpenSource/demo/include/googletest/googletest/samples/sample1_unittest.cc:79: Failure                 # 测试失败，并指出错误case的位置 Expected equality of these values:           # 期待的值   10   Factorial(-5)                              # 实际计算出的值     Which is: 1 [  FAILED  ] FactorialTest.Negative (0 ms)   # 这组case测试状态：FAILED [ RUN      ] FactorialTest.Zero              # 下面继续运行 [       OK ] FactorialTest.Zero (0 ms) 
[ RUN      ] FactorialTest.Positive 
[       OK ] FactorialTest.Positive (0 ms) 
[----------] 3 tests from FactorialTest (0 ms total)  # ...  
[----------] Global test environment tear-down [==========] 6 tests from 2 test suites ran. (0 ms total) [  PASSED  ] 5 tests.           [  FAILED  ] 1 test, listed below:     # 1个test失败 [  FAILED  ] FactorialTest.Negative    # 失败的test suite及其组   1 FAILED TEST
```

此外，在TEST宏函数中，也可以像个普通函数一样，定义变量之类的行为。

比如在sample2_unittest.cc中，测试一个自定义类MyString的复制构造函数是否表现正常：

```c
const char kHelloString[] = "Hello, world!";  // 在 TEST内部，定义变量TEST(MyString, CopyConstructor) {   
	const MyString s1(kHelloString);   
	const MyString s2 = s1;   
	EXPECT_EQ(0, strcmp(s2.c_string(), kHelloString)); 
}
```

为获得进一步学习，读者可以自行调整sample1_unittest.cc、sample2_unittest.cc中的TEST行为，加深对gtest的TEST宏的理解。

## 五、TEST_F

下面介绍gtest中更为高级的功能：test fixture，对应的宏函数是TEST_F(TestFixtureName, TestName)。

fixture，其语义是固定的设施，而test fixture在gtest中的作用就是为每个TEST都执行一些同样的操作。

比如，要测试一个队列Queue的各个接口功能是否正常，因此就需要向队列中添加元素。如果使用一个TEST函数测试Queue的一个接口，那么每次执行TEST时，都需要在TEST宏函数中定义一个Queue对象，并向该对象中添加元素，就很冗余、繁琐。

怎么避免这部分冗余的过程？

TEST_F就是完成这样的事情，它的第一个参数TestFixtureName是个类，需要继承testing::Test，同时根据需要实现以下两个虚函数：

virtual void SetUp()：在TEST_F中测试案例之前运行； virtual void TearDown()：在TEST_F之后运行。 可以类比对象的构造函数和析构函数。这样，同一个TestFixtureName下的每个TEST_F都会先执行SetUp，最后执行TearDwom。

此外，testing::Test还提供了两个static函数：

static void SetUpTestSuite()：在第一个TEST之前运行 static void TearDownTestSuite()：在最后一个TEST之后运行 以sample3-inl中实现的class Queue为例：

```c
class QueueTestSmpl3 : public testing::Test { // 继承了 testing::Test
	protected:        
	static void SetUpTestSuite() {     
		std::cout<<"run before first case..."<<std::endl;   
	}     

	static void TearDownTestSuite() {  
		std::cout<<"run after last case..."<<std::endl;   
	}      

	virtual void SetUp() override {    
		std::cout<<"enter into SetUp()" <<std::endl; 
		q1_.Enqueue(1);     
		q2_.Enqueue(2);    
		q2_.Enqueue(3);   
	}    

	virtual void TearDown() override {     
		std::cout<<"exit from TearDown" <<std::endl;   
	}      

	static int Double(int n) {    
		return 2*n;
	}      

	void MapTester(const Queue<int> * q) {     
		const Queue<int> * const new_q = q->Map(Double);
		ASSERT_EQ(q->Size(), new_q->Size());      
		for (const QueueNode<int>*n1 = q->Head(), *n2 = new_q->Head();          n1 != nullptr; n1 = n1->next(), n2 = n2->next()) 
		{       
			EXPECT_EQ(2 * n1->element(), n2->element());    
		}      
		delete new_q;   
	}    

	Queue<int> q0_;   
	Queue<int> q1_;   
	Queue<int> q2_; 
};
```

下面是sample3_unittest.cc中的TEST_F：

```c
// in sample3_unittest.cc
// Tests the default c'tor.
TEST_F(QueueTestSmpl3, DefaultConstructor) {   
// !!! 在 TEST_F 中可以使用 QueueTestSmpl3 的成员变量、成员函数    
	EXPECT_EQ(0u, q0_.Size()); 
}  

// Tests Dequeue().
TEST_F(QueueTestSmpl3, Dequeue) {   
	int * n = q0_.Dequeue();   
	EXPECT_TRUE(n == nullptr);    
	n = q1_.Dequeue();   
	ASSERT_TRUE(n != nullptr);   
	EXPECT_EQ(1, *n);   
	EXPECT_EQ(0u, q1_.Size());  
	delete n;    
	n = q2_.Dequeue();   
	ASSERT_TRUE(n != nullptr);   
	EXPECT_EQ(2, *n);   
	EXPECT_EQ(1u, q2_.Size());   
	delete n; 
}  
// Tests the Queue::Map() function.
TEST_F(QueueTestSmpl3, Map) {   
	MapTester(&q0_);   
	MapTester(&q1_);   
	MapTester(&q2_); 
}
```

以TEST_F(QueueTestSmpl3, DefaultConstructor)为例，再具体讲解下TEST_F的运行流程：

gtest构造一个QueueTestSmpl3对象t1； t1.setUp初始化t1 第一个TEST_F即DefaultConstructor开始运行并结束 t1.TearDwon运行，用于清理工作 t1被析构 因此，sample3_unittest.cc输出如下：

```c
% ./sample3_unittest Running main() from /Users/self_study/Cpp/OpenSource/demo/include/googletest/googletest/samples/sample3_unittest.cc 
[==========] Running 3 tests from 1 test suite. 
[----------] Global test environment set-up. 
[----------] 3 tests from QueueTestSmpl3 run before first case...    # 所有的test case 之前运行 [ RUN      ] QueueTestSmpl3.DefaultConstructor enter into SetUp()          # 每次都会运行 exit from TearDown 
[       OK ] QueueTestSmpl3.DefaultConstructor (0 ms) 
[ RUN      ] QueueTestSmpl3.Dequeue enter into SetUp()          # 每次都会运行 exit from TearDown 
[       OK ] QueueTestSmpl3.Dequeue (0 ms) 
[ RUN      ] QueueTestSmpl3.Map enter into SetUp()          # 每次都会运行 exit from TearDown [       OK ] QueueTestSmpl3.Map (0 ms) run after last case...      # 所有test case结束之后运行 [----------] 3 tests from QueueTestSmpl3 (0 ms total)  
[----------] Global test environment tear-down [==========] 3 tests from 1 test suite ran. (0 ms total) [  PASSED  ] 3 tests.
```

TEST_F相比较TEST可以更加简洁地实现功能测试。

gtest的基础入门教程就到此为止

## 六、MOCKER

对打桩的函数，使用MOCKER，可以按照预期值返回

示例：

```c
#include "gtest/gtest.h"
#include "mockercpp/mockercpp.hpp"

TEST_F(QueueTestSmpl3, xxx_test)
{
	MOCKER(hal_kernel_get_soc_type)
	.stub()
	.with(any(), outBoundP())
	.will(return (0));

	xxx_test();
	GlobalMockObject::verify();
}
```

C++中MOCKER类中的函数可以使用MOCKERCPP

GlobalMockObject::verify();是验证mock是否正常按照预期传入参数，清除后续mocker