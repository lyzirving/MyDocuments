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