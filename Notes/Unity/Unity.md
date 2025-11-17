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

# 引擎系统

## UGUI

### 1) 基础概念

#### 1.1) Graphic类

`Graphic`类是所有可视化UI组件(如`Image`、`Text`、`RawImage`)的抽象基类。

- 数据生成和传递

  `Graphic`的核心方法`OnPopulateMesh`负责生成定义UI形状的网格数据(顶点、颜色、UV等)，提交给同级的`CanvasRenderer`组件。

- 增量更新的标记系统

  UGUI性能优化的关键。当UI元素的属性发生变化时，`Graphic`不会立即重新计算网格，而是调用相应的`SetVerticesDirty()`, `SetMaterialDirty()`, 或 `SetLayoutDirty()`方法，将自己标记为“脏”状态，表示需要重建顶点、材质或布局。

- 统一重建

  所有被标记为“脏”的 `Graphic`元素都会在每帧渲染前，由 `CanvasUpdateRegistry`统一调用其 `Rebuild`方法。其会根据脏标记的类型，有选择地调用`UpdateGeometry`和`UpdateMaterial`来完成实际的数据更新。

#### 1.2) CanvasRenderer

UI 元素(如 `Image`、`Text`)与底层图形管线之间的桥梁。

- 接收网格

  `Graphic`组件(例如 `Image`)会调用 `OnPopulateMesh`方法来生成顶点数据，然后通过 `canvasRenderer.SetMesh()`将网格设置给 `CanvasRenderer`。

- 材质设置

  `Graphic`组件会计算最终使用的材质，通过 `canvasRenderer.SetMaterial()`进行设置。

- 提交数据

  `CanvasRenderer`将网格和材质信息提交给底层的`Canvas`系统。`Canvas`会负责对所有 UI 元素进行排序和合批，最终生成指令并提交给 GPU 渲染。

#### 1.3) CanvasUpdateRegistry

`CanvasUpdateRegistry`是Unity UGUI 系统的核心调度中心，它确保所有 UI 元素都能高效、有序地更新和渲染。

- 中央调度器

  作为一个单例类，统一管理所有需要更新的UI元素。

- 脏标记收集

  维护两个独立队列：布局重建队列(`m_LayoutRebuildQueue`)和图形重建队列(`m_GraphicRebuildQueue`)，收集被标记为“脏”的元素。

- 有序更新管线

  在每帧渲染前，严格按照特定阶段(布局：Prelayout→Layout→PostLayout；渲染：PreRender→LatePreRender)执行重建。

- 依赖关系处理

  在布局更新前对元素进行排序(按层级深度从父到子)，确保父容器先于子元素计算，解决尺寸依赖问题。

#### 1.4) Canvas

- 放置UI元素，控制绘制顺序

  UI元素的绘制顺序和其在Canvas层级中的显示顺序一致。层级中第一个元素最先绘制，依次类推。

  可通过拖拽来改变UI元素的绘制顺序，也可通过Transform组件的：`SetAsFirstSibling`，`SetAsLastSibling`和`SetSiblingIndex`方法改变顺序。

- Canvas嵌套

  Canvas组件可以嵌套在另一个Canvas组件下，即**子Canvas**。

  子Canvas可以把它的子物体与父Canvas分离。当子Canvas被标记为Dirty时，不会强制Rebuild父Canvas，反之亦然。

- 驱动UI更新

  通过**CanvasUpdateRegistry**系统，每帧管理并触发需要更新的**Layout**和**Graphic**组件进行重建。

  **Layout Rebuild**：当布局改变时（如文本长度变化），重新计算 UI 元素的位置和大小；

  **Graphic Rebuild**：当图形的视觉属性改变时（如颜色、纹理），重新生成图形的网格。

- 执行批处理

  Canvas会将其下所有UI元素的网格几何体合并，根据材质和渲染顺序等进行**批处理**，生成一个合并的大网格，尽可能减少 **Draw Call** 的数量。

### 2) UI Batching合批

本小节参考自：[Unity3D UGUI系列之合批](https://blog.csdn.net/sinat_25415095/article/details/112388638)。

Canvas通过合并UI元素的网格，生成合适的渲染命令发送给Unity图形渲染流水线。

合批的目的是为了**一次性发送尽可能多的数据**，减少Draw Call的调用。

如果Draw Call过多，那么CPU就会把大量的时间花在准备数据和设置渲染状态上，造成性能问题。

#### 2.1) 合批的前提

- 合批是以Canvas为单位，不包含子Canvas，子Canvas会是另外一个批次；
- 合批是在**子线程**中完成；
- **材质**和**纹理**需要相同。

#### 2.2) Depth计算规则

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

#### 2.3) VisibleList排序流程

- 计算UI元素的Depth；

- 按Depth升序排序，剔除Depth = -1的元素，Depth是**最高优先级**；
- Depth相同的元素，按material ID升序排序；
- material ID相同的元素，按texture ID升序排序；
- texture ID相同的元素，再按Hierarchy中的顺序排序；
- 最终得到合批前的元素，即VisibleList。

#### 2.4) 执行合批

判断VisiableList中相邻的元素是否有相同的材质和贴图，若满足要求，则进行合批。

注意，这里只考虑材质和纹理，不用再考虑Depth和其他元素了。

#### 2.5) 示例

<img src="/pic_ui_batch_eg.png" alt="pic_ui_batch_eg" style="zoom:70%;" />

Depth计算：Image1.Depth = 0; Image2.Depth = 0; 由于Image2和Image3重叠，且纹理不同，所以有Image3.Depth = 1;

Depth相同的，按material排序：Image1 -> Image2 -> Image3；

material id相同的，再按texture id排序，由于Image2.textureId < Image1.textureId，所以有：Image2 -> Image1 -> Image3；

texture id相同的，再按hierarchy排序，由于Image1在Image3前，所以顺序不变：Image2 -> Image1 -> Image3

最后合批，batch 0: Image2；batch 1: Image1, Image3。

#### 2.6) UI合批优化

- 使用图集，统一纹理；
- Text如果能用纹理代替，尽量用纹理；
- 避免频繁删除/增加UI元素，会引起Canvas的Rebuild；
- 尽量不要使用Mask，内部使用了模版缓冲，至少增加两个Draw Call；
- 动静分离，动态UI和静态UI分别使用不同的Canvas。

### 3) 常见问题

#### 3.1) UI元素移出屏幕后，DrawCall为什么没降低

在Unity的UGUI系统中，Canvas下的所有可见UI元素会被CanvasRenderer收集并批量渲染。
即使UI元素移出了屏幕(比如RectTransform的位置超出了可视区域)，只要：

- 它的GameObject是激活的；
- CanvasRenderer的enabled是true；
- 透明度大于0；
- 没有被禁用Raycast Target；

UGUI还是会把它当作“需要渲染”的对象处理，只是这些像素因为超出视口范围，被裁剪掉了。

#### 3.2) 如何让移出屏幕的UI不再消耗DrawCall

- SetActive(false)：彻底禁用GameObject，UI不会被渲染，也不会消耗DrawCall。

- CanvasRenderer.enabled = false：只禁用渲染，不影响逻辑。

- 分Canvas管理：把动态UI和静态UI分到不同Canvas，动态UI隐藏时可以整体禁用Canvas，进一步减少DrawCall。

- **ScrollRect/Mask**：这些组件只是限制了UI的显示区域(像素遮罩)，但并没有减少被渲染的UI数量。

  高性能的列表(如ListView虚拟化)会只生成可见区域的Item，未显示的Item直接SetActive(false)，这样才能真正减少DrawCall和CPU消耗。
