---
title: C#基础
date: 2022-08-08 21:57:53
categories: C#
---

# C# 程序的通用结构

C# 程序由一个或多个文件组成。 每个文件均包含零个或多个命名空间。 一个命名空间包含类、结构、接口、枚举、委托等类型或其他命名空间。 以下示例是包含所有这些元素的 C# 程序主干。

```c#
using System;


Console.WriteLine("Hello world!");

namespace YourNamespace
{
    class YourClass
    {
    }

    struct YourStruct
    {
    }

    interface IYourInterface
    {
    }

    delegate int YourDelegate();

    enum YourEnum
    {
    }

    namespace YourNestedNamespace
    {
        struct YourStruct
        {
        }
    }
}
```

前面的示例使用顶级语句作为程序的入口点。 C# 9 中添加了此功能。 在 C# 9 之前，入口点是名为 `Main` 的静态方法，如以下示例所示：

```c#
using System;
namespace YourNamespace
{
    class YourClass
    {
    }

    struct YourStruct
    {
    }

    interface IYourInterface
    {
    }

    delegate int YourDelegate();

    enum YourEnum
    {
    }

    namespace YourNestedNamespace
    {
        struct YourStruct
        {
        }
    }

    class Program
    {
        static void Main(string[] args)
        {
            //Your program starts here...
            Console.WriteLine("Hello world!");
        }
    }
}
```



# C# 类型系统

C# 是一种强类型语言。 **每个变量和常量都有一个类型**，每个求值的表达式也是如此。 每个方法声明都为每个输入参数和返回值指定名称、类型和种类（值、引用或输出）。 .NET 类库定义了内置数值类型和表示各种构造的复杂类型。 其中包括文件系统、网络连接、对象的集合和数组以及日期。 典型的 C# 程序使用类库中的类型，以及对程序问题域的专属概念进行建模的用户定义类型。

类型中可存储的信息包括以下项：

* 类型变量所需的存储空间。
* 可以表示的最大值和最小值。
* 包含的成员（方法、字段、事件等）。
* 继承自的基类型。
* 它实现的接口。
* 允许执行的运算种类

编译器使用类型信息来确保在代码中执行的所有操作都是类型安全的。 例如，如果声明 [`int`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/integral-numeric-types) 类型的变量，那么编译器允许在加法和减法运算中使用此变量。 如果尝试对 [`bool`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/bool) 类型的变量执行这些相同操作，则编译器将生成错误，如以下示例所示：

```c#
int a = 5;
int b = a + 2; //OK
bool test = true;
int c = a + test; // Error
```

> 注意:C 和 C++ 开发人员请注意，在 C# 中，`bool` 不能转换为 `int`。

编译器将类型信息作为元数据嵌入可执行文件中。 公共语言运行时 (CLR) 在运行时使用元数据，以在分配和回收内存时进一步保证类型安全性。

## 内置类型

C# 提供了一组标准的内置类型。 这些类型表示整数、浮点值、布尔表达式、文本字符、十进制值和其他数据类型。 还有内置的 `string` 和 `object` 类型。 这些类型可供在任何 C# 程序中使用。 有关内置类型的完整列表

下表列出了 C# 内置值类型:

|                        C# 类型关键字                         |                          .NET 类型                           |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
| [`bool`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/bool) | [ System.Boolean](https://docs.microsoft.com/zh-cn/dotnet/api/system.boolean) |
| [`byte`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/integral-numeric-types) | [System.Byte](https://docs.microsoft.com/zh-cn/dotnet/api/system.byte) |
| [`sbyte`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/integral-numeric-types) | [System.SByte](https://docs.microsoft.com/zh-cn/dotnet/api/system.sbyte) |
| [`char`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/char) | [System.Char](https://docs.microsoft.com/zh-cn/dotnet/api/system.char) |
| [`decimal`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/floating-point-numeric-types) | [System.Decimal](https://docs.microsoft.com/zh-cn/dotnet/api/system.decimal) |
| [`double`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/floating-point-numeric-types) | [System.Double](https://docs.microsoft.com/zh-cn/dotnet/api/system.double) |
| [`float`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/floating-point-numeric-types) | [System.Single](https://docs.microsoft.com/zh-cn/dotnet/api/system.single) |
| [`int`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/integral-numeric-types) | [System.Int32](https://docs.microsoft.com/zh-cn/dotnet/api/system.int32) |
| [`uint`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/integral-numeric-types) | [System.UInt32](https://docs.microsoft.com/zh-cn/dotnet/api/system.uint32) |
| [`nint`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/integral-numeric-types) | [System.IntPtr](https://docs.microsoft.com/zh-cn/dotnet/api/system.intptr) |
| [`nuint`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/integral-numeric-types) | [System.UIntPtr](https://docs.microsoft.com/zh-cn/dotnet/api/system.uintptr) |
| [`long`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/integral-numeric-types) | [System.Int64](https://docs.microsoft.com/zh-cn/dotnet/api/system.int64) |
| [`ulong`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/integral-numeric-types) | [System.UInt64](https://docs.microsoft.com/zh-cn/dotnet/api/system.uint64) |
| [`short`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/integral-numeric-types) | [System.Int16](https://docs.microsoft.com/zh-cn/dotnet/api/system.int16) |
| [`ushort`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/integral-numeric-types) | [System.UInt16](https://docs.microsoft.com/zh-cn/dotnet/api/system.uint16) |

下表列出了 C# 内置[引用](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/keywords/reference-types)类型：

|                        C# 类型关键字                         |                          .NET 类型                           |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
| [`object`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/reference-types#the-object-type) | [System.Object](https://docs.microsoft.com/zh-cn/dotnet/api/system.object) |
| [`string`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/reference-types#the-string-type) | [System.String](https://docs.microsoft.com/zh-cn/dotnet/api/system.string) |
| [`dynamic`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/reference-types#the-dynamic-type) | [System.Object](https://docs.microsoft.com/zh-cn/dotnet/api/system.object) |

在上表中，左侧列中的每个 C# 类型关键字（[dynamic](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/reference-types#the-dynamic-type) 除外）都是相应 .NET 类型的别名。 它们是可互换的。 例如，以下声明声明了相同类型的变量：

```c#
int a = 123;
System.Int32 b = 123;
```

## 自定义类型

可以使用 [`struct`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/struct)、[`class`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/keywords/class)、[`interface`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/keywords/interface)、[`enum`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/enum) 和 [`record`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/record) 构造来创建自己的自定义类型。 .NET 类库本身是一组自定义类型，以供你在自己的应用程序中使用。 默认情况下，类库中最常用的类型在任何 C# 程序中均可用。 其他类型只有在显式添加对定义这些类型的程序集的项目引用时才可用。 编译器引用程序集之后，你可以声明在源代码的此程序集中声明的类型的变量（和常量）。 

## 通用类型系统

对于 .NET 中的类型系统，请务必了解以下两个基本要点：

- 它支持继承原则。 类型可以派生自其他类型（称为*基类型*）。 派生类型继承（有一些限制）基类型的方法、属性和其他成员。 基类型可以继而从某种其他类型派生，在这种情况下，派生类型继承其继承层次结构中的两种基类型的成员。 所有类型（包括 System.Int32 (C# keyword: `int`) 等内置数值类型）最终都派生自单个基类型，即 System.Object(C# keyword: [`object`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/reference-types))。 这样的统一类型层次结构称为[通用类型系统](https://docs.microsoft.com/zh-cn/dotnet/standard/base-types/common-type-system) (CTS)。 若要详细了解 C# 中的继承，请参阅[继承](https://docs.microsoft.com/zh-cn/dotnet/csharp/fundamentals/object-oriented/inheritance)。
- CTS 中的每种类型被定义为值类型或引用类型。 这些类型包括 .NET 类库中的所有自定义类型以及你自己的用户定义类型。 使用 `struct` 关键字定义的类型是值类型；所有内置数值类型都是 `structs`。 使用 `class` 或 `record` 关键字定义的类型是引用类型。 引用类型和值类型遵循不同的编译时规则和运行时行为。

下图展示了 CTS 中值类型和引用类型之间的关系。

![image-20220808220437295](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220808220437295.png)

类和结构是 .NET 通用类型系统的两种基本构造。 C# 9 添加记录，记录是一种类。 每种本质上都是一种数据结构，其中封装了同属一个逻辑单元的一组数据和行为。 数据和行为是类、结构或记录的成员。 这些行为包括方法、属性和事件等，本文稍后将具体列举。

类、结构或记录声明类似于一张蓝图，用于在运行时创建实例或对象。 如果定义名为 `Person` 的类、结构或记录，则 `Person` 是类型的名称。 如果声明和初始化 `Person` 类型的变量 `p`，那么 `p` 就是所谓的 `Person` 对象或实例。 可以创建同一 `Person` 类型的多个实例，每个实例都可以有不同的属性和字段值。

类是引用类型。 创建类型的对象后，向其分配对象的变量仅保留对相应内存的引用。 将对象引用分配给新变量后，新变量会引用原始对象。 通过一个变量所做的更改将反映在另一个变量中，因为它们引用相同的数据。

结构是值类型。 创建结构时，向其分配结构的变量保留结构的实际数据。 将结构分配给新变量时，会复制结构。 因此，新变量和原始变量包含相同数据的副本（共两个）。 对一个副本所做的更改不会影响另一个副本。

记录类型可以是引用类型 (`record class`) 或值类型 (`record struct`)。

一般来说，类用于对更复杂的行为建模。 类通常存储计划在创建类对象后进行修改的数据。 结构最适用于小型数据结构。 结构通常存储不打算在创建结构后修改的数据。 记录类型是具有附加编译器合成成员的数据结构。 记录通常存储不打算在创建对象后修改的数据。

### .NET 中的类型

NET 中的所有类型不是值类型就是引用类型。
值类型是使用对象实际值来表示对象的数据类型。 如果向一个变量分配值类型的实例，则该变量将被赋以该值的全新副本。
引用类型是使用对对象实际值的引用（类似于指针）来表示对象的数据类型。 如果为某个变量分配一个引用类型，则该变量将引用（或指向）原始值。 不创建任何副本。
.NET 中的通用类型系统支持以下五种类别的类型：

* [类](https://docs.microsoft.com/zh-cn/dotnet/standard/base-types/common-type-system#classes)
* [结构](https://docs.microsoft.com/zh-cn/dotnet/standard/base-types/common-type-system#structures)
* [枚举](https://docs.microsoft.com/zh-cn/dotnet/standard/base-types/common-type-system#enumerations)
* [接口](https://docs.microsoft.com/zh-cn/dotnet/standard/base-types/common-type-system#interfaces)
* [委托](https://docs.microsoft.com/zh-cn/dotnet/standard/base-types/common-type-system#delegates)

### 类型成员的特征

通用类型系统允许类型成员具有多种特征，但并不要求语言能支持所有这些特征。 下表介绍了这些成员特征。

| 特征                                                         | 可应用到               | 描述                                                         |
| :----------------------------------------------------------- | :--------------------- | :----------------------------------------------------------- |
| abstract                                                     | 方法、属性和事件       | 类型不提供方法的实现。 继承或实现抽象方法的类型必须提供方法的实现。 只有当派生的类型本身是抽象类型的时候，情况例外。 所有的抽象方法都是虚的。 |
| private、family、assembly、family 和 assembly、family 或 assembly，或者 public | 全部                   | 定义成员的可访问性：  private 只能在与成员相同的类型或在嵌套类型中访问。  family 在与成员相同的类型中和从它继承的派生类型访问。  程序集 只能在定义该类型的程序集中访问。  family 和 assembly 只能从同时具备族和程序集访问权的类型进行访问。  family 或 assembly 只能从具备族和程序集访问权的类型进行访问。  public 可从任何类型访问。 |
| final                                                        | 方法、属性和事件       | 虚方法不能在派生类型中被重写。                               |
| initialize-only                                              | 字段                   | 该值只能被初始化，不能在初始化之后写入。                     |
| 实例                                                         | 字段、方法、属性和事件 | 如果成员未标记为 `static`（C# 和 C++）、`Shared` (Visual Basic)、`virtual`（C# 和 C++）或 `Overridable` (Visual Basic)，则它是一个实例成员（没有实例关键字）。 内存中这些成员的副本数将会像使用它们的对象数一样多。 |
| 文本                                                         | 字段                   | 分配给该字段的值是一个内置值类型的固定值（在编译时已知）。 文本字段有时指的是常数。 |
| newslot 或 override                                          | 全部                   | 定义成员如何与具有相同签名的继承成员进行交互：  newslot 隐藏具有相同签名的继承成员。  override 替换继承的虚方法的定义。  默认为 newslot。 |
| 静态                                                         | 字段、方法、属性和事件 | 成员属于定义它的类型，而不属于该类型的特定实例；即使不创建类型的实例，成员也会存在，并且它由该类型的所有实例共享。 |
| virtual                                                      | 方法、属性和事件       | 此方法可以由派生类型实现，并且既可静态调用，也可动态调用。 如果使用动态调用，在运行时执行调用的实例类型（而不是编译时已知的类型）将确定调用方法的哪一种实现。 若要静态调用虚方法，可能必须将变量强制转换为使用所需方法版本的类型。 |

### 重载

每个类型成员都有一个唯一的签名。 方法签名由方法名称和一个参数列表（方法的参数的顺序和类型）组成。 可以在一种类型内定义具有相同名称的多种方法，只要这些方法的签名不同。 当定义两种或多种具有相同名称的方法时，就称作重载。 例如，在 System.Char中，重载了 IsDigit方法。 一个方法采用 Char。 另一个方法采用 String和 Int32。

### 继承、重写和隐藏成员

派生类型继承其基类型的所有成员；也就是说，会在派生类型上定义这些成员，并供派生类型使用。 继承成员的行为和质量可以通过以下两种方式来修改：

- 派生类型可通过使用相同的签名定义一个新成员，从而隐藏继承的成员。 将先前的公共成员变成私有成员，或者为标记为 `final` 的继承方法定义新行为时，可以采取这种方法。
- 派生类型可以重写继承的虚方法。 重写方法提供了对方法的一种新定义，将根据运行时的值的类型，而不是编译时已知的变量类型来调用方法。 只有在虚拟方法未标记为 `final` 且新方法至少可以像虚拟方法一样进行访问的情况下，方法才能重写虚拟方法。

# 成员

类型的成员包括所有方法、字段、常量、属性和事件。 C# 没有全局变量或方法，这一点其他某些语言不同。 即使是编程的入口点（`Main` 方法），也必须在类或结构中声明（使用顶级语句时，隐式声明）。

下面列出了所有可以在类、结构或记录中声明的各种成员。

- 字段
- 常量
- 属性
- 方法
- 构造函数
- 事件
- 终结器
- 索引器
- 运算符
- 嵌套类型

| 成员                                                         | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [字段](https://docs.microsoft.com/zh-cn/dotnet/csharp/programming-guide/classes-and-structs/fields) | 字段是在类范围声明的变量。 字段可以是内置数值类型或其他类的实例。 例如，日历类可能具有一个包含当前日期的字段。 |
| [常量](https://docs.microsoft.com/zh-cn/dotnet/csharp/programming-guide/classes-and-structs/constants) | 常量是在编译时设置其值并且不能更改其值的字段。               |
| [属性](https://docs.microsoft.com/zh-cn/dotnet/csharp/programming-guide/classes-and-structs/properties) | 属性是类中可以像类中的字段一样访问的方法。 属性可以为类字段提供保护，以避免字段在对象不知道的情况下被更改。 |
| [方法](https://docs.microsoft.com/zh-cn/dotnet/csharp/programming-guide/classes-and-structs/methods) | 方法定义类可以执行的操作。 方法可接受提供输入数据的参数，并可通过参数返回输出数据。 方法还可以不使用参数而直接返回值。 |
| [事件](https://docs.microsoft.com/zh-cn/dotnet/csharp/programming-guide/events/) | 事件向其他对象提供有关发生的事情（如单击按钮或成功完成某个方法）的通知。 事件是使用委托定义和触发的。 |
| [运算符](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/) | 重载运算符被视为类型成员。 重载运算符时，将其定义为类型中的公共静态方法。 有关详细信息，请参阅[运算符重载](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/operator-overloading)。 |
| [索引器](https://docs.microsoft.com/zh-cn/dotnet/csharp/programming-guide/indexers/) | 使用索引器可以用类似于数组的方式为对象建立索引。             |
| [构造函数](https://docs.microsoft.com/zh-cn/dotnet/csharp/programming-guide/classes-and-structs/constructors) | 构造函数是首次创建对象时调用的方法。 它们通常用于初始化对象的数据。 |
| [终结器](https://docs.microsoft.com/zh-cn/dotnet/csharp/programming-guide/classes-and-structs/finalizers) | C# 中很少使用终结器。 终结器是当对象即将从内存中移除时由运行时执行引擎调用的方法。 它们通常用来确保任何必须释放的资源都得到适当的处理。 |
| [嵌套类型](https://docs.microsoft.com/zh-cn/dotnet/csharp/programming-guide/classes-and-structs/nested-types) | 嵌套类型是在其他类型中声明的类型。 嵌套类型通常用于描述仅由包含它们的类型使用的对象。 |

# 关键字

关键字是预定义的保留标识符，对编译器有特殊意义。 除非前面有 `@` 前缀，否则不能在程序中用作标识符。 例如，`@if` 是有效标识符，而 `if` 则不是，因为 `if` 是关键字。

此主题中的第一个表列出了是 C# 程序任意部分中的保留标识符的关键字。 此主题中的第二个表列出了 C# 中的上下文关键字。 上下文关键字仅在一部分程序上下文中有特殊含义，可以在相应上下文范围之外用作标识符。 一般来说，C# 语言中新增的关键字会作为上下文关键字添加，以免破坏用旧版语言编写的程序。

# C# 运算符和表达式

C# 提供了许多运算符。 其中许多都受到[内置类型](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/built-in-types)的支持，可用于对这些类型的值执行基本操作。 这些运算符包括以下组：

- [算术运算符](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/arithmetic-operators)，将对数值操作数执行算术运算
- [比较运算符](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/comparison-operators)，将比较数值操作数
- [布尔逻辑运算符](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/boolean-logical-operators)，将对 [`bool`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/bool) 操作数执行逻辑运算
- [位运算符和移位运算符](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/bitwise-and-shift-operators)，将对整数类型的操作数执行位运算或移位运算
- [相等运算符](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/equality-operators)，将检查其操作数是否相等

通常可以[重载](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/operator-overloading)这些运算符，也就是说，可以为用户定义类型的操作数指定运算符行为。

## 运算符重载：

使用 `operator` 关键字来声明运算符。 运算符声明必须符合以下规则：

- 同时包含 `public` 和 `static` 修饰符。
- 一元运算符有一个输入参数。 二元运算符有两个输入参数。 在每种情况下，都至少有一个参数必须具有类型 `T` 或 `T?`，其中 `T` 是包含运算符声明的类型。

下面的示例定义了一个表示有理数的简单结构。 该结构会重载一些算术运算符：

```c#
public readonly struct Fraction
{
    private readonly int num;
    private readonly int den;

    public Fraction(int numerator, int denominator)
    {
        if (denominator == 0)
        {
            throw new ArgumentException("Denominator cannot be zero.", nameof(denominator));
        }
        num = numerator;
        den = denominator;
    }

    public static Fraction operator +(Fraction a) => a;
    public static Fraction operator -(Fraction a) => new Fraction(-a.num, a.den);

    public static Fraction operator +(Fraction a, Fraction b)
        => new Fraction(a.num * b.den + b.num * a.den, a.den * b.den);

    public static Fraction operator -(Fraction a, Fraction b)
        => a + (-b);

    public static Fraction operator *(Fraction a, Fraction b)
        => new Fraction(a.num * b.num, a.den * b.den);

    public static Fraction operator /(Fraction a, Fraction b)
    {
        if (b.num == 0)
        {
            throw new DivideByZeroException();
        }
        return new Fraction(a.num * b.den, a.den * b.num);
    }

    public override string ToString() => $"{num} / {den}";
}

public static class OperatorOverloading
{
    public static void Main()
    {
        var a = new Fraction(5, 4);
        var b = new Fraction(1, 2);
        Console.WriteLine(-a);   // output: -5 / 4
        Console.WriteLine(a + b);  // output: 14 / 8
        Console.WriteLine(a - b);  // output: 6 / 8
        Console.WriteLine(a * b);  // output: 5 / 8
        Console.WriteLine(a / b);  // output: 10 / 4
    }
}
```

### 用户定义转换运算符

用户定义类型可以定义从或到另一个类型的自定义隐式或显式转换。

隐式转换无需调用特殊语法，并且可以在各种情况（例如，在赋值和方法调用中）下发生。 预定义的 C# 隐式转换始终成功，且永远不会引发异常。 用户定义隐式转换也应如此。 如果自定义转换可能会引发异常或丢失信息，请将其定义为显式转换。

is和 as运算符不考虑使用用户定义转换。 强制转换表达式用于调用用户定义显式转换。

`operator` 和 `implicit` 或 `explicit` 关键字分别用于定义隐式转换或显式转换。 定义转换的类型必须是该转换的源类型或目标类型。 可用两种类型中的任何一种类型来定义两种用户定义类型之间的转换。

下面的示例展示如何定义隐式转换和显式转换：

```c#
using System;

public readonly struct Digit
{
    private readonly byte digit;

    public Digit(byte digit)
    {
        if (digit > 9)
        {
            throw new ArgumentOutOfRangeException(nameof(digit), "Digit cannot be greater than nine.");
        }
        this.digit = digit;
    }

    public static implicit operator byte(Digit d) => d.digit;
    public static explicit operator Digit(byte b) => new Digit(b);

    public override string ToString() => $"{digit}";
}

public static class UserDefinedConversions
{
    public static void Main()
    {
        var d = new Digit(7);

        byte number = d;
        Console.WriteLine(number);  // output: 7

        Digit digit = (Digit)number;
        Console.WriteLine(digit);  // output: 7
    }
}
```

### 可重载运算符

下表提供了 C# 运算符可重载性的相关信息：

| 运算符                                                       | 可重载性                                                     |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [+x](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/arithmetic-operators#unary-plus-and-minus-operators), [-x](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/arithmetic-operators#unary-plus-and-minus-operators), [!x](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/boolean-logical-operators#logical-negation-operator-), [~x](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/bitwise-and-shift-operators#bitwise-complement-operator-)、[++](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/arithmetic-operators#increment-operator-)、[--](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/arithmetic-operators#decrement-operator---)、[true](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/true-false-operators)、[false](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/true-false-operators) | 这些一元运算符可以进行重载。                                 |
| [x + y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/addition-operator)、[x - y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/subtraction-operator)、[x * y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/arithmetic-operators#multiplication-operator-)、[x / y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/arithmetic-operators#division-operator-)、[x % y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/arithmetic-operators#remainder-operator-)、[x & y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/boolean-logical-operators#logical-and-operator-)、[x \| y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/boolean-logical-operators#logical-or-operator-)、[x ^ y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/boolean-logical-operators#logical-exclusive-or-operator-)、[x << y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/bitwise-and-shift-operators#left-shift-operator-)、[x >> y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/bitwise-and-shift-operators#right-shift-operator-)、[x == y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/equality-operators#equality-operator-)、[x != y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/equality-operators#inequality-operator-)、[x < y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/comparison-operators#less-than-operator-)、[x > y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/comparison-operators#greater-than-operator-)、[x <= y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/comparison-operators#less-than-or-equal-operator-)、[x >= y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/comparison-operators#greater-than-or-equal-operator-) | 这些二元运算符可以进行重载。 某些运算符必须成对重载；有关详细信息，请查看此表后面的备注。 |
| [x && y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/boolean-logical-operators#conditional-logical-and-operator-)、[x \|\| y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/boolean-logical-operators#conditional-logical-or-operator-) | 无法重载条件逻辑运算符。 但是，如果具有已重载的 [`true` 和 `false` 运算符](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/true-false-operators)的类型还以某种方式重载了 `&` 或 `|` 运算符，则可针对此类型的操作数分别计算 `&&` 或 `||` 运算符。 有关详细信息，请参阅 [C# 语言规范](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/language-specification/readme)的[用户定义条件逻辑运算符](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/language-specification/expressions#11133-user-defined-conditional-logical-operators)部分。 |
| [a[i\]](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/member-access-operators#indexer-operator-)、[`a?[i\]`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/member-access-operators#null-conditional-operators--and-) | 元素访问不被视为可重载运算符，但你可定义[索引器](https://docs.microsoft.com/zh-cn/dotnet/csharp/programming-guide/indexers/)。 |
| [(T)x](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/type-testing-and-cast#cast-expression) | 强制转换运算符不能重载，但可以定义可由强制转换表达式执行的自定义类型转换。 有关详细信息，请参阅[用户定义转换运算符](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/user-defined-conversion-operators)。 |
| [+=](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/arithmetic-operators#compound-assignment)、[-=](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/arithmetic-operators#compound-assignment)、[*=](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/arithmetic-operators#compound-assignment)、[/=](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/arithmetic-operators#compound-assignment)、[%=](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/arithmetic-operators#compound-assignment)、[&=](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/boolean-logical-operators#compound-assignment)、[\|=](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/boolean-logical-operators#compound-assignment)、[^=](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/boolean-logical-operators#compound-assignment)、[<<=](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/bitwise-and-shift-operators#compound-assignment)、[>>=](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/bitwise-and-shift-operators#compound-assignment) | 复合赋值运算符不能显式重载。 但在重载二元运算符时，也会隐式重载相应的复合赋值运算符（若有）。 例如，使用 `+`（可以进行重载）计算 `+=`。 |
| [^x](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/member-access-operators#index-from-end-operator-)、[x = y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/assignment-operator)、[x.y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/member-access-operators#member-access-expression-)、[`x?.y`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/member-access-operators#null-conditional-operators--and-)、[c ? t : f](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/conditional-operator)、[x ?? y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/null-coalescing-operator)、[x ??= y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/null-coalescing-operator)、[x..y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/member-access-operators#range-operator-)、[x->y](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/pointer-related-operators#pointer-member-access-operator--)、[=>](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/lambda-operator)、[f(x)](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/member-access-operators#invocation-expression-)、[as](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/type-testing-and-cast#as-operator)、[await](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/await)、[checked](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/keywords/checked)、[unchecked](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/keywords/unchecked)、[default](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/default)、[delegate](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/delegate-operator)、[is](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/type-testing-and-cast#is-operator)、[nameof](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/nameof)、[new](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/new-operator)、[sizeof](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/sizeof)、[stackalloc](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/stackalloc)、[switch](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/switch-expression)、[typeof](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/type-testing-and-cast#typeof-operator)、[with](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/with-expression) | 这些运算符无法进行重载。                                     |



## 相等运算符 ==

### 值类型的相等性

如果内置值类型的值相等，则其操作数相等：

```c#
int a = 1 + 2 + 3;
int b = 6;
Console.WriteLine(a == b);  // output: True

char c1 = 'a';
char c2 = 'A';
Console.WriteLine(c1 == c2);  // output: False
Console.WriteLine(c1 == char.ToLower(c2));  // output: True
```

> 对于 `==`、[`<`、`>`、`<=` 和 `>=`](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/comparison-operators) 运算符，如果任何操作数不是数字（[Double.NaN](https://docs.microsoft.com/zh-cn/dotnet/api/system.double.nan) 或 [Single.NaN](https://docs.microsoft.com/zh-cn/dotnet/api/system.single.nan)），则运算的结果为 `false`。 这意味着 `NaN` 值不大于、小于或等于任何其他 `double`（或 `float`）值，包括 `NaN`

如果基本整数类型的相应值相等，则相同枚举类型的两个操作数相等。

用户定义的 struct类型默认情况下不支持 `==` 运算符。 要支持 `==` 运算符，用户定义的结构必须重载它。

#### 元组相等

从 C# 7.3 开始，元组类型支持 `==` 和 `!=` 运算符。 这些运算符按照元组元素的顺序将左侧操作数的成员与相应的右侧操作数的成员进行比较。

```c#
(int a, byte b) left = (5, 10);
(long a, int b) right = (5, 10);
Console.WriteLine(left == right);  // output: True
Console.WriteLine(left != right);  // output: False

var t1 = (A: 5, B: 10);
var t2 = (B: 5, A: 10);
Console.WriteLine(t1 == t2);  // output: True
Console.WriteLine(t1 != t2);  // output: False
```

如前面的示例所示，`==` 和 `!=` 操作不会考虑元组字段名称。

同时满足以下两个条件时，两个元组可比较：

- 两个元组具有相同数量的元素。 例如，如果 `t1` 和 `t2` 具有不同数目的元素，`t1 != t2` 则不会进行编译。
- 对于每个元组位置，可以使用 `==` 和 `!=` 运算符对左右侧元组操作数中的相应元素进行比较。 例如，`(1, (2, 3)) == ((1, 2), 3)` 不会进行编译，因为 `1` 不可与 `(1, 2)` 比较。

`==` 和 `!=` 运算符将以短路方式对元组进行比较。 也就是说，一旦遇见一对不相等的元素或达到元组的末尾，操作将立即停止。 但是，在进行任何比较之前，将对所有元组元素进行计算，如以下示例所示：

```c#
Console.WriteLine((Display(1), Display(2)) == (Display(3), Display(4)));

int Display(int s)
{
    Console.WriteLine(s);
    return s;
}
// Output:
// 1
// 2
// 3
// 4
// False
```

### 引用类型的相等性

默认情况下，如果两个非记录引用类型操作符引用同一对象，则这两个操作符相等：

```c#
public class ReferenceTypesEquality
{
    public class MyClass
    {
        private int id;

        public MyClass(int id) => this.id = id;
    }

    public static void Main()
    {
        var a = new MyClass(1);
        var b = new MyClass(1);
        var c = a;
        Console.WriteLine(a == b);  // output: False
        Console.WriteLine(a == c);  // output: True
    }
}
```

如示例所示，默认情况下，用户定义的引用类型支持 `==` 运算符。 但是，引用类型可重载 `==` 运算符。 如果引用类型重载 `==` 运算符，使用 Object.ReferenceEquals= 方法来检查该类型的两个引用是否引用同一对象。

### 记录类型相等性

在 C# 9.0 和更高版本中提供，记录类型(引用类型)支持 `==` 和 `!=` 运算符，这些运算符默认提供值相等性语义。 也就是说，当两个记录操作数均为 `null` 或所有字段的对应值和自动实现的属性相等时，两个记录操作数都相等。

```c#
public class RecordTypesEquality
{
    public record Point(int X, int Y, string Name);
    public record TaggedNumber(int Number, List<string> Tags);

    public static void Main()
    {
        var p1 = new Point(2, 3, "A");
        var p2 = new Point(1, 3, "B");
        var p3 = new Point(2, 3, "A");

        Console.WriteLine(p1 == p2);  // output: False
        Console.WriteLine(p1 == p3);  // output: True

        var n1 = new TaggedNumber(2, new List<string>() { "A" });
        var n2 = new TaggedNumber(2, new List<string>() { "A" });
        Console.WriteLine(n1 == n2);  // output: False
    }
}
```

如前面的示例所示，对于非记录引用类型成员，比较的是它们的引用值，而不是所引用的实例。

### 字符串相等性

如果两个字符串均为 `null` 或者两个字符串实例具有相等长度且在每个字符位置有相同字符，则这两个[字符串](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/reference-types#the-string-type)操作数相等：

```c#
string s1 = "hello!";
string s2 = "HeLLo!";
Console.WriteLine(s1 == s2.ToLower());  // output: True

string s3 = "Hello!";
Console.WriteLine(s1 == s3);  // output: False
```

这就是区分大小写的序号比较。 有关字符串比较的详细信息，请参阅[如何在 C# 中比较字符串](https://docs.microsoft.com/zh-cn/dotnet/csharp/how-to/compare-strings)。

### 委托相等

若两个运行时间类型相同的[委托](https://docs.microsoft.com/zh-cn/dotnet/csharp/programming-guide/delegates/)操作数均为 `null`，或其调用列表长度系统且在每个位置具有相同的条目，则二者相等：

```c#
Action a = () => Console.WriteLine("a");

Action b = a + a;
Action c = a + a;
Console.WriteLine(object.ReferenceEquals(b, c));  // output: False
Console.WriteLine(b == c);  // output: True
```

有关详细信息，请参阅 [C# 语言规范](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/language-specification/readme)中的[委托相等运算符](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/language-specification/expressions#11119-delegate-equality-operators)部分。

通过计算语义上相同的 [Lambda 表达式](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/lambda-expressions)生成的委托不相等，如以下示例所示：

```c#
Action a = () => Console.WriteLine("a");
Action b = () => Console.WriteLine("a");

Console.WriteLine(a == b);  // output: False
Console.WriteLine(a + b == a + b);  // output: True
Console.WriteLine(b + a == a + b);  // output: False
```

### 不等运算符 !=

如果操作数不相等，不等于运算符 `!=` 返回 `true`，否则返回 `false`。 对于[内置类型](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/built-in-types)的操作数，表达式 `x != y` 生成与表达式 `!(x == y)` 相同的结果。 有关类型相等性的更多信息，请参阅[相等运算符](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/equality-operators#equality-operator-)部分。

下面的示例演示 `!=` 运算符的用法：

```c#
int a = 1 + 1 + 2 + 3;
int b = 6;
Console.WriteLine(a != b);  // output: True

string s1 = "Hello";
string s2 = "Hello";
Console.WriteLine(s1 != s2);  // output: False

object o1 = 1;
object o2 = 1;
Console.WriteLine(o1 != o2);  // output: True
```

### 运算符可重载性

用户定义类型可以[重载](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/operator-overloading)`==` 和 `!=` 运算符。 如果某类型重载这两个运算符之一，它还必须重载另一个运算符。

记录类型不能显式重载 `==` 和 `!=` 运算符。 如果需要更改记录类型 `T` 的 `==` 和 `!=` 运算符的行为，请使用以下签名实现 [IEquatable.Equals](https://docs.microsoft.com/zh-cn/dotnet/api/system.iequatable-1.equals) 方法：

```c#
public virtual bool Equals(T? other);
```

### ？？和？？=运算符

如果左操作数的值不为 `null`，则 null 合并运算符 `??` 返回该值；否则，它会计算右操作数并返回其结果。 如果左操作数的计算结果为非 null，则 `??` 运算符不会计算其右操作数。

C# 8.0 及更高版本中可使用空合并赋值运算符 `??=`，该运算符仅在左侧操作数的求值结果为 `null` 时，才将其右侧操作数的值赋值给左操作数。 如果左操作数的计算结果为非 null，则 `??=` 运算符不会计算其右操作数。

```c#
List<int> numbers = null;
int? a = null;

//当numbers为null时把右操作数赋值给numbers
(numbers ??= new List<int>()).Add(5);
Console.WriteLine(string.Join(" ", numbers));  // output: 5

numbers.Add(a ??= 0);
Console.WriteLine(string.Join(" ", numbers));  // output: 5 0
Console.WriteLine(a);  // output: 0
```

`??=` 运算符的左操作数必须是变量、[属性](https://docs.microsoft.com/zh-cn/dotnet/csharp/programming-guide/classes-and-structs/properties)或[索引器](https://docs.microsoft.com/zh-cn/dotnet/csharp/programming-guide/indexers/)元素。

在 C# 7.3 及更早版本中，`??` 运算符左操作数的类型必须是[引用类型](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/keywords/reference-types)或[可以为 null 的值类型](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/nullable-value-types)。 从 C# 8.0 版本开始，该要求替换为以下内容：`??` 和 `??=` 运算符的左操作数的类型必须是可以为 null 的值类型。 特别是从 C# 8.0 开始，可以使用具有无约束类型参数的 null 合并运算符：

```c#
private static void Display<T>(T a, T backup)
{
    Console.WriteLine(a ?? backup);
}
```

null 合并运算符是右结合运算符。 也就是说，是窗体的表达式

```c#
a ?? b ?? c
d ??= e ??= f
```

会像这样求值

```c#
a ?? (b ?? c)
d ??= (e ??= f)
```

### :: 运算符

使用命名空间别名限定符 `::` 访问已设置别名的命名空间的成员。 只能使用两个标识符之间的 `::` 限定符。 左侧标识符可以是以下任意别名：

使用 [using 别名指令](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/keywords/using-directive)创建的命名空间别名：

```c#
using forwinforms = System.Drawing;
using forwpf = System.Windows;

public class Converters
{
    public static forwpf::Point Convert(forwinforms::Point point) => new forwpf::Point(point.X, point.Y);
}
```

- [外部别名](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/keywords/extern-alias)。

- `global` 别名，该别名是全局命名空间别名。 全局命名空间是包含未在命名空间中声明的命名空间和类型的命名空间。 与 `::` 限定符一起使用时，`global` 别名始终引用全局命名空间，即使存在用户定义的 `global` 命名空间别名也是如此。

  以下示例使用 `global` 别名访问 .NET [System](https://docs.microsoft.com/zh-cn/dotnet/api/system) 命名空间，该命名空间是全局命名空间的成员。 如果没有 `global` 别名，则将访问用户定义的 `System` 命名空间（该命名空间是 `MyCompany.MyProduct` 命名空间的成员）：\

System 放在了 MyCompany.MyProduct.System 的非最开头的其余地方，作为子命名空间名称出现。在这种情况下，你完全无法使用任何 System 命名空间下的 .NET 自带 API（即库 API）。

那么，你非得用的话，C# 1 是做不到的。在 C# 2 里，为了解决这一点，C# 新创建了一个关键字 global，用在命名空间的最开头，表示你使用的是，紧挨着 global 关键字后的这个 System，是整个你表达的命名空间的开头。啥意思呢？比如这样的代码：

```c#
namespace MyCompany.MyProduct.System
{
    class Program
    {
        static void Main() => global::System.Console.WriteLine("Using global alias");
    }

    class Console
    {
        string Suggestion => "Consider renaming this class";
    }
}
```

仅当 `global` 关键字是 `::` 限定符的左侧标识符时，该关键字才是全局命名空间别名。

# 集合：

## Cast<T>()和OfType<T>()使用:

```c#
static void CastAndOfType()
        {
            ArrayList fruits = new ArrayList(4);
            fruits.Add("Apple");
            fruits.Add("Pear");
            fruits.Add(2);
            fruits.Add(5);
 
            //下面这句会报错，因为ArrayList没有实现IEnumerable<T>接口，故无法使用标准查询运算
            //IEnumerable<string> s=fruitShop.Select(str => str);
            //但是使用Cast<T>和OfType<T>来进行转换
 
            IEnumerable<string> s1 = fruits.Cast<string>();
            IEnumerable<string> s2 = fruits.OfType<string>();
            //虽然Cast<T>和OfType<T>都可以用来进行转换，但是OfType<T>比Cast<T>更加强大，
            //它可以对结果进行筛选，而Cast<T>遇到无法强制转换的则会报错
 
            try
            {
                foreach (string fruit in s1)
                {
                    Console.WriteLine(fruit);
                }
            }
            catch(InvalidCastException invalid)
            {
                Console.WriteLine(invalid.Message);
            }
 
            foreach (string fruit in s2)
            {
                Console.WriteLine(fruit);
            }
       }

```

