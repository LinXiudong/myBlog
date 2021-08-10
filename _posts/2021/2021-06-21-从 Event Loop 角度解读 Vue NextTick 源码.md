---
layout: mypost
title: 从 Event Loop 角度解读 Vue NextTick 源码
categories: [javascript]
---

[文章来源](https://mp.weixin.qq.com/s/iHE--LdngufCExeF_7lfPw)
[文章来源](https://www.jianshu.com/p/a7550c0e164f)

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

# 另一篇文章的原理

Vue.nextTick 用于延迟执行一段代码，它接受 2 个参数（回调函数和执行回调函数的上下文环境），如果没有提供回调函数，那么将返回 promise 对象。

```js
/**
 * Defer a task to execute it asynchronously.
 */
export const nextTick = (function () {
  const callbacks = [];
  let pending = false;
  let timerFunc;

  function nextTickHandler() {
    pending = false;
    const copies = callbacks.slice(0);
    callbacks.length = 0;
    for (let i = 0; i < copies.length; i++) {
      copies[i]();
    }
  }

  // the nextTick behavior leverages the microtask queue, which can be accessed
  // via either native Promise.then or MutationObserver.
  // MutationObserver has wider support, however it is seriously bugged in
  // UIWebView in iOS >= 9.3.3 when triggered in touch event handlers. It
  // completely stops working after triggering a few times... so, if native
  // Promise is available, we will use it:
  /* istanbul ignore if */
  if (typeof Promise !== "undefined" && isNative(Promise)) {
    var p = Promise.resolve();
    var logError = (err) => {
      console.error(err);
    };
    timerFunc = () => {
      p.then(nextTickHandler).catch(logError);
      // in problematic UIWebViews, Promise.then doesn't completely break, but
      // it can get stuck in a weird state where callbacks are pushed into the
      // microtask queue but the queue isn't being flushed, until the browser
      // needs to do some other work, e.g. handle a timer. Therefore we can
      // "force" the microtask queue to be flushed by adding an empty timer.
      if (isIOS) setTimeout(noop);
    };
  } else if (
    !isIE &&
    typeof MutationObserver !== "undefined" &&
    (isNative(MutationObserver) ||
      // PhantomJS and iOS 7.x
      MutationObserver.toString() === "[object MutationObserverConstructor]")
  ) {
    // use MutationObserver where native Promise is not available,
    // e.g. PhantomJS, iOS7, Android 4.4
    var counter = 1;
    var observer = new MutationObserver(nextTickHandler);
    var textNode = document.createTextNode(String(counter));
    observer.observe(textNode, {
      characterData: true,
    });
    timerFunc = () => {
      counter = (counter + 1) % 2;
      textNode.data = String(counter);
    };
  } else {
    // fallback to setTimeout
    /* istanbul ignore next */
    timerFunc = () => {
      setTimeout(nextTickHandler, 0);
    };
  }

  return function queueNextTick(cb?: Function, ctx?: Object) {
    let _resolve;
    callbacks.push(() => {
      if (cb) {
        try {
          cb.call(ctx);
        } catch (e) {
          handleError(e, ctx, "nextTick");
        }
      } else if (_resolve) {
        _resolve(ctx);
      }
    });
    if (!pending) {
      pending = true;
      timerFunc();
    }
    if (!cb && typeof Promise !== "undefined") {
      return new Promise((resolve, reject) => {
        _resolve = resolve;
      });
    }
  };
})();
```

首先，先了解 nextTick 中定义的三个重要变量。

- callbacks
  - 用来存储所有需要执行的回调函数
- pending
  - 用来标志是否正在执行回调函数
- timerFunc
  - 用来触发执行回调函数

接下来，了解 nextTickHandler()函数。

```js
function nextTickHandler() {
  pending = false;
  const copies = callbacks.slice(0);
  callbacks.length = 0;
  for (let i = 0; i < copies.length; i++) {
    copies[i]();
  }
}
```

这个函数用来执行 callbacks 里存储的所有回调函数。

接下来是将触发方式赋值给 timerFunc。

- 先判断是否原生支持 promise，如果支持，则利用 promise 来触发执行回调函数；
- 否则，如果支持 MutationObserver，则实例化一个观察者对象，观察文本节点发生变化时，触发执行所有回调函数。
- 如果都不支持，则利用 setTimeout 设置延时为 0。

最后是 queueNextTick 函数。因为 nextTick 是一个即时函数，所以 queueNextTick 函数是返回的函数，接受用户传入的参数，用来往 callbacks 里存入回调函数。

![05](05.png)

上图是整个执行流程，关键在于 timeFunc()，该函数起到延迟执行的作用。

从上面的介绍，可以得知 timeFunc()一共有三种实现方式。

- Promise
- MutationObserver
- setTimeout

其中 Promise 和 setTimeout 很好理解，是一个异步任务，会在同步任务以及更新 DOM 的异步任务之后回调具体函数。

下面着重介绍一下 MutationObserver。

MutationObserver 是 HTML5 中的新 API，是个用来监视 DOM 变动的接口。他能监听一个 DOM 对象上发生的子节点删除、属性修改、文本内容修改等等。

调用过程很简单，但是有点不太寻常：你需要先给他绑回调：

> var mo = new MutationObserver(callback)

通过给 MutationObserver 的构造函数传入一个回调，能得到一个 MutationObserver 实例，这个回调就会在 MutationObserver 实例监听到变动时触发。

这个时候你只是给 MutationObserver 实例绑定好了回调，他具体监听哪个 DOM、监听节点删除还是监听属性修改，还没有设置。而调用他的 observer 方法就可以完成这一步:

```js
var domTarget = 你想要监听的dom节点;
mo.observe(domTarget, {
  characterData: true, //说明监听文本内容的修改。
});
```

![06](06.png)

在 nextTick 中 MutationObserver 的作用就如上图所示。在监听到 DOM 更新后，调用回调函数。

其实使用 MutationObserver 的原因就是 nextTick 想要一个异步 API，用来在当前的同步代码执行完毕后，执行我想执行的异步回调，包括 Promise 和 setTimeout 都是基于这个原因。其中深入还涉及到 microtask 等内容，暂时不理解，就不深入介绍了。
