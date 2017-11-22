---
title: runtime 笔记一：使用 runtime 创建一个类
date: 2016-08-17 09:19:20
tags:
---
从做 iOS 开发后，一直都有学过 runtime 相关的知识，也在工作时某些情况下用过，但是从来都没有认真学一下相关的具体知识。借着最近想看一下 JSPatch 的具体实现原理，就又把 runtime 给拿出来看了一遍，具体的相关知识感觉没必要说了（网上相关的东西感觉都被说烂了），我们直接从用 runtime 来动手实现一个类以及为这个类添加变量和方法。
<!-- more -->

用 Objective-C 来写一个类相信没有人不会，代码如下，就不具体解释了：

```Objective-C
@interface People : NSObject

@property (strong, nonatomic) NSString *name;
@property (strong, nonatomic) NSNumber *age;

@end

@implementation People

- (void)introduce {
    NSLog(@"My name is %@, and I'm %@ years old", self.name, self.age);
}

@end
```

这里我们创建了一个 Person 类，有两个 <code>property</code>，以及一个叫 <code>introduce</code> 的方法。接下来我们用 runtime 来动态生成这样的类。

接下来，用 <code>objc\_allocateClassPair</code> 来创建一个类，并使用 <code> class\_addIvar</code> 为该类添加两个 ivar, 用 <code>objc\_registerClassPair</code> 注册该类：

```Objective-C
Class People = objc_allocateClassPair([NSObject class], "People", 0);
    
class_addIvar(People, "_name", sizeof(NSString *), 0, @encode(NSString *));
class_addIvar(People, "_age", sizeof(NSNumber *), 0, @encode(NSNumber *));

objc_registerClassPair(People);
```

上述代码中的 <code>@encoding(NSString *)</code> 方法，[官方文档](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)里面有详细的解释，简单来说就是将一个给定类型的编码转换成 runtime 内部表示的字符串，比如 <code>NSString \*</code> 类型转换结果就是 @ , 具体的介绍看文档。

好，到现在，我们有了一个拥有 name 和 age 两个实例属性的 People 类，然后我们要给它添加一个方法，我们先把这个方法写出来：

```Objective-C
void introduce(id self, SEL _cmd) {
    Ivar nameIvar = class_getInstanceVariable([self class], "_name");
    Ivar ageIvar = class_getInstanceVariable([self class], "_age");
    NSLog(@"name is %@, age is %@", object_getIvar(self, nameIvar), object_getIvar(self, ageIvar));
}
```

你会发现这个方法里面有两个参数，这两个参数分别代表的是消息接受对象和方法的 <code>selector</code> 。我们都知道 Objective-C 的方法调用实际上最终是会走到 <code>objc\_msgSend</code>，所以不算是调用方法，而是在发送消息，而在 <code>objc\_msgSend</code> 中是没有显示声明上面这两个参数的，而是会在编译期间插入代码，所以就算没有显示声明，也是可以调用的。

有了方法后，我们使用 <code>class_addMethod</code> 方法来添加 <code>introduce</code>：

```Objective-C
class_addMethod(People, method, (IMP)introduce, "v@:");
```

这个代码中又会看到 "v@:@" 这样奇怪的字符，这个其实和上面的 <code>@encoding</code> 是类似的，一个叫 Type Encoding，这个叫 Method Encoding，这里的 v 代表返回值 <code>void</code>，@ 代表参数 <code>id</code>, : 代表参数 <void>SEL</void>。关于这个具体的信息，[文档](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)里面都有解释。

好了，到现在我们有了一个 People 类，为它添加了两个 ivar 以及一个 method，我们通过赋值和调用来验证一下：

```Objective-C
id people = [[People alloc] init];

[people setValue:@"Lily" forKey:@"_name"];
[people setValue:@12 forKey:@"_age"];

((void (*)(id, SEL))objc_msgSend)(people, method);

```
运行一下，会发现结果符合我们预期。

到这里并没有结束，我们发现之前我们没有用 runtime 来生成的类是使用了 <code>property</code>，而我们用 runtime 来实现的代码并没有添加 property，所以我们现在来继续做这个。在 Objective-C 中，使用 <code>property</code> （非 @dynamic 情况）时会自动生成 getter、setter 方法，所以我们这里给 name 和 age 添加一下 getter 和 setter 方法，并将它们添加到 People 中去：

```Objective-C
NSString *nameGetter(id self, SEL _cmd) {
    Ivar nameIvar = class_getInstanceVariable([self class], "_name");
    return object_getIvar(self, nameIvar);
}

void nameSetter(id self, SEL _cmd, NSString *newName) {
    Ivar nameIvar = class_getInstanceVariable([self class], "_name");
    id old = object_getIvar(self, nameIvar);
    if (old != newName) {
        object_setIvar(self, nameIvar, [newName copy]);
    }
}

NSNumber *ageGetter(id self, SEL _cmd) {
    Ivar ageIvar = class_getInstanceVariable([self class], "_age");
    return object_getIvar(self, ageIvar);
}

void ageSetter(id self, SEL _cmd, NSNumber *newAge) {
    Ivar ageIvar = class_getInstanceVariable([self class], "_age");
    id old = object_getIvar(self, ageIvar);
    if (old != newAge) {
        object_setIvar(self, ageIvar, newAge);
    }
}

// main.m
class_addMethod(People, sel_registerName("name"), (IMP)nameGetter, "@@:");
class_addMethod(People, sel_registerName("setName:"), (IMP)nameSetter, "v@:@");
class_addMethod(People, sel_registerName("age"), (IMP)ageGetter, "@@:");
class_addMethod(People, sel_registerName("setAge:"), (IMP)ageSetter, "v@:@");
```

接下来我们使用 <code>class\_addProperty</code> 方法来添加 <code>property</code>，这个方法接受一个 attributes 的参数，所以我们需要提前使用 <code>objc\_property\_attribute\_t</code> 方法生成 attributes：

```Objective-C
objc_property_attribute_t nameType = {"T", "@\"NSString\""};
objc_property_attribute_t nameNona = {"N", ""};
objc_property_attribute_t nameOwnership = {"C", ""};
objc_property_attribute_t nameBackingIvar = {"V", "_name"};
objc_property_attribute_t nameAttr[] = {nameType, nameNona, nameOwnership, nameBackingIvar};
class_addProperty(People, "name", nameAttr, 4);
    
objc_property_attribute_t ageType = {"T", "@\"NSNumber\""};
objc_property_attribute_t ageNona = {"N", ""};
objc_property_attribute_t ageOwnership = {"&", ""};
objc_property_attribute_t ageBackingIvar = {"V", "_age"};
objc_property_attribute_t ageAttr[] = {ageType, ageNona, ageOwnership, ageBackingIvar};
class_addProperty(People, "age", ageAttr, 4);
```

上述代码不难理解，只是关于 T、N、C、V 这些可能有点糊涂，其实知道这些字符代表的意思就很容易懂了：

|Key		 |   Stand for   |  Value |
|----------|:-------------:|-------:|
| N 		 |    nonatomic  |        |
| T 		 |      type     |  NSString/NSNumber |
| C 		 |      copy     |        |
| V 		 |      ivar     |  ivar name  |

这些字符的具体含义其实[文档](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtPropertyIntrospection.html)里面也有写，看一下就好。

刚刚我们把  <code>property</code> 给添加进去了，接下来我们可以使用 <code>class\_copyPropertyList</code> 来看看我们上面的代码到底有没有起作用：

```Objective-C
unsigned int propertiesCount;
objc_property_t *properties = class_copyPropertyList(People, &propertiesCount);
for (unsigned i = 0; i < propertiesCount; i++) {
    objc_property_t property = properties[i];
    NSString *propertyName = [NSString stringWithUTF8String:property_getName(property)];
    NSLog(@"%@", propertyName);
    NSLog(@"%s", property_getAttributes(property));
}
free(properties);
```

运行一下，打印出来的结果也都符合预期，证明 <code>property</code> 添加成功了，接下来我们去调用一把试一下：

```Objective-C
[people performSelector:sel_registerName("setName:") withObject:@"Lily"];
[people performSelector:sel_registerName("setAge:") withObject:@14];
NSLog(@"name is: %@, age is %@", [people performSelector:sel_registerName("name")], [people performSelector:sel_registerName("age")]);
```

这里用 <code>performSelector</code> 方法是因为在 ARC 下，直接使用 <code>setName</code> 方法编译期间会说找不到。好，打印结果为

```Console
name is: Lily, age is 14
```

好了，至此我们就用 runtime 生成了一个类了。后续呢，我们会再在此基础上直接实现一个动态添加方法。