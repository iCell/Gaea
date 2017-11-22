---
title: JavaScriptCore 的使用
date: 2017-09-18 14:22:43
tags:
---
在 iOS 7 之前，想进行 Objective-C 代码和 JavaScript 代码进行通信，普遍做法是使用 <code>UIWebView</code>，通过 <code>stringByEvaluatingJavaScriptFromString:</code> 来实现 OC 调用 JS 代码，在使用 webview 来加载一个自定义的 url 让本地进行拦截实现 JS 调用 OC 代码。而从 iOS 7 之后 <code>JavaScriptCore</code> 的推出，native 和 JS 代码可以直接利用 <code>JavaScriptCore</code> 来进行相互通信而不需要依赖于 <code>UIWebView</code>。比较知名的 [React Native](https://facebook.github.io/react-native/) 和 [JSPatch](https://jspatch.com/)，就是极大地发挥了 <code>JavaScriptCore</code> 的作用。
<!-- more -->
### JSContext / JSValue / JSVirtualMachine

<code>JSContext</code> 代表 JavaScript 的运行环境，负责执行 JS 代码，可以将它理解为 web 开发中的 window 对象。JS 代码执行后的返回结果就是 <code>JSValue</code>，它可以代表几乎任何 js 类型，比如 String／NSNumber／Array／Dictionary／function 等类型，包括 JS 中的 <code>undefined</code> 类型和 <code>null</code> 类型，且都有对应的方法将 js 类型的对象转换为 native 对应的类型的对象。

```Swift
// 1
let context = JSContext()!
// 2
context.evaluateScript("var num = 5 + 5")
context.evaluateScript("var names = ['Grace', 'Ada', 'Margaret']")
context.evaluateScript("var triple = function(value) { return value * 3 }")
// 3
let tripleNum = context.evaluateScript("triple(num)")
tripleNum?.toInt32()
tripleNum?.toString()
```

1. 创建一个 <code>JSContext</code> 对象
2. 使用 <code>evaluateScript</code> 方法执行 js 代码
3. 执行完 js 代码之后的结果对应为 native 代码中的 <code>JSValue</code> 对象，然后可以使用 <code>JSValue</code> 的 toInt32 或者 toString 方法转换为相应的 native 对象

<code>JSVirtualMachine</code> 代表 js 运行的虚拟机，它有自己的堆结构和垃圾回收机制，所以不能在不同的 <code>JSVirtualMachine</code> 中进行对象传递。一般一个 <code>JSVirtualMachine</code> 内只能有一个线程在工作，而要处理多线程问题的时候就需要使用多个 <code>JSVirtualMachine</code>.

<code>JSContext</code>、<code>JSValue</code>、<code>JSVirtualMachine</code> 三者的关系可以用下图来说明：

![](https://koenig-media.raywenderlich.com/uploads/2016/02/javascriptcore.png)

每个 <code>JSContext</code> 都是在一个 <code>JSVirtualMachine</code> 中执行的，一个 <code>JSVirtualMachine</code> 中可以运行多个 <code>JSContext</code>，每个 <code>JSValue</code> 都是绑定在一个 <code>JSContext</code> 中的，不同的 <code>JSContext</code> 之间可以相互进行 <code>JSValue</code> 的传递，而不同的 <code>JSVirtualMachine</code> 是不能进行对象的相互传递的。

### native 代码调用 JS 代码

##### 下标使用

在前面的代码中，我们用 js 代码创建了一个数组 names，我们可以用 <code>objectForKeyedSubscript</code> 方法获取 js 代码中创建的对象，另外，<code>JavaScriptCore</code> 也提供了 <code>objectAtIndexedSubscript</code> 方法直接进行 js 数组的使用：

```Swift
let names = context.objectForKeyedSubscript("names")
let name = names?.objectAtIndexedSubscript(1)
```

上述两个方法返回同样是 <code>JSValue</code> 对象。

##### 方法调用

上面我们说到可以使用 <code>objectForKeyedSubscript</code> 方法来获取 js 代码创建的对象，但如果我们获取的是 js 代码创建的 function 对象，<code>JavaScriptCore</code> 也直接提供了 <code>callWithArguments</code> 方法进行 js function 的调用。<code>callWithArguments</code> 接受一个数组类型的参数，作为传递给 js function 的参数。

```Swift
let tripleFunction = context.objectForKeyedSubscript("triple")
let result = tripleFunction?.call(withArguments: [5])
```
##### 错误处理

通过 <code>JSContext</code> 执行的 js 代码，如果出现了类似于类型错误、语法错误、运行时错误等，可以通过 <code>JSContext</code> 中提供的 <code>exceptionHandler</code> 属性来进行异常的捕获：

```Swift
let context = JSContext()!
context.exceptionHandler = { context, exception in
    print(exception?.description ?? "unknown error")
}
```
###JS 代码调用 native 代码

##### Blocks

当使用 Objective-C 创建一个 block，然后将其赋值给 <code>JSContext</code> 之后，<code>JavaScriptCore</code> 会自动将改 block 转换为 js 代码中的 function 对象。这里注意，<code>JavaScriptCore</code> 只能识别 Objective-C 中的 block，所以当我们使用 Swift 代码时，需要使用 @convention 将 Swift 中的闭包转成 Objective-C 的 block:

```Swift
let simplifyString: @convention(block) (String) -> String = { input in
    return input + "test"
}
context.setObject(simplifyString, forKeyedSubscript: "simplifyString" as NSString)
print(context.evaluateScript("simplifyString('test')"))
```

#### 内存管理

既然使用到了 block，那么就会可能遇到循环引用的问题，所以在 block 中如果需要使用到 <code>JSContext</code>，可以使用 <code>JSContext.current()</code> 来获取当前 <code>JSContext</code> 的上下文。但是呢，有时候还是有无法解决循环引用的情况，比如有一个类，它的实例变量是一个 <code>JSValue</code>，而我们想通过 native 代码来实例化这个类提供给 js 代码使用：

```Swift
class CustomView: UIView {
    
    var context: JSContext?
    var handler: JSValue?
    
    init(handler: JSValue?, context: JSContext?) {
        super.init(frame: CGRect.zero)
        self.context = context
        self.handler = handler
    }
    
    required init?(coder aDecoder: NSCoder) {
        fatalError("")
    }
}

let demo: @convention(block) (JSValue?) -> Void = { handler in
    let context = JSContext.current()
    let customObject = CustomView(handler: handler, context: context)
}
context.setObject(demo, forKeyedSubscript: "customObject" as NSString)
```
分析一下上面的代码，demo 这个 block 持有 <code>CustomView</code> 这个对象，<code>CustomView</code> 持有 <code>JSValue</code>，而 <code>JSValue</code> 会绑定一个 <code>JSContext</code>，所以一个循环引用就产生了。<code>JavaScriptCore</code> 提供了 <code>JSManagedValue</code> 这个类用于应对这种循环引用的情况，只需要将上述 <code>CustomView</code> 中的 <code>JSValue</code> 通过 <code>managedValueWithValue</code> 方法转换成 <code>JSManagedValue</code> 之后，使用 <code>JSVirtualMachine</code> 中提供的方法 <code>addManagedReference(_ object: Any!, withOwner owner: Any!)</code> 将 <code>JSManagedValue</code> 添加到 virtualMachine 中，保证在运行期间正常 native 和 js 正常访问对象，当不需要继续使用时，调用 <code>removeManagedReference(_ object: Any!, withOwner owner: Any!)</code> 方法将其销毁。


##### JSExport

如果我们想要在 js 代码中调用我们用 native 代码创建的自定义对象，只需要让其遵循 JSExport 协议，则会自动将协议中声明的属性、方法等提供给 js 代码来进行使用。如果我们是想在 js 代码中调用已经定义好的系统类或者某个第三方库的类，我们不太好去改这种类的源码，可以使用 runtime 中的 <code>class_addProtocol</code> 方法为该类动态添加该协议。

比如我们写一个 <code>Person</code> 类，然后让 js 代码中来进行调用 <code>Person</code> 类中的方法：

```Swift
@objc protocol PersonProtocol: JSExport {
    var name: String { get }
    func printName()
}

@objc class Person: NSObject, PersonProtocol {
    var name: String {
        return "iCell"
    }
    
    func printName() {
        print(self.name)
    }
}

let person = Person()
context.setObject(person, forKeyedSubscript: "p" as NSString)
context.evaluateScript("p.printName()")
```

### 参考资料
1. [NSHipster Java​Script​Core](http://nshipster.com/javascriptcore/)
2. [RayWenderlich JavaScriptCore tutorial](https://www.raywenderlich.com/124075/javascriptcore-tutorial)