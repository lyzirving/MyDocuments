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

## C#泛型

### 原理

- 编译时
  - 当定义泛型类、方法、接口等时，编译器会进行类型检查，确保代码的类型安全。
  - 编译器会将泛型代码编译为**IL（中间语言）代码**，并在元数据中记录**泛型参数**。
- 运行时(JIT编译时)
  - 当**第一次**使用某个具体类型实例化泛型时，JIT编译器会生成该特定类型的本地代码。这个过程称为“**具现化**”（instantiation）。
  - 对于引用类型，引用类型变量只是指针，操作方式相同，且共享相同的**内存布局**（引用的大小相同），因此可以共享代码。
  - 对于值类型，每种值类型都会生成一份独立的本地代码，因为值类型的大小和内存布局可能不同。

### IL2CPP下的泛型

由于IL2CPP是一个提前(AOT)编译的解决方案，它先将C#代码编译为C++代码，然后再编译为本地机器码。

因此，**泛型的具现化发生在编译阶段**，而不是运行时。

#### 1) 具现化流程

- 在C#代码编译为IL代码时，泛型信息被保留。
- 当使用IL2CPP时，IL代码被转换为C++代码。在这个过程中，泛型类型和方法会被具现化（实例化）为具体的类型。

由于是AOT编译，**所以必须知道所有可能被使用的泛型实例化类型**，并在编译时生成对应的代码。

实例化分析大致通过下述步骤完成：

- 分析IL代码，找出所有泛型类型。
- 找出所有在代码中显式使用的泛型实例化(例如，List\<int>，List\<string>等)。
- 为这些实例化生成具体的C++代码。

#### 2) IL2CPP泛型使用限制

下述情况，可能会引起IL2CPP在编译时，不会发现泛型类和方法，可能引起运行时异常：

- 运行时通过反射创建新的泛型实例。

  ```c#
  // case1
  Activator.CreateInstance(typeof(SomeGenericType<>).MakeGenericType(someType));
  // case2
  Type genericType = typeof(List<>);
  Type specificType = genericType.MakeGenericType(typeof(MyRuntimeType));
  ```

- 调用泛型实例的静态方法。

  ```c#
  typeof(SomeGenericType<>).MakeGenericType(someType).GetMethod("AMethod")
      .Invoke(null, null);
  ```

- 调用静态泛型方法。

  ```c#
  typeof(SomeType).GetMethod("GenericMethod").MakeGenericMethod(someType)
      .Invoke(null, null);
  ```

- 某些在编译时无法推断的泛型虚函数调用。

- 深层嵌套的泛型值类型调用，例如 `Struct<Struct<Struct<...<Struct<int>>>>>`。

为了支持上述这些情况，IL2CPP会生成适用于**任何类型参数的泛型代码**。

但这类代码性能较低，因为它无法确定类型的大小，也无法区分是引用类型还是值类型。

如果需要确保生成性能更优的泛型方法，可采取以下措施：

- 如果泛型参数**始终为引用类型**，添加 `where : class`约束。

  IL2CPP会通过引用类型共享生成回退方法，不会造成性能损失。

- 如果泛型参数**始终为值类型**，添加 `where : struct`约束。

  这会启用部分优化，但由于值类型大小可能不同，代码性能仍会较低。

- 创建一个名为 `UsedOnlyForAOTCodeGeneration`的方法，并在其中添加希望 IL2CPP 生成的泛型类型和方法的引用。该方法无需被调用。以下示例确保生成 `GenericType<MyStruct>`的专用化实现：

  ```c#
  public void UsedOnlyForAOTCodeGeneration()
  {
      // Ensure that IL2CPP will create code for MyGenericStruct
      // using MyStruct as an argument.
      new GenericType<MyStruct>();
  
      // Ensure that IL2CPP will create code for SomeType.GenericMethod
      // using MyStruct as an argument.
      new SomeType().GenericMethod<MyStruct>();
  
      public void OnMessage<T>(T value) 
      {
          Debug.LogFormat("Message value: {0}", value);
      }
  
      // Include an exception so we can be sure to know if this
      // method is ever called.
      throw new InvalidOperationException(
          "This method is used for AOT code generation only. " +
          "Do not call it at runtime.");
  }
  ```

### 优点&&缺点

- 优点

  - 避免装箱/拆箱。例如，使用`List<int>`比使用`ArrayList`(存储object)性能更好。

  - 类型安全，在编译时进行类型检查，减少强制类型转换。

  - 代码共享。对于引用类型，在JIT时可共享一份本地代码。

  - 小的泛型方法可被JIT内联

    ```c#
    public T Add<T>(T a, T b) where T : struct
    {
        return a + b;  // 需要运算符约束
    }
    
    // 优化后等价于：
    int result = a + b;  // 直接内联代码
    ```

- 缺点

  - 代码膨胀。对于值类型，会造成类型膨胀。
  - 首次调用开销。当**第一次**使用某个具体类型实例化泛型时，JIT编译器需要生成对应的本地代码，这会带来一定的启动开销。

- C#泛型和C++模板的比较

  - C++模板是编译时多态，它在编译时生成所有类型特化的代码。这可能导致代码膨胀，但运行效率高，且可以进行更复杂的元编程。
  - C#泛型是运行时多态，在JIT编译时生成代码，在类型安全性和性能之间取得了平衡，但不如C++模板灵活。

## CRTP泛化

### C++中的CRTP

CRTP(Curiously Recurring Template Pattern，奇异递归模板模式)是一种**C++**模板元编程技术。

核心思想：**一个基类将其派生类作为泛型参数**，从而实现编译时的多态。

```c++
template <typename Derived>
class Base {    
    void Interface() {
        // 在基类中可以使用Derived类型
        static_cast<Derived*>(this)->Implementation();
    }
};

class Derived : public Base<Derived> {
    // 派生类继承自以自身为模板参数的基类
};
```

- 原理：静态多态

  CRTP的核心原理是静态多态（编译时多态）：基类通过模板参数知道派生类的类型，因此可以在**编译时**进行静态转换，调用派生类的方法，而无需虚函数（动态多态）的开销。

- 使用场景

  - **静态多态**：当需要多态行为但又不想使用虚函数（避免运行时开销）时。
  - **代码复用**：在基类中实现通用功能，而派生类提供特定细节。例如，实现单例模式、对象计数等。
  - **实现“编译时多态”的接口**：例如，实现一个通用的Clone函数，每个派生类都需要实现Clone，但基类可以提供通用的克隆框架。

### C#中CRTP

```c#
// CRTP基类
public abstract class BaseClass<T> where T : BaseClass<T>
{
    // 基类中可以调用派生类的方法
    public void CommonOperation()
    {
        T derived = this as T;  // 安全的类型转换
        derived.DerivedSpecificOperation();
    }
    
    protected abstract void DerivedSpecificOperation();
}

// 派生类将自己作为泛型参数
public class DerivedClass : BaseClass<DerivedClass>
{
    protected override void DerivedSpecificOperation()
    {
        Debug.Log("Derived class specific operation");
    }
}
```

- C#的CRPT和C++的区别

  - C++的CRTP是编译时多态，没有运行时开销。
  - C#的泛型在运行时实例化，且有类型检查的开销。
  - C#的版本仍然使用虚方法（override），因此实际上还是动态多态，只是基类可以通过泛型参数知道派生类的类型。

- Unity中CRPT的应用

  ```c#
  // 一个简单的CRTP风格的单例模式基类
  public class SingletonBehaviour<T> : MonoBehaviour where T : SingletonBehaviour<T>
  {
  	// ......
  }
  ```

## 老版本foreach引起内存分配

- List简化源码

  ```c#
  public class List<T> : IEnumerable<T>
  {
      private T[] _items;
      private int _size;
      
      // 获取枚举器
      public Enumerator GetEnumerator()
      {
          return new Enumerator(this);
      }
      
      // 值类型的枚举器（结构体）
      public struct Enumerator : IEnumerator<T>
      {
          private List<T> _list;
          private int _index;
          private T _current;
          
          internal Enumerator(List<T> list)
          {
              _list = list;
              _index = 0;
              _current = default(T);
          }
          
          public bool MoveNext()
          {
              List<T> localList = _list;
              if ((uint)_index < (uint)localList._size)
              {
                  _current = localList._items[_index];
                  _index++;
                  return true;
              }
              return false;
          }
          
          public T Current
          {
              get { return _current; }
          }
          
          public void Dispose()
          {
              // 清理资源
          }
      }
  }
  ```

- 示例

  ```c#
  // 原始代码
  List<int> list = new List<int>{1, 2, 3};
  foreach (var item in list)
  {
      // do something.
  }
  ```

  新版本中，编译后优化如下：

  ```c#
  List<int> list = new List<int>{1, 2, 3};
  // // 编译器直接使用结构体枚举器，避免装箱
  List<int>.Enumerator enumerator = list.GetEnumerator();
  try
  {
      while (enumerator.MoveNext()) 
      {
          int item = enumerator.Current;
          // do something
      }
  } 
  finally
  {
      enumerator.Dispose();  // 确保资源释放
  }
  ```

  老版本中，要执行装箱操作，引起内存分配：

  ```c#
  List<int>.Enumerator enumerator = list.GetEnumerator();
  // 在老版本的Mono中，这个结构体会被装箱到堆上
  IEnumerator<int> boxedEnumerator = (IEnumerator<int>)enumerator; 
  ```

- 新版本的Unity进行了优化，避免了装箱操作，从而减少了内存的分配。

## Burst编译器优化异步编程

本小节参考自：[Manual / Burst compiler](https://docs.unity3d.com/Packages/com.unity.burst@1.8/manual/index.html)。

### 概述

- 核心作用

  Burst**核心作用**是将特定部分的C#代码编译成运行效率极高的**原生机器码**，以释放CPU的全部性能潜力。

- 受限的支持

  Burst只支持受限的C#子集：`HPC#(High Performance C#)`，其使用**LLVM**将 .NET的**中间语言(IL)**转换为针对目标CPU架构进行过性能优化的代码。

- 核心应用场景 && [BurstCompile]属性

  Burst支持Unity的`JobSystem`和HPC#的静态方法。

  需显示使用**[BurstCompile]**标记，Burst才会对其优化。

### HPC#的限制 && 语言支持

HPC#能支持C#中大部分的特性，具体可参考：[Supported C# features in HPC#](https://docs.unity3d.com/Packages/com.unity.burst@1.8/manual/csharp-hpc-overview.html)。

#### 1 HPC#中的限制

- 禁用基于 `try/catch`的异常捕获

  在HPC#中，可以使用 `throw`抛出异常，但**无法使用 `try-catch`块来捕获和处理异常**。

- 严格限制静态字段的写入

  大多数情况下，静态字段必须是**read-only**的。

  不能随意向静态字段赋值，唯一的例外是通过Unity提供的**Shared Static** 机制。

  这项限制是为了保证多线程(尤其是在 Job System 并行环境下)数据访问的安全性与确定性，防止发生不可预见的竞争条件。

- 禁止使用**托管类型**(managed code)及其方法

  这是 HPC# 最核心的限制之一。它**完全不允许操作任何托管对象**，其中最典型的例子就是 `string`类型及其所有方法。

  因为托管对象存在于由垃圾回收器（GC）管理的内存堆中，其分配、访问和回收都会带来性能开销和非确定性。

  HPC# 要求所有数据都必须使用值类型(`struct`)或基于非托管内存的原生容器(如`NativeArray`)。

#### 2 支持异常表达式

Burst 支持 `throw`表达式，但其行为和处理方式受到严格限制。

- **编辑器模式**：异常可被捕获和记录（在控制台查看），适用于调试。

- **发布版本**：任何异常都会导致程序**立即终止**，这是不可恢复的致命错误。

- Burst编译器的警告

  为了防止开发者误用异常进行流程控制，Burst编译器会**主动发出警告**。只有明确标记了 `[Conditional("ENABLE_UNITY_COLLECTIONS_CHECKS")]`属性的方法，其中抛出异常才不会引发警告。

#### 3 支持Foreach and While

Burst编译器支持常用的`foreach`和`while`循环结构，但存在一个重要约束：**无法对通过泛型类型参数约束(如`where T : IEnumerable<U>`)传入的集合进行迭代**。

原因：Burst 需要在编译时**确切知道集合的具体类型**以进行深度优化，而泛型参数在编译时无法完全确定其底层实现细节。

```c#
// 类型编译时明确, 支持
public static void IterateThroughConcreteCollection(NativeArray<int> list)
{
    foreach (var element in list)
    {
        // Do something
    }
}

// 类型编译时不明确, 不支持
public static void IterateThroughGenericCollection<S>(S list) where S : struct, IEnumerable<int>
{
    foreach (var element in list)
    {
        // Do something
    }
}
```

#### 4 支持静态只读字段 && 静态构造函数 && 语言支持

- **编译时求值** && **只读** 

  Burst会在**编译阶段**就尝试计算并确定所有静态字段和静态构造函数的结果，而不是将这些初始化工作延迟到运行时。这能最大化地消除运行时开销。

  静态字段必须是 **`readonly`** 的。这防止了字段在初始化后被修改，从而保证了编译时求值结果的有效性。

- **全有或全无**的评估策略 && **降级机制**

  对于一个结构体，其所有静态成员的评估是一个整体。**任何一个静态成员评估失败，会导致整个结构体的静态初始化都无法在编译时完成**。这确保了评估结果的完整性和一致性。

  如果无法在编译时完成求值，Burst会将这些初始化代码**打包成一个运行时函数**，在程序开始运行时执行一次。但这要求相关代码本身是符合Burst规范的。

- 静态初始化中的特殊许可 && 有限的数组支持

  作为一个特例，Burst允许初始化**静态只读数组**，但前提是初始化的数据来源必须是编译时可知的，如下：

  ```c#
  static readonly int[] MyArray0 = { 1, 2, 3, .. };
  static readonly int[] MyArray1 = new int[10];
  ```

- Burst 明确禁止调用**外部函数和函数指针**，确保代码的完全的可预测性和可优化性。

- 在一些特定场景下，支持使用string，具体参考：[String support](https://docs.unity3d.com/Packages/com.unity.burst@1.8/manual/csharp-string-support.html)。

#### 5 调用Burst编译的代码

##### 1) 托管代码中直接调用

Burst编译的代码**可以直接从普通的托管C#代码中调用**，无需通过Job System或复杂的函数指针机制。

这极大地简化了在现有代码库中集成性能关键函数的过程。

- 调用约束：**被调用的方法及其声明类型都不得是泛型**。
- 参数传递：为了获得最佳性能并避免不必要的数据拷贝，Burst编译的方法在接收或返回值类型(如`float4`这样的`struct`)时，应使用引用传递(`in`, `out`, `ref`)。

```c#
[BurstCompile]
public static class MyBurstUtilityClass
{
    [BurstCompile]
    public static void BurstCompiled_MultiplyAdd(in float4 mula, in float4 mulb, 
                                                 in float4 add, out float4 result)
    {
        result = mula * mulb + add;
    }
}
// 调用
public class MyMonoBehaviour : MonoBehaviour
{
    void Start()
    {
        var mula = new float4(1, 2, 3, 4);
        var mulb = new float4(-1,1,-1,1);
        var add = new float4(99,0,0,0);
        MyBurstUtilityClass.BurstCompiled_MultiplyAdd(mula, mulb, add, out var result);
    }
}
```

##### 2) 函数指针机制

Burst之所以能在托管代码中被调用，因为：

- 利用 **IL后处理** ，在编译时自动将标记了`[BurstCompile]`的方法包装成**函数指针**。
- 开发者无需手动处理复杂的指针操作，Burst在幕后自动完成了**托管代码与高效原生代码之间的连接**。

##### 3) DisableDirectCall

DisableDirectCall默认为false，Burst 会为标记的方法生成"双重接口"——既可以通过普通 C# 直接调用，也可以通过函数指针调用。

DisableDirectCall设置为`true`，则**强制**只能通过函数指针调用。

`DisableDirectCall = true`的使用场景：

- 在某些架构设计中，某些方法**本意就只应在特定上下文(如 Job 内部)被调用**，而不应该从任意地方调用。

- 确保 AOT 编译兼容性。

  在某些平台，Unity 使用**提前编译(AOT)**。若方法在AOT编译时没被调用，AOT 编译器可能会将其优化掉。

  但你可能仍然想在运行时通过函数指针动态调用它。禁用直接调用可以确保该方法在 AOT 阶段被正确处理。

- 避免委托开销

  当通过函数指针调用时，Burst 可以生成更优化的代码。在极端性能敏感的场景，开发者希望确保调用通过函数指针进行，以获得最佳性能。

  ```c#
  // 场景：极端性能要求的数学库
  public class MathLibrary
  {
      [BurstCompile(DisableDirectCall = true)]
      public static void MatrixMultiply4x4(in float4x4 a, in float4x4 b, 
                                           out float4x4 result)
      {
          // 这个操作被频繁调用，必须通过函数指针优化
      }    
      // 对外提供预计算的函数指针
      public static readonly FunctionPointer<MatrixMultiplyDelegate> MultiplyPtr = 
          new FunctionPointer<MatrixMultiplyDelegate>(MatrixMultiply4x4);
  }
  ```

#### 6 函数指针

##### 1) 简介

在Burst编译器上下文中，**函数指针(function pointers)** 是一种**特殊的委托机制**。

允许让Burst编译的本地代码作为可调用指针传递给C#托管代码或其他本地代码，实现高性能的跨边界函数调用。

```c#
[BurstCompile]
public class FunctionPointerExample
{
    delegate float MathOperationDelegate(float a, float b);
    
    [BurstCompile]
    static float Multiply(float a, float b) => a * b;
    
    [BurstCompile]
    static float Add(float a, float b) => a + b;
    
    public void RunExample()
    {
        var multiplyPtr = BurstCompiler
            .CompileFunctionPointer<MathOperationDelegate>(Multiply);
        var addPtr = BurstCompiler
            .CompileFunctionPointer<MathOperationDelegate>(Add);
        
        //通过指针调用
        float result1 = multiplyPtr.Invoke(3.0f, 4.0f);  // 返回 12.0f
        float result2 = addPtr.Invoke(3.0f, 4.0f);       // 返回 7.0f
    }
}
```

| 特性     | Burst函数指针                                                | 普通C#委托   |
| -------- | ------------------------------------------------------------ | ------------ |
| 编译     | AOT                                                          | JIT          |
| 性能     | 接近原生C++                                                  | 托管代码     |
| 优化     | 最高级别Burst优化                                            | 有限优化     |
| 类型限制 | 无泛型：必须在编译阶段完全确定所有类型信息<br />泛型委托和开放式泛型方法会引入类型不确定性。 | 灵活支持泛型 |

##### 2) 函数指针与IL2CPP

Burst编译器中，函数指针与IL2CPP互操作时，需在委托上使用下述属性：

- System.Runtime.InteropServices.UnmanagedFunctionPointerAttribute：CLR中定义**非托管函数指针**的元数据属性。
- 调用约定设置为CallingConvention.Cdecl：C语言标准的调用约定。

上述属性，即使开发者不显示添加，Burst也会添加此属性：

```c#
// 1) 手动添加的情况
[UnmanagedFunctionPointer(CallingConvention.Cdecl)]
delegate int MyDelegate(int x, int y);

// 2) 自动添加的情况
// 开发者看到的代码
var ptr = BurstCompiler.CompileFunctionPointer<MyDelegate>(MyFunction);
// Burst实际生成的代码
[UnmanagedFunctionPointer(CallingConvention.Cdecl)]
internal delegate int MyDelegate_Internal(int x, int y);
```

- 编译管线
  - C#源代码编译为IL
  - Burst编译器处理标记[BurstCompile]的方法
  - 为函数指针委托自动添加UnmanagedFunctionPointerAttribute
  - IL2CPP读取属性，生成对应的C++函数指针定义
  - 平台编译器生成最终原生代码
  - 运行时：基于统一调用约定的函数调用

##### 3) 使用函数指针

按下述步骤，使用Burst编译的函数指针：

- 为静态函数添加`[BurstCompile]`属性；

- 为包含这些静态函数的类添加`[BurstCompile]`属性。这能帮助Burst编译器找到这些静态函数；

- 声明一个委托，作为这些函数的接口；

- 为这些函数添加`MonoPInvokeCallbackAttribute`属性，让它们能兼容IL2CPP。

  ```c#
  // Instruct Burst to look for static methods with [BurstCompile] attribute
  [BurstCompile]
  class EnclosingType {
      [BurstCompile]
      [MonoPInvokeCallback(typeof(Process2FloatsDelegate))]
      public static float MultiplyFloat(float a, float b) => a * b;
  
      [BurstCompile]
      [MonoPInvokeCallback(typeof(Process2FloatsDelegate))]
      public static float AddFloat(float a, float b) => a + b;
  
      // A common interface for both MultiplyFloat and AddFloat methods
      public delegate float Process2FloatsDelegate(float a, float b);
  }
  ```

- 在C#代码中，编译函数指针：

  ```c#
      // Contains a compiled version of MultiplyFloat with Burst
      FunctionPointer<Process2FloatsDelegate> mulFunctionPointer = BurstCompiler.CompileFunctionPointer<Process2FloatsDelegate>(MultiplyFloat);
  
      // Contains a compiled version of AddFloat with Burst
      FunctionPointer<Process2FloatsDelegate> addFunctionPointer = BurstCompiler.CompileFunctionPointer<Process2FloatsDelegate>(AddFloat);
  ```

- 通常情况下，`MonoPInvokeCallbackAttribute`在AOT的命名空间中是有效的。

  如果它不可用，可以在本地显示声明它：

  ```c#
  public class MonoPInvokeCallbackAttribute : Attribute
  {
  }
  ```

- 默认情况下，Burst为job异步编译函数指针。若想使用同步编译，可使用下述属性：

  `[BurstCompile(SynchronousCompilation = true)]`。

- 最佳性能

  若要从常规C#代码中使用这些函数指针，应将FunctionPointer<T>.Invoke（即委托实例）缓存到静态字段，以获得最佳性能：

  ```c#
  private readonly static Process2FloatsDelegate mulFunctionPointerInvoke = BurstCompiler.CompileFunctionPointer<Process2FloatsDelegate>(MultiplyFloat).Invoke;
  
  // Invoke the delegate from C#
  var resultMul = mulFunctionPointerInvoke(1.0f, 2.0f);
  ```

##### 4) 性能考量

在Burst中，使用job优于函数指针，尤其是在涉及`NativeContainer`(如 `NativeArray`)时。

- Job能获得Burst编译器更深层次的优化；
- `NativeContainer`(如 `NativeArray`)内若包含了托管类型的引用，函数指针无法高效处理这些container的类型检查，但是Job可以。

- 不推荐的示例：

  ```c#
  ///Bad function pointer example
  [BurstCompile]
  public class MyFunctionPointers
  {
      public unsafe delegate void MyFunctionPointerDelegate(float* input, float* output);
  
      [BurstCompile]
      public static unsafe void MyFunctionPointer(float* input, float* output)
      {
          *output = math.sqrt(*input);
      }
  }
  
  [BurstCompile]
  struct MyJob : IJobParallelFor
  {
       public FunctionPointer<MyFunctionPointers.MyFunctionPointerDelegate> FunctionPointer;
  
      [ReadOnly] public NativeArray<float> Input;
      [WriteOnly] public NativeArray<float> Output;
  
      public unsafe void Execute(int index)
      {
          var inputPtr = (float*)Input.GetUnsafeReadOnlyPtr();
          var outputPtr = (float*)Output.GetUnsafePtr();
          FunctionPointer.Invoke(inputPtr + index, outputPtr + index);
      }
  }
  ```

  上述示例使用函数指针会导致严重的性能损失，因为：

  - **无法向量化**：函数指针每次只处理单个数据(标量)，使得Burst编译器无法使用SIMD指令进行并行计算，导致损失了最大的潜在性能增益(4-8倍)。
  - **别名信息丢失**：调用方(Job)已知的、关于数据内存不会重叠(不互为别名)的重要优化信息，无法传递给函数指针，这阻碍了编译器进行进一步的优化。.
  - **调用开销**：每次调用函数指针本身存在固定的**跳转开销**，在频繁调用（如循环中）时，这会累积成明显的性能负担。

- 更优的示例：

  ```c#
  [BurstCompile]
  public class MyFunctionPointers
  {
      public unsafe delegate void MyFunctionPointerDelegate(int count, float* input, 
                                                            float* output);
  
      [BurstCompile]
      public static unsafe void MyFunctionPointer(int count, float* input, 
                                                  float* output)
      {
          for (int i = 0; i < count; i++)
          {
              output[i] = math.sqrt(input[i]);
          }
      }
  }
  
  [BurstCompile]
  struct MyJob : IJobParallelForBatch
  {
       public FunctionPointer<MyFunctionPointers.MyFunctionPointerDelegate> FunctionPointer;
  
      [ReadOnly] public NativeArray<float> Input;
      [WriteOnly] public NativeArray<float> Output;
  
      public unsafe void Execute(int index, int count)
      {
          var inputPtr = (float*)Input.GetUnsafeReadOnlyPtr() + index;
          var outputPtr = (float*)Output.GetUnsafePtr() + index;
          FunctionPointer.Invoke(count, inputPtr, outputPtr);
      }
  }
  ```

  优化后的 `MyFunctionPointer`接收一个表示要处理元素数量的参数，并循环遍历输入和输出指针以执行大量计算。

  `MyJob`则变为一个 `IJobParallelForBatch`作业，并且这个数量参数被直接传递给函数指针。

  上述优化的思想为：

  - **实现向量化**：函数指针现在在内部循环中处理连续数据，使 Burst 编译器能够应用 **SIMD 向量化优化**，解决了之前最大的性能瓶颈。
  - 优先使用作业，并采用批处理设计。

- 最优的示例

  ```c#
  [BurstCompile]
  struct MyJob : IJobParallelFor
  {
      [ReadOnly] public NativeArray<float> Input;
      [WriteOnly] public NativeArray<float> Output;
  
      public unsafe void Execute(int index)
      {
          Output[i] = math.sqrt(Input[i]);
      }
  }
  ```

# Effective C#

本小节参考自：[【《Effective C#》提炼总结】提高Unity中C#代码质量的22条准则](https://zhuanlan.zhihu.com/p/24553860)。

## 原则1: 尽可能使用属性

尽可能使用属性，而不是可直接访问的数据成员。

- 属性允许将数据成员作为共有接口的一部分暴露，同时仍旧提供面向对象环境下所需的封装。

- 使用属性，可以非常轻松的在get和set代码段中加入检查机制。

- **属性底层是通过方法实现的**，其拥有方法所有的语言特性：

  1. 属性增加多线程的支持是非常方便的。可通过加强 get 和 set 访问器来提供数据访问的同步。

  2. 属性可以被定义为**virtual**，也可扩展为**abstract**，也可定义为**接口**，也可使用**泛型**。

     ```c#
     // abstract属性
     public abstract class Shape
     {
         public abstract double Area { get; }       // 抽象属性，无实现   
         public abstract string Name { get; set; }  // 可读写的抽象属性
     }
     
     // virtual属性
     public class Vehicle
     {
         public virtual int MaxSpeed 
         { 
             get { return 100; }  // 默认实现 
         }
         
         // 自动属性, C# 3.0引入的一种简化属性定义的语法 
         // 编译器会自动生成一个私有字段和基本的get和set访问器。
         public virtual string Model 
         {  get;  set; }
     }
     
     // 泛型属性
     public class ResourceManager<T> : MonoBehaviour where T : UnityEngine.Object
     {
         private Dictionary<string, T> m_Resources = new Dictionary<string, T>();
         
          // 泛型索引器
         public T this[string key]
         {
             get
             {
                 if (Resources.TryGetValue(key, out T resource))
                     return resource;
                 return null;
             }
             set => Resources[key] = value;
         }
     }
     ```

  3. 无论何时，需要在类型的公有或保护接口中暴露数据，都应该使用属性。

     如果可以也应该使用索引器来暴露序列或字典。

     现在多投入一点时间使用属性，换来的是今后维护时的更加游刃有余。

## 原则2: 偏向使用运行时常量而不是编译时常量

| 特性       | 编译时常量const            | 运行时常量readonly           |
| ---------- | -------------------------- | ---------------------------- |
| 值确定时机 | 编译时                     | 运行时(构造函数)             |
| 内存位置   | 嵌入到IL中                 | 存储在托管堆中(Managed Heap) |
| 类型限制   | 只能用于数值和字符串       | 可用于任何类型               |
| 访问方式   | 静态访问，实际为类静态变量 | 类实例变量                   |
| 序列化     | 不序列化                   | 可序列化                     |
| 反射修改   | 不能通过反射修改           | 可通过反射修改               |

编译时常量在编译后的IL代码(概念上)：在使用 MathConstants.PI 的地方，编译器会直接替换为 3.14159265358979，而不是引用 MathConstants 类

```c#
// 编译时常量示例
public class MathConstants
{
    public const double PI = 3.14159265358979;
    public const int MaxRetryCount = 3;
}
```

综上，在编译器必须得到确定数值时，一定要使用const。例如特性(attribute)的参数和枚举的定义，还有那些在各个版本发布之间不会变化的值。除此之外的所有情况，都应尽量选择更加灵活的readonly常量。

## 原则3: 推荐使用as或is操作符而不是强制类型转换

- as：作用与强制类型转换是一样，但**不会抛出异常**。若转换不成功，会返回null。
- is : 检查一个对象是否兼容于其他指定的类型，并返回一个Bool值，永远不会抛出异常。

- as相比强制类型转换，更安全和高效

  as操作符在 CIL(Common Intermediate Language)中编译为**isinst**指令：

  ```c#
  // C# 源代码
  object obj = GetSomeObject();
  string str = obj as string;
  
  // 编译后的 IL 代码（简化）
  .locals init (
      [0] object obj,
      [1] string str
  )
  ldloc.0                // 加载 obj
  isinst [mscorlib]System.String  // 执行类型检查
  stloc.1                // 存储结果到 str
  ```

  在JIT层面，**isinst**实现更加高效(流程伪代码如下)：

  ```c#
  // 实际运行时的大致流程
  public static T AsFast<T>(object obj) where T : class
  {
      if (obj == null)
          return null;
      
      // 获取对象的类型句柄
      IntPtr objTypeHandle = obj.GetTypeHandle();
      IntPtr targetTypeHandle = typeof(T).TypeHandle;
      
      // 快速路径：检查是否匹配目标类型或其派生类型
      if (IsInstanceOfClassFast(objTypeHandle, targetTypeHandle))
          return Unsafe.As<T>(obj);  // 无检查的直接转换
      
      // 慢速路径：完整的类型检查
      if (RuntimeTypeHandle.CanCastTo(objTypeHandle, targetTypeHandle))
          return (T)obj;
      
      return null;
  }
  ```

- **as运算符对值类型无效**。当不能使用as进行转换时，才应该使用is，否则is就是多余的。

  ```c#
  // is 操作符（C# 7.0+）
  if (obj is int intValue)
  {
      // ......
  }
  
  // 对于可空值类型, 可行，但有装箱/拆箱开销
  int? nullable = obj as int?;  
  ```

## 原则4: 理解等同性判断

- Object类的**静态方法**

  - ReferenceEquals判断对象引用是否相等

    ```c#
    public static bool ReferenceEquals(object? objA, object? objB)
    {
        return objA == objB;  // 总是比较引用
    }
    ```

  - Equals (object left, object right)判断变量**运行时类型**是否相等，会调用到Object的虚函数Equals

    ```c#
    // Object.Equals 静态方法的实现
    public static bool Equals(object? objA, object? objB)
    {
        if (objA == objB)  // 引用相等或都为null
        {
            return true;
        }
        
        if (objA == null || objB == null)
        {
            return false;
        }
        
        // 调用实例的Equals方法
        return objA.Equals(objB);
    }
    ```

- **IEquatable**接口函数Equals判断对象是否值相等，并配合Object的**虚函数**Equals。

  Object.Equals**默认实现**是比较引用。

  ```c#
  public class Point : IEquatable<Point>
  {
      public int X { get; }
      public int Y { get; }   
      
      // 实现 IEquatable<T> 接口
      public bool Equals(Point? other)
      {
          if (other is null) return false;
          
          // 比较所有相关字段
          return X == other.X && Y == other.Y;
      }
      
      // 重写 Object.Equals
      public override bool Equals(object? obj)
      {
          return Equals(obj as Point);
      }
      
      // 重写 GetHashCode（必须与 Equals 一致）
      public override int GetHashCode()
      {
          return HashCode.Combine(X, Y);
      }   
  }
  ```

- 自定义类重写**静态函数**operator==()；

  ```c#
  public static bool operator ==(Point? left, Point? right)
  {
      if(left is null) return right is null;
      return left.Equals(right);
  }
  
  // 重载 != 运算符
  public static bool operator !=(Point? left, Point? right) => !(left == right);
  ```

综上，不应该覆写Object.referenceEquals()静态方法和Object.Equals()静态方法，因为它们已经完美的完成了所需要完成的工作，提供了正确的判断，并且该判断与运行时的具体类型无关。

对于**值类型**，我们应该总是覆写Object.Equals()实例方法和operatior==(),以便为其提供效率更高的等同性判断。

对于**引用类型**，仅当你认为相等的含义并非是对象标识相等时，才需要覆写Object.Equals( )实例方法。在覆写Equals()时也要实现IEquatable<T>。

## 原则5: 理解短小方法的优势

- 将C#代码翻译成可执行的机器码需要两个步骤：

  ① C#编译器将生成IL，并放在程序集中。

  ② JIT将根据需要逐一为方法(或是一组方法，如果涉及内联)生成机器码。

  短小的方法让JIT编译器能够**更好地平摊编译的代价**。短小的方法也**更适合内联**。

- 简化**控制流程**也很重要。控制分支越少，JIT编译器也会越容易地找到最适合放在寄存器中的变量。

- 所以，短小方法的优势，并不仅体现在代码的**可读性**上，还关系到程序**运行时的效率**。

## 原则6: 选择变量初始化而不是赋值语句

### 类成员初始化

无论调用的是哪一个构造函数，**成员初始化器**是保证类型中成员均被初始化的最简单的方法。

**成员初始化器将在所有构造函数执行之前执行**。

```c#
public class Example
{
    // 字段初始化器, 在所有构造函数前对字段进行初始化
    private int count = 0;                    // 直接初始化字段
    private string name = "Default";          // 字符串初始化
    private List<string> items = new List<string>();  // 对象初始化
    // ........
}
```

其优点是：简洁、易读、防止构造器初始化遗漏。

### 静态初始化

- **静态构造函数**：特殊的构造函数，用于初始化类的静态成员。
  - 它会在类的任何静态成员被访问之前或类的任何实例被创建之前自动调用。
  - 在整个应用程序域的生命周期内最多只执行一次。
- **静态初始化器**：**静态字段的初始化器**，它是在静态构造函数执行之前被运行的。

```c#
class MyClass
{
    private static int myStaticField = 42;
    
    static MyClass()
    {
        // 初始化静态成员
    }
}
```

## 原则7: 理解虚函数和接口函数的区别

- 接口中声明的成员方法默认情况下是**非虚方法**。

  派生类不能覆写基类中实现的非虚接口成员。若要覆写的话，将接口方法声明为virtual即可。

- 基类可以为接口中的方法提供默认的实现。

  派生类继承了基类，所以也默认实现了该接口。

- 虚函数是通过**虚函数表**实现的：

  - **虚表(VTable)**：每个类型(类或接口)都有一个虚表，虚表中存放着该类型的**虚方法指针**。

    当类被加载时，CLR会为它创建虚表。

  - **虚表指针(VPtr)**：每个对象实例都有一个指向其类型虚表的指针。

    在C#中，这个指针通常位于对象实例的内存布局的开头(但具体位置可能因CLR版本和平台而异)。

  - **虚方法调用**：当调用一个虚方法时，CLR会通过对象的虚表指针找到虚表，然后在虚表中找到对应的方法指针，并通过该指针调用方法。

    虚方法调用比非虚方法调用**多一次间接寻址**，因此有轻微的性能开销。但在大多数情况下，这种开销可以**忽略不计**。

  - **继承和重写**：当子类重写父类的虚方法时，子类的虚表中对应的方法指针会被替换为子类重写后的方法地址。这样，通过子类对象调用虚方法时，就会调用子类的方法。

- 接口函数也是通过**虚函数表**实现的：

  接口方法的调用也是通过虚表来实现的，但接口的虚表结构更复杂，因为一个类可以实现多个接口。

  CLR会为**每个接口创建一个单独的虚表**，对象实例中会有**多个虚表指针指向不同的接口虚表**。

## 原则8: 避免返回对内部类对象的引用

- 若将引用类型通过公有接口暴露给外界，那么对象的使用者即可绕过我们定义的方法和属性来更改对象的内部结构，这会导致常见的错误。
- 共有四种不同的策略可以防止类型内部的数据结构遭到有意或无意的修改：
  1. 值类型。当客户代码通过属性来访问值类型成员时，实际返回的是值类型的对象副本。
  2. 常量类型。如System.String。
  3. 定义接口。将客户对内部数据成员的访问限制在一部分功能中。
  4. 包装器(wrapper)。提供一个包装器，仅暴露该包装器，从而限制对其中对象的访问。

## 原则9: 仅用new修饰符处理基类更新

使用new操作符修饰类成员可以重新定义继承自基类的非虚成员，用于解决基类方法和派生类方法冲突的问题。

new操作符必须小心使用。若随心所欲的滥用，会造成对象调用方法的二义性。

new是**方法隐藏**，不具备多态性。

```c#
public class BaseClass
{
    public void NormalMethod() => Console.WriteLine("Base Normal");
    public virtual void VirtualMethod() => Console.WriteLine("Base Virtual");
}

public class DerivedClass : BaseClass
{
    // 使用 new 隐藏基类方法
    public new void NormalMethod() => Console.WriteLine("Derived New");
    
    // 使用 override 重写虚方法  
    public override void VirtualMethod() => Console.WriteLine("Derived Override");
}

// 测试代码
public class TestNewVsOverride
{
    public static void Demo()
    {
        DerivedClass derived = new DerivedClass();
        BaseClass asBase = derived;
        
        Console.WriteLine("=== 直接调用 ===");
        derived.NormalMethod();       // 输出: Derived New
        derived.VirtualMethod();      // 输出: Derived Override
        
        Console.WriteLine("=== 通过基类引用调用 ===");
        asBase.NormalMethod();        // 输出: Base Normal (new 不具多态性)
        asBase.VirtualMethod();       // 输出: Derived Override (override 具多态性)
    }
}
```



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

# 内存管理

本小节参考自：[Understanding Memory Management in Unity](https://discussions.unity.com/t/understanding-memory-management-in-unity/1700107)。

## 操作系统内存工作方式

### 基本概念

| 核心概念       | 定义与功能                                 | 关键细节                                                     |
| -------------- | ------------------------------------------ | ------------------------------------------------------------ |
| 物理内存 (RAM) | 设备上的硬件内存，用于实时加载和访问数据。 | 容量固定； 应用程序不直接操作物理内存                        |
| 虚拟内存 (VM)  | 为每个应用程序提供的独立、沙箱化的内存空间 | 操作系统高效管理物理内存的核心机制； 对应用程序透明          |
| 页 (Page)      | **虚拟内存**的划分单位                     | 典型大小：移动设备16KB，桌面设备4KB； 总VM用量 = 页数 × 页大小。 |
| 帧 (Frame)     | **物理内存**中对应于虚拟内存“页”的单元     | 页与帧通过映射关系关联，这是虚拟内存工作机制的基础           |

### 换页Page In && Page Out

**Page in**：将虚拟内存的page加载到物理内存的帧中；

**Page out**：从物理内存中释放不需要的帧。

<img src="/pic_page_in_out.png" alt="pic_page_in_out" style="zoom:90%;" />

调用malloc，分配虚拟内存，此时物理帧还未被分配：

```c++
const int sizeOfPage = 16384;
const int numberOfPages = 3;
// Allocate an int array corresponding to 3 pages
int* array = (int *)malloc(numberOfPages * sizeOfPage);
```

<img src="/pic_malloc_virtual_mem.png" alt="pic_malloc_virtual_mem" style="zoom:80%;" />

赋值时，触发**缺页异常**：访问未分配物理帧的虚拟内存页，OS将该页加载到物理内存中(**Page In**)。

此块内存页就被称为"**dirty**"：表示它已经被修改了。

```c++
array[0] = 141;//赋值
```

<img src="/pic_page_load.png" alt="pic_page_load" style="zoom:80%;" />

| 页面状态分类 | 定义                   | 对内存占用的影响                                             |
| ------------ | ---------------------- | ------------------------------------------------------------ |
| 干净页       | 内容未被修改           | 可从原文件（如代码、资源）重新加载。<br />**可被OS安全回收**，对实际内存压力影响小。 |
| 脏页         | 内容已被应用程序修改。 | **不可被OS丢弃**。<br />是构成应用程序**实际物理内存占用**的主要部分，直接影响内存压力。 |

## Unity的托管堆(Managed Heap)

| 核心概念             | 说明                                                         | 细节                                                         |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 托管堆(Managed Heap) | 分配所有C#对象的地方                                         | 被垃圾回收器管理                                             |
| 段(Segment)          | GC不直接操作虚拟内存<br />Segment作为GC进行内存管理的基本单元 | 在操作系统层面，一个段由**一个或多个页**组成。<br />Unity当前默认GC配置下，Segment与操作系统页通常是 **1:1 的映射关系**。 |
| Segment和对象分配    | 一个段可以容纳多个托管对象。                                 | 即使一个小对象(如 4 字节的 float)的分配也会导致其所在的整个OS页被标记为“脏”，但**该段/页内剩余的空间仍然可用于分配其他小对象**。 |
| GC(垃圾回收器)       |                                                              | GC：负责在**段**的级别进行对象的分配、标记和清理。<br />**操作系统**：负责在**页**的级别管理物理内存（如标记脏页）。 |

<img src="/pic_segment.png" alt="pic_segment" style="zoom:60%;" />

## 在托管堆中分配/回收对象

### 对象分配

如果Segment内部有足够空间，对象会被分配在现有的段中。

如果现有空间不足以容纳新的分配，GC会创建一个新的Segment并将新对象放入其中。

<img src="/pic_new_allocation_1.png" alt="pic_new_allocation_1" style="zoom:60%;" />

<center style="font-size:16px; color:#7c7b7b; text-decoration:underline">在Segment内部分配对象</center>

<img src="/pic_new_allocation_2.png" alt="pic_new_allocation_2" style="zoom:60%;"/>

<center style="font-size:16px; color:#7c7b7b; text-decoration:underline">新建Segment分配对象</center>

### 对象回收

当垃圾回收执行时，不可达的对象在逻辑上会被回收。

Segment内`Dead Object`所占用的内存区域会被记录在一个空闲列表(**Free List**)中，以供将来重用。

`Dead Object`不再被引用，但它们所在的页仍然映射在物理内存中，并被操作系统标记为脏页(Dirty)。

<img src="/pic_dead_object.png" alt="pic_dead_object" style="zoom:70%;" />

<center style="font-size:16px; color:#7c7b7b; text-decoration:underline">Dead Object被灰色标记</center>

### 对象分配(存在Dead Object)

- 如果新分配的对象大小适配已有的空闲空间，GC会复用那些存放`Dead Object`的内存区域。
- 如果这些段中的剩余空闲空间不够大或不够连续，GC会在现有的堆中搜索另一个合适的区域。
- 如果找不到合适的区域，GC会通过向操作系统申请新的内存段来扩展托管堆。

<img src="/pic_object_allocation.png" alt="pic_object_allocation" style="zoom:90%;" />

## Unity内存使用量不会下降

| 现象                   | 原因                                                         | 影响                                                         |
| ---------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| GC后内存占用不立即下降 | **GC视角**：存放`Dead Object`的段，被GC在逻辑上标记为**可重用**。<br />**OS视角**：认为这些Segment对应的页仍然是脏的，因为它们包含被修改过的数据。 | **大多数情况下是良性的**：保留这些“空但脏”的段有利于后续快速分配，避免频繁向OS申请内存，是**以空间换时间的性能优化**。 |
| "空但脏"的内存段       | 内存段（Segment）内所有对象已被GC回收，逻辑为空，但其底层对应的一个或多个OS页仍驻留在物理内存中，并被标记为脏。 | **长期空置会造成内存浪费**。                                 |
| Unity GC 的平衡策略    | GC会在**连续多个GC周期后(如每第6次)** 自动取消映射这些空段，将其物理内存释放回OS。 |                                                              |

## Larget Object Heap(LOH)

| 特性                 | 描述                                                         | 备注                                                         |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 大对象               | size >= `MAXOBJBYTES`                                        |                                                              |
| 大对象堆(LOH)        | 不占用小对象的Free List：分配大对象时，GC会**绕过小对象空闲列表**，不会尝试复用小对象间的碎片化空闲空间。<br />**独占性**：分配的空间即使有剩余，也不能用于分配其他小对象。<br />**回收**：优先整体复用给类似大小的大对象。 | 警惕大对象分配，尤其是在Update中<br />重用大对象，避免过渡分配<br />使用对象池 |
| 如何分配大对象       | ① **向上取整**至堆块数量的整数倍；<br />② 分配一个连续的堆块序列；<br />③ 整个范围被视为一个**单一/原子**的对象。 | 示例，假设堆块大小为 4 KB<br />请求6 KB → 分配 8 KB(2个块)<br />请求17 KB → 分配 20 KB(5个块) |
| 对象释放后的即时处理 | ① **加入空闲列表**：被释放的大对象内存块会立即进入专门的**大对象空闲列表**，供后续大对象分配复用。<br/>② **合并相邻块**：GC会尝试**合并相邻的空闲块**，形成更大的连续空间，以提高复用几率。<br/>③ **块保持完整**：释放后的内存块**不会立即被分割**用于小对象分配。 |                                                              |
| GC 的内存复用策略    | ① **优先复用**：新的大对象分配会**优先尝试复用**空闲列表中的现有块（“快速路径”）。<br />② **惰性分割**：只有当没有合适大小的空闲块可用时，GC才会**分割**一个更大的空闲块来满足需求。这是一种**按需、保守的策略**，旨在避免不必要的分割操作，平衡内存利用率和性能。 |                                                              |

## Unity当前的GC特性

Unity当前基于Boehm GC，有下述特性：

- 保守式与非移动式，不会在内存中移动存活对象，容易产生内存碎片。
- 以空间换性能：**优先分配速度**，避免移动对象的高昂开销，使单次分配更快速。但以**长期内存碎片化**为代价。
- **保守扫描**：将任何看似指针的值都当作潜在引用，避免误删存活对象，**安全性高但可能误报**（导致少量内存无法回收）。

- 

# 资产系统

## Asset 和 Object

### 1) 概述

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

### 2) 对象间的引用

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

### 3) 序列化和实例化

- PersistentManger

  Unity内部通过*PersistentManager*维护一块**缓存**(cache)。

  cache管理了File GUID、Local ID和Instance ID的映射。

- Instance ID

  - 将File GUID、Local ID转换为会话唯一的整数，即Instance ID。

- Instance ID是单调递增的。当新创建的Object时，会为其赋予Instance ID。

  - 当访问File GUID、Local ID的AssetBundle被卸载时，Instance ID会在cache中被移除。如果相同的AssetBundle被re-load，会为其分配一个全新的Instance ID。

- 在运行时比较File GUID、Local ID是**性能不友好**的，这是设计Instance ID的初衷。

### 4) MonoScripts

*MonoBehavior*持有一个*MonoScripts*的引用，*MonoScripts*持有定位*script class*的信息。

*MonoScripts*包含三个string：程序集名、类名、命名空间。

**在Plugin外**的C#脚本会被编译至*Assembly-CSharp.dll*；

**在Plugin内**的C#脚本会被编译至*Assembly-CSharp-firstpass.dll*。当然Unity也允许自定义程序集。

正是由于存在 MonoScript 对象，AssetBundle（或场景或预制件）中包含的任何 MonoBehaviour 组件实际上都不包含可执行代码。这使得不同的 MonoBehaviour 即使位于不同的 AssetBundle 中，也可以引用特定的共享类。

## AssetBundle

- AssetBundle的本质是一个**归档格式**。

  它将多个独立的资源文件（如模型、纹理、音频等）打包整合成一个单一的文件，类似于ZIP压缩包。

  Unity引擎能够高效地识别、索引和序列化这种特定格式。

- Unity进行**非代码内容热更新**的"主要工具"。

  - 减小初始安装包体积：非核心资源可放在服务器上，用户只需按需下载。
  - 灵活的运行时内存管理：允许你按需加载资源，并在使用完毕后及时卸载。
  - 支持内容差异化与优化：针对不同的设备性能（如高端/低端GPU）、屏幕分辨率或语言地区，创建不同版本的AssetBundle（即AssetBundle Variants）。在运行时能够有选择性地加载。
  
- AssetBundle内部组成(参考自：[effective-asset-management-in-unity-with-addressables](https://discussions.unity.com/t/effective-asset-management-in-unity-with-addressables/1621379))

  | 组成         | 作用                                                         | 影响                                                         |
  | ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
  | 游戏资源     | 加载后在游戏中使用                                           |                                                              |
  | 目录表       | 资源包内部资产的索引地图，用于按名称查找和定位具体资源。     | 内存和磁盘占用很小，是必要开销。                             |
  | 预加载表     | 列出包内每个资产的依赖关系，确保依赖资产被正确加载。         | 通常内存占用可忽略。**极长的依赖链**可能会使其体积增大，需留意。<br />通过合理的资源分组来管理依赖深度，避免过长的依赖链。 |
  | 引擎版本信息 | 记录构建该资源包所用的 Unity 引擎版本。                      | 可能导致**资源包被错误判定为已更改**。<br />使用 **ContentBuildFlags.StripUnityVersion** 构建标志，剥离版本信息(存为0.0.0)，避免无效变更。 |
  | 类型树       | 定义资源包内对象的序列化数据格式，是**跨不同Unity版本加载资源包时的兼容性保障**。 | 运行时类型树在内存中为**全局缓存**，相同类型的对象不会重复占用内存。 |

## AssetBundle && Addressable

本小节参考自：[【Unity 底层与原理向】02_Addressables与AssetBundle设计哲学](https://zhuanlan.zhihu.com/p/1962282846694637775)。

### 框架对比

- AssetBundle

  ```c#
  AssetBundle 架构：
  ┌─────────────────────────────────────────┐
  │       Your Game Code                    │
  ├─────────────────────────────────────────┤
  │  ┌──────────────────────────────────┐   │
  │  │  自定义资源管理器                   │   │  ← 你需要自己实现
  │  │  - 依赖管理                       │   │
  │  │  - 引用计数                       │   │
  │  │  - 缓存策略                       │   │
  │  └──────────────────────────────────┘  │
  ├─────────────────────────────────────────┤
  │     AssetBundle API (Unity 原生)        │  ← 低级 API
  │  - LoadFromFile/Memory/Stream          │
  │  - LoadManifest                         │
  │  - Unload(bool)                         │
  └─────────────────────────────────────────┘
  ```

- Addressable

  ```c#
  Addressables 架构：
  ┌─────────────────────────────────────────────┐
  │       Your Game Code                        │
  ├─────────────────────────────────────────────┐
  │    Addressables High-Level API              │
  │  - LoadAssetAsync<T>("Address")             │
  │  - InstantiateAsync("Address")              │
  │  - ReleaseInstance / ReleaseAsset           │
  ├────────────────────────────────────────────-┤
  │    Addressables System Layer                │  ← Unity 实现的中间层
  │  ┌──────────────────────────────────────┐   │
  │  │  Resource Manager                    │   │
  │  │  - 依赖管理                           │   │
  │  │  - 引用计数                           │   │
  │  │  - 异步加载队列                        │   │
  │  └──────────────────────────────────────┘  │
  │  ┌──────────────────────────────────────┐  │
  │  │  Resource Locator                    │  │
  │  │  - Address → Location 映射           │  │
  │  │  - Catalog 管理                      │  │
  │  └──────────────────────────────────────┘ │
  ├────────────────────────────────────────────┤
  │     AssetBundle API (底层)                  │  ← 底层仍然是 AB
  └────────────────────────────────────────────┘
  ```

Addressables 并不是替代 AssetBundle，而是在其之上构建了一个完整的资源管理框架。

### 核心差异

#### 1) 依赖管理

- AssetBundle手动处理依赖链

  ```c#
  AssetBundleManifest manifest = LoadManifest();
  string[] dependencies = manifest.GetAllDependencies("ui/mainpanel.bundle");
  
  // 先加载所有依赖
  foreach (var dep in dependencies)
  {
      LoadAssetBundle(dep);
  }
  
  // 再加载目标 Bundle
  AssetBundle bundle = AssetBundle.LoadFromFile("ui/mainpanel.bundle");
  GameObject prefab = bundle.LoadAsset<GameObject>("MainPanel");
  ```

  Unity 构建时生成`.manifest`文件记录**依赖关系**。如下示例：

  ```c#
  ManifestFileVersion: 0 CRC: 1234567890 Dependencies:
  common/atlas.bundle
  common/shaders.bundle
  ```

  如果不加载依赖，资源会出现：粉红色材质、缺失引用等问题。

- Addressable自动管理依赖

  ```c#
  // Addressables 自动处理依赖
  var handle = Addressables.LoadAssetAsync<GameObject>("MainPanel");
  await handle.Task;
  GameObject prefab = handle.Result;
  ```

  Addressables 在构建时生成 `catalog.json`;

  Catalog 记录了完整的**依赖图**；

  内部的`ResourceManager` 在加载时自动遍历**依赖树**并按顺序加载。

#### 2) 引用计数

| 特性             | AssetBundle             | Addressables                    |
| ---------------- | ----------------------- | ------------------------------- |
| 引用计数         | ❌ 不提供（需自己实现）  | ✅ 自动引用计数                  |
| 内存泄漏风险     | ⚠️ 高（容易忘记 Unload） | ✅ 低（Handle 模式防护）         |
| 卸载时机         | 手动调用 Unload         | Release 引用计数为 0 时自动卸载 |
| 多次加载同一资源 | 可能重复加载            | 引用计数+1，不重复加载          |

用户为AssetBundle实现引用计数(**简易，未考虑依赖**)：

```c#
public class AssetBundleRefCounter
{
    class AssetEntry
    {
        public Object asset;
        public int refCount;
        public AssetBundle sourceBundle;
    }
    
    private Dictionary<string, AssetEntry> assets = new();
    
    public T Load<T>(string bundlePath, string assetName) where T : Object
    {
        string key = $"{bundlePath}/{assetName}";
        
        if (assets.TryGetValue(key, out var entry))
        {
            entry.refCount++;
            return entry.asset as T;
        }
        
        // 首次加载
        var bundle = AssetBundle.LoadFromFile(bundlePath);
        var asset = bundle.LoadAsset<T>(assetName);
        
        assets[key] = new AssetEntry 
        { 
            asset = asset, 
            refCount = 1,
            sourceBundle = bundle
        };
        
        return asset;
    }
    
    public void Release(string bundlePath, string assetName)
    {
        string key = $"{bundlePath}/{assetName}";
        
        if (assets.TryGetValue(key, out var entry))
        {
            entry.refCount--;
            
            if (entry.refCount <= 0)
            {
                entry.sourceBundle.Unload(true);
                assets.Remove(key);
            }
        }
    }
}
```

#### 3) 异步加载模型

- AssetBundle(回调/协程式)

  ```c#
  // 协程方式
  IEnumerator LoadAssetCoroutine()
  {
      AssetBundleCreateRequest request = AssetBundle.LoadFromFileAsync("path");
      yield return request;
      
      AssetBundle bundle = request.assetBundle;
      AssetBundleRequest assetRequest = bundle.LoadAssetAsync<GameObject>("prefab");
      yield return assetRequest;
      
      GameObject obj = assetRequest.asset as GameObject;
  }
  ```

- Addressable(Task/Handle式)

  ```c#
  // Task 方式（现代异步模式）
  async Task LoadAssetAsync()
  {
      var handle = Addressables.LoadAssetAsync<GameObject>("prefab");
      GameObject obj = await handle.Task;
      
      // 使用完毕后释放
      Addressables.Release(handle);
  }
  
  // Handle 方式（更灵活）
  var handle = Addressables.LoadAssetAsync<GameObject>("prefab");
  handle.Completed += (op) => 
  {
      if (op.Status == AsyncOperationStatus.Succeeded)
      {
          GameObject obj = op.Result;
      }
  };
  ```

- 性能对比

  | 指标         | AssetBundle              | Addressables                       |
  | ------------ | ------------------------ | ---------------------------------- |
  | 首次加载开销 | 基准（100%）             | 约 110-120%（额外的 Catalog 查找） |
  | 重复加载开销 | 完全重新加载             | 引用计数，几乎无开销               |
  | 内存占用     | 更低（无中间层）         | 约 +2-5MB（ResourceManager 等）    |
  | 代码复杂度   | 高（需要自己实现管理器） | 低（开箱即用）                     |

#### 4) 整体加载流程对比

- AssetBundle

  ```c#
  ┌──────────────┐
  │ Game Code    │
  └──────┬───────┘
         ↓
  ┌──────────────────────────────────┐
  │ Your Resource Manager            │
  │ 1. 查找 Manifest 获取依赖          │
  │ 2. 遍历加载依赖 Bundle             │
  │ 3. 加载目标 Bundle                │
  │ 4. 手动管理引用计数                │
  └──────┬───────────────────────────┘
         ↓
  ┌──────────────────────────────────┐
  │ AssetBundle API                  │
  │ - LoadFromFile                   │
  │ - LoadAsset<T>                   │
  └──────┬───────────────────────────┘
         ↓
  ┌──────────────┐
  │  Native I/O  │
  └──────────────┘
  ```

- Addressable

  ```c#
  ┌──────────────┐
  │ Game Code    │
  └──────┬───────┘
         ↓
  ┌──────────────────────────────────┐
  │ Addressables API                 │
  │ LoadAssetAsync("Address")        │
  └──────┬───────────────────────────┘
         ↓
  ┌──────────────────────────────────┐
  │ Resource Manager(Unity内部)       │
  │ 1. 查找 Catalog                   │
  │ 2. 解析 Address → Location       │
  │ 3. 检查引用计数                   │
  │ 4. 自动加载依赖                   │
  └──────┬───────────────────────────┘
         ↓
  ┌──────────────────────────────────┐
  │ Resource Provider(Unity内部)      │
  │ - AssetBundleProvider            │
  │ - BundledAssetProvider           │
  └──────┬───────────────────────────┘
         ↓
  ┌──────────────────────────────────┐
  │ AssetBundle API (底层)           │
  └──────┬───────────────────────────┘
         ↓
  ┌──────────────┐
  │  Native I/O  │
  └──────────────┘
  ```

## Addressable

本小节参考自：[Effective asset management in Unity with Addressables](https://discussions.unity.com/t/effective-asset-management-in-unity-with-addressables/1621379)。

### 分组原则

- **按功能/使用时机分组**(如：“教程组”、“关卡1组”)，而非按资源类型分组(如：“材质组”、“UI组”)。

- 将公共依赖项放入独立组。

- **倾向于使用多个更小的分组**，尤其是在使用时机不确定时。

  更细的粒度允许**更精确的加载和卸载**，减少一次性加载不必要内容的内存压力，便于后续更新。

### 依赖管理

在运行时加载一个asset bundle时，该包的所有**依赖项也需要被加载**。若该资源包含不清晰的依赖关系树，可能导致加载单个资源引发数百个资源下载。

冗长依赖链的常规解决方案：将某个资源移出至某个公共的asset bundle中，以打破冗长的依赖链。

可通过下述方式**检查依赖**：

- **Select Dependencies**：在Editor中右键单击某个资源，点击**Select Dependencies**，Unity会在Inspector中显示其所有的依赖。

- 运行时，可使用**Addressables Profiler Module**跟踪哪些资源被加载了。

- 在代码中使用 **AssetReferences**。

  **AssetReferences** 是 Addressable 系统中的一种**弱引用类型**，它允许你在脚本中声明一个可寻址资源的引用，但**不会**在场景加载时自动加载该资源，从而让你能精确控制资源的加载与卸载时机。

  其使用方式如下：

  | 步骤     | 操作                                                         | 说明                                                         |
  | -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
  | 声明字段 | public AssetReference myPrefabRef;<br/>或使用类型安全的子类。 | 在 `MonoBehaviour`或 `ScriptableObject`中声明。<br />使用泛型子类**在 Inspector 中限制可分配的资源类型**，避免错误。 |
  | 分配资源 | 在**Inspector** 中，将资源**拖拽**到该字段上。               | 即使资源尚未标记为 Addressable，拖拽后也会**自动将其标记并添加到默认组**。 |
  | 异步加载 | LoadAssetAsync\<T\>                                          |                                                              |
  | 释放     | Addressables.Release(handle)                                 | 需要手动释放，避免内存泄漏。                                 |
  | 优势     |                                                              | 1.传统的 `public GameObject`是强引用，迫使资源在场景加载时全部载入。AssetReference允许按需加载，减少内存峰值。<br />2.资源在项目中的**路径或名称发生改变时**，只要Addressable的Key不变，**引用它的 AssetReference 字段就无需修改**。<br />3.依赖关系优化：通过将共享资源设为 AssetReference，可以避免它们被捆绑到特定的功能包中，从而减少不必要的依赖链和包体体积。 |

### 使用特性

- 混合使用Addressable和非Addressable的陷阱

  当一个Addressable资源直接引用一个非Addressable资源时，它将该资源的一个**副本**带入其所在的asset bundle中。

  例如，假设有两个prefab，分属不同的group和不同的asset bundle，但它们都使用了同一个**非Addressable材质**。那么，每个预制体所在的asset bundle都会**包含一份该材质的副本**，因为它不是Addressable资源。

  这会导致构建体积增大，运行时内存使用也会增加。因此：

  - 当你迁移到Addressables时，应确保将所有可能的资源都移入至Addressables系统。如果选择混合方案，可能会带来很多陷阱。
  - 对于场景，正确的方法是用一个**轻量级的非Addressable引导(bootstrap)场景**，负责初始Addressable场景，然后确保所有其他场景都标记为Addressable。

- 加载和卸载

  - 每当使用asset bundle中的一个资源时，Unity会确保对应的资源包已加载到内存中，然后才将该资源加载到内存。

  - **一旦asset bundle中的任何一个资源被加载，只有在该资源包中的所有资源都不再需要时，它才能被卸载**。

    因此需要避免将很多资源划分到一个bundle中，资源分组不当，很容易造成OOM。

- Loading Cache

  - 一个**全局共享**的内存池，用于缓存所有 AssetBundle 最近被访问的数据(如序列化信息)。
  - 在旧系统中(Unity 2019.4 LTS 之前)，每个 AssetBundle 都有自己独立的“序列化文件缓冲区”。**大量小资源包会导致每个包都占用独立的缓冲内存，内存开销线性增长**。
  - 引入全局缓存后，无论有多少个小资源包，它们都**共享同一块缓存区域**。这**极大地降低了对小资源包策略的内存惩罚**。
  - **默认值**：1 MB (`AssetBundle.memoryBudgetKB = 1024`)。

# URP

## 基础概念

### Drall Call && SetPass Call

- 概要

  | 特性     | Drall Call                             | SetPass Call                                                 |
  | -------- | -------------------------------------- | ------------------------------------------------------------ |
  | 目的     | 绘制网格                               | 设置渲染状态(Set Render Pass)                                |
  | 开销     | 相对较小                               | 相对较大                                                     |
  | 触发条件 | 每次绘制网格<br />SetPass Call之后执行 | 每次切换渲染状态，如着色器、纹理绑定、材质属性、混合模式、深度测试等<br />在Draw Call之前执行 |
  | 优化目标 | 减少数量                               | 减少数量                                                     |

- 批处理

  如果多个Draw Call使用**相同的渲染状态**，那么一个SetPass Call后可以跟随多个Draw Call：这就是批处理Batching的基本原理。

  如果渲染状态不变(即没有发生SetPass Call)，连续多个Draw Call的开销会小很多。

- 减少SetPass Call

  - **合并材质**：尽可能让多个物体共享同一个材质。
  - **使用纹理图集**：将多个纹理合并到一张大纹理上，从而减少材质数量。
  - **减少着色器变体**：避免使用过多的Shader变体，因为每个变体都可能导致额外的SetPass Call。

- 减少Drall Call

  - **静态批处理**：对于不会移动的物体，标记为Static，Unity会自动合并。
  - **动态批处理**：对于小网格且使用相同材质的物体，Unity会在每帧动态合并(注意顶点数限制)。
  - **GPU Instancing**：对于相同网格和材质的物体，使用GPU实例化。

### 性能瓶颈SetPass Call

如下图所示，绘制一个蓝色和红色三角形，由于它们的渲染参数不一样。

所以首先要执行SetPass Call设置渲染状态，再执行Drall Call绘制。**真正的性能瓶颈在SetPass Call**，而非Draw Call。

<img src="/pic_setpasscall.png" alt="pic_setpasscall" style="zoom:50%;" />

### Build-In管线多Pass的影响

假设场景中两个白色立方体，两个红色立方体，会产生2个SetPass Call，4个Drall Call：

<img src="/pic_build-in_multiple_pass_1.png" alt="pic_build-in_multiple_pass_1" style="zoom:67%;" />

若给每个立方体新增一个pass绘制描边，那么则会产生8个SetPass Call，8个Draw Call：

<img src="/pic_build-in_multiple_pass_2.png" alt="pic_build-in_multiple_pass_2" style="zoom:70%;" />

**这就是URP把处理多Pass干掉的原因**。类似需求通过定制Render Feature实现，从而优化游戏性能。

## SRP Batcher

本小节参考自：[SRP Batcher原理](https://edu.uwa4d.com/lesson-detail/283/1323/0?isPreview=0)。

| 特性     | 传统Pipeline                               | SRP Batcher                                                  |
| -------- | ------------------------------------------ | ------------------------------------------------------------ |
| 执行流程 | 每帧收集数据->提交数据->绑定数据->DrawCall | 数据保存在GPU内存中，数据变化后更新GPU内存<br />提交数据一次，每帧只需绑定数据 |
| 打断方式 | shader相同，材质属性不同，打断SetPassCall  | 着色器/着色器变种变化，打断SRP Batcher<br />只要着色器变种相同，即使不同材质也不会打断<br />多Pass会打断SRP Batcher |

<img src="/pic_pipeline_contrast.png" alt="pic_pipeline_contrast" style="zoom:80%;" />

### 内存布局

SRP Batcher会将模型的坐标信息、材质信息、主光阴影参数、非主光阴影参数保存到不同的**CBUFFER**，CBUFFER发生变化才会提交。

Shader在显存中通过每帧变化的坐标信息和每帧不一定变化的材质信息渲染出来。

<img src="/pic_cbuffer.png" alt="pic_cbuffer" style="zoom:70%;" />

```glsl
┌─────────────────────────────────────────────────┐
│            SRP Batcher 内存结构                  │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │        Per-Object Constant Buffer       │    │
│  │                                         │    │
│  ├─────────────────────────────────────────┤    │
│  │  Object 1:                              │    │
│  │    • _ObjectToWorld (float4x4)          │    │
│  │    • _WorldToObject (float4x4)          │    │
│  │    • ...                                │    │
│  ├─────────────────────────────────────────┤    │
│  │  Object 2:                              │    │
│  │    • _ObjectToWorld                     │    │
│  │    • _WorldToObject                     │    │
│  │    • ...                                │    │
│  └─────────────────────────────────────────┘    │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │       Per-Material Constant Buffer      │    │
│  │       (MyLitShader)                     │    │
│  ├─────────────────────────────────────────┤    │
│  │  Material 1:                            │    │
│  │    • _Color (float4)                    │    │
│  │    • _MainTex_ST (float4)               │    │
│  │    • _Metallic (float)                  │    │
│  ├─────────────────────────────────────────┤    │
│  │  Material 2:                            │    │
│  │    • _Color                             │    │
│  │    • _MainTex_ST (float4)               │    │
|  |    • _Metallic (float)                  |    |
│  └─────────────────────────────────────────┘    │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │       Per-Material Constant Buffer      │    │
│  │       MyLitShader(keyword: _NORMALMAP)  │    │
│  ├─────────────────────────────────────────┤    │
│  │  Material 1:                            │    │
│  │    • _Color (float4)                    │    │
│  │    • _BumpScale (float)                 │    │
│  │    • _Specular (float)                  │    │
│  ├─────────────────────────────────────────┤    │
│  │  Material 2:                            │    │
│  │    • _Color                             │    │
│  │    • _BumpScale (float)                 │    │
|  |    • _Specular (float)                  |    |
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
```

所有Mesh的CBUFFER共享同一套内存模版，放在一个较大的内存中。

材质的CBUFFER是根据**着色器/着色器变体**分块的，所以SRP Batcher能通过**减少SetPassCall**来提升性能。

- 同一个SRP Batch内的材质，属性共享**同一个内存布局模板**，从而被打包到**同一个UnityPerMaterial CBUFFER 池**中。
- 不同着色器或不同着色器变体的材质，其 `UnityPerMaterial`CBUFFER是**物理隔离**的，位于 GPU 内存的不同区域。
- 为了最大化SRP Batcher的批处理效率，应尽可能减少**着色器变体数量**：谨慎使用`multi_compile`和`shader_feature`。

### 自定义材质CBUFFER

```glsl
CBUFFER_START(UnityPerMaterial)
float _BaseMap_ST;
half4 _BaseColor;
half4 _SpecColor;
half4 _EmissionColor;
half4 _Cutoff;
// 其他属性......
CBUFFER_END
```

同一个材质只能写一个`CBUFFER_START(UnityPerMaterial)`。

## SRP渲染管线

SRP渲染管线是基于CPU层的，主要做了：收集渲染数据、裁剪与排序、提交数据到GPU。

### CPU渲染管线

- 准备渲染数据：场景中已启动的、能被摄像机看见的游戏对象(Render组件)。

- 裁剪数据：落在摄像机外的游戏对象会被剔除。

- 渲染排序：不透明物体**从前往后**渲染，半透明物体**从后往前**渲染。半透明物体不写入深度。

- 提交GPU

- 引起卡顿的原因：CPU Bound、GPU Bound。

  <img src="/pic_performance.png" alt="pic_performance" style="zoom:80%;" />

### GPU渲染管线

渲染管线是硬件的行为，软件无法控制。目前移动端主要采用高通的芯片和Mali的芯片，本小节以Mali的GPU架构进行讨论。

#### 1 TBR架构(Tile Based Rendering)

**芯片内**保留了**片上内存**(Local Tile Memory)，**芯片外**是主存，芯片访问主存需要经过**带宽**。

理想情况下，芯片应尽可能地访问片上内存，但是**片上内存的容量是有限的**。

<img src="/pic_tbr_workflow.png" alt="pic_tbr_workflow" style="zoom:50%;" />

- GPU从主存读取顶点信息，执行顶点着色器。
- GPU将顶点数据以**Tile的形式写回主存**。
- GPU执行片元着色器，同时将顶点的Tile数据从主存读取进来，再采样主存中的贴图，进行着色。
- GPU将着色信息保存在Local Tile Memory中。这一步在片上，不占用带宽。
- 最后将着色的Tile信息写回到主存的FrameBuffer中。**若前后2帧中，某个Tile的数据是一致的，就不用回写，体现了TBR的优势**。

#### 2  顶点着色器

#### 3 片元着色器

# 物理系统

## Tips && Tricks

### Force Mode总结

本小节参考自：[Know the difference between ForceModes](https://www.reddit.com/r/Unity3D/comments/psukm1/know_the_difference_between_forcemodes_a_little/)。

<img src="/pic_force_mode.png" alt="pic_force_mode" style="zoom:30%;" />

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

Unity不再对Animation系统进行维护，现主攻新的Mecanim动画系统，其特点如下：

- 使用**多线程**进行计算，比Animation单线程性能更高。

- Animator**状态机**可以让动画在不同的条件下轻松切换，**Layer**能让人物上下半身播放不同的动画。

- Avatar系统使**Retargeting**便捷：把互不兼容的骨骼结构，转换映射到**统一的Unity肌肉系统中**。使一套动画可在不同的骨骼系统上播放。

  AvartarA：存储SkeletonA至Unity肌肉系统的映射；

  AvartarB：存储SkeletonB至Unity肌肉系统的映射；

  SkeletonA的动画ClipA根据AvatarA转换到Unity肌肉系统中。其再通过AvartarB转换到SkeletonB中，从而使不同的骨骼播放相同的动画。

## Root Motion

本小节参考自：[【Unity动画系统详解 十八】Humanoid动画中的Root Motion机制及相关配置](https://www.bilibili.com/video/BV11T4y1Y7RX?vd_source=05b55243d77a9d5af441b9f57356ea36)。

将角色根骨骼的绝对运动(位移/旋转)，转换为游戏对象的**相对位移和旋转**。

- Body Transform：Avatar系统根据骨骼结构计算出整体的**质心**，**质心会随着动画在世界空间中运动**。

  可通过`Animator.bodyPosition`、`Animator.bodyRotation`等获取其运动。

- Root Transform：**质心运动在水平平面的投影**，这个投影被当作**人形动画**代表root motion的根骨骼节点。

  可通过`Animator.rootPosition`、`Animator.rootRotation`等获取其运动。

- Root Motion本质：在humanoid动画中，unity会计算出Root Transform。Root Motion会把动画文件描述的Root Transform坐标和旋转，**转换为相对位移和转角**(delta)，以此驱动游戏对象运动。

- Root Motion构成

  - Root Transform Rotation：这个设置控制角色整体的旋转。
  - Root Transform Position(Y)：控制角色在垂直方向(上下)的移动。
  - Root Transform Position(XZ)：控制角色在水平面上的移动。

- Bake Into Pose属性

  本小节参考自：[【Unity动画系统详解 十七】Root Motion（Generic）基础设置](https://www.bilibili.com/video/BV1fq4y1Y7Sz?vd_source=05b55243d77a9d5af441b9f57356ea36)；

  示例可参考：[根运动 (Root Motion) – 工作原理](https://blog.csdn.net/MyArrow/article/details/45505085)。

  - **勾选**：动画中的变换(位移/旋转)被视为**“姿势”**(Pose) 的一部分，不会应用到GameObject上，只会应用到蒙皮的骨骼变换上。
  - **未勾选**：动画中的变换将被转换为差值(Delta)，应用到GameObject的Transform上，驱动角色在场景中移动。

## 动画状态过渡

- **Has Exit Time**：决定何时开始动画过渡。

  - **标准化时间**，理解为动画播放的百分比。

  - 与**过渡条件(Conditions)**协同工作。

    若**未设置**过渡条件，到达Exit Time就开始状态过渡；

    若设置了过渡条件，到达Exit Time后还需判断过渡条件是否触发。

    对于循环动画，每一轮循环都需要进行一次上述检查。

- **Fixed Duration**：决定两个动画之间过渡时间和计算方式。

  - 勾选：此时`Transition Duration`是以秒为单位的绝对值。无论源动画有多长，过渡的混合过程都会严格按照设定的时间执行。

  - 未勾选：此时`Transition Duration`是一个相对于源动画长度的比例值(标准化时间)。

    例如，如果**源动画**长度为5秒，`Transition Duration`设为0.2，那么实际的过渡时间就是**5秒 * 0.2 = 1秒**。

- **Transition Offset**：定义动画过渡中，目标动画开始播放的时间点。

  - **标准化时间(百分比)**。例如，0.5 表示从目标动画的中间(50%进度)开始播放；
  - 只影响目标动画，对源动画没有任何影响；
  - 与`Exit Time`配合，可精确控制动画何时退出，何时进入。

## Animation Rigging

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

# 渲染原理

## Early-Z GPU硬件优化技术

### 简述

- 后置深度测试的缺陷

  深度测试是在片元计算完毕后才做的测试，因此大部分被遮挡的片元在被剔除前就经历了着色器的计算，造成了**资源的浪费**。

- Early-Z(前置深度测试)的优化

  Early-Z专门为这种情况做了优化：其在光栅化后，片元着色之前进行一次深度测试。

  若深度测试失败，直接跳过片元计算阶段，节省大量GPU算力。

- 最终汇集到后置深度测试

  Early-Z阶段无论是否被遮挡，最终都汇集到ZTest深度测试再测试一次，由后置的深度测试最终决定是否抛弃该片元：Early-Z阶段未通过测试的片元，在后置深度测试上会直接被抛弃；反之，则会继续渲染流程。

  <img src="/pic_early_z.png" alt="pic_early_z" style="zoom:30%;" />

### 渲染流程

- Z-pre-pass

  对于所有写入深度数据的物体，先用一个简单的Pass写入深度缓存，不写入像素；

- 第二个pass

  **关闭深度写入**，开启深度测试，使用正常渲染流程渲染。

- Alpha-Test的限制

  Alpha-Test的做法(discard一个像素)可以让我们在片元着色器中**抛弃像素**。

  若像素在片元着色器中被抛弃了，Early-Z的结果就会出现问题：测试通过的可见片元被抛弃后，被它遮挡的片元就成为可见片元，导致Early-Z的测试成果出现问题。

  因此GPU在优化算法中，对片元着色器**抛弃像素**、**修改深度值**的操作做了检测，如果检测到片元着色器存在上述操作，则Early-Z将被放弃使用。

## 显存工作原理

### GPU内存结构

- GPU显存：GPU独立主内存，物理上**位于GPU芯片旁**。通常只存在于**PC**上，移动端没有。

  其容量大，速度较慢，存储GPU当前任务所需的所有数据。

  OpenGL的glTexImage2D是把数据**从CPU传到显存**。当这个纹理或其局部需要被使用时，缓存机制才会由GPU硬件自动启动。

- GPU缓存：集成在GPU芯片内部，**被所有处理单元共享的高速缓存**，主要是L2缓存。

  数据被以**缓存行**为单位从显存调入**缓存**。

- GPU每个处理单元的缓存：每个流式处理器内部的存储单元，主要包括：

  - L1缓存：一个可配置的高速存储器块。在计算卡上，它通常被明确分为**L1缓存**和**程序员可操控的共享内存**。在游戏卡上，这两者通常是同一块物理内存，由硬件/驱动动态分配。
  - 寄存器：专属于单个**线程**。每个线程的局部变量、中间计算结果都存放在自己的寄存器中。

### 移动设备处理流程

在PC上，数据要从系统内存**拷贝**到显存中。

移动设备上，由于没有独立显存，GPU会共享CPU的物理内存作为**“显存”**，当使用glTexImage2D上传数据时：

- 若数据满足GPU的访问要求，直接映射到GPU的虚拟地址空间。
- 若数据不满足GPU的访问要求，GPU驱动会分配一块新内存，通过CPU/DMA拷贝数据，映射到GPU的虚拟地址空间。
- 该数据要被使用时，触发缓存机制，加载到GPU芯片的缓存中。

### GPU带宽

GPU与其存储系统之间(主要是**显存**)的数据传输速率，衡量了**GPU单位时间内能“吞下”多少数据**。

是GPU性能的三大支柱之一(带宽、计算能力FLOPs和延迟)。

```
显存带宽 = 显存位宽 × 显存频率 × 倍增系数

示例：RTX 4090
显存位宽：384位（bit）
显存频率：21 Gbps（Giga-bit per second）
倍增系数：GDDR6X采用PAM4编码，有效数据速率翻倍

计算：384位 × 21 Gbps × 2 ÷ 8位/字节 = 1008 GB/s
```

