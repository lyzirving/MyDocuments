# Node

## 1 Control

​	覆写虚函数\_draw()，实现绘制自定义形状。若需\_draw()重绘，要调用queue_redraw()。

## 2 CharacterBody3D

​	用于角色控制，该Node的脚本模版具有角色控制功能。

# GDScript关键字

## 1 @tool

​	让脚本在editor中运行，方便调试，如自定义UI。

​	在脚本中添加@tool后，需要reload scene。

# Navigation

<img src=".\pic\godot_navigation.png" alt="godot_navigation" style="zoom:85%;" />

​	NavigationServer3D清楚Map的一切。

​	NavigationAgent3D是客户端(即Enemy)，若客户端想知道如何从A到B，需要向NavigationServer3D请求。

​	NavigationServer3D会根据NavigationRegion3D中烘焙的Mesh进行计算，找到一条**避开障碍物的路径**，反馈给客户端。

# 快捷键

## 1 GDScript

- 注释：ctrl + k