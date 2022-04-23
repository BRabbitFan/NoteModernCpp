# `constexpr` : 真正的 "常量"  

`constexpr` 关键字在 C++11 中被引入标准, 它用于定义一个常量. 这里常量的含义是编译器即可确定的结果, 这个结果可以是 : 

1. 变量的值
2. 函数发返回值
3. 条件分支的判断结果 (C++17 起)  

`constexpr` 与传统 C++ 中名字相似的 `const` 相比, 二者并不是子集与超集的关系, 但在语义上确有一些交集. 

**如何定义一个常量?**

在传统 C++ 中, 主要分为两个流派: 宏定义 或 常量 `const`.  

```cpp
#define CONST_NUMBER 128
const int Const_Number = 128;
```

而在现代 C++ 中, 更推荐使用 `constexpr` 定义常量, 以替代宏定义与常量 `const` .   

```cpp
constexpr int Constexpr_Number = 128;
```

---

### 使用 `constexpr` 避免宏定义的污染性  

预处理语句无视代码块的访问限制, 这将污染代码, 并可能在难以预见的地方留下隐患. 现假设如下情况 : 两个进行联合编译的工程内均使用了一个常量 : 当前平台渲染器的最大数量. 代码如下:  

```cpp
// renderer project
namespace emeet::media {  // C++17 新语法 : 嵌套命名空间缩写
#define MAX_RENDERER_NUMBER 64
}
// main project
namespace emeet {
//const int MAX_RENDERER_NUMBER = 128;  // error   : const int 64 = 128
#define MAX_RENDERER_NUMBER 128         // warning : 宏重定义. 这可能会破坏 media 的配置
}
```

即便这个同名常量在两个语境下可能表达的具体含义是不一样的, emeet::media 中的宏依然会污染到主要工程. 这种情况下使用 `const` 定义常量可能会与宏的命名冲突, 当然这还只是小问题, 改一个名字就好了. 但如果我们定义了同名的宏, 则可能会在运行期影响 emeet::media 的配置.

又如微软的经典历史遗留问题 : 

```cpp
// stdlib.h : 
// #ifndef __cplusplus  // msvc 使用 _MSVC_LANG 标识语言版本, 默认情况下 __cplusplus 永远不会被定义
//   #define max(a,b) (((a) > (b)) ? (a) : (b))
//   #define min(a,b) (((a) < (b)) ? (a) : (b))
// #endif

// untility.h : 
// constexpr const _Ty&(max) (const _Ty& _Left, const _Ty& _Right, _Pr _Pred) noexcept;
// constexpr const _Ty&(min) (const _Ty& _Left, const _Ty& _Right, _Pr _Pred) noexcept;

#include <windows.h>

template<typename _Type = int>
_Type limit(const _Type& min, const _Type& val, const _Type& max) {
  return std::max(min, std::min(val, max));  // 加上 #include <windows.h> 后此处非法标记
}

// 参照 untility.h 的安全做法
template<typename _Type = int>
_Type limit(const _Type& min_val, const _Type& val, const _Type& max_val) {
  return (std::max)(min_val, (std::min)(val, max_val));
}
```

使用 `constexpr` 定义一个常量不会造成代码的污染 :  

```cpp
// renderer project
namespace emeet::media {
constexpr int Max_Render_Number = 64;
}
// main project
namespace emeet {
constexpr int Max_Render_Number = 128;
}
```

---

### 使用 `constexpr` 避免 `const` 的双重含义 : "常量" 与 "只读变量"  

定义常量的目的 : 在编译期就确定值, 从而达成运行期零开销  

```cpp
// 三者均为运行期零开销的常量
#define CONST_NUMBER 128;
const int Const_Number = 128;
constexpr int Constexpr_Number = 128;
```

但使用 `const` 关键字时, 需要判断其是常量还是只读量:  

```cpp
// lib.h
int add_one(const int& const_value);

// main.cpp
int main(int argc, char* argv[]) {
  int number = 128;
  const int Const_Number_1 = 128;     // 常量
  const int Const_Number_2 = number;  // 只读量 : readonly int Const_Number_2 = number;

  std::cout << Const_Number_1 << std::endl;  // 128
  std::cout << Const_Number_2 << std::endl;  // 128

  std::cout << add_one(Const_Number_1) << std::endl;  // 129
  std::cout << add_one(Const_Number_2) << std::endl;  // 129

  std::cout << Const_Number_1 << std::endl;  // 128 常量的值在编译期已经确定, 即使被解 const 也不会修改值
  std::cout << Const_Number_2 << std::endl;  // 129 只读变量的值在运行期确定, 可以被解 const 修改
}

// lib.cpp
int add_one(const int& const_value) {
  int& value = const_cast<int&>(const_value);
  value++;
  return value;
}
```

> C 中的 const 是只读变量，有自己的储存空间，能通过地址间接修改其的值;  
> C++ 中的 const 是常量或只读变量, 存放在符号表中. 当对其进行引用时才会临时分配栈空间.  

使用 `constexpr` 可以区分常量与只读变量 :  

```cpp
int number = 128;                            // ok    : 变量, 在运行期确定值
const int Const_Number = number;             // ok    : 只读量, 在运行期确定值
constexpr int Constexpr_Number = 128;        // ok    : 常量, 在编译期确定值
// constexpr int Constexpr_Number = number;  // errer : 无法在编译期获得一个运行期才确定的值
```

上述这些案例, 只要你足够细心, 理论上也是可以使用 `const` 完全替代 `constexpr` 的, 但下面这些情况就是只能使用 `constexpr` 而非 `const` 了.

---

### 使用 `constexpr` 修饰函数与分支  

`constexpr` 关键字除了可以对变量进行修饰外, 还可以对函数与分支语句进行修饰.  

使用 `constexpr` 进行修饰的函数, 其返回值必须是编译期就可以计算出来的. 这意味着所有参与计算出返回值的变量都必须是编译期就可以确定其值的. 就如同通过一个对象的 `const` 引用只能访问 `const` 成员函数一样.

```cpp
int constexpr count_array_max_size() {
  return 1024;
}
int constexpr count_array_max_size(int arg_1, int arg_2) {
  return arg_1 + arg_2;
}

int number = 128;
constexpr int Constexpr_Number = 128;

std::array<int, count_array_max_size()> int_array_1;                      // ok : max size == 1024;
std::array<int, count_array_max_size(10, 10)> int_array_2;                // ok : max size == 20;
std::array<int, count_array_max_size(Constexpr_Number, 10)> int_array_3;  // ok : max size == 138;
// std::array<int, count_array_max_size(number, 10)> int_array_4;
// error : 无法通过运行期确定值的参数计算出编译期确定值的返回值;
```

`constexpr if` 常量分支在 C++17 时被引入标准库. 使用 `constexpr` 修饰分支语句时, 判断条件也必须是编译期就可以确定的. 如同修饰函数一样, 所有参数者都必须是一个常量.  

```cpp
bool constexpr check_condition() { return false; }

bool condition = true;
constexpr bool Coustexpr_Condition = true;

if constexpr (true)                { /* do something */ }  // ok    : 编译期可确定判断结果
if constexpr (Coustexpr_Condition) { /* do something */ }  // ok    : 编译期可确定判断结果
if constexpr (check_condition())   { /* do something */ }  // ok    : 编译期可确定判断结果
// if constexpr (condition)        { /* do something */ }  // error : 编译期内无法通过运行期的值确定结果
```

---

### 引申 : 利用编译器对 `constexpr` 的优化策略提高代码可读性  

编译器对判断一段代码是否可被安全的优化这件事, 很大程度上取决于这段代码所使用的修饰符. 如被 `constexpr` 或 `const` 修饰过的变量, 编译器会尽可能地在编译期内就确定变量的值; 又如被 `inline` 修饰过的函数, 编译器将更倾向于在调用该函数的地方直接展开函数而非进行函数调用.  
但是不同的关键字对代码优化起到的影响效果也是不一样的, 对于 `inline` 编译器大多数情况下只是将其视作"建议", 对于 `const` 编译器则需要严格地把编译期间所有可计算值的变量加入符号表内. 反观 `constexpr` , 这是一个约束性比 `const` 更强的关键字, 因此编译器甚至会将 `constexpr` 修饰过的函数返回值都放入符号表内.  

在遇到某些复杂但可预测的数学计算时, 利用这一特性可以极大提高代码可读性. 以下是约翰卡马克在 <<雷神之锤3>> 中写的的快速求平方根倒数算法 : 

```cpp
// /code/game/q_math.c
float Q_rsqrt( float number )
{
	long i;
	float x2, y;
	const float threehalfs = 1.5F;
 
	x2 = number * 0.5F;
	y  = number;
	i  = * ( long * ) &y;						// evil floating point bit level hacking
	i  = 0x5f3759df - ( i >> 1 );               // what the fuck?
	y  = * ( float * ) &i;
	y  = y * ( threehalfs - ( x2 * y * y ) );   // 1st iteration
//	y  = y * ( threehalfs - ( x2 * y * y ) );   // 2nd iteration, this can be removed
 
#ifndef Q3_VM
#ifdef __linux__
  assert( !isnan(y) ); // bk010122 - FPE?
#endif
#endif
	return y;
}
```

这个函数的效率是标准库版本的 4 倍. 而最令人费解的地方就在于那个神秘数字 0x5f3759df, 为了解释这个数字的来由, 一些学者甚至为此发表过论文. 从现代 C++ 的眼光来看, 若能够利用上编译器对 `constexpr` 函数优化的特性, 则可以在不增加运行期开销的情况下极大增强代码可读性 : 

```cpp
int constexpr count_magic_number() {
  int result = 0x00000000;
  // 计算过程
  return result;
}

float Q_rsqrt( float number )
{
	long i;
	float x2, y;
	const float threehalfs = 1.5F;
 
	x2 = number * 0.5F;
	y  = number;
	i  = * ( long * ) &y;						// evil floating point bit level hacking
  	// i  = 0x5f3759df - ( i >> 1 );               // what the fuck?
	i  = count_magic_number() - ( i >> 1 );     // 不会产生运行期的额外开销
	y  = * ( float * ) &i;
	y  = y * ( threehalfs - ( x2 * y * y ) );   // 1st iteration
//	y  = y * ( threehalfs - ( x2 * y * y ) );   // 2nd iteration, this can be removed
 
#ifndef Q3_VM
#ifdef __linux__
  assert( !isnan(y) ); // bk010122 - FPE?
#endif
#endif
	return y;
}
```

> 对于神秘数字 0x5f3759df, 相关论文对其的解释是这个函数利用了 内存位运算时的工作原理 与 牛顿迭代法的特性, 把精度控制在常用浮点数可表示的范围内. 想了解具体原理可以点击下面的链接:  
https://en.wikipedia.org/wiki/Fast_inverse_square_root
https://wenku.baidu.com/view/80b84d1fb7360b4c2e3f644b.html
https://zhuanlan.zhihu.com/p/74728007  
https://blog.csdn.net/xtlisk/article/details/51249371  

---

### `constexpr` 的演化  

`constexpr` 一开始只是为了修复 `const` 拥有双重含义这个历史遗留问题. 但随着后续的逐步发展, 它已成为现代 C++ 的重要关键字. 以下是 `constexpr` 的发展历程简述 :  

- C++11 引入 `constexpr` 关键字, 允许修饰变量与函数;  
- C++14 起使用 `constexpr` 修饰类成员函数的前提是该成员函数已经被 `const` 修饰过;  
- C++14 起允许使用 `constexpr` 修饰 `std::array` , `std::tuple` , `std::chrono` 等类型;  
- C++17 起允许使用 `constexpr` 修饰分支语句以及 lambda 表达式;  
- C++20 起允许在进行 `constexpr` 评估时使用临时的 `std::vector` , `std::string` 等类型变量;  

> 虽然 `std::vector` , `std::string` 等类型可以被 `constexpr` 利用, 但不代表它们的构造函数可以在编译期间执行. 这本质上是让这些组件在设计上支持常量这个特性 (主要是通过对 `std::allocator` 进行特殊设计), 而非 `constexpr` 本身功能扩展了.

---