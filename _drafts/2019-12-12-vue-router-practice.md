---
layout: default
title:  "vue-router最佳实践"
description: ""
date:   2019-12-12 09:37:00 +0800
categories: fend
---

# vue router practice

{% include codepen.html hash="YzPQpgW" title="test-embed-code" %}

## 嵌套路由
- 导航栏
- 嵌套keep-alive无法exclude的问题（提出了问题但是没有解决）

## 编程式的导航
- 用name而非path

## 路由组件传参
### 解耦合
在页面组件中使用`$route.query`会使得组件与路由耦合，解耦合的办法是将`$route.query`/`$route.params`改为通过props传参给页面组件，vue-router提供了三种方法解决这个问题，即路由组件传参的布尔模式、对象模式、函数模式，其中函数模式是最灵活的、应用最广泛的，使用函数模式可以将query和params附加到props上。
```javascript
const router = new VueRouter({
  routes: [
    { path: '/search', component: SearchUser, props: (route) => ({ query: route.query, params1: route.params.params1 }) }
  ]
})
```

### 传参的两种方式——params还是query？
乍看之下，params和query的功能几乎一致，不过两者还是有一个重要的差别——query方式传参时参数会以查询参数的形式附加到url上，参数格式与GET请求的参数一致；而params方式传递的参数则不会以查询参数的形式出现，如果是动态路由匹配传参，参数会出现在url的路径当中。
```javascript
  // 
    const router = new VueRouter({
      routes: [
        // 动态路径参数 以冒号开头
        { path: '/user/:id', component: User }
      ]
    })
  this.$router.push({
    name: 'page',
    query: {
      q1: {}, // × 不允许传递对象
      q2: () => {} // 可以这样传递，但是刷新后无效
    }
  })
```


### params还是query？
乍看之下，params和query的功能几乎一致，不过两者还是有一个重要的差别——query方式传参时参数会附加到url上，参数格式与GET请求的参数一致；而params方式传递的参数则不会出现在url上（除非是动态路由匹配的方式传参）。这个差别既决定了开发者的使用偏好，也极大地影响了用户的体验。
1. 一般而言，开发者倾向于不把参数暴露在url（除非是明显的“查询”类场景），而且query方式不支持以对象作为参数的值，也不绝对地支持function作为参数的值。
```javascript
  // 例3.1
  this.$router.push({
    name: 'page',
    query: {
      q1: {}, // × 不允许传递对象
      q2: () => {} // 可以这样传递，但是刷新后无效
    }
  })
```
请看例3.1。q1的值是对象，我们马上可以想到可以将对象序列化成json格式，在接收方再将json字符串转为对象字面量。
```javascript
  // 发起方
  this.$router.push({
    name: 'page',
    query: {
      q1: JSON.stringify({}), // × 不允许传递对象
      q2: () => {} // 可以这样传递，但是刷新后无效
    }
  })
  // 接收方
  const q1 = JSON.parse(this.$route.query.q1)
```
在看q2之前，插入一个query传参编解码的知识。query将参数附加到url之前会对其编码（可以暂且认为是`encodeURIComponent()`函数），然后在定位到目标路由后将url的参数解码，最后可以通过`this.$route.query`取得这些值。了解了这点，就很好理解为什么不能传递值为对象的参数了，似乎也意识到传递函数也是有问题的——函数编码后再解码，就成了一个字符串了。
然而事实并非如此，我们可以这样用：
```javascript
  // 发起方
  this.$router.push({
    name: 'page',
    query: {
      q1: JSON.stringify({}), // × 不允许传递对象
      q2: () => {} // 可以这样传递，但是刷新后无效
    }
  })
  // 接收方
  const q2 = this.$route.query.q2
  q2() // 可以执行！
```
其中的缘由我也没有搞明白，但是可以确定并没有使用`evel()`或`new Function()`的方式将字符串“恢复”为函数。我猜测，作者可能同时使用了与params相同的机制将参数直接传递过去了，我这样猜测是有理由的——刷新页面之后，会发现`this.$route.query.q2`取到的值又变成了字符串（总算正常了）。鉴于这一点，我们可以认为使用query的方式传递函数是不可靠的，同理，传递其他非字符串的值也是不可靠的（包括Number、Boolean、null、未序列化的对象等等）。

2. 如果用户刷新了接收参数的页面，应用会被销毁重新加载，params传递的参数也会随之销毁，结果页面因为缺少传递的参数而变得不可用。不过通过query方式传递的参数却保留了下来，它们就url，这一点是query方式优于params方式的地方，query方式能够尽量地挽救用户的体验感。

现在，我们已经了解query和params的区别，正是这些区别造就了两者不同的适用场景。
1. query方式相对params方式更加灵活。url的查询参数是允许缺失的，但是路径却要求完整（不完整会导致无法匹配路由），除非参数简单、数量少、且都是必须的，否则我不使用动态路由的方式。
2. 考虑到用户体验，也推荐尽量使用query的方式。
3. 如果使用query也无法保证用户的体验，而且需要传递对象、布尔值、函数等复杂的数据，那就使用params吧。

### 路由组件处理传递参数
有时候，我们在跳转到下一个页面执行完一系列交互后，需要返回上一页，或者执行上一页设置的回调函数。一个常见的场景是提交表单时登录态缺失，这时候必须得先跳转到登录页，待用户登录成功后再返回到提交表单的页面（或直接执行提交表单的动作）。一个比较好的方法是访问登录页的路由时将一些参数/函数传递过去，待登录完成后再取用参数或执行函数。

![路由组件通信的时序图](/assets/image/posts/vue-router-practice/route-component-query.jpg){: .img-center}

![登录组件处理传递参数的流程图](/assets/image/posts/vue-router-practice/login-hanlder.jpg){: .img-center}

上方是路由组件通信的时序图和登录组件处理传递参数的流程图。接着看看如何用代码实现。我们先为login组件设置一个`post()`方法，这个方法执行的时机是登录完成后，所以post方法应当包含处理传递参数的所有流程。值得注意的是执行`callback()`前我做了类型检测`typeof callback === 'function'`，我这样做是考虑到两点：第一点自然是调用登录的组件可能没有传递`callback`参数或传递的参数不是一个函数；第二点是用户可能在登录页面刷新了，或者在登录页点了注册等其他不可控的行为，由上一节可知，`callback`会变成字符串，也就无法作为函数去执行了。
```javascript
  // login组件
  export default {
      methods: {
        post() {
          // 这些流程请参照登录组件处理传递参数的图例
          const { callback, backName, ...query } = this.$route.query
          if (typeof callback === 'function') {
            callBack()
          } else if (backName) {
            this.$router.replace({
              name: backName,
              query,
            })
          } else {
            this.$router.replace('/')
          }
        },
        login() {
          this.post(loginUrl).then(res => {
            if (res === loginSuccess) {
              // 登录成功则执行post()方法
              this.post()
            }
          })
        }
      }
  }
  
  // 表单组件A
  export default {
      methods: {
        // 后端返回未登录时执行该方法
        handleNotLogin() {
          this.$route.push({
            name: 'login',
            query: {
              callback: () => { // 这里的callback是执行提交表单
                this.post(this.formData).then(res => {
                  if (res.isSuccess) {
                    this.$router.push({ name: 'result' })
                  }
                })
              },
              backName: 'A', // 设置返回的路由，你也可以设置其他的
              query: { // 设置返回路由的参数
                data1: '你想要加上的数据',
                ...this.$route.query, // 最好把这个路由的query参数也带上，重新定向到该路由时就能还原url
              }
            },
          })
        }
      }
  }
```
经过这样的设置，表单组件A可以在登录完成后马上提交表单，并且在提交成功后跳转到提交结果页，这种情形下用户的体验是最好的。假如很不幸用户刷新了页面，也可以通过`backName`参数回到指定的页面。啰嗦一句，这种情况下用户返回表单页没有多大的意义，除非你用sessionStorage/localStorage缓存了数据，如果没有缓存，还是回到表单页的上一级吧。

## 导航守卫
vue-router的路由导航守卫特别多，官网有所有守卫的介绍和[完整的导航解析流程](https://router.vuejs.org/zh/guide/advanced/navigation-guards.html#%E7%BB%84%E4%BB%B6%E5%86%85%E7%9A%84%E5%AE%88%E5%8D%AB)，我就不介绍了。
我打算将重点放在几个常见的路由，讲述与它们有关的一些实践经验。

还是补充一下吧。列个表区分下。

### global守卫
1. 全局前置守卫`beforeEach`
当一个导航触发时，全局前置守卫按照创建顺序调用。也就是说，我们可以在引入vue-router前后为其创建多个前置守卫，这些前置守卫在每次路由发生变化的时候都会被调用。
一个常见的应用就是路由变化（页面切换）的动效，页面从上一级切换到下一级时，我们希望是从左到右的，而页面从下一级返回到上一级时，我们希望页面是从右到左的（这就好比是）
```html
<template>
    <transition :name="name" :mode="mode">
        <router-view />
    </transition>
</template>
```

```javascript
const router = new VueRouter({ ... })

router.beforeEach((to, from, next) => {
    if (!from || !from.path || !from.meta.level) {
      store.commit('pageTranSet', {
        transitionType: 'fade',
        transitionMode: 'out-in',
      });
    } else {
      let transitionMode = null;
      let transitionType = '';
      const _from = from.meta.level;
      const _to = to.meta.level;

      if (_from < _to) {
        transitionType = 'slidel';
      } else if (_from > _to) {
        transitionType = 'slider';
      } else {
        transitionType = 'fade';
        transitionMode = 'out-in';
      }
      store.commit('pageTranSet', {transitionType, transitionMode});
    }
    next();
})
```

	- 与过度动效结合

- beforeResolve
- afterEach

### route守卫

- beforeEnter

### component守卫

- beforeRouteEnter

	- next可以传递一个回调函数
	- 与keep-alive结合

- beforeRouteUpdate

	- 同一路由更新的时候用到
	- 在此钩子导航还未被确认，所以通过query和params传递的props还未更新（等于from.query/from.params），只能从to.query/to.params取用。

- beforeRouteLeave

	- 与keep-alive结合

### 各个守卫的解析流程 + activated + deactivated 的时序

## 过度动效

### beforeEach

### watch $route

## 数据获取

### 导航完成后 created...

### 导航完成前 beforeRouteEnter / beforeRouteUpdate

## 滚动行为

### scrollBehavior

## 路由懒加载

## History模式

### hash的缺点

### history模式后端的配置和前端的注意点

- 匹配不到路由时不能返回404，要返回index的内容
- 前端要解析所有的路由，不能匹配是展示404

