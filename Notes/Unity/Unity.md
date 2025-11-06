---
typora-root-url: pic
---

# C#基础

## 委托Delegate

在现代C#编程中，委托Delegate是一种类型安全的**函数指针**，**能够引用与其签名匹配的方法**，并在运行时动态调用。

```c#
// 语法
[访问修饰符] delegate [返回类型] 委托名([参数列表]);

// 示例
public delegate void Notify(string message);
```

### 1) 本质

- 委托是一个类类型

  当使用`delegate`关键字定义一个委托类型时，编译器会自动生成一个继承自`System.MulticastDelegate`的**密封类**(sealed)。

- 封装调用目标

  每个委托实例封装了**两个核心信息**：方法指针（`_methodPtr`）和目标对象（`_target`）；

  对于实例方法，`_target`指向方法所属的对象；对于静态方法，`_target`为 `null`。

- 支持多播

  委托继承自`MulticastDelegate`，内部通过`_invocationList`字段维护一个委托链表，使一个委托实例可以按顺序调用多个方法。这是通过 `Delegate.Combine`和 `Remove`方法实现的，为事件模型提供了底层支撑。

- 类型安全

  委托是类型安全的函数指针，它在编译时就会检查方法的签名是否与委托声明匹配。

### 2) 使用

- 通过`+=`添加的方法，再不需要使用后，要通过`-=`移除，避免内存泄漏；
- 使用时会new一个委托实例，分配在Heap上。因此delegate若使用不当，可能会造成GC压力或内存泄漏。
- 使用event关键字，确保事件的安全性。

### 3) Action和Func

