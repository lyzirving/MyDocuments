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

# 编译&&构建

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
