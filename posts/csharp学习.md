---
title: 'c# 学习'
titleColor: '#aaa,#0ae9ad'
titleIcon: 'asset:markdown'
tags: [ 'c#' ]
categories: [ 'Code' ]
description: '记录c# 的学习'
publishDate: '2023-03-13'
updatedDate: '2023-03-13'
---

#### 运行 c#程序

```csharp
dotnet new console -o ./CsharpProjects/TestProject
dotnet build
dotnet run

Console.WriteLine($"Discount: {a}");
```

#### 数据类型

int，适用于大部分整数  
float，适用于大部分小数
decimal，适用于表示资金的数字  
bool，适用于 true 或 false 值  
string，适用于字母数字值

byte：用于来自其他计算机系统或使用不同字符集的编码数据。  
double：用于几何学或科研用途。 double 常用于生成涉及运动的游戏。  
System.DateTime，适用于特定的日期和时间值。  
System.TimeSpan，适用于年/月/日/小时/分钟/秒/毫秒范围。

#### 空间大小

```csharp
int size = sizeof(int)

float f1 = 3.14f;
double d1 = 3.14;
decimal m1 = 3.14m;

sbyte  : -128 to 127
short  : -32768 to 32767
int    : -2147483648 to 2147483647
long   : -9223372036854775808 to 9223372036854775807
byte   : 0 to 255
ushort : 0 to 65535
uint   : 0 to 4294967295
ulong  : 0 to 18446744073709551615
float  : -3.4028235E+38 to 3.4028235E+38 (with ~6-9 digits of precision)
double : -1.7976931348623157E+308 to 1.7976931348623157E+308 (with ~15-17 digits of precision)
decimal: -79228162514264337593543950335 to 79228162514264337593543950335 (with 28-29 digits of precision)
```

#### 可空类型

```csharp
bool? = null;
```

#### Null 合并运算符

```csharp
string s = null ?? "World";
```

#### 引用类型

c#中内置的引用类型包括 object、string、array、delegate、event、multicast delegate、type、dynamic。  
object 类型是所有类的基类，可以作为所有类的父类。  
string 类型是不可变的字符序列。  
array 类型是一组相同类型元素的集合。  
delegate 类型是指向方法的引用。  
event 类型是通知事件发生的机制。  
multicast delegate 类型是可以包含多个方法的 delegate。  
type 类型是表示已加载程序集中类型或类型的元数据的对象。  
dynamic 类型是表示运行时类型和值之间的双向转换的机制。

#### 方法类型与参数

ref 关键字：用于方法参数传递引用。  
out 关键字：用于方法参数传递输出参数。  
params 关键字：用于方法参数传递可变参数。

#### 数组

```csharp
int[] arr = new int[5];
arr[0] = 1;
arr[1] = 2;
```

```csharp
int[] arr = { 1, 2, 3, 4, 5 };
```

分割字符串

```csharp
string[] arr = "1,2,3,4,5".Split(',');
```

二维数组

```csharp
int[,] arr = new int[2, 2];
arr[0, 0] = 1;
arr[0, 1] = 2;
arr[1, 0] = 3;
arr[1, 1] = 4;

int[,] arr = {
    {1, 2},
    {3, 4}
}

int[,,] arr = new int[2, 2, 2];
```

#### 枚举类型

```csharp
enum Color { Red, Green, Blue }

internal partial class Program
{
    static void Main()
    {
        Color color = Color.Green;
        int val = (int)color;
        Console.WriteLine(val); // Output: 1 (int)
        color = (Color)2;
        Console.WriteLine(color); // Output: Blue (Color)
        string str = Enum.GetName(typeof(Color), val) ?? "Unknown";
        Console.WriteLine(str); // Output: Green (string)
    }
}
```

#### 结构体

结构体是值类型，在赋值时进行复制。  
结构体是值类型，而类是引用类型。  
结构体可以在不使用 new 操作符的情况下实例化。

与 class 的差异：

1. 赋值与修改

```csharp
// struct 示例
public struct Point
{
    public int X;
    public int Y;
}

Point p1 = new Point { X = 1, Y = 2 };
Point p2 = p1; // 复制值
p2.X = 3;      // 修改 p2 不影响 p1
Console.WriteLine(p1.X); // 输出 1

// class 示例
public class Person
{
    public string Name;
}

Person a = new Person { Name = "Alice" };
Person b = a;    // 复制引用
b.Name = "Bob";  // 修改 b 会影响 a
Console.WriteLine(a.Name); // 输出 "Bob"
```

2. ​ 默认构造函数

```csharp
public struct MyStruct
{
    // public MyStruct() { } // 错误：结构体不能显式定义无参构造函数
    public int Value;
}

public class MyClass
{
    public MyClass() { } // 合法
    public int Value;
}
```

3. null 值

```csharp
MyStruct s1 = null; // 编译错误
MyStruct? s2 = null; // 合法（使用 Nullable<T>）

MyClass c1 = null; // 合法
```

总结  
​ 值类型（struct）​：独立、复制快、适合轻量数据。  
​ 引用类型（class）​：共享、灵活、适合复杂对象。

### 面向对象

三大特性：封装、继承、多态。

对象是类的实例

class 默认是 internal  
属性默认是 private

构造函数：没有返回值，名称与类名相同，可以带参数

实例构造函数：

```csharp
public class Person
{
    public Person() {}
}
```

// 静态构造函数：

```csharp
public class Person
{
    static Person() {}
}
```

私有构造函数：

```csharp
public class Person
{
    private Person() {}
}
```

get set 方法：

```csharp
public class Person
{
    private string _name;
    private string _age;
    public string Name
    {
        get { return _name; }
        set { _name = value; }
    }
    public string Age { get; set; }
}

```

#### 继承

```csharp
public class Animal
{
    public void Eat()
    {
        Console.WriteLine("Animal is eating");
    }
}

public class Dog : Animal
{
    public void Bark()
    {
        Console.WriteLine("Dog is barking");
    }
}

Dog dog = new Dog();
dog.Eat(); // 输出 "Animal is eating"
dog.Bark(); // 输出 "Dog is barking"
```

抽象类：abstract
可以在派生类中实现
不能实例化

```csharp
public abstract class Animal
{
    public abstract void Eat();
}

public class Dog : Animal
{
    public override void Eat()
    {
        Console.WriteLine("Dog is eating");
    }
}
```

#### override

在 C#中，override 关键字用于在派生类中提供对基类中虚方法、抽象方法或重载方法的具体实现。当你在基类中定义了一个虚方法（使用 virtual 关键字）或抽象方法（使用 abstract 关键字），你可以在派生类中使用 override 来重新定义这个方法的具体行为。

#### 多态：

方法重载

```csharp
public class Dog : Animal
{
    public int Add( int a,int b)
    {
        return a+b;
    }
    public float Add( float a,float b)
    {
        return a+b;
    }
}
```

多态：父类引用指向子类对象，调用子类的方法

```csharp
Animal animal = new Dog();
animal.Eat(); // 输出 "Dog is eating"
```

#### 接口：interface

定义格式
不能实例化
一个类可以实现多个接口
默认是 public

```csharp
public interface IAnimal
{
    void Eat();
}

public interface IAnimal2
{
    void Run();
}

public class Dog : IAnimal,IAnimal2
{
    public void Eat()
    {
        Console.WriteLine("Dog is eating");
    }
    public void Run()
    {
        Console.WriteLine("Dog is running");
    }
}
```

#### 结构体 类 接口的区分

在 C#中，结构体（struct）、类（class）和接口（interface）有着不同的用途和特性，主要区别如下：  

结构体（struct）：  

结构体是值类型，这意味着当你将结构体实例赋值给另一个变量时，会复制整个结构体，而不是像引用类型那样只复制引用。  
结构体不能继承自其他结构体或类，但可以实现接口。  
结构体通常用于表示轻量级的数据结构，例如表示一个颜色、一个点或一个矩形等。  
结构体不能包含无参数的构造函数，除非是静态构造函数。  

类（class）：  

类是引用类型，当你将类实例赋值给另一个变量时，复制的是引用而不是对象本身。  
类支持继承和多态，可以继承自一个基类并可以实现多个接口。  
类可以包含构造函数、析构函数、字段、属性、方法等。  
类可以包含状态（字段）和行为（方法），适合用于创建复杂的数据模型和实现面向对象的设计模式。  

接口（interface）：  

接口定义了一组操作（方法、属性等），但没有实现这些操作，实现接口的类或结构体必须提供接口中定义的所有成员的具体实现。  
接口可以被类和结构体实现，一个类型可以实现多个接口，从而实现多重继承的效果。  
接口通常用于定义可以由不同类实现的通用功能或行为。  
选择使用结构体、类还是接口，通常取决于具体的需求和设计考虑：  

如果你需要一个轻量级的数据结构，且不需要继承和多态，结构体可能是一个更好的选择。  
如果你需要创建一个具有复杂行为的对象，并且希望利用继承和多态来创建一个面向对象的设计，类是更合适的选择。  
如果你需要定义一组操作，但不关心这些操作的具体实现方式，接口是很好的选择。  
