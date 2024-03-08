## 数学基础

### 1 叉乘

#### 1) 叉乘的定义

​	向量叉乘的结果由两个属性定义：

- 模长：$|\vec{a} \times \vec{b}| = |\vec{a}| \cdot |\vec{b}| \cdot sin\theta$，其中$\theta$是两个向量的夹角，$0 \leq \theta \leq 180$。
- 方向：$\vec{c}= \vec{a} \times \vec{b}$的方向与$\vec{a}$和$\vec{b}$所在平面垂直，且遵守`右手法则`：当右手的手指从$\vec{a}$以不超过180°的角度旋转到$\vec{b}$时，竖起的大拇指方向则是$\vec{c}$的方向。

#### 2) 补充性质

​	二维平面上，有向量$\vec{P}=(x_{1},y_{1})$和向量$\vec{Q}=(x_{2},y_{2})$，则有$\vec{P}\times\vec{Q}=(x_{1}y_{2} - x_{2}y_{1})$。

​	若$\vec{P}\times\vec{Q} > 0$，$\vec{P}$在$\vec{Q}$的顺时针方向(右侧)；

​	若$\vec{P}\times\vec{Q} < 0$，$\vec{P}$在$\vec{Q}$的逆时针方向(左侧)；

​	若$\vec{P}\times\vec{Q} = 0$，$\vec{P}$和$\vec{Q}$在同一条直线上，可能同向，也可能异向。

## 矩阵

### 1 齐次坐标

​	向量：$\vec{v} = \begin{pmatrix}a\\b\\c\\0\end{pmatrix}$		点：$p = \begin{pmatrix}a\\b\\c\\1\end{pmatrix}$

​	用4个代数分量表示3D几何概念的方式是一种齐次坐标表示。

​	齐次坐标表示是计算机图形学的重要手段之一，它既能够用来明确**区分向量和点**，同时也**更易于进行仿射(线性)几何变换**。

<img src="./pic/cg_matrix_0.png" alt="cg_matrix_0" style="zoom:100%;" />

​	从上述矩阵变换可知，平移变换只对于点才有意义，因为普通向量没有位置概念，只有大小和方向。

​	旋转和缩放对于向量和点都有意义，可用类似上面齐次表示来检测。因此，齐次坐标使仿射变换更方便。

### 2 渲染管线

​	图形渲染管线：原始图形数据途经输送管道，经过各种变化处理最终出现在屏幕的过程。

​	在概念上可以分为四个阶段：**应用程序阶段**、**几何阶段**、**光栅化阶段**和**像素处理阶段**。

<img src="./pic/cg_pipeline_1.png" alt="cg_pipeline_1" style="zoom:80%;" />

​	应用阶段通常是在CPU端进行处理，包括碰撞检测、动画物理模拟以及视椎体剔除等任务，这个阶段会将数据送到渲染管线中；

​	顶点着色器、投影变换、曲线细分、几何着色器、图元装配、裁剪、屏幕映射；

​	光栅化将图元离散化片段的过程；

​	像素处理阶段包括像素着色和混合的功能。如下图所示：

<img src="./pic/cg_pipeline.png" alt="cg_pipeline" style="zoom:85%;" />

#### 1) 管线概述

- **图元组装**

​	图元组装将输入的顶点组装成指定的图元。

​	图元组装阶段会进行**裁剪**和**背面剔除**相关的优化，以减少进入光栅化的图元的数量，加速渲染过程。

​	在光栅化之前，还会进行屏幕映射的操作：**透视除法**和**视口变换**。

- **光栅化**

​	经过图元组装以及屏幕映射后，物体坐标被变换到了**窗口坐标**。

​	光栅化是个**离散化**的过程，将3D连续的物体转化为离散屏幕像素点的过程。

​	光栅化会确定图元所覆盖的片段，利用顶点属性**插值**得到片段的属性信息，然后送到片段着色器进行颜色计算。

​	注意，**片段是像素的候选者**，只有通过后续的测试，片段才会成为最终显示的像素点。

- **片段着色器**

​	片段着色器用来决定屏幕上像素的最终颜色，可能会进行大量效果计算(光照和阴影)。

- **测试混合**

​	管线的**最后一个阶段**是测试混合阶段。测试包括裁切测试、Alpha测试、模板测试和深度测试(**late-z**)。

​	没有经过测试的片段会被丢弃，不需要进行混合阶段；经过测试的片段会**进入混合阶段**。

​	注意，绘制半透明物体最好遵循**画家算法(painter algorithm)**，将物体排序，**由远及近**进行绘制。因为半透明物体的绘制顺序会影响混合的结果。	

<img src="./pic/cg_painter_algothimn.png" alt="cg_painter_algothimn" style="zoom:75%;" />

#### 2) 对late-Z的优化

​	像素处理阶段，片元被着色后(fragmet shader)，通过深度测试，才转换为像素，显示在屏幕上。

​	因此，被着色后的片元有可能被舍弃，这就引起了**过渡绘制(OverDraw)**。

​	下述几个方案都是为了优化过渡绘制，提升效率。

##### 2.1) early-z

​	early-z是GPU硬件层的优化，在**光栅化后**和**片元着色前**添加early-z阶段。

​	early-z执行的操作和late-z完全一样，但early-z的优化效果**不稳定**。

###### 2.1.1) early-z被关闭

​	① 手动**写入深度值**；② **开启alpha test**；③ **执行丢弃像素操作**。

​	若执行上述操作，GPU就会关闭early-z，直到下一次clear z-buffer。

​	这些操作在**片元着色和late-z之间执行**，会修改z-buffer中的值，导致early-z的结果不正确。

###### 2.1.2) early-z不稳定

​	若按由近及远的顺序绘制，early-z可以完美避免过度绘制；

​	若由远及近绘制，early-z不起任何作用。

##### 2.2) z-culling

​	z-culling是GPU硬件层的优化。

###### 2.2.1) z-culling和early-z的区别

​	early-z以pixel-quad为单位，**逐像素**比较；

​	z-culling以tile为单位，按tile**整体**进行比较。其中，tile即tile based rendering(TBR)中的概念。

###### 2.2.2) z-culling比较方式

​	获取当前tile的深度最值：$Z^{tile}_{min}$、$Z^{tile}_{max}$；

​	获取tile所属的深度缓冲区的最值：$Z_{min}$、$Z_{max}$；

​	若$Z^{tile}_{max} < Z_{min}$，则tile全部可见，保留整个tile，并在late-z阶段省去读缓冲的操作，直接进行写buffer；

​	若$Z^{tile}_{min} > Z_{max}$，则tile全部不可见，丢弃整个tile；

​	对于其他情况，则交给后续的late-z判断。

​	z-culling所需要的比对数据储存在on-chip缓存中的某个固定区域，特点即是容量小但速度快。

​	由于在z-culling阶段，对深度缓存是只读的，所以**不会**因为① 手动写入深度值；② 开启alpha test；③ 执行丢弃像素操作，导致z-buffer修改，导致测试失效。因此z-culling弥补了early-z的第一个缺点。

##### 2.3) z-prepass

​	z-perpass是软件层的技术，主要配合early-z使用，优化early-z的第二个缺点：不稳定。

​	z-prepass思路类似延迟渲染：使用**两个pass**，第一个pass渲染时**只写入深度**；第二个pass关闭深度写入，设置深度比较函数为“**相等**”。

​	z-prepass**必须配合early-z使用**。若和late-z配合使用，那么无用的片元仍然会经历片元着色，造成大量计算的浪费。

### 3 坐标变换流程

<img src="E:\Code\0_personal\MyDocuments\Notes\pic\cg_coordinate.png" alt="cg_coordinate" style="zoom:90%;" />

​	$局部坐标\stackrel{(1)模型矩阵}{\longrightarrow}世界坐标 \stackrel{(2)相机矩阵}{\longrightarrow} 视图坐标 \stackrel{(3)投影矩阵}{\longrightarrow} 裁剪坐标 \stackrel{(4)透视除法}{\longrightarrow} 标准设备空间(NDC) \stackrel{(5)视口变换}{\longrightarrow} 窗口坐标$

​	其中，第三步透视变换包含两个步骤：$ 投影矩阵=\left\{ \begin{matrix} 相似变换 \\ X、Y、Z三轴线性插值至[-1,1]\end{matrix} \right.$

<img src="./pic/cg_matrix_flow.png" alt="cg_matrix_flow" style="zoom:100%;" />

### 4 相机变换

### 5 透视变换

​	世界空间的点经过相机矩阵后，被转换到相机空间。此时，多边形可能会被视椎体裁剪，但在不规则体中裁剪很难，所以裁剪被安排到规则观察体(`Canonical View Volume, CVV`)中(`齐次裁剪空间`)。

​	CVV是一个正方体，其x, y, z的范围都是[-1, 1]，多边形裁剪就是利用这个规则体完成的。综上，透视变换的作用如下：

- 透视变换矩阵把相机空间中的顶点从视锥体中变换到裁剪空间的CVV中(`相似变换`)；
- CVV裁剪完成后进行透视除法。

<img src="./pic/cg_matrix_1.png" alt="cg_matrix_1" style="zoom:100%;" />

#### 1) 投影点坐标

<img src="./pic/cg_matrix_2.png" alt="cg_matrix_2" style="zoom:100%;" />

​	上图是右手坐标系中顶点在相机空间中的情形。设p(x,z)是经过相机变换之后的点，视锥体由eye——眼睛位置，np——近裁剪平面，fp——远裁剪平面组成。N是眼睛到近裁剪平面的距离，F是眼睛到远裁剪平面的距离。这里选择**近裁剪面**作为投影面。

​	设p’(x’,z’)是投影之后的点，则有z’ = -N。通过相似三角形性质，有：

$\frac{x'}{x} = \frac{z'}{z}$，其中有$z' = -N$，所以有$\frac{x'}{x} = \frac{-N}{z}$，有$x' = -N\frac{x}{z}$

​	同理有$y' = -N\frac{y}{z}$，因此投影点坐标为 $p'=(-N\frac{x}{z}, -N\frac{y}{z}, -N)$。

​	从上面可以看出，投影的结果z’始终等于-N。实际上，z’对于投影后的p’已经没有意义了，这个信息点已经没用了。

​	但对于3D图形管线来说，为了便于进行后面的片元操作，例如z缓冲消隐算法，有必要把投影前`相机空间中的z`保存下来，方便后面使用。因此，我们利用这个没用的信息点存储z：

$p'=(-N\frac{x}{z}, -N\frac{y}{z}, z)\qquad\qquad\qquad\qquad\qquad\qquad\qquad\qquad\qquad\qquad\qquad(1)$

​	公式(1)其中x, y, z是相机空间点的坐标，N是近平面的距离。

#### 2) 用齐次坐标表达投影点

​	公式(1)有点生搬硬套的意思，现开始结合CVV进行思考，把它写得在数学上更优雅，更易于程序处理。如前所述，第3个分量可以是任意值，因此有公式(2)：

$p'=(-N\frac{x}{z}, -N\frac{y}{z}, -\frac{az+b}{z})\qquad\qquad\qquad\qquad\qquad\qquad\qquad\qquad\qquad\qquad(2)$

<img src="./pic/cg_matrix_3.png" alt="cg_matrix_3" style="zoom:100%;" />其中，<img src="./pic/cg_matrix_4.png" alt="cg_matrix_4" style="zoom:100%;" />

​	步骤一：投影矩阵乘法：首先进行`相似变换`，将点投影到裁剪平面；其次，使用`线性插值`进行`坐标归一化`，使[left, right]、[bottom, far]、[-N, -F]之间的点变为[-1, 1]，方便裁剪；

​	步骤二：CVV中执行裁剪；裁剪时，使用的是齐次坐标，因为透视除法后会丢失一些必要信息，如-z。

​	步骤三：透视除法，将齐次坐标变换为普通坐标。

​	将第三个分量写为$-\frac{az+b}{z}$的原因有如下三个：

- 投影之后的光栅化阶段，要通过顶点的x'、y'对z进行线性插值，求出图元内部各片元的z，进行深度测试；

  数学上，投影后的x'和y'，与z不是线性关系，与`1/z`才是线性关系，而$-\frac{az+b}{z}=-(a+\frac{b}{z})$正是$\frac{1}{z}$的线性关系；

  因此，利用这个$\frac{1}{z}$的线性组合与x'、y'插值，才是`透视正确`的。

- p’的3个代数分量统一除以-z，易于使用齐次坐标变为普通坐标来完成，使得处理更加一致、高效；

- CVV是一个x, y, z的范围为[-1, 1]的规则体，便于进行多边形裁剪。我们可以适当的选择系数a和b，使得![fig19.GIF](https://p-blog.csdn.net/images/p_blog_csdn_net/popy007/fig19.GIF)这个式子在z = -N的时候值为-1，在z = -F的时候值为1，从而在z方向上构建CVV。

#### 3) 投影矩阵

​	投影矩阵需要在x, y, z三个方向上构建CVV，CVV中的齐次左边变为普通坐标后，最终形式为：<img src="./pic/cg_matrix_5.png" alt="cg_matrix_5" style="zoom:100%;" />

##### 3.1) 在Z方向构建CVV

​	z的范围是[-N, -F]，因此有：

<img src="./pic/cg_matrix_cvv_z.png" alt="cg_matrix_cvv_z" style="zoom:100%;" align="center" />

##### 3.2) 在X和Y方向构建CVV

​	$-N\frac{x}{z}$是投影平面上的点，范围是[left, right]，$x'$是标准化后的点，范围是[-1, 1]。

​	二者是**同一个值的不同形式**，$x'$是最终的目标值，$-N\frac{x}{z}$是推导过程的`中间值`，用于理解。因此有：

<img src="./pic/cg_matrix_cvv_x.png" alt="cg_matrix_cvv_x" style="zoom:100%;" align="center"/>

##### 3.3) 标准化后的投影点坐标

<img src="./pic/cg_matrix_cvv_std.png" alt="cg_matrix_cvv_std" style="zoom:100%;" />

##### 3.4) 坐标反推投影矩阵

​	首先，做透视除法的逆处理：每个分量乘以-z，得到：

<img src="./pic/cg_matrix_cvv_inverse.png" alt="cg_matrix_cvv_inverse" style="zoom:100%;" align="left"/>，又有：<img src="./pic/cg_matrix_cvv_mutiply.png" alt="cg_matrix_cvv_mutiply" style="zoom:100%;" />

​	可以反求得到投影矩阵M：

<img src="./pic/cg_matrix_cvv_prj.png" alt="cg_matrix_cvv_prj" style="zoom:100%;" align="left" />

​	综上，投影矩阵最后一行是：$\begin{pmatrix}0 & 0 & -1 & 0\end{pmatrix}$，而不是$\begin{pmatrix}0 & 0 & 0 & 1\end{pmatrix}$，可知透视变换**不是仿射变换**，它是非线性的。

​	进入CVV的点在[-1, 1]中，从而造成了**投影失真现象**。

​	投影失真的解决方法就是之后的`视口变换`：把归一化的顶点按照和`投影面上相同的比例`变换到视口中，从而解除透视投影变换带来的失真现象。

### 6 3d拾取

#### 1) 屏幕上的点转换到视口坐标

​	屏幕坐标原点在屏幕左上角，x向右，y向下；

​	视口坐标原点在左下角，x向右，向上。

#### 2) 视口中的点转换到投影平面上

​	投影平面的点被实施视口变换(线性插值)后，被变换到视口中。在这里，我们要实施一个逆变换。

$\frac{X_{vp} - Left_{vp}}{Right_{vp} - Left{vp}} = \frac{X_{p1} -(-1)}{1-(-1)}$

$\frac{Y_{vp} - Left_{vp}}{Right_{vp} - Left{vp}} = \frac{Y_{p1} -(-1)}{1-(-1)}$

​	综上，屏幕点击的位置被转换到了投影平面上，有：$P_{1}(X_{p1},Y_{p1},-N)$。

​	此时，该点是处于CVV中的，需要通过再次一次线性插值，把其转换到[left, right]、[bottom, top]中，得到$P_{2}(X_{p2},Y_{p2},-N)$：

$\frac{X_{p1} -(-1)}{1-(-1)} = \frac{X_{p2}-Left_{prj}}{Right_{prj}-Left{prj}}$

$\frac{Y_{p1} -(-1)}{1-(-1)} = \frac{Y_{p2}-Bottom_{prj}}{Top_{prj}-Bottom{prj}}$

#### 3) 向三维空间拓展

<img src="./pic/cg_matrix_3d_pickup.png" alt="cg_matrix_3d_pickup" style="zoom:100%;" />

​	使用射线ray，把2维空间的点拓展到3维中。

​	图中的模型处于相机空间中。射线起点为eye的位置，射线的方向朝近平面上的$P_{2}$延伸，得到射线ray；

​	通过ray和三维空间的三角面求交，来完成拾取。

## PBR

### 1 渲染方程的物理意义

### 2 PBR 里面的 D、F、G 项？菲涅尔项会带来怎样的视觉效果？

### 3 PBR 材质贴图多，纹理槽位不够应该怎么处理

## 算法应用

### 1 点和射线的距离

​	本小节参考自[这里](https://blog.csdn.net/LIQIANGEASTSUN/article/details/119598965)。

​	点和射线有如下四种位置关系：

<img src=".\pic\cg_ray_point.png" alt="cg_ray_point" style="zoom:75%;" align="left" /><img src=".\pic\cg_ray_point_1.png" alt="cg_ray_point" style="zoom:75%;" />



<img src=".\pic\cg_ray_point_2.png" alt="cg_ray_point" style="zoom:75%;" align="left" /><img src=".\pic\cg_ray_point_3.png" alt="cg_ray_point" style="zoom:75%;" />

#### 1) 点到射线的垂足坐标

$\vec{PO}=0 - P$;

$dot = \vec{PO}\cdot\vec{Ray}$

$D = P + dot*\vec{Ray}$

#### 2) 点到射线的距离

​	两点先验知识：

​	① 向量叉积模长公式：$|\vec{a}\otimes\vec{b}|=|\vec{a}||\vec{b}|sin\theta$。

​	② 平行四边形面积公式：$S=ab*sin\theta$，其中a、b是平行四边形的相邻边长度，为标量。

### 2 射线和三角形求交

#### 1) 解析法

​	假设射线和三角形所在平面的交点为p，在平面上任取一点a，有$(p-a)\cdot\vec{n}=0$，可计算求得点p。

​	然后通过叉积法判断点p是否在三角形内部：

<img src=".\pic\cg_algorithmn.png" alt="cg_algorithmn" style="zoom:85%;" />

​	依次求解$\vec{AP} \otimes \vec{AB}$、$\vec{BP} \otimes \vec{BC}$、$\vec{CP} \otimes \vec{CA}$，若结果向量的方向一致，则在三角形内部。

#### 2) moller-Trumbore射线三角相交算法

​	一种快速计算射线与三角形在三个维度上的交点的方法，通过向量与矩阵计算可以快速得出交点与重心坐标，而无需对包含三角形的平面方程进行预计算。算法推导[这里](https://blog.csdn.net/zhanxi1992/article/details/109903792)。

### 3 射线和球求交

#### 1) 解析法

​	将射线代入球的解析表达式，解方程，求解出交点坐标。

#### 2) 几何法

​	已知射线起点o和方向p，球的圆心为c，球的半径为r。

​	从c作到射线的垂线，求出距离h。若$h<=r$，则射线与球有交点。

### 4 判断任意点在凸多边形内部/外部

​	射线法、转角法参考[此处](https://blog.csdn.net/WilliamSun0122/article/details/77994526)。

#### 1) 射线法

​	以被测点Q为端点，向任意方向作射线(一般水平向右作射线)。统计该射线与多边形的交点数。如果为奇数，Q在多边形内；如果为偶数，Q在多边形外。

​	如下图有些特殊情况：

<img src=".\pic\cg_ray_intersect.png" alt="cg_ray_intersect" style="zoom:80%;" />

​	情况a)，射线和两条边的共同顶点相交，此时交点只能算一个；

​	情况b)，射线和两条线的最低处共同顶点相交，该点不能算；

​	情况c)，射线和多边形的一边平行，该边应忽略不计。

#### 2) 转角法

<img src=".\pic\cg_intersect_turning_angle.png" alt="cg_intersect_turning_angle" style="zoom:30%;" />

<img src=".\pic\cg_intersect_turning_angle_1.png" alt="cg_intersect_turning_angle_1" style="zoom:75%;" />

​	多边形内部的点连接各个顶点，其所形成的角度和在精度范围内应等于360度，如果小于360度或者大于360度，则证明该点不在多边形中。

​	转角法简单，但是由于涉及要使用反三角函数，会耗时，且造成较大的精度误差。

#### 3) 转角法优化

​	从P点向右做射线R，如果边从射线R下方跨到上方，那么穿越+1，如果从上方跨到下方，则是-1。最终和为wn环绕数。

<img src=".\pic\cg_intersect_turning_angle_optimize.png" alt="cg_intersect_turning_angle_optimize" style="zoom:85%;" />

​	这种方法不必去计算射线和边的交点，但需要判断点P和边的左右关系，而且对于方向向上和向下的边的判断规则不同。

​	对于方向向上的边，如果穿过射线，那么P是在有向边的左侧；对于方向向下的边，如果穿过射线，那么P在有向边的右侧。

<img src=".\pic\cg_intersect_turning_angle_optimize_1.png" alt="cg_intersect_turning_angle_optimize_1" style="zoom:75%;" />

### 5 判断多边形的凹凸性

​	依次顺时针遍历多边形的顶点。若向量的叉积保持一致，则是凸多边形，反之是凹多边形。

​	求$\vec{p_{0}p_{1}}\times\vec{p_{0}p_{2}}$：

```c++
double cross(Point& p1, Point& p2, Point& p0)
{
    return (p1.x - p0.x) * (p2.y - p0.y) - (p1.y - p0.y) * (p2.x - p0.x);
}
```

​	判断是否为凸多边形。若Polygon是顺时针排序，叉乘结果应该小于0；若Polygon是逆指针排序，叉乘结果应该大于0：

```c++
bool isConvexPolygon(QVector<Point> Polygon)
{
    int len = Polygon.size();
    int s = 0, e = len;
    if(e == 2)
        return true;
    while (s <= e-3) {
        if (cross(Polygon[s+1], Polygon[s+2],Polygon[s]) < 0)
            s++;
        else 
            return false;
    }
    return true;
}
```

## 纹理贴图

## 抗锯齿技术

## ShadowMap

### 1 DepthMap

#### 1) DepthMap格式、精度和分辨率

- OpenGL中DepthMap格式可以为`GL_DEPTH_COMPONENT24`、`GL_DEPTH_COMPONENT32`等等，数值格式为`float`；增加深度通道的位数可以提高精度，解决深度冲突；

  分辨率通常使用`1024 x 1024`；

- 点经过视图变换、投影变换、透视除法后，z值大小在near和far之间的数会被转换到`-1~1`中；在Viewport变换中，默认情况下，把`-1~1`转换到`0~1`；

- 接近 near 平面，z值越密；距离越远，z值越稀，这样距离照相机越近精度越高。

#### 2) 深度的非线性

<img src="./pic/cg_depth_non_linear.png" alt="cg_depth_non_linear" style="zoom:65%;" />

​	z：模型点经过模型、视图变换后，在相机空间中的z值；

​	d：z经过透视变换、透视除法后在ndc空间的深度值。

##### 2.1) z -> d为何非线性

- 实用性：计算机存储精度有限，离摄像机越远的物体，信息越少，对画面的贡献也越少，没必要为其提供高精度。所以深度值在靠近近平面精度越高，越远精度越低，是合理的设定。

- 数学上：投影面上等距的多个点，在三维空间中不是等距的；

  <img src="./pic/cg_depth_non_linear_1.png" alt="cg_depth_non_linear_1" style="zoom:40%;" />

  上图中三维空间点$(x_{1},z_{1})$和$(x_{2},z_{2})$投影到z为-e的平面上得到两点$(p_{1},-e)$和$(p_{2},-e)$。在投影点间取几个等距点还原，可以看到还原的三维点并不等距。

##### 2.2) 线性化深度

$z_{linear}=\frac{z-n}{f-n}$

$z=\frac{2nf}{f+n-(2d-1)*(f-n)}$

​	d：深度图中取出的数据；n、f：近平面和远平面的距离；

#### 3) 深度值和顶点属性插值

​	在使用光栅化的图形学方法中，法线，颜色，纹理坐标这些属性通常是绑定在图元的顶点上。

​	3D空间中，这些属性值在图元上是线性变化的。但当3D顶点被透视投影到2D屏幕之后，如果在2D投影面上对属性值进行线性插值，会有问题，如下图所示：

<img src="./pic/cg_depth_non_linear_2.png" alt="cg_depth_non_linear_2" style="zoom:60%;" />

​	图中将A和B是带属性的两个顶点；

​	将A和B通过camera投影到距离为d的平面上，得到投影点a和b。注意本图中camera的视线方向为+z，通常可自定义视线为+z或-z。

​	$c(u_{s}, d)$是a和b在二维平面上的线性插值点：$c = a + s \ast (b-a)$。

​	通过下式，可以由投影平面上的插值点，得到三维空间中透视正确的插值：

$\frac{1}{Z_{s}} = \frac{1}{Z_{1}}(1-s) + \frac{1}{Z_{2}}s$

​	综上，**深度值的倒数是线性相关的**，具体推导参考：[图形学基础之透视校正插值](https://blog.csdn.net/n5/article/details/100148540)。

​	有了透视正确的顶点位置后，由于顶点属性和位置在3维空间中是线性相关的，因此也可利用上式求出顶点属性的正确插值。

#### 4) 深度冲突

​	DepthBuffer是非线性的，因此在远离摄像机的地方，由于精度不足，可能造成深度冲突。

​	有如下种解决方案：

- 增大深度缓冲数据位数(一般不采用，为解决深度冲突而增大显存不明智)；
- 修改摄像机的近远平面，让其范围更小，范围变小后数据能够表示的精度自然就上升；
- 略微在场景中移动物体坐标，错开那些靠的很近的物体(实用，基本能解决问题)；
- Offset语法【**待理解**】

#### 5) 平行光的视图矩阵、投影矩阵

```c++
distLightCam.setPosition(lightEnv.DirectionalLight.Position);
distLightCam.setLookAt(lightEnv.DirectionalLight.Dest);	
distLightCam.setOrtho(-10.f, 10.f, -10.f, 10.f);
distLightCam.setNearFar(camera.near(), camera.far());
```

### 2 点光源CubeMap

#### 1) 写入深度图

- 顶点着色器：将mesh转换到世界空间中，作为几何着色器的输入；

- 几何着色器：将单个图元的顶点，通过`6个`light space矩阵，转换到以光源为原点透视空间中，作为光栅化输入。

  同时，将世界坐标中的顶点作为属性，传入到片元着色器。

- 片元着色器：光栅化上述不同光照空间的图元后，片元着色器中根据几何着色器传入的world position和常量光源位置，计算距离，写入不同面的深度缓冲。

#### 2) CubeMap采样

```glsl
vec3 sampleVec = fragPos - lightPos; 
float closestDepth = texture(depthMap, sampleVec).r;
```

​	上述用于采样的向量是以光源为坐标原点的方向向量；

​	深度缓冲是以光源为中心写入的，因此可以采样到包围光源所有方向的深度值。

## 骨骼动画

### 1 基础概念

- Bind Pose(绑定姿态)：美术在建模软件中定义的骨骼**默认姿态**，绑定空间可以理解为**模型本地空间**。

- Offset Matrix：将顶点从绑定空间转换到某个关节的骨骼空间(**Bone Space**)，也可称为**局部空间**。

- Local Transform(局部变换)：Local Transform是父节点Bone Space下的变化矩阵，通常用于**编辑某个子关节**。

- Global Transform(全局变换)：在最简单的情况下，已知一个关节的局部变换，**连乘**它的所有父骨骼的局部变换矩阵，就能得到该关节的**全局变换矩阵|组合变换矩阵**(Global Transform)。**根节点的父节点处于模型空间中**。

- Mesh Transform(蒙皮变换矩阵)：Mesh Transform = Global Transform x OffsetMatrix。顶点应用了蒙皮矩阵后，就确定了其在绑定空间下的最终位置。
