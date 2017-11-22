---
title: 在 macOS 上开发 Cocoa App 遇到的两个问题
date: 2017-09-20 13:07:14
tags:
---
最近自己写了个 mac app，遇到了两个稍微困扰的问题，在这里简单记录一下。一个是在 app 中进行网络请求的时候总是报错 “A server with the specified hostname could not be found”；另一个问题是 `NSTableCellView` 内容不垂直居中的问题。其实这两个问题的解决方法都很简单，但是自己遇到的时候脑子估计就是转不过来，debug 了半天才找出问题。
<!-- more -->

#### 网络请求报错的问题

在 Xcode（版本 9.0） 里使用了 Cocoa App 的模版创建了一个工程，在里面使用我自己写的一个 [CryptoCurrencyKit](https://github.com/iCell/CryptoCurrencyKit) 来进行网络请求的时候，总是报错，错误如下

> error Domain=NSURLErrorDomain Code=-1003 "A server with the specified hostname could not be found."

然后再仔细看了一下 log，有这么一行 log 打印了出来：

> dnssd_clientstub ConnectToServer: connect() failed path:/var/run/mDNSResponder Socket:7 Err:-1 Errno:1 Operation not permitted

看到了这个 log 我就断定那肯定是因为 DNS 解析失败导致的，然后尝试了切换网络环境、清除 DNS 缓存，发现问题还是一直存在，然后就开始怀疑人生了。后来我又把请求的 url 在浏览器里打开进行请求，发现完全正常，于是我随便写了个国内可以访问的网址在项目里进行请求，发现还是有这个问题，那就只能是我的工程配置有问题了。所以最终呢，找出了问题，是在 Xcode -> Capabilities 中打开了 App Sandbox，里面有个选项如下图：

![App Sandbox](http://7xjbza.com1.z0.glb.clouddn.com/xcode-capabilities-app-sandbox.png)

勾选一下就可以了，很尴尬...

#### NSTableCellView 内容无法垂直居中的问题

之前在做 iOS 开发的时候，tableView 用的很溜，本以为到 macOS 开发上应该也是很容易就转换过来，发现其实差别还是蛮大的，自己照着[这篇教程](https://www.raywenderlich.com/143828/macos-nstableview-tutorial)把想展示的内容给弄了出来，就发现了一个问题，cell 的文本无法垂直居中。我用的是 `NSTextFieldCell`，因为只需要展示纯文本即可，于是上网查了一下，发现这个问题好像还是老大难问题，当然也有很多人给出了[解决方法](https://stackoverflow.com/questions/1235219/is-there-a-right-way-to-have-nstextfieldcell-draw-vertically-centered-text)，方法无非就是继承 `NSTextFieldCell` 然后手动去改文本的位置让它居中，但是我照着这些方法试了一下发现没有用，还是纠结了很久。最后发现解决方法依然很简单，你 cell 内的文本不是不居中吗，给你用 AutoLayout 加一个垂直居中就解决了...

![NSTextFieldCell center vertically](http://7xjbza.com1.z0.glb.clouddn.com/nstextfieldcell-center-vertically.png)
