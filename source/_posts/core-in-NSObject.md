---
title: NSObject 对象本质
date: 2018-04-19 19:03:18
tags:
---
我们都知道，Objective-C 中的对象、类都是基于 C/C++ 中的结构体实现的。而在 Apple 的接口文档中，可以发现 NSObject 的定义如下：
<!-- more -->

```objc
@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}
@end
```

既然已知 NSObject 是基于 C/C++ 的结构体实现的，我们直接将 NSObject 的定义改为结构体的定义，也可以通过

```sh
xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc [FILE] -o [OUTPUT]
```

命令将 Objective-C 的代码转换成 arm64 架构下的 C++ 代码，可以发现 NSObject 的定义如下：

```c
struct NSObject_IMPL {
  Class isa;
}
```

而这个 Class 又是什么，从转换之后的 C++ 代码中可以看出，关于 Class 的定义如下：

```c
typedef struct objc_class *Class;
```

也就是说这个 NSObject 结构体中其实只存在一个成员变量，这个成员变量是一个 isa 的指针。那么这里的 Class 是什么？我们直接下载苹果开源的 objc 的代码来看，会发现是如下结构：

```c
struct objc_class : objc_object {
	// Class ISA;
	Class superclass;
	cache_t cache;             // formerly cache pointer and vtable
	class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

	class_rw_t *data() { 
		return bits.data();
	}
	//...
}

struct class_rw_t {
	// Be warned that Symbolication knows the layout of this structure.
	uint32_t flags;
	uint32_t version;

	const class_ro_t *ro;

	method_array_t methods;
	property_array_t properties;
	protocol_array_t protocols;

	Class firstSubclass;
	Class nextSiblingClass;

	char *demangledName;
	//...
}
```

从上面的结构体信息可以看出来，objc\_class 结构体，或者说 Class 对象包含了 isa 指针，superclass 指针，类的属性、对象方法（非类方法）信息，协议信息、成员变量信息等信息（objc\_class 中的 isa 指针被注释掉了，而存在于 objc\_object 中）。

### **isa 指针**

在 Objective-C 中，存在三种对象，instance 对象，Class 对象和 meta-class 对象。一个 instance 对象在内存中存储的信息包括 isa 指针和其他的成员变量，这个 isa 指针指向的是这个 instance 所对应的类的 Class 对象，Class 对象则又包含 isa 指针，superclass 指针，类的属性、对象方法（非类方法）信息，协议信息、成员变量信息等，然后 Class 对象的 isa 指针又指向 Class 对象所对应的 meta-class 对象，meta-class 对象的结构跟 Class 对象类似，同样包含 isa 指针和 superclass 指针，还保存着类的类方法等信息。

这样解释有点晕，加入我们有这样一段代码：

```objc
@interface Person: NSObject {
  int _age;
}

- (void)instanceMethod;
+ (void)classMethod;

@end
```

按照之前的逻辑，如果我们将 Person 进行实例化则得到了一个 instance 对象，

* 该对象存储了 age 信息和 isa 指针，这个 isa 指针指向了 Person 这个 Class 对象；
* Person 的 Class 对象里又包含了 isa 指针、superclass 指针、- (void)instanceMethod 这个方法的相关信息等。其中 superclass 指针指向 NSObject 对象，而 isa 指针则指向 Person 所对应的 meta-class 对象；
* meta-class 对象包含 isa 指针，superclass 指针和 + (void)classMethod 的方法信息。meta-class 的 isa 指针指向该 meta-class 所继承的 root class 也就是 NSObject，superclass 则指向 NSObject 的 meta-class.
* root class，在这里就是 NSObject，NSObject 的 Class 对象的 superclass 指针指向 nil，而 NSObject 的 meta-class 对象的 superclass 指向 NSObject 的 Class 对象，NSObject 的 meta-class 对象的 isa 指针指向了自己。

希望别被绕晕了，上面的逻辑可以通过下面经典的示意图来表示出来：

![1716191-18a209b6024389d8.png](http://7xjbza.com1.z0.glb.clouddn.com/class&metaclass.png)

结合图片以及上面的绕口令似的的解释，应该能够理解 instance 对象、Class 对象和 meta-class 对象三者间的关系以及 isa 指针的作用。我们可以通过 runtime 提供的一系列方法可以获取相应的对象：

```objc
// 创建 NSObject 实例对象
NSObject *obj = [NSObject new];
// 获取 NSObject 的 Class 对象
object_getClass(obj);
// 获取 NSObject Class 对象的 meta class 对象
object_getClass([NSObject class]) ;
// 判断是否是 meta-class
class_isMetaClass();
```

所以按照上面的结论，一个 Person 对象的实例所占用的内存大小应该为 NSObject 对象的内存大小加上 int 类型的内存大小，8 + 4 = 12，但真的是这样吗，我们使用

```objc
class_getInstanceSize([Person class])
```

打印输出发现是 16，所以并不是简单的累加，这里就又牵涉到内存对齐的知识了。

### 内存对齐

对于现代处理器来说，C 编译器在内存中放置基本数据类型的方式是收到约束的，通过这种约束对内存进行自动的对齐，从而达到加快内存访问速度的目的。按照我自己的理解，达到内存对齐分为两部，第一步是实现内部数据的填充，规则是**结构体内前一个成员变量的必须是后一个成员变量地址的正整数倍，如果不满足则要在前一个成员变量之后补齐**；第二部是进行整体对齐，即**整个结构体的长度必须是其最宽成员长度的整数倍**。

比如以下结构体：

```c
struct foo {
  char *p;
  char c;
  short x;
}
```

假设现在处于 64 位系统中，按照之前第一步的做法，char \*p 是一个指针，占用 8 个字节，char c 占用 1 个字节，那么正好，p 的大小是后面 c 的整数倍。继续，x 占用 2 个字节，这样的话，c 所占用的 1 不是 x 所占用 2 的整数倍，需要在 c 之后补齐 1 个字节。然后进入到第二步，通过第一步得出的结果，目前是 ：

```c
struct foo {
  char *p; // 8 bytes
  char c; // 1 byte
  char padding[1] // 1 byte
  short x; // 2 bytes
}
```

可以发现，8+1+1+2=12 并不是 8 的整数倍，所以需要进行尾填充，填充 4 个字节，整个 foo 的结构体所占用的字节数应该是 16 个字节。

现在看一下复杂情况，如果一个结构体中含有另一个结构体成员，那么要求**内层结构体需要和最宽的成员变量保持相同的对齐宽度，其实也就是满足之前的第二步**。比如下面的结构体：

```c
struct anotherFoo {
  char c;
  struct innerFoo {
    char *p;
    short x;
  } inner;
}
```

在 64 位系统下，c 占 1 个字节，然后往 inner 中看，\*p 是一个指针 8 个字节，所以在 c 后填充 7 位，x 占 2 个字节，inner 要满足同内部最宽成员保持一致，所以补 6 位，其内存分布如下所示：

```c
struct anotherFoo {
  char c; // 1 byte
  char padding[7]; // 7 bytes
  struct innerFoo {
    char *p; // 8 bytes
    short x; // 2 bytes
    char padding2[6]; // 6 bytes
  }
}
```

可以看出，这样的结构体占 24 个字节，其中有 13 个字节都是填充的字节，很浪费。

未完待续...