# GP 泛型编程的强化 : 可变长参数模板与折叠表达式  

传统 C++ 对泛型编程方式的支持程度, 相对于其他主流语言而言高出很多. 但即便是 C++ 中大量使用模板的工程也很少, 而完全以 GP 的代替 OOP 作为基本逻辑的工程更是寥寥无几. 这里的主要原因还是模板代码不太符合人的直观逻辑, 难以理解. 其次是因为传统 C++ 中依然缺少针对泛型编程的一些基础功能.  

在现代 C++ 中添加了很多 GP 的新内容, 这其中既包括了实际的语法功能, 也补全了一些传统 C++ 对 GP 核心功能的缺失.  

---

### 泛型编程 (Generic Programming)  

#### OOP 与 GP  

OOP 的基础思想在于对事物的"抽象化"与"具象化", 将一些行为与物类以人的直观思维归纳为抽象的类型, 而属于这个类型每一个具体的对象则拥有这个类型的特征.    

```csharp
// 抽象化
class Object {
  public void Method() {}
  public int attribute {
    get { return attribute; }
    set { attribute = value; }
  }
}
// 具象化
static int Main(string[] args) {
  Object obj = new Object();
  return 0;
}
```

GP 的基础思想在于对事物的"泛化"与"特化", 将某一行为或物类根据其在不同的环境中状态之间的共性抽象出一个泛化的版本, 并根据差异性分别提供不同环境下的特化的版本.  

```cpp
// 泛化
template<typename _Type>
_Type function(_Type value) {
  return value;
}
// 特化
template<>
int function<int>(int value) {
  return value + value;
}
```

#### 引申 : OOP 中与 GP 中多态的区别

OOP 中强调的多态属于动态多态, 多态现象出现在运行期; GP 中强调的多态属于静态多态, 多态现象出现在编译期.

提取针对某些类型共性再次抽象出类的基类, 在基类中提供通用版本, 并在派生类中提供特殊版本. 在使用时只需知道这些类的共性特征, 当前具体的对象决定将要使用的具体版本. 是为 OOP 的动态(运行期)多态.

```csharp
namespace System {
  public class Object {
    public virtual string? ToString() {  // 基础版本
      throw null;
    }
  }
  public readonly struct DateTime : ...... {
    public override string ToString() {  // 特殊版本
      ......
    }
  }
}
namespace Brabbit {
  public class Accout : System.Object {
    public override string ToString() {  // 特殊版本
      ......
    }
  }
  // 多态
  static int Main(string[] args) {
    Object obj_1 = new DateTime();  // DateTime obj_1
    obj_1.ToString();               // DateTime::ToString
    Object obj_2 = new Accout();    // Accout obj_2
    obj_2.ToString();               // Accout::ToString
    return 0;
  }
}
```

将行为或物类本身根据大多数环境进行泛化, 根据少部分环境进行特化. 使得在使用前只需知道抽象的目标行为或物类, 当前的所处环境决定这个未来的行为或物类的具体版本. 而当前的整体环境决定了未来有必要出现的具体版本. 是为 GP 的静态(编译期)多态.  

```cpp
namespace std {
  template<typename _Ty>  // 通用版本
  inline string to_string(_Ty _Val) {
    ......
  }
}
namespace brabbit {
  class Account {
    ......
  }
}
template<>  // 特殊版本
inline void std::to_string<brabbit::Account>(brabbit::Account account) {
  ......
}
// 多态
int main(int argc, char* argv[]) {
  std::to_string(int());               // std::to_string<int>(int());
  std::to_string(brabbit::Account());  // std::to_string<brabbit::Account>(brabbit::Account());
  return 0;
}
```

---

### 可变长模板

在传统 C++ 中, 并没有原生的可变长参数列表语法, 我们只能使用 C 语言提供的可变长参数列表即相关宏实现简陋的功能. 于是便有了于 C++11 被引入标准的可变长摸板机制, 用以解决传统 C++ 中定长参数列表的限制.  
与 Java, C# 等一众支持可变长参数列表语法的语言不同, C++ 利用可变长模板将参数列表展开的时机提早到了编译时, 并且利用模板本身的特性可实现不同类型不同长度的参数列表语法.  

模板形参前与函数形参声明时在前面加上 `...` 即表示此为可变长参数, 当调用可变长参数时需要在参数名后面加上 `...` . 使用 `sizeof...` 可统计变长参数个数, 当使用 `sizeof...` 或折叠表达式时无需再使用 `...` 修饰参数. 例子见下:  

```cpp
template<typename... _Types>
auto count_size(const _Types&... args) {
  return sizeof...(args);
}
```

该函数只有一个简单的功能, 即输出参数的个数. 通常提供一个可变长参数列表的语法同时还需提供统计参数个数的方式以及解包参数的方式. 这里我们已经知道如何实现前者了, 但后者的实现在 C++ 中有一段演进过程.  

#### 解包参数的方式 : 重载递归法  

现在我们来实现一个函数, 其用于输出所有传入的参数, 并且需要支持任意可输出的参数类型与个数. 最初的解包方式利用了递归与重载函数的原理, 步骤如下:  

1. 将可变长参数列表展开为一个参数与一个可变长参数包, 该函数用于解包;  
3. 重载一个只接受一个参数的模板函数, 该函数用于处理单个参数的情况;  
2. 利用递归的方式每次递归处理第一个参数, 后续参数包传入下一次递归;  

实现见下:  

```cpp
// 接受一个参数的重载, 用于处理每个参数
template<typename _Type>
inline void print_line(const _Type& last) {
  std::cout << last << std::endl;  // 处理单个参数
}
// 接受可变长参数的重载, 用于解包
template<typename _Type, typename... _Types>
inline void print_line(const _Type& first, const _Types&... others) {
  print_line(first);      // 处理首个参数
  print_line(others...);  // 递归后续参数
}

int main(int argc, char* argv[]) {
  print_line("I", "hate", "cpp", 666);  // => I \n hate \n cpp \n 666
  return 0;
}
```
我们知道在实现解包的过程中关键在于针对单个参数与多个参数的情况分别作出处理参数和展开参数包的处理. 递归重载法就是将不同的行为通过重载放入不同的函数, 而判断参数个数的情况则交由编译器去完成. 除非我们可以在编译期就执行分支判断语句, 否则这就是最优解.

> 重载递归法是最经典也最常见的一种做法. 额外需要记住的是这里的递归调用全部是编译期的行为, 而且它们同时也是内联函数, 当他们展开后在运行期里调用时只是几行输出语句罢了.

#### 解包参数的方式 : 常量分支递归法

或许你还记得在前文介绍 `constexpr` 时说到过, C++17 以后 `constexpr` 可用于修饰 `if` 分支语句, 以使得该分支的判断过程可以提前到编译期. 既然我们有了编译期可用的分支语法, 自然可以利用这个特性, 避免所有可变长参数的函数都需要写一个处理用的重载和一个解包用的重载. 相当于从编译器帮我们判断参数个数转为用户主动判断参数个数, 实现如下 :  

1. 将可变长参数列表展开为一个参数与一个可变长参数包;
2. 先处理首个参数, 再判断参数包内的参数个数 : 若后续有参数则递归处理, 否则执行完毕.

```cpp
// C++17
template<typename _Type, typename... _Types>
inline void print_line(const _Type& first, const _Types&... others) {
  std::cout << first << std::endl;
  if constexpr (sizeof...(others) > 0) {  // 判断后续参数个数
    print_line(others...);                // 递归处理
  }
}
```

常量分支递归法比起重载递归法的优势很明显, 不需要把处理逻辑生硬地拆分到两个函数中, 代码量也更少.  

#### 解包参数的方式 : 折叠表达式法

上述这两种解包方式都是利用其他的机制实现的功能, 比起其他语言原生提供解包方法还是有所不便. 因此, C++17 提供了原生的解参数包方式, 即"折叠表达式". 同样的功能使用折叠表达式的实现如下 :  

```cpp
// C++17
template<typename... _Types>
inline void print_line(const _Types&... args) {
  (std::cout << ... << args) << std::endl;  // (((std::cout << a) << b) << c) << std::endl;
}
int main(int argc, char* argv[]) {
  print_line("I", "hate", "cpp", 666);  // => Ihatecpp666
  return 0;
}
```

上面的语法被称作"二元左折叠表达式", 在编译期展开后将将如注释所示, 其最大的优势就在于避免了递归. "折叠表达式"的具体语法将在下一小节介绍. 这里我们可以简单对比一下几种编程语言关于可变长参数语法的设计差异 :  

上面已经展示过 C++ 的设计了, 下面我们看一下 C# 与 Java 的设计 : 

```cs
// C#
class Program {
  public static void PrintLine(params string[] args) {  // params 关键字
    foreach (var arg in args) {
      Console.Write(arg);
    }
  }
  static int Main(string[] args) {
    PrintLine("I", "hate", "csharp", Convert.ToString(666));  // => Ihatecsharp666
    return 0;
  }
}
```

```java
//Java
public class Main {
  public static void printLine(String... args) {  // ... 符号
    for (var arg : args) {
      System.out.print(arg);
    }
  }
  public static void main(String[] args) {
    printLine("I", "hate", "java", String.valueOf(666));  // => Ihatejava666
  }
}
```

相比之下, C# 与 Java 的优势是解包时拥有更灵活的语法, 而 C++ 的优势是可支持不同类型的参数一起传参并且运行期开销更小.

### 折叠表达式

折叠表达式是 C++ 17 起专门用于对可变长参数包进行解包工作的机制, 其支持四种语法形式 : 

|语法名|折叠状态|展开状态|
|-|-|-|
|一元右折叠|(... op pack)|(((pack1 op pack2) op pack3) ... op packN)|
|一元左折叠|(pack op ...)|(pack1 op (... op (packN-1 op packN)))|
|二元右折叠|(init op ... op pack)|(((init op pack1) op pack2) ... op packN)|
|二元左折叠|(pack op ... op init)|(pack1 op (... op (packN op init)))|

表中 `pack` 代表参数包名, `...` 代表参数包具体内容, 参数包名的位置代表着折叠方向.  
表中 `init` 代表初始值, 当表达式内存在 `init` 和 `...` 两个运算元时为二元折叠, 否则为一元折叠.  
表中 `op` 代表运算符, 一个表达式内可能会出现多次运算符, 但所有运算符必须是相同的.  

目前折叠表达式支持 32 个运算符 : 

| +  | -   | *  | /  | %  | ^  | &  | \|   | <  | >  | <<  |  >> |
|-|-|-|-|-|-|-|-|-|-|-|-|
| += | -=  | *= | /= | %= | ^= | &= | \|=  | <= | >= | <<= | >>= |
| =  |  == | != | -> | .  | ,  | && | \|\| |

---