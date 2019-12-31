---
layout: default
title:  "使用<transition>和vue-router实现页面过渡效果"
description: ""
date:   2019-12-30 6:00:00 +0800
categories: fend
---

单页应用的一个优点是可以自行实现页面切换时的过渡效果，一个常见的场景是顶部导航栏固定不变，下方的页面随着路由组件的切换而呈现一些过渡效果。例如，我们希望从当前页导航下一页时，两个页面是从右到左过渡的；从当前页导航到上一页时，我们希望两个页面是从左过渡到右的，效果见下面的demo。

{% include codepen.html hash="WNbOKvJ" title="页面切换" %}
## 前言
这篇文章的目的是介绍一种可复用的单页应用路由的组件过渡效果的实现思路，我平时开发用的是vue生态的工具，所以实现部分是依靠vue生态的（当然，思路是可以举一反三的）。本文我会遵循循序渐进的原则，由最简单的实现开始，最终得出一个可用于生产环境的方案。在文章伊始，我会先罗列一些`<transition>`的用法，在不引入`vue-router`的情况下实现页面切换的效果，让读者熟悉`<transition>`。接着，我会介绍`<transition>`和`vue-router`是如何配合实现两个路由组件间的切换效果的。最后，我会提出带导航栏页面和多层级页面，并引入vuex管理这种复杂场景下的页面切换效果。

## 使用transition实现过渡效果
我只会对用到的知识做总结性的介绍，想要全面了解`<transition>`的用法，建议通读官方的文档[《进入/离开 & 列表过渡》](https://cn.vuejs.org/v2/guide/transitions.html){:target="_blank"}。

官网是这样介绍`<transition>`的：`<transition>`是vue的一个封装组件，在下列情形中，可以给任何元素和组件添加进入/离开过渡：
- 条件渲染 (使用 v-if)
- 条件展示 (使用 v-show)
- 动态组件
- 组件根节点

我们知道vue-router中页面发生变化是因为`<router-view>`渲染的路由组件不同，其本质就是动态组件的切换，可以归入到第三种情景。而我们的第一个demo是为了熟悉`<transition>`的使用，会用最简单的`v-if`情景。

{% include codepen.html hash="BaydXVE" title="transition的用法1" %}
这个demo中我两个`<div>`元素分别代表两个页面，点击中间的文字就可以切换页面。

接下来我们为切换加上过渡效果。如文章开头所说，我们希望从当前页导航下一页时是从右到左过渡的，从当前页导航到上一页时是从左过渡到右的。所以，我们要实现两种过渡状态，而这，正是`<transition>`发挥作用的地方。

`<transition>`组件同时支持使用CSS+class或JS钩子实现过渡，我用的是CSS+class。在进入/离开的过渡中，会有6个class依次切换。

![过渡的类名](https://cn.vuejs.org/images/transition.png){: .img-center}

这6个class的调用时机，官网是这样说的：
1. `v-enter`：定义进入过渡的开始状态。在元素被插入之前生效，在元素被插入之后的下一帧移除。
2. `v-enter-active`：定义进入过渡生效时的状态。在整个进入过渡的阶段中应用，在元素被插入之前生效，在过渡/动画完成之后移除。这个类可以被用来定义进入过渡的过程时间，延迟和曲线函数。
3. `v-enter-to`: 2.1.8版及以上 定义进入过渡的结束状态。在元素被插入之后下一帧生效 (与此同时 v-enter 被移除)，在过渡/动画完成之后移除。
4. `v-leave`: 定义离开过渡的开始状态。在离开过渡被触发时立刻生效，下一帧被移除。
5. `v-leave-active`：定义离开过渡生效时的状态。在整个离开过渡的阶段中应用，在离开过渡被触发时立刻生效，在过渡/动画完成之后移除。这个类可以被用来定义离开过渡的过程时间，延迟和曲线函数。
6. `v-leave-to`: 2.1.8版及以上 定义离开过渡的结束状态。在离开过渡被触发之后下一帧生效 (与此同时 v-leave 被删除)，在过渡/动画完成之后移除。

其中，`v`是可以替换成用户自定义的`name`属性的（没错，就是`<transition>`的`name`属性）。例如，`slide-enter`对应`<transition name="slide">`。而且，**`name`是可以动态绑定的**（`<transition :name="transitionName">`），这一特性非常重要，是实现动态改变过渡效果所依赖的基础。

我们约定导航到下一页和上一页的`name`分别为`slide-left`和`slide-right`。下面以`slide-left`为例详解各个阶段的class对应的css，`slide-right`的实现类似就不再赘述了。

### 详解：slide-left的css
我们可以观察到`slide-left-enter-to`和`slide-left-leave`的状态是一样的，都是一个至少占满整个屏幕的页面。

```css
.slide-left-enter-to, .slide-left-leave {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  min-height: 100%;
}
```

`slide-left-leave-to`是当前页离开的重点，应当相对视窗左偏移100%；`slide-left-enter`是下一页进入的起点，应当相对视窗右偏移100%。进入视窗和离开视窗时，两者的透明度应该也要有所变化，在视窗中间时都是`opacity: 1`，在视窗外时都是`opacity: 0`。

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

最后，我们为进入和离开的中间状态加上过渡：

```css
.slide-left-enter-active, .slide-left-leave-active {
  transition: all .5s;
}
```

### demo: slide-left & slide-right
{% include codepen.html hash="YzPQpgW" title="transition的用法-last" %}

这个demo加入了`slide-left`和`slide-right`。用户切换页面时，监听的处理函数会根据切换的页面改变`transitionName`，从而实现从右到左或从左到右的过渡效果。

你应该注意到，我为`<transition>`包裹的元素添加了`key`属性，如果没有添加这个属性，页面的切换是没有过渡效果的，原因是：当有相同标签名的元素切换时，需要通过 key 特性设置唯一的值来标记以让 Vue 区分它们，否则 Vue 为了效率只会替换相同标签内部的内容。仅替换内容相同的内容，当然不会有过渡效果。

到这里为止，我们的`<transition>`部分算是完成了，我们进入到下一步：`transition`与`vue-router`结合的应用。

## 使用vue-router管理组件切换
在开始本节前，我希望你已经理解vue-router的基本概念[《vue router guide》](https://router.vuejs.org/zh/guide/){:target="_blank"}。我们平时称之为"页面"的，在vue-router中的概念是"路由组件"，这很好理解——真正的页面只有一个，我们切换的"页面"实际上是失活或激活组件，使用了vue-router后，这些组件的激活与失活是由路由决定的，所以我们称之为"路由组件"，后文我就直接这样称呼了。

平时开发，我们一般用vue component，将路由组件写在一个单独的`.vue`文件。为了写起来方便，在这个demo中，我会用`X-Template`的方式定义路由组件，写法是这样的：
```html
<script type="text/x-template" id="page1">
  <div style="background: #FF8A65;">
    <h2>page1</h2>
    <router-link to="/page2">下一页</router-link>
  </div>
</script>
```
官方手册有说明[《X-Template》](https://cn.vuejs.org/v2/guide/components-edge-cases.html#X-Template){:target="_blank"}。

{% include codepen.html hash="VwYWxdV" title="使用vue-router管理组件切换" %}

### 详解demo
我在`routes`中定义了两个路由，并将组件page1和page2映射到路由，使用了<router-link>实现page1和page2之间的导航。
我们重点看看如何动态改变`<transition>`的`name`属性。改变`name`的时机应当是在路由变更之前，所以我选用了`vue-router`的导航守卫`beforeEach`，这是一个全局前置守卫，只要触发导航，就会按定义的顺序执行`beforeEach`注册的代码。`beforeEach`就是使用vue-router管理路由组件切换的关键。
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

至此，我们已经实现了路由组件的动态过渡效果。但是这个demo的实现有一些不足：
1. 我是直接根据目标路由的名称变更`name`属性，但是在大型的应用中，最好还是引入路由层级的概念（不是'路由嵌套'）。
我将在下一节讲述解决这些问题的思路。
2. `transitionName`挂载到了全局`vue`实例中，生产环境是一般不这样写，我们会使用`vuex`管理`transitionName`。

## 增强过渡效果管理的可拓展性
### 路由层级
当路由的数量很多的时候，我们如何在`beforeEach`管理路由切换的过渡效果呢？如果你要处理的事情很多，最好的习惯是对事情分类、分优先级，处理规模大的路由也是同一个道理。我们可以将所有路由想象成一棵树，树的最顶端是整个应用的入口路由，`level`记为1，入口路由可以导航到第二层的路由，第二层的路由`level`记为2，以此类推。当然，现实中的路由组织结构是一个图，任何一个路由都可以导航到任意的路由，我们探讨问题时，尽量在不产生误解的情况下使用简化的模型。

![路由树](/assets/image/posts/vue-transition-and-router/router-tree.png){: .img-center}

对于某一个路由，可以用"同级路由"、"上级路由"、"下级路由"三个概念描述它所处的位置。观察上图，路由A(`level: 2`)的同级路由是B(`level: 2`)，下级路由是C、D、E(`level: 3`)，上级路由是入口路由(`level: 1`)。我们将这几个概念再延扩——**所有level相同的都是同级路由，所有level小于自身的都是上级路由，所有level大于自身的都是下级路由**，那么，F可以看作是A的下级路由。

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

`meta`是路由的[元信息](https://router.vuejs.org/zh/guide/advanced/meta.html#%E8%B7%AF%E7%94%B1%E5%85%83%E4%BF%A1%E6%81%AF){:target="_blank"}，可以通过`route.meta.level`获取路由的层级。现在，让我们改写`router.beforeEach`：
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
这是增加了路由层级之后的demo。有了路由层级，后续增加路由只需要定义level，不需要修改原有的代码，增强了路由的可拓展性。

### 使用vuex管理transition name
在大型应用中，我们的路由设计通常会使用"嵌套路由"，以提升应用界面的模块化水平和实现组件的高度复用。因为`<transition>`只对直接后代元素有效，所以在被嵌套的路由中，我们还需要嵌套一个`<transition>`元素，你可以直观地理解为这样的结构：
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
因为各个vue组件的实例是不共享数据的，所以为了让两个`<transition>`都能用上`transitionName`，`transitionName`必须要放到一个存储数据的单例——`vuex`。我们先基于上一个demo做简单的改造：

{% include codepen.html hash="NWPajjx" title="使用vuex管理transition name" %}
使用vuex后，我们只需要在state定义`transitionName`，并在vue实例中的计算属性中引用。同时，我们在mutations也定了修改`transitionName`的方法，并在`router.beforeEach`中使用。
我们为这个demo加点难度，设想一个场景：视窗顶部有一个固定的导航栏，路由组件切换的时候导航栏不需要重新渲染，只有路由组件才有过渡的效果，这时候，就需要用到嵌套路由。

### demo：嵌套路由与transitionName
