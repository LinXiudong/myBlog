---
layout: mypost
title: 从 Event Loop 角度解读 Vue NextTick 源码
categories: [javascript]
---

[文章来源](https://mp.weixin.qq.com/s/iHE--LdngufCExeF_7lfPw)

# 什么是 event loop

![01](01.gif)

1. 先执行同步阻塞任务，同步任务会等待上一个执行完毕以后执行下一个，当同步任务执行完毕，再执行异步任务，遇到异步任务会将异步任务的回调函数注册在异步任务队列里。注意，如果主线程上没有同步任务会直接调用异步任务的微任务。
2. 执行宏任务，遇到微任务将都添加到微任务队列里。
3. 开始执行微任务队列，当宏任务执行完后执行微任务队列，直到微任务队列全部执行完，微任务队列为空。
4. 执行宏任务，如果在执行宏任务期间有微任务，将微任务添加到微任务队列里，执行完宏任务之后执行微任务，直到微任务队列全部执行完。
5. 继续执行宏任务队列。

重复 2, 3, 4，5……直到宏微任务为空。

# $nextTick 的实现原理

从字面意思理解，next 下一个，tick 滴答（钟表）来源于定时器的周期性中断（输出脉冲），一次中断表示一个 tick，也被称做一个“时钟滴答”，nextTick 顾名思义就是下一个时钟滴答。看源码，在 Vue 2.x 版本中，nextTick 在 src\core\util 中的一个单独的文件 next-tick.js ，可见 nextTick 的重要性，虽然短短 200 多行，尤大却单独创建一个文件去维护。

接下来我们来看整个文件。

1. 声明了三个全局变量，callbacks: [] ，pending: Boolean，timerFunc: undefined。
2. 声明了一个函数 flushCallbacks。
3. 一堆 **if，else if **判断。
4. 抛出了一个函数 nextTick。

## nextTick 函数

![02](02.png)

1. 声明一个局部变量 \_resolve 。
2. 把所有回调函数压进 callbacks 中，以栈的形式的存储所有 callback。
3. 当 pending 为 false 时，执行 timerFunc 函数。
4. 当没有 callback 的时候，返回一个 Promise 的调用方式，可以用 .then 接收。

## timerFunc 函数

我们开始说了，timerFunc 为全局变量，现在调用 timerFunc ，timerFunc 是什么时候被赋值为一个函数，并且函数里执行代码又是什么？

![03](03.png)

我们看到，这段判断代码总共有四个分支，四个分支里对 timerFunc 有不同的赋值，我们先来看第一个分支。

- Promise 分支

```js
if (typeof Promise !== "undefined" && isNative(Promise)) {
  const p = Promise.resolve();
  timerFunc = () => {
    p.then(flushCallbacks);
    // In problematic UIWebViews, Promise.then doesn't completely break, but
    // it can get stuck in a weird state where callbacks are pushed into the
    // microtask queue but the queue isn't being flushed, until the browser
    // needs to do some other work, e.g. handle a timer. Therefore we can
    // "force" the microtask queue to be flushed by adding an empty timer.
    if (isIOS) setTimeout(noop);
  };
  isUsingMicroTask = true;
}
```

1. 判断环境是否支持 Promise 并且 Promise 是否为原生。
2. 使用 Promise 异步调用 flushCallbacks 函数。
3. 当执行环境是 iPhone 等，使用 setTimeout 异步调用 noop ，iOS 中在一些异常的 webview 中，promise 结束后任务队列并没有刷新所以强制执行 setTimeout 刷新任务队列。

- MutationObserver 分支

```js
else if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  // PhantomJS and iOS 7.x
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
  // Use MutationObserver where native Promise is not available,
  // e.g. PhantomJS, iOS7, Android 4.4
  // (#6466 MutationObserver is unreliable in IE11)
  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  observer.observe(textNode, {
    characterData: true
  })
  timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
  }
  isUsingMicroTask = true
}
```

1. 对非 IE 浏览器和是否可以使用 HTML5 新特性 MutationObserver 进行判断。
2. 实例一个 MutationObserver 对象，这个对象主要是对浏览器 DOM 变化进行监听，当实例化 MutationObserver 对象并且执行对象 observe，设置 DOM 节点发生改变时自动触发回调。
3. 把 timerFunc 赋值为一个改变 DOM 节点的方法，当 DOM 节点发生改变，触发 flushCallbacks 。（这里其实就是想用利用 MutationObserver 的特性进行异步操作）

- setImmediate 分支

```js
else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  // Fallback to setImmediate.
  // Technically it leverages the (macro) task queue,
  // but it is still a better choice than setTimeout.
  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
}
```

1. 判断 setImmediate 是否存在，setImmediate 是高版本 IE （IE10+） 和 edge 才支持的。
2. 如果存在，传入 flushCallbacks 执行 setImmediate 。

- setTimeout 分支

```js
else {
  // Fallback to setTimeout.
  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}
```

1. 当以上所有分支异步 api 都不支持的时候，使用 macro task （宏任务）的 setTimeout 执行 flushCallbacks 。

**执行降级**

我们可以发现，给 timerFunc 赋值是一个降级的过程。为什么呢，因为 Vue 在执行的过程中，执行环境不同，所以要适配环境。

![04](04.png)

这张图便于我们更清晰的了解到降级的过程。

- flushCallbacks 函数

```js
function flushCallbacks() {
  pending = false;
  const copies = callbacks.slice(0);
  callbacks.length = 0;
  for (let i = 0; i < copies.length; i++) {
    copies[i]();
  }
}
```

循环遍历，按照 队列 数据结构 “先进先出” 的原则，逐一执行所有 callback 。

# 总结

到这里就全部讲完了，nextTick 的原理就是利用 Event loop 事件线程去异步重新渲染，分支判断首要选择 Promise 的原因是当同步 JS 代码执行完毕，执行栈清空会首先查看 micro task （微任务）队列是否为空，不为空首先执行微任务。在我们 DOM 依赖数据发生变化的时候，会异步重新渲染 DOM ，但是比如像 echarts ，canvas……这些 Vue 无法在初始状态下收集依赖的 DOM ，我们就需要手动执行 nextTick 方法使其重新渲染。
