---
layout: p
title: NSTextAttachment 的使用
date: 2017-08-23 15:04:14
tags:
---
这是一篇老文章了，两年之前在简书里面写的，忽然间想起来，现在把文章搬过来（可能略有改动）。

记得好久之前想要实现一个在 iOS 上用 native 的代码实现 Markdown 的图文混排需求，由于需要支持 iOS 6，所以一个高效的实现方法就是把 Markdown 解析成 <code>NSAttributedString</code>，然后再用<code>Core Text</code> 进行文字的渲染，至于图片，是非常非常麻烦。当时是将 [Nimbus](https://github.com/NimbusKit/attributedlabel) 的一个开源的代码拿来改了改，虽说是站在大神的肩膀上，可是还是遇到了挺多坑。之后 iOS 7 出来了 <code>Textkit</code>，比写 <code>Core Text</code> 代码舒服多了，一直没有去用。正好这段时间要重构一些代码，就连学带写实现了一把，别的不想说，看 WWDC 上面有很详细的介绍，这里只想说一下里面关于 <code>NSTextAttachment</code> 的使用。
<!-- more -->
关于想使用 <code>TextKit</code> 在 <code>UITextView</code> 中实现插入图片，其实真的很简单，只需要按照下面三步就搞定了：

```Swift
let attachment = NSTextAttachment()
attachment.image = UIImage(named: "whatever.png")
let attributedStr = NSAttributedString(attachment: attachment)
```

其实只是使用 <code>NSTextAttachment</code> 将想要插入的图片作为一个字符处理，转换成 <code>NSAttributedString</code>，然后 <code>UITextView</code> 直接进行渲染就搞定了。上面的代码是初始化一个 <code>NSTextAttachment</code>，然后 set 一下 image 属性，也可以使用 <code>NSTextAttachment</code> 的 <code>init(data contentData: NSData?, ofType uti: String?)</code> 方法来设置图片。
做到上面这一步就可以将图片插入到 <code>UITextView</code> 中了，但是可能会出现下面这样的效果：

![](http://7xjbza.com1.z0.glb.clouddn.com/textattachment-too-large.png)

这是因为把图片添加之后，如果不进行处理的话，图片的大小是按照原图的尺寸来的。想要控制一下图片大小其实也很简单，<code>NSTextAttachment</code> 里面也提供了相应的方法，只需要继承一下 <code>NSTextAttachment</code>，然后实现下面的方法即可：

```Swift
func attachmentBoundsForTextContainer(textContainer: NSTextContainer, proposedLineFragment lineFrag: CGRect, glyphPosition position: CGPoint, characterIndex charIndex: Int) -> CGRect {
    let attachmentWidth = CGRectGetWidth(lineFrag) - textContainer.lineFragmentPadding * 2    
    return scaleImageSizeToWidth(attachmentWidth)
}

func scaleImageSizeToWidth:(CGFloat)width {
    let factor = CGFloat(width / self.image.size.width)
    return CGRect(x: 0, y: 0, width: self.image.size.width * factor, self.image.size.height * factor)
}
```

这样就可以让图片自动适应 <code>UITextView</code> 中的 <code>textContainer</code> 的宽度了。当然，如果你就是不想继承 <code>NSTextAttachment</code> 的话，你也可以自己显示的设置一下 bounds 属性。
另外，想要获取 <code>NSTextAttachment</code> 中的图片也很简单，可以自己加手势通过 <code>glyphRange</code> 什么的来获取，也可以直接实现 <code>UITextView</code> 中的 <code>delegate</code> 方法 <code>func textView(textView: UITextView, shouldInteractWithTextAttachment textAttachment: NSTextAttachment, inRange characterRange: NSRange) -> Bool</code> 来获取。

**One More Thing：**

在显示文本之后，想更改 <code>NSTextStorage</code> 中的 <code>NSFontAttributeName</code> 的字体，直接更改的话，图片显示就不正常了，显示的是被截取掉一部分的图片。所以我的解决方法是对文本中所有的 <code>attributes</code> 进行遍历，排除掉所有包含 <code>NSTextAttachment</code> 的文字来更改字体，这样造成的结果是如果文本很大的话，更换字体速度很慢。最后用的方法是插入图片之后记录图片的 <code>range</code> ，根据这些 <code>range</code> 来对整个文本内容进行分割，分割成几个不包含 <code>NSTextAttachment</code> 的文本，对这些文本更改字体即可。另外，还有一个图片显示不正确的情况是对段落设置了 <code>MaximumLineHeight</code> ，这样图片就是直接覆盖到文字之上，并没有文字根据图片来延伸的效果。