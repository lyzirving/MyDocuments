# Framework

## 1 工作流

① 引擎初始化，场景加载

② 调用Node`_ready`函数。Node的遍历采用**深度优先模式**，若有如下节点：

​               Scene_Root

​          |----------|----------|

​          |            |             | 

​      Level    Player   Enemy

​          |

   |---------|

Key        Door

​	那么`_ready`被调用的顺序为Key -> Door -> Level -> Player -> Enemy -> Scene_Root。

③ 进入主循环。

④ 事件输入，Input单例更新，调用Node的`_input`。

⑤ 调用Node的`_process`和`_physics_process`([Gogot官方：Idle and Physics Processing](https://docs.godotengine.org/en/3.6/tutorials/scripting/idle_and_physics_processing.html))：

​	引擎调用`_process`依赖应用的fps和平台特性，因此会传入delta来执行不依赖帧率的计算。

​	`_physics_process`在物理线程中执行，通常它按固定帧率调用(默认60fps)。

​	`_physcis_process`**调用后**物理引擎会执行`physcis step`，进行物理模拟。

​	综上，`_process`不与`_physics_process`同步。但是在**单线程**游戏中(可配置)，它们的执行顺序是`_physics_process` -> `physics step` -> `_process`。

⑥ 执行运行时对Node的插入、删除等节点操作。

⑦ 执行渲染，然后**开始新的一帧**(回到步骤④)。

⑧ 若游戏被结束，调用Node的`_notification`。

⑨ 释放引擎资源，关闭引擎。

## 2 \_process和\_physics_process

​	本小节参考自：

[Difference between _process(delta) and _physics_process(delta)?](https://gamedev.stackexchange.com/questions/192180/difference-between-processdelta-and-physics-processdelta/192182#192182)

[Make it clear when to use _process and _physics_process](https://github.com/godotengine/godot-proposals/discussions/4937)

[Godot Docs: Physics process callback](https://docs.godotengine.org/en/3.3/tutorials/physics/physics_introduction.html#physics-process-callback)

- 图形

​	\_process在每个图形帧中被调用。在_process中只能处理**图形元素**，如移动**MeshInstance3D**等。

​	在Vsync没有开启的情况下，图形帧率只与应有程序的处理速度相关。VSync(默认开启)会影响_process，硬件显示帧率和应用的帧率会影响\_process的处理周期。\_process中传入了delta参数，让开发者进行帧率无关的计算。

​	在\_process中**避免**读取物理属性(position、raycast等)，这些物理属性会由物理服务更新。

​	可使用`set_process`来禁用\_process。

- 物理

​	\_physics_process是物理处理，Godot默认会开启**物理处理线程**，其处理帧率固定在60fps(可配置)。

​	Godot在调用\_physics_process之后，会进行物理模拟(physics step)，计算物理属性。若需要物理模拟的反馈，就需要使用\_physics_process。

​	在\_physics_process中，只能处理物理元素，如**PhysicsBody3D、KinematicBody**等。

​	可使用`set_physics_process`禁用\_physics_process。

- 非物理，非图形的情况

​	如要处理非图形且非物理的固定周期逻辑，可考虑使用`Timer`。

- 输入事件Input

​	可以在\_process或\_physics_process中处理Input。若要获取最准确的结果，最好在\_input中处理。

# Node

## 1 Control

​	覆写虚函数\_draw()，实现绘制自定义形状。若需\_draw()重绘，要调用queue_redraw()。

## 2 CharacterBody3D

​	用于角色控制，该Node的脚本模版具有角色控制功能。

## 3 粒子系统GPUParticles3D

​	使用Node：GPUParticles3D，其内部主要包含被重复绘制的Mesh、粒子控制和显示控制。在节点内部，有如下属性：

- DrawPass：控制粒子的Mesh。

- Process Material：控制粒子的运动和显示属性。
- 父类GeometryInstance3D：可控制粒子Mesh的材质。

# GDScript关键字

## 1 @tool

​	让脚本在editor中运行，方便调试，如自定义UI。

​	在脚本中添加@tool后，需要reload scene。

# 功能

## 1 Navigation

<img src=".\pic\godot_navigation.png" alt="godot_navigation" style="zoom:85%;" />

​	NavigationServer3D清楚Map的一切。

​	NavigationAgent3D是客户端(即Enemy)，若客户端想知道如何从A到B，需要向NavigationServer3D请求。

​	NavigationServer3D会根据NavigationRegion3D中烘焙的Mesh进行计算，找到一条**避开障碍物的路径**，反馈给客户端。

## 2 Animation trick ---- lerp

​	本小节参考自：[Lerp (rachsmith.com)](https://rachsmith.com/lerp/)。lerp是线性插值，即：

```c++
interpolation = A * (1 - t) + B * t = A + (B - A) * t
```

​	lerp可以实现简单的easing effect：

```javascript
// to run on each frame
function lerp(position, targetPosition) {
    // update position by 20% of the distance between position and target position
    position.x += (targetPosition.x - position.x) * 0.2;
    position.y += (targetPosition.y - position.y) * 0.2;
}
```

​	position是当前位置，每帧都在变化。targetPosition是目标值，不会变化。

​	随着targetPosition和position之间的差距变小，就会形成渐进的、平滑的动画效果。

# 光照工作流

​	总体工作流分为4步：

① color transform(tone mapping)；② white balance；③ Lighting和exposure；④ PostFX & Color grade。

## 1 环境光设置

### 1.1) 使用HDR天空图

① color transform(tone mapping)

​	在**WorldEnvironment**节点中，Tone Map表示如何把linear RGB转换到sRGB。Blender使用Filmic，UE4使用Aces。Tone Map参数建议：● Mode：Filmic；● White：6(4~16)，会影响背景光和模型上的光照

② white balance

​	3d场景中只用白色的光，除非有特殊需求，不要给环境光上色，否则会混乱色彩变换，使后期处理困难。

③ Lighting和exposure，设置光照和曝光

​	调整DirectionalLight节点：

​	● 调整DirectionalLight位置和旋转，使光线和环境光贴图上的太阳光线在视觉上匹配。

​	● 调整光的强度和阴影颜色。推荐参数：Energy：2，Shadow：(153, 153, 153)

​	WorldEnvironment中AmbientLight，推荐参数：Color：(128, 128, 128)，Energy：3，Sky Contribution：(0.3)。

### 1.2) 使用程序式环境光

​	WorldEnvironment的Background属性中，参数Background/Mode：Sky，参数Background/Sky：new Procedural Sky，得到由程序参数控制的环境光。

① color transform && ② white balance，配置程序式环境光

​	程序式环境光可控制三个部分：Sky、Ground和Sun。

​	Background/Sky/Sky参数建议：color：#36508d，horizon：#8bafcd，Curve：0.13。

​	Background/Sky/Ground参数建议：bottom color：#8bafcf，horizon color：#8bafcf。

​	Background/Sky/Sun参数建议：Angle max：30，Energy：30。

​	WorldEnvironment的Ambient属性中，调整其color和sky contribution，设置环境光对场景物体的影响。推荐参数见1.1)小节。

③ 设置光照和曝光：同1.1)小节。

### 1.3) 使用shader环境光

# 设计模式

## 1 class

​	在脚本中，通过：

```c++
extends Node3D #在extends 之后使用class_name
class_name XXX
```

​	可以为脚本附着的Node定义一个类型，有如下作用：

- 类型是**全局的**，可以在**所有脚本**中访问。
- 通过类型标识不同的Node。

## 2 继承场景

<img src=".\pic\godot_inheritance.png" alt="godot_inheritance" style="zoom:80%;" />

​	节点可以使用继承/派生的设计模式，从而可以：

- 重用脚本中的代码
- @export的属性，在不同子节点中设置不同的参数。

​	在文件系统中找到基类节点的**场景文件**，右键 -> New Inherited Scene，即可创建子节点。

# 快捷键

## 1 GDScript

- 注释：ctrl + k