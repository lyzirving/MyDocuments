---
typora-root-url: pic
---

# 基础概念&&操作

## 快捷键

- ctrl + d：复制actor；
- alt + 移动gizmo：复制actor到指定位置；

## UE基础

### 1 基础概念

- Objects：Collection of data and functionality，即数据和方法的集合。
- Actors：**Object** that can go in a level，即可在关卡中行动的对象。
  Cube是一个**actor**，其内部包含各种数据。
  <img src="/UE_Cube_actor.png" alt="UE_Cube_actor" style="zoom:50%;" />
- Component：**Objects** that can go on an actor，即可挂载到actor上的对象(数据和方法)。
- Reference：Where to find an object，即通过内存地址找到某个Object。
  在蓝图中，可以为actor创建reference，从而在蓝图中操作actor。
- Spawning：Create an object while playing。
- Actor的**父子关系**：
  可将新增actor作为场景中已存在actor的子节点，使其以父子关系管理，如下所示，SM_DoorFrame是SM_Shelf中StaticMeshComponent的子节点：
  <img src="/UE_parent_child_actor.png" alt="UE_parent_child_actor" style="zoom:60%;" />

### 2 坐标系和单位

​	UE的坐标系是**左手系**，且每个坐标轴的意义和普通的坐标系不太一致。

- X轴朝前，默认屏幕向外。
- Y轴朝右，默认屏幕左侧。
- Z轴朝上(z-up)，默认屏幕上方。
- UE默认长度单位是**厘米**。

<img src="/UE_coordinate.png" alt="UE_coordinate" style="zoom:50%;" />

### 3 PlayerStart

​	PlayerStart是C++类，表示：游戏开始时，玩家的**起始位置**。

<img src="/UE_player_start.png" alt="UE_player_start" style="zoom:50%;" />

​	PlayerStart相关操作：

- F8：在游戏时，点击**F8**，可以为当前的PlayerStart生成一个实体，用于Debug。
- **Get Player Pawn**：蓝图中根据Get Player Pawn节点，传入Player索引，可获得PlayerStart这个Actor。
  在单人游戏中，Player索引通常为0。
- Get Control Rotation：在蓝图中，通过Get Player Pawn -> Get Control Rotation，可获得相机当前的旋转角度。

## Blueprint基础

### 1 基础概念

​	蓝图是一种可视化编程语言，与C++相反，它是一种文本语言，专为UE编写的。下述是蓝图相关的概念：

- Node：Premade functionality，节点是预设的功能。
- Event：A “when” node，Event也是节点，代表某个触发时机。其被罗列在**EventGraph**中。
- Pin：Sockets we can connect up，引脚，用于连接节点。引脚按数据流向，可分为输入引脚和输出引脚。
  
  Input pin：When to run this node。
  
  Output pin：What to do after。
  
  引脚按类型，可分为数据引脚、执行引脚。
  **Data pin**(Input/Output)：The input or output data for a node，用于挂载数据。
  **Execution pin**(Input/Output)：When to run this Node。Input Pin指启动节点的时机。Output Pin指节点结束运行的时机。
  
- Branch Nodes：let us check if something is true or not and then do something based upon the result。分支节点，如下所示：
  <img src="/UE_branch_node.png" alt="UE_branch_node" style="zoom:60%;" />

- Comparison：比较节点，如Less than、Greater than、Equal等等，并返回一个boolean。

- 添加注释：点击C键，可为蓝图添加注释块。

​	下述是使用蓝图的简单例子：

<img src="/UE_level_blue_print_simple_sample.png" alt="UE_level_blue_print_simple_sample" style="zoom:70%;" />

### 2 蓝图类

​	如下步骤可为常见中的Actor创建一个蓝图类：

<img src="/UE_create_bp_class.png" alt="UE_create_bp_class" style="zoom:60%;" />

### 3 蓝图函数

​	Functions execute blocks of blueprints that makes our game do things。

​	Functions可以让蓝图块保持**整洁**，让蓝图块可以**重用**。

- 创建蓝图function的两种方法：
  ① 在左侧菜单栏FUNCTIONS旁点击加号。
  ② 框选要组成方法的节点，点击鼠标右键，选择Collapse to Function如下所示：
  <img src="/UE_make_blueprint_function.png" alt="UE_make_blueprint_function" style="zoom:80%;" />
- 入参/出参：在方法蓝图右侧的详细面板中，可为方法添加入参/出参。
  注意，出参是一个**Return Node**，需要在最后被执行。
- Pure Function
  Side Effect：Observable effect of a function。
  Pure Function：Function with no side effect。Pure Function通常是Get某个值，或比较某个值。
  按下述方法可以将一个函数转变为pure函数：
  <img src="/UE_make_pure_function.png" alt="UE_make_pure_function" style="zoom:50%;" />
  此时，在pure函数就**没有执行引脚**了：
  <img src="/UE_pure_function.png" alt="UE_pure_function" style="zoom:60%;" />

### 4 成员函数

​	成员函数不在关卡的蓝图上，而在某个特定的**蓝图类**中。蓝图类的成员函数对面向对象编程很重要。

​	每个蓝图成员函数都有一个参数，**该参数是要调用该函数的蓝图类实例**。

- Self：A node available in member function, always points to the current instance。Self是个节点，用于在成员函数中，获取当前实例。
- 创建成员函数：
  <img src="/UE_make_member_function.png" alt="UE_make_member_function" style="zoom:60%;" />

