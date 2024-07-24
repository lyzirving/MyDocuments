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

## 图形学基础

### 1 向量叉乘

​	下图展示了目标向量各分量计算规律：

① A和B的x分量不参与目标向量的x分量计算，A和B的y分量不参与目标向量的y分量计算，A和B的z分量不参与目标向量的z分量计算。

② 按顺序计算：x由y、z计算，y由z、x计算，z由x、y计算。

<img src="/vector_cross.png" alt="vector_cross" style="zoom:50%;" />

​	向量叉乘的几何意义：

① 得到平面的法向量。

② 计算两个向量的左右关系。

### 2 坐标系

​	OpenGL使用**右手系**：屏幕向右+X(红色)，屏幕向上+Y(绿色)，**屏幕向外**+Z(蓝色)，角度**逆时针**为正向。

​	Unity使用**左手系**：屏幕向右+X(红色)，屏幕向上+Y(绿色)，**屏幕向内**+Z(蓝色)，角度**顺时针**为正向。

<img src="/axis.png" alt="axis" style="zoom:70%;" />

### 3 四元数

#### 3.1 使用四元数的原因

​	欧拉角存在缺陷：

① 同一旋转的表示不唯一。

② 万向节死锁。

#### 3.2 四元数的构成

​	四元数包含一个标量和一个3d向量：$\begin{pmatrix}w & x & y & z\end{pmatrix}$，w为标量，$\vec{v} = \begin{pmatrix}x & y & z\end{pmatrix}$为3d向量。

​	通过四元数得到的欧拉角范围在[-180, 180]之间。

- 单位四元数

​	单位四元数表示没有旋转量，角度为0或360：[1, (0, 0, 0)]或[-1, (0, 0, 0)]；

- 轴-角对

​	绕着轴$\vec{n}$，旋转$\theta$弧度，可由四元数表示为：$[cos(\theta/2),\;\;sin(\theta/2)n]$，各个分量拆开，即如下：

$[cos(\theta/2),\;\;sin(\theta/2)x, \;\;sin(\theta/2)y,\;\;sin(\theta/2)z]$

#### 3.3 四元数相关工具

- 四元数插值和转向函数

​	Unity提供了Quaternion.**Lerp**和Quaternion.**Slerp**作为四元数的插值函数。Lerp比Slerp更快，但如果旋转范围越大，Lerp效果越差，一般使用Slerp。

​	Unity提供了Quaternion.**LookRotation**(面朝向量)，可以得到对应朝向的旋转量。

- 四元数相乘

​	四元数相乘表示**旋转量的累加**：$q_{3} = q_{1} * q_{2}$。

​	四元数的旋转是以**局部坐标系为参考**，因此**旋转顺序**为：先按$q_{1}$旋转，在新的局部坐标系下，按$q_{2}$旋转。

​	与其对应的，以**固定/世界坐标为参考**，有旋转矩阵$M = M_{2} * M_{1}$，此时的顺序为，先转$M_{1}$，再转$M_{2}$。

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

### 2 Vector3

```c#
public struct Vector3 : IEquatable<Vector3>, IFormattable
{
    //线性插值
    public static Vector3 Lerp(Vector3 a, Vector3 b, float t);
    //球形插值
    public static Vector3 Slerp(Vector3 a, Vector3 b, float t);
    ........
}
```

<img src="/lerp&&slerp.png" alt="lerp&&slerp" style="zoom:50%;" />

### 3 多线程

​	Unity**支持**多线程，可使用C#中System.Threading.Thread类。

​	工作线程**无法访问**Unity相关对象的内容，若在工作线程中调用相关函数，会报错。

​	Unity的线程需要手动关闭。一般在脚本的**生命周期函数**中关闭线程(abort和置空)。

​	Unity的线程一般用于执行复杂计算，然后使用**共享容器**读取计算结果。

### 4 协程

#### 4.1 协程是什么

​	协同程序简称协程，它是“假”的多线程，本质是在**当前线程**上，将逻辑**分时分段执行**，从而缓解当前线程的压力。

#### 4.2 Unity中使用协程

- 继承MonoBehavior的类，都可以开启协程函数；
- 声明协程函数

① 返回值为IEnumerator类型及其子类；

② 函数中通过yield return进行返回，yield return是C#的语法糖。

```c#
//返回值必须是IEnumerator或者继承它的类型
IEnumerator MyCoroutine(int i, string str)
{
    //步骤一
    print(i);
    //协程函数当中必须使用yield return进行返回
    //返回
    yield return new WaitForSeconds(5f);//返回, 等待5s后再执行后续步骤
    //步骤二
    print(str);
}
```

- 开启协程函数

```c#
//StartCoroutine是MonoBehavior的方法
Coroutine c1 = StartCoroutine(MyCoroutine(1, "123"));
```

- 关闭协程函数

```c#
//关闭所有协程函数
StopAllCoroutines();

//关闭指定协程函数
Coroutine c1 = StartCoroutine(MyCoroutine(1, "123"));
StopCoroutine(c1);
```

- 协程中可插入无限循环

​	下述代码不会阻塞主线程，且能无限执行下去：

```c#
IEnumerator MyCoroutine(int i, string str)
{
    while(true)
    {
        print("5");
        yield return new WaitForSeconds(5f);
    }
}
```

- 跳出协程

```c#
//跳出协程, 类似于StopCoroutine()
yield break;
```

- 协程受载体的影响

​	协程开启后，组件和物体销毁，协程不执行；组件失活协程执行，物体失活协程不执行。

​	综上，协程唯有**组件失活时**不受影响，其它情况协程均会停止。

#### 4.3 yiled return不同含义

```c#
//等待指定秒后执行, 在Update()和LateUpdate()之间执行
yield return new WaitForSeconds(5f);

//在下一帧执行, 在Update()和LateUpdate()之间执行
yield return 1;   //任意数字
yield return null;

//等待下一个固定物理帧更新时执行, 在FixedUpdate和碰撞检测相关函数之后执行
yield return new WaitForFixedUpdate();

//等待摄像机和GUI渲染完成后执行, 在LateUpdate之后的渲染相关处理完毕后之后
yield return new WaitForEndOfFrame();
```

#### 4.4 协程的本质

​	协程可分为两部分：① 协程函数；② 协程调度器。

​	Unity内部实现了协程调度器，帮助我们管理协程函数。

- C#的迭代器结构

```c#
namespace System.Collections
{
    public interface IEnumerator
    {
        //当前yiled return的返回值
        object Current { get; }
		//执行协程指令, 直到遇到yield return
        //若返回true, 表示之后仍有指令需要执行; 若返回false, 表示之后没有指令
        bool MoveNext();
        void Reset();
    }
}
```

- 不依靠内置调度器，手动执行协程

```c#
IEnumerator Test()
{
    print("execute 1");
    yield return 1;
    print("execute 2");
    yield return 2;
    print("execute 3");
}

void Start()
{
    IEnumerator ie = Test();//构建协程对象
	while(ie.MoveNext())
	{
    	print(ie.Current);
	}
}
```

- 自定义协程调度器

```c#
//自定义协程函数返回值
public class YieldReturnTime
{
    public IEnumerator ie;
    public float time; //执行时间
}

public class CoroutineMgr : MonoBehaviour
{
    private static CoroutineMgr instance;
    //单例
    public static CoroutineMgr Instance => instance;
    //协程函数容器
    private List<YieldReturnTime> list = new List<YieldReturnTime>();
    
    void Awake()
	{
    	instance = this;
	}
    
    //启动自定义协程函数
    public void MyStartCoroutine(IEnumerator ie)
	{
        if(ie.MoveNext())
        {
            //自定义协程返回值处理逻辑
            if(ie.Current is int)
            {
                YieldReturnTime y = new YieldReturnTime();
                y.ie = ie;
                y.time = Time.time + (int)ie.Current;//记录下一次执行时间
                list.Add(y);//最终将自定义的协程添加到list中
            }
        }
    }
    
    void Update()
    {
    	//倒序遍历, 因为遍历时要移除元素
        for (int i = list.Count - 1; i >= 0; i--)
        {
            if( list[i].time <= Time.time )//到达执行时间
            {
                if(list[i].ie.MoveNext())//执行协程, 且之后仍有逻辑需要执行
                {	
                    if(list[i].ie.Current is int)
                    {
                        list[i].time = Time.time + (int)list[i].ie.Current;//更新下次处理时间
                    }
                    else
                    {
                        list.RemoveAt(i);
                    }
                }
                else
                {
                    //执行完所有协同程序, 从容器的末尾删除
                    list.RemoveAt(i);
                }
            }
        }
    }
}
```

## Resource资源

### 1 特殊文件夹

- 工程路径

​	由Application.dataPath获取(**$PROJECT_PATH\$\Assets**)。该变量只能在编辑模式下使用，游戏发布后失效。

- Resources资源文件夹

​	Resources文件夹需要开发者自己**手动构建**，一般作为Assets的**直接子目录**，且Resources文件夹可以**存在多个**(分布在不同子目录中)。

​	通过**Resources API**使用的资源都需放到Resources文件夹中；

​	游戏发布后，该文件夹下所有文件都会被Unity**加密打包**，且为**只读**。

- StreamingAssets

​	由Application.streamingAssetsPath获取(**\$PROJECT_PATH\$/Assets/StreamingAssets**)，需要开发者**手动构建**。

​	该文件夹下的文件打包出去**不会被压缩加密**，在移动平台只读，PC平台可读可写。

​	该文件夹下可放入一些需自定义动态加载的初始资源。

- persistentDataPath，持久数据文件夹

​	由Application.persistentDataPath获取。

​	不需开发者创建，所有平台可读可写。

​	一般用来存取游戏中动态改变的数据或文件等。

- Plugins插件文件夹
- Editor编辑器文件夹

​	需要**开发者创建**。

​	开发Unity编辑器时，编辑器相关脚本放在该文件夹中。该文件夹中内容不会被打包出去。

### 2 资源及其加载/释放

#### 2.1 常用资源和同步加载

- 预设体——GameObject

​	预设体实质是GameObject的**配置文件**，后缀为.prefab。

```c#
// 加载预设体(配置文件)到内存中
Object obj = Resources.Load("Cube");
// 通过配置文件, 实例化对象
Instantiate(obj);
```

- 音效资源——AudioClip

```c#
//类型转换, 保证Music/BkMusic一定是音乐资源
AudioClip clip = Resources.Load("Music/BkMusic") as AudioClip;
//泛型
AudioClip clip1 = Resources.Load<AudioClip>("Music/BkMusic");
```

- 文本资源——TextAssets

​	Unity支持的文本格式有：.txt、.xml、.bytes、.json、.html、.csv等等。

- 图片资源——Texture

```c#
Texture tex = Resources.Load<Texture>("Image/bk");
```

- 资源缓存

​	资源被加载后，就会被**缓存在内存中**，除非主动释放该资源。

​	**多次**通过Resources.Load()加载**同名资源**，在第一次调用时，会将资源从磁盘加载到内存。其他时候，Load()发现资源已被缓存，便返回缓存对象。

#### 2.2 异步加载

- 通过事件完成异步加载

```c#
//定义内部变量
private Texture tex;

private void LoadOver( AsyncOperation rq)
{
    tex = (rq as ResourceRequest).asset as Texture;
}

void Start()
{
    //LoadAsync会开启一条线程, 进行异步加载
    ResourceRequest rq = Resources.LoadAsync<Texture>("Tex/TestJPG");
    //添加资源加载结束的监听
    rq.completed += LoadOver;
}
```

​	其中，completed的声明如下：

```c#
public event Action<AsyncOperation> completed {}
```

- 通过协程完成异步加载

```c#
private Texture tex;

// 在MonoBehavior的Start()生命周期中, 启动协程
void Start()
{
    StartCoroutine(Load());
}

IEnumerator Load()
{
    // LoadAsync会启动一个线程, 进行异步加载
    ResourceRequest rq = Resources.LoadAsync<Texture>("Tex/TestJPG");
    //通过yield return ResourceReuqest, Unity的协程调度知道你在异步加载资源
    yield return rq;  
    //判断资源是否加载结束
    while(!rq.isDone)
    {
        //加载进度, 不是特别准确, 过渡不明显
        print(rq.progress);
        yield return null;
    }
    tex = rq.asset as Texture;
}
```

#### 2.3 资源释放

- 卸载未使用的资源

```c#
Resources.UnloadUnusedAssets();
GC.Collect();
```

- 卸载指定资源

```c#
Resources.UnloadAsset(tex);
tex = null;
```

​	注意，**GameObject**不能被如此卸载，GameObeject只能被**Destroy()**。

#### 2.4 资源加载封装

```c#
public class ResourcesMgr
{
    private static ResourcesMgr instance  = new ResourcesMgr();
    //单例
    public static ResourcesMgr Instance => instance;

    private ResourcesMgr()
    {
    }
	//泛型方法
    public void LoadRes<T>(string name, UnityAction<T> callBack) where T:Object
    {
        ResourceRequest rq = Resources.LoadAsync<T>(name);
        rq.completed += (a) =>
        {
            //外部传入callback, 在完成异步加载后直接调用
            callBack((a as ResourceRequest).asset as T);
        };
    }
}
```

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

- FixedUpdate

​	在Edit/ProjectSetting/Time/Fixed Timestep可设置物理更细间隔Time.fixedDeltaTime；

​	FixedUpate**每秒**调用次数是一定的，但**每帧**调用的次数**不是一定的**，因游戏中每帧时间不一样。

​	在脚本的生命周期内，FixedUpdate处有一个循环。这个循环累计物理时间，时间间隔大于0.02了，调用一次。若有很多物体进行物理更新，那么FixedUpdate的调用频率也会慢下来。

- LateUpdate

​	LateUpdate在**所有的**Update执行完后再执行。

​	将相机更新放在LateUpdate中主要有两个原因：

① 多个脚本的Update的**执行顺序不确定**；

② Unity内部**动画的逻辑更新**在Update和LateUpdate之间，若把相机更新放在Update，那么场景物体的动画还没完成更新。

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

### 1 GameObject和场景

- GameObject是所有场景对象的**基类**。

​	场景对象本质都是GameObject，由于挂载了**不同组件**，使各对象表现不同。**脚本也是一种组件**。

<img src="/game_object.png" alt="game_object" style="zoom:80%;" />

- 场景**本质**是后缀为.unity的**配置文件**

​	Unity通过自己的机制读场景文件，动态创建GameObject对象，并向其关联组件(脚本)，使对象在场景中各司其职。

### 2 静态方法

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

### 3 成员方法

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

