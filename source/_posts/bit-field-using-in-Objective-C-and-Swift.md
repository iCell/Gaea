---
title: 位运算在 Objective-C 和 Swift 中的一点运用
date: 2017-01-30 14:25:51
tags:
---
在 iOS 开发中，经常会使用枚举表示一些选项或者状态，而有的时候一个状态是需要枚举中的多个选项进行组合起来表示的，比如最常见的 <code>UIViewAutoresizing</code> 就可能同时需要使用 <code>UIViewAutoresizingFlexibleWidth</code> 和 <code>UIViewAutoresizingFlexibleHeight</code> 进行表示，所以会通过位操作中的 | 运算进行组合。而位运算实际上是直接对二进制数据进行操作的，在处理速度上会比较快，所以这里就总结一下 Objective-C 和 Swift 中位运算符常见的一些使用。
<!-- more -->

首先呢，位运算符以及具体用法如下：

|   运算符     |   规则  |
|---            |---    |
|       & 与   |   都为 1 时结果为 1                       |
|   ∣ 或    |   都为 0 时结果为 0                     |
|   ^ 异或   |   相同为 0 否则为 1                     |
|   \~ 取反   |   0 为 1，1 为 0                     |
|   << 左移   |   左移，高位丢弃，0 补低位               |
|   >> 右移   |  右移，高位根据编译器补 0 或者补符号位     |

其实，单就论用位操作做一些数据的处理其实很巧妙，比如用来判断奇数还是偶数可以通过二进制数最后一位来判断，0 为偶数，1 为奇数，那么只需要通过对一个数字 a 进行 <code>a & 1</code> 操作通过结果就能判断出来这个数字是奇数还是偶数。再比如两个数 a b 进行数据交换，一般引入第三个数 c 当作中间转换用的，但其实通过 ^ 操作符，执行下列代码即可进行交换。

```Objective-C
a ^= b;
b ^= a;
a ^= b;
```

回到主题，比如 UIKit 中关于 <code>UIViewAutoresizing</code> 的声明如下：

```Objective-C
typedef NS_OPTIONS(NSUInteger, UIViewAutoresizing) {
    UIViewAutoresizingNone                 = 0,
    UIViewAutoresizingFlexibleLeftMargin   = 1 << 0,
    UIViewAutoresizingFlexibleWidth        = 1 << 1,
    UIViewAutoresizingFlexibleRightMargin  = 1 << 2,
    UIViewAutoresizingFlexibleTopMargin    = 1 << 3,
    UIViewAutoresizingFlexibleHeight       = 1 << 4,
    UIViewAutoresizingFlexibleBottomMargin = 1 << 5
};
```

> 这里会发现使用了 NS\_OPTIONS 而不是 NS\_ENUM, 关于这两者的却别可以看一下这篇文章[NS\_ENUM vs NS\_OPTIONS](http://nshipster.com/ns_enum-ns_options/)

关于 <code>UIViewAutoresizingNone</code> 就没什么好说的了，但是如何通过进行按位或操作就能同时组合实现多个选项呢，原理其实也不难，比如现在我们实现 <code>UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight</code> 这样的代码，那么 autoresizing 属性就能既实现了 width 上的动态调整又实现了 height 上的动态调整，我们先一步一步进行计算。

<code>UIViewAutoresizingFlexibleWidth = 1 << 1</code> , 1 转换成二进制为 00000001，然后执行 1 << 1 之后结果为 00000010；

<code>UIViewAutoresizingFlexibleHeight = 1 << 4</code> , 1 转换成二进制然后执行左移 4 为操作结果为 00010000

现在 <code>UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight</code> 也就相当于上述两个结果执行按位或操作：

> 00000010 | 00010000 = 00010010

也就是说我有一个 <code>UIView</code>, 现在对这个 view 的 <code>autoresizingMask</code> 属性赋值

```Objective-C
view.autoresizingMask = UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
```

结果按照 | 操作的结果来定的话，那么这个 <code>autoresizingMask</code> 的值就是 00010010. 所以接下来 <code>UIView</code> 内部根据 <code>autoresizingMask</code> 进行一些 resize 操作的话，就要看看这个属性包含哪些选项，这时候只需要将  <code>autoresizingMask</code> 和某个选项进行 & 操作即可判断出属性是否包含这一选项。比如刚刚的 <code>autoresizingMask</code> 值为 00010010，而 <code>UIViewAutoresizingFlexibleWidth</code> 的值为 00000010，现在
> 00010010 & 00000010 = 00000010

转换为十进制为 2，如果对这个结果执行 true / false 判断的话，结果为 true，也就是说 <code>autoresizingMask</code> 属性包含了 <code>UIViewAutoresizingFlexibleWidth</code> 选项，就能针对此做一些操作了。按照这个逻辑继续对其他的一些选项进行判断，也能得到相应正确的结果。

上面所说的都是 Objective-C 下的情况，那么如果我们使用的是 Swift，情况就大不同了。在 Swift 中，<code>enum</code> 是不能够多个选项进行相互组合的，也就是说类似于上述 Objective-C 中使用 <code>|</code> 运算符进行 <code>enum</code> 多项组合在 Swift 中是完全不能做到的，那 Swift 是如何处理这种情况的呢？看一下 Swift 版本的 <code>UIViewAutoresizing</code> 就知道了：

```Swift
public struct UIViewAutoresizing : OptionSet {

    public init(rawValue: UInt)

    public static var flexibleLeftMargin: UIViewAutoresizing { get }
    public static var flexibleWidth: UIViewAutoresizing { get }
    public static var flexibleRightMargin: UIViewAutoresizing { get }
    public static var flexibleTopMargin: UIViewAutoresizing { get }
    public static var flexibleHeight: UIViewAutoresizing { get }
    public static var flexibleBottomMargin: UIViewAutoresizing { get }
}
```

这里可以看到，在 Swift 中的 <code>UIViewAutoresizing</code> 并不是一个 <code>enum</code>，而是一个 <code>struct</code>，且引入了 <code>OptionSet</code> 这个协议。之所以是 <code>struct</code> 而不是 <code>enum</code> 就是因为 Swift 中的 <code>enum</code> 是不支持多个值进行组合的，而 <code>OptionSet</code> 的引入就是为了解决 Swift 关于这种需求的方案。

<code>OptionSet</code> 遵循 <code>SetAlgebra</code> 协议，所以在声明的时候可以直接使用类似于数组声明的方式进行创建，比如

``` Swift
let resizingMasks = [UIViewAutoresizing.flexibleHeight, UIViewAutoresizing.flexibleWidth]
```

但是呢，它看起来像数组，并不是数组，它没有遵循 <code>Sequence</code> 或者 <code>Collection</code> 这类协议，没有 <code>count</code> 方法，就算上述代码看似像是赋了两个值，但其实结果只有一个值。<code>OptionSet</code> 其实也算是替代本文一开始说的需求来的，所以实际上内部也是进行了位运算，比如 <code>OptionSet</code> 有 <code>contain</code> 方法判断某一选项是否包含在内，还有 <code>union</code> 方法对两个数进行位运算操作。所以如果使用 Swift 遇到多个状态互相组合的情况，可以创建一个遵循 <code>OptionSet</code> 协议的 struct 即可解决问题。

参考：

* [StackOverflow: multiple value enum in Objc-C](https://stackoverflow.com/questions/4176149/multiple-value-enum-in-obj-c)
* [StackOverflow: OptionSetType and enums](https://stackoverflow.com/questions/36819163/optionsettype-and-enums)
* [Option Sets in Swift](https://oleb.net/blog/2016/09/swift-option-sets/)