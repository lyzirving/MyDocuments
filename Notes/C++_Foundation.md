## 模板

### 1 可变参数模板

1. 参数个数未知，参数类型相同

   标准库类型initializer_list：容器类型，容器元素为常量。

2. 参数个数未知，参数类型可能不同

   ```c++
   template<class... T>      // T是模板参数包
   void function(T ... args) // args是函数参数包
   {
   }
   ```

   …代表一个包含0到n的任意参数包，其中的任意代表任意数目和任意类型。

   …在参数右侧代表一组实参，…在参数左侧代表可变参数。

3. 可变参数包展开

   - 查询个数：

     ```c++
     template<typename... Args>
     unsigned int length(Args... args) { return sizeof...(args); }
     ```

   - 递归展开：

     ```c++
     // 边界函数
     void print() {}
     
     template<typename T, typename... Types> //(1)可变参数
     void print(const T& firstArg, const Types&... args) //(2)可变参数
     {
     	std::cout << firstArg << " " << sizeof...(args) << std::endl; 
     	print(args...); //(3)一组实参
     }
     ```

   - 逗号运算符展开：

     ```c++
     template<typename T>
     void show_arg(const T& t) {}
     
     template <typename ... Args>
     void expand(const Args&... args)
     {
     	int arr[] = {(show_arg(args), 0)...}; 
     }
     ```

     解析如下：

     ```c++
     // 函数调用
     expand(1, 2, 3, 4);
     // expand内部展开后如下
     int arr[] = { (show_arg(1), 0), 
                   (show_arg(2), 0), 
                   (show_arg(3), 0),
                   (show_arg(4), 0)};
     // 结果代码如下
     int arr[] = {0, 0, 0, 0}; 
     ```

### 2 逗号运算符

​	C++提供了一种特殊的运算符：逗号运算符(**顺序求值**运算符)，可将两个表达式连接起来。

​	一般形式为：表达式1, 表达式2

​	求解过程为：**先求解表达式1的值，再求解表达式2的值，整个表达式的值是表达式2的值**。

```c++
int main()
{
  int a;
  a=3*5,4*5;     // 不加括号的逗号表达式
  cout<<a<<endl; // 输出15
  a=(3*5,4*5);   // 加括号的逗号表达式
  cout<<a<<endl; // 输出20
  return 0;
}
```

​	不加括号时，赋值运算符的**优先级**高于逗号运算符，因此会先求解a=3 \* 5。

​	程序是自左向右运行的，后面的4 \* 5也会运行，但没有存储结果。

## 多态

### 1 静态多态

​	编译期间的多态，编译器在**编译期间**完成，通过实参类型推断出要调用的函数；

​	静态多态有两种实现方式：1.函数重载；2.模板。

​	**函数重载**的关键是**参数列表**：参数数目、类型和排序，与函数名和返回值无关。

​	**模板**使用泛型来定义函数，泛型可由具体的类型替换。通过将类型作为参数，传递给模板，通过编译器生成该类型的具体函数。

### 2 动态多态

​	在程序执行期间判断所引用对象的实际类型，根据其实际类型调用相应的方法。

​	对于有虚函数的类，编译器会维护一张虚函数表(虚表)，对象的前四个字节就是指向虚表的指针(虚表指针)。

​	虚函数表的创建遵循下述原则：

- 先拷贝基类的虚函数表；
- 若派生类重写了基类的某个虚函数，就用派生类的虚函数替换虚表同位置的基类虚函数；
- 创建派生类独有的虚函数；

​	无覆盖的情况：

```c++
class Base
{
    virtual void fun1();
    virtual void fun2();
    virtual void fun3();
}
class Derived : public Base
{
    virtual void fun4();
    virtual void fun5();
    virtual void fun6();
}
```

<img src="./pic/note_pic_0.png" alt="note_pic_0" style="zoom:100%;" />

​	有覆盖的情况：

```c++
class Base
{
    virtual void fun1();
    virtual void fun2();
    virtual void fun3();
}
class Derived : public Base
{
    virtual void fun2();
    virtual void fun3();
    virtual void fun4();
}
```

<img src="./pic/note_pic_1.png" alt="note_pic_1" style="zoom:100%;" />

### 3 虚析构函数

​	虚析构函数确保派生类的析构函数被正确调用，有如下示例：

```c++
class Base {
public:
    Base() { cout << "执行基类构造函数" << endl; }
    virtual ~Base() { cout << "执行基类析构函数" << endl; }
};

class Derived : public Base {
public:
    Derived() { cout << "执行派生类构造函数" << endl; }
    ~Derived() { cout << "执行派生类析构函数" << endl; } // 派生类不需要显示声明为virtual
};
```

​	执行下述代码：

```c++
Base *pt = new Derived;
delete pt;

//输出如下
//执行基类构造函数
//执行派生类构造函数
//执行派生类析构函数
//执行基类析构函数
```

## 内存布局

### 1 内存分区

#### 1) 区域划分

​	在C++中，内存分成**5个区**，分别是：**堆、栈、自由存储区、全局/静态存储区、常量存储区**。

​	有的书籍也把内存划分为**堆区**、**栈区**和**静态存储区**，其中把自由存储区、全局/静态存储区和常量存储区总结为一块区域。

- **栈**：内存由编译器在需要时自动分配和释放。通常用来存储局部变量和函数参数、返回地址。栈运算分配内置于处理器的指令集中，**效率很高**，但是分配的内存容量有限。
- **堆**：使用new分配的内存，使用delete或delete[]释放。如果未能对内存进行正确的释放，会造成**内存泄漏**。程序结束时，由操作系统自动回收。
- **自由存储区**：使用malloc进行分配，使用free进行回收。和堆类似。
- **全局/静态存储区**：全局变量和静态变量被分配到同一块内存中，C语言中区分初始化和未初始化的，C++中不再区分。
-  **常量存储区**：存储常量，不允许被修改。

<img src="./pic/note_pic_mem_area.png" alt="note_pic_mem_area" style="zoom:100%;" />

#### 2) 区域间的不同

##### 2.1) 静态存储区和栈区

```c++
char* p = “Hello World1”;   //1 指针p在栈区, "Hello World1"是字符串常量, 存储在静态存储区
char a[] = “Hello World2”;  //2 初始化分配的数组，所以a[]和"Hello World2"都在栈上 
p[2] = ‘A’;                 //3 抛出运行时异常, 因为*p指向的区域是常量区
a[2] = ‘A’;                 //4 可以修改，因为a[]在栈上
char* p1 = “Hello World1;”  //5 p1在栈上，且p和p1指向同一块内存
```

##### 2.2) 堆与栈

- 内存分配和释放：对于**栈**来讲，内存由编译器管理，无需手工控制；对于**堆**来说，分配和释放工作由程序员控制，容易产生memory leak。
- 空间大小：在32位系统下，**堆内存**可以达到4G的空间；**栈内存**有空间限制，在VC6.0下面默认的栈空间大小是1M。
- 内存碎片：对于**堆**，频繁的new/delete会造成内存空间的不连续，产生大量碎片，影响性能；对于**栈**，内存按照先进后出的方式管理，不会产生碎片。
- 生长方向：对于**堆**，生长方向是**向上**的，向着内存地址增加的方向；对于**栈**，生长方向是**向下**的，是向着内存地址减小的方向增长。
- 分配效率：对于**堆**，内存分配是C/C++函数库提供的，其机制复杂。为了分配一块内存，库函数会按照一定的算法在堆内存中搜索可用的足够大小的空间，如果没有足够大小的空间(可能是由于内存碎片太多)，就有可能调用系统函数去**增加程序数据段的内存空间**，从而获取到足够大小的内存，然后返回。上述可能会经历用户态和内核态的切换。对于**栈**，其是机器系统提供的数据结构，计算机会在**底层对栈提供支持**：分配专门的寄存器存放栈的地址，压栈出栈都有专门的指令执行，这就决定了栈的效率比堆更高。

### 2 虚函数表

​	若类中存在虚函数，编译器会为该类生成一个虚函数表，虚函数表按照虚函数的声明顺序存放了各个虚函数的地址。

​	虚函数表不存在于类中，而对于每个类对象，编译器都会为其生成一个不可见的指针，指向同一个地址，这个指针就是虚函数表指针。

​	虚函数表指针位于该类对象内存中的开头，指向了虚函数表的位置。

​	因此，在32位机器上，通过类的前4个字节就可找到虚函数表的地址，在64位机器上，通过前8字节找到虚函数表的地址。

​	有如下示例：

```c++
using VoidFunc = void(*)();

class Base
{
public:
	Base(int i = 0) : m_baseI(i) {};
	virtual void func0() { std::cout << "Base::func0" << std::endl; }
	virtual void func1() { std::cout << "Base::func1" << std::endl; }
	virtual ~Base() { std::cout << "Base::Destructor" << std::endl; }
private:
	int m_baseI;
};

Base a{0};
int v_tableAddr = *(int *)(&a);
```

​	上述去虚函数解释如下：

- &a得到对象a的地址；
- (int *)(&a) ：把(&a)强制转换为(int *)，把从(&a)开始的4个字节当作一个整体；
- *(int *)(&a) ：对a的前4个字节解引用，获得虚函数表的地址。

​	获得虚函数func0()的地址：

```c++
int func0Addr = *(int *)*(int *)(&a);
((VoidFunc)func0Addr)();
```

- (*(int *)(&a))是虚函数表的地址，(int \*)\*(int \*)(&a)将其强制转换为(int* *)指针；
- \*(int \*)\*(int \*)(&a)：取int\*第一个4字节上的值，得到func0()的地址；
- 将地址值强制转换为void(*)()，进行调用。

​	获取func1()的地址：

```c++
int func1Addr = *((int *)*(int *)(&a) + 1);
((VoidFunc)func1Addr)();
```

### 3 对象模型

- nonstatic 数据成员被置于每一个类对象中，static数据成员被置于类对象之外；
- static与nonstatic函数被置于类对象之外；
- virtual函数通过虚函数表+虚指针来支持；
- 虚函数表前设置了一个指向type_info的指针，用以支持RTTI。RTTI是为多态而生成的信息，包括对象继承关系，对象本身的描述等。

#### 1) 非继承下的对象模型

​	有基类Base：

```c++
class Base
{
public:
    Base(int i) :baseI(i){};
    int getI(){ return baseI; }
    static void countI(){};
 
    virtual ~Base(){}
    virtual void print(void){ cout << "Base::print()"; }
private:
    int baseI;
    static int baseS;
};
```

​	类的静态成员在**全局区**，函数在**代码区**，下述展示了堆栈上的类对象布局：

<img src="./pic/note_pic_2.png" alt="note_pic_2" style="zoom:100%;" />

#### 2) 单继承下的对象模型

​	在基类Base下，有派生类如下：

```c++
class Derive : public Base
{
public:
    Derive(int d) :Base(1000), DeriveI(d) {};
    //覆写的虚函数
    virtual void print() override { cout << "Drive::print()" ; }
    //新增的虚函数
    virtual void Drive_print(){ cout << "Drive::Drive_print()" ; }
    virtual ~Derive(){}
private:
    int DeriveI;
};
```

<img src="./pic/note_pic_derive.png" alt="note_pic_derive" style="zoom:100%;" />

- 对于一般继承（相对于虚拟继承），子类有**独有**的虚函数表；
- 若子类重写了父类的虚函数，子类虚函数将覆盖虚表中对应的父类虚函数；
- 若子类没有覆写父类虚函数，而是声明新的虚函数，则扩充虚函数地址至虚函数表最后；

#### 3) 多继承(非菱形继承)

​	有如下多继承的示例：

```c++
class Base
{
public:
    Base(int i) :baseI(i){};
    virtual ~Base(){}
    virtual void print(void){ cout << "Base::print()"; }
    int getI(){ return baseI; }
    static void countI(){};
private:
    int baseI;
    static int baseS;
};

class Base_2
{
public:
    Base_2(int i) :base2I(i){};
    virtual ~Base_2(){}
    virtual void print(void){ cout << "Base_2::print()"; }
    int getI(){ return base2I; }
    static void countI(){};  
private:
    int base2I; 
    static int base2S;
};

class Drive_multyBase :public Base, public Base_2
{
public:
    Drive_multyBase(int d) :Base(1000), Base_2(2000) ,Drive_multyBaseI(d){};
    virtual void print(void){ cout << "Drive_multyBase::print" ; } 
    virtual void Drive_print(){ cout << "Drive_multyBase::Drive_print" ; } 
private:
    int Drive_multyBaseI;
};
```

​	多继承的规则如下：

- 子类的虚函数放在声明的第一个基类的虚函数表中；
- 覆写时，所有基类的相关函数都被子类覆写；
- 内存布局中，父类按照其声明顺序排列；

​	多继承的对象模型如下：

<img src="./pic/note_pic_mutiple_base_class.png" alt="note_pic_mutiple_base_class" style="zoom:100%;"/>

#### 4) 菱形继承

​	菱形继承中，基类被某个派生类简单重复继承了多次，导致派生类对象中**拥有多份基类实例**，如下例子所示：

```c++
class B
{
public:
    int ib;
public:
    B(int i=1) : ib(i) {}
    virtual void f() { cout << "B::f()" << endl; }
    virtual void Bf() { cout << "B::Bf()" << endl; }
};

class B1 : public B
{
public:
    int ib1;
public:
    B1(int i = 100 ) :ib1(i) {} 
    virtual void f() { cout << "B1::f()" << endl; } 
    virtual void f1() { cout << "B1::f1()" << endl; } 
    virtual void Bf1() { cout << "B1::Bf1()" << endl; }
 
};

class B2 : public B
{
public:
    int ib2;
public: 
    B2(int i = 1000) :ib2(i) {}
    virtual void f() { cout << "B2::f()" << endl; } 
    virtual void f2() { cout << "B2::f2()" << endl; } 
    virtual void Bf2() { cout << "B2::Bf2()" << endl; } 
};

class D : public B1, public B2
{ 
public: 
    int id;
public: 
    D(int i= 10000) :id(i){} 
    virtual void f() { cout << "D::f()" << endl; } 
    virtual void f1() { cout << "D::f1()" << endl; } 
    virtual void f2() { cout << "D::f2()" << endl; }
    virtual void Df() { cout << "D::Df()" << endl; } 
};
```

​	此时D的内存布局如下：

<img src="./pic/note_pic_mutiple_base_class_2.png" alt="note_pic_mutiple_base_class_2" style="zoom:50%;" />

​	D间接继承了B两次，导致D内部会有两个B的成员ib。这不仅增大了内存空间，也会引起程序歧义：

```c++
D d;
d.ib =1;       //二义性错误
d.B1::ib = 1;  //正确
d.B2::ib = 1;  //正确
```

#### 5) 虚继承

​	虚继承解决了菱形继承中派生类拥有多个间接父类实例的情况。

​	虚基表：

- 虚基类表指针总在虚函数表指针之后；
- 对某个类实例来说，虚基类指针可能在实例的0字节偏移处，也可能在4字节偏移处；
- 虚基表由多个条目组成，每个条目存放的是偏移值。其中第一个条目存放的是虚基表指针到该类内存首地址偏移值，该值为0或-4。
- 虚基类表的第二、第三...个条目依次为该类的最左虚继承父类、次左虚继承父类...的内存地址相对于虚基类表指针的偏移值。

​	包含虚基表的情况：

<img src="./pic/note_pic_vbptr.png" alt="note_pic_vbptr" style="zoom:80%;" />

​	不含虚基表的情况：

<img src="/pic/note_pic_vbptr1.png" alt="note_pic_vbptr1" style="zoom:80%;" />

##### 5.1) 简单虚继承

```c++
class B{...}
class B1 : virtual public B {...}
```

​	B1的对象模型如下：

​	下图中虚基表中实际有两个条目，其中第二个条目没有画出来，即为最左边虚基类地址到虚基表地址的偏移值：[4]-[1]。

​	因此，可通过虚基表的条目一和条目二，得到虚基类Base的地址。

​	下述代码得到第二个条目地址：

```c++
*(int *)((int *)*((int *)(&a) + 1) + 1);
```

<img src="./pic/note_pic_vbptr2.png" alt="note_pic_vbptr2" style="zoom:50%;" />

##### 5.2) 菱形虚拟继承

```c++
class B {...}
class B1: virtual public B {...}
class B2: virtual public B {...}
class D : public B1, public B2 {...}
```

​	其中D的对象模型如下：

<img src="./pic/note_pic_vbptr3.png" alt="note_pic_vbptr3" style="zoom:70%;" />

#### 6) 空壳类

​	有如下例子的空壳类：

```c++
class B {};
class B1 : public virtual B{};
class B2 : public virtual B{};
class D : public B1, public B2{};
```

​	每个类对象的大小如下：

```c++
B b;    //1
B1 b1;  //4
B2 b2;  //4
D d;    //8
```

​	编译器为空壳类分安插了1一个字节，得以分配地址。

### 4 字节对齐

#### 1) 自然对界alignment

- 对界的定义：

  ​	在struct中，编译器为其每个成员按其自然对界(alignment)条件分配空间；

  ​	各个成员按照它们被声明的顺序在内存中顺序存储(成员之间可能插入空字节)；

  ​	第一个成员的地址和整个结构的地址相同。

- 结构体内成员字节补齐方式：

  编译器缺省的结构成员自然对界条件为“N字节对齐”，N即该成员数据类型的长度。

  如int型成员的自然对界条件为4字节，double类型的结构成员的自然对界条件为8字节。

  若该成员的**起始偏移**不位于该成员的“默认自然对界条件”上，则在前一个成员后添加适当空字节。

- 结构体的自然对界：

  编译器缺省的结构整体的自然对界条件为：该结构所有成员中要求的**最大自然对界条件**。

  若结构体各成员长度之和不为“结构整体自然对界条件的整数倍”，则在最后一个成员后填充空字节。

- 示例：

  ```c++
  struct testlength1 
  {
    int aa1;   // offset 0
    char bb1;  // offset 4
    short cc1; // offset 6
    char dd1;  // offset 9
  };
  //内存布局 1111 1011 1000
  struct testlength2
  {
    char bb2;  // offset 0
    int aa2;   // offset 4
    short cc2; // offset 8
    char dd2;  // offset 10
  };
  //内存布局 1000 1111 1110
  ```

#### 2) 改变缺省的对界

​	使用伪指令#pragma pack (n)，编译器将按照n个字节对齐。

​	使用伪指令#pragma pack ()，取消自定义字节对齐方式。

​	此时对齐规则如下：

- struct/union内数据成员规则：

  第一个数据成员offset为0，以后每个数据成员的对齐按照#pragma pack指定的数值和这个数据成员自身长度中，比较小的那个进行。

- 整个struct/union规则：

  在数据成员完成各自对齐之后，结构(或联合)本身也要进行对齐；

  对齐将按照#pragma pack指定的数值和结构(或联合)最大数据成员长度中，比较小的那个进行。

- 无效的情况：

  根据上述两点，当#pragma pack的n值等于或超过所有数据成员长度的时候，这个n值的大小将不产生任何效果。

- 内嵌子结构体：

  子结构中的成员按照#pragma pack指定的数值和子结构最大数据成员长度中，比较小的那个进行对齐。

#### 3) struct和union

##### 3.1) 定义区别

- 编译器会为struct的每个成员分配存储空间；
- union的所有成员共用一块存储空间。对union成员赋值，会重写其他成员。此时使用其他成员会出现未定义行为。
- union中占用内存最多的成员定义了整个union的内存大小。

##### 3.2) 对齐方式

​	union，struct，class按成员中最大自然对界执行。

```c++
union u
{
　double a;
　int b;
};
union u2
{
　char a[13];
　int b;
};
union u3
{
　char a[13];
　char b;
};
cout<<sizeof(u)<<endl;  // 8,  按照double对齐
cout<<sizeof(u2)<<endl; // 16, 按照int对齐
cout<<sizeof(u3)<<endl; // 13
```

```c++
struct s1
{
　char a;   // offset 0
　double b; // offset 8
　int c;    // offset 16
　char d;   // offset 20
};
//内存布局 1000 0000 1111 1111 1111 1000 
struct s2
{
　char a;   // offset 0
　char b;   // offset 1
　int c;    // offset 4
　double d; // offset 8
};
//内存布局 1100 1111 1111 1111
cout<<sizeof(s1)<<endl; // 24
cout<<sizeof(s2)<<endl; // 16
```

## 语法

### 1 extern "C"

​	头文件中一般存在如下结构，主要分析extern "C"的作用：

```c++
//incvxworks.h
#ifndef __INCvxWorksh
#define __INCvxWorksh

#ifdef __cplusplus
extern "C" {
#endif
    /*...*/
#ifdef __cplusplus
}
#endif

#endif /* __INCvxWorksh */
```

​	extern "C" 包含双重含义：1) 被它修饰的目标是**extern**的；2) 被它修饰的目标是**C**的。

#### 1) extern

​	`extern`是`C/C++`语言中表明函数和全局变量作用范围（可见性）的关键字，该关键字告诉编译器，其声明的函数和变量可以在本模块或**其它模块**中使用。

- 语句 `extern int a;` 仅是对变量的声明，并未为 `a` 分配内存空间。
- 引用全局变量/函数前，必须有变量/函数的声明(或者定义)。例如，模块`B`欲引用模块`A`中定义的全局变量/函数，需包含模块`A`的头文件。
- 编译阶段，模块`B`虽然找不到变量/函数，但不会报错；**链接阶段**`B`会从模块`A`编译生成的**目标代码**中找到变量/函数。
- 与`extern`对应的关键字是`static`，被它修饰的全局变量和函数只能在**本模块**中使用。

#### 2) extern "C"修饰的变量/函数是按照C的方式编译和链接的

​	作为一种面向对象的语言，`C++`支持函数重载，过程式语言`C`不支持。

​	所以，函数被`C++`编译后在符号库中的名字与`C`语言的有所不同。有如下函数：

```c++
void foo( int x, int y );
```

- `C++`编译器则会产生像`_foo_int_int`之类的名字(不同的编译器生成的名字不同，但采用了相同机制，生成的新名字称为`mangled name`)。
- `C++`类成员变量/函数以`.`来区分，其可能与全局变量/函数重名。本质上，编译器编译时，也为类中的变量/函数取了一个独一无二的名字，与全局变量的名字不同。
- `C`由于没有实现重载，会产生`_foo`之类的名字。

#### 3) extern "C"是实现C++和C程序混合编程所使用

##### 3.1) C++中引用C

```c++
/* c语言头文件：cExample.h */
#ifndef C_EXAMPLE_H
#define C_EXAMPLE_H
extern int add(int x,int y);
#endif

/* c语言实现文件：cExample.c */
#include "cExample.h"
int add( int x, int y )
{
    return x + y;
}

// c++实现文件，调用add：cppFile.cpp
extern "C" {
    #include "cExample.h"
}
int main(int argc, char* argv[])
{
    add(2,3);
    return 0;
}
```

##### 3.2) C中引用C++

```c++
//C++头文件 cppExample.h
#ifndef CPP_EXAMPLE_H
#define CPP_EXAMPLE_H
extern "C" int add( int x, int y );
#endif

//C++实现文件 cppExample.cpp
#include "cppExample.h"
int add( int x, int y ) { return x + y; }

/* C实现文件 cFile.c */
/* 这样会编译出错：#include "cExample.h" */
extern int add( int x, int y );

int main( int argc, char* argv[] )
{
    add( 2, 3 );   
    return 0;
}
```

​	注意extern "C"是C++中的符号，不能在C中使用，否则会编译报错。

### 2 右值引用

#### 1) 区分左值、右值

​	左值**可以取地址、位于等号左边**；而右值**没法取地址，位于等号右边**。

```c++
//a可以通过&取地址，位于等号左边，所以a是左值
//5位于等号右边，5没法通过&取地址，所以5是纯右值。
int a = 5;
```

​	右值能被细分为**纯右值**和**将亡值**：

​	纯右值： 就是指等号右边的常数；

​	将亡值：中间变量的过渡，过渡之后就消亡，可以细分两种：① 函数的临时返回值；② 表达式；

```c++
int main() {
    int x,y=10;
    x+y;               //将亡值:表达式
    int c = func(x,y); //将亡值:函数临时返回右值，拷贝给c后，消失
}
```

#### 2) 左值引用、右值引用

​	引用本质是别名，可以通过引用修改变量的值，传参时传引用可以避免拷贝，其**实现原理和指针类似**。

##### 2.1) 左值引用

​	左值引用就是对左值的引用，给左值取别名，通过“&”来声明：

```c++
int a = 5;
int &ref_a = a; // 编译通过，左值引用指向左值
int &ref_a = 5; // 编译失败，左值引用指向了右值
```

​	**由于右值没有地址，没法被修改，所以左值引用无法指向右值。**

​	但是，**const左值引用**可以指向右值的，const左值引用不会修改指向值，因此可以指向右值：

```c++
const int &ref_a = 5;  // 编译通过
```

##### 2.2) 右值引用

​	右值引用的标志是`&&`，专门为右值而生，**可以指向右值，不能指向左值**。

```c++
int &&ref_a_right = 5; //OK
ref_a_right = 6; //OK, 右值引用的用途：可以修改右值

int a = 5;
int &&ref_a_left = a; // 编译失败，右值引用不可以指向左值

int x, y;
x = 2; y = 3;
int &&r = x + y; //OK
int &&rr = func(x, y); //OK
const int &&crr = func(x, y); //OK
```

#### 3) 右值引用本质讨论

##### 3.1) std::move使右值引用指向左值

​	使用std::move让右值引用指向左值：

```c++
int a = 5; // a是个左值
int &ref_a_left = a; // 左值引用指向左值
int &&ref_a_right = std::move(a); // 通过std::move将左值转化为右值，可以被右值引用指向
 
cout << a; // 打印结果：5
```

​	上述代码中，左值a通过std::move移动到了右值ref_a_right中，且**a的值仍然是5**(不会丢失)。

​	**std::move本质不移动任何数据，其唯一的功能是把左值强制转化为右值**，其实现等同于类型转换：`static_cast<T&&>(lvalue)`，从而让右值引用可以指向左值。

​	**单纯的std::move(xxx)不会有性能提升**。

##### 3.2) 右值引用能指向右值

```c++
int &&ref_a = 5;
ref_a = 6;  
// 等价于下述代码 
int temp = 5;
int &&ref_a = std::move(temp);
ref_a = 6;
```

​	右值引用能指向右值，本质上也是把右值**提升**为一个左值，并定义一个右值引用通过std::move指向该左值。
