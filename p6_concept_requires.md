# GP 泛型编程的强化 : 使用 `concept` 与 `requires` 进行泛型约束

传统 C++ 提供了很强大的泛型编程功能, 但是这个功能是难以约束的, 导致难读懂. 既然人都不容易看懂, 就更别指望编辑器与编译器能给我们多好的帮助了. 而 C++20 提供的 `concept` 与 `requires` 则提供了很好的约束模板的方式.  

请注意这里的"约束"概念, 如同上文所说的"多态"在 OOP 与 GP 中概念的差异一样, 此处的"约束"也有不同的含义:  
- OOP 中的"约束"是在运行前约束, 使得错误在进入运行期之前就暴露出来.
- GP 中的"约束"是在编译前进行约束, 使得错误在进入编译期之前就暴露出来.  

## 引入泛型约束前 C++ 中的编译期约束

在引入泛型约束前, C++ 中 GP 的约束与 OOP 一致, 只要错误能在程序运行之前暴露就相当于约束成功. 而所有的模板都将在编译阶段进行推导展开, 因此约束即代表着推导的成功或失败. 

现在我们来设计这么一个函数, 我们希望用它来统计一些数字值的总合. 根据之前介绍过的现代 C++ 内容, 我们可以这样实现 : 

```cpp
template<typename _Type, typename... _Types>
double sum(_Type arg, _Types... args) {
  if constexpr (sizeof...(args) > 0) {  // 静态分支递归解包法 C++17
    return arg + sum(args...);
  }
  return arg;
}

int main(int argc, char* argv[]) {
  // 推导展开后 : double sum(int arg, long args, float args, double args);
  std::cout << sum(1, 2L, 3.3F, 4.4) << std::endl;  // 理想的情况 => 10.7
  // 推导展开后 : double sum(int arg, const char* args);
  std::cout << sum(1, "string") << std::endl;       // 可能的情况 => 可编译, 但编译失败
  return 0;
}
```

在上面理想的使用方式里, 这个函数是可以正常编译通过的. 而在可能的情况里, 用户可能使用一个字符串传参, 此时编辑器不认为这种行为有任何问题的, 因此依然运行进行编译. 但显然编译器将展开失败, 因此最终还是导致编译失败.  

在泛型约束被引入之前, 我们认为这是约束成功的, 因为还没进入运行期就暴露出了问题. 但这也是传统 C++ 里模板没有被人们大量运用的原因之一, 所有使用到模板的地方都必须进入编译期后才能发现问题.  

## 使用 `requires` 进行泛型约束

C++20 引入了 `requires` 关键字, 它用于进行泛型约束, 也就是在进入编译期之前就进行约束. 语法为 :

`requires(expression)` 或 `requires expression` .  

括号写不写都合法. `requires` 表达式需要可推导出一个 `bool` 值. 你可以把它放在 `template` 语句与函数返回值之间, 也可以放在函数参数列表与函数体之间.

例如 : 

```cpp
template<typename _Type, typename..._Types>
requires std::is_same<_Type, int>::value  // 泛型约束 C++20
double sum(_Type arg, _Types...args) {
  if constexpr (sizeof...(args) > 0) {
    return arg + sum(args...);
  }
  return arg;
}
```

这里的 `std::is_same`, 用于检查两个类型是否一致. 它是 C++20 提供的一种 `Type Traits` , 我们可以把它简单理解为一种"泛型约束的规则" . 而 `std::is_same::value` 则是一个 `static constexpr const bool` 变量, 当它为真时才被认为通过约束. 此时再使用字符串传参将不会进入编译阶段就提示错误. 而常量被认为可能会在后续代码中使用到, 因此一些 IDE 也会提前将值推导出来(如 VS ). 

> 我们在这里只针对模板参数列表的第一个参数进行约束, 但由于函数体里采用递归的方式展开后续的参数包, 因此后续所有的参数当展开后最周都会作为某次递归调用的第一个参数传入函数. 从而所有参数在展开后都会被约束.  

当然, 这种约束将导致我们的函数只允许传入 `int` 值, 我们可以扩展一下这个表达式使其只允许传入基本数值类型 :  

```cpp
template<typename _Type, typename..._Types>
requires std::is_same<_Type, short             >::value ||  // 泛型约束 C++20
         std::is_same<_Type, int               >::value ||
         std::is_same<_Type, long              >::value ||
         std::is_same<_Type, long long         >::value ||
         std::is_same<_Type, unsigned short    >::value ||
         std::is_same<_Type, unsigned int      >::value ||
         std::is_same<_Type, unsigned long     >::value ||
         std::is_same<_Type, unsigned long long>::value ||
         std::is_same<_Type, float             >::value ||
         std::is_same<_Type, double            >::value
double sum(_Type arg, _Types...args) {
  if constexpr (sizeof...(args) > 0) {
    return arg + sum(args...);
  }
  return arg;
}

int main(int argc, char* argv[]) {
  std::cout << sum(1, 2L, 3.3F, 4.4) << std::endl;  // => 10.7
  // std::cout << sum(1, "string") << std::endl;    // 不会进入编译期
  return 0;
}
```

至此, 我们已经在完成目标函数的功能的同时使用泛型约束在编译前就将参数值限定在基本数值类型中.  

### 引申 : 萃取器 `Traits` 与"泛型约束规则"  

现代 C++ 都提供了许多中"泛型约束规则", 如上一小节中提到的 `std::is_same` . 这类功能其实由来已久, 如果我们检查它是实现不难发现, 它就是运用了 C++ STL 中 `Traits` 机制.  

**Traits 与 STL**  

当我们谈论 C++ STL 时, 主要是围绕着它 6 大板块 : 容器 Containers, 算法 Algorithms, 迭代器 Iterators, 仿函数 Functors, 适配器 Adapters, 分配器 Allocators. 而萃取器 Traits 仿佛不常被提及, 但在 STL 的内部使用中确实大量地使用了这个机制. 而一些涉及大量使用模板的工程里, 也经常需要使用这个机制.  

举一个最简单的例子, 假设我们需要实现一个函数, 它用于拷贝出任意可迭代 STL 容器对象的首个成员. 我们可以这么做 :  

```cpp
template<typename _Type>
_Type::value_type copy_front(_Type _Val) {  // 萃取出 _Type 类型容器内存放的数据类型
  if (_Val.empty()) {
    return _Type::value_type();
  }
  return _Val.front();
}

int main(int argc, char* argv[]) {
  std::list<std::string> string_list = {"test", "string"};
  std::vector<int> number_vector = {1, 2, 3};

  std::cout << copy_front(string_list) << std::endl;    // => "test"
  std::cout << copy_front(number_vector) << std::endl;  // => 1
  return 0;
}
```

这里我们使用了一个萃取器 `value_type` , 它将萃取出 _Type 类型容器内部存放的数据类型. 它的原理非常简单, 其实就是在推到容器具体的特化类型的同时记录下内部数据类型, 以 `std::vector` 为例, 我们查看它的头文件 :  

```cpp
template <class _Ty, class _Alloc = allocator<_Ty>>  // 第一个模板参数为内部数据类型
class vector { // varying size array of values
private:
  template <class>
  friend class _Vb_val;
  friend _Tidy_guard<vector>;

  using _Alty        = _Rebind_alloc_t<_Alloc, _Ty>;
  using _Alty_traits = allocator_traits<_Alty>;

public:
  static_assert(!_ENFORCE_MATCHING_ALLOCATORS || is_same_v<_Ty, typename _Alloc::value_type>,
    _MISMATCHED_ALLOCATOR_MESSAGE("vector<T, Allocator>", "T"));

  using value_type      = _Ty;  // 此处记录
......
}

int main(int argc, char* argv[]) {
  std::vector<int> _;  // _Ty == int =>  value_type == int
  return 0;
}
```

这是最基础的一种特征萃取方式, 随着可能的类型推导链越来越复杂, 也演化出各种各样的复杂特征萃取方式.  

**Traits 与 反射**  

如果你经常使用 Java 或 C# 等语言, 应该能对上面的行为产生一些共鸣. 它很像这些语言的"反射"机制. 但与这些语言在运行期进行的"动态反射"不同, `Traits` 机制实际上是一种很原始的"静态反射"机制. 它的原理完全依赖于编译器的类型推导.  

现代 C++ 始终没有提供反射机制因此显得与各大主流语言格格不入, 其中原因便是目前所有的动态反射机制都是让标准委员会难以接受的. C++ 标准委员会更希望引入一种标准的静态反射方案, 可喜的是目前已经有几个正在推进中提案了(如 Reflection TS). 目前几个主流 C++ 实现版本中, Clang 对于这方面的支持是最好的.  

## 使用 `concept` 自定义约束规则  

`concept` 关键字同样在 C++20 中被引入, 它运行用户自定义一套"泛型约束规则", 以实现对于规则的封装与复用. 它的基本语法如下 :  

- `template<template-param-list> concept concept-name expression;`  
- `template<template-param-list> concept concept-name requires(param-list) {expression-list};`  

以上文中的求总和函数为例, 我们可以把它对于基本数字类型的约束打包为一个 `concept` . 并在其他同样需要使用到基本数字类型约束的地方复用 :  

```cpp
template<typename _Type>
concept number_only =  // 定义约束规则 : 只允许基础数字类型 C++20
  std::is_same<_Type, short             >::value ||
  std::is_same<_Type, int               >::value ||
  std::is_same<_Type, long              >::value ||
  std::is_same<_Type, long long         >::value ||
  std::is_same<_Type, unsigned short    >::value ||
  std::is_same<_Type, unsigned int      >::value ||
  std::is_same<_Type, unsigned long     >::value ||
  std::is_same<_Type, unsigned long long>::value ||
  std::is_same<_Type, float             >::value ||
  std::is_same<_Type, double            >::value;

template<typename _Type, typename..._Types>
requires number_only<_Type>  // 使用约束规则 : 只允许基础数字类型
double sum(_Type arg, _Types...args) {
  if constexpr (sizeof...(args) > 0) {
    return arg + sum(args...);
  }
  return arg;
}

template<number_only _TypeLeft, number_only _TypeRight>  // 使用约束规则(语法糖) : 只允许基础数字类型
auto subtract(_TypeLeft left, _TypeRight right) {
  return left - right;
}

int main(int argc, char* argv[]) {
  std::cout << sum(1, 2L, 3.3F, 4.4) << std::endl;  // => 10.7
  std::cout << subtract(123, 23.0) << std::endl;    // => 100.0
  return 0;
}
```

`concept` 实现了对推导规则的封装, 它的用法非常多变, 到这里我们仅仅只是介绍了其中的两种写法.  

## 使用 `std::is_same` 与 `std::decay` 进行类型约束  

标准库中提供了许多对于类型的约束方式, 它们大都保存在头文件 `<type_traits>` 里面. 这里我们只介绍两种比较简单的 `std::is_same` 与 `std::decay` .  

`std::is_same` 在前文中已经出现过了. 以上一小节中的 concept number_only 为例, 我们希望用它来过滤所有非基础数字类型. 但这个实现方式依然存在缺陷. 因为 `std::is_same` 的检查规则是非常严格的, 针对基本变量类型加入任何修饰符号都会被它视作不同的类型. 因此如果我们使用了这个 number_only 约束, 但传参时却以引用等方式传递, 将导致不通过约束 :  

```cpp
int main(int argc, char* argv[]) {
  int number = 100;
  std::cout << subtract<int, int>(number, number) << std::endl;         // OK
  // std::cout << subtract<int&, int&>(number, number) << std::endl;    // Failed
  return 0;
}
```

解决方案是在 `concept` 中使用 `std::decay` 萃取器将模板参数类型退化为最基础的参数类型. 它用于将除了 `*` 以外的所有修饰符移除. 它的实现方式就是一个封装类型推导方式的结构体, 我们可以通过 `std::decay::type` 获取其推导后的类型 :  

```cpp
template<typename _Type>
concept number_only =  // C++20
  std::is_same<typename std::decay<_Type>::type, short             >::value ||
  std::is_same<typename std::decay<_Type>::type, int               >::value ||
  std::is_same<typename std::decay<_Type>::type, long              >::value ||
  std::is_same<typename std::decay<_Type>::type, long long         >::value ||
  std::is_same<typename std::decay<_Type>::type, unsigned short    >::value ||
  std::is_same<typename std::decay<_Type>::type, unsigned int      >::value ||
  std::is_same<typename std::decay<_Type>::type, unsigned long     >::value ||
  std::is_same<typename std::decay<_Type>::type, unsigned long long>::value ||
  std::is_same<typename std::decay<_Type>::type, float             >::value ||
  std::is_same<typename std::decay<_Type>::type, double            >::value;
```

这样一来, 所有的 `const` , `&` , `&&` 修饰符都会被移除, 因此经过这些符号修饰过的基础数字类型都能通过这个约束了. 需要注意的是此处需要使用 `typename` 声明 `std::decay<_Type>::type` 是一个类型, 因为在进行类型推导前我们是不确定它究竟是类型还是变量或函数, 此处实际上是消除歧义. 

> 经过了 `std::decay` 的类型退化, 我们可以移除所有 `const` , `&` , `&&` 符号, 但不包括 `*` 符号. 若是想移除 `*` 符号, 可以使用 `std::remove_pointer` 萃取器.

## 使用 `concept` 与 `requires` 进行行为约束  

C++20 提供的泛型约束主要通过使用 `concept` 与 `requires` 进行. 上文中我们介绍了如何去约束模板参数的类型, 其实它们其实还可以针对模板类型对象的行为进行约束.  

当定义约束时, 语法上 `concept` 与 `requires` 可以结合使用, 此时可以对模板类型对象的行为进行约束. 以上文中拷贝容器首个元素的函数为例 , 原函数如下 : 

```cpp
template<typename _Type>
_Type::value_type copy_front(_Type _Val) {  // 萃取出 _Type 类型容器内存放的数据类型
  if (_Val.empty()) {
    return _Type::value_type();
  }
  return _Val.front();
}
```

该函数目前没有使用任何约束, 但函数体内却调用了 `empty` 与 `front` 方法. 若不加以约束参数的行为, 我们是无法在编译前就暴露错误的. 我们可以使用 `concept` 与 `requires` 约束参数必须有这两个方法, 改进方式如下: 

```cpp
// 约束 _Type 类对象必须有 front 方法
template<typename _Type>
concept has_front = requires(_Type _Val) {
  _Val.front();
};
// 约束 _Type 类对象必须有 empty 方法
template<typename _Type>
concept has_empty = requires(_Type _Val) {
  _Val.empty();
};
// 将所有条件列出, 每个条件都必须符合
template<typename _Type>
concept has_empty_and_front = requires(_Type _Val) {
  _Val.front();  // 约束 _Type 类对象必须有 front 方法
  _Val.empty();  // 约束 _Type 类对象必须有 empty 方法
};
// 将约束进行组合, 约束 _Type 类对象必须有 front 和 empty 方法
template<typename _Type>
concept has_empty_and_front_ex = has_front<_Type>
                              && has_empty<_Type>;

// 新函数
template<has_empty_and_front_ex _Type>
_Type::value_type copy_front(_Type _Val) {
  if (_Val.empty()) {
    return _Type::value_type();
  }
  return _Val.front();
}

// 使用
int main(int argc, char* argv[]) {
  // 有 front 方法, 有 empty 方法 => 通过约束
  std::list<std::string> string_list;
  std::cout << copy_front(string_list) << std::endl;

  // 没有 front 方法, 没有 empty 方法 => 不通过约束
  struct Foo {} foo;
  copy_front(foo);

  // 没有 front 方法, 有 empty 方法 => 不通过约束
  std::map<int, std::string> number_string_map;
  copy_front(number_string_umap);
  return 0;
}
```

上述的各种定义约束的方式都是可行的, 至此我们就完成了对所有非法行为的约束了.   

至此, 我们也介绍完了 C++20 中泛型约束的基本用法, 当然文中提及的也只是冰山一角, 实际的功能与用法是比这更强大更具多样性的. 

---
