---
layout: default
title:  "管理单页应用的页面过渡效果——vue生态的实现"
description: ""
date:   2019-12-30 6:00:00 +0800
categories: fend
---

单页应用的一个优点是可以自行实现页面切换时的过渡效果，一个常见的场景是顶部导航栏固定不变，下方的页面随着路由组件的切换而呈现一些过渡效果。例如，我们希望从当前页导航下一页时，页面是从右到左过渡的；从当前页导航到上一页时，页面是从左过渡到右的，效果见下面的demo。

{% include codepen.html hash="WNbOKvJ" title="效果图" %}

## 前言
这篇文章会介绍一种管理单页应用的页面切换效果的实现思路，该实现具备低耦合、高拓展性的特点。我平时开发用的是vue生态的工具，所以实现部分是倚靠vue生态的（当然，思路是可以举一反三的，也可用于其他的框架）。

我会遵循循序渐进的原则，由最小可用的实现开始，逐渐引出趋于完善的解决方案。理解`<transition>`的高级用法是实现页面过渡的基础，所以在文章伊始，我会先罗列一些`<transition>`的用法，在不引入`vue-router`的情况下实现页面切换的效果，引导读者熟悉`<transition>`。接着，我会介绍`<transition>`和`vue-router`是如何配合以实现两个路由组件(页面)之间的过渡效果的。最后，我会引入vuex和路由层级的概念来管理大规模的路由组件。

## 一、使用transition实现过渡效果
这一节介绍的`<transition>`的知识可以在官方文档找到，想要全面了解的用法，建议通读[《进入/离开 & 列表过渡》](https://cn.vuejs.org/v2/guide/transitions.html){:target="_blank"}。

官网是这样介绍`<transition>`的适用情景的：`<transition>`是vue的一个封装组件，在下列情形中，可以给任何元素和组件添加进入/离开过渡：
- 条件渲染 (使用 v-if)
- 条件展示 (使用 v-show)
- 动态组件
- 组件根节点

第一个demo是为了熟悉`<transition>`的使用，为了简化使用场景，会用最简单的`v-if`情景。

{% include codepen.html hash="BaydXVE" title="transition的用法1" %}

这个demo是无任何过渡效果的，观察它的HTML结构，两个`<div>`元素分别代表单页应用的两个页面，点击中间的文字可以切换页面。

接下来，我们为页面切换加上过渡效果。`<transition>`组件同时支持使用CSS+class或JS钩子实现过渡，我用的是CSS+class。在进入/离开的过渡中，会有6个class依次切换。

![过渡的类名](https://cn.vuejs.org/images/transition.png){: .img-center width="500px"}

这6个class的调用时机，官网是这样说的：
1. `v-enter`：定义进入过渡的开始状态。在元素被插入之前生效，在元素被插入之后的下一帧移除。
2. `v-enter-active`：定义进入过渡生效时的状态。在整个进入过渡的阶段中应用，在元素被插入之前生效，在过渡/动画完成之后移除。这个类可以被用来定义进入过渡的过程时间，延迟和曲线函数。
3. `v-enter-to`: 2.1.8版及以上 定义进入过渡的结束状态。在元素被插入之后下一帧生效 (与此同时 v-enter 被移除)，在过渡/动画完成之后移除。
4. `v-leave`: 定义离开过渡的开始状态。在离开过渡被触发时立刻生效，下一帧被移除。
5. `v-leave-active`：定义离开过渡生效时的状态。在整个离开过渡的阶段中应用，在离开过渡被触发时立刻生效，在过渡/动画完成之后移除。这个类可以被用来定义离开过渡的过程时间，延迟和曲线函数。
6. `v-leave-to`: 2.1.8版及以上 定义离开过渡的结束状态。在离开过渡被触发之后下一帧生效 (与此同时 v-leave 被删除)，在过渡/动画完成之后移除。

其中，`v`是可以替换成用户自定义的`name`属性的（对应`<transition>`的`name`属性）。例如，`slide-enter`对应`<transition name="slide">`。而且，**`name`是可以动态绑定的**（`<transition :name="transitionName">`），这一特性非常重要，是动态改变过渡效果依赖的基础。

### demo: 实现slide-left和slide-right

我们约定导航到下一页和上一页的`name`分别为`slide-left`和`slide-right`，实现的效果如下：

{% include codepen.html hash="YzPQpgW" title="transition的用法-last" %}

我们以`slide-left`为例详解各个阶段的class对应的css。可以想象到页面在`slide-left-enter-to`和`slide-left-leave`的状态是一样的，都是一个至少占满整个屏幕的页面，所以需要绝对布局：

```css
.slide-left-enter-to, .slide-left-leave {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  min-height: 100%;
}
```

`slide-left-leave-to`是当前页离开的结束态，应当相对视窗左偏移100%；`slide-left-enter`是下一页进入的初始态，应当相对视窗右偏移100%。进入视窗和离开视窗时，页面的透明度应该也要有所变化，在屏幕（视窗）中间都是`opacity: 1`，在屏幕外都是`opacity: 0`。

```css
.slide-left-enter {
  transform: translateX(100%);
  opacity: 0;
}
.slide-left-leave-to {
  transform: translateX(-100%);
  opacity: 0;
}
```

我们为页面进入和离开的中间状态加上过渡声明：

```css
.slide-left-enter-active, .slide-left-leave-active {
  transition: all .5s;
}
```

`slide-right`的实现类似，就不再赘述了。用户切换页面时，监听点击事件的处理函数会改变`transitionName`，从而实现从右到左或从左到右的过渡效果。

```
changeView() {
  this.view = this.view === 'view1' ? 'view2' : 'view1'
  if (this.view === 'view2') {
     this.transitionName = 'slide-left'
  } else {
     this.transitionName = 'slide-right'
  }
}
```

你应该注意到，我为`<transition>`包裹的元素添加了`key`属性，如果没有添加这个属性，页面的切换是没有过渡效果的——当有相同标签名的元素切换时，需要通过`key`特性设置唯一的值来标记以让`Vue`区分它们，否则`Vue`为了效率只会替换相同标签内部的内容。仅替换内容相同的内容，当然不会有过渡效果。

至此，我们使用`<transition>`及其class的钩子实现了单页应用页面切换的效果。生产中我们一般会使用`vue-router`管理路由组件，接下来我们尝试结合`<transition>`和`vue-router`。

## 二、使用vue-router管理路由组件
在本节开始前，我希望你已经了解vue-router的基本概念[《vue router guide》](https://router.vuejs.org/zh/guide/){:target="_blank"}。我们平时称之为"页面"的，在vue-router中的概念是"**路由组件**"，这很好理解——真正的页面只有一个，在单页应用中所谓的"页面"实际上是激活或失活的组件，使用了vue-router后，这些组件的激活与失活是由路由决定的，所以我们称之为"路由组件"，后文我就直接称"页面"为路由组件了。

平时开发我们一般用单文件的方式定义路由组件，写在一个单独的`.vue`文件上。而在demo演示时，`X-Template`的定义方式更小巧，不了解的读者可以查阅官方手册[《X-Template》](https://cn.vuejs.org/v2/guide/components-edge-cases.html#X-Template){:target="_blank"}。其写法是这样的：

```html
<script type="text/x-template" id="page1">
  <div style="background: #FF8A65;">
    <h2>page1</h2>
    <router-link to="/page2">下一页</router-link>
  </div>
</script>
```

我们在demo中引入`vue-router`：

{% include codepen.html hash="VwYWxdV" title="使用vue-router管理组件切换" %}

我在`routes`中定义了两个路由，并将组件page1和page2映射到路由，使用了`<router-link>`来实现page1和page2之间的导航。

我们在合适更改`<transition>`的`name`属性呢？答案很明显——改变`name`的时机必须是在路由变更之前。所以，我选用了`vue-router`的导航守卫`beforeEach`，这是一个全局前置守卫，只要触发导航，就会按定义的顺序执行`beforeEach`注册的代码。

```javascript
router.beforeEach((to, from, next) => {
  if (to.name === 'page1') {
     vm.transitionName = 'slide-right' // vm是vue实例
  } else {
     vm.transitionName = 'slide-left'
  }
  next()
})
```

每次`<router-view>`挂载的路由组件变化，就会先触发`beforeEach`钩子，从而执行修改`transitionName`的代码，实现路由组件的动态过渡效果。但是这个demo的实现仍有一些不足：
1. 我是直接根据目标路由的名称变更`name`属性，但是在大型的应用中，最好还是引入路由层级的概念管理路由（注意，不是'路由嵌套'）。
2. `transitionName`挂载到了全局`vue`实例中，实际生产环境中`transitionName`可能在多个组件用到，使用`vuex`管理`transitionName`更合理。

我将在下一节讲述解决这些问题的思路。

## 三、提升路由管理的可拓展性
### 1. 路由层级
如何管理大规模的路由呢？如果你要处理的事情很多，一个很好的习惯是先分优先级，处理大规模的路由也是同样的道理。我们可以将路由的结构想象成一棵树，树的最顶端是整个应用的入口路由，记为`level: 1`。入口路由可以导航到第二层的路由，记第二层的路由为`level: 2`，第三层、第四次、第N层路由以此类推。当然，现实中的路由组织结构应当是图，任意一个路由都可以导航到其他的路由，但是我们探讨问题时，尽量使用简化的模型。

![路由树](/assets/image/posts/vue-transition-and-router/router-tree.png){: .img-center width="350px"}

对于某一个路由，可以用"同级路由"、"上级路由"、"下级路由"三个概念描述它所处的位置。观察上图，路由A(`level: 2`)的同级路由是B(`level: 2`)，下级路由是C、D、E(`level: 3`)，上级路由是入口路由(`level: 1`)。我们延扩这几个概念的含义——**对于某一个特定路由，level与之相同的路由是同级路由，level较之更小的是上级路由，level较之更大的是下级路由**，那么，图中F可以看作是A的下级路由。

对于demo中的page1和page2，我们可以这样定义：
```javascript
const routes = [
  { 
    name: 'page1',
    path: '',
    component: Page1,
    meta: { level: 1 },
  },
  { 
    name: 'page2',
    path: '/page2',
    component: Page2,
    meta: { level: 2 },
  },
]
```

`meta`是路由的[元信息](https://router.vuejs.org/zh/guide/advanced/meta.html#%E8%B7%AF%E7%94%B1%E5%85%83%E4%BF%A1%E6%81%AF){:target="_blank"}，为我们提供了自定义路由属性的手段，可以在`meta`中定义`level`，并通过`route.meta.level`获取路由的层级。

为路由分配好`level`后，我们可以设计一个根据路由层级动态变更过渡效果的方案：

```javascript
router.beforeEach((to, from, next) => {
  const toLevel = to.meta.level
  const fromLevel = from.meta.level
  if (toLevel < fromLevel) {
     vm.transitionName = 'slide-right'
  } else if (toLevel > fromLevel) {
     vm.transitionName = 'slide-left'
  } else {
    vm.transitionName = '' // 或者其他你定义的css类
  }
  next()
})
```

{% include codepen.html hash="eYmGvLz" title="使用vue-router管理组件切换" %}

这是增加了路由层级之后的demo。有了路由层级，后续增加路由只需要定义其level，不需要修改`router.beforeEach`的代码，达到"对拓展开放、对修改封闭"的设计目的。

### 2. 使用vuex管理transition name
在大型应用中，我们的路由设计通常会用到[嵌套路由](https://router.vuejs.org/zh/guide/essentials/nested-routes.html){:target="_blank"}，以提升应用界面的模块化水平、实现组件的复用。在嵌套路由的场景中，我们通常也希望子`<router-view>`替换组件时也有过渡效果，但是`<transition>`只对直接后代元素有效，所以我们还需要在子`<router-view>`外部加一个`<transition>`元素。你可以直观地理解为这样的结构：

```html
<div id='app'>
  <transition>
    <router-view>
      <transition>
        <router-view></router-view>
      </transition>
    </router-view>
  </transition>
</div>
```

因为各个vue组件的实例是不共享数据的，为了让两个`<transition>`都能用上`transitionName`，`transitionName`必须要放到一个用于存储数据的单例中——`vuex`。我们在demo中引入vuex：

{% include codepen.html hash="NWPajjx" title="使用vuex管理transition name" %}

我在vuex增加了`state.transitionName`和`mutations.transitionName`，分别在计算属性和`router.beforeEach`中用到它们。

现在，让我们回顾我在前言中提到的场景：视窗顶部有一个固定的导航栏，路由组件切换的时候导航栏不需要重新渲染，只有路由组件才有过渡的效果。

#### demo：嵌套路由的过渡效果
我们新定义一个`Layout`组件，组件包含顶部的导航栏和一个`<router-view>`，page1和page2交由`Layout`组件渲染。

```html
<script type="text/x-template" id="layout">
  <div>
    <div class="nav">
      <a>返回</a>
    </div>
    <transition :name="transitionName" appear>
      <router-view class="inner-view"></router-view> <!-- page1和page2 -->
    </transition>
  </div>
</script>
```
```javascript
const Layout = {
  template: '#layout',
  computed: {
    transitionName() {
      return store.state.transitionName // 在计算属性中引用transitionName
    }
  }
}
```
我们在`Layout`的`<router-view>`外部也包了一个`<transition>`，并将`transitionName`绑定到了`name`属性，因此`Layout`内路由切换的过渡效果也会遵循既有的规则。下面是完整的demo：

{% include codepen.html hash="WNbOKvJ" title="嵌套路由与transitionName" %}

看到这里，想必你也已经掌握了如何使用路由层级和vuex去管理大规模的路由了。我想重申下这套方案是符合"开闭原则"的——如果你想增加路由，只需要添加`meta.level`；如果你想增加嵌套路由，只需要在上层的路由组件中也引入`store.state.transitionName`。

## 结语
最后，让我们用一张简单扼要的图来总结这篇文章。

```
vue-router
+----------------------+                 +----------------------+
| App                  |                 | App                  |
| <transition>         |                 | <transition>         |
| +------------------+ |                 | +------------------+ |
| | Layout           | |                 | | Layout           | |
| | <transition>     | |                 | | <transition>     | |
| | +--------------+ | |    vue-router   | | +--------------+ | |
| | | Page1        | | |  +------------> | | | Page2        | | |
| | |              | | |       vuex      | | |              | | |
| | +--------------+ | |                 | | +--------------+ | |
| +------------------+ |                 | +------------------+ |
+----------------------+                 +----------------------+

```
