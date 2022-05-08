# `auto` 与 `decltype` : 类型推导  

变量类型推导其实在 C++ 中一直存在, 例如我们在使用泛型函数时编译器将帮助我们隐式地推导参数类型 :  

```cpp
template<typename _Type>
void func(_Type value) { /* do something */ }
func(114514);  // func<int>(114514);
```

但直到 C++11 起才允许用户主动要求编译器进行类型推导. 现代 C++ 中提供的主动类型推导功能主要是通过 `auto` 的 `decltype` 两个关键字实现.  

---

## 使用 `auto` 进行变量类型推导  

当你声明一个变量为 `auto` 类型时, 编译器将自动帮助你推导出合适的数据类型. 这个变量可以是 :  

1. 声明后立即赋值的普通变量;  
2. 函数的返回值;  
3. 函数的形参 (C++14 起)  

需要注意的是, 使用 `auto` 进行类型推导时, 将忽略顶层的 `const` , `&` , `*` 等修饰符, 以便用户更细化的控制推导, 示例见下 : 

```cpp
int number = 8;
auto var_1 = number;       // int var_1 = number;
auto var_2 = 8.0f;         // float var_2 = 8.0f;
auto var_3 = var_1;        // int var_3 = var_1;
const auto var_4 = var_2;  // const float var_4 = var_2;
auto& var_5 = number;      // int& var_5 = number;
```

这种基础的用法主要是用于省略一些很长的类型名, 一定程度上增加代码可读性. 一种经典用法是简写迭代器类型以遍历容器, 传统 C++ 中, 我们需要完整写出迭代器的类型或是使用局部的 `typedef` 进行简写, 如下 :

```cpp
std::unordered_map<std::string, std::string> key_value_map;
// C++98/03, 迭代器遍历容器完整写法
for (std::unordered_map<std::string, std::string>::const_iterator itor = key_value_map.begin(); itor != key_value_map.end(); itor++) {
  std::cout << itor->first << " " << itor->second << std::endl;
}
// C++98/03, 使用迭代器别名进行缩写
typedef std::unordered_map<std::string, std::string>::const_iterator unordered_map_const_itor;
for (unordered_map_const_itor itor = key_value_map.begin(); itor != key_value_map.end(); itor++) {
  std::cout << itor->first << " " << itor->second << std::endl;
}
```

而现代 C++ 中, 一方面我们通常使用 `using` 代替 `typedef`, 但更方便的方式是使用 `auto` 简写迭代器类型 :  

```cpp
// C++11 起, 使用 using 代替 typedef
using hash_map_const_itor = std::unordered_map<std::string, std::string>::const_iterator;
for (hash_map_const_itor itor = key_value_map.begin(); itor != key_value_map.end(); itor++) {
  std::cout << itor->first << " " << itor->second << std::endl;
}
// C++11 起, 使用 auto 自动推导迭代器类型
for (auto itor = key_value_map.begin(); itor != key_value_map.end(); itor++) {
  std::cout << itor->first << " " << itor->second << std::endl;
}
```

这里我们稍微引申一下, 在现代 C++ 中迭代一个容器的方式还有很多, 具体可见下面的例子 : 

```cpp
// C++11 起, auto + 范围 for 循环
for (auto& pair : key_value_map) {
  std::cout << pair.first << " " << pair.second << std::endl;
}
// C++17 起, auto + 范围 for 循环 + 结构化绑定
for (auto& [first, second] : key_value_map) {
  std::cout << first << " " << second << std::endl;
}
// C++20 起, auto + 范围 for 循环 + 结构化绑定 + range 机制, 细化控制方式 (MSVC /std:c++laest)
for (auto& [first, second] : key_value_map | std::views::all) {  // 全部遍历
  std::cout << first << " " << second << std::endl;
}
for (auto& [first, second] : key_value_map | std::views::reverse) {  // 全部倒序遍历
  std::cout << first << " " << second << std::endl;
}
for (auto& [first, second] : key_value_map | std::views::drop(2)) {  // 顺序遍历忽略前两个
  std::cout << first << " " << second << std::endl;
}
```

除了普通的变量外, 我们还可以使用 `auto` 设置函数的返回值, 此时需要我们在参数列表后使用 `->` 符号标注具体的返回值类型. 这种写法被称作"返回类型后置语法", 在一些脚本语言中比较常见 : 

```cpp
auto add(int left, int right)->int { return left + right; }

// 引申 : lambda 表达式的书写格式借鉴了返回值后置语法
auto lambda = [](int left, int right)->int { return left + right; }
```

到了 C++14, 我们甚至可以使用 `auto` 进行参数类型推导, 在传统 C++ 中我们想进行参数类型的推导需要用到泛型机制, 但有了 `auto` 进行推导参数类型后, 我们可以简化一些工作 : 

```cpp
// C++98/03
template<typename _Type>
void print_value(_Type value) { std::cout << value << std::endl; }
// C++20
void print_value(auto value) { std::cout << value << std::endl; }
```

`auto` 关键字在单独使用时, 大部分是用于简化书写 当它与其它机制结合使用时可以衍生出更多的功能, 同样在下文里细说.  

---

## 使用 `decltype` 进行表达式类型推导  

`auto` 关键字用于推导变量类型, 与之相对的 `decltype` 则是用来推导表达式结果的类型. 熟悉 GCC 的用户可能对这个关键字不陌生, `decltype` 的标准化提案就是源自 GCC 的扩展关键字 `__decltype`, 而后者又是源自于 GCC 一个很古老的扩展关键字 `__typeof__` . 例如你可能需要推导某两个变量相加的结果类型 :  

```cpp
auto number_a = 16LL;                                          // long long number_a = 16LL;
auto number_b = 16.0F;                                         // float number_b = 16.0F;
decltype(number_a + number_b) number_c = number_a + number_b;  // float number_c = number_a + number_b;
```

`decltype` 的推导规则遵循如下几点 : 

1. 若表达式是一个 不带括号的标记符表达式 或 类/结构体成员访问表达式, 那么推导的结果是所代表实体的类型;  
2. 若表达式是一个函数调用(包括操作符重载), 那么推导的结果是函数的返回类型, 若返回值是基础类型则抛弃 `const` 限定符;  
3. 若表达式是一个字符串字面量, 则推到为 `const` 左值引用;  
4. 上述情况以外, 若表达式结果为左值则推导为左值引用, 否则推导为本类型;  

示例见下 :  

```cpp
int number = 4;
const int const_number = 8;
int number_array[2] = { 0 };
int* number_array_ptr = number_array;
struct MyStruct { double member; } my_struct;
const bool func_1(int) { return true; };
const MyStruct func_2(int) { return MyStruct(); };

// 规则 1 : 不带括号的标记符表达式 或 类/结构体成员访问表达式
decltype(number_array) var_1;      // int[2] var_1; 标记符表达式 => 本类型
decltype(number_array_ptr) var_2;  // int*   var_2; 标记符表达式 => 本类型
decltype(my_struct.member) var_3;  // double var_3; 成员访问表达式 => 本类型

// 规则 2 : 函数调用
decltype(func_1(1)) var_5 = true;       // bool var_5; 基础类型返回值, 丢弃 const 限定符
decltype(func_2(1)) var_7 = func_2(1);  // const MyStruct var_7; 类类型返回值, 保留 const 限定符

// 规则 3 : 字符串字面量
decltype("hello") var_8 = "hello";      //const char& var_8[6]; const 左值引用

// 规则 4 : 其他情况下表达式结果
decltype((number)) var_9 = number;                 // int& var_9; 带括号的标记符表达式 => 左值引用
decltype(true ? number : number) var_10 = number;  // int& var_10; 条件表达式返回左值 => 左值引用
decltype(++number) var_11 = number;                // int& var_11; 表达式结果为左值 => 左值引用
decltype(number_array[5]) var_12 = number;         // int& var_12; 表达式结果为左值 => 左值引用
decltype(*number_array_ptr) var_13 = number;       // int& var_13; 表达式结果为左值 => 左值引用
decltype(1) var_14 = 10;                           // int var_14; 纯右值字面量 => 本类型
decltype(number++) var_15 = number;                // int var_15; 表达式结果为右值 => 本类型
```

你可能在部分平台上使用过关键字 `typeof` , `__typeof__` 或 `__decltype` , 它们同样可用于推导表达式结果类型, 并且可以视作 `decltype` 功能的子集. 但这些关键字从来都不是标准 C++ 的一部分, 只是部分编译器支持的功能, 并且它们的推导规则也有很强的平台差异性. 相比之下 `decltype` 的标准化程度和适用面更广. 除此之外 `typeof` 进行类型推导时 `&` 引用符号很可能将不做保留, 至少 GCC 上是这样的, 参考以下示例 :  

```cpp
int           var = 1; 
int&          ref = var; 
typeof(var)   var_1 = 1;  // int GCC 4.3.x
typeof(ref)   var_2 = 1;  // int GCC 4.3.x
decltype(var) var_3;      // int 
decltype(ref) var_4 = a;  // int& 
```

总之, 当你的工程所使用的 C++ 版本若是等于或高于 C++11 , 我推荐全盘使用 `decltype` 代替 `typeof` .  

---

## 结合 `auto` 与 `decltype` 进行自动推导返回值类型  

`auto` 与 `decltype` 单独使用的时候, 在功能上的突破本质还是向用户开放了主动要求类型推导的权限. 但如果二者结合使用的话, 就可以突破传统 C++ 中一些限制了.  在这里我们思考一个问题 : **如何实现一个满足所有类型之间进行 `+` 运算的函数?** 

在传统 C++ 中的最优解是这样的 : 

```cpp
// C++98/03
template<typename _TypeLeft, typename _TypeRight, typename _TypeResult>
_TypeResult add(_TypeLeft left, _TypeRight right) {
  return left + right;
}
```

这种做法的确可以满足所有类型的 `+` 运算, 但很明显使用上有着很大的局限性. 因为我们必须预知返回值的类型, 而题目的隐藏含义是一定要做到通用性的. 那么我们现在已经知道如何使用 `decltype` 可以进行表达式结果的类型推导, 那何不直接用其直接推导函数体的结果呢 ? 

```cpp
template<typename _TypeLeft, typename _TypeRight, typename _TypeResult>
decltype(left + right) add(_TypeLeft left, _TypeRight right) {  // 未定义的标识符
  return left + right;
}
```

很遗憾这种行为是无效的, 因为形式参的定义在参数列表里, 而处于参数列表左侧的返回值类型是无法获取形参名的. 除非我们能将返回值类型放在参数列表的右侧, 实现这个目标的方式就是使用 `auto` 书写返回类型后置语法 :  

```cpp
// auto 返回类型后置语法
template<typename _TypeLeft, typename _TypeRight, typename _TypeResult>
auto add(_TypeLeft left, _TypeRight right)->_TypeResult {
  return left + right;
}
// auto 返回类型后置语法 结合 decltype 表达式结果类型推导
template<typename _TypeLeft, typename _TypeRight>
auto add(_TypeLeft left, _TypeRight right)->decltype(left + right) {
  return left + right;
}
```

若只是单纯的只用 `auto` 进行返回类型后置, 则只是换了种语法. 但如果将 `auto` 与 `decltype` 相结合, 就可以突破传统 C++ 的限制了. 到此, 我们已经完全实现了题目里的需求. 但还有继续优化的空间, 首先是上文中提到的, 自 C++14 起, 我们可以利用 `auto` 进行参数类型的推导 :  

```cpp
// C++14
auto add(auto left, auto right)->decltype(left + right) { return left + right; }
```

其次是由于这种二者结合的模式被大量的使用, 自 C++14 起, 我们在使用 `auto` 描述返沪类型时无需在参数列表后写上返回类型, 编译器将自动通过函数体进行推导, 因此这个方法最终将演化成这种形式 :  

```cpp
// C++14
auto add(auto left, auto right) { return left + right; }
```

**引申 : 上例中的 "最终版本" 真的完美吗?**

> 返回类型后置这种语法看似只是语法上的一些取巧手法, 但实则通过这种方式可以突破编译器的桎梏, 因为编译器始终是由上至下由左至右理解代码的.

正如上文中所说, `auto` 与 `decltype` 的功能并不是现代 C++ 才出现的, 而且在它们单独使用时更多的时候是一种简化代码书写的方式. 但当二者结合起来时, 将可以做出一些语言功能上的突破.

---

## `auto` 与 `decltype` 的演化  

传统 C++ 里, 类型推导一般是在模板传参时进行隐式类型推导, 而 `auto` 的出现是将类型推导的控制权开放给用户进行显示类型推导; 而传统 C++ 里表达式结果类型的推导通常由不同平台上各种 `typeof` 非标准扩展关键字实现, 而 `decltype` 的出现则是这个功能的标准化. 当 `auto` 与 `decltype` 相结合后, 由衍生出许多新的功能, 它们二者构成了现代 C++ 类型推导功能的核心. 以下的 `auto` 与 `decltype` 的发展历程简述 :  

1. C++11 :  
    - 允许使用 `auto` 进行普通变量的主动类型推导;
    - 允许使用 `auto` 书写返回值后置语法;
    - 使用标准 `decltype` 进行表达式结果类型推导以替代各平台的 `typeof` 扩展关键字;  
2. C++14 :  
    - 允许使用 `auto` 进行形参类型推导;
    - 允许使用 `auto` 进行返回值类型推导 (书写返回值后置语法时不适用类型标识符表面返回类型);

---