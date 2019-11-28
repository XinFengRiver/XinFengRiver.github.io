---
layout: default
title:  "swiper的CSS-3D实现"
description: "一种基于透视和平移变换的方案"
date:   2019-11-28 09:37:00 +0800
categories: fend
---

最近做了一个移动端的需求，页面的主体是三个红包，手指在屏幕上左右滑动可以切换红包的位置。

![红包原型图](/assets/image/posts/swiper-css-3d/red-envelope-prototype-graphic.jpg){: .img-center}

这个场景本质上是一个无限循环的轮播组件，有成熟的实现思路，例如凹凸实验室的[《实现一个 Swiper》](https://aotu.io/notes/2017/07/17/design-a-swiper/index.html){:target="_blank"}，我的实现就参考了这篇文章及其demo代码。
swiper的实现可以拆解为三个关键部分：数据结构、touch事件、css3d及动画。本文的目的是仅对css3d部分提出更好的实现方法，重点是css-3d中的透视原理与相关的值计算，除了一个必要的简化模型，不会涉及其他部分的实现。如果读者希望先详细地了解全部的实现，建议先阅读《实现一个 Swiper》。
我fork了仓库并提交了这次改动相关的核心代码和demo，可以结合这些代码阅读这篇文章。[仓库地址→](https://github.com/XinFengRiver/mobile-swiper)

## 一个简化的模型
在描述如何实现CSS之前（局部），先建立一个最简单的模型（总体）。

![简化模型](/assets/image/posts/swiper-css-3d/simplified-model.jpg){: .img-center}

假设轮播组件内有5张图，使用一**个双向的循环队列**来存储它们，可以用JS的数组来模拟循环队列，向左滑动就是队列头元素出列、插入到队列尾（`queue.push(queue.shift())`），向右滑动就是队列尾元素出列、插入到队列头（`queue.unshift(queue.pop())`）。

下方的图片分别是初始状态的数据结构、向左滑动时的数据操作、向右滑动时的数据操作。

![数据结构](/assets/image/posts/swiper-css-3d/data-structure.jpg){: .img-center}

以上便是swiper的存储结构和操作方法。

将队列元素的值与DOM节点映射，就可以得到每个DOM节点在队列中的位置了。同时，新建一个数组来存储特定位置的DOM节点的样式。这样，队列元素、DOM节点、样式数组元素就建立一一对应的关系。当swiper切换之时，可以对循环队列做一次切换，即`queue.push(queue.shift())`或`queue.unshift(queue.pop())`，此时DOM节点在队列的位置就发生了一次变化，这时候再根据【位置-样式】的映射关系更新DOM节点的CSS。

![队列与css的映射关系](/assets/image/posts/swiper-css-3d/queue-and-css-mapping.jpg){: .img-center}

```html
<ul>
    <li>1</li>
    <li>2</li>
    <li>3</li>
    <li>4</li>
    <li>5</li>
</ul>
```
```javascript
const css = [
    '位置1的css',
    '位置2的css',
    '位置3的css',
    '位置4的css',
    '位置5的css',
]

```

到这里，已经概括了swiper的几个实现要点，更加详细的内容请参考[《实现一个 Swiper》](https://aotu.io/notes/2017/07/17/design-a-swiper/index.html)。前戏看完，接下来就是本文的主要内容了——*如何为swiper添加css属性，使之3d效果更加合理，其原理又是几何*？

## 使用css3实现远小近大的透视视觉
### 原版的CSS代码

![原版css](/assets/image/posts/swiper-css-3d/css-original.jpg){: .img-center}

原版的代码摘抄：
```javascipt
  const rem = px => px / 40 + 'rem'; // 将长度单位px转化为rem
  transition = "-webkit-transition: -webkit-transform .3s ease";
  $ul.style["-webkit-transform-style"] = "preserve-3d";
  css = [
    "z-index: 3; -webkit-transform: translate3d(0, 0, 10px) scale3d(1, 1, 1); visibility: visible;", 
    "z-index: 2; -webkit-transform: translate3d(" + rem(-148) + ", 0, 6px) scale3d(.8, .8, 1); visibility: visible;", 
    "z-index: 2; -webkit-transform: translate3d(" + rem(148) + ", 0, 6px) scale3d(.8, .8, 1); visibility: visible;", 
    "z-index: 1; -webkit-transform: translate3d(" + rem(-240) + ", 0, 2px) scale3d(.667, .667, 1); visibility: visible;", 
    "z-index: 1; -webkit-transform: translate3d(" + rem(240) + ", 0, 2px) scale3d(.667, .667, 1); visibility: visible;"
  ]; 
```
根据原文的分析，这三个属性分别有以下目的：
1. `z-index`是为了区分swiper中5张图的层次，其中1在最上层，2与5在次层，4和3在最底层。
2. `scale3d`是为了实现“远小近大”的透视感，层数越深的图片比例缩小得越大。
3. `translate3d`有两个作用，第一个作用是为了让图片摊开，即等同于`translateX`；第二个作用是解决动画切换过程中有一小段过渡是后面的元素的位置变成在前方的元素上方的问题，这个现象文字表达比较困难，建议直接看原文[《实现一个 Swiper》](https://aotu.io/notes/2017/07/17/design-a-swiper/index.html)。

![](https://misc.aotu.io/leeenx/swiper/20170716_9.png?v=2){: .img-center}

然而，利用透视和平移变换可以更简洁地实现层叠与“远小近大”的效果。

### 新的实现方式
先来看一看最终的效果图。

![改版css](/assets/image/posts/swiper-css-3d/css-reversion.jpg){: .img-center}

```javascipt
  transition = "-webkit-transition: -webkit-transform .3s ease";
  $ul.style["-webkit-transform-style"] = "preserve-3d";
  $ul.style["perspective"] = rem(1000);
  css = [
    "-webkit-transition: -webkit-transform .3s ease; -webkit-transform: translate3d(0, 0, 0); visibility: visible;",
    "-webkit-transition: -webkit-transform .3s ease; -webkit-transform: translate3d(" + rem(-185) + ", 0, " + rem(-250) + "); visibility: visible;",
    "-webkit-transition: -webkit-transform .3s ease; -webkit-transform: translate3d(" + rem(185) + ", 0, " + rem(-250) + "); visibility: visible;",
    "-webkit-transition: -webkit-transform .3s ease; -webkit-transform: translate3d(" + rem(-360) + ", 0, " + rem(-500) + "); visibility: visible;",
    "-webkit-transition: -webkit-transform .3s ease; -webkit-transform: translate3d(" + rem(360) + ", 0, " + rem(-500) + "); visibility: visible;"
  ]; 
```
我为父元素`ul`的样式添加了`perspective`属性，使得子元素`li`产生了透视的效果。为了将swiper的第二层和第三层元素（图片）分别按0.8和0.667缩放，在透视效果的前提下，我修改了这几个元素的`translateZ`属性，使得它们在产生层次的时候同时也缩小比例，而不使用`scale3d`缩小元素。另外，为了让第二、三层的元素（图片）摊开，我还要修改它们的`translateX属性`。
那么问题来了：*父元素的`perspective`和子元素的`translate3d`两者存在什么样的关联？又如何计算两者的值呢？*

### 透视的原理与值计算

![perspective图解](/assets/image/posts/swiper-css-3d/perspective-graphic.jpg){: .img-center}

想象一张保鲜膜（就是图中的视窗），保鲜膜前有一双眼睛（观察位置），保鲜膜后方是一个正面朝着眼睛的立方体，立方体的光穿过保鲜膜在眼睛成像，在这条光路上，光在保鲜膜形成的是一个缩小的图像。如果正方体距离保鲜膜越远（`translateZ`越大），那么保鲜膜上的图像就越小；如果眼睛距离保鲜膜越近（`perspective`越小），那么保鲜膜的图像就越小。
假设原来的立方体宽高分别为`width`和`height`，保鲜膜上的图像宽高分别是`width'`和`height'`，缩小的比例为`scale`，观察位置为`perspective`，立方体在Z轴的位移为`translateZ`，可以由以下公式表示`scale`、`perspective`、`translateZ`三者的关系：

\\[ scale = \frac{perspective}{perspective - translateZ} \\]

设`perspective`为`rem(1000)`，那么可以算得第二层（`scale=0.8`）和第三层元素（`scale=0.667`）的`translateZ`分别为`rem(-250)`和`rem(-500)`。
值得注意的是，因为透视的原因横纵向的距离都变短了，此时`translateX`也不能直接取原来的值了。记原来元素在x轴方向上的变换为`translateX`，透视后的变换为`translateX'`，以下关系成立：
\\[ translateX = {translateX'}\times{\frac{1}{scale}} 
  = {translateX'}\times{(1-\frac{transalateZ}{perspective})}
\\]
计算得第二层（`scale=0.8`）和第三层元素（`scale=0.667`）的`translateX`分别为`rem(-185)`/`rem(185)`和`rem(-360)`/`rem(360)`。
最后，可以得到四个元素的平移变换分别为`translate3d(rem(-185), 0, rem(-250))`、`translate3d(rem(185), 0, rem(-250))`、`translate3d(rem(-360), 0, rem(-500))`、`translate3d(rem(360), 0, rem(-500))`。

## 总结
这篇文章讲述了一种实现swiper样式的方法。借助于《实现一个 Swiper》一文所述，我们建立了swiper的最简模型，并分析了原作的css代码。接着，我们阐述了css种透视的原理，给出了目标元素宽高尺寸缩小比、`perspective`、`translate3d`三者之间的计算公式，最终得出了一个更简洁的解决方法。最后再罗列一次相关的属性：
- 父元素：`transform-style`、`perspective`
- 子元素：`transform: translate3d()`

