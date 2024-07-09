---
typora-root-url: pic
---

​	Unity开发的本质：利用反射和引擎提供的各种功能进行拓展开发。

​	Unity在运行之中，通过反射解析用户提供的脚本，得到脚本信息，进行逻辑处理。

# 快捷键

- 操作工具：

<img src="/operation_tool.png" alt="operation_tool" style="zoom:60%;" />

- 热键集：

<img src="/hot_key.png" alt="hot_key" style="zoom:60%;" />

# 基础知识

## 坐标系

​	Unity使用**左手系**：屏幕向右+X(红色)，屏幕向内+Z(蓝色)，屏幕向上+Y(绿色)。

<img src="/axis.png" alt="axis" style="zoom:70%;" />

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

