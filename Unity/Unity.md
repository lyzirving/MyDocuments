---
typora-root-url: pic
---

# 编译&&构建

## Mono && IL2CPP

本小节参考自：[关于Mono .Net IL2CPP的理解](https://blog.csdn.net/m0_74022070/article/details/131721987)。

### 1) .Net

#### 1.1) .Net是一种规范

.Net是微软的一种技术平台/一种规范，而不是一种语言，可以理解为接口。

.NET平台支持多种语言开发，如：C#、F#、Visual Basic等。

目前.Net有三种主流实现：

- .Net Framework：主要是基于Windows上开发；
- .Net Core：支持跨平台开发；
- Mono：支持跨平台开发。

跨平台的并不是C#，而是C#编译器编译后的**中间语言CIL**(Common Intermediate Language)。

#### 1.2) .Net代码的编译过程

- 某个语言(C#、Visual Basic)通过其特定的编译器生成**CIL**，它是一种**托管代码**，类似Java虚拟机；
- CIL是一种伪代码，不能被计算机直接识别。它与平台操作系统无关、与CPU无关，是一种**中间语言**，为跨平台奠定了基础。它会存储在.DLL或.EXE的程序集中。
- 程序运行时再通过**CLR**(Common Language Runtime)内部的JIT编译器将CIL编译成计算机可以识别的CPU指令(**机器码**：01010101)。
- IL语言是在CLR中运行的，而CLR并不知道IL是由哪种语言编译而来。

<img src="/pic_cil.png" alt="pic_cil" style="zoom:60%;" />

#### 1.3) 托管代码和非托管代码

- 托管代码

  托管代码包含中间语言，需要经过虚拟机/CLR转换为CPU指令，例如C#，Java等。

  托管代码不依赖于操作系统和CPU，在各个操作系统上都能执行，它是运行在虚拟机/CLR上的。

- 非托管代码

  非托管代码是直接对接CPU指令，代码执行效率高。

  不同的操作系统需要单独编写代码，重复低效，它是运行在机器上的。

  C++能跨平台可以理解为每个平台都实现了一套解析C++的运行库。

### 2) Mono

- Mono是什么

  因为.Net Framework只能在Windows平台上运行，对于跨平台的需求，Mono就应运而生。

  Mono是基于CLI和C#的ECMA标准提供的**.Net的另一种实现**，与.Net不同的是它**将CLR在所有支持的平台上重新实现了一遍**(安卓、Switch，PS4)并且还将.Net Framework提供的基础类库也重新实现了一遍。

- Mono的组成

  - C#编译器：C#编译器称为mcs，可以完成C#的编译工作：将C#源码编译成中间语言CIL。
  - Mono运行时(Mono VM)：实现了ECMA公共语言架构，提供了一个即时编译器(JIT)、预编译器(AOT)、类库加载器、垃圾回收器、线程系统和互操作性功能。
  - 基础类库(.Net类库)：提供一组全面的类，这些类兼容.Net框架并保持一致，是构建程序的结实基础。
  - Mono类库：提供了很多超越基础类库的类，提供了额外的功能，例如一些处理Gtk+，Zip文件，LDAP，OpenGL、Cairo、POSIX等等。

- Mono运行时(Mono VM)

  C#编译器mcs的作用是将C#源码编译成中间语言CIL，Mono运行时的作用是将CIL转换成机器语言。

  - 即时编译(JIT)

    在运行过程中，将CIL编译成机器码，**解释一条语句执行一条语句**，同时也会将编译过的代码进行缓存。

  - 提前编译(AOT)

    在运行前，将CIL编译成机器码并储存起来，但还是有一部分编译需要用到JIT。

  - 完全静态编译(Full AOT)

    运行前，将所有CIL编译成机器码。如IOS平台上是禁止JIT的，所以Mono只能以Full AOT模式运行。

<img src="/pic_il.png" alt="pic_il" style="zoom:70%;" />

### 3) IL2CPP

C#编译器编译得到中间语言后，使用IL2CPP将他们重新变回C++代码，然后再由各个平台的C++编译器直接编译成能执行的机器码：**将IL代码转换成CPP文件**。

IL2CPP出现的原因如下：

- Mono VM在各个平台移植，维护非常耗时，有时甚至不可能完成；
- Mono版本授权受限，Unity无法升级Mono版本导致一些新的C#特性无法使用；
- 提高运行效率，换成IL2CPP以后，程序的运行效率有了1.5-2.0倍的提升。

<img src="/pic_il2cpp.png" alt="pic_il2cpp" style="zoom:70%;" />

## 程序集定义Assembly Definition Asset

Assembly Definition核心：改变 Unity **默认的**”所有脚本编译成单一程序集“的模式，转向**基于依赖关系的模块化编译**。

<img src="/pic_ScriptCompilation.png" alt="pic_ScriptCompilation" style="zoom:80%;" />

### 1) 默认程序集

在没有 .asmdef 文件时，Unity 会将绝大多数脚本编译进一个名为**`Assembly-CSharp.dll`**的程序集中。

**任何脚本的微小改动都会触发整个项目所有脚本的重新编译**。随着项目扩大，编译时间会急剧变长。

### 2) .asmdef工作原理

- 递归定义

  在一个文件夹中创建 .asmdef 文件后，该**文件夹及其所有子文件夹内的脚本**会被编译成一个独立的动态链接库，除非子文件夹下有自己的.asmdef。

- 声明程序集依赖

  可以在.asmdef 文件中明确声明它需要引用的由其他.asmdef定义的程序集，Unity 的编译器会分析这些依赖关系，并按照正确的顺序进行编译。

- 依赖分析与增量编译

  当某个脚本被修改后，Unity 的编译系统会识别它所属的程序集，如A.dll。然后**只重新编译 A.dll 以及所有直接或间接依赖于 A.dll 的程序集**。这极大地减少了每次代码变更后的编译范围，从而提升迭代效率。

### 3) .asmref工作原理

.asmdef 是“定义”一个新的程序集；

.asmref是**“引用”**一个**已存在**的程序集，并将当前文件夹的脚本“加入”到那个程序集中。

<img src="/pic_asm.png" alt="pic_asm" style="zoom:30%;" />

核心工作机制：

- **脚本归属的“引用”原则**

  它归属于在**当前文件夹或父文件夹，层级上离它最近的那个 .asmdef 或 .asmref 文件**。

- **基于GUID的稳定引用**

  若在面板中勾选**`Use GUIDs`**，这意味着引用关系是通过目标程序集定义文件的 **GUID（全局唯一标识符）** 来记录的，而不是其名称。即使你重命名了目标.asmdef文件或其所在文件夹，所有引用它的.asmref文件都无需更新。

- **依赖关系统一管理**

  通过 .asmref 聚合到同一程序集中的所有脚本，**共享该程序集定义的依赖和设置**：

  ① 它们可以相互访问internal的变量(因为属于同一程序集)；

  ② 它们对外部的访问权限完全由目标程序集（.asmdef）所声明的引用决定。你不需要也无法在 .asmref 文件中单独设置依赖。

### 4) 程序集定义的优缺点

- 优点
  - 强制模块化，降低耦合；
  - 精确的平台控制；
  - 访问内部(internal)成员：使用.asmref，可以将**不同物理位置**的脚本文件夹逻辑上聚合到同一个程序集中。
- 缺点
  - 架构设计复杂化；
  - 循环引用问题：程序集之间**不允许**出现循环引用(A 引用 B，B 引用 C，C 又引用 A)。
  - 隐式全局依赖失效：在默认模式下，所有脚本都能直接访问所有类型。使用.asmdef后，跨程序集的类型访问必须通过`public`修饰符并显式添加引用。

# C#基础

## 值类型 && 引用类型

### 1) 栈内存和堆内存

- 栈内存

  - 栈是存放对象的容器，遵循**先进后出**的原则。
  - 它是一段**连续的内存**，对栈数据的定位很快。
  - 栈内存是生命周期确定的内存，栈中元素的生命周期跟随栈的生命周期，销毁时按后往前的次序销毁。

- 堆内存

  - 堆是随机分配的内存，处理的数据较多，任何情况，至少需要**两次才能定位**。

  - 堆内存可存放生命周期不明确的内存块，满足需要删除时再删除的需求，因此堆内存更灵活。

  - 一般堆内存在**托管堆内**，便于GC进行内存回收和控制。

    同时存在**非托管堆内存**，比如C++接口生成的内存块，将指针交给了C#程序，这个非托管堆的内存就需要我们**自行管理**。

### 2) 值类型

值类型的变量**直接存储数据**。

- 常见的值类型：byte、short、int、long、float、double、decimal、char、bool、**struct**。

- struct是值类型，赋值操作时，通常是通过**拷贝数据**完成的。
- 值类型对象的内存可以在栈内，也可在堆内，只是生命周期不同。

### 3) 引用类型

引用类型的变量持有的是**数据的引用**，其真实数据存储在**堆中**。

- 常见的引用类型：类实例、接口、**委托**、**数组**和内置的object与string。

- 当**声明**一个类实例时，只在堆/栈上分配一块小内存，用于**容纳地址**。此时还未为其分配堆上的内存空间。

  因此它是空的，为**null**。

- 使用**new**创建一个类的实例时，在堆上分配了空间，并把空间的地址保存给了上述的引用变量。

- 引用类型指针的内存，可在栈上，也可在堆上。

### 4) 装箱 && 拆箱

装箱/拆箱会不对分配/销毁内存，不仅会**消耗CPU**，也会造成**内存碎片**。

```c#
int a = 5;
//执行装箱
object obj = a;
//执行拆箱
int b = (int)obj;
```

#### 4.1) 装箱

- 流程
  - 在堆内存中分配一个内存块，大小：值类型实例 + 一个方法指针 + SyncBlockIndex类；
  - 值类型实例的值复制到内存块中；
  - 返回新分配对象的地址作为对象引用。

#### 4.2) 拆箱

- 流程
  - 检查对象实例，确保它是给定值类型的一个装箱值；
  - 将该装箱值复制值类型对象的内存中。

#### 4.3) 优化手段

- struct通过**重载函数**避免装箱/拆箱

  比如常用的ToString()、GetType()方法，如果没有重载，就会在struct实例调用它们前**先装箱，再调用**，导致内存重新分配。

- 通过泛型避免装箱/拆箱

  struct可以继承的，在不同、但相似的、父子关系的struct之间可使用泛型传递参数，避免装箱和拆箱。

  如B、C继承自A，可使用如下泛型函数，避免object引用类型传递参数，导致装箱。

  ```
  void Test(T t) where T : A
  ```

- 通过继承统一接口避免装箱/拆箱。

## 委托Delegate

在现代C#编程中，委托Delegate是一种类型安全的**函数指针**，**能够引用与其签名匹配的方法**，并在运行时动态调用。

```c#
// 语法
[访问修饰符] delegate [返回类型] 委托名([参数列表]);

// 示例
public delegate void Notify(string message);
```

### 1) 本质

- 委托是一个类类型

  当使用`delegate`关键字定义一个委托类型时，编译器会自动生成一个继承自`System.MulticastDelegate`的**密封类**(sealed)。

- 封装调用目标

  每个委托实例封装了**两个核心信息**：方法指针（`_methodPtr`）和目标对象（`_target`）；

  对于实例方法，`_target`指向方法所属的对象；对于静态方法，`_target`为 `null`。

- 支持多播

  委托继承自`MulticastDelegate`，内部通过`_invocationList`字段维护一个委托链表，使一个委托实例可以按顺序调用多个方法。这是通过 `Delegate.Combine`和 `Remove`方法实现的，为事件模型提供了底层支撑。

- 类型安全

  委托是类型安全的函数指针，它在编译时就会检查方法的签名是否与委托声明匹配。

### 2) 使用委托

#### 2.1) 定义委托类型

```c#
// 定义一个委托类型：返回类型为int, 有两个int参数
public delegate int MathOperation(int a, int b);

// 定义一个委托类型：无返回值, 有一个string类型参数
public delegate void LogHandler(string message);
```

- 定义委托类型，规定了待匹配方法的**返回类型**和**参数列表**。
- 编译器在后台会生成一个派生自`System.MulticastDelegate`的类。

#### 2.2) 使用委托实例

```c#
class Calculator
{
    public static int Add(int x, int y) {
        return x + y;
    }
    
    public static int Multiply(int x, int y) {
        return x * y;
    }
    
    public static int Divide(int x, int y) {
        return x / y;
    }
}

static void Main()
{
    // 创建委托实例    
    MathOperation mathOp = new MathOperation();
    
    // 添加方法
    mathOp += Calculator.Add;
    mathOp += Calculator.Multiply;
    
    // 先执行Add(10,5)，再执行 Multiply(10,5)，但lastResult为50
    int lastResult = mathOp(10, 5);
    
    // 清除前述累积的所有方法, 新建链表, 添加Divide方法
    mathOp = Calculator.Divide;
    
    // 移除方法
    mathOp -= Calculator.Divide;
}
```

- 使用委托对象，需要new一个**委托类实例**，分配在Heap上。

  delegate若使用不当，可能会造成GC压力或内存泄漏。

- `=`赋值操作

  设置/替换整个调用列表；

  `=`会清除之前的所有订阅，使用时需要留意。

- `+=`添加操作

  添加方法至调用列表，内部调用`Delegate.Combine`，将指定方法添加到当前委托的调用列表末尾；

- `-=`退订方法

  移除调用列表中的方法，内部调用`Delegate.Remove`，从当前委托的调用列表中查找并移除指定方法的最后一次出现。需要和`+=`配合使用。

- 避免内存泄漏

  `+=`不仅会持有实例方法，还会隐式持有**实例对象的引用**。因此，若不及时移除，可能会造成内存泄漏。

#### 2.3) 作为函数参数

```c#
//将委托作为参数的方法
static int Calculate(MathOperation operation, int x, int y)
{
    return operation(x, y); // 调用传入的函数
}

static void Main()
{
    int sum = Calculate(Calculator.Add, 5, 3);
    int product = Calculate(Calculator.Multiply, 5, 3);
    Console.WriteLine($"5+3={sum}, 5 * 3={product}");
}
```

#### 2.4) event关键字

`event`在委托的基础上又做了一层封装，其目的是限制用户操作`delegate`实例中变量的权限。

被`event`声明的委托**不能使用**`=`操作符，只能使用`+=`和`-=`操作符。

该操作避免了前面累积的函数链表被清空的风险，且保证了“谁注册就必须谁销毁”的秩序。

### 3) 内置委托: Action和Func

- Action

  Action是 .NET Framework 中**预定义好的泛型委托**。

  ```c#
  namespace System
  {
      // 封装方法: 无参数, 无返回值
      public delegate void Action();
      // 封装方法: 有一个参数, 无返回值
      public delegate void Action<in T>(T obj);  
      // ... 还有其他支持最多16个参数的重载
  }
  ```

  - 本质：`Action`本质上是一个委托类型：每一行使用`delegate`关键字定义`Action`及其泛型重载，都是在声明一种特定的委托类型。
  - 返回类型：所有的`Action`委托都返回`void`，这是它与`Func`委托的**根本区别**：`Func`委托必须指定一个返回类型。

- Func

  Func也是 .NET Framework 中**预定义好的泛型委托**。

  它与Action的主要区别是：必须有返回值，**最后一个泛型参数`TResult`固定为返回值类型**。

  ```c#
  namespace System
  {
      // 封装方法: 无参数, TResult类型的返回值
      // 可以接受 ref struct 类型作为类型参数
      public delegate TResult Func<out TResult>() 
          where TResult : allows ref struct;
      
      // ........
  }
  ```

## 闭包Closure

闭包是捕获了**外部变量**的**lambda**表达式，其本质是一个**对象(编译后)**。

### 1) 生成闭包

- **匿名类**：当它发现内部函数捕获了作用域外的变量时，编译器会在后台创建一个**sealed匿名类**。
- **变量提升**：原来在栈上的局部变量，会变成这个匿名类的一个**公共字段**。
- **函数迁移**：内部的闭包函数则被转换为这个匿名类的一个**实例方法**。
- **实例化**：在运行时，外部函数会**实例化这个匿名类**，调用类实例方法。

通过这种方式，捕获的变量就从栈内存“转移”到了托管堆上，其生命周期不再受限于原始函数调用的栈帧，而是由垃圾回收器GC管理，与引用它的**闭包对象**共存亡。

### 2) 常见问题

#### 2.1) 循环陷阱

参考自：[C# 中的闭包一个小问题](https://www.cnblogs.com/freesfu/p/17053318.html)，其中工具网站为：[sharplab](https://sharplab.io/)。

```c#
public void M() {
	var funs = new Action[10];

	for (var i = 0; i < 10; i++)
    	funs[i] = () => Console.WriteLine(i);

	foreach (var fn in funs)
    	fn();
}
//输出为 10, 10, 10, 10, 10, 10, 10, 10, 10, 10
```

通过sharplab编译，有：

```c#
[CompilerGenerated]
private sealed class <>c__DisplayClass0_0
{
    public int i;
    
    internal void <M>b__0()
    {
        Console.WriteLine(i);
    }
}

public void M()
{
    Action[] array = new Action[10];
    <>c__DisplayClass0_0 <>c__DisplayClass0_ = new <>c__DisplayClass0_0();
    <>c__DisplayClass0_.i = 0;
    while (<>c__DisplayClass0_.i < 10)
    {
        array[<>c__DisplayClass0_.i] = new Action(<>c__DisplayClass0_.<M>b__0);
        <>c__DisplayClass0_.i++;
    }    
    // ........
}
```

综上，在循环外面创建了**一个闭包实例**，导致了该问题。要解决改问题，需要在每个循环中创建一个闭包实例。

```c#
public void M() {
    var funs = new Action[10];
    for (var i = 0; i < 10; i++)
    {	
        int j = i;
        funs[i] = () => Console.WriteLine(j);
    }
}
```

使用sharplab编译：

```c#
public void M()
{
    Action[] array = new Action[10];
    int num = 0;
    while (num < 10)
    {
        <>c__DisplayClass0_0 <>c__DisplayClass0_ = new <>c__DisplayClass0_0();
        <>c__DisplayClass0_.j = num;
        array[num] = new Action(<>c__DisplayClass0_.<M>b__0); 
        num++;
    }
}
```

#### 2.2) 内存泄漏风险

闭包会延长其捕获的外部变量的生命周期，只要闭包对象还被引用，那么这些变量就会在内存中。

#### 2.3) 多线程下的竞态条件

当多个闭包在不同的线程中并发地修改同一个捕获的变量时，就会产生线程安全问题。

## await/async

### 1) Task与await/async

#### 1.1)基本用法

- **Task**：表示一个异步操作，用于执行异步任务并返回结果。
- **async**：声明异步方法，让方法内部可使用`await`。
- **await**：等待异步操作完成，避免阻塞主线程。

基本用法如下：

```c#
using System;
using System.Threading.Tasks;
using UnityEngine;
 
public class AsyncExample : MonoBehaviour
{
    void Start()
    {
        RunTask();
    }
 
    async void RunTask()
    {
        Debug.Log("任务开始");
        // 等待DoWorkAsync执行完毕后返回结果
        string result = await DoWorkAsync();
        // 没有显示切换主线程, 后续代码不一定会执行在主线程上
        Debug.Log($"任务完成，结果：{result}");
    }
 
    async Task<string> DoWorkAsync()
    {
        await Task.Delay(2000); // 模拟耗时任务
        return "Hello, Async!";
    }
}
```

进阶用法：

```c#
async void RunMultipleTasks()
{
    Task<int> task1 = GetNumberAfterDelay(1, 2000);
    Task<int> task2 = GetNumberAfterDelay(2, 1000);
    
    int[] results = await Task.WhenAll(task1, task2);
    // 没有显示切换主线程, 后续代码不一定会执行在主线程上
    Debug.Log($"任务完成：{results[0]}, {results[1]}");
}
 
async Task<int> GetNumberAfterDelay(int number, int delay)
{
    await Task.Delay(delay);
    return number;
}
```

#### 1.2) 显示切换至主线程上下文

- 使用SynchronizationContext

  ```c#
  private void Start()
  {
      // 在主线程启动时捕获主线程的同步上下文
      _mainThreadContext = SynchronizationContext.Current;
      StartAsyncWork();
  }
  
  private async void StartAsyncWork()
  {
      // 在后台线程执行耗时计算
      var result = await Task.Run(() => SomeHeavyCalculation());
       
      // 此时不一定回到了主线程
      // 因为Task.Run内部的await可能会在某个线程池的线程上恢复
      // 因此, 为了确保100%安全, 需要显示切换为主线程
      // 当前有两种方法：
      // 1. 使用捕获的上下文切换回主线程
      // 2. 使用_mainThreadContext.Post进行更精细控制
      await _mainThreadContext; 
      
      // 此时已回到主线程，可安全操作Unity对象
      GameObject.CreatePrimitive(PrimitiveType.Cube);
      Debug.Log("Result: " + result);
  }
  
  private int SomeHeavyCalculation()
  {
      // 模拟耗时计算
      Thread.Sleep(1000);
      return 42;
  }
  ```

- 自定义主线程分发器

  ```c#
  using UnityEngine;
  using System.Collections.Concurrent;
  using System.Threading.Tasks;
  
  public class MainThreadDispatcher : MonoBehaviour
  {
      private static readonly ConcurrentQueue<System.Action> _executionQueue = 
          new ConcurrentQueue<System.Action>();
      private static MainThreadDispatcher _instance;
  
      public static MainThreadDispatcher Instance
      {
          get
          {
              if (_instance == null)
              {
                  var go = new GameObject("MainThreadDispatcher");
                  _instance = go.AddComponent<MainThreadDispatcher>();
                  DontDestroyOnLoad(go);
              }
              return _instance;
          }
      }
  
      // 供其他线程调用的公共方法
      public static void ExecuteOnMainThread(System.Action action)
      {
          if (action == null) return;
          _executionQueue.Enqueue(action);
      }
  
      private void Update()
      {
          // 在主线程的每一帧处理队列中的动作
          while (_executionQueue.TryDequeue(out var action))
          {
              action?.Invoke();
          }
      }
  }
  ```

- 使用TaskScheduler

  `TaskScheduler`控制 `Task`的执行方式，通过获取主线程的`TaskScheduler`，可调度代码在主线程运行。

  ```c#
  using UnityEngine;
  using System.Threading.Tasks;
  
  public class TaskSchedulerExample : MonoBehaviour
  {
      private TaskScheduler _mainThreadScheduler;
  
      private void Start()
      {
          // 在主线程捕获任务调度器
          _mainThreadScheduler = TaskScheduler.FromCurrentSynchronizationContext();
          StartAsyncWork();
      }
  
      private async void StartAsyncWork()
      {
          var data = await Task.Run(() => "来自后台线程的数据");
  
          // 使用捕获的调度器继续执行后续代码
          await Task.Factory.StartNew(() => 
          {
              // 此代码块将在主线程执行
              Debug.Log("数据: " + data);
              new GameObject("MainThreadObject");
          }, CancellationToken.None, TaskCreationOptions.None, _mainThreadScheduler);
  
          Debug.Log("后续工作...");
      }
  }
  ```

### 2) await/async原理详解

本小节参考自：[C# await\async原理](https://zhuanlan.zhihu.com/p/1949842363578057547)。

`await`是.Net的新增特性，通过**状态机**和**回调/延续**实现。

整个过程涉及**编译器**、**运行时(任务调度器)**和**底层基础设置(如Unity的PlayerLoop)**的协作。

#### 2.1) 编译器生成状态机

当你编写一个`async`方法并使用`await`时，C#编译器会**重写**你的方法。

它会将你的方法体转换成一个**状态机类**：

- **跟踪当前执行位置：** 它记录`await`语句出现的位置和状态；
- **存储局部变量：**将原方法中的局部变量提升为状态机类的字段，以便在挂起和恢复时保持它们的值。
- **实现`IAsyncStateMachine`接口：**该接口包含核心方法`MoveNext()`，它驱动状态机的执行。

#### 2.2) await：挂起而非阻塞

当执行流遇到`await someTask`：

① **检查任务状态：** 首先检查 `someTask` 是否**已经完成**；

② **如果Task已完成(同步完成)：** 状态机直接执行`await`之后的代码，如同没有`await`，且没有挂起发生。

③ **如果Task未完成，挂起方法(返回控制权)：**

- 状态机保存当前状态(执行位置)；

- 状态机向`someTask`**注册一个回调(Continuation)**。

  这个回调本质上是一个**委托**，指向状态机的`MoveNext()`方法，并告诉它：当 `someTask` 完成后，从这里继续执行。

- `async`方法**立即返回一个`Task`(或 `UniTask`)给调用者**。

  这个返回的任务代表整个异步操作的最终完成状态(**可能尚未完成**)。

- **当前线程被释放** 

  这是最重要的点。调用`await`的线程**不会被阻塞**，可以去做其他事情。

#### 2.3) 异步任务完成

- 当`someTask`所代表的底层异步操作**最终完成**时：

  - **任务被标记为完成：**`someTask`的内部状态被设置为`Completed`(或 `Faulted`/`Canceled`)。
  - **调度回调：** 完成`someTask`的机制(可能是 I/O 完成端口、Unity 的 PlayerLoop、线程池、事件系统等)负责**调度**之前注册的回调(即状态机的 `MoveNext()` 方法)执行。

- 回调执行上下文

  - 在Unity 中，如果`await`时指定了`PlayerLoopTiming.Update`或类似参数，或使用了 `UniTask` 的默认配置，这个回调通常会被安排到Unity 的**主线程** ，在下一帧指定阶段执行。

    这是`UniTask`相对于标准`Task`的一个巨大优势，它天然理解 Unity 的线程模型。

  - 标准 .NET `Task` 的回调默认可能在线程池线程执行。

#### 2.4) 恢复执行：状态机推进

当调度器，如Unity的PlayerLoop，执行了注册的回调(调用了状态机的MoveNext)方法：

- **恢复状态：** 状态机加载之前保存的执行位置(状态)和局部变量值；
- **获取结果(optional)：**如果 `await`的是一个有返回值的任务(如 `UniTask<int>`)，状态机会获取任务的结果；
- **继续执行：**状态机从 `await` 语句**之后**的代码开始继续执行，就像它从未离开过一样。局部变量的值也保持原样。

### 3) await/async结合UniTask

#### 3.1) UniTask的优势

- SynchronizationContext是单点回调机制

  .NET原生的`Task`使用`SynchronizationContext`来调度`await`之后的代码。

  它虽然能确保代码回到主线程，但无法精确控制回调发生的**时机**。

- **UniTask深度集成到Unity的PlayerLoop中**

  UniTask将自身的调度逻辑直接**注入**到了PlayerLoop的多个关键时间点。

  当异步操作完成时，其回调会被放入一个队列，由PlayerLoop在**下一帧或指定的帧阶段**取出并执行。

- **UniTask的回调触发流程**

  - 如果任务未完成，状态机挂起，并向 `UniTask` 注册一个回调；
  - `UniTask`内部确保这个回调在**指定的 Unity 主线程帧阶段**(如 `Update`, `LateUpdate`, `FixedUpdate`, `EndOfFrame` 等)被调用；
  - 当异步操作完成时，`UniTask`会将该状态机的`MoveNext()`方法**加入**到对应帧阶段的待执行回调队列中。
  - Unity 引擎在运行到那个特定的帧阶段时，会**遍历并执行**该队列中的所有回调，从而恢复你的`await`后的的代码。

- **性能优化**

  `UniTask` 通过避免`Task`的GC分配、使用值类型状态机、与PlayerLoop高效集成等方式，大幅提升了 Unity 中异步编程的性能。

#### 3.2) 示例

- 按严格顺序执行

  ```c#
  public class WorkflowManager : MonoBehaviour
  {
      async UniTaskVoid StartGameSequence()
      {
          // 顺序执行：有严格依赖关系的步骤
          await LoadConfigAsync();        // 必须先加载配置
          await AuthenticateUser();       // 然后进行用户认证
          await PreloadEssentialAssets(); // 接着预加载核心资源
  
          // 并行执行：同时加载多个独立资源，提升效率
          var (uiAssets, audioClips, sceneData) = await UniTask.WhenAll(
              LoadUIAssetsAsync(),
              LoadAudioClipsAsync(),
              LoadSceneDataAsync()
          );
  
          Debug.Log("所有准备工作完成，进入游戏！");
      }
  
      private async UniTask LoadConfigAsync() { /* ... */ }
      private async UniTask AuthenticateUser() { /* ... */ }
      private async UniTask PreloadEssentialAssets() { /* ... */ }
      private async UniTask<GameObject[]> LoadUIAssetsAsync() { /* ... */ }
      private async UniTask<AudioClip[]> LoadAudioClipsAsync() { /* ... */ }
      private async UniTask<Texture2D> LoadSceneDataAsync() { /* ... */ }
  }
  ```

- 指定回调的帧阶段

  下述展示了`UniTask.Yield`的方式，当然还有其他方式：

  ```c#
  using Cysharp.Threading.Tasks;
  using UnityEngine;
  
  public class YieldExample : MonoBehaviour
  {
      private async UniTaskVoid Start()
      {
          // 在 EarlyUpdate 阶段继续执行
          await UniTask.Yield(PlayerLoopTiming.EarlyUpdate);
          Debug.Log("这段代码在 EarlyUpdate 阶段执行");
  
          // 在 FixedUpdate 阶段继续执行
          await UniTask.Yield(PlayerLoopTiming.FixedUpdate);
          Debug.Log("这段代码在 FixedUpdate 阶段执行");
  
          // 在 PostLateUpdate 阶段继续执行（适合低优先级任务）
          await UniTask.Yield(PlayerLoopTiming.PostLateUpdate);
          Debug.Log("这段代码在 PostLateUpdate 阶段执行，用于资源清理等");
      }
  }
  ```

#### 3.3) 线程切换的显示控制

尽管UniTask能自动切回主线程，但它也提供了在需要时进行**显式、可控线程切换**的能力。

```c#
async UniTaskVoid FlexibleMethod()
{
    // 开始在主线程...

    // 显式切换到线程池执行耗时计算
    await UniTask.SwitchToThreadPool();
    // 在线程池执行
    var result = PerformHeavyCalculation(); 
	........
        
    // 计算完成后，再显式切换回主线程
    await UniTask.SwitchToMainThread();
    // 在主线程执行，安全操作Unity对象
    this.gameObject.name = $"Result: {result}"; 
}
```

## Burst优化异步编程

# 引擎基础

## MonoBehaviour

<img src="/pic_hierarchy.png" alt="pic_hierarchy" style="zoom:100%;" />

- 继承关系

  C#所有引用类型的基类是System.Object；

  UnityEngine.Object继承System.Object，它是Unity中所有引用类型的基类；

  Component继承UnityEngine.Object，它是**组件的基类**，代表所有附加在GameObject上的对象；

  Behaviour是Component的子类，在Component的基础上，**添加了enable（启用或禁用）的能力**；

  Behaviour有很多子类，比如Animator、Light、Camera等，其中就有MonoBehaviour。

- MonoBehaviour的作用

  - **生命周期管理**

    提供Awake、Start、Update等回调函数，是我们可以精确控制脚本在不同阶段的初始化、更新和销毁逻辑。

  - **组件化架构**

    每个MonoBehaviour脚本都是一个**独立的功能模块**，可以灵活地挂载到GameObject上，这是一种遵循**单一职责原则的设计**，使得游戏对象的功能组合更加灵活。

  - **丰富的引擎继承功能**

    提够了提供了系统协程、物理回调、消息传递能功能。

## 协程

Unity协程的实现包括**两个部分**：1）编译器生成状态机；2）Unity引擎在**主线程**统一管理和调度。

```c#
IEnumerator MyCoroutine() {
    Debug.Log("A");
    yield return new WaitForSeconds(1);
    Debug.Log("B");
}
```

### 1) 状态机类

C#会将其编译为如下的**状态机类**（编译后的隐藏类），通过状态变量和分段执行实现协程的暂停/恢复功能。其中`MoveNext()`中的逻辑：1）执行当前代码语句；2）执行`yield`指令后的代码；3）变量更新；

```c#
class <MyCoroutine> : IEnumerator {
    private int _state; //状态变量
    private object _current; //yield指令暂停的分割点
    
    bool MoveNext() {
        switch(_state) {
            case 0: // 对应yield之前的代码
                Debug.Log("A");
                _current = new WaitForSeconds(1);
                _state = 1;
                return true;
            case 1: // 对应第一个yield之后的代码
                Debug.Log("B");
                _state = -1; // 结束标记
                return false;
        }
        return false;
    }
}
```

### 2) 执行时机

- 协程不是多线程，它和 `Update`、`Start` 这些生命周期函数一样，始终运行在Unity的**主线程**上；

- 调用`StartCoroutine`时，协程会**立即开始执行**，直到遇到第一个`yield return`语句；
- 遇到`yield return`后，协程被**暂停**，控制权交还给Unity引擎的主循环；
- 引擎会在**每一帧的特定时刻（`LateUpdate`方法执行完毕之后）**，检查所有被暂停的协程的等待条件是否满足；
- 如果某个协程的等待条件满足了，就会在上次暂停的地方继续执行，直到遇到下一个 `yield return` 或协程方法结束。

### 3) Unity引擎的协程调度

Unity引擎底层维护了一个活跃协程列表：

- 每帧末尾检查所有活跃协程；
- 通过`MoveNext()`推进协程到下一个`yield`点；
- 自动移除已完成的协程。

伪代码如下：

```c#
class CoroutineScheduler {
    List<IEnumerator> _activeCoroutines;
    
    void Update() {
        for(int i=0; i<_activeCoroutines.Count; i++) {
            var coroutine = _activeCoroutines[i];
            // 检查yield条件是否满足
            if(IsYieldConditionMet(coroutine.Current)) { 
                if(!coroutine.MoveNext()) { // 推进状态机
                    _activeCoroutines.RemoveAt(i--);
                }
            }
        }
    }
    
    bool IsYieldConditionMet(object yieldInstruction) {
        if(yieldInstruction == null) return true; // yield return null
        if(yieldInstruction is WaitForSeconds wfs) 
            return Time.time >= wfs._resumeTime;
        // 其他yield类型判断...
    }
}
```

## 资产系统

### Asset 和 Object

#### 1) 概述

- Asset：硬盘上的文件，放在项目的Asset目录下

  纹理、模型、音频都是常见的Asset。

  一些资产是Unity**原生类型**；

  一些资产不是原生类型，需要通过**AssetImporter**导入，例如fbx。

  被导入的资产被缓存在**Library**文件夹下，重启Editor后就不需要再导入。

- UnityEngine.Object：序列化数据的集合，用于描述某一类资源

  UnityEngine.Object是所有对象的基类，如mesh、sprite、AudioClip等。

- 两个特殊的对象类型：ScriptObject、MonoBehavior。

  - ScriptObject：开发者可定义其自己的资产类型。
  - MonoBehavior：可链接到MonoScript。

- Asset和Object是一对多的关系

  一个特定的资产文件，包含一个或多个Objects。

#### 2) 对象间的引用

材质对象(Object)通常持有一个或多个纹理对象的引用，这些纹理对象由一个或多个纹理资产导入(比如JPG或PNG)。

- 引用的组成：**File GUID** 和 **Local ID**

  - File GUID：存放在**.meta文件**中，用于标识资源被存储在哪个文件中，是对**文件具体位置的抽象**。
  - Local ID：由于一个资产可以包含多个Object，Local ID用于标识资产内部不同的Object。
  - 通过组合File GUID和Local ID，Object可以被**唯一标识**。

- 引用系统的特性

  - 使资产与硬盘位置无关

    File GUID可唯一标识一个文件。即使文件路径变化了，也不需要更新其序列化的Object。

  - 若文件的File GUID失效了，那么由该资产序列化的所有Object也会失效。

- Unity Editor维护**文件路径和File GUID的映射**

  编辑器会维护资产路径和GUID的映射。当资产被导入时，会构建entry<path, GUID>。

  当编辑打开时，若.meta文件失效了，且资产的路径没变，Unity会保证资产重新获取其File GUID。

  若.meta丢失且编辑器关闭，那么该资产会失效，连同相关的Objects。

#### 3) 序列化和实例化

- PersistentManger

  Unity内部通过*PersistentManager*维护一块**缓存**(cache)。

  cache管理了File GUID、Local ID和Instance ID的映射。

- Instance ID

  - 将File GUID、Local ID转换为会话唯一的整数，即Instance ID。
- Instance ID是单调递增的。当新创建的Object时，会为其赋予Instance ID。
  - 当访问File GUID、Local ID的AssetBundle被卸载时，Instance ID会在cache中被移除。如果相同的AssetBundle被re-load，会为其分配一个全新的Instance ID。
- 在运行时比较File GUID、Local ID是**性能不友好**的，这是设计Instance ID的初衷。

#### 4) MonoScripts

*MonoBehavior*持有一个*MonoScripts*的引用，*MonoScripts*持有定位*script class*的信息。

*MonoScripts*包含三个string：程序集名、类名、命名空间。

**在Plugin外**的C#脚本会被编译至*Assembly-CSharp.dll*；

**在Plugin内**的C#脚本会被编译至*Assembly-CSharp-firstpass.dll*。当然Unity也允许自定义程序集。

正是由于存在 MonoScript 对象，AssetBundle（或场景或预制件）中包含的任何 MonoBehaviour 组件实际上都不包含可执行代码。这使得不同的 MonoBehaviour 即使位于不同的 AssetBundle 中，也可以引用特定的共享类。

### 资源对象的生命周期

为了减少加载时间并管理应用程序的内存占用，理解 `UnityEngine.Object`的资源生命周期至关重要。

#### 1) 加载Object

- 加载时机

  - **解引用**Object的Instance ID

    当Object的源数据在磁盘上能够被定位，且没有被加载到内存中；

    若需要通过Instance ID去访问Object对应的源数据，此时就会通过File GUID、Local ID去执行源数据加载。

  - 调用resource-loading API或显示创建Object。

- 解析Object内部关联的所有Object

  当一个父Object被解析时，Unity会解析其内部所有的关联Object。

  此时这些Object并未被真正加载，Unity只是将其File GUID、Local ID转换为Instance ID。当这些Object需要真正被使用时，才解引用Instance ID，加载对应的源数据。

#### 2) 卸载Object

- 执行未使用的资产清理

  当场景被销毁或调用了*Resource.UnloadUnusedAssets*的API。

  这个过程只卸载**未被引用的对象**。

- 在*Resources*文件夹下的对象可以被*Resource.UnloadAsset* API卸载

  这些对象的Instance ID仍然有效，File GUID、Local ID的映射仍然存在。

  若其他地方解引用了该对象，其对应的源数据会被re-load。

- 从*AssetBundle*加载的对象可被*AssetBundle.Unload(true)*卸载

  这些对象的File GUID、Local ID会失效。

  任何对该对象的引用都会成为"Missing Reference"，访问这些对象会产生*NullReferenceException*。

### Resources文件夹

#### 核心建议——不使用Resources系统

- 使用Resources文件夹会使得精细化的内存管理变得更加困难。

  ResourcesAPI不提供复杂的生命周期管理。开发者需要手动跟踪资源的加载和卸载，容易导致资源泄露或重复加载。

- 对Resources文件夹的不当使用会增加应用程序的启动时间和构建时间。

  所有`Resources`文件夹下的资产会被合并到一个序列化文件中，并为其生成一个用于快速查找的索引。

  应用启动时必须加载这个庞大的索引文件，**即使里面的很多资源在初始场景中并不会立即用到**，这会显著拖慢启动速度。

- 随着Resources文件夹数量的增加，管理这些文件夹中的资产会变得非常困难。

- Resources系统会降低项目向特定平台交付定制内容的能力，并且消除了进行增量内容更新（如热更新）的可能性。

  由于`Resources`中的资源被直接**打包进应用本体**，除非更新整个应用，否则无法单独更新这些资源。

AssetBundle系统是整个Unity资源管理的核心，它解决了`Resources`系统的几乎所有短板。

#### 正确使用Resources系统的场景

- **Resources文件夹的易用性使其成为快速原型开发的绝佳系统**。当项目进入全面生产阶段时，应停止使用Resources文件夹。
- 在一些简单的情况下，Resources文件夹也可能有用：
  - 资源通常在项目的整个生命周期中都需要；
  - 非内存密集型（不占用大量内存）；
  - 不易需要打补丁，或者不因平台或设备而异；
  - 用于最小化的启动引导（Bootstrapping）。

#### Resources系统的序列化

- 当项目构建时，所有名为 "Resources" 的文件夹中的**资产（Assets）**和**对象（Objects）** 会被合并到**一个单一的序列化文件**中。该文件还包含**元数据和索引信息**，类似于一个AssetBundle。

- 该索引包含一个**序列化的查找树结构**，用于将给定的对象名称解析为对应的**文件GUID**和**本地ID**。

  它也用于在序列化文件主体中定位特定字节偏移量处的对象。

  在大多数平台上，该查找数据结构是一个**平衡搜索树**，其构建时间以 **O(n log(n))** 的速率增长。

- 随着Resources文件夹中对象数量的增加，索引的加载时间会以**超线性**的方式增长。

  此操作**无法跳过**，并且在应用程序启动时、显示启动画面的期间发生。

  它会构建所有资源的索引，即使它们在初始场景中不会用到。

### AssetBundle原理

- AssetBundle的本质是一个**归档格式**。

  它将多个独立的资源文件（如模型、纹理、音频等）打包整合成一个单一的文件，类似于ZIP压缩包。

  Unity引擎能够高效地识别、索引和序列化这种特定格式。

- Unity进行**非代码内容热更新**的"主要工具"。

  - 减小初始安装包体积：非核心资源可放在服务器上，用户只需按需下载。
  - 灵活的运行时内存管理：允许你按需加载资源，并在使用完毕后及时卸载。
  - 支持内容差异化与优化：针对不同的设备性能（如高端/低端GPU）、屏幕分辨率或语言地区，创建不同版本的AssetBundle（即AssetBundle Variants）。在运行时能够有选择性地加载。

#### AssetBundle数据布局

AssetBundle由两部分组成：**头文件（header）** 和**数据段（data segment）**。

- 头文件

  **头文件**包含关于AssetBundle本身的信息，例如其标识符、压缩类型和一个**清单(manifest)**。

  manifest是一个以Object名称作为key的查找表。每个条目(entry)使用一个字节作为索引，指明Object在AssetBundle数据段中的位置。

- manifest是**平衡搜索树**

  特别的，Windows和OSX衍生平台(包括iOS)采用**红黑树**。

  随着AssetBundle内资源数量的增长，构建此清单所需的时间将会以**超线性**的方式增加。

- 数据段 && 压缩

  **数据段**包含将通过AssetBundle中的资源序列化后生成的原始数据。

  如果压缩方案指定为**LZMA**，则所有序列化资源的完整字节数组会被一起压缩。

  如果指定的是**LZ4**，那么不同资源的字节数据会被**分别独立压缩**。

  如果不使用压缩，数据段将保持为原始的字节流。

- Unity5.3之前

  Unity 5.3之前，对象无法在AssetBundle内部进行独立压缩。

  因此，如果需要从压缩的AssetBundle中读取一个或多个对象，Unity 5.3之前的版本将不得不解压整个AssetBundle。

  通常，Unity会**缓存一份AssetBundle解压后的副本**，以提高后续对同一AssetBundle加载请求的性能。

#### 加载AssetBundle

AssetBundle可以通过四种不同的API进行加载。这四种API的行为差异取决于两个条件：

1) AssetBundle是经过LZMA压缩、LZ4压缩，还是未压缩的。
1) 加载AssetBundle所在的平台。

##### 1) AssetBundle.LoadFromMemory(Async)

**Unity 建议不要使用此 API。**

- AssetBundle.LoadFromMemoryAsync从一个**托管**的字节数组（C# 中的 byte[]）加载 AssetBundle

- 加载过程中，它将源数据从**托管**代码的字节数组复制到一个新分配的、连续的**本地(native)**内存块中。

- 如果 AssetBundle 采用 LZMA 压缩，它会在复制过程中解压该 AssetBundle。

  未压缩和 LZ4 压缩的 AssetBundle 则会原样复制。

- 严重缺陷：加载流程及**三次内存消耗**

  - **托管数组副本**：开发者准备的原始 `byte[]`数组本身**占用一份内存**。
  - **本地内存副本**：API 内部会将 `byte[]`的完整数据复制到 Unity 引擎管理的**本地内存**中，并在此过程中**根据需要进行解压**(如果是 LZMA 格式，需要解压缩)。这是**第二份**内存占用。
  - **资源本身内存**：当从AssetBundle中加载出具体的资源(如纹理、模型)时，这些资源数据会被载入显卡或系统内存中供游戏使用。这是**第三份**内存占用。

在Unity5.3.3之前，该API为**AssetBundle.CreateFromMemory**。

##### 2) AssetBundle.LoadFromFile(Async)

AssetBundle.LoadFromFile是一个**高效的 API**，应作为**首选**。

- 按需加载：仅加载头文件，数据按需从磁盘读取

  在桌面独立平台、游戏主机和移动平台上，该 API **仅加载AssetBundle的头文件(header)**。

  最终，在调用加载方法(例如 `AssetBundle.Load`)时或Object的InstanceID被解引用时，数据才被加载。

- 编辑器的特例

  在编辑器模式下，其**将整个AssetBundle加载到内存**。这是因为编辑器环境下的I/O操作和行为与真机不同。

  因此，在编辑器下进行性能剖析Profiling时，会观察到较大的内存峰值，但这**并不代表最终发布版本的真实性能**。

##### 3) AssetBundleDownloadHandler

##### 4) WWW.LoadFromCacheOrDownload

#### 从AssetBundle中加载资产

可以使用三种不同的API从AssetBundle中加载`UnityEngine.Object`，这些API都有同步和异步版本。

| API                    | 用途                   | 适用场景                                                     | 性能特点                                         |
| ---------------------- | ---------------------- | ------------------------------------------------------------ | ------------------------------------------------ |
| LoadAsset              | 加载单个独立资源       | 需从AssetBundle中加载一个或少量几个特定资源                  | 同步快于异步，异步可避免卡顿                     |
| LoadAllAssets          | 加载包内大量或全部资源 | **66%规则**。如果要频繁加载一个AssetBundle中超过三分之二的资源，使用`LoadAllAssets`是高效的 | 批量加载效率高于多次调用`LoadAsset`              |
| LoadAssetWithSubAssets | 加载复合资源及其子资源 | 处理**单一主资源包含多个子对象**的情况                       | 能一次性加载逻辑上关联的所有子对象，避免多次请求 |



# 引擎工具

## Cinemachine

本小节参考自[Unity官方文档Cinemachine注解](https://docs.unity3d.com/Packages/com.unity.cinemachine@3.1/manual/get-started.html)。

### 1) 场景必要元素

<img src="/pic_cinemachine.png" alt="pic_cinemachine" style="zoom:70%;" />

- 一个捕捉场景的Unity Camera
  - 实际是一个包含Camera组件的GameObject，它会被Cinemachine控制。
  - Cinemachine的环境中有且只能有一个Unity Camera。
- Unity Camera中需要一个`Cinemachine Brain`组件
  - 监控场景中所有`Cinemachine Camera`;
  - 决定哪个`Cinemachine Camera`会控制`Unity Camera`；
  - 当`Cinemachine Camera`控制权变化时，处理转场。
- 一个或多个`Cinemachine Camera`，用于轮流控制Unity Camera。
  - `Cinemachine Camera`在旧的版本中被称为`Virtual Camera`。
  - 实际是持有`Cinemachine Camera`组件的GameObject，动态覆盖Unity Camera的行为和属性。

### 2) Cinemachine Camera的状态

#### 2.1) 状态列表

除了转场混合，其他任何时刻，`Cinemachine Camera`只会有一个处于Alive状态。

| 状态     | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| Alive    | Alive状态下，Cinemachine Camera正在控制Unity Camera。<br />当一个Cinemachine Camera向另一个Cinemachine Camera过渡时，会有两个处于Alive状态。完成后仍然只会有一个处于Alive。 |
| Standby  | 此时不控制Unity Camera，但仍然跟随、瞄准target，且会执行update。<br />当前状态下GameObject仍然是active的，但优先级小于等于Alive状态下的Cinemachine Camera。 |
| Disabled | 此状态不会控制Unity Camera，且不会跟踪、瞄准target；<br />要禁用Cinemachine Camera，直接deactive它的go就可以了。此状态下CinemachineCamera不会有调用消耗。 |

#### 2.2) 如何选择Live的Cinemachine Camera

默认情况下，Cinemachine Brain会负责选择Live的CinemachineCamera：

- 按最高优先级选取；
- 如果多个CinemachineCamera有相同的优先级，那么选取最近被激活的CinemachineCamera；
- 在相机混合的过程中，deactive或低优先级的CinemachineCamera可以被激活。混合结束后，又被设置到deactive。

#### 2.3) CinemachineCamera的过渡效果

- 混合Blends

  针对CinemachineCamera的位置、旋转和其他属性，执行一个从A到B的平滑动画。如下所示，动画完成后有CinemachineCamera2来控制Unity Camera

  <img src="/pic_concept-transition-blend.png" alt="pic_concept-transition-blend" style="zoom:80%;" />

- 突变Cuts

  Blend是平滑过渡，Cuts是突变，即Unity Camera掌控权变化时，没有过渡。

  <img src="/pic_concept-transition-cut.png" alt="pic_concept-transition-cut" style="zoom:80%;" />

## Timeline

使用Unity Timeline构建的剪辑、影片、游戏序列会包含一个**Timeline资产**和**Timeline对象**。

### 1) Timeline asset

存储轨道track、动画片段clip。这些内容是独立的，与GameObject无关。

<img src="/pic_timeline_asset.png" alt="pic_timeline_asset" style="zoom:80%;" />

### 2) Timeline instance

Timeline instance是Timeline asset和GO之间的**链接**，若使用Timeline asset来驱动GO执行动画，需要创建一个**Timeline instacne**。

Timeline instance通过**Playable Director组件**来关联Timeline asset和GO。

当你选择场景中持有Playeable Director组件的GO时，关联的Timeline instance会显示在Timeline Window中：

<img src="/pic_spec-tl-overview-instance.png" alt="pic_spec-tl-overview-instance" style="zoom:90%;" />

上图中，GroundATL这个Timeline asset**被绑定**到Ground这个GO上。

### 3) 复用Timeline asset

由于Timeline asset和Timeline instance是分开的，因此Timeline asset是可以重用到多个Timeline instance上。

下图所示，Timeline asset VictoryTL，内部包含了一个animation、music和粒子特效，被绑定到了Player这个GO上。

<img src="/pic_reuse_1.png" alt="pic_reuse_1" style="zoom:90%;" />

同样的，Timeline asset VictoryTL也被绑定到Enemy这个GO上：

<img src="/pic_reuse_2.png" alt="pic_reuse_2" style="zoom:90%;" />

由于Timeline asset是重用的，因此对Timeline asset VictoryTL的修改会影响多个GO。

### 4) Timeline操作

本小节参考自：[官方文档: Timeline workflows](https://docs.unity3d.com/Packages/com.unity.timeline@1.8/manual/wf-overview.html)。

#### 4.1) 录制动画

- 把GO拖拽到Timeline window的track list中，Timeline window会创建一个空的轨道，并为GO添加**Animator**组件；

- 点击录制后，拖动进度条到指定位置，修改GO的属性，会自动添加一个关键帧。

  若在Inspector中右键点击GO的属性，也会弹出弹窗，并有“Add Key”选项。

- 完成录制后，会为Timeline asset下创建一个子对象。

- 如下操作，可把Infinite clip转变为Animation Clip：

  <img src="/pic_spec-tl-menus-convert-clip.png" alt="pic_spec-tl-menus-convert-clip" style="zoom:80%;" />

# UGUI

## 原理

### 1) 概述

- 每个UI元素都是3D网格

  UGUI每个可显示元素都是通过**3D网格**构建起来的，每个网格绑定一个材质，材质里存放要显示的图片。

- drawcall优化

  如果材质太多，会导致drawcall过高，给GPU造成很大负担，UGUI通过合批优化了这种情况。

- 合批与重建

  - 制作**图集**，统一材质，把**相同层级**的、拥有统一材质参数的UI元素的网格合并，制作为一个统一的静止模型。
  - 若销毁了任何元素、改变了任何元素的参数、移动了任何元素，原来合批的网格就不符合要求，需要销毁并重建。
  - 需要想办法尽可能合并更多的网格，减少网格的重构次数，以达到更少的性能开销。

### 2) Canvas

#### 2.1) Canvas的作用

各类UI元素放到Canvas(*画布*)下后，Canvas的工作就是合并这些元素。

- 控制UI元素绘制顺序

  - UI元素的绘制顺序和其在Canvas层级中的显示顺序一致，如层级中第一个元素最先绘制，依次类推。
  - 可通过拖拽来改变UI元素的绘制顺序，也可通过Transform组件的：`SetAsFirstSibling`，`SetAsLastSibling`和`SetSiblingIndex`方法改变顺序。
  - 最后绘制的元素在最上层。

- Canvas嵌套

  - Canvas组件可以嵌套在另一个Canvas组件下，即**子Canvas**。
- 子Canvas可以把它的子物体与父Canvas分离。
  - 当子Canvas被标记为Dirty时，不会强制Rebuild父Canvas，反之亦然。

- 驱动UI更新

  - 通过**CanvasUpdateRegistry**系统，每帧管理和触发重建：
- **Layout Rebuild**：当布局改变时（如文本长度变化），重新计算 UI 元素的位置和大小；
  - **Graphic Rebuild**：当图形的视觉属性改变时（如颜色、纹理），重新生成图形的网格。

- Canvas下UI元素的合并规则

  同一个Canvas中，将**相同层级**(Depth)和**相同材质**的元素进行合并，相同层级指的是**覆盖层级**。

  - Canvas里，如果两个UI元素的网格存在覆盖，则认为它们是上下层关系，其中Hierarchy里靠后的UI元素，层级Depth + 1；
  - 计算Canvas下所有UI元素的层级数Depth；
  - 从第0层开始，将同层中材质相同的UI元素合批，再将第1、2、3........层的元素合并。
  
- 执行批处理

  Canvas会将其下所有UI元素的网格几何体合并，根据材质和渲染顺序等进行**批处理**，生成一个合并的大网格，尽可能减少 **Draw Call** 的数量。

#### 2.2) CanvasUpdateRegistry

`CanvasUpdateRegistry`是Unity UGUI系统的核心调度中心，它确保所有 UI 元素都能高效、有序地更新和渲染。

- 中央调度器

  作为一个单例类，统一管理所有需要更新的UI元素。

- 脏标记收集

  维护两个独立队列：布局重建队列(`m_LayoutRebuildQueue`)和图形重建队列(`m_GraphicRebuildQueue`)，收集被标记为“脏”的元素。

- 有序更新管线，在每帧渲染前，严格按照特定阶段执行重建。

  布局：Prelayout→Layout→PostLayout；

  渲染：PreRender→LatePreRender。

- 依赖关系处理

  在布局更新前对元素进行排序(按层级深度从父到子)，确保父容器先于子元素计算，解决尺寸依赖问题。

#### 2.3) CanvasRenderer

CanvasRenderer是每个绘制元素都必须有的组件，是画布与渲染的连接组件，通过CanvasRenderer才能把网格绘制到画布上。

- 接收网格

  `Graphic`组件(例如 `Image`)会调用 `OnPopulateMesh`方法来生成顶点数据，然后通过 `canvasRenderer.SetMesh()`将网格设置给 `CanvasRenderer`。

- 材质设置

  `Graphic`组件会计算最终使用的材质，通过 `canvasRenderer.SetMaterial()`进行设置。

- 提交数据

  `CanvasRenderer`将网格和材质信息提交给底层的`Canvas`系统。`Canvas`会负责对所有 UI 元素进行排序和合批，最终生成指令并提交给 GPU 渲染。

### 3) 工具类

#### VertexHelper

UI 与 Mesh 之间的数据桥梁，负责存储生成网格所需的所有顶点数据。

为了节省内存和CPU，它内部采用**List容器对象池**，将所有使用过的废弃的数据都存储在对象池的容器中，当需要时再拿旧的继续使用：

```c#
public class VertexHelper : IDisposable
{
    private List<Vector3> m_Positions;
    private List<Color32> m_Colors;
    private List<Vector4> m_Uv0S;
    private List<Vector4> m_Uv1S;
    private List<Vector4> m_Uv2S;
    private List<Vector4> m_Uv3S;
    private List<Vector3> m_Normals;
    private List<Vector4> m_Tangents;
    private List<int> m_Indices;
    // ........
    // ........
    private void InitializeListIfRequired() {
    	if (!m_ListsInitalized)
    	{
        	m_Positions = ListPool<Vector3>.Get();
        	m_Colors = ListPool<Color32>.Get();
        	m_Uv0S = ListPool<Vector4>.Get();
        	m_Uv1S = ListPool<Vector4>.Get();
        	m_Uv2S = ListPool<Vector4>.Get();
        	m_Uv3S = ListPool<Vector4>.Get();
        	m_Normals = ListPool<Vector3>.Get();
        	m_Tangents = ListPool<Vector4>.Get();
        	m_Indices = ListPool<int>.Get();
        	m_ListsInitalized = true;
    	}
	}
    // ........
}
```

使用示例：

```c#
Mesh m;
Color32 color32 = Color.red;
using (var vh = new VertexHelper())
{
    vh.AddVert(new Vector3(0, 0), color32, new Vector2(0f, 0f));
    vh.AddVert(new Vector3(0, 100), color32, new Vector2(0f, 1f));
    vh.AddVert(new Vector3(100, 100), color32, new Vector2(1f, 1f));
    vh.AddVert(new Vector3(100, 0), color32, new Vector2(1f, 0f));
    
    vh.AddTriangle(0, 1, 2);
    vh.AddTriangle(2, 3, 0);
    // 将构建好的顶点与索引，填充到Mesh中
    vh.FillMesh(m);
}
```

### 4) Graphic

`Graphic`类是所有可视化UI组件(如`Image`、`Text`、`RawImage`)的抽象基类。

#### 4.1) 作用

- 数据生成和传递

  `Graphic`的核心方法`OnPopulateMesh`负责生成定义UI形状的网格数据(顶点、颜色、UV等)，提交给同级的`CanvasRenderer`组件。

- 增量更新的标记系统

  UGUI性能优化的关键。当UI元素的属性发生变化时，`Graphic`不会立即重新计算网格，而是调用相应的`SetVerticesDirty()`, `SetMaterialDirty()`, 或 `SetLayoutDirty()`方法，将自己标记为“脏”状态，表示需要重建顶点、材质或布局。

- 统一重建

  所有被标记为“脏”的 `Graphic`元素都会在每帧渲染前，由 `CanvasUpdateRegistry`统一调用其 `Rebuild`方法。其会根据脏标记的类型，有选择地调用`UpdateGeometry`和`UpdateMaterial`来完成实际的数据更新。

#### 4.2) 源码分析

##### 4.2.1) 图元设置更新

```c#
//Graphics.cs
public virtual void SetAllDirty()
{
    if (m_SkipLayoutUpdate)
    {
        m_SkipLayoutUpdate = false;
    }
    else
    {
        // 内部会调用CanvasUpdateRegistry.TryRegisterCanvasElementForLayoutRebuild()
        SetLayoutDirty();
    }

    if (m_SkipMaterialUpdate)
    {
        m_SkipMaterialUpdate = false;
    }
    else
    {
        SetMaterialDirty();
    }

    SetVerticesDirty();
    SetRaycastDirty();
}
```

- `SetVerticesDirty`、`SetMaterialDirty`会调用`CanvasUpdateRegistry`的函数；

- 该函数将需要重构的图元加入到**RebuildQueue**(IndexedSet)中，等待下次重构。

- `CanvasUpdateRegistry`只负责收集网格，不负责渲染和合并。

- 注册待更新图元：

  ```c#
  // CanvasUpdateRegistry.cs
  
  public static void RegisterCanvasElementForGraphicRebuild(ICanvasElement element)
  {
      // 单例实例
      instance.InternalRegisterCanvasElementForGraphicRebuild(element);
  }
  
  private bool InternalRegisterCanvasElementForGraphicRebuild(ICanvasElement element)
  {
      if (m_PerformingGraphicUpdate)
      {
          Debug.LogError("xxxx");
          return false;
      }
  
      return m_GraphicRebuildQueue.AddUnique(element);
  }
  
  private readonly IndexedSet<ICanvasElement> m_GraphicRebuildQueue = 
      new IndexedSet<ICanvasElement>();
  ```

##### 4.2.2) 图元重建

```c#
// CanvasUpdateRegistry.cs
private void PerformUpdate()
{
    CleanInvalidItems();
    m_PerformingLayoutUpdate = true;
    m_LayoutRebuildQueue.Sort(s_SortLayoutFunction);
    for (int i = 0; i <= (int)CanvasUpdate.PostLayout; i++)
    {
        for (int j = 0; j < m_LayoutRebuildQueue.Count; j++)
        {
            var rebuild = m_LayoutRebuildQueue[j];
            if (ObjectValidForUpdate(rebuild))
                rebuild.Rebuild((CanvasUpdate)i);
        }
    }
    
    for (int i = 0; i < m_LayoutRebuildQueue.Count; ++i)
        m_LayoutRebuildQueue[i].LayoutComplete();
    
    m_LayoutRebuildQueue.Clear();
	m_PerformingLayoutUpdate = false;
    
    m_PerformingGraphicUpdate = true;
    for (var i = (int)CanvasUpdate.PreRender; i < (int)CanvasUpdate.MaxUpdateValue;
         i++)
    {
        for (var k = 0; k < m_GraphicRebuildQueue.Count; k++)
        {
            var element = m_GraphicRebuildQueue[k];
            if (ObjectValidForUpdate(element))
                element.Rebuild((CanvasUpdate)i);
        }
    }
    
    for (int i = 0; i < m_GraphicRebuildQueue.Count; ++i)
        m_GraphicRebuildQueue[i].GraphicUpdateComplete();
    
    m_GraphicRebuildQueue.Clear();
    m_PerformingGraphicUpdate = false;
}
```

- 让`m_LayoutRebuildQueue`中的元素进行重构，并调用`LayoutComplete`；
- 让`m_GraphicRebuildQueue`中的元素进行重构，并调用`GraphicUpdateComplete`。

##### 4.2.3) 图元网格构建

```c#
//Graphics.cs
private void DoMeshGeneration()
{
    if (rectTransform != null && rectTransform.rect.width >= 0 && rectTransform.rect.height >= 0)
        OnPopulateMesh(s_VertexHelper);
    else
        s_VertexHelper.Clear(); // clear the vertex helper so invalid graphics dont draw.

    var components = ListPool<Component>.Get();
    GetComponents(typeof(IMeshModifier), components);

    for (var i = 0; i < components.Count; i++)
        ((IMeshModifier)components[i]).ModifyMesh(s_VertexHelper);

    ListPool<Component>.Release(components);

    s_VertexHelper.FillMesh(workerMesh);
    canvasRenderer.SetMesh(workerMesh);
}
```

- 先调用**OnPopulateMesh**创建自己的网格；
- 然后调用所有需要修改网格的修改者(IMeshModifier)，也就是效果组件（描边等效果组件）进行修改；
- 最后放入CanvasRenderer。

## UI Batching合批

本小节参考自：[Unity3D UGUI系列之合批](https://blog.csdn.net/sinat_25415095/article/details/112388638)。

Canvas通过合并UI元素的网格，生成合适的渲染命令发送给Unity图形渲染流水线。

合批的目的是为了**一次性发送尽可能多的数据**，减少Draw Call的调用。

如果Draw Call过多，那么CPU就会把大量的时间花在准备数据和设置渲染状态上，造成性能问题。

### 1) 合批的前提

- 合批是以Canvas为单位，不包含子Canvas，子Canvas会是另外一个批次；
- 合批是在**子线程**中完成；
- **材质**和**纹理**需要相同。

### 2) Depth计算规则

当前所说的Depth和Image属性里的Depth是两个不同的东西。

- 名词解释

  - UI元素相交：UI元素的**网格**相交，而不是Rect区域：

    <img src="/pic_ugui_explain_1.png" alt="pic_ugui_explain_1" style="zoom:70%;" />

  - **LowerUI**：在Hierarchy面板中，CurrentUI之上的元素，如下所示：

    <img src="/pic_ugui_explain.png" alt="pic_ugui_explain" style="zoom:80%;" />

- 计算流程

  - 按照**Hierarchy**的顺序**从上往下**遍历UI元素；

  - 计算CurrentUI的Depth：

    ① 如果CurrentUI不渲染(透明度为0，长宽为0，diabled，active为false)，Depth = -1；

    ② 如果CurrentUI要渲染，且与其他UI元素的**网格不相交**，Depth = 0；

    ③ 若CurrentUI下面**只有一个**LowerUI与其相交；

    若CurrentUI、LowerUI材质和贴图相同，则可以合批，CurrentUI.Depth = LowerUI.Depth；

    若CurrentUI、LowerUI不能合批，CurrentUI.Depth = LowerUI.Depth + 1；

    ④ CurrentUI下有n个LowerUI的网格与其相交，则CurrentUI.Depth = max(Depth_1, Depth_2, Depth_3，…)；

- 相交计算优化

  源码中使用**分组计算包围盒矩形**的方法加快计算，即16个UI元素为一组计算Group网格Rect。

  检查是否与底层UI元素相交时，先计算是否与**底层Group相交**，如果相交再与Group中的元素做判定。

### 3) VisibleList排序流程

- 计算UI元素的Depth；

- 按Depth升序排序，剔除Depth = -1的元素，Depth是**最高优先级**；
- Depth相同的元素，按material ID升序排序；
- material ID相同的元素，按texture ID升序排序；
- texture ID相同的元素，再按Hierarchy中的顺序排序；
- 最终得到合批前的元素，即VisibleList。

### 4) 执行合批

判断VisiableList中相邻的元素是否有相同的材质和贴图，若满足要求，则进行合批。

注意，这里只考虑材质和纹理，不用再考虑Depth和其他元素了。

### 5) 示例

<img src="/pic_ui_batch_eg.png" alt="pic_ui_batch_eg" style="zoom:70%;" />

Depth计算：Image1.Depth = 0; Image2.Depth = 0; 由于Image2和Image3重叠，且纹理不同，所以有Image3.Depth = 1;

Depth相同的，按material排序：Image1 -> Image2 -> Image3；

material id相同的，再按texture id排序，由于Image2.textureId < Image1.textureId，所以有：Image2 -> Image1 -> Image3；

texture id相同的，再按hierarchy排序，由于Image1在Image3前，所以顺序不变：Image2 -> Image1 -> Image3

最后合批，batch 0: Image2；batch 1: Image1, Image3。

### 6) UI合批优化

- 使用图集，统一纹理；
- Text如果能用纹理代替，尽量用纹理；
- 避免频繁删除/增加UI元素，会引起Canvas的Rebuild；
- 尽量不要使用Mask，内部使用了模版缓冲，至少增加两个Draw Call；
- 动静分离，动态UI和静态UI分别使用不同的Canvas。

## UGUI事件系统

事件系统分为四个模块：事件数据模块、输入事件捕获模块、射线检测模块、事件逻辑处理模块。

### 1) 事件数据模块

该模块包含`PointerEventData`、`AxisEventData`、`BaseEventData`，分别为点位事件、滚轮事件和事件基类。

`PointerEventData`存储了大部分事件逻辑处理模块所需的数据，包括按下时位置、松开与按下的时间差、拖曳的位移差、点击的物体等。

事件数据模块的意义是为事件逻辑处理做好准备。

### 2) 输入事件捕获模块

该模块由BaseInputModule、PointerInputModule、StandaloneInputModule、TouchInputModule四个模块组成。

#### 2.1) 模块组成

- BaseInputModule是抽象基类；

- PointerInputModule继承自BaseInputModule，扩展了点位的输入逻辑，增加了输入类型和状态；

- StandaloneInputModule和TouchInputModule继承自PointerInputModule。

  StandaloneInputModule是对标准键盘、鼠标输入的拓展；

  TouchInputModule是对触控板输入的拓展。

#### 2.2) 鼠标输入简析

## 常用GUI组件

### Image和RawImage

`Image`是为精心设计的**用户界面**(UI)而生的，功能丰富且智能；

`RawImage`则更像一个灵活的“展示窗口”，**直接显示原始的纹理数据**，更为轻量和直接。

- 特性对比

  | 特性               | Image                                                        | RawImage                                                     |
  | :----------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
  | **核心资源类型**   | **Sprite**                                                   | **Texture**                                                  |
  | **主要功能与特性** | 支持四种显示模式：**Simple**（普通）、**Sliced**（九宫格切片）、**Tiled**（平铺）、**Filled**（填充，用于进度条等） | 功能简单，主要提供 **UV Rect** 属性，用于调整纹理的显示区域  |
  | **性能与合批**     | 可通过图集优化，**易于合批**                                 | 使用独立纹理，**打断合批**，增加 Draw Call                   |
  | **布局元素的支持** | 实现了 `ILayoutElement`接口，可以被自动布局组件管理和排列    | 不参与自动布局                                               |
  | **代码复杂程度**   | 源码复杂（近千行），提供丰富功能                             | 源码轻量（约120行），功能纯粹                                |
  | **典型应用场景**   | 按钮背景、自适应界面、进度条、血条等所有需要高质量、可伸缩的UI元素 | 显示游戏内渲染纹理（如小地图）、视频流、动态创建的纹理或需要动态切换纹理部分的帧动画 |

- 网格生成方式

  - Image网格生成复杂

    根据 `ImageType`，在 `OnPopulateMesh`方法中调用不同的生成函数。

    在 `Sliced`（九宫格）模式下，它会生成**36个顶点**来确保四个角不变形；

    在 `Tiled`（平铺）模式下，它会计算平铺数量并生成多个网格单元。

  - RawImage网格生成简单

    它的 `OnPopulateMesh`方法永远只生成一个由**4个顶点**构成的简单矩形网格。

- 事件检测的精细度

  - Image

    `Image`实现了`ICanvasRaycastFilter`接口，它可以将点击位置映射到`Sprite`纹理的像素上，检查该点的透明度是否超过设定的`eventAlphaThreshold`(事件Alpha阈值)。

    如果透明度低于阈值，点击事件会被“穿透”，这可以用来忽略对完全透明区域的点击。

  - RawImage不具备基于Alpha值的精细点击检测。

### Canvas Scaler组件

Canvas Scaler 是 Unity UGUI 系统中用于管理 UI 整体缩放和像素密度的核心组件，确保游戏界面在不同分辨率和屏幕比例下都能有恰当的显示效果。

| 模式                                        | 机制                                                         | 优点                                           | 缺点                                                         | 场景                                                         |
| ------------------------------------------- | ------------------------------------------------------------ | ---------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **Constant Pixel Size** (恒定像素大小)      | UI元素保持设计时的**像素值**不变                             | 像素精确，表现一致                             | UI布局不随分辨率变化，在不同分辨率的屏幕上可能失衡           | 固定分辨率的项目，如PC或主机游戏                             |
| **Scale With Screen Size** (随屏幕尺寸缩放) | 根据**参考分辨率**和**实际屏幕分辨率**的比例进行动态缩放     | UI元素整体大小和相对位置能保持稳定，适配能力强 | 极端屏幕比例可能需要额外处理（如适配刘海屏）                 | **移动端项目**的主流选择，适合绝大多数需要多分辨率适配的场景 |
| **Constant Physical Size** (恒定物理大小)   | 理论上使UI在不同设备上保持相同的**物理尺寸**（如英寸、厘米） | 理论上能实现跨设备的物理尺寸一致性。           | 依赖设备的DPI（每英寸像素数）数据，该数据常不准确，实际体验差 | 使用极少，多为实验性目的                                     |

## UI优化

### 1) 动静分离

动：移动、放大/缩小频率比较高的UI。静：界面上不会移动、旋转、缩放、更换贴图和颜色的UI。

#### 1.1) 原因

UGUI是用网格模型构建UI画面的，构建后都执行了合并网格的操作。合批能减少DrawCall，提升性能。

任何UI元素，发生了变化就需要重新合并网格。

**那些原本不需要重新构建的内容也得一并重构**，造成了不必要的CPU开销。

因此要将静止的UI分离，元素变化后，只合并那些会动的UI元素，节省了CPU的开销。

#### 1.2) 优化方法

以Canvas为节点拆分动与静，把会动的UI元素放入专门为它们准备的Canvas上。

### 2) 拆分过重的UI和UI预加载

- 拆分重UI

  如有些界面只有点击后才会显示，那么这些内容就可以视为二次显示的内容。可以考虑将其拆分出来设置成为一个预置体，当需要时它再加载。

- UI预加载

  在游戏开始前或在进入某个场景前预先加载一些UI，让实例化和初始化的消耗在游戏前**平均分摊到等待的时间线上**。

  - 游戏开始前**加载UI资源但不实例化**：把资源提前加载到内存中，使用时减少了IO的时间，把更多的时间花在实例化和初始化上。
  - 游戏开始前就加载、实例化、初始化UI，但对其进行隐藏。当需要它出现时，再显示出来。
  - Preload功能：Unity编辑器平台的设置里，可以把需要预加载的预置体加入列表中。Unity3D会在进入程序应用时将这些预置体进行预加载，CPU的消耗更多是在启动页面上。

### 3) UI图集Alpha分离

压缩UI图集能减少App包的大小，减少内存使用量。

压缩后I/O消耗减少了，内存的使用减少了，CPU的消耗自然就会少很多，能减缓卡顿现象。

但同时也会引起一些问题：显示的效果有时会不尽如人意，模糊、锯齿、线条等劣质的出现。

#### 3.1) 原因

使用压缩模式ECT或PVRTC时将，透明通道也一并压缩，导致渲染的扭曲。

因此需要把透明通道Alpha分离出来单独压缩。

这样既可以压缩图集，达到缩小内存的目的，图像显示又不会太失真。

#### 3.2) 优化方法

这里主要给出的是针对NGUI的方案，而UGUI由于是内部集成的，所以Alpha分离在Unity3D中已经完成了，因此这里不再细讲。

在NGUI中，Alpha分离的原理就是将ETC的方式将图片压缩为一张没有Alpha的图和一张有Alpha的图，这样分开压缩更有针对性，图片显示的质量也会提高很多。

### 4) 网格合批的优化

#### 4.1) 原因

在UGUI系统中，当元素需要改变颜色/透明度时，是通过改变**顶点的属性**来实现的。

改变顶点属性就会导致网格的重构，从而消耗大量CPU。

#### 4.2) 优化方法

建一个材质球，告诉GUI使用特殊的材质球进行渲染。

当动画对颜色和Alpha进行改动时，是基于自定义的材质球改变颜色和Alpha，这样UGUI就不需要重构网格了。

因为材质球是通过改变材质球属性实现颜色和Alpha变化的，这样就减少了UGUI重构网格的消耗。

#### 4.3) 实施方案

步骤：

- 下载UGUI的着色器；
- 建立材质球，在材质球里使用下载的UGUI着色器；
- 把材质球放入Image或RawImage的Material中，与Image或RawImage绑定。

伴随的风险：

- 因为启用了自定义的材质球，每个材质球都会单独增加一次drawcall。
- 当Alpha不是1的时候，会与原有的UGUI系统产生的半透明材质球形成混淆的渲染排序。

### 5) UI展示与关闭的优化

- 移出屏幕不会降低DrawCall

  它的GameObject是激活的，且CanvasRenderers是激活的；

  透明度大于0，且没有禁用Raycast Target。

  UGUI还是会把它当作“需要渲染”的对象处理，只是这些像素因为超出视口范围，被裁剪掉了。

- 使用SetActive(false)优化

  关闭时隐藏节点，界面打开时再次激活节点。

  能减少DrawCall，减少节点本身的CPU/GPU消耗，触发OnDisable()；

  但会触发Canvas重建。

### 6) 使用对象池

对象池本质是重复利用内存。

- 能减少内存碎片，内存连续的可能性变大，CPU缓存命中的概率也大了。
- 大块内存不会因为碎片问题而申请新内存，因此内存的使用量也会减少。
- 节省实例化对象时CPU的消耗。

### 7) UI贴图设置优化

Unity3D会重置全部贴图格式：无论是.jpg、.png格式等，Unity3D就会读取图片内容，然后重新生成一个自己格式的图。

因此在Unity3D中使用图片，其实不必关心用什么格式的图，只要你做好内容就可以。

- 是否需要Alpha通道。如果需要Alpha通道，则要把Alpha通道点开，否则最好关闭。
- 是否需要进行2次方大小的纠正。
- 去除读、写权限。这里常会默认勾选，导致**内存大增**，此选项会使贴图在内存中存储两份，内存会比不勾选时大1倍。
- 去除Mipmap。在2D界面上没有远近之分，所以不需要Mipmap，且Mipmap会导致内存和磁盘空间加大，会使得UI看起来模糊。
- 选择压缩方式。压缩方式的选择，主要是为了降低内存的消耗、加载时的消耗，降低CPU与GPU之间的带宽消耗，以及减少包的大小，在清晰度足够的情况下，我们可以针对性地选择一些压缩方式来优化内存和包体。

# 物理系统

## Tips && Tricks

### Force Mode总结

本小节参考自：[Know the difference between ForceModes](https://www.reddit.com/r/Unity3D/comments/psukm1/know_the_difference_between_forcemodes_a_little/)。

<img src="/pic_force_mode.png" alt="pic_force_mode" style="zoom:50%;" />

### Collider和Trigger

#### 1) 区别

| 特性维度 | Collider(碰撞器)                                             | Trigger(触发器)                                              |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 本质     | 定义物体物理形状的组件(如Box Collider)                       | Collider组件上的一个**属性**(Is Trigger)                     |
| 物理效果 | **有物理效果**。物体会被阻挡、弹开，符合物理规律             | **无物理效果**。物体会相互穿透                               |
| 回调函数 | OnCollisionEnter/Stay/Exit()                                 | OnTriggerEnter/Stay/Exit()                                   |
| 触发条件 | 双方都有Collider<br />至少一方有Rigidbody<br />**都不勾选Is Trigger** | 双方都有Collider<br />至少一方有Rigidbody<br />**至少一方勾选了Is Trigger** |
| 典型应用 | 墙壁阻挡、小球落地弹跳、汽车碰撞                             | 收集道具、触发机关/剧情、特定区域                            |

#### 2) 原理

- Collider：Unity的物理引擎会计算碰撞力，阻止它们相互穿透，并产生真实的物理交互。
- Trigger：物理引擎会忽略它们之间的阻挡计算，但会检测它们的几何体是否发生重叠。一旦重叠，就会调用`OnTrigger`系列函数。

#### 3) 事件触发和父子关系

- 原则

  拥有**刚体**的游戏对象会被Unity视为一个独立的**物理实体**，由它来统一管理和报告其自身及其所有子物体碰撞体上发生的物理事件。

- 物理实体的统一性

  刚体(Rigidbody)组件定义了一个物理对象的质心、质量、速度等属性。

  当父物体有刚体，而子物体只有碰撞体，没有刚体时，Unity会将这个子物体的碰撞体视为整个物理实体的一部分。

  因此，当有物体与子物体的触发器发生交互时，这个事件会传递到父物体的脚本中。

- 对比

  | 刚体(Rigidbody)配置            | 父物体有触发器脚本 | 子物体有触发器脚本 | 触发结果(碰撞发生在子物体时)                                 |
  | ------------------------------ | ------------------ | ------------------ | ------------------------------------------------------------ |
  | 父物体有刚体<br />子物体无刚体 | 有                 | 有                 | 父物体和子物体脚本都会收到通知                               |
  |                                | 有                 | 无                 | 仅父物体收到通知                                             |
  |                                | 无                 | 有                 | 仅子物体收到通知                                             |
  | 父物体无刚体<br />子物体有刚体 | 不限               | 有                 | 仅父物体收到通知                                             |
  |                                | 不限               | 无                 | 无通知                                                       |
  | 父物体和子物体都有刚体         | 有                 | 有                 | 父物体和子物体作为独立的物理实体，只有碰撞双方的脚本才会收到通知 |

# 动画系统

## Mecanim

Unity已不再对Animation系统进行维护，现主攻新的动画系统Mecanim。

- Mecanim系统的特点
  - 使用**多线程**进行计算，比Animation单线程性能更高。
  - Animator的功能更多，**Retargeting**功能可让不同的角色使用同一套动画资源。
  - Animator**状态机**可以让动画在不同的条件下轻松切换，**Layer**能让人物上下半身播放不同的动画。

## 角色动画

### Root Motion

#### 核心原理

**让动画控制角色在世界空间中的位移和旋转**，而不是通过脚本修改角色的位置。

这确保了动画(如走路、跑步)与角色在场景中的实际移动完全同步。

##### Body Transform && Root Transform

- Body Transform

  由Unity的Avatar系统根据骨骼结构计算出角色的**质心**(Center of Mass)。它代表了角色整体的位置和朝向，是AnimationClip中存储的**唯一的世界空间曲线数据**。

- Root Transform

  **Body Transform在水平面(XZ平面)上的投影**，在运行时每帧计算其变化。

  Root Motion最终应用给游戏对象(GameObject)的移动和旋转。

- 本质

  Unity通过分析动画每一帧Body Transform的变化，计算出 Root Transform 的位移和旋转**差值**，然后将这个差值应用到角色 GameObject 上，从而驱动角色移动。

##### RootMotion相关属性

在模型文件的导入设置中，点击顶部的 **“Animation”**页签，就可以看到各个AnimationClip的与Root Motion相关的属性。

###### 1) 属性

- Root Transform Rotation：这个设置控制角色整体的旋转。
- Root Transform Position(Y)：控制角色在垂直方向（上下）的移动。
- Root Transform Position(XZ)：控制角色在水平面上的移动。

###### 2) Bake Into Pose

本质上是一个“开关”，它决定了动画中所包含的位移和旋转信息由谁“消化”。

- 勾选时

  动画中的变换（位移/旋转）被视为 **“姿势”** (Pose) 的一部分。

  这意味着这些变化只会体现在角色骨骼动画本身，而**不会**传递给游戏对象的Transform组件。

  因此不会驱动游戏对象在实际游戏世界中移动，你可以理解为动画在原地表演。

- 取消勾选

  这些变换则被视作 **“根运动”** (Root Motion)。

  Unity会计算每一帧的变换差值（Delta），并将其**应用**到角色游戏对象的Transform组件上，从而驱动角色在场景中移动。

###### 3) 实例说明

实例参考自：[根运动 (Root Motion) – 工作原理](https://blog.csdn.net/MyArrow/article/details/45505085)。

- 勾选`Bake into Pose`，不勾选`Apply Root Motion`

  勾选`Bake into Pose`后，变换属于Body Transform，所以即使这里未勾选`Apply Root Motion`，但是动画依然会在场景中体现，人物会按照动画的路径行走。

  但是观察Inspector中模型的Transform，其值一直不变。

  此时，如果开始一个新的动画，模型会瞬间回到起始位置（新的动画开始时候，模型处于行走动画开始时的位置）。

- 勾选`Bake into Pose`，并勾选`Apply Root Motion`

  这里跟上面的情况唯一不同的是，动画结束后，在开始新的动画之前，Transform变化会应用到模型。

- 不勾选`Bake into Pose`，勾选`Apply Root Motion`

  不勾选`Bake into Pose`后，变换属于Root Transform，且勾选了`Apply Root Motion`，变换会随动画进度实时应用到模型。

  新的动画开始时候，模型处于上个动画结束时的位置。

- 不勾选`Bake into Pose`，不勾选`Apply Root Motion`

  这里变换还是属于Root Transform，但是因为没有勾选`Apply Root Motion`，所以变换将不被应用。

  因此模型将一直在本地不动。

### 动画过渡

#### Has Exit Time属性

决定动画状态何时开始转换的关键属性。

- 标准化时间

  `Exit Time`是一个**标准化时间**(Normalized Time)，通常理解为动画播放的百分比。

  例如，设置为 `0.75`表示在动画播放到 **75%** 的那一帧，时间条件变为“真”。

- 与`Conditions`的协同工作

  如果同时设置了 `Has Exit Time`和转换条件(Conditions)，Unity 会先等待动画播放到 `Exit Time`，**然后**才去判断其他条件是否全部满足。只有时间条件和其他逻辑条件都为真，才会发生转换。

- 对于循环动画

  对于循环播放的动画(如待机 idle)，如果`Exit Time`小于1(如 0.9)，那么**每一轮循环**播放到 90% 时都会检查转换条件。

  如果设置成大于1(如 3.5)，则意味着在播放完**3个完整循环+半个循环**后才会检查一次转换条件。
  
- 特征对比

  | 特性     | 勾选Has Exit Time                                           | 不勾选                                                 |
  | -------- | ----------------------------------------------------------- | ------------------------------------------------------ |
  | 转换机制 | 必须等待当前动画播放到指定的 **Exit Time** 才可能触发转换。 | 只要设置的 **Conditions** 条件满足，**立即**触发转换。 |
  | 响应速度 | 延迟响应，即使条件早已满足，也需等待时间点。                | 瞬时响应，确保动画切换迅速。                           |
  | 典型应用 | 不需要被打断的完整动画，如换弹、喝药水。                    | 需要快速切换的动画，如行走转奔跑、受击打断攻击。       |

#### Fixed Duration

`Fixed Duration`决定两个动画之间过渡时间和计算方式。

- 勾选`Fixed Duration`

  此时`Transition Duration`是一个以秒为单位的绝对值。无论源动画有多长，过渡的混合过程都会严格按照设定的时间执行。

- 未勾选

  此时`Transition Duration`是一个相对于源动画长度的比例值(标准化时间)。

  例如，如果**源动画**长度为5秒，`Transition Duration`设为0.2，那么实际的过渡时间就是**5秒 * 0.2 = 1秒**。

- 动画过渡混合规则

  在整个动画过渡的**混合期**内，动画A和动画B是**同时播放、权重此消彼长**的。

  过渡开始时A的权重为100%(完全播放A)，过渡结束时B的权重为100%(完全播放B)。

#### Transition Offset

- 作用：定义动画过渡中，目标动画开始播放的时间点；
- 取值：**标准化时间（百分比）**。例如，0.5 表示从目标动画的中间(50%进度)开始播放；
- 只影响目标动画，对源动画没有任何影响；
- 与`Exit Time`配合：可精确控制动画何时退出，何时进入。

### Animation Rigging

#### 核心组件

Animation Rigging的核心组件按如下组织：

<img src="/pic_animation_rig.png" alt="pic_animation_rig" style="zoom:60%;" />

- Animator

  Animation Rigging是依赖于unity动画系统的，因此需要为root gameobject添加Animator组件。

- Rig Builder

  Rig Builder和Animator一样，需要添加到**root** gameobject上。

  Rig Builder允许一个或多个Rig修改动画，并将Rigs按layer组织。

  每个layer可以有不同的权重，可根据不同场景灵活组合所有Rig的最终效果：

  <img src="/pic_rig_layer.png" alt="pic_rig_layer" style="zoom:60%;" />

- Rig组件

  - Rig需被添加到Rig Builder的layer中。Rig需为root节点的子节点，且独立于骨骼节点之外。
  - Rig组件的主要作用就是**收集和管理其本地层级下的所有Constraint**。
  - 可修改Rig的权重，来控制其对最终姿势的影响。

- Constraints

  - 各种类型的Constraints规定了Rig要执行的操作。
  
  - Rig会为其下所有的*Constraint*会创建持有*IAnimationJobs*的**有序队列**。
  
    在Animator计算完骨骼姿态后，这个任务队列会被应用到动画上，形成最终姿态。
  
  - Rig通过*GetComponentsInChildren*来寻找其层级下的*Constraint*。因此*Constraint*排列的顺序会决定*IAnimationJobs*作用于骨骼姿态的顺序。它是按照**深度优先**来遍历的，因此有：
  
    <img src="/pic_eval_order.png" alt="pic_eval_order" style="zoom:70%;" />

## 动态合批 && 静态合批

- 子网格

  模型中可以包含很多网格，一个模型可以由多个网格组成。

  在Unity3D中，一个网格可以由多个子网格(*SubMesh*)组成，一个子网格需要匹配一个*材质球*。

  可以将一个网格拆分成多个子网格，也可将多个子网格合并为一个网格。

- draw call过多

  如果drawcall过多，CPU会忙于发送状态数据给GPU，此时GPU大多数时间处于等待状态，这将导致画面帧率下降及强烈的卡顿感。

- 合批

  许多时候，场景中不同的模型，使用相同的材质，若不优化，就容易产生很多draw call。

  这时候就需要合批发挥作用。

### 动态合批Dynamic Batch

- 动态合批执行的条件

  - 若顶点属性只有顶点坐标、UV0，则能处理900个顶点；

    若顶点属性有顶点坐标、法线和单独的UV，则能处理300个顶点；

    若顶点属性有顶点坐标、法线、UV0，UV1和切线，能则能处理180个顶点；

  - 物体间的缩放比例必须一致；

  - 使用相同的材质球；

  - 多管线着色器会中断合批。

- 动态合批的CPU开销

  - 动态合批执行的条件相当严苛；
  - 动态合批在CPU上执行；
  - 动态合批需要把顶点转换到世界坐标，也会带来开销。

综上，如果draw call减少的优化抵不过动态合批的开销，那么就是负优化。

### 静态合批Static Batch

静态批处理允许引擎在**离线**的情况下，进行模型合并的处理，以减少draw call。

其将所有持有静态标记的物体放入世界空间，以材质球为分类标准，将它们分别合并，构成一个大的顶点集合和索引缓存。

- 静态合批执行条件
  - **无论模型有多大**，只要使用**同一个材质球**，都会被静态批处理；
  - 设置**静态标记**，且**认定物体是静态**的：不能移动、旋转、缩放；
- 静态合批的开销
  - 离线执行，不需要实时转换顶点坐标，不消耗运行时的CPU；
  - 增加额外的内存，存储合并的模型；

### 手动合并模型

# 性能

## 内存&&垃圾回收GC

### Unity3D的内存运作

#### 1) Mono

- 跨平台

  Unity3D通过Mono来跨平台**解析并运行**C#代码。

  只需一份C#代码(IL，中间语言)，各大平台的Mono自己实现各自系统的执行接口。

  如，在Android系统上App的lib目录下存在的**libmono.so**文件，就是Mono在Android系统上的实现。

- 内存管理

  - C#代码通过Mono这个虚拟机解析并执行，需要用到的内存自然也由Mono来进行分配和管理了。

  - Mono的堆内存大小在运行时是**只会增加不会减少**。

    可以将Mono的堆内存理解为一个内存池，每次C#向Mono内存申请的堆内存都会在池内进行分配，释放的时候也是归还给池里去，而不是归还给操作系统。

- 内存划分

  - 栈内存：函数和临时变量；
  - Mono堆内存：程序需要使用的内存块，例如静态实例以及这些实例中的变量和数组、类定义数据、虚函数表等。
  - Native堆内存：Unity3D的**资源**是通过Unity3D的C++层读取的，即分配在Native堆内存上，其与Mono堆内存是分开来管理的。

#### 2) IL2CPP

Unity3D将C#翻译成IL中间语言后再翻译成C++。

翻译成C++语言后**内存依然托管**，只是这次由C++编写VM（虚拟机）来接管内存。

不过这个VM**只实现内存托管**，它并不会解析和执行任何代码，它只是个管理器。

#### 3) Mono和IL2CPP的区别

| 区别     | Mono                             | IL2CPP                               |
| -------- | -------------------------------- | ------------------------------------ |
| 编译     | C#编译为IL                       | C#编译为IL，再编译为C++              |
| 运作方式 | 解析、执行代码，托管内存         | 托管内存                             |
| 平台实现 | 不同平台需要实现各自的Mono虚拟机 | 统一的C++虚拟机，适配各平台的C++接口 |
| 内存使用 | 堆内存只会增加，不会减少         | 堆内存只会增加，不会减少             |

### 垃圾回收算法

C#是基于垃圾回收(GarbageCollection, GC)机制的**内存托管语言**。

GC通过一定的算法找到“垃圾”，并且自动将“垃圾”占用的内存回收，但每次运行垃圾回收都会**消耗一定量的CPU**。

找“垃圾”的算法有两种，一种是**引用计数**的方式，另一种是**跟踪收集**的方式。

- 引用计数

  当被分配的内存块地址赋值给引用时，增加引用计数1；相反当引用清除内存块地址时，减少引用计数1。

  引用计数变为0就表明没有人再需要此内存块了，可以把内存块归还给系统，此时这个内存块就是垃圾回收机制要找的“垃圾”。

- 跟踪收集

  遍历引用内存块地址的根变量，以及相关联的变量，对内存资源没有引用的内存块进行标记，在回收时还给系统。

- 难以解决的问题

  - 对象之间循环引用
  - 对象环状的引用链

### 如何优化GC

整体思路

- 减少GC的运行次数。
- 减少单次GC的运行时间。
- 将GC的运行时间延迟，避免在关键时刻触发，比如可以在加载场景的时候调用GC。

方法

- 缓存变量，达到重复利用的目的，减少不必要的内存垃圾。

- 减少逻辑调用。堆内存分配，最坏的情况就是在反复调用的函数中进行堆内存分配，例如，在每帧都调用的Update()和LateUpdate()函数里，如果有内存分配，就会放大内存垃圾。

- 进行链表分配时清除链表，而不是不停地生成新的链表。

  ```c#
  private List myList = new List();
  void Update()
  {
  	myList.Clear();
  	DoSomething();
  }
  ```

- 使用对象池。

- 协程。

  - 调用*StartCoroutine()*会产生少量的内存垃圾，因为Unity3D会生成实体来管理协程。

  - *yield return 0*会引起**装箱**，这种情况，应使用*yield return null*。

  - ```c#
    // 每个while循环都会产生实例, 引起垃圾
    while(!isComplete)
    {
        yield return new WaitForSeconds(1f);
    }
    
    // 优化方案
    WaitForSeconds delay = new WaiForSeconds(1f);
    while(!isComplete)
    {
        yield return delay;
    }
    ```

- 优化字符串。C#中的字符串是不可变更的，每次在对字符串进行操作的时，C#会新建一个字符串来存储新的字符串，使得旧的字符串被废弃，造成内存垃圾。

  - 字符串缓存；
  - 使用*StringBuilderClass()*函数，它能减少字符串中间状态的分配，但不能免除字符串内存分配。

