# GP 泛型编程的强化 : 外部模板与非类型模板参数   

外部模板与非类型模板参数是现代 C++ 对泛型模板功能的补充, 前者用于消除特化模板符号重定义问题, 后者这是基础语法功能上的扩展.

---

## 外部模板  

在 C 语言中, `extern` 关键字用于扩展全局变量/函数的作用域. 通过使用 `extern` 声明一个外部的全局变量/函数, 使得我们可以跨编译单元调用全局变量/函数. C++ 继承了这一特性. 而在现代 C++ 中 `extern` 关键字得到了扩展, 除了变量与函数外, 也可以用来修饰一个模板. 其目的同样是为了扩展特化模板的作用域.

### `extern` 关键字与外部变量  

在 C/C++ 编译流程中, 每个源文件作为一个独立的编译单元进行编译, 将工程中所有编译单元编译过后才进行链接过程并生成可执行文件. 因此一个源文件中的全局变量, 默认情况下是无法被其他源文件(编译单元)所访问的. 参考下面示例:  

```cpp
// source.cpp                       // 编译单元 source
int number = 10;
int func() { return 20; }

// main.cpp                         // 编译单元 main
#include <iostream>
int main(int argc, char* argv[]) {
  std::cout << number << std::endl;
  std::cout << func() << std::endl;
  return 0;
}
```

上述这个工程在编译过程中, `main.cpp` 在编译阶段无法通过编译, 因为 `number` 变量与 `func` 函数在这个编译单元中是未定义的. 若我们希望其访问到 `test.cpp` 中的定义, 解决方案则是使用 `extern` 修饰声明, 如下: 

```cpp
// source.cpp                       // 编译单元 source
int number = 10;
int func() { return 20; }

// main.cpp                         // 编译单元 main
#include <iostream>

extern int number;                  // 声明外部全局变量 number
/* extern */ int func();            // 声明外部全局函数 func (函数默认 extern, 可省略关键字)

int main(int argc, char* argv[]) {
  std::cout << number << std::endl;
  std::cout << func() << std::endl;
  return 0;
}
```

我们在 `main.cpp` 中使用 `extern` 修饰符声明 `number` 变量与 `func` 函数, 以表明它们是在(该编译单元)外部所定义的全局变量与全局函数. 如此, 编译器将在链接阶段尝试使用在该编译单元外部定义的变量/函数, 即在 `source.cpp` 中所定义的 `number` 与 `func`. 若此时其他编译单元中均没有对其二者的定义, 则会咋链接阶段产生未定义引用错误(undefined reference).  

> 这一小结的内容与外部模板没有太大关系, 主要是为了介绍 `extern` 关键字的用法. 因为在 C++ 中这个关键字的使用频率远没有在其在 C 中那么高, 我们普遍更习惯用类中的静态变量/方法来实现一个跨编译单元的全局变量/方法.

### 使用 `extern` 实现外部模板声明  

如果了解了 `extern` 关键字的用法, 则不难将其与模板结合起来理解外部模板的概念了. 如果我们使用了一个泛化的模板, 而在该编译单元内不存在用户自定义的特化版本, 则在编译阶段将为其自动产生一个行为与泛化版本一致的特化版本. 而此时若在其他编译单元中存在用户自定义的特化版本, 则会在链接阶段产生符号重定义错误. 参考如下示例:  

```cpp
// header.hpp
#pragma once

template<class _Type>                       // 定义泛化模板
_Type func(_Type value) {
  return value;
}

// source.cpp                               // 编译单元 source
#include "test.h"

template<>                                  // 用户自定义的特化版本
int func(int value) {
  return value + value;
}

// main.cpp                                 // 编译单元 main
#include <iostream>
#include "test.h"

int main(int argc, char* argv[]) {
  std::cout << func<int>(10) << std::endl;  // 直接调用模板函数, 编译时自动产生一个特化版本
  return 0;
}
```

此时在 `source.cpp` 中存在一个模板函数 `func` 关于类型 `int` 的用户自定义特化版本, 而在 `main.cpp` 中由于无法访问到该特化版本, 因此在这个编译单元中将自动生成一个与泛化版本行为完全相同的特化版本. 如此, 我们的程序中实际存在了同一个函数的两个定义, 则自然地将在链接阶段产生符号重定义错误.  

若想要在 `main.cpp` 中访问到 `source.cpp` 中由我们自定义的特化版本, 则在需要 `main.cpp` 中使用 `extern` 关键字声明我们使用的特化版本是来自于外部定义的, 如下:  

```cpp
// main.cpp                                 // 编译单元 main
#include <iostream>
#include "test.h"

extern template int func<int>(int value);   // 声明模板函数 func 关于类型 int 的外部特化版本

int main(int argc, char* argv[]) {
  std::cout << func<int>(10) << std::endl;  // 由于声明过了外部模板, 不会自动产生特化版本
  return 0;
}
```

其逻辑与声明外部变量/外部函数相同, 需要注意的是声明外部模板是需要同时加上 `template` 关键字.  

> 部分编译器会主动在链接阶段删除掉多余的模板特化版本, 并且优先保留由用户自定义的特化版本(如 GCC). 因此在一些平台上, 外部模板的使用并不多. 但这并不是 C++ 标准所要求的, 如 MSVC 就不会进行上述优化. 

---

## 非类型模板参数  

传统 C++ 中模板参数只能指定数据类型, 自 C++ 11 起, 模板参数中还可以指定以下几种非数据类型 :  

- 布尔值;  
- 整数;  
- 整数指针;  
- 浮点数指针;  
- 枚举;  
- 全局函数指针;  
- 成员函数指针;  
- 对象指针;  

一种常见的使用场景是使用非类型模板参数限制容器的空间, 最典型的示例就是 `std::array` : 

```cpp
// GCC
template<typename _Tp, std::size_t _Nm>
struct array {
  typedef _Tp         value_type;
  ......
  typedef std::size_t size_type;
  ......
  constexpr size_type size() const noexcept { return _Nm; }
  constexpr size_type max_size() const noexcept { return _Nm; }
  ......
};

// main.cpp
#include <array>
int main(int argc, char* argv[]) {
  std::array<int, 128> numbers;
  return 0;
}
```

---