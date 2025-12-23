---
typora-root-url: pic
---

# Shader

## 创建Shader

- 编译各阶段shader，如vertex、fragment等等；
- 创建着色程序；
- shader和着色程序关联；
- 链接着色程序;

## 普通shader示例

```glsl
#version 430 core
#pragma stage : vert
precision mediump float;

layout (location = 0) in vec3 a_Pos;
layout (location = 1) in vec2 a_Texcoord;

out v2f {
    vec2 v_TexCoord;
};

void main() {
	v_TexCoord = a_Texcoord;	
	gl_Position = vec4(a_Pos, 1.f);
}

#version 430 core 
#pragma stage : frag 
precision mediump float;

in v2f {
    vec2 v_TexCoord;
};

out vec4 o_FragColor;

void main() {
	o_FragColor = texture(u_Texture, v_TexCoord);
}
```

## 缓存Uniform和Attribute信息

shader成功生成后，可根据下述API获取shader中有效的uniform、顶点attribute信息和UniformBlock的信息：

```c++
glGetActiveUniform(...)
glGetActiveAttrib(...)
glGetActiveUniformBlockiv(...)
```

其中，顶点属性如下：

```glsl
layout (location = 0) in vec3 a_Pos;
layout (location = 1) in vec2 a_Texcoord;
```

Uniform定义如下：

```glsl
uniform sampler2D u_AlbedoMap;
uniform sampler2D u_EmissionMap;
```

UniformBlock定义如下：

```glsl
layout (std140) uniform CameraBlock
{
	mat4 ViewMat;
	mat4 PrjMat;
	mat4 OrthoMat;
    vec3 Position;
    float FrustumFar;
	int PrjType;
} u_Camera;
```

## ShaderMutant变体

着色器根据不同的宏，在原有的基础上，产生的Shader。

# 数据缓冲区

## 顶点缓冲

### 1 VBO

存放顶点属性数据。

- 生成Buffer

  ```c++
  glGenBuffers(1, &m_ID);
  glBindBuffer(GL_ARRAY_BUFFER, m_ID);
  glBufferData(GL_ARRAY_BUFFER, m_Size, m_Data, m_Usage);//开辟内存
  glBindBuffer(GL_ARRAY_BUFFER, 0);
  ```

- 修改数据

  ```c++
  glBufferSubData(GL_ARRAY_BUFFER, offset, size, data);
  ```

- 绑定顶点槽位，指定每个顶点属性的stride

  ```c++
  /*
  * 参数如下
  * slot index, 槽位索引
  * component, 数量
  * component的类型
  * 是否normalized
  * 顶点属性的总长度
  * 当前属性和起始位置的偏移量
  */
  // 设置slot0属性vec3
  glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float), (void*)0);
  glEnableVertexAttribArray(0);
  // 设置slot1属性vec3
  glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float), (void*)(3 * sizeof(float)));
  glEnableVertexAttribArray(1);
  ```

### 2 EBO

Element Buffer Object，存放索引数据。

- 生成Buffer

  ```c++
  glGenBuffers(1, &m_ID);
  
  glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, m_ID);
  glBufferData(GL_ELEMENT_ARRAY_BUFFER, m_Size, m_Data, m_Usage);
  glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0);
  ```

- 修改数据

  ```c++
  glBufferSubData(GL_ELEMENT_ARRAY_BUFFER, offset, size, data);
  ```

### 3 VAO

VAO，即**顶点数组对象**，是高效管理顶点数据状态的核心工具，它能显著简化代码并提升渲染效率。

#### 3.1 优势

- 渲染效率

  将顶点属性配置预先记录在VAO中，渲染时只需绑定对应的VAO，避免了每次绘制都进行大量重复的OpenGL API调用；

- 状态管理

  封装了与顶点数据相关的状态，例如顶点属性指针的配置、顶点属性数组的启用状态，以及VBO的绑定关系等。

  这使得在不同物体(即不同顶点数据配置)之间切换变得非常简单，只需绑定不同的VAO即可；

- 代码简化

#### 3.2 使用

- 初始化

  ```c++
  glGenVertexArrays(1, &m_ID);
  ```

- 将VBO绑定到当前VAO

  ```c++
  glBindVertexArray(m_ID);
  
  // m_VBO_0
  glBindBuffer(GL_ARRAY_BUFFER, m_VBO_0);
  glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float), (void*)0);
  glEnableVertexAttribArray(0);
  
  glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float), (void*)(3 * sizeof(float)));
  glEnableVertexAttribArray(1);
  
  // m_VBO_1
  glBindBuffer(GL_ARRAY_BUFFER, m_VBO_1);
  glVertexAttribPointer(2, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float), (void*)0);
  glEnableVertexAttribArray(2);
  
  glVertexAttribPointer(3, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float), (void*)(3 * sizeof(float)));
  glEnableVertexAttribArray(3);
  
  // ........
  
  glBindVertexArray(0);
  ```

  如果使用了EBO，也可把EBO绑定到当前VAO下。

## UniformBlock

- shader中UniformBlock示例

  ```glsl
  layout (std140) uniform CameraBlock
  {
  	mat4 ViewMat;
  	mat4 PrjMat;
  	mat4 OrthoMat;
      vec3 Position;
      float FrustumFar;
  	int PrjType;
  } u_Camera;
  ```

- 生成Buffer，开辟内存

  ```c++
  glGenBuffers(1, &m_ID);
  glBindBuffer(GL_UNIFORM_BUFFER, m_ID);
  glBufferData(GL_UNIFORM_BUFFER, m_Size, nullptr, m_Usage);
  ```

- 绑定layout

  ```c++
  // blockName: block在shader中的名字
  uint32_t index = glGetUniformBlockIndex(program, blockName.c_str());
  // index: 着色器槽位
  // bingdingPt: CPU槽位
  glUniformBlockBinding(program, index, m_Binding);
  ```

- 使用前绑定

  ```c++
  glBindBufferBase(GL_UNIFORM_BUFFER, m_Binding, m_ID);
  ```

- 写入数据

  ```c++
  glBindBuffer(GL_UNIFORM_BUFFER, m_ID);
  glBufferSubData(GL_UNIFORM_BUFFER, offset, size, data);
  ```

# 图元绘制

## GL_TRIANGLE_STRIP

### 1 绘制原理

<img src="/pic_triangle_strip.png" alt="pic_triangle_strip" style="zoom:80%;" />

上述通过GL_TRIANGLE_STRIP的形式通过6个顶点构建了4个三角形。

其中，顶点的环绕规则是(逆时针下)：

- 偶数顶点，环绕规则为[n-2, n-1, n]；
- 奇数顶点，环绕规则为[n-1, n-2, n]；

顶点$v_{2}$为偶数顶点，通过$v_{0}$、$v_{1}$、$v_{2}$构建了三角形；

顶点$v_{3}$为奇数顶点，通过$v_{2}$、$v_{1}$、$v_{3}$构建了三角形；

顶点$v_{5}$为奇数顶点，通过$v_{4}$、$v_{3}$、$v_{5}$构建了三角形。

注意，以索引较大的一边为基准，新增三角形。

比如$\bigtriangleup v_{2}v_{1}v_{3}$是以边$v_{1}v_{2}$为基准构建的；

$\bigtriangleup v_{4}v_{3}v_{5}$是以边$v_{3}v_{4}$为基准构建的。

### 2 优点

- 绘制N个三角形只需要定义N+2个顶点，相比下`GL_TRIANGLES`模式则需要定义3N个顶点；
- 减少顶点数量，缓解CPU-GPU之间的带宽压力，提升性能；
- 减少顶点数量，顶点数据所需内存更少，对复杂模型尤其有益；
- 与现代GPU缓存更友好，能提高缓存命中率，减少顶点着色器的重复计算。

### 3 适用场景

- 它最适合绘制**拓扑结构为连续条带状的表面**，例如地形、管道、道路或大型模型的连续网格。对于离散、不连续的三角形，`GL_TRIANGLES`仍然是更好的选择。
