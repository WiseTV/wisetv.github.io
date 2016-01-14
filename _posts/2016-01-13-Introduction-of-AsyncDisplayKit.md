---
author: 李岩
comments: true
date: 2016-01-13 16:15:10+00:00
layout: post
title: AsyncDisplayKit原理简介
description: instroduce Facebook's AsyncDisplayKit
categories:
- iOS
---

Facebook发布了iOS UI框架AsyncDisplayKit（ASDK），这个框架被用于Facebook自家的应用Paper中，能够提高UI的流畅性并缩短响应时间。由于现在网上有关ASDK的资料不是太多，所以我们团队根据网上现有的资料整理了相关内容，给大家介绍一下AsyncDisplayKit的原理。
<!-- more -->
##AsyncDisplayKit
在我们开发APP时，为了保证它的用户界面的流畅和响应，APP渲染帧率应该和iOS基准保持同步，即60帧每秒。这意味着主线程对每一帧有60分之一的时间来推送，这大约是16毫秒，需要在如此短时间内来执行所有的布局和绘制的代码。而由于系统的性能开销，在导致丢帧之前，你的代码通常只有不到10毫秒的时间来执行。但苹果的标准技术运行在应用的主线程中，用户输入需要等到主线程执行完毕才能进行。所以在复杂的UI界面上其流畅性难以得到保证。

AsyncDisplayKit并没有使用特别高深的绘制或者如GPU优化等优化技巧，而是在UI展现过程中将几个重要阶段剥离主线程，能够提高UI的流畅性并缩短响应时间。除了绘制操作会损耗性能以外，在复杂屏幕上布局计算也非常昂贵,ASDK可以做到布局和渲染都不在主线程上。

ASDK的基本单元是Node，Node是UIView以及对应Layer的抽象层，与UIView的最大区别在于，Node是线程安全的，并可以设置对应的Node内部层次后在后台线程中运行，而UIView只能将大量的耗时操作全都放在主线程上进行。
ASDisplayNode对view和Layer拥有很好的抽象，你可以很方便的访问CALayer属性。



###ASDK主要运用场景：
* 复杂多元素视图界面

* 低性能硬件设备



###ASDK的优化内容：
AsyncDisplayKit是在UI展现过程中将几个重要阶段剥离主线程从而将UI的流畅性提高到极致。这几个重要阶段分别是：

* 布局

* 绘制

* 图像



####布局
UI框架中要布局一个UIView一般需要改写layoutSubviews或者layoutSublayers等函数，并且一旦修改了frame等属性必然会触发这类layout函数。而这些函数的实现往往自上而下，要根据container来measure子view的大小并且层层递归计算下去。一旦layoutSubview的计算成本过大，必然会导致UI的响应缓慢或者刷新有delay。
在计算和布局自定义视图时在主线程一次做完所有的。例如在最小范围内些一个textView和imageView会是这样：

{% highlight objective-c %}
- (CGSize)sizeThatFits:(CGSize)size
{
  // size the image
  CGSize imageSize = [_imageView sizeThatFits:size];

  // size the text view
  CGSize maxTextSize = CGSizeMake(size.width - imageSize.width, size.height);
  CGSize textSize = [_textView sizeThatFits:maxTextSize];

  // make sure everything fits
  CGFloat minHeight = MAX(imageSize.height, textSize.height);
  return CGSizeMake(size.width, minHeight);
}

- (void)layoutSubviews
{
  CGSize size = self.bounds.size; // convenience

  // size and layout the image
  CGSize imageSize = [_imageView sizeThatFits:size];
  _imageView.frame = CGRectMake(size.width - imageSize.width, 0.0f,
                                imageSize.width, imageSize.height);

  // size and layout the text view
  CGSize maxTextSize = CGSizeMake(size.width - imageSize.width, size.height);
  CGSize textSize = [_textView sizeThatFits:maxTextSize];
  _textView.frame = (CGRect){ CGPointZero, textSize };
}
{% endhighlight %}
我们对子视图做了两次计算，第一次算出了视图需要多大，第二次计算是布局时，然而当我们的布局算法很简单时，我们也在计算文本时阻塞了主线程。

在控制View的frame的时候，ASDisplayNode采用了另外一套形式，分别在以下两个方法中。

{% highlight objective-c %}
// perform expensive sizing operations on a background thread
- (CGSize)calculateSizeThatFits:(CGSize)constrainedSize
{
  // size the image
  CGSize imageSize = [_imageNode measure:constrainedSize];

  // size the text node
  CGSize maxTextSize = CGSizeMake(constrainedSize.width - imageSize.width,
                                  constrainedSize.height);
  CGSize textSize = [_textNode measure:maxTextSize];

  // make sure everything fits
  CGFloat minHeight = MAX(imageSize.height, textSize.height);
  return CGSizeMake(constrainedSize.width, minHeight);
}

// do as little work as possible in main-thread layout
- (void)layout
{
  // layout the image using its cached size
  CGSize imageSize = _imageNode.calculatedSize;
  _imageNode.frame = CGRectMake(self.bounds.size.width - imageSize.width, 0.0f,
                                imageSize.width, imageSize.height);

  // layout the text view using its cached size
  CGSize textSize = _textNode.calculatedSize;
  _textNode.frame = (CGRect){ CGPointZero, textSize };
}
{% endhighlight %}

需要注意以下几点：</br>
1. Nodes在-calculateSizeThatFits:的实现方法中递归地计算他们的子节点。</br>
2、Nodes会执行其他的费时的预布局计算在-calculateSizeThatFits:，会适当地缓存中间的结果。</br>
3、Nodes必要时会调用[self invalidateCalculatedSize]。例如ASTextNode会释放他们计算结果当它的attributedString的属性改变时。


#####ASRangeController-工作范围控制器
工作范围的控制器跟踪可见范围，即当前屏幕可见内容的一部分，并且异步渲染下一步需要展示的几屏内容。当用户滚动时，一屏或两屏之前的内容被保存；其余的都可以清掉来节约内存。如果用户开始在另一个方向滚动，工作范围放弃现在的渲染队列并且开始在滚动的新方向进行预渲染—因为原有内容的缓冲，这整个过程通常是不可见（看不出来）的。 

#####ASTableView

ASTableView是一个集成基于node的cell和工作范围的UITableView子类。 
ASTableView与UITableView的区别：

1.设置asyncdelegate和asyncdatasource，而不是.dataSource,.delegate。</br>
 2.而替代实现TableView cellForRowAtIndexPath：：，您的数据源必须实现tableview：nodeforrowatindexpath：。这个方法必须是线程安全的，不应实现重用。不同于UITableView版本，它不会在行要显示时被调用。 </br>
 3.- TableView：heightforrowatindexpath：已删除，ASTableView让你的cell nodes自己调整自己的大小。这意味着你不再需要手动为动态大小的UITableViewCell复制或计算布局，改变大小的逻辑。 (在cellNodel的- (CGSize)calculateSizeThatFits:(CGSize)constrainedSize方法中写计算的方法)

AsyncDisplayKit在子线程中分批次计算行（row）以及其子元素的布局，计算好后通过dispatch_group_notify通知UI线程将row插入到view中。 AsyncDisplayKit有一个比较细腻的方式是考虑到设备的CPU核数，根据核数来分批次计算row的大小。每当row被sized之后，TableView便会触发row UI实体cell的生成

####绘制
AsyncDisplayKit另一个强大之处在于将UI CALayer的backing image的绘制放入到工作线程。

我们知道UIView的真正展现内容在于其CALayer的contents属性，而contents属性对应一个Backing image，我们可以将其理解成一个像素位图（CGImageRef）。/* An object providing the contents of the layer, typically a CGImageRef*/默认情况下UIView一旦需要展现，其会自动创建一个Backing image。

AsyncDisplayKit就是通过CALayer的delegate控制backing image的生成，并且通过Core Graphic的方式在工作线程上将View以及其子节点绘制到backing image上，这些绘制工作会根据UIView的层次构建一个绘制数组统一执行，绘制好之后在UI线程上将这个backing image传给CALayer的contents，最后将CALayer的渲染树交给GPU进行渲染。

####图像

目前网络上流行的图片格式基本都是压缩格式，而图片的显示大致可分为以下几部分：

* 图像的IO加载
* 图像的解压
* 图像的处理，如blend，crop等等
* 图像的渲染

AsyncDisplayKit在此主要优化第二和第三阶段图像


#####图像的解压

UIImage对其内部图像格式的解压发生在即将将图片交给GPU渲染的时候。从API上来看，一般我们调用[UIImage drawInRect]函数或者将UIView可见（放置于window上）的时候，UIImage会将其内部图像格式如PNG，JPEG进行解压。AsyncDisplayKit就是采用后面那个方式将UIView预先放置入一个frame为空得workingview中以达到将view内部的image进行预解压的目的。

#####图像的处理

一图像需要一些blend运算或者图像需要strech或crop，这些工作其实可以留在GPU中进行快速计算，但因为UIKit并没有提供此类的API，所以我们一般都是通过CoreGraphic来做，但CoreGraphic是CPU drawing比较费时。AsyncDisplayKit将这些图像处理放在工作线程中来处理，虽然也是CPU drawing，但不会影响到UI得顺滑响应