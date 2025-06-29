---
typora-root-url: pic
---

# 基础操作

## 1 快捷键

### 1) 场景快捷键

- ctrl + d：复制actor；
- alt + 移动gizmo：复制actor到指定位置；
- ctrl + p：打开资产；

### 2) 蓝图快捷键

- home：将选中的蓝图聚焦至视口中间；
- q：多选蓝图节点，按"q"键，可将他们对齐；

## 2 降低项目默认消耗

- 取消全局光照

  <img src="/UE_cancel_gi.png" alt="UE_cancel_gi" style="zoom:80%;" />

- 设置反走样方案

  <img src="/UE_set_anti_aliase.png" alt="UE_set_anti_aliase" style="zoom:80%;" />

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

# UE蓝图

## 1 蓝图通信

​	本小节参考自：[虚幻官方：蓝图通信用法](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/blueprint-communication-usage-in-unreal-engine?application_version=5.5)。

​	主要是① 事件分发器；② 蓝图接口；③ 类型转换。

# UE GamePlay

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

# UE动画系统

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

### 1) 优化角色旋转

​	默认情况下PlayerController控制角色的移动。但PlayerController的tick周期约为0.1s，不平滑。

​	CharacterMovementComponent中处理完善了角色移动表现，通过该组件移动，会更平滑。

- 在角色蓝图中，选择Character，在详细列表的Pawn下，取消Use Controller Rotation Yaw：

  <img src="/UE_ControllerYaw.png" alt="UE_ControllerYaw" style="zoom:80%;" />

- 选择CharacerMoveComponent，在详细列表中勾选Use Controller Desired Rotation。

  此时当Contoller的朝向更新时，移动组件会用Controller新的朝向以及设置的**旋转速度**，实施一个平滑的旋转。

  <img src="/UE_UseControllerRotation.png" alt="UE_UseControllerRotation" style="zoom:80%;" />

## 3 动画重定向

### 1) 原理

- 重定向原理

  动画需要绑定到骨骼资产才能使用，骨骼资产实际是一组骨骼名称和层级数据。

  层级数据保存了骨骼的**初始比例**(来自定义该骨骼的**网格体**)，这些比例数据通常保存在骨骼的**位移数据**中。

  重定向系统只修改骨骼的位移，骨骼的旋转通常来自动画数据。

- IK Rig

  IK Rig系统提供了交互式创建解算器的方法，用于为骨骼网格体执行姿势编辑。

### 2) 手动生成重定向器

#### 2.1) 创建IK Rig资产

​	为两个骨骼网格体**创建IK Rig**。打开IK Rig，在详情页面，为对应的IK Rig设置与其**匹配**的预览骨骼网格体。

<img src="/UE_IK_Rig.png" alt="UE_IK_Rig" style="zoom:67%;" />

#### 2.2) 设置重定向根和链

​	当两个IK Rig都具备重定向链时，可通过**重定向链**来**传递动画数据**。

​	为骨骼设置重定向根骨骼，通常是**骨盆**或**臀部的骨骼**，用于成比例的定义和传输角色的根骨骼运动。

<img src="/UE_RetargetRoot.png" alt="UE_RetargetRoot" style="zoom:60%;" />

<img src="/UE_RetargetRootDetail.png" alt="UE_RetargetRootDetail" style="zoom:60%;" />

​	为所有需要重定向的肢体创建**重定向链**。

​	通过长按Shift，点选链中的骨骼，然后点击鼠标右键，选择创建重定向链，就得到一条重定向链。

<img src="/UE_RetargetChain.png" alt="UE_RetargetChain" style="zoom:50%;" />

详情页中展示了链名、链首骨骼、末端骨骼等信息。最终的重定向链，一般如下：

<img src="/UE_RetargetChain_Final.png" alt="UE_RetargetChain_Final" style="zoom:70%;" />

#### 2.3) 创建重定向器

​	创建重定向器，重定向器会统揽两个IK Rig，根据源动画，导出目标动画。

<img src="/UE_IK_Retargeter.png" alt="UE_IK_Retargeter" style="zoom:67%;" />

​	在重定向器中，设置源/目标的IK Rig资产，并检查链映射链表中源和目标链是否匹配。

<img src="/UE_ChainMapping.png" alt="UE_ChainMapping" style="zoom:60%;" />

​	检查两个骨骼是否都为T-Pose或A-Pose。若不匹配，要为重定向目标创建新的姿势，并按骨骼调整。

<img src="/UE_EditRetargetPose.png" alt="UE_EditRetargetPose" style="zoom:50%;" />

​	最终，在资产浏览器中预览动画，导出目标动画。

### 3) UE5.4自动重定向

​	自动重定向，需要使用UE支持的**常见骨骼模版**，否则只能手动重定向。

​	右键选中需要重定向的动画，选择重定向动画选项：

<img src="/UE_AutoRetargetAnim.png" alt="UE_AutoRetargetAnim" style="zoom:60%;" />

​	按下图选择目标骨骼，勾选自动生成重定向器，选择要导出的动画，导出即可：

<img src="/UE_ExportAutoRetargetAnim.png" alt="UE_ExportAutoRetargetAnim" style="zoom:50%;" />

## 4 动画混合

​	下图描述了在动画蓝图中，如何混合上半身动画和下半身动画：

<img src="/UE_BlendAnimation.png" alt="UE_BlendAnimation" style="zoom:60%;" />

## 5 Root Motion

​	Root Motion：动画控制**角色本身**运动的一种**动画机制**。

### 1) Root Motion产生的原因

​	角色动画**本质**：**胶囊体的位移**，动画在角色(胶囊体)位移时播放，使其看上去在走/跑/跳。实际拥有**运动权**角色，不是动画。

​	有些行为要表现得生动形象，需要精确且复杂的位移，很难靠程序控制。

​	综上，产生了Root Motion：**让动画拥有角色的控制权**——做动画的时候，让角色产生相应的位移。

### 2) Root Motion的实现原理

- 设置一个不呈现任何视觉表现的**Root**节点，只用于**绑定角色位置**；
- 当**角色**拥有运动控制权时，因为Root和角色绑定，Root跟随角色一起运动，从而带动动画运动；
- 当**动画**拥有运动控制权时，因为角色和Root绑定，角色会跟随动画一起运动；

### 3) 使用Root Motion

- 让**动画资产**可以使用RootMotion机制——**动画资产**中勾选EnableRootMotion：

  <img src="/UE_EnableRootMotion.png" alt="UE_EnableRootMotion" style="zoom:80%;" />

- 让角色把运动控制权交给Root Motion机制——在**动画蓝图**中，选择对应的RootMotionMode：

  <img src="/UE_RootMotionMode.png" alt="UE_RootMotionMode" style="zoom:80%;" />

  默认是Root Motion from Montages Only：仅从启动了根运动的**蒙太奇**中提取运动数据，应用到角色；

  No Root Motion Extraction：不管动画是否启动了Root Motion，角色**不提取**根运动，因此动画不会影响角色；

  Ignore Root Motion：提取根运动，但不应用到角色，因此动画不会影响角色(除非有自定义操作)；

  Root Motion from Everything：提取**每个**会构建最终姿势的动画资源的**根运动**。各个根运动数据根据构成该姿势的权重进行混合，并影响角色位置。

- 注意，只有动画蓝图或动画蒙太奇能提起根运动，并作用于角色。

### 4) RootMotion动画和InPlace动画的区别

​	RootMotion动画和InPlace动画的本质区别：**根节点是否有位移**。

- InPlace动画根节点没有位移

  InPlace动画可以使用RootMotion机制播放。播放时，角色将由于根节点位置的绑定而待在原地。

- RootMotion动画根节点有位移

  RootMotion动画按RootMotion机制播放时，角色把运动权交给动画，角色跟随动画进行位移。

  若RootMotion动画不按RootMotion机制播放，会出现下述问题：

  <img src="/UE_NoRootMotion.gif" alt="UE_NoRootMotion" style="zoom:67%;" />

  可以看到，代表角色的胶囊体没有移动，只是角色的网格发生了位移，并在动画完成后会重置。

- 总结

  分类为InPlace的动画，不应使用RootMotion机制播放；分类为RootMotion的动画，应该使用RootMotion播放。

### 5) 使用RootMotion机制可能会遇到的问题

#### 5.1) 动画的根节点在Z轴方向有位移

- 原因：这个位移和**重力**对角色在Z轴上的控制权产生**冲突**，角色将不会在Z轴上随着根骨骼进行运动。
- 解决方案：通常，动画**根节点**负责角色在**水平方向**上的移动，让脊椎末端节点跟随根节点上下移动(此时根节点的上下移动由**重力**控制)。
- 总结：动画根节点不要有Z方向的位移。

## 6 动画蒙太奇

# UE物理系统

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

## 1 C++工程

- C++工程

  <img src="/UE_CPlusPlusProject.png" alt="UE_CPlusPlusProject" style="zoom:70%;" />

- UBT(Unreal Build Tool)

  自定义的工具，负责管理和配置在不同配置下UE源代码的构建过程。﻿

  识别和处理UE的模块化架构，确保各个模块正确地编译和链接。

  扫描头文件，查找UHT相关的关键字，生成包含所有需要解析的模块和源文件的清单，供UHT使用。

- UHT(Unreal Head Tool)

  负责解析C++头文件中的特定宏，生成与反射相关的代码。

  处理UObject类的相关代码，如生成GENERATED_BODY()宏以及.generated.h文件中的相关定义。

  当头文件发生变化时，UHT会被调用来更新反射数据，确保反射系统与代码同步。

### 2) 清理编译缓存

① 在UE5.4中，删除Binaries、DerivedDataCache、Intermediate、Saved、.vs这5个文件夹以及.sln工程文件。

② 右键点击.uproject文件，点击Generate Visual Studio project files。

## 2 C++命名规则

- 类名前缀
  - U：UObject派生类，如UComponent；
  - A：AActor派生类，如APlayerController；
  - S：SWidget派生类，用于Slate UI框架控件；
  - F：大多数其他类，如FString、FVector；
  - I：抽象接口类。
- 特定前缀
  - T：模板类，如TArray、TMap；
  - E：枚举类，如ECollisionChannel；
  - G：全局对象，如GWorld。

## 3 反射系统

​	虚幻引擎的**反射系统**用大量的宏来封装C++类，为该类提供引擎和编辑器的内部功能。

​	本小节主要参考自：[虚幻引擎反射系统](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/reflection-system-in-unreal-engine?application_version=5.5)。

### 1) 反射系统中的宏和函数

#### 1.1) UCLASS宏

- UCLASS宏的作用：生成**UClass对象**

​	被UCLASS宏标记的类，UHT会解析该类的头文件，生成对应的`.generated.h`和`.gen.cpp`文件，其中包含`UClass`相关代码。

- UClass注册流程

  程序启动时，每个`UClass`通过静态构造函数注册到全局`GUObjectArray`全局对象数组中。

- UClass是UE反射系统的**核心**，它存储所有与反射相关的元数据：

  - UClass 内存布局示例：

    ```c++
    +-----------------------+
    |   UObjectBase (基类)  |
    +-----------------------+
    |   FName Name          | → 类名 (如 "AActor")
    |   UClass* SuperClass  | → 父类指针
    |   UObject* ClassDefaultObject | → 类默认对象(CDO)
    |   TArray<UProperty*>  | → 属性列表
    |   TArray<UFunction*>  | → 函数列表
    |   TMap<FName, ...>    | → 元数据字典
    +-----------------------+
    ```

  - 所有通过`UPROPERTY()`, `UFUNCTION()`, `UCLASS()`, `USTRUCT()`等标记的元数据都会存储在`UClass`及其相关的`UProperty`/`UFunction`列表中。

  - 可从类默认对象(`UObject* ClassDefaultObject`，`CDP`)中获取默认属性；

- UClass为引擎提供了如下支持：
  - 运行时类型查询和转换；
  - 使用UClass*动态创建对象和反射操作；
  - 驱动编辑器(细节面板等)和蓝图(蓝图继承C++类)或调用C++函数；

#### 1.2) GENERATED_BODY()宏

​	GENERATED_BODY()主要为**反射系统**和**序列化机制**生成必要的代码。

​	这些代码在**正式编译前**由UHT生成，保存在"xxx.generated.h"中。

​	在类的定义中，GENERATED_BODY()应被放在**类的最开始位置**，确保生成的代码能够正确地被编译器处理。

```c++
#include "MyClass.generated.h"

UCLASS()
class MYPROJECT_API UMyClass : public UObject
{   //放在类的最开始位置
    GENERATED_BODY()
public:
    //成员变量和函数
};
```

#### 1.3) UPROPERTY宏

​	在类声明中标记成员变量为UPROPERTY，使其获得引擎支持，如下所示：

```c++
UCLASS()
class MYPROJECT_API AMyActor : public AActor
{
    GENERATED_BODY()
        
    UPROPERTY()
    int32 Count;    
    
    UPROPERTY(EditAnywhere, Category="Stats")
    float Health;
};
```

​	UPROPERTY的作用流程如下：

UHT扫描代码 -> 发现UPROPERTY -> 生成反射数据 -> 写入执generated.h -> 编译器注册

#### 1.4) ATTRIBUTE_ACCESSORS

​	与 `UPROPERTY()` 配合使用，会生成下述三个方法：

- `Get{PropertyName}()`：获取属性值；
- `Set{PropertyName}(NewValue)`：设置属性值，可包含验证逻辑；
- `{PropertyName}Property()`：返回属性的元数据句柄，用于反射操作；

​	其C++原理如下：

```c++
#define ATTRIBUTE_ACCESSORS(ClassName, PropertyType, PropertyName) \
PropertyType Get##PropertyName() const; \
void Set##PropertyName(PropertyType NewValue); \
static FProperty* PropertyName##Property();
```

#### 1.5) 默认构造函数

- UE中默认构造函数由引擎的对象管理系统(如 `NewObject`、`SpawnActor`)**隐式触发**；
- 所有指定了UCLASS宏的类，必须显示指定无参的默认构造函数；
- 由于是通过反射触发的，必须在默认构造函数中显示初始化成员对象；

### 2) 反射系统中的模版

#### 2.1) TSubclassOf

​	`TSubclassOf`是一个**强类型的UClass包装模板**，它在编译时提供类型安全机制。

- `TSubclassOf`内部存储的是`UClass*`，而不是`UObject*`，如下源码所示：

  ```c++
  template <typename T>
  class TSubclassOf
  {
      ......
  private:
  	TObjectPtr<UClass> Class = nullptr;
  }
  ```

- UClass指针指向的必须是模板参数T或者T的子类；

- 当使用`TSubclassOf<AActor>`时：

  - 可以安全存储AActor的UClass；
  - 可以存储ACharacter的UClass(ACharacter继承自AActor)；
  - 但不能存储UObject的UClass，因为UObject是AActor的父类。

- 当`TSubclassOf`指定为泛型UClass时，它会执行运行时类型检查：

  ```c++
  UClass* ClassA = UDamageType::StaticClass();
  
  TSubclassOf<UDamageType> ClassB;
  ClassB = ClassA;// Performs a runtime check
  
  TSubclassOf<UDamageType_Lava> ClassC;
  ClassB = ClassC; // Performs a compile time check
  ```

### 3) 智能指针库

​	**虚幻智能指针库** 是C++11智能指针的自定义实现，旨在减轻内存分配和追踪的负担。

​	UObject的垃圾回收系统与UE的智能指针是两套**独立的机制**，但它们可以一起工作，但要循序下述原则：

- **UObject必须由GC管理**：所有派生自UObject的对象都应该通过GC系统进行生命周期管理。
- **智能指针用于非UObject对象**的生命周期管理。

#### 3.1) 共享指针TSharedPtr

- 通过引用计数**共享所有权**；
- **自动失效**；
- 弱指针中断循环引用。

```c++
// 创建一个空白的共享指针
TSharedPtr<FMyObjectType> EmptyPointer;
// 为新对象创建一个共享指针
TSharedPtr<FMyObjectType> NewPointer(new FMyObjectType());
// 从共享引用创建一个共享指针
TSharedRef<FMyObjectType> NewReference(new FMyObjectType());
TSharedPtr<FMyObjectType> PointerFromReference = NewReference;
// 创建一个线程安全的共享指针
TSharedPtr<FMyObjectType, ESPMode::ThreadSafe> NewThreadsafePointer = MakeShared<FMyObjectType, ESPMode::ThreadSafe>(MyArgs);
```

#### 3.2) 共享引用TSharedRef

​	共享引用实际为：不能为未初始化或分配为空的智能指针类型。

​	共享引用无法**重置**共享引用、向其指定空对象，或创建空白引用。因此共享引用固定包含有效对象，甚至未拥有 `IsValid` 方法。

```c++
//创建新节点的共享引用
TSharedRef<FMyObjectType> NewReference = MakeShared<FMyObjectType>();
```

​	在无有效对象的情况下尝试创建的共享引用将不会编译：

```c++
//以下两者均不会编译：
TSharedRef<FMyObjectType> UnassignedReference;
TSharedRef<FMyObjectType> NullAssignedReference = nullptr;
//以下会编译，但如NullObject实际为空则断言。
TSharedRef<FMyObjectType> NullAssignedReference = NullObject;
```

#### 3.3) 弱指针TWeakPtr

​	**弱指针**存储对象的弱引用，不会阻止其引用的对象被销毁。

​	在访问弱指针引用的对象前，应使用 `Pin` 函数生成共享指针，确保其引用的对象存在。

​	如只需要确定弱指针是否引用对象，可将其与 `nullptr` 比较，或在之上调用 `IsValid`。

- 构造

  ```c++
  //分配新的数据对象，并创建对其的强引用。
  TSharedRef<FMyObjectType> ObjectOwner = MakeShared<FMyObjectType>();
  //创建指向新数据对象的弱指针。
  TWeakPtr<FMyObjectType> ObjectObserver(ObjectOwner);
  ```

- 使用

  ```c++
  if (ObjectObserver.Pin())
  {
      //只当ObjectOwner非对象的唯一拥有者时，此代码才会运行。
      doSomething();
  }
  ```

#### 3.4) TUniquePtr

### 4) UObject

​	UE最核心的基类之一，几乎所有的UE类都**直接或间接继承自UObject**，提供了许多重要的基础功能。

​	UObject包含GC+反射+序列化，构成UE的核心框架。

​	UObject是**跨系统集成中心**：连接编辑器、蓝图、网络、物理等子系统。

#### 4.1) 功能

- 垃圾回收

  UObject只能在运行时通过NewObject构建，不能使用c++的new或delete：

  ```c++
  // 创建对象（自动加入GC系统）
  UMyObject* Obj = NewObject<UMyObject>();
  ...
  // 当 Obj 不再被引用时，GC自动回收
  ```

- 运行时类型信息

  ```c++
  // 检查对象类型
  if (Obj->IsA(UCharacter::StaticClass())) {...}
  
  // 安全类型转换
  if (UPrimitiveComponent* Prim = Cast<UPrimitiveComponent>(Obj)) {...}
  ```

- 序列化

- 网络复制

- 默认属性变化及更新

  ```c++
  // 获取 CDO 并读取默认值
  UMyObject* CDO = GetDefault<UMyObject>();
  float DefaultHealth = CDO->Health;
  ```

#### 4.2) 自定义UObject

```c++
#pragma once

#include 'Object.h'
#include 'MyObject.generated.h'

UCLASS()
class MYPROJECT_API UMyObject : public UObject
{
	GENERATED_BODY()
};
```

① #include 'MyObject.generated.h'必须是**最后一个**#include指令；

② UCLASS使`UMyObject`能被UHT识别，并生成反射信息；

③ MY_PROJECT_API让UMyObject能够被其他模块使用；

④ GENERATED_BODY()生成引擎代码。

#### 4.3) 构造UObject

- NewObject是最简单的UObject工厂方法。

  ```c++
  template< class T >	
  T* NewObject(
      UObject* Outer=(UObject*)GetTransientPackage(),
      UClass* Class=T::StaticClass()	
  )
  ```

  Outer主要用于管理对象的生命周期(通过垃圾回收)和组织对象树(用于资源管理和序列化等)：

  - **垃圾回收**中，新对象将成为Outer的子对象；
  - 当Outer被垃圾回收时，它的所有子对象(包括这个新对象)也会被回收；
  - 组织**对象层次结构**，比如一个Actor的组件，其Outer通常设置为该Actor；
  - 在**序列化**(保存/加载)时，对象树的结构依赖于Outer的层级关系；
  - 如果没有提供Outer，新对象会被视为“根对象”，只有通过显式标记或添加根引用才能避免被垃圾回收。

- NewNamedObject通过允许为新实例指定一个名称以及对象标记]和一个要指定为参数的模板对象。

  ```c++
  template< class TClass >	
  TClass* NewNamedObject(
      UObject* Outer,		
      FName Name,	
      //对象标记: 快速而简洁地描述对象。
      //各种标志来描述对象的类型、垃圾收集如何处理它、对象在其生命周期中所处的阶段等。
      EObjectFlags Flags = RF_NoFlags,	
      UObject const* Template=NULL
  )
  ```

- ConstructObject

  ConstructObject有完全灵活性。

  - 其调用 `StaticConstructObject()` ，分配对象，执行 `ClassConstructor` ；
  - 执行任何初始化：如加载配置属性，加载本地化属性和实例化组件；
  - 自UE4.16以后，`ConstructObject`已被废弃，取而代之的是`NewObject`模板函数。

#### 4.4) 销毁UObject

​	对象不被引用后，垃圾回收系统将自动进行对象销毁。这意味着没有任何`UPROPERTY`指针、引擎容器、`TStrongObjectPtr`或类实例拥有对它的强引用。

- MarkPendingKill()

  可在对象上直接调用，数将把指向对象的所有指针设为`NULL`，并从全局搜索中移除对象。对象将在下一次垃圾回收过程中被删除。

- 强引用

  强引用会将UObject保留。如果不想使用强引用，那么可使用弱指针，或者变为一个普通指针由程序员手动清除。

## 4 字符串

​	虚幻引擎支持三种核心类型的字符串：

- FString是典型的"动态字符数组"字符串类型；
- FName是对全局字符串表中**不可变**且**不区分大小写**的字符串的引用。相较于FString，它的大小更小，更能高效的传递，但更难以操控；
- FText是指定用于处理本地化的更可靠的字符串表示。

## 5 硬引用和软引用(资产加载)

​	本小节主要参考自：[资源加载(一) 硬&软引用加载资源](https://blog.csdn.net/qq_52179126/article/details/130061974)。

### 1) 硬引用(Hard Reference)

​	使用硬引用加载对象A时，若A内部引用对象B，会导致B直接被加载到内存中；若B内部引用了C，C也会被加载到内存中。

​	硬引用的加载链是递归的，可能会导致**内存迅速降低**，造成卡顿。

### 2) 软引用(Soft Reference)

​	软引用通过下述方式存在：

- 对象A通过间接机制引用对象B，如FSoftObjectPath

#### 2.1) FSoftObjectPath

​	[FSoftObjectPath的API文档在此](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/CoreUObject/UObject/FSoftObjectPath)。

​	对象A通过字符串形式的**对象路径**来引用对象B，软引用对象不会主动地将对象加载到内存中，需要**手动进行同步/异步加载**。

​	此时，软引用不存放资源本身。

​	**FSoftObjectPath**是一个简单的结构体，其内部有一个字符串包含**资源的完整名称**。

​	如果在类中添加这个类，它就会像`UObject *`属性一样显示在编辑器中。它还会正确处理烘焙和重定向：

```c++
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
FSoftObjectPath ObjectSoftRef;

//AllowedClasses主要用于在Editor中进行筛选, 不影响其在C++中的使用
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, meta = (AllowedClasses = "SkeletalMesh,StaticMesh"))
FSoftObjectPath MeshSoftRef;

//---------------- API ----------------------
//尝试通过路径下查找已加载的资产, 若资产未加载, 返回空指针
UObject* ResolveObject()

//尝试去加载资产, 该方法会调用很耗时的LoadObject()
FSoftObjectPath::TryLoad()
```

#### 2.2) FSoftClassPath

​	FSoftClassPath功能类似于FSoftObjectPath，不过FSoftClassPath专门用于对**UClass***的软引用。

#### 2.3) FSoftObjectPtr

​	本小节主要参考自：[虚幻官方文档](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/asynchronous-asset-loading-in-unreal-engine)。

​	FSoftObjectPtr是一个对UObject的弱引用，其内部包含FSoftObjectPath成员对象。

- 设置特定类的模板，可以**限制编辑器UI**仅允许选择特定类；
- 如果被引用资源存不在内存中，可调用`ToSoftObjectPath()`找到资源路径，使用同步/异步的方式加载资源。这对于需要**按需异步加载**的资产很有用；
- 不能暴露给蓝图。
- **TSoftObjectPtr**是对FSoftObjectPtr模板化，内部是基于FSoftObjectPtr二次封装。

​	其有如下接口：

```c++
//父类的接口, 检查是否没有引用当前有效的UObject;返回true, 即当前的UObject是无效的, 但未来可能被加载
bool IsPending ()

//父类接口, 检查当前是否引用有效的UObject
bool IsValid ()
```

#### 2.4) 资源弱引用类图

<img src="/UE_SoftObjectPtr.png" alt="UE_SoftObjectPtr" style="zoom:80%;" />

## 6 委托Delegate

​	本小节参考自：[一文理解透UE委托Delegate](https://zhuanlan.zhihu.com/p/460092901)。

​	UE委托的详细的API文档可参考**Delegate.h**文件，具体模版参见**DelegateCombinations.h**文件。

### 1) 本质和设计思想

​	委托是一种类型，属于C++的模版元编程，其包含了一种**类型安全的回调机制**。

- 谁发出通知：拥有委托实例的对象，即**发布者**；
- 接接收通知：其他类对象可将成员函数绑定到委托实例上；
- 何时通知：发布者调用委托实例的方法，且可携带参数。

### 2) 委托分类

#### 2.1) 单播委托

​	一次只能绑定一个函数；

​	通过下述宏，生成关联的委托类实例：

| 函数签名类型                       | 宏声明                                                       |
| ---------------------------------- | :----------------------------------------------------------- |
| void Function()                    | DECLARE_DELEGATE( DelegateName )                             |
| void Function(Param1)              | DECLARE_DELEGATE_OneParam(DelegateName, Param1Type)          |
| void Function(Param1>, Parm2)      | DECLARE_DELEGATE_TwoParams(DelegateName, Param1Type, Param2Type) |
| void Function(Param1,  Parm2, ...) | DECLARE_DELEGATE_\<Num\>Params(DelegateName, Param1Type, Param2Type, ...) |
| RetVal Function()                  | DECLARE_DELEGATE_RetVal(RetValType, DelegateName )           |
| RetVal Function(Parm1)             | DECLARE_DELEGATE_RetVal_OneParam(RetValType, DelegateName, Param1Type) |
| RetVal Function(Parm1, Parm2)      | DECLARE_DELEGATE_RetVal_TwoParams(RetValType, DelegateName, Param1Type, Param2Type) |
| RetVal Function(Param1,Parm2, ...) | DECLARE_DELEGATE_RetVal_\<Num\>Params(RetValType, DelegateName, Param1Type, Param2Type, ...) |

​	委托类中，设定了很多成员函数，用来绑定不同类型的函数。

#### 2.2) 多播委托

​	可以绑定多个函数。广播时，所有绑定的函数会按**绑定顺序**依次执行。

```c++
//定义void Function()的多播委托
#define DECLARE_MULTICAST_DELEGATE( DelegateName ) FUNC_DECLARE_MULTICAST_DELEGATE( DelegateName, void )

//定义void Function(Param1)的多播委托
#define DECLARE_MULTICAST_DELEGATE_OneParam( DelegateName, Param1Type ) FUNC_DECLARE_MULTICAST_DELEGATE( DelegateName, void, Param1Type )

//其余和单播类似, 不再赘述
......
```

#### 2.3) 动态委托

​	一种**特殊的单播/多播委托**，主要用于**在蓝图和 C++ 之间通信**。

​	动态委托本质集成了**UObject的反射系统**，让其可支持序列化存储到本机和蓝图操作。

​	动态委托概念上与前两类无异，性能和功能弱于前两类。

# UE GAS系统

​	GAS即GamePlay Ability System。

## 1 属性集AttribueSet

​	AttributeSet是**FGameplayAttributes**结构体的集合，它通过**AbilitySystemComponent**注册，设置为Actor的**子对象**。

### 1) 架构和使用

<img src="/UE_AttributeSet.png" alt="UE_AttributeSet" style="zoom:80%;" />

​	下述是AttrbuteSet实现示例：

```c++
UCLASS()
class UHealthSet : public UAttributeSetBase 
{
	GENERATED_BODY()
public:
	UHealthSet();

	//最大生命值
	UPROPERTY(BlueprintReadOnly)
	FGameplayAttributeData MaxHealth;
	ATTRIBUTE_ACCESSORS(UHealthSet, MaxHealth);
	//生命恢复速度
	UPROPERTY(BlueprintReadOnly)
	FGameplayAttributeData HealthRegenRate;
	ATTRIBUTE_ACCESSORS(UHealthSet, HealthRegenRate);
};
```

​	下述的Character，向AbilityComponentSystem中注册了AttributeSet：

```c++
#include "RLCharacterBase.h"
#include "AbilitySystemComponent.h"

// Sets default values
ARLCharacterBase::ARLCharacterBase() {
    PrimaryActorTick.bCanEverTick = true;
	if (bUseGAS) 
    {
		//创建ASC组件
		AbilitySystemComponent = 
            CreateDefaultSubobject<UAbilitySystemComponent>(TEXT("AbilitySystemComponent"));
	}
}

UAbilitySystemComponent* ARLCharacterBase::GetAbilitySystemComponent() const {
	return AbilitySystemComponent;
}

void ARLCharacterBase::BeginPlay() 
{
	if (bUseGAS) 
    {
		for (auto Element : AttributeSetClasses) 
        {
			UAttributeSet* AttributeSet = NewObject<UAttributeSet>(this, Element);
			AbilitySystemComponent->AddSpawnedAttribute(AttributeSet);
		}
	}
	Super::BeginPlay();
}
```

### 2) GameEffect

​	自定义GameEffect，制定规则，用于修改AbilitySystemComponent中某个属性集的特定属性。

<img src="/UE_GameEffect.png" alt="UE_GameEffect" style="zoom:90%;" />

# UE架构

​	本小节参考自：[虚幻官方：虚幻架构](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/programming-in-the-unreal-engine-architecture?application_version=5.5)。

## 1 UE中的主要线程

- 游戏线程(GameThread)

  承载游戏逻辑、运行流程的工作，也是其它线程的数据发起者；

  在FEngineLoop::Tick 函数执行每帧逻辑的更新；

  引擎启动时会把 GameThread 的线程 id 存储到全局变量GGameThreadId 中，稍后会设置到 TaskGraph 系统中。

- 渲染线程(RenderThread)

  RenderThread 在 TaskGraph 系统中有一个任务队列，其他线程（主要是GameThread）通过宏 ENQUEUE_RENDER_COMMAND 向该队列中填充任务；

  RenderThread 不断从这个队列中取出任务来执行，从而生成与**平台无关**的 **Command List**（渲染指令列表）。

- RHI线程(Render Hardware Interface Thread)

  RenderThread 作为前端(**frontend**)产生的 Command List 是平台无关的，是抽象的图形 API 调用；

  RHIThread 作为后端(**backend**)会转换Command List 为指定图形 API 的调用(Graphical Command)，提交到 GPU 执行。

  RHI 线程的工作是转换渲染指令到指定图形 API，创建、上传渲染资源到 GPU。
