---
typora-root-url: pic
---

# 基础操作

## 快捷键

场景

- ctrl + d：复制actor；
- alt + 移动gizmo：复制actor到指定位置；
- ctrl + p：打开资产；

蓝图

- home：将选中的蓝图聚焦至视口中间；

## 降低项目默认消耗

- 取消全局光照

  <img src="/UE_cancel_gi.png" alt="UE_cancel_gi" style="zoom:80%;" />

- 设置反走样方案

  <img src="/UE_set_anti_aliase.png" alt="UE_set_anti_aliase" style="zoom:80%;" />

# 蓝图基础

## 1 基础概念

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

## 2 命名规则

<img src="/UE_naming_rule.png" alt="UE_naming_rule" style="zoom:70%;" />

## 3 蓝图类

​	如下步骤可为常见中的Actor创建一个蓝图类：

<img src="/UE_create_bp_class.png" alt="UE_create_bp_class" style="zoom:60%;" />

### 1) 蓝图函数

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

### 2) 成员函数

​	成员函数不在关卡的蓝图上，而在某个特定的**蓝图类**中。蓝图类的成员函数对面向对象编程很重要。

​	每个蓝图成员函数都有一个参数，**该参数是要调用该函数的蓝图类实例**。

- Self：A node available in member function, always points to the current instance。Self是个节点，用于在成员函数中，获取当前实例。
- 创建成员函数：
  <img src="/UE_make_member_function.png" alt="UE_make_member_function" style="zoom:60%;" />

### 3) 蓝图函数库

​	将函数放到蓝图函数库(一种资产)中，让每个蓝图都能访问。

# UE基础

## 1 坐标系和单位

​	UE的坐标系是**左手系**，且每个坐标轴的意义和普通的坐标系不太一致。

- X轴朝前，默认屏幕向外。
- Y轴朝右，默认屏幕左侧。
- Z轴朝上(z-up)，默认屏幕上方。
- UE默认长度单位是**厘米**。

<img src="/UE_coordinate.png" alt="UE_coordinate" style="zoom:50%;" />

## 2 UE多线程

### 1) 传统API的多线程特性

​	DirectX11尝试从硬件层面解决多线程渲染的问题。它支持了两种设备上下文：**即时上下文(Immediate Context)**和**延迟上下文(Deferred Context)**。

​	不同的延迟上下文可以同时在不同的线程中使用，生成将在即时上下文中执行的命令列表。这种多线程策略允许将复杂的场景分解成并发任务。

​	此外，延迟上下文在**某些驱动**的支持下，可实现硬件级的加速，而**不必在即时上下文执行Command List**。

​	不过，由于硬件层面的加速不是必然支持的，所有在Deferred Context记录的指令和在Immediate Context的指令必须由同一个线程(通常是渲染线程)提交给GPU执行。

​	这种非硬件支持的多线程渲染只是节省了部分CPU时间(多线程录制指令和绘制同步等待)，并不能从硬件层面真正发挥多线程的威力。

<img src="/UE_TranditionalAPIThreads.png" alt="UE_TranditionalAPIThreads" style="zoom:75%;" />

### 2) UE的线程模型

​	默认情况下，UE存在游戏线程(Game Thread)、渲染线程(Render Thread)、RHI线程(RHI Thread)，它们都独立地运行在专门的线程上(FRunnableThread)。

​	游戏线程通过某些接口向渲染线程的Queue入队指令。渲染线程执行时，从Queue获取指令，一个个地执行，从而生成了Command List；

​	渲染线程作为前端(frontend)产生的Command List是平台无关的，是抽象的图形API调用；

​	RHI线程作为后端(backtend)会执行和转换渲染线程的Command List成为指定图形API的调用(称为Graphical Command)，并提交到GPU执行。

​	这些线程处理的数据通常是不同帧的，譬如游戏线程处理第N帧数据，渲染线程和RHI线程处理第N-1帧数据。渲染线程不能落后游戏线程一帧，否则游戏线程会卡住，直到渲染线程处理所有指令。

​	渲染指令是可以**并行**地被生成，RHI线程也可以**并行**地转换这些指令。

<img src="/UE_ThreadModel.png" alt="UE_ThreadModel" style="zoom:65%;" />

## 3 延迟渲染管线

​	UE默认使用延迟渲染管线。

​	其核心思想是将场景的物体绘制分离成两个Pass：**几何Pass**和**光照Pass**。

​	目的是将计算量大的光照Pass延后，和物体数量、光照数量解耦，以提升着色效率。

​	在目前的主流渲染器和商业引擎中，有着广泛且充分的支持。

<img src="/UE_DefferedRendering.png" alt="UE_DefferedRendering" style="zoom:90%;" />

### 1) 通用延迟渲染管线简述

- 几何通道

  这个阶段将场景中所有的**不透明物体**(Opaque)和**Masked物体**用无光照(Unlit)的材质执行渲染，然后将物体的几何信息写入到对应的渲染纹理(Render Target)中。

  其中物体的几何信息有：

  ① 位置(Position，也可以是深度，因为只要有深度和屏幕空间的uv就可以重建出位置)。

  ② 法线(Normal)。

  ③ 材质参数(Diffuse Color, Emissive Color, Specular Scale, Specular Power, AO, Roughness, Shading Model, ...)。

  通用的伪代码如下：

  ```c++
  void RenderBasePass()
  {
      SetupGBuffer(); // 设置几何数据缓冲区。
      
      // 遍历场景的非半透明物体
      for each(Object in OpaqueAndMaskedObjectsInScene)
      {
          SetUnlitMaterial(Object);    // 设置无光照的材质
          DrawObjectToGBuffer(Object); // 渲染Object的几何信息到GBuffer，通常在GPU光栅化中完成。
      }
  }
  ```

- 光照通道

  利用几何通道阶段生成的GBuffer数据执行光照计算。

  核心逻辑是遍历渲染分辨率数量相同的像素，根据每个像素的UV坐标从GBuffer采样获取该像素的几何信息，从而执行光照计算。

  伪代码如下：

  ```c++
  void RenderLightingPass()
  {
      BindGBuffer(); // 绑定几何数据缓冲区。
      SetupRenderTarget(); // 设置渲染纹理。
      
      // 遍历RT所有的像素
      for each(pixel in RenderTargetPixels)
      {
          // 获取GBuffer数据。
          pixelData = GetPixelDataFromGBuffer(pixel.uv);
          // 清空累计颜色
          color = 0;    
          // 遍历所有灯光，将每个灯光的光照计算结果累加到颜色中。
          for each(light in Lights)
          {
              color += CalculateLightColor(light, pixelData);
          }
          // 写入颜色到RT。
          WriteColorToRenderTarget(color);
      }
  }
  ```

- 延迟渲染的优势

  由于最耗时的光照计算延迟到后处理阶段，所以跟场景的物体数量解耦，只跟Render Targe的尺寸相关，复杂度是：$O(N_{light}×W_{RT}×H_{RT})$。

  延迟渲染在应对带有复杂光源的场景比较得心应手，往往能得到非常好的性能提升。

  但是，也存在一些**缺点**：如需多一个通道来绘制几何信息，需要多渲染纹理（MRT）的支持，更多的CPU和GPU显存占用，更高的带宽要求，有限的材质呈现类型，难以使用MSAA等硬件抗锯齿，存在画面较糊的情况等等。

  此外，应对简单场景时，可能反而得不到渲染性能方面的提升。

### 2) 延迟渲染管线的变种

#### 2.1) Tiled-Based Deffered Rendering

​	基于瓦片的渲染，简称**TBDR**，它的核心思想在于将渲染纹理分成规则的一个个四边形(称为Tile)，然后利用四边形的包围盒剔除该Tile内无用的光源，只保留有作用的光源列表，从而减少了实际光照计算中的无效光源的计算量。

<img src="/UE_TBDR.png" alt="UE_TBDR" style="zoom:80%;" />

① 将渲染纹理分成一个个均等面积的小块；

② 根据Tile内的Depth范围计算出其Bounding Box，如下：

<img src="/UE_TBDRBox.png" alt="UE_TBDRBox" style="zoom:70%;" />

③ 根据Tile的Bounding Box和Light的Bounding Box，执行求交；

④ 摒弃不相交的Light，得到对Tile有作用的Light列表；

⑤ 遍历所有Tile，获取每个Tile的有效的光源索引列表，计算该Tile内所有像素的光照结果。

​	由于TBDR可以摒弃很多无作用的光源，能够避免很多无效的光照计算，目前已被广泛采用与移动端GPU架构中，形成了**基于硬件加速的TBDR**。

<img src="/UE_TBDRHardwareAcc.png" alt="UE_TBDRHardwareAcc" style="zoom:80%;" />

### 3) UE延迟渲染管线中的Pass

- PrePass

  PrePass又被称为提前深度Pass、Depth Only Pass、Early-Z Pass，用来渲染不透明物体的深度。

  此Pass只会写入深度而不会写入颜色，写入深度时有disabled、occlusion only、complete depths三种模式，视不同的平台和Feature Level决定。

- BasePass

  UE的BasePass就是延迟渲染里的几何通道，用来渲染不透明物体的几何信息，包含法线、深度、颜色、AO、粗糙度、金属度等等，这些几何信息会写入若干张GBuffer中。

- LightingPass

  UE的LightingPass就是前面章节所说的光照通道。

  此阶段会计算开启阴影的光源的阴影图，也会计算每个灯光对屏幕空间像素的贡献量，并累计到Scene Color中。

  此外，还会计算光源也对translucency lighting volumes的贡献量。

  Lighting Pass的负责的渲染逻辑多而杂，包含间接阴影、间接AO、透明体积光照、光源计算、LPV、天空光、SSS等等，但光照计算的核心逻辑在`RenderLights`中。

- Translucency

  Translucency是渲染半透明物体的阶段，所有半透明物体在视图空间由**远到近逐个绘制到离屏渲染纹理**(separate translucent render target)中，接着用单独的pass以正确计算和混合光照结果。

- PostProcessing

  后处理阶段，也是`FDeferredShadingSceneRenderer::Render`的最后一个阶段。

  包含了不需要GBuffer的Bloom、色调映射、Gamma校正等以及需要GBuffer的SSR、SSAO、SSGI等。

  此阶段会将半透明的渲染纹理混合到最终的场景颜色中。

## 4 UE的材质

​	本小节参考自：[UE：材质系统](https://mp.weixin.qq.com/s?__biz=MzA5MDcyOTE5Nw==&mid=2650549692&idx=1&sn=d23db44e95de518437a4f90dff057baf&chksm=880fb23ebf783b2860456c2dd3104236d47b0ecf562a058f4f75096f12580291a77b24b35626&scene=178&cur_album_id=2518511104424198145&search_click_id=#rd)。

​	UE的材质定义为：Controlling the appearance of surfaces in the world using shaders.

​	材质的三大要素为：

- UMaterial类：对应材质编辑器中的资源属性。
- FMaterialResource类：依据`RHIFeatureLevel`(DirectX/Vulkan/Metal/OpenGL ES等)和材质质量`EMaterialQualityLevel`(高/中/低)，将UMaterial生成HLSL代码。
- FMaterialRenderProxy类：将编译后的shader传递给渲染层，通过材质函数完成渲染结果。

​	其中，UMaterial面向艺术家，FMaterialResource面向机器，FMaterialRenderProxy产生最终显示效果，面向人。

​	注意，UMaterial : FMaterialResource : FMaterialRenderProxy是1 : N : 1的关系。

# GamePlay Framework

​	GamePlay框架是虚幻引擎为开发者提供的工具箱：

- 是用于构建游戏基础的类的集合。
- 提供游戏的核心架构，如角色控制、物理交互、网络同步等。
- 通过添加组件或继承现有的类，快速实现复杂的游戏对象。

## 1 简介

### 1) GameInstance

<img src="/UE_GameInstance.png" alt="UE_GameInstance" style="zoom:50%;" />

- 特殊的实例，在引擎启动时创建，并一直保持活动状态，直到引擎关闭；
- GameInstance是一个**管理类Actor**，在游戏世界中没有实体表现，只用于跟踪数据和运行代码；
- GameInstance不能被复制，一个运行的游戏，对应唯一一个游戏实例；
- 在不同关卡间需要访问的数据，应保存在GameInstance中；
- 游戏实例会创建、管理一些子系统。

### 2) GameMode

- GameMode是**管理类Actor**，但不是永久的Actor，是GamePlay框架的**核心**；
- GameMode在加载关卡Level的时候创建，事实上它是关卡中**第一个**被实例化的Actor；
- GameMode的**生命周期**限制于关卡中，可在WorldSetting中**重载**每个关卡对应的GameMode；
- 它是游戏的核心，定义了游戏的规则和行为；在创建时，实例化了**其他的框架Actor**，告诉引擎要使用哪些框架Actor；

- GameMode负责把HUD、Pawn、Controller这些分散的东西整理在一起，如下所示：

  <img src="/UE_GameModeAssemble.png" alt="UE_GameModeAssemble" style="zoom:40%;" />

### 3) GameState和PlayerState

<img src="/UE_PlayerState&&GameState.png" alt="UE_PlayerState&&GameState" style="zoom:50%;" />

​	GameState和PlayerState是非实体的管理Actor，分别用于跟踪游戏的整体状态和玩家状态。

​	它们会复制到服务器和所有玩家客户端上，提供高效、可供网络访问的数据存储库。

- GameState：包含与**所有玩家**相关的数据和逻辑
  - 每个关卡中只有一个GameState，由GameMode创建。

- PlayerState：处理与**关联玩家**相关的数据和逻辑
  - 每当玩家加入游戏，或进入关卡时，都会为玩家创建PlayerState。

### 4) Player

​	一个Player在实例化时，会关联两个**Actor：Pawn**和**Controller**，如下：

<img src="/UE_Player.png" alt="UE_Player" style="zoom:50%;" />

#### 4.1) Pawn

<img src="/UE_Pawn.png" alt="UE_Pawn" style="zoom:50%;" />

- Pawn
  - 角色在游戏世界的物理实体，可以和游戏环境交互，如玩家、敌人、载具等等。
  - 最简单的Pawn实现为Default Pawns。
  - Default Pawns有三个重要组件：碰撞、网格体和移动。
  - Spectator Pawns(旁观者Pawn)：本质和Default Pawn一样，但会渲染一台看不见的悬浮相机。
- Character：Pawn重要的子类
  - 包含移动逻辑的特殊Pawn，移动逻辑由CharacterMovementComponent提供。
  - 支持网络同步
  - 包含组件：CapsuleComponent、CharacterMoveComponent、SkeletalMeshComponent、ArrowComponent、SpringArmComponent、CameraComponent。
  - CharacterMoveComponent：为Pawn的移动奠定了基础，支持网络复制。

#### 4.2) Controller

<img src="/UE_Controller.png" alt="UE_Controller" style="zoom:50%;" />

​	Controller是管理类Actor，可以控制Pawn，决定了Pawn的行为逻辑，即Pawn可以被Controller占有(Possess)。

​	即使Pawn消亡了，Controller也可一直存在。

​	Controller有两个关键子类：**PlayerController**和**AIController**。

- PlayerController
  - 控制用户输入；
  - 处理和玩家Pawn有关的游戏逻辑。
- AIController
  - 不代表玩家，不处理用户输入；
  - 只存在于服务器中；
  - 可高度依赖UE提供的行为树系统、导航系统等。

### 5) Actor和Component

#### 5.1) Actor

- 场景中的一个实体，可以满足网络同步需求；
- 通常包含一个或多个ActorComponent，赋予其特定的功能；
- Actor本身不定义位置和旋转，其位置和旋转依赖于**RootComponent**。

#### 5.2) ActorComponent

​	组件：可添加到Actor的可重复利用的功能，包含很多函数和事件。

​	组件之间是有**父子关系**的。当你调整父组件的属性时，子组件也会跟随变化。

# 动画蓝图

## 1 创建动画蓝图角色

- 使用蓝图类Character作为基类，创建角色蓝图基类BP_CharacterBase。

  再以BP_CharacterBase为基类，创建角色子类，如BP_Ninja：
  <img src="/UE_BP_Character.png" alt="BP_Character" style="zoom:80%;" />

- 打开角色蓝图，更新网格体；根据胶囊体更细网格位置和胶囊体大小；根据前进方向调整网格朝向。

- 创建混合空间，设置动画混合。

- 创建动画蓝图，将上一步创建的混合空间，设置给蓝图，作为动画输出：

  <img src="/UE_ABP_Character_AnimFlow.png" alt="ABP_Character_AnimFlow" style="zoom:80%;" />

- 在动画蓝图的EventGraph中，设置驱动动画蓝图的逻辑：

  <img src="/UE_ABP_EventGraph.png" alt="ABP_EventGraph" style="zoom:90%;" />

- 在角色蓝图中，将角色的动画模式设置为：使用蓝图，并设置刚创建的动画蓝图。

  ![UE_UseAnimationBP](/UE_UseAnimationBP.png)

## 2 动画优化

### 1) 优化AI旋转

- 原因

  默认，AI的旋转由Controller决定。Controller的tick周期约为0.1s，因此并不平滑：

  <img src="/UE_ControllerYaw.png" alt="UE_ControllerYaw" style="zoom:80%;" />

- 解决方案

  **取消上述勾选**，在角色蓝图中，选择如下:

  <img src="/UE_UseControllerRotation.png" alt="UE_UseControllerRotation" style="zoom:80%;" />

  其意思是：当Contoller的朝向更新时，角色蓝图会用设置的**旋转速度**和**新的朝向**，实施一个平滑的旋转。

# 物理系统

## 1 三种碰撞响应

​	UE中物理系统的碰撞响应有三种类型：**阻挡**、**重叠**、**忽略**。三者的响应关系如下：

- 若两个物体碰撞类型**都是阻挡**，则会进行**物理模拟**；
- 若一个物体是重叠，另一个无论是**重叠/阻挡**，会产生**碰撞事件**，但不会进行物理反馈；
- 若一个物体是忽略，另一个无论是**忽略/重叠/阻挡**，不会产生碰撞事件，也不会进行物理反馈。

## 2 自定义对象类型和碰撞预设

- 创建对象类型：

  <img src="/UE_ObjectChannel.png" alt="UE_ObjectChannel" style="zoom:90%;" />

- 创建碰撞预设：

  <img src="/UE_CollisionPreset.png" alt="UE_CollisionPreset" style="zoom:70%;" />

- 应用到角色蓝图：

  <img src="/UE_BP_Character_Collsion_Preset.png" alt="UE_BP_Character_Collsion_Preset" style="zoom:80%;" />

  在角色蓝图中，胶囊体组件和网格体组件都有碰撞预设。

  将Mesh的碰撞预设设置为None，把外部胶囊体的碰撞预设设置为目标类型，从而可以节约一点性能。

# UE C++

## 1 基础

### 1) C++工程编译缓存清理

① 在UE5.4中，删除Binaries、DerivedDataCache、Intermediate、Saved、.vs这5个文件夹以及.sln工程文件。

② 右键点击.uproject文件，点击Generate Visual Studio project files。

### 2) 字符串

- L前缀
  C++字符串前加L表示该字符串是**Unicode字符串**。
  L"我的字符串"表示将ANSI字符串转换成unicode的字符串，即**每个字符占用两个字节**。

  ```c++
  strlen("asd") = 3;
  strlen(L"asd") = 6;
  ```

- UE的TEXT宏
  TEXT("my str")将括号内的字符串转换为特定的字符集，默认实现为：

  ```c++
  #define WIDETEXT_PASTE(x)  L ## x
  //省略了中间的转换
  // ##表示连接, 即在字符串前添加L前缀
  ```

- FString：UE封装的动态字符串。
