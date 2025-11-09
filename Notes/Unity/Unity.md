---
typora-root-url: pic
---

# C#基础

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

### 2) 使用和意义

使用：

- 使用时会new一个委托实例，分配在Heap上。因此delegate若使用不当，可能会造成GC压力或内存泄漏。
- 通过`+=`添加的方法，不需要使用后，要通过`-=`移除，避免内存泄漏。

  因为`+=`不仅会持有实例方法，还会隐式持有实例对象的引用。
- 使用event关键字，确保事件的安全性。

意义：

- 事件驱动编程；
- 回调（Callback）机制；

### 3) Action和Func

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

