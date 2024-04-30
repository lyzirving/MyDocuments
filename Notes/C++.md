# C++三大特性

## 1 封装

​	目的：隐藏实现细节，实现模块化。

​	特性：① 定义类，定义类成员属性、成员方法。

​	② 通过访问权限(public、protected、private)控制，将合理的属性、成员函数暴露给外部，供对象与外部交互。

​	③ 可通过友元类打破private的限定。

## 2 继承

​	目的：无需修改基类的情况下，通过继承实现对基类功能的扩展。

​	特性：① 权限继承：

<img src=".\pic\c++_derive.png" alt="c++_derive" style="zoom:80%;" />

​	② 多继承。

​	③ 接口继承。

## 3 多态

​	目的：一个接口多种形态，通过实现接口重用，增强可扩展性。

​	特性：① 静态多态(函数重载、模版)；② 动态多态：虚函数。

# 模板

## 1 可变参数模板

### 1) 参数个数未知，参数类型相同

标准库类型initializer_list：容器类型，容器元素为常量。

### 2) 参数个数未知，参数类型可能不同

#### 2.1) 形式

```c++
template<class... T>      // T是模板参数包
void function(T ... args) // args是函数参数包
{
}
```

…代表一个包含0到n的任意参数包，其中的任意代表任意数目和任意类型。

…在参数右侧代表一组实参，…在参数左侧代表可变参数。

#### 2.2) 可变参数包展开

##### 2.2.1) 查询个数

```c++
template<typename... Args>
unsigned int length(Args... args) { return sizeof...(args); }
```

##### 2.2.2) 递归展开

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

##### 2.2.3) 逗号运算符展开

```c++
template<typename T>
void show_arg(const T& t) {}

template <typename ... Args>
void expand(const Args&... args)
{
	int arr[] = {(show_arg(args), 0)...}; 
}
```

​	解析如下：

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

## 2 模板实参推断(Template Argument Deduction)

​	编译器利用调用中的**函数实参来确定模板参数**，这个过程就叫模板实参推断。本小节参考自[模板实参推断](https://blog.csdn.net/qq_41453285/article/details/104447573)。

### 1) 模板实参推导和类型转换

​	将实参传递给带模板函数的函数形参时，能够自动应用的类型转换只有：**const转换**、**数组转换**、**函数到指针转换**。

​	const转换：将非const对象的引用/指针转化为const对象的引用/指针；

​	数组转换：数组实参可转换为一个指向其首元素的指针；

​	函数转换：函数实参可转换为该函数类型的指针。

```c++
template<typename T>
T fobj(T, T);
 
template<typename T>
T fref(const T&, const T&);
 
int main()
{
    string s1("a value");
    const string s2("another value");
    fobj(s1, s2); //OK，调用fobj(string,string); s2的const会忽略
    fref(s1, s2); //OK，调用fref(const string&,const string&); 将s1转换为const是允许的
 
    int a[10], b[42];
    fobj(a, b);  //OK, 调用f(int*,int*)
    fref(a, b);  //Error，数组类型不匹配，数组a和b的大小不同，所以类型不同
 
    return 0;
}
```

### 2) 模板实参推导和引用

#### 2.1) 左值引用函数参数推导

​	当模板函数参数是左值引用(形如T&)，则：

①**只能传递给它一个左值**(变量、返回引用类型的表达式等)；

②实参可以是const，也可以不是。

```c++
template<typename T>
void f1(T&) { }
 
int main()
{
    int i;
    const int ci=10; 
    f1(i);  //OK, T是int
    f1(ci); //OK, T是const int
    f1(5);  //Error, 5为右值
    return 0;
}
```

​	当模板函数参数是const T&，则：

①可以传递给它**任何类型的实参：**对象(const/非const)、临时对象、字面量常量；

②T的类型推断**不会是一个const类型**，因为const已经是函数参数的一部分。

```c++
template<typename T>
void f1(const T&) { }
 
int main()
{
    int i;
    const int ci=10;
 
    f1(i);  //OK, T是int
    f1(ci); //OK, T是int
    f1(5);  //OK, const&可以绑定到右值上, T是int
	return 0;
}
```

#### 2.2) 万能引用函数参数推导

​	万能引用模板中传入**左值实参**，模板参数会被推导为**左值引用**，并发生引用折叠：

```c++
template<typename T>
void f1(T&&){}
 
int main()
{
    int i;
    const int ci = 10;
 
    f1(42); //OK，42为右值，T为int 
    f1(i);  //OK，T是int&
    f1(ci); //OK，T是const int &
    return 0;
}
```

​	上述推导流程如下：

```c++
//f1(i)调用的伪代码形式
void f1<int&>(int& &&);
 
//又int& &&会被折叠为&
void f1<int&>(int&);
```

​	其他万能引用的推导场景，可参考：<a href="#2) 图解万能引用推导规则">图解万能引用推导规则</a>

## 3 逗号运算符

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

# 多态

## 1 静态多态

​	编译期间的多态，编译器在**编译期间**完成，通过实参类型推断出要调用的函数；

​	静态多态有两种实现方式：1.函数重载；2.模板。

​	**函数重载**的关键是**参数列表**：参数数目、类型和排序，与函数名和返回值无关。

​	**模板**使用泛型来定义函数，泛型可由具体的类型替换。通过将类型作为参数，传递给模板，通过编译器生成该类型的具体函数。

## 2 动态多态-虚函数

​	在程序执行期间判断所引用对象的实际类型，根据其实际类型调用相应的方法。

​	对于有虚函数的类，编译器会维护一张虚函数表(虚表)，对象的前四个字节就是指向虚表的指针(虚表指针)。

### 1) 虚函数表创建原则

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

### 2) 虚函数表和虚函数指针构建时机

- 虚函数构建时机

​	类中存在virtual关键字声明的函数时，编译器在**编译时**就会生成虚函数表。

​	虚函数表实际是一个**数组**，数组元素是各虚函数指针。

​	虚函数表存在于编译后生成的.**o文件**中。编译完成后，其所在的内存段就已确定，运行时被加载到进程的**全局数据区只读代码段中**。

- 虚函数指针的赋值时机

​	编译器会为存在虚函数的类**添加一个成员变量**：虚函数指针。

​	类对象在构造时，编译器会在构造器中为类对象的虚函数指针赋值。因此，若类存在虚函数，但没有构造函数，编译器会创建默认构造函数，就是为给虚函数指针赋值。

​	虚函数指针在类对象的**最前端**。

## 3 虚析构函数

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

# 内存布局

## 1 内存分区

### 1) 区域划分

​	在C++中，内存分成**5个区**，分别是：**堆、栈、自由存储区、全局/静态存储区、常量存储区**。

​	有的书籍也把内存划分为**堆区**、**栈区**和**静态存储区**，其中把自由存储区、全局/静态存储区和常量存储区总结为一块区域。

- **栈**：内存由编译器在需要时自动分配和释放。通常用来存储局部变量和函数参数、返回地址。栈运算分配内置于处理器的指令集中，**效率很高**，但是分配的内存容量有限。

- **堆**：使用new分配的内存，使用delete或delete[]释放。如果未能对内存进行正确的释放，会造成**内存泄漏**。程序结束时，由操作系统自动回收。

  其中，还包含**自由存储区**：使用malloc进行分配，使用free进行回收。

- **全局/静态存储区**：全局变量和静态变量被分配到同一块内存中，C语言中区分初始化和未初始化的，C++中不再区分。

  **虚函数表**存放在全局数据区。

- **常量存储区**：存储常量，不允许被修改。

-  **代码区**：存放代码，不允许修改，但可以执行。编译后的二进制文件存放在这里。

<img src="./pic/note_pic_mem_area.png" alt="note_pic_mem_area" style="zoom:100%;" />

### 2) 区域间的不同

#### 2.1) 静态存储区和栈区

```c++
char* p = “Hello World1”;   //1 指针p在栈区, "Hello World1"是字符串常量, 存储在静态存储区
char a[] = “Hello World2”;  //2 初始化分配的数组，所以a[]和"Hello World2"都在栈上 
p[2] = ‘A’;                 //3 抛出运行时异常, 因为*p指向的区域是常量区
a[2] = ‘A’;                 //4 可以修改，因为a[]在栈上
char* p1 = “Hello World1;”  //5 p1在栈上，且p和p1指向同一块内存
```

#### 2.2) 堆与栈

- 内存分配和释放：对于**栈**来讲，内存由编译器管理，无需手工控制；对于**堆**来说，分配和释放工作由程序员控制，容易产生memory leak。
- 空间大小：在32位系统下，**堆内存**可以达到4G的空间；**栈内存**有空间限制，在VC6.0下面默认的栈空间大小是1M。
- 内存碎片：对于**堆**，频繁的new/delete会造成内存空间的不连续，产生大量碎片，影响性能；对于**栈**，内存按照先进后出的方式管理，不会产生碎片。
- 生长方向：对于**堆**，生长方向是**向上**的，向着内存地址增加的方向；对于**栈**，生长方向是**向下**的，是向着内存地址减小的方向增长。
- 分配效率：对于**堆**，内存分配是C/C++函数库提供的，其机制复杂。为了分配一块内存，库函数会按照一定的算法在堆内存中搜索可用的足够大小的空间，如果没有足够大小的空间(可能是由于内存碎片太多)，就有可能调用系统函数去**增加程序数据段的内存空间**，从而获取到足够大小的内存，然后返回。上述可能会经历用户态和内核态的切换。对于**栈**，其是机器系统提供的数据结构，计算机会在**底层对栈提供支持**：分配专门的寄存器存放栈的地址，压栈出栈都有专门的指令执行，这就决定了栈的效率比堆更高。

## 2 虚函数表

​	若类中存在虚函数，编译器会为该类生成一个虚函数表，虚函数表按照虚函数的声明顺序存放了各个虚函数的地址。

​	虚函数表不存在于类中，虚函数表存放在**全局数据区**。

​	而对于每个类对象，编译器都会为其生成一个不可见的指针，**指向同一个地址**，这个指针就是虚函数表指针。

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

## 3 对象模型

- nonstatic 数据成员被置于每一个类对象中，static数据成员被置于类对象之外；
- static与nonstatic函数被置于类对象之外；
- virtual函数通过虚函数表+虚指针来支持；
- 虚函数表前设置了一个指向type_info的指针，用以支持RTTI。RTTI是为多态而生成的信息，包括对象继承关系，对象本身的描述等。

### 1) 非继承下的对象模型

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

### 2) 单继承下的对象模型

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

### 3) 多继承(非菱形继承)

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

### 4) 菱形继承

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

### 5) 虚继承

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

#### 5.1) 简单虚继承

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

#### 5.2) 菱形虚拟继承

```c++
class B {...}
class B1: virtual public B {...}
class B2: virtual public B {...}
class D : public B1, public B2 {...}
```

​	其中D的对象模型如下：

<img src="./pic/note_pic_vbptr3.png" alt="note_pic_vbptr3" style="zoom:70%;" />

### 6) 空壳类

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

## 4 字节对齐

### 1) 自然对界alignment

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

### 2) 改变缺省的对界

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

### 3) struct和union

#### 3.1) 定义区别

- 编译器会为struct的每个成员分配存储空间；
- union的所有成员共用一块存储空间。对union成员赋值，会重写其他成员。此时使用其他成员会出现未定义行为。
- union中占用内存最多的成员定义了整个union的内存大小。

#### 3.2) 对齐方式

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

# 语法

## 1 extern "C"

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

### 1) extern

​	`extern`是`C/C++`语言中表明函数和全局变量作用范围（可见性）的关键字，该关键字告诉编译器，其声明的函数和变量可以在本模块或**其它模块**中使用。

- 语句 `extern int a;` 仅是对变量的声明，并未为 `a` 分配内存空间。
- 引用全局变量/函数前，必须有变量/函数的声明(或者定义)。例如，模块`B`欲引用模块`A`中定义的全局变量/函数，需包含模块`A`的头文件。
- 编译阶段，模块`B`虽然找不到变量/函数，但不会报错；**链接阶段**`B`会从模块`A`编译生成的**目标代码**中找到变量/函数。
- 与`extern`对应的关键字是`static`，被它修饰的全局变量和函数只能在**本模块**中使用。

### 2) extern "C"修饰的变量/函数是按照C的方式编译和链接的

​	作为一种面向对象的语言，`C++`支持函数重载，过程式语言`C`不支持。

​	所以，函数被`C++`编译后在符号库中的名字与`C`语言的有所不同。有如下函数：

```c++
void foo( int x, int y );
```

- `C++`编译器则会产生像`_foo_int_int`之类的名字(不同的编译器生成的名字不同，但采用了相同机制，生成的新名字称为`mangled name`)。
- `C++`类成员变量/函数以`.`来区分，其可能与全局变量/函数重名。本质上，编译器编译时，也为类中的变量/函数取了一个独一无二的名字，与全局变量的名字不同。
- `C`由于没有实现重载，会产生`_foo`之类的名字。

### 3) extern "C"是实现C++和C程序混合编程所使用

#### 3.1) C++中引用C

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

#### 3.2) C中引用C++

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

## 2 右值引用(C++11)

### 1) 区分左值、右值

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

### 2) 左值引用、右值引用

​	引用本质是别名，可以通过引用修改变量的值，传参时传引用可以避免拷贝，其**实现原理和指针类似**。

#### 2.1) 左值引用

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

#### 2.2) 右值引用

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

### 3) std::move源码剖析

​	下述代码来自gcc-13.2.0：

```c++
template<typename _Tp>
constexpr typename std::remove_reference<_Tp>::type&& move(_Tp&& __t) noexcept
{ 
    return static_cast<typename std::remove_reference<_Tp>::type&&>(__t); 
}
```

① 利用引用折叠原理，经过万能引用传递，若实参是右值，即T推导为T；

​	若实参是左值，则变为普通的左值引用，即T推导为T&。综上，保证模板可以传递任意实参，且保持类型不变。

② 通过std::remove_reference移除引用，得到具体的类型T。

③ 最后通过static_cast<>进行强制类型转换，返回T&&右值引用。

### 4) 右值引用本质讨论

#### 4.1) std::move使右值引用指向左值

​	使用std::move让右值引用指向左值：

```c++
int a = 5; // a是个左值
int &ref_a_left = a; // 左值引用指向左值
int &&ref_a_right = std::move(a); // 通过std::move将左值转化为右值，可以被右值引用指向
std::cout << "a = " << a << ", ref_a_right = " << ref_a_right << std::endl; // 打印 a = 5, ref_a_right = 5

ref_a_right = 7;
std::cout << "a = " << a << ", ref_a_right = " << ref_a_right << std::endl; // 打印 a = 7, ref_a_right = 7
```

​	上述代码中，左值a通过std::move移动到了右值ref_a_right中，且**a的值仍然是5**(不会丢失)。

​	**std::move本质不移动任何数据，其唯一的功能是把左值强制转化为右值**，其实现等同于类型转换：`static_cast<T&&>(lvalue)`，从而让右值引用可以指向左值。

​	**单纯的std::move(xxx)不会有性能提升**。

#### 4.2) 右值引用能指向右值

```c++
int &&ref_a = 5;
ref_a = 6;  

// 等价于下述代码 
int temp = 5;
int &&ref_a = std::move(temp);
ref_a = 6;
```

​	右值引用能指向右值，本质上也是把右值**提升**为一个左值，并定义一个右值引用通过std::move指向该左值。

## 3 完美转发(C++11)

### 1) 万能引用

```c++
template<typename T>
void PerfectForward(T&& t)//万能引用
{
	//……
}
```

​	右值引用和万能引用的区别：右值引用需要`确定类型`，万能引用是根据传入实参的类型进行`推导`。

​	如果传入的实参是一个左值，那形参t就是左值引用；如果传入的实参是一个右值，那形参t就是右值引用。	

​	万能引用也叫做**引用折叠**，若传入的是个左值，那&&折叠成一个&；若传入的是一个右值，那还是&&。

### 2) 图解万能引用推导规则

<img src=".\pic\c++_universal_ref.png" alt="c++_universal_ref" style="zoom:100%;" />

<img src=".\pic\c++_universal_ref_1.png" alt="c++_universal_ref_1" style="zoom:100%;" />

<img src=".\pic\c++_universal_ref_2.png" alt="c++_universal_ref_2" style="zoom:100%;" />

<img src=".\pic\c++_universal_ref_3.png" alt="c++_universal_ref_3" style="zoom:100%;" />

### 3) std::forward

```c++
void Fun(int& x) { cout << "左值引用" << endl; }
void Fun(const int& x) { cout << "const 左值引用" << endl; }
void Fun(int&& x) { cout << "右值引用" << endl; }
void Fun(const int&& x) { cout << "const 右值引用" << endl; }

template<typename T>
void PerfectForward(T&& t) { Fun(t); }

int main()
{
	PerfectForward(10);//右值, 输出左值引用
	int a;
	PerfectForward(a);//左值, 输出左值引用
	PerfectForward(std::move(a));//右值, 输出左值引用
	const int b = 8;
	PerfectForward(b);//const左值, 输出const 左值引用
	PerfectForward(std::move(b));//const右值, 输出const 左值引用
	return 0;
}
```

​	实际条用Fun()时，通过参数t进行引用，已经退化为左值。

```c++
template<class T>
void PerfectForward(T&& t)
{
	Func(std::forward<T>(t));
}
```

​	通过std::forward进行转发，就能达到想要的效果。

## 4 const相关

### 1) 指针常量和常量指针

**判断指针常量和常量指针**：遵循指针的定义技巧，从指针标识符开始，由右往左读，const修饰最靠近它的那个。

**指针常量(constant pointer)**：指针修饰的常量，指针指向的**地址不能被修改**，但地址里的内容可以被修改；

```c++
int a = 8;
int* const p = &a; //指针常量, const靠近变量
*p=9; //OK
p=&b; //Error
```

**常量指针(pointer to const)**：定义指针变量的时，数据类型前用const修饰，该指针即指向常量的指针。指针的地址可以被修改，但**地址的内容不能被修改**。

```c++
int a，b;
const int *p = &a; //常量指针, const靠近被修饰的类型
*p=9; //Error
p=&b; //OK
// ---------------------------------------------
int const *q = &a; //同p
```

### 2) 顶层const和底层const

#### 2.1) 定义

​	顶层const：指针常量，指针的指向不能被改变，称其为顶层const属性；

​	底层const：常量指针，指针指向的地址内容是个常量，称其为底层const属性。

```c++
int* const p1 = &a;//p1是顶层const
const int* p2 = &a;//p2是底层const
```

#### 2.2) 赋值规律

- 顶层const在赋值给其他变量时，**可以忽略**顶层属性；
- 底层const在赋值给其他变量时，**不能忽略**底层属性；
- int*类型可以转换为顶层和底层const，所以它可以给顶层和底层的const赋值；
- 底层const**无法转换**为顶层const。

#### 2.3) 示例

```c++
int a = 2,
int *p = &a;

const int *p1 = p; // (1) OK, int*可以转换为底层const
int* const p2 = p; // (2) OK, int*可以转换为顶层const

int* const p3 = p1; // (3) Error, p1为底层const, 赋值时不能忽略
const int* p4 = p2; // (4) OK, p2为顶层const, 赋值时可以忽略

int* p5 = p1; // (5) Error, 原因同(3)
int* p6 = p2; // (6) OK, 原因同(4)

const int* const p7 = p1; // (7) OK, p7具有底层属性, 且p1有底层属性
const int* const p8 = p2; // (8) OK, p2具有顶层属性, 可被忽略

const int* p10 = p7; // (9) OK, p10具备底层属性
int* const p11 = p7; // (10) Error, p11没有底层属性, 但p7有, 不能忽略
```

## 5 auto相关(C++11)

### 1) auto推导规则

- 规则1：声明为**auto**的变量，**忽视**掉初始化表达式的**顶层const**。

  对有const的普通类型(int 、double等)，忽略const；

  对指针常量(顶层const)变为普通指针；

  对常量指针(底层const)变为指向常量指针(保持底层const属性)。

- 规则2：声明为**auto&**的变量，**保持**初始化表达式的顶层**const**或**volatile**属性。

- 规则3：若希望auto推导的是顶层const，加上const，即**const auto**。

### 2) 示例

#### 2.1) auto示例

```c++
int i = 0, &ri = i;

auto a = i;   //a为int型变量
auto a1 = ri; //a1为int型变量

auto p = &i;   //&i是int*, p是int*
auto p1 = &ri; //同上
// ----------------------------------------------------
const int ci = 2, &rci = ci , ci2 = 9;

auto b = ci;   //b为int型变量: 规则1忽略顶层const
auto b1 = rci; //同上
b = 4; b1 = 5; //b和b1的值可以改变

auto cp = &ci; //cp是常量指针const int*，保持底层const
cp = &ci2;     //cp的指向可以改变

// ----------------------------------------------------
int z = 9, z1 = 10;

int* const pz1 = &z;       //指针常量(顶层const)
const int* pz2 = &z;       //常量指针(底层const)
const int* const pz3 = &z; //同时包含底层和顶层const

auto apz1 = pz1;//apz1为int*, 忽略顶层const
auto apz2 = pz2;//apz2为const int*, 保持底层const
auto apz3 = pz3;//apz3为const int*, 保持底层const
```

#### 2.2) auto&示例

```c++
int i = 0, &ri = i;
const int ci = 2, &rci = ci;
	
auto& j = i;  //j为int &
auto& k = ci; //k为const int&
auto& h = 42; //Error, 不能将非常量引用绑定字面值, 这是引用&规则决定的

const auto& j2 = i;  //j2为const int&，因为规则3, j2被提升为顶层const
const auto& k2 = ci; //k2为const int&
const auto& h2 = 42; //正确，可以为常量绑定字面值 

auto& m = &i;        //Error，无法从“int *”转换为“int *&”
auto& m1 = &ci;      //Error，无法从“const int *”转换为“const int *&”, 这是引用&规则决定的
const auto& m2 = &i; //m2为int* const&
const auto& m3 = &ci;//m3为const int* const&
```

#### 2.3) const auto示例

```c++
int i = 0, &ri = i;
const int ci = 2, &rci = ci ;

const auto cb = i;   //cb为const int型。规则3, cb被提升为const
const auto cb1 = ci; //同上

const auto ca1 = &i; //cal为指针常量。&i本是int*，因为规则3，强行将cal提升为指针常量int *const
const auto ccp = &ci;//本来&ci为const int *，因为规则3，加了const后，提升为const int* const
```

### 3) auto的作用

- 代替冗长复杂的变量声明；

- 定义模板参数时，用于声明依赖模板参数的变量：

  ```c++
  template <typename _Tx,typename _Ty>
  void Multiply(_Tx x, _Ty y)
  {
      auto v = x+y;
      std::cout << v;
  }
  ```

- 定义模板函数依赖于模板参数的返回值类型：

  ```c++
  template <typename _Tx, typename _Ty>
  auto multiply(_Tx x, _Ty y)->decltype(x*y)
  {
      return x*y;
  }
  ```

### 4) auto推断的原理

​	本小节内容参考自[https://www.zhihu.com/question/294048058](https://www.zhihu.com/question/294048058)。

#### 4.1) auto使用模板实参推断机制

​	auto使用的是**模板实参推断(*Template Argument Deduction*)**的机制；

​	编译器产生一个函数模板，auto被模板类型参数T替代，把待推导的变量作为函数**实参**。

```c++
template<typename Container>
void useContainer(const Container& container)
{
    auto pos = container.begin();  //(1)第一处推导
    while (pos != container.end())
    { 
        auto& element = *pos++;    //(2)第二处推导
        … // 对元素进行操作
    }
}
```

​	第一处推导等价如下：

```c++
// auto pos = container.begin();
----------------------------------
template<typename T>
void deducePos(T pos);

deducePos(container.begin());
```

​	此时T的推导类型就是auto。

​	第二处推导等价如下：

```c++
// auto& element = *pos++;
-------------------------------
template<typename T>
void deduceElement(T& element);

deduceElement(*pos++);
```

#### 4.2) auto和初始化列表

​	对于初始化列表，auto会将其视为**std::initializer_list**，但模板不能对其进行推断。

```c++
auto x = { 1, 2 }; // C++14禁止了对auto用initializer_list直接初始化，必须用=
auto x2 { 1 };    // 保留了单元素列表的直接初始化，但不会将其视为initializer_list
std::cout << typeid(x).name();  // class std::initializer_list<int>
std::cout << typeid(x2).name(); // C++14中为int

--------------------------------
template<typename T>
void deduceX(T x);
deduceX(x); // 错误：不能推断T
```

## 6 decltype(C++11)

### 1) decltype和auto的区别

​	C++ Primer中写道：`有时希望从表达式的类型推断出要定义的变量的类型`，`同时不想用该表达式的值初始化变量`。

​	auto推导变量依赖于初始化它的表达式，且auto声明的变量**必须被初始化**；

​	decltype是直接通过某一个表达式来获取数据类型，**不用使用表达式的值**。

```c++
int a = 10, b = 11;
auto c = a + b;    //c为int型
decltype(a + b) d; //d为int型
```

### 2) decltype用法

#### 2.1) decltype变量

```c++
decltype(var)
```

​	与auto不同，decltype会**保留**const属性和引用属性：

```c++
const int ci = 0, &cj = ci;
--------------------- decltype --------------------
decltype(ci) x = 0; //x的类型为const int
decltype(cj) y = x; //y的类型为const int&
decltype(cj) z;     //Error，z的类型为const int&，必须初始化
----------------------- auto ----------------------
auto w = ci;//w的类型是int， 忽略顶层const
w = 9;
auto n = cj;//n的类型是int
```

#### 2.2) decltype表达式

```c++
decltype(expr)
```

​	表达式作右值，推导为该数据类型：

```c++
int i = 42, &r = i;
decltype(r + 0) b; //b类型是int，而不是int&
```

​	表达式作左值，推导为该类型的引用：

```c++
int ii = 42, *p = &ii;
decltype(*p) c;   //Error, c是int&，必须初始化
decltype((ii)) d; //Error, ii是变量, (ii)是表达式, 且ii可以被赋值, 所以d是int&，必须初始化
```

#### 2.3) decltype函数

##### 2.3.1) decltype(f())

```c++
decltype(f()) sum = x; 
```

​	sum的类型就是假如函数f被调用，其返回的类型。**注意**：若函数的返回值为**void**，编译报错。

```c++
template <typename T>
T add(T a, T b) {	return a+b; }

decltype(add(1,2)) m = 10;      //m的类型是int
decltype(add(1.0,2.0)) m2 = 20; //m2的类型是double
```

##### 2.3.2) decltype(f)

​	下述例子中，`decltype(add_to)`直接返回函数类型，所以pf是一个函数指针。

```c++
int add_to(int a, int b) { return a + b; }

decltype(add_to) *pf = add_to; //pf就是一个函数指针，类型为int(int,int)
pf(1,2);
```

​	如果函数是**模板的**、**重载的**，无法通过函数名来推导函数指针的类型。

### 3) decltype主要作用

​	用于申明返回值类型依赖于其参数类型的模板函数：

```c++
template <typename _Tx, typename _Ty>
auto multiply(_Tx x, _Ty y)->decltype(x*y)
{
    return x*y;
}
```

​	注意这里的auto没有做任何类型推断，只是用来表明这里使用的是C++11的**拖尾返回类型**`(trailing return type)`语法：函数返回类型在参数列表之后进行声明(在"->"之后)。

​	拖尾返回类型的优点：可以使用函数参数来声明函数返回类型。

## 7 智能指针(C++11)

​	本小节参考自[https://blog.csdn.net/ithiker/article/details/51532484](https://blog.csdn.net/ithiker/article/details/51532484)。

### 1) 简介

​	std::unique_ptr：**独享**被管理对象，同一时刻只能有一个unique_ptr拥有对象的所有权，当其被赋值时对象的所有权也发生**转移**，当其被销毁时被管理对象也自动被销毁。

​	std::shared_ptr：**共享**被管理对象，同一时刻可以有多个std::shared_ptr拥有对象的所有权，当最后一个shared_ptr对象销毁时，被管理对象自动销毁。

​	std::weak_ptr：**不拥有**对象的所有权，但是它可以判断对象是否存在和返回指向对象的shared_ptr类型指针；它的用途之一是解决多个对象内部含有**std::shared_ptr循环指向**，导致对象无法释放的问题。

### 2) 类结构及其作用

<img src=".\pic\c++_shared_ptr.png" alt="c++_shared_ptr" style="zoom:90%;" />

- std::shared_ptr类结构

​	shared_ptr内部含有一个指向**被管理对象(managed object)**T的指针以及一个\__shared_count类对象。

​	\__shared_count类对象包含一个指向**管理对象(manager object)**的基类指针；

​	管理对象由具有原子属性的\_M_use_count、\_M_weak_count、被管理对象T的指针、以及用来销毁被管理对象的deleter组成。

​	\__M_use_count主要用来标记**被管理对象**的生命周期；

​	\__M_weak_count主要用来标记**管理对象**的生命周期。

<img src=".\pic\c++_shared_ptr_class.png" alt="c++_shared_ptr_class" style="zoom:50%;" />

- std::weak_ptr类结构

​	weak_ptr类结构和shared_ptr相似，不过管理对象的变为了\__weak_count(子类)。

<img src=".\pic\c++_weak_ptr_class.png" alt="c++_weak_ptr_class" style="zoom:50%;" />

- **两个被管理对象指针**

​	在shared_ptr中，被管理对象的指针有**两个**。

​	① shared_ptr直接包含的裸指针是为了实现一般指针的->,\*等操作；

​	② \__shared_count间接包含的指针是为了管理对象的生命周期，回收相关资源。

- **__weak_count**

​	\__weak_count类相关的赋值、拷贝、析构只会影响到\_M_weak_count的值，对\_M_use_count没有影响；

​	当\_M_weak_count为0时，释放管理对象，也就是说weak_ptr不影响被管理对象的生命周期。

- 分享管理对象指针

​	当weak_ptr、shared_ptr自身或相互赋值时，它们共享同一个管理对象指针。

## 8 NULL和nullptr区别(C++11)

### 1) C语言中的NULL

​	C语言中，NULL被定义为：**#define NULL ((void \*)0)**，NULL实际是个空指针。

```c
int  *pi = NULL;//OK, C语言中发声隐式类型转换
char *pc = NULL;//OK, C语言中发声隐式类型转换
```

### 2) C++中的NULL

​	C++是**强类型语言**，void***不能隐式转换**为其他类型的指针。

```c++
#ifdef __cplusplus
#define NULL 0
#else
#define NULL ((void *)0)
#endif
```

​	**可以看到NULL在C++中是0**，有如下例子：

```c++
void func(void* i) { cout << "func1" << endl; }
 
void func(int i) { cout << "func2" << endl; }
 
void main(int argc,char* argv[])
{
	func(NULL);
	func(nullptr);
}
// ------------ 输出 ------------
func2
func1
```

​	使用NULL，重载了形参为int的函数。

​	因此，在C++程序中用**NULL代替空指针存在二义性**。

### 3) C++中的nullptr

​	**为解决NULL代指空指针存在的二义性问题**，在C++11中引入了nullptr这一新的关键字来代指空指针。

​	C++11引入的nullptr，保证在**任何情况下都代表空指针**。建议，**NULL只作为0来使用**。

​	[NOTE]在没引入C++11的nullptr时，可通过下述方法来解决二义性问题：

```c++
const class nullptr_t
{
public:
    template<class T>
    inline operator T*() const { return 0; } // 类型转换运算符， 转换为指针
 
    template<class C, class T>
    inline operator T C::*() const { return 0; } // 类型转换运算符，转换为数据成员指针
    
private:
void operator&() const;
};
```

## 9 隐式类型转换操作符

​	隐式类型转换操作符形式如下：

```c++
operator type() const;
```

​	有如下注意事项：

- 不允许转换成数组或者函数类型，但**允许**转换为指针（包括数组指针以及函数指针）或者引用类型
- 类型转换运算符**没有显式的返回类型**，也**没有形参**
- 必须定义成类的**成员函数**
- 类型转换函数通常应该为const类型
- 并且转换的类型要与return结果类型相同
- 类类型转换函数不能调用

​	正确使用示例如下：

```c++
class SmallInt {
private:
    std::size_t val;
public:
    SmallInt(int i = 0) :val(i) { if (i < 0 || i>255) { throw std::out_of_range("Bad SmallInt value");} }
    operator int() const { return val; } //隐式类型转换
    explicit operator float() const { return static_cast<float(val)>; }//显示类型转换, 编译器不会自动执行
};
// ------------- 示例 ---------------
SmallInt si;
si = 4; //首先将4隐式转换为SmallInt，然后调用SmallInt::operator=()
std::cout << si + 3 << endl; //调用SmallInt::operator int(); 首先将si隐式地转换为int，然后执行整数的加法。
```

## 10 constexpr

​	constexpr表达式是指**值不会改变**且在**编译过程**就能得到结果的表达式。

​	最基础的常量表达式就是字面值或全局变量/函数的地址或sizeof等关键字返回的结果。

​	其它常量表达式都是由基础表达式通过各种确定的运算得到的，constexpr值可用于enum、switch、数组长度等场合。

```c++
constexpr int mf = 20;  //OK, 20是常量表达式
constexpr int limit = mf + 1; //OK, mf + 1是常量表达式
constexpr int sz = size(); //Error, 除非size()是constexpr函数

constexpr int Inc( int  i) { return  i + 1; }
constexpr int a = Inc(1); // OK
```

​	constexpr的好处：

- 是一种很强的约束，更好地保证程序的正确语义不被破坏；
- 编译器可以在编译期对constexpr的代码进行优化，比如将用到的constexpr表达式都直接替换成最终结果等；
- 相比宏来说，没有额外的开销，但更安全可靠

​	constexpr还有很多用法，下面只调了部分进行说明，参考自[https://blog.csdn.net/janeqi1987/article/details/103542802](https://blog.csdn.net/janeqi1987/article/details/103542802)。

### 1) constexpr赋予变量顶层const属性

```c++
const int*p = nullptr;        //p是一个指向整形常量的指针
constexpr int* q = nullptr;   //q是一个指向整数的常量指针

int i = 100;
int j = 200;
constexpr int* k = &i;
*k = 8; //OK
k = &j; //Error
```

### 2) if constexpr

​	if constexpr必须是编译期能确定结果的常量表达式。条件结果一旦确定，编译器将只编译符合条件的代码块。

### 3) constexpr函数

​	constexpr能定义常量表达式函数，即constexpr函数。

​	常量表达式函数的返回值可以在编译阶段就计算出来。不过在定义常量表示函数的时候，我们会遇到更多的约束规则。

## 11 volatile关键字

### 1) volatile的特性

#### 1.1) 易变性

​	`volatile`提醒编译器它后面所定义的变量随时都有可能改变，因此编译后的程序每次必须从内存中读取变量的数据。

​	假设有写、读两条语句，依次对同一个 `volatile` 变量进行操作，那么后一条的读操作不会直接使用前一条的写操作对应的的**寄存器**里的内容，而是重新**从内存中读取**该 `volatile` 变量的值。

​	编译器有时候会从寄存器处取变量的值，而**不是每次都从内存中取**。因为① 编译器认为变量并没有变化，所以认为寄存器里的值是最新的。② 访问寄存器比访问内存要快很多，编译器通常为了效率，可能会读取寄存器中的变量。

​	但是，**变量在内存中的值可能会被其它元素修改**，比如：硬件或其它线程等。

#### 1.2) 不可优化

​	编译器不会对 `volatile` 声明的变量进行各种激进的优化(甚至将变量直接消除)，保证代码中的指令一定会被执行。

```c++
volatile int nNum;  // 将nNum声明为volatile
nNum = 1;
printf("nNum is: %d", nNum);
```

​	上述代码中，如果变量 `nNum` 没有声明为 `volatile` 类型，则编译器在编译过程中中就会对其进行优化，直接使用常量“1”进行替换，这样优化之后，生成的汇编代码很简介，执行时效率很高。

​	当使用 `volatile` 进行声明后，编译器则不会对其进行优化，nNum 变量仍旧存在，编译器会将该变量从内存中取出，放入寄存器之中，然后再调用 printf() 函数进行打印。

#### 1.3) 顺序性

​	`volatile` 变量之间的顺序性不会被**编译器**进行乱序优化。

### 2) volatile不能保证原子性

​	`volatile`不保证原子性。它只是保证每次都做内存访问、编译器不做优化。要实现原子性需要加锁完成。

​	有下述伪代码：

```c++
// global shared data
bool flag = false; // (1)

thread2(Type* value) 
{
    value->update(/* parameters */);
    flag = true;
    return;
}

thread1() 
{
    flag = false; // (2)
    Type* value = new Type;
    thread2(value);
    while (true)
    {
        if (flag == true) // (3)
        {
            apply(value);
            break;
        }
    }
    thread2.join();
    if (nullptr != value) { delete value; }
    return;
}
```

​	上述代码的语义是：thread1等待thread2将value更新后，再继续执行。

​	对于多线程编程，上述代码有两个问题：

① 在thread1中，从(2)到(3)，代码没有对flag修改，因此编译器可能将`if(flag == true)`优化掉；

② 在thread2中，尽管逻辑上`update()`在`flag = true`前执行，但编译器和CPU并不知道。因此实际执行时，二者顺序可能发生变化。

​	假如在(1)，将flag声明为`volatile`的，由于`if(flag == true)`是对volatile变量的访问，因此编译器不会优化它，从而肯定能保留该条件的判断，问题①得到了解决。但是问题②仍然存在。

​	若把`value`也声明为volatile，如`volatile Type *value = new Type;`，那么问题②能解决吗？

​	`volatile` 只作用在编译器上。代码最终是要运行在 CPU 上的。尽管编译器不会将(3)处换序，但CPU的乱序执行仍然可能交换`value` 和 `flag` 的赋值顺序。且new操作符不是原子的，它要执行分配空间、构造调用、指针赋值这三个操作。

​	最终，只能通过**atomic**或**加锁**来修改上述程序。

## 12 inline关键字

### 1) inline关键字的作用

​	允许一个函数在**多个编译单元中重复存在**(**每个源文件**即一个编译单元)。

### 2) 内联函数

​	内联函数以**代码膨胀为代价**，省去了**函数调用的开销**，从而提高执行效率。

​	**每一处**内联展开都要复制代码，将使程序的总代码量增大，消耗更多的内存空间。下述函数默认是内联函数：

① 模板函数默认是内联函数；

② 类定义中直接定义的成员函数，默认是内联函数。

### 3) inline关键字和内联函数没有关系

​	本小节主要参考自[C++ inline有什么用？](https://www.zhihu.com/question/24185638)。

​	现代的编译器在决定是否将函数执行内联展开时，**不参考**函数声明中inline修饰符。

​	inline关键字不仅能修饰函数，也可修饰命名空间(C++11以后)，修饰变量(C++17以后)。

​	inline主要作用是允许**同一个函数或变量的定义出现在多个编译单元之中**。

#### 3.1) inline修饰函数

##### (1) 函数在头文件中

```c++
/* foo.h */
inline int foo(int x) {
    static int factor = 1;
    return x * (factor++);
}

/* bar1.cc */
#include "foo.h"
int bar1() {
    return foo(1);
}

/* bar2.cc */
#include "foo.h"
int bar2() {
    return foo(2);
}

/* main.cc */
int Bar1(), Bar2();
int main() {
    return Bar1() + Bar2();
}
```

​	编译源文件，并链接生成可执行程序，有：

```c++
g++ -c main.cc bar1.cc bar2.cc -fno-gnu-unique  # ok
g++ -o main main.o bar1.o bar2.o                # ok
./main; echo $? # 5
```

​	上述没有发生multiple definition错误，并且main的输出表明两次调用使用了同一个局部静态变量factor。

​	使用**readelf**查看输出的可执行main文件的**符号表**，

```c++
readelf -s main
// --------------------------
Num:    Value          Size Type    Bind   Vis      Ndx Name
...
49: 0000000000004010    4 OBJECT  WEAK   DEFAULT   23 _ZZ3FooiE6factor
...
53: 000000000000115f    32 FUNC    WEAK   DEFAULT   14 _Z3Fooi
...
```

​	可以发现main中Foo和静态变量factor的定义只有一份，且Foo和factor都是**WEAK**符号。

##### (2) 函数在源文件中

```c++
/* bar1.cc */
inline int Foo(int x) {
    static int factor = 1;
    return x * (factor++);
}
int Bar1() {
    return Foo(1);
}

/* bar2.cc */
inline int Foo(int x) {
    static int factor = 2;
    return x * (factor++);
}
int Bar2() {
    return Foo(2);
}

/* main.cc */
int Bar1(), Bar2();
int main() {
    return Bar1() + Bar2();
}
```

​	对于上述例子，编译器很可能根据**源文件的编译顺序**从而决定使用哪个Foo：

```c++
g++ -o main main.cc bar1.cc bar2.cc -fno-gnu-unique # ok
./main; echo $? # 5
g++ -o main main.cc bar2.cc bar1.cc -fno-gnu-unique # ok
./main; echo $? # 8
```

​	故应该尽量避免上述情况发生：既无法保证对方的定义与你相同，也无法保证链接器最终选择的定义。

​	如果此时有个编译单元中的Foo没有声明为inline，该单元对应的版本的符号是**全局的强符号**，链接器在面对多个弱符号和一个强符号时一定会采用强符号对应的定义。因此该版本的定义会覆盖其它单元所定义的inline版本。

​	如果一定要在多个编译单元中定义同名函数，要么将其声明为**static**，要么将其声明在**不同的命名空间中**。

#### 3.2) inline修饰命名空间(C++11)

#### 3.3) inline修饰变量(C++17)

# 标准库数据结构

## 1 map和unordered_map

### 1) map

​	map内部实现了一个**红黑树**。红黑树有**自动排序**的功能，因此map内所有元素是有序的。

​	map中的元素是按照二叉树(二叉查找树)存储的：**左子树**上所有节点的键值都小于根节点的键值，**右子树**所有节点的键值都大于根节点的键值。使用**中序遍历**可将键值按照从小到大遍历出来。

​	红黑树的每一个节点都代表着map的一个元素。因此，对于map进行的查找，删除，添加等一系列的操作都相当于是对红黑树进行的操作。

​	pros：① 有序；② 基于红黑树实现，查找的时间复杂度是O(nlogn)

​	cons：① 空间占用率高。

### 2) unordered_map

​	unordered_map内部实现了一个**哈希表**，查找的时间复杂度可达到O(1)，其在海量数据处理中有着广泛应用。

​	因此，其元素的排列顺序是**无序的**。

## 2 vector

​	vector是**线性的连续空间**，底层存储是一个**数组**，可变大小的数组，支持随机访问 O(1) 。

​	在**尾部位置**插入/删除是O(1)，在其他位置插入/删除 O(N)。

```c++
struct _Vector_impl : public _Tp_alloc_type {
	pointer 	_M_start;
	pointer 	_M_finish;
	pointer		_M_end_of_storage;
};
```

​	`start` 和 `finish` 之间是已经被使用的空间范围，即 `vector.size()` 的大小。

​	`start` 和 `end_of_stroage` 之间是vector底层数组的整个空间，即 `vector.capacity()` 的大小。

### 1) vector扩容&迭代器问题

​	vector在增加元素时，如果超过自身最大的容量，vector则将自身的容量扩充为**原来的两倍**。

​	扩充空间需要经过的步骤：①完全弃用现有的内存空间，重新申请更大的内存空间；②将旧内存空间中的数据，按原有顺序移动到新的内存空间中；③释放旧的内存空间。

​	完成扩容后，再插入新增的元素。

​	一旦vector空间重新配置，则指向原来vector的**所有迭代器**都失效了，因为**vector的地址改变了**。

### 2) 深入vector扩容机制

​	先上结论：

- vector在push_back以成倍增长可以在均摊后达到**O(1)**的时间复杂度，相对于增长指定大小的**O(n)**时间复杂度更好。
- 为了防止申请内存的浪费，现在使用较多的有2倍与1.5倍的增长方式，而1.5倍的增长方式可以更好的实现对**内存的重复利用**。

#### 2.1) 以倍数的方式扩容

​	本小节参考自：[面试题：C++vector的动态扩容，为何是1.5倍或者是2倍](https://blog.csdn.net/qq_44918090/article/details/120583540)。

​	假如以**等长个数**扩容：即每次扩容时，容量增加相同个数k。现要扩容至n，操作的时间复杂度约为O(n)：

<img src=".\pic\c++_vector_enlarge_k.png" alt="c++_vector_enlarge_k" style="zoom:80%;" />

​	假如以倍数扩容，现要扩容至n，时间复杂度约为O(1)：

<img src=".\pic\c++_vector_enlarge_times.png" alt="c++_vector_enlarge_times" style="zoom:80%;" />

​	综上，以倍数扩容的效率更高。

#### 2.2) 选择1.5、2倍的方式扩容

​	扩容原理为：申请新空间，拷贝元素，释放旧空间，**理想的分配方案是在第N次扩容时，复用之前N-1次释放的空间。**

​	若按照2倍方式扩容，第i次扩容空间大小如下：1、2、4、8、16、32........2^i。

​	比如，第4次扩容时，所需空间是16，前述空间是1+2+4+8=15，小于16。每次需要的新空间都大于之前分配的空间，**内存肯定得不到复用**。

​	若按照1.5倍方式扩容，第i次扩容空间大小如下：1、2、3、4、6、9、13、19、28........。

​	可以看到$c(n-2) + c(n-1) >= c(n)$的，因此，在理论上，空间是能重用的。

​	若选择 >2 的倍数，空间肯定不能复用。

​	STL标准并没有严格说明需要按何种方式进行扩容，因此不同的实现厂商都是按照自己的方式扩容的，即：linux下是按照2倍的方式扩容的，而vs下是按照1.5倍的方式扩容的。

### 3) 快速删除对象(要求O(1)的时间复杂度)

​	将要删除的对象和尾部对象swap，然后直接pop_back即可。

### 4) vector减容

​	vector不会自动减容，pop_back后vector也不会减容，只是size变化，但capacity不会变。

​	回收不必要的内存，可调用函数shrink_to_fit()，使capacity和size一致。

​	可以使用swap()，用空的vector对内存清空。

### 5) 从内存的角度解释数组遍历和链表遍历的快慢

​	数组元素在内存中是**连续相邻的**，链表元素在内存中**离散、不连续的**。

​	CPU寄存器在读取数据时，会从缓存、内存中查找数据。缓存的最小处理单元是**缓存行**(连续的内存块)。

​	读取数组数据时，连续的数组成员会被加载到同一个缓存行中，因此不需要多次读取。

​	读取链表时，由于链表元素的地址是离散的，因此在处理多个元素时，可能需要**加载多个缓存行**。

## 3 空间配置器

​	allocator是STL的重要组成，一般用户不怎么熟悉它，因为allocator隐藏在所有容器身后，默默**完成内存配置与释放，对象构造和析构的工作**。

<img src=".\pic\c++_allocator.png" alt="c++_allocator" style="zoom:80%;" />

​	上图中，左边是用户代码，右边是STL内部实现。

​	vector的模板参数class T被替换为int，同时第二个模板参数因为没有指定，所以为**默认模板参数**，即`allocator<int>`。

​	SGI STL 实现了两个allocator：一个是标准的std::allocator，另一个是特殊的std::alloc。

### 1) 符合STL接口的自定义空间分配器

```c++
#include <new>
#include <cstddef>
#include <cstdlib>
#include <climits>
#include <iostream>

namespace my_alloc
{
    // allocate的实际实现，简单封装new，当无法获得内存时，报错并退出
    template <class T>
    inline T* _allocate(ptrdiff_t size, T*) {
        set_new_handler(0);
        T* tmp = (T*)(::operator new((size_t)(size * sizeof(T))));
        if (tmp == 0) {
            cerr << "out of memory" << endl;
            exit(1);
        }
        return tmp;
    }

    // deallocate的实际实现，简单封装delete
    template <class T>
    inline void _deallocate(T* buffer) { ::operator delete(buffer); }

    // construct的实际实现，placement new在分配的内存上调用对象的构造函数
    template <class T1, class T2>
    inline void _construct(T1* p, const T2& value) { new(p) T1(value); }

    // destroy的实际实现，直接调用对象的析构函数
    template <class T>
    inline void _destroy(T* ptr) { ptr->~T(); }

    template <class T>
    class allocator {
    public:
        typedef T           value_type;
        typedef T*          pointer;
        typedef const T*    const_pointer;
        typedef T&          reference;
        typedef const T&    const_reference;
        typedef size_t      size_type;
        typedef ptrdiff_t   difference_type;

        // 构造函数
        allocator() {}
        
        template <class U>
        allocator(const allocator<U>& c){}

        // rebind allocator of type U
        template <class U>
        struct rebind { typedef allocator<U> other; };

        // allocate，deallocate，construct和destroy函数均调用上面的实际实现
        // hint used for locality. ref.[Austern],p189
        pointer allocate(size_type n, const void* hint = 0) 
        {
            return _allocate((difference_type)n, (pointer)0);
        }
        void deallocate(pointer p, size_type n) { _deallocate(p); }
        void construct(pointer p, const T& value) { _construct(p, value); }
        void destroy(pointer p) { _destroy(p); }

        pointer address(reference x) { return (pointer)&x; }
        const_pointer const_address(const_reference x) { return (const_pointer)&x; }

        size_type max_size() const { return size_type(UINT_MAX / sizeof(T)); }   
    };
} // end of namespace myalloc
```

​	现在，可以使用自己编写的allocator来为vector分配空间：

```c++
std::vector<int, my_alloc::allocator<int> > v;
```

### 2) SGI STL 标准配置器(std::allocator)

​	这个allocator部分符合STL标准，它在文件 defalloc.h 中实现。

​	但是SGI STL的容器并不使用它，它存在的意义仅在于为用户提供一个兼容老代码的折衷方法，其实现仅仅是对**new和delete的简单包装**。这里我们不再深究。

### 3) SGI STL 特殊配置器(std::alloc)

​	这个配置器是SGI STL的默认配置器，它在`<memory>`中实现。

​	其**主要作用**是为了解决内存的申请和释放时引入的**内存碎片**问题，SGI使用的方法是 “双层级配置器”。

<img src=".\pic\c++_alloc.png" alt="c++_alloc" style="zoom:95%;" />

​	std::alloc接口如下：

- `static T* allocate()`函数负责空间配置，返回一个T对象大小的空间。
- `static T* allocate(size_t)`函数负责批量空间配置。
- `static void deallocate(T*)`函数负责空间释放。
- `static void deallocate(T*,size_t)`函数负责批量空间释放。

​	当配置区块**大于128 bytes**时，调用第一级配置器。第一级直接调用 malloc()、deallocate()、free()等系统调用分配内存。

​	当配置区块**小于128 bytes**时，调用第二级配置器。第二级配置器实现了**内存池**和**自由链表**：配置维护16个自由链表，负责16种小型区块的配置能力。当程序多次进行小空间的配置时，可以从内存池和自由链表中获取空间，减少系统调用，提升性能。

​	最终，它们都是用`malloc()`和`free()`来配置和释放空间。

## 4 list

​	list的底层是一个**双向链表**，以结点为单位存放数据，结点的地址在内存中**不连续**，每次插入或删除一个元素，就配置或释放一个元素空间。

​	list按需申请/释放内存，不需要扩容；

​	list中元素地址不连续，不支持下标随机访问，CPU高速缓存命中率低。

​	list插入、查找、删除时间复杂度分别为：O(1)、O(n)、O(1)。其中，插入/删除元素效率高，因为只需修改相邻节点的指针。

## 5 deque

​	双端队列，它维护了**两级**的连续空间。第一级由**数组**组成，维护各段等长连续空间的**地址**；第二级是各段连续空间，存放实际的数据。

### 1) 底层实现

​	deque 容器需要维护 map 数组、start、finish 迭代器，如下图所示：

<img src=".\pic\c++_deque.png" alt="c++_deque" style="zoom:65%;" />

​	start迭代器记录着map数组中首个连续空间的起始(first)/结束(last)地址，以及第一个对象的位置(cur)；

​	finish迭代器记录着map数组中最后一个连续空间的的起始(first)/结束(last)地址，以及end的位置(cur)。

​	综上，可以在不同内存段中遍历，并快速定位首/尾节点。

### 2) 优缺点

​	deque支持**随机访问**O(1)：要根据访问偏移量的大小和内存段的大小，判断是否跳转到下个内存段。

​	deque支持高效的**头部和尾部**插入/删除操作O(1)。

​	在功能上，deque合并了vector和list，但占用更多的**内存**。

# 多线程

## 1 C++11中的几种锁

​	本小节参考自[C++11线程中的几种锁](https://blog.csdn.net/xy_cpp/article/details/81910513)。

### 1) 互斥锁Mutex

​	互斥锁是一个信号量，是一种**sleep-waiting的锁**，用于控制多个线程对共享资源互斥访问，避免多个线程在某一时刻同时操作一个共享资源。

​	在某一时刻，只有一个线程可以获取互斥锁，在释放互斥锁之前其他线程都不能获取该互斥锁。如果其他线程想要获取这个互斥锁，那么这个线程只能以**阻塞方式**进行等待。

​	当线程被阻塞，阻塞的线程被放入到**等待队列**中去，将当前核心的**时间片段**让出来，处理其他事务。

```c++
//用互斥元保护列表
#include <list>
#include <mutex>

std::list<int> some_list;
std::mutex some_mutex;

void add_to_list(int new_value)
{
    std::lock_guard<std::mutex> guard(some_mutex);
    some_list.push_back(new_value);
}
```

### 2) 条件锁

​	条件锁即条件变量，某一个线程因为某个条件未满足时可以使用条件变量使该程序处于阻塞状态。

​	一旦条件满足以“信号量”的方式唤醒一个因为该条件而被阻塞的线程。

```c++
//使用std::condition_variable等待数据
std::mutex mut;
std::queue<data_chunk> data_queue;
std::condition_variable data_cond;

void data_preparation_thread()
{
    while(more_data_to_prepare())
    {
        data_chunk const data = prepare_data();
        std::lock_guard<std::mutex> lk(mut);
        data_queue.push(data);
        data_cond.notify_one();
    }
}

void data_processing_thread()
{
    while(true)
    {        
        std::unique_lock<std::mutex> lk(mut);//这里使用unique_lock是为了后面方便解锁
        data_cond.wait(lk,{[]return !data_queue.empty();});
        data_chunk data = data_queue.front();
        data_queue.pop();
        lk.unlock();
        
        process(data);
        
        if(is_last_chunk(data))
            break;
    }
}
```

​	当来自数据线程中对notify_one()的调用通知条件变量时，线程从睡眠状态中苏醒（解除其阻塞），重新获得互斥元上的锁，并再次检查条件.

​	如果条件已经满足，就从wait()返回值，互斥元仍被锁定；如果条件不满足，该线程解锁互斥元，并恢复等待。

### 3) 自旋锁

​	本小节参考自：[使用C++ 11原子量实现自旋锁](https://www.cnblogs.com/FateTHarlaown/p/9170474.html)。

​	自旋锁是一种基础的同步原语，用于保障对共享数据的互斥访问。

​	与互斥锁的相比，在获取锁失败的时候**不会使线程阻塞、不会让出CPU、不会切换线程状态**，而是**一直自旋尝试获取锁**。

​	当线程等待自旋锁的时候，**CPU不能做其他事情**，而是**一直处于轮询忙等的状态**。

​	自旋锁主要适用于**被持有时间短**，线程**不希望在重新调度上花过多时间**的情况。如果在持锁时间很长的场景下使用自旋锁，则会导致CPU在这个线程的时间片用尽之前一直消耗在无意义的忙等上，造成计算资源的浪费。

#### 3.1) CAS思想

​	CAS(Compare and Swap)，实现并发算法时常用到的一种技术。这种操作提供了硬件级别的原子操作(通过锁总线的方式)。CAS操作的原型可以认为是：

```c++
//V: 内存中的变量, A: 期待值, B: 新值
//当V的值与A相等时，将V与B的值交换
bool CAS(V, A, B)
{
    if (V == A)
    {
        swap(V, B);
        return true;
    }
    return false;
}
```

​	上述CAS操作必须是**原子的**：要么不做，要么全部完成。

#### 3.2) CAS自旋锁设计思路

##### 3.2.1) 非原子操作

​	以bool变量来表示是否可以进入临界区，有如下代码：

```c++
// 若flag为true, 表示其他地方已进入临界区, 需等待
while(flag == true) {}
flag = true; // 进入临界区, flag设置为true
// do something ...
flag = false;// 退出临界区, flag设置为false
```

​	上述对临界区的判断**不是原子的**，有下述执行步骤：

| step | thread1                | thread2                |
| ---- | ---------------------- | ---------------------- |
| 1    | while(flag == true) {} |                        |
| 2    |                        | while(flag == true) {} |
| 3    | flag = true            |                        |
| 4    |                        | flag = true            |
| 5    | do something           | do something           |
| 6    | flag = false           |                        |
| 7    |                        | flag = false           |

​	可以看到，两个线程同时进入了临界区，不满足初衷。

##### 3.2.2) 原子操作

```c++
b = true;
while(!CAS(flag, false, b)) {}
//do something
flag = false;
```

​	判断操作与写入操作已经成为了一个整体，当一个线程的CAS操作成功的时候会阻止其他线程进入临界区，到达互斥访问的目的。

##### 3.2.3) 自旋锁实现

​	C++ 11提供了原子量，使用其来实现自旋锁，原子量提供了下述接口：

```c++
//CAS weak操作, 可能会失败, 但性能会更好
std::atomic::compare_exchange_weak( T& expected, T desired,
                                    std::memory_order order =
                                    std::memory_order_seq_cst ),
//CAS strong操作, 一定会成功                               
std::atomic::compare_exchange_strong( T& expected, T desired,
                                    std::memory_order order =
                                    std::memory_order_seq_cst )
//赋值
void store( T desired, std::memory_order order = std::memory_order_seq_cst )
```

​	自旋锁的设计如下：

```c++
#include <atomic>

class SpinLock 
{

public:
    SpinLock() : flag_(false) {}

    void lock()
    {
        bool expect = false;
        while (!flag_.compare_exchange_weak(expect, true))
        {       
            expect = false;//一定要将expect复原，执行失败时expect结果是未定的
        }
        //方法结束, lock()成功, 进入临界区, 此时flag_为true
    }

    void unlock()
    {
        flag_.store(false);
    }

private:
    std::atomic<bool> flag_;
};
```

​	自旋锁的使用：

```c++
SpinLock myLock;
myLock.lock();
//do something
myLock.unlock();
```

### 4) 读写锁

​	读写锁也叫做**共享-排他锁**。

​	当读写锁以**读模式**锁住时，它是以**共享模式**锁住的。当它以写模式锁住时，它是以**独占模式**锁住的。

#### 4.1) C++17 std::shared_mutex

本小节参考自[C++ 读写锁的用法](https://gukaifeng.cn/posts/c-du-xie-suo-de-yong-fa/index.html)。

​	顾名思义，shared_mutex是共享互斥锁，在使用上：

- 读锁(`共享锁`)使用std::shared_lock()操作互斥量。
- 写锁(`排他锁`)使用std::lock()操作互斥量，但通常用**std::lock_guard**或**std::unique_lock**包装器，因为其提供了RAII机制。
- 读线程持有的共享锁全部释放**之前**，写线程获取排他锁时会阻塞。
- 写线程释放排他锁**之前**，读线程获取共享锁时会被阻塞。

```c++
class TestSharedMutex {
    mutable std::shared_mutex _m;
    std::vector<int> _data;
public:
    TestSharedMutex() {}
    
    void read_data() const
    {
        std::shared_lock lk(_m);
        // read data......
    }
    
    void write_data(int value) 
    {
        std::lock_guard lk(_m);
        // write data......
    }
};
```

#### 4.2) C++14 读写锁包装器std::shared_lock

​	shared_lock是shared_mutex的包装器，需要用shared_mutex来构造shared_lock。

​	对shared_lock上锁，实际是以**共享模式**对关联的shared_mutex上锁。若要以**独占模式**对shared_mutex上锁，使用**unique_lock**包装器。

```c++
std::string file = "Original content."; // Simulates a file
std::mutex output_mutex; // mutex that protects output operations.
std::shared_mutex file_mutex; // reader/writer mutex

void read_content(int id)
{
    std::string content;
    {
        std::shared_lock lock(file_mutex, std::defer_lock); // Do not lock it first.
        lock.lock(); // Lock it here.
        content = file;
    }
    std::lock_guard lock(output_mutex);
    std::cout << "Contents read by reader #" << id << ": " << content << '\n';
}

void write_content()
{
    {
        std::lock_guard file_lock(file_mutex);// exclusive lock
        file = "New content";
    }
    std::lock_guard output_lock(output_mutex);
    std::cout << "New content saved.\n";
}
```

## 2 C++ 11的原子量和内存序

​	本小节参考自这里：[C++ 11的原子量和内存许浅析](https://www.cnblogs.com/FateTHarlaown/p/8919235.html)。

## 3 C++锁的包装

​	本小节参考自：[锁管理](https://juejin.cn/post/7069550372934647845)。

​	C++的锁管理和互斥量Mutex息息相关，其都使用**RAII**的风格进行锁管理。

​	RAII：Resource Acquisition is initialization，即资源获取即初始化。栈对象构造时进行初始化，栈对象**在对象生命周期结束时进行**自动销毁。如下示例：

```c++
std::mutex mut;
{
  std::lock_guard<std::mutex> lockGuard(mut);  // lock in lock_guard 构造函数
  sharedVariable++;
}  // unlock in lock_guard 析构函数
// -----------------------------------------
// ---------------不使用包装器----------------
std::mutex mut;
mut.lock();
sharedVariable++;
mut.unlock();
```

### 1) C++11 std::lock_guard

​	std::lock_guard是互斥量mutex的包装器，在其作用域中为互斥量提供RAII的风格机制，如下所示：

```c++
int g_i = 0;
std::mutex g_i_mutex;  // protects g_i
 
void safe_increment()
{
    const std::lock_guard<std::mutex> lock(g_i_mutex);
    ++g_i; 
    // g_i_mutex is automatically released when lock goes out of scope
}
```

​	优点：创建即加锁，作用域结束自动析构解锁，无需手工解锁。

​	缺点：① 不能中途解锁，必须等作用域结束才能解锁。② 如果定义域范围很大的话，**锁的粒度**就会很大，影响效率。

### 2) C++11 std::unique_lock

​	std::unique_lock也是通用互斥量mutex的包装器，也为互斥量提供RAII的机制，但它具备更多功能接口。lock_guard是unique_lock的缩减版。

​	它允许延迟锁定、锁定的有时限尝试、递归锁定、所有权转移和与条件变量一同使用，它可移动，不可复制。

```c++
//延迟锁定(构造时不锁定)
unique_lock( mutex_type& m, std::defer_lock_t t ) noexcept;

//有时限尝试获取锁, 阻塞直到时限到达或拿到锁的控制权
template< class Rep, class Period >
bool try_lock_for( const std::chrono::duration<Rep,Period>& timeout_duration );

//移动构造, 所有权转移
unique_lock( unique_lock&& other ) noexcept;

//向mutex那样调用lock
void lock();
```

​	std::unique_lock使用示例：

```c++
std::mutex mut,
{
  std::unique_lock<std::mutex> unilock(mut);// lock in constructor
  sharedVariable++;
  unilock.unlock();// 这里可以直接解锁，更灵活了
  
  // do other time consuming thing
  
  if (unilock.try_lock_for(1s)) 
  {
      // read share data
  }
}  
```

​	优点：可以通过lock()、unlock()决定锁的粒度。

​	缺点：① 析构时会判断当前的锁状态来决定是否解锁。② unique_lock是写锁，被锁定后，其他线程无法使用shared_lock或unique_lock相关代码。

### 3) std::lock_guard和std::unique_lock对比

- 相同点

​	lock_guard和unique_lock都是自释放锁。

- 不同点

​	`lock_guard`功能单一，只能用作自释放锁，但其在时间和空间上都比`unique_lock`更快；

​	`unique_lock`具备`lock_guard`的所有能力，同时提供更多的能力，且不会引入double lock / double unlock。

- 锁的粒度

​	锁头锁住的代码的多少称为**锁的粒度**，粒度一般用粗细来描述。

​	锁住的代码少，这个粒度叫细，执行效率高。 锁住的代码多，粒度叫粗，执行效率就低。

​	要学会尽量选择合适粒度的代码进行保护，力度太细，可能漏掉共享数据的保护；粒度太粗，影响效率。

- 不要混用unique_lock和mutex

​	只用了unique_lock，就不要在外部使用mutex本身。原因是unique_lock**内部会维护一个标识**记录自己管理的 锁的状态，如果直接使用mutex，unique_lock无法更新内部mutex的状态。

### 4) C++11 std::lock防止死锁

```c++
template< class Lockable1, class Lockable2, class... LockableN >
void lock( Lockable1& lock1, Lockable2& lock2, LockableN&... lockn );
```

​	如上所述，std::lock()可传入多个可加锁对象，其内部使用了一套**避免死锁的算法**，来给各个锁对象加锁。

​	当其中存在**任意对象**不能获得其锁时，std::lock会阻塞。

​	下述示例可能出现死锁：

```c++
std::mutex mt1, mt2;
// thread 1
void fun1()
{
    std::lock_guard<std::mutex> lck1(mt1);//先lock mt1
    std::lock_guard<std::mutex> lck2(mt2);
    // do something
}
// thread 2
void fun2()
{
    std::lock_guard<std::mutex> lck2(mt2);//先lock mt2
    std::lock_guard<std::mutex> lck1(mt1);
    // do something
}
 
int main ()
{
    // 两个线程的互斥量锁定顺序不同，可能造成死锁
    std::thread t1(func1);
    std::thread t2(func2);
 
    t1.join();
    t2.join();
 
    return 0;
}
```

​	使用std::lock避免死锁。注意，传入std::lock的是**锁包装器**，因此，无需手动unlock。

```c++
std::mutex mt1, mt2;
// thread 1
void fun1()
{
    std::unique_lock<std::mutex> lck1(mt1, std::defer_lock);
    std::unique_lock<std::mutex> lck2(mt2, std::defer_lock);
    std::lock(lck1, lck2);// lck1和lck2顺序可以任意
    // do something
}
// thread 2
void fun2()
{
    std::unique_lock<std::mutex> lck1(mt1, std::defer_lock);
    std::unique_lock<std::mutex> lck2(mt2, std::defer_lock);
    std::lock(lck2, lck1);// lck1和lck2顺序可以任意
    // do something
}
```

### 5) C++14 std::shared_lock

​	std::shared_lock内容，可参考小节 <a href="#4.2) C++14 读写锁包装器std::shared_lock">4.2) C++14 读写锁包装器std::shared_lock</a>。

### 6) C++17 std::scope_lock

​	scope_lock是以RAII形式的互斥量包装器，它可以包装一个或多个互斥量。

​	如果scope_lock被赋予了多个互斥量，其内部会使用**std::lock来避免死锁**。当scope_lock离开作用域，它会依次解锁。

​	上锁流程：scoped_lock可以接受任意数量的mutex，并将这些mutex传给std::lock来同时上锁。它会对其中一个mutex调用 lock()，对其他调用 try_lock()，若 try_lock() 返回 false 则对已经上锁的 mutex 调用 unlock()，然后重新进行**下一轮上锁**。标准未规定下一轮的上锁顺序，可能不一致，重复此过程直到所有 mutex 上锁，从而达到同时上锁的效果。

## 4 死锁

### 1) 死锁是什么

​	如果你将某个`mutex`上锁了，却一直不释放，另一个线程访问该锁保护的资源的时候，就会发生死锁：

```c++
Thread A              Thread B
_mu.lock()          _mu2.lock()
   // 死锁               // 死锁
_mu2.lock()         _mu.lock()
```

​	上述两个线程访问锁的顺序不同，就会导致死锁。

### 2) 避免死锁的建议

#### 2.1) 使用层次锁(即给锁定义上锁顺序)

​	将互斥锁包装一下，给锁定义一个层次的属性，每次按层次由高到低的顺序上锁。

​	下述按地址大小顺序上锁，也是一种变向的层次锁：

```c++
if(&_mu < &_mu2)
{
    _mu.lock();
    _mu2.unlock();
}
else
{
    _mu2.lock();
    _mu.lock();
}
```

#### 2.2) 使用std::lock避免死锁

```c++
std::mutex m1, m2;
std::unique_lock<std::mutex> lk1(m1, std::defer_lock);
std::unique_lock<std::mutex> lk2(m2, std::defer_lock);
std::lock(lk1, lk2);
```

#### 2.3) 尽量同时只对一个互斥锁上锁

```c++
{
    std::lock_guard<std::mutex> guard(_mu);
    //do something
}

{
    std::lock_guard<std::mutex> guard2(_mu2);
    //do something
}
```

#### 2.4) 不要在锁保护区域使用用户自定义代码

```c++
{
    std::lock_guard<std::mutex> guard(_mu2);
    user_function(); // 避免调用user_function, 因为其内部可能操作了其他互斥锁, 导致锁循环
}
```

# C++中的设计模式

​	参考[知乎：C++ 有哪些常用又简单的设计模式？](https://www.zhihu.com/question/299975615)。

# C++进程间通信

​	参考[C++进程间通信](https://blog.yanjingang.com/?p=4503)。

## 1 进程间通信目的

- 数据传输

- 多进程间共享资源

- 事件通知

- 进程控制：有些进程希望完全控制另一个进程的执行(如Debug进程)，希望能够拦截另一个进程的所有操作，并能够及时知道它的状态改变。

## 2 进程间通信原理

​	每个进程各自有**不同的用户地址空间**，任何一个进程的全局变量在另一个进程中都看不到，所以进程之间要交换数据**必须通过内核**。

​	**在内核中开辟一块缓冲区**，进程1把数据从用户空间拷到内核缓冲区，进程2再从内核缓冲区把数据读走，内核提供的这种机制称为**进程间通信机制**。

<img src=".\pic\c++_ipc.png" alt="c++_ipc" style="zoom:60%;" />

## 3 进程间通信方式

### 1) 匿名管道(Pipe)

​	管道可用于具有**亲缘关系**的进程间通信：允许一个进程和另一个与它有**共同祖先**的进程进行通信。

```c++
#include <unistd.h>

// linux中创建匿名管道
int _pipe[2] = {0, 0};
int ret = pipe(_pipe);  // _pipe[0]管道读端, _pipe[1]管道写端, ret: 0成功, -1失败
```

​	向两个文件描述符读写数据，实际是在内核的内存缓冲区读写。匿名管道实际是**存在于内存中的文件**。

​	匿名管道由**环形队列**组织，一个进程输入信息，一个进程读出信息。

#### 1.1) 管道通信流程

​	父进程开辟管道，得到管道的读端/写端

<img src=".\pic\c++_pipe-1.png" alt="c++_pipe-1" style="zoom:50%;" align="left"/>

​	父进程调用fork创建子进程，子进程也有两个文件描述符指向同一管道：

<img src=".\pic\c++_pipe-2.png" alt="c++_pipe-2" style="zoom:50%;" align="left"/>

​	父进程关闭读端，子进程关闭写端。父进程可以往管道里写，子进程可以从管道里读。

​	管道是用**环形队列**实现的，**数据从写端流入**，**从读端流出**，就实现了进程间通信。

<img src=".\pic\c++_pipe-3.png" alt="c++_pipe-3" style="zoom:50%;" align="left"/>

#### 1.2) 管道的四种特殊情况

- 读端不关闭，写端关闭

​	管道中剩余的数据都被读取后，再次read会**返回0**，就像读到文件末尾一样。

- 读端关闭，写端不关闭

​	此时该进程会收到信号SIGPIPE，通常会导致进程异常终止。

- 读端不关闭，写端不关闭，但不写数据，

​	管道中剩余的数据都被读取后，再次read会**被阻塞**，直到管道中有数据可读了才重新读取数据并返回。

- 读端不关闭，但不读数据，写端不关闭

​	当写端被写满之后，再次write会阻塞，直到管道中有空位置了才会写入数据并重新返回。

#### 1.3) 管道的缺点

1. 两个进程通过一个管道只能实现**单向通信**。如果想双向通信必须再重新创建一个管道或者使用sockpair。
2. 只能用于具有**亲缘关系**的进程间通信，例如父子，兄弟进程。

### 2) 命名管道(fifo)

​	命名管道实质是一个**设备文件**，它解决了匿名管道只有亲缘进程通信的局限性。只要可以**访问**设备文件的**路径**，就能够通过FIFO相互通信。

​	匿名管道存在于内存中，而命名管道是存在于**磁盘上的文件**。

```c++
/* 
 *	返回值: 成功返回0，失败返回-1
 *	path: 命名管道的全路径
 *  mode：创建模式，指明存取权限
 *  dev: 设备值，取决于文件创建的种类，只有在创建设备文件时，才会用到该值
*/

#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
int mknod(const char *pathname, mode_t mode, dev_t dev);

#include <sys/types.h>
#include <sys/stat.h>
int mkfifo(const char *pathname, mode_t mode);
```

### 3) 共享内存

​	本小节参考自[C++进程间通信之共享内存](https://blog.csdn.net/cylddrmm123/article/details/134096333)。

​	共享内存是一种特殊的内存区域，是在多个进程之间**共享数据的机制**。它允许不同的进程在同一块内存空间中访问和操作共享的数据，而**无需进行数据拷贝或传输**，因此其是效率最高的进程通信方式。

​	如下图所示，两个进程将**同一块物理内存**映射到各自的地址空间中：

<img src=".\pic\shared memory.png" alt="shared memory" style="zoom:100%;" />

#### 3.1) 共享内存的特性

1. 将**物理内存**映射到多个进程的虚拟地址空间中；
2. 通过操作系统提供的**系统调用和机制**，进程可以创建共享内存区域并映射到自己的虚拟地址空间中；
3. 直接访问共享内存中的数据进行读写操作，通过适当的**同步机制**保证数据的正确和一致。

#### 3.2) 共享内存读写简单示例

​	下述只是简单的示例，没有考虑同步问题。

​	进程A写入共享内存：

```c++
#include <iostream>
#include <sys/ipc.h>
#include <sys/shm.h>
 
int main() {
    // 创建共享内存的key
    key_t key = ftok("shared_mem_example", 1234);
 
    // 创建共享内存区域(内核空间)
    int shmid = shmget(key, sizeof(int), IPC_CREAT | 0666);
    if (shmid == -1) 
    {
        std::cerr << "Failed to create shared memory segment." << std::endl;
        return 1;
    }
 
    // 将共享内存映射到进程的地址空间
    int* shared_data = (int*)shmat(shmid, NULL, 0);
    if (shared_data == (int*)-1) 
    {
        std::cerr << "Failed to attach shared memory segment." << std::endl;
        return 1;
    }
 
    // 写入数据到共享内存
    *shared_data = 42;
 
    // 用户空间断开与共享内存的连接
    if (shmdt(shared_data) == -1) 
    {
        std::cerr << "Failed to detach shared memory segment." << std::endl;
        return 1;
    }
 
    // 将内核空间的内存释放
    if (shmctl(shmid, IPC_RMID, NULL) == -1) 
    {
        std::cerr << "Failed to delete shared memory segment." << std::endl;
        return 1;
    }
    return 0;
}
```

​	进程B读共享内存：

```c++
#include <iostream>
#include <sys/ipc.h>
#include <sys/shm.h>
 
int main() {
    // 获取共享内存的key
    key_t key = ftok("shared_mem_example", 1234);
 
    // 获取共享内存区域(内核空间)
    int shmid = shmget(key, sizeof(int), 0666);
    if (shmid == -1) 
    {
        std::cerr << "Failed to get shared memory segment." << std::endl;
        return 1;
    }
 
    // 将共享内存映射到进程的地址空间
    int* shared_data = (int*)shmat(shmid, NULL, 0);
    if (shared_data == (int*)-1) 
    {
        std::cerr << "Failed to attach shared memory segment." << std::endl;
        return 1;
    }
 
    // 从共享内存读取数据
    int data = *shared_data;
    std::cout << "Data from shared memory: " << data << std::endl;
 
    // 用户空间断开与共享内存的连接
    if (shmdt(shared_data) == -1) 
    {
        std::cerr << "Failed to detach shared memory segment." << std::endl;
        return 1;
    }
 
    return 0;
}
```







​	





​	

