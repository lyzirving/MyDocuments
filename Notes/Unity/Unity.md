---
typora-root-url: pic
---

​	Unity开发的本质：利用反射和引擎提供的各种功能进行拓展开发。

​	Unity在运行之中，通过反射解析用户提供的脚本，得到脚本信息，进行逻辑处理。

# C#基础

## getter/setter

```c#
// name属性未封装, 通过public关键字暴露给系统中的其他类
public class person1
{
    public string name;
}
// Name属性通过get和set关键字进行了封装, 分别对应的是可读和可写
public class person2
{
    public string Name{set;get;}
}
// ------------------------
// person2的Name属性相当于如下, 通过Name, 类可以控制外部访问name的逻辑
private string name;
public string Name
{
    get { return name; }
    set { name = value; }
}
```

​	实例化person2时，系统会先分配一个叫name的private私有的内存空间。name的读与写的操作都是通过Name这个public属性，这个属性类似于函数指针，以此达到封装的目的。

​	属性在调用者看来就像一个普通的变量，但作为类的设计者，**可利用属性来隐藏类中的一些字段**，使外界只能通过属性来访问你的字段，

​	get/set中可添加**逻辑判断**：

```c#
class Person
{
    private string name;
    public string Name
    {
    	get { return name; }
    	set {
        	name = String.IsNullOrEmpty(value) ? "null" : value;
    	}
	}   
}
```

## 装箱/拆箱

​	可参考：[C#装箱和拆箱](https://blog.csdn.net/qiaoquan3/article/details/51439726)，效率优化可参考：[使用泛型时避免装箱](https://www.cnblogs.com/minotauros/p/10041644.html)。

## 委托Delegate

### 1 定义

​	委托是**数据类型**，可以像定义**结构体**一样定义一个委托类型。

​	委托是一种**引用类型变量**，其存有对**某个方法**的引用，且引用可在运行时被改变。

```c#
//语法
delegate <return type> <delegate-name> <parameter list>
//实例
public delegate int MyDelegate (string s);
```

​	C#的委托类似**C++中指向函数的指针**。

### 2 实例

```c#
using System;

delegate int NumberChanger(int n);

namespace DelegateAppl
{
   class TestDelegate
   {
      static int num = 10;
      public static int AddNum(int p)
      {
         num += p;
         return num;
      }

      public static int MultNum(int q)
      {
         num *= q;
         return num;
      }
      public static int getNum()
      {
         return num;
      }

      static void Main(string[] args)
      {
         // 创建委托实例, 传入被委托的函数
         NumberChanger nc1 = new NumberChanger(AddNum);
         NumberChanger nc2 = new NumberChanger(MultNum);
         // 使用委托对象调用方法
         nc1(25);
         nc2(5);
      }
   }
}
```

### 3 多播

​	委托对象可使用 “+” 运算符进行合并。

​	一个合并委托调用它所合并的多个委托，只有相同类型的委托可被合并。

​	"-" 运算符可用于从合并的委托中移除组件委托。

```c#
static void Main(string[] args)
{  
    NumberChanger nc;// 创建委托实例
    NumberChanger nc1 = new NumberChanger(AddNum);
    NumberChanger nc2 = new NumberChanger(MultNum);
    nc = nc1;
    nc += nc2;    
    nc(5);// 调用多播
}
```

## 泛型

```c#
// where T:IComparable是对泛型的约束, 使实际类型必须实现ICompareble接口
public class Compare<T> where T:ICompareble 
{
    public static T CompareGeneric(T t1,T t2)
    {
        if(t1.CompareTo(t2)>0)
        {
            return t1;
        } 
        else
        {
        	return t2;
        }
    }
}
```

​	泛型的优势：

- 实现代码的重用。
- 无需装箱和折箱。泛型类在实例化时，按照传入的类型生成**本地代码**。本地代码数据类型已确定，所以无需装箱和折箱，提升运行时间。

​	更多的泛型知识，可参考：[C#泛型的理解](https://blog.csdn.net/qq_44116353/article/details/122455528)。

# 工具

## 快捷键

- 操作工具：

<img src="/operation_tool.png" alt="operation_tool" style="zoom:60%;" />

- 热键集：

<img src="/hot_key.png" alt="hot_key" style="zoom:60%;" />

## 标签

- ContextMenu

​	向编辑器添加命令：点击按钮会调用关联方法，通常用于测试。

```c#
[ContextMenu("按钮名称")]
void TestFun()//关联的测试方法
{ 
}
```

## 项目打包

<img src="/build_settings.png" alt="build_settings" style="zoom:100%;" />

# 基础知识

## 坐标系

​	Unity使用**左手系**：屏幕向右+X(红色)，屏幕向内+Z(蓝色)，屏幕向上+Y(绿色)。

<img src="/axis.png" alt="axis" style="zoom:70%;" />

## 工具类

### 1 Mathf

```c#
namespace UnityEngine
{
    //
    // Summary:
    //     A collection of common math functions.
    public struct Mathf
    {
        //向上取整
        public static float Ceil(float f);
        //向下取整
        public static float Floor(float f);
        //四舍五入
        public static int RoundToInt(float f);
        //判断正负数
        public static float Sign(float f);
        //插值函数: 线性插值, 使用场景如下:
        //场景一: start按先快后慢的节奏变化, 结果无限趋近于dest
        //       start = Mathf.Left(start, dest, Time.deltaTime);      
        //场景二: 匀速变换
        //       time += Time.deltaTime;
        //       result = Mathf.Lerp(start, dest, time);    
        public static float Lerp(float a, float b, float t)
		{
    		return a + (b - a) * Clamp01(t);
		}
        ........
        ........
    }
}    
```

## GameObject和场景

- GameObject是所有场景对象的**基类**。

​	场景对象本质都是GameObject，由于挂载了**不同组件**，使各对象表现不同。**脚本也是一种组件**。

<img src="/game_object.png" alt="game_object" style="zoom:80%;" />

- 场景**本质**是后缀为.unity的**配置文件**

​	Unity通过自己的机制读场景文件，动态创建GameObject对象，并向其关联组件(脚本)，使对象在场景中各司其职。

## 预设体和资源包

- 预设体(prefab)：保存单个场景对象(GameObject)的信息，后缀为.prefab

​	创建预设体：将GameObject拖动到Assets目录下。

​	修改预设体：

方法1. 重新拖入到Assets目录下；

方法2.在Inspector中，在GameObject的Panel下，点击Prefab/Overrides/Apply All

方法3.右键，选择unpack prefab

- 资源包.unitypackage

​	选择Assets中需要被复用的资源、脚本等，导出为资源包，供其他unity工程使用。

​	**插件**就是由资源包组成。

## 脚本

### 1 基本操作

- 脚本**文件名**和**类名**必须一致，因为反射会通过文件名去查找类型。
- 没有特殊需求，不用考虑命名空间。
- 可在编辑器的路径下(Editor/Data/Resources/ScripTemplates)，修改各种脚本的默认模版。
- 脚本和Game Object关联后，该脚本就处理该Object的逻辑。

### 2 MonoBehavior类

#### 2.1) 脚本默认继承自MonoBehavior

- Unity**不允许**在脚本中new继承自MonoBehavior的对象，因为MonoBehavior的设计目的是挂载到GameObject对象上。
- 继承了MonoBehavior的类，**不要覆写它的构造函数**。
- 一个GameObject可挂载多个不同类型的脚本(实际是**Component**)，也可挂载多个相同类型的脚本。可在类上声明标记**DisallowMutipleComponent**，禁止GameObject挂载多个**同类型脚本**。

#### 2.2) 未继承MonoBehavior的类

- 这些类不能被挂载到GameObject上，但可以创建/覆写构造函数，需要使用new构建。
- 这些类一般用作单例类(管理类)或数据类，遵循面向对象的编程规则。

### 3 生命周期函数

#### 3.1) 函数列表

<img src="/script_lifecycles.png" alt="script_lifecycles" style="zoom:70%;" />

#### 3.2) 不是MonoBehavior的成员函数

​	生命周期函数**不是MonoBehavior的成员函数**。

​	它们是脚本类中首次声明、定义的函数，可以是private、protected和public。

​	Unity在特定的执行时期，使用**反射**，通过**函数名**得到对应生命周期函数，并执行。

#### 3.3) 函数特点

- 生命周期函数在**游戏主线程**中按先后顺序执行

- Awake()、OnDestroy()在脚本的生命周期中只调用一次
- OnEnable()、OnDisnable()在脚本对象每次使能/失效时被调用
- FixedUpdate()、Update()、LateUpdate()游戏主循环中调

### 4 Inspector中可编辑变量

#### 4.1) public变量默认可被编辑

​	脚本中的public变量默认可在Inspector中编辑，但可通过添加标签，使其不能被编辑：

```c#
public class Test : MonoBehaviour
{
    [HideInInspector]
    public int publicInt2 = 50;
}
```

#### 4.2) private/protected变量不能被编辑

​	脚本中的private/protected默认不能在Inspector中编辑，但可通过添加标签，使其被编辑：

```c#
public class Test : MonoBehaviour
{
    //序列化:把一个对象保存到文件或数据库
    [SerializeField]
    private int privateInt;
    [SerializeField]
    protected string protectedStr;
}
```

#### 4.3) 少数类型不能被编辑

```c#
public enum E_TestEnum
{
    Normal,
    Player,
    Monster
}

public struct MyStruct
{
    public int age;
    public bool sex;
}

public class Test : MonoBehaviour
{
    //可编辑类型
    public int[] array;
	public List<int> list;
	public E_TestEnum type;
	public GameObject gameObj;
    
    //不可编辑类型
    public Dictionary<int, string> dic;
    public MyStruct myStruct;//自定义类型变量
}
```

​	自定义类型默认不能被编辑，但是添加了**序列化标签**后，可被编辑：

```c#
[System.Serializable]
public class MyClass
{
    public int age;
    public bool sex;
}
```

# 重要组件

## GameObject

### 1 静态方法

- 实例化/克隆对象

```c#
//一般克隆预设体, 也可传场景对象
GameObject.Instantiate();
```

- 删除对象

```c#
#region
//删除对象、组件或资源
//不会立刻删除对象, 标记对象, 在下一帧将其从内存移除
GameObject.Destroy(obj);
//立即删除对象, 建议使用GameObject.Destroy, 降低卡顿概率
GameObject.DestroyImmediate(obj);
#endregion
```

​	切换场景时，场景中的对象**默认会被删除**。下述方法使场景切换时，不删除指定对象：

```c#
GameObject.DontDestroyOnLoad();
```

### 2 成员方法

- 脚本中new一个GameObject，就会被添加到场景中

```c#
GameObject obj = new GameObject();
```

- 动态添加脚本，获取脚本

```c#
Test comp  = obj.AddComponent<Test>();
Test comp1 =  obj.GetComponent<Test>();
```

- 比较标签

```c#
bool equal1 = obj.CompareTag("Player");
bool equal2 = (obj.tag == "Player");
```

- 一些低效的成员函数(不建议使用)

```c#
// 遍历挂载的所有脚本, 寻找名为"TestFunc"函数, 并执行
obj.SendMessage("TestFunc");

//广播行为: 遍历自身/子对象的所有脚本, 找到TestFunc, 并执行
obj.BroadcastMessage("TestFunc");

//广播行为: 遍历自身/父对象的所有脚本, 找到TestFunc, 并执行
obj.SendMessageUpwards("TestFunc");
```

## Camera

### 1 属性总览

<img src="/cam_attr_0.png" alt="cam_attr_0" style="zoom:70%;" />

<img src="/cam_attr_1.png" alt="cam_attr_1" style="zoom:70%;" />

### 2 属性解释

- Depth

​	场景中多个Camera都会渲染到屏幕上，渲染的顺序通过Depth控制。

​	Main Camera的Depth为-1，Depth越大，顺序越靠后。

​	若两个Camera指定的Viewport一致，那么后执行的Camera，渲染内容会覆盖之前的。

- Target Texture

​	将Camera的渲染内容输出到指定的纹理上。

- Occlusion Culling

​	执行渲染前，在CPU端计算在Camera视角下的遮挡，从而不提交被遮挡的mesh，避免GPU资源的浪费。

### 3 监听/委托

​	Camera在特定时机/生命周期上，通过委托，实现回调：

```c#
// 在Culling前回调
Camera.onPreCull += (cam) =>
{
};
// 在渲染前回调
Camera.onPreRender += (cam) =>
{ 
};
// 在渲染后回调
Camera.onPostRender += (cam) =>
{
};
```

# 核心系统

## 物理系统

### 1 RigidBody刚体

#### 1.1 参数

<img src="/rigidbody_attr_0.png" alt="rigidbody_attr_0" style="zoom:60%;" />

<img src="/rigidbody_attr_1.png" alt="rigidbody_attr_1" style="zoom:60%;" />

#### 1.2 碰撞产生必要条件

- 碰撞的两个物体都有**碰撞器**

- 其中一个物体包含**刚体**组件

​	碰撞器用于描述物体的**碰撞区域**，执行**碰撞检测**；刚体用于进行**物理模拟**。

#### 1.3 碰撞检测的几种模式

​	离散检测是性能消耗最低，但精度最差的检测模式。

​	两个刚体可能设置了不同的检测模式，下表列出了两种检测模式相遇时的组合结果。

<img src="/collision_detection_modes.png" alt="collision_detection_modes" style="zoom:60%;" />

### 2 Collider

#### 2.1 Collider参数

<img src="/collider.png" alt="collider" style="zoom:70%;" />

​	其中Mesh Collider、Wheel Collider和Terrain Collider不常用。

#### 2.2 碰撞检测回调

​	碰撞检测在物理更新(FixedUpdate)中执行和回调：

<img src="/collision_detection_cb.png" alt="collision_detection_cb" style="zoom:80%;" />

### 3 物理材质

<img src="/physics_material.png" alt="physics_material" style="zoom:50%;" />

### 4 改变物体位置的四种方式

<img src="/physics_change_pos.png" alt="physics_change_pos" style="zoom:85%;" />

# 输入相关

- 屏幕原点

​	根据Input.mousePosition，屏幕坐标原点在窗口左下角，向右为+x，向上为+y。

- 鼠标输入API

Input.GetMouseButtonDown、Input.GetMouseButtonUp、Input.GetMouseButton、Input.mouseScrollDelta、Input.GetKeyDown、Input.GetKeyUp、Input.GetKey

- 默认轴输入，配置在Edit/Project Settings/Input Manager/Axis中

Input.GetAxis()：返回float，-1~0~1，有中间过渡值

Input.GetAxisRaw()：返回-1、0、1，无渐变

<img src="/default_axis.png" alt="default_axis" style="zoom:80%;" />

