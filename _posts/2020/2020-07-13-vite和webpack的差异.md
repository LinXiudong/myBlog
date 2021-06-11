---
layout: mypost
title: vite和webpack的差异
categories: [javascript]
---

# webpack打包过程

1. 识别入口文件
2. 通过逐层识别模块依赖。（Commonjs、amd或者es6的import，webpack都会对其进行分析。来获取代码的依赖）
3. webpack做的就是分析代码。转换代码，编译代码，输出代码
4. 最终形成打包后的代码

# webpack打包原理

1. 先逐级递归识别依赖，构建依赖图谱
2. 将代码转化成AST抽象语法树
3. 在AST阶段中去处理代码
4. 把AST抽象语法树变成浏览器可以识别的代码， 然后输出

![01](01.png)


# vite原理

当声明一个 script 标签类型为 module 时。如：
```js
<script type="module" src="/src/main.js"></script>
```

浏览器就会向服务器发起一个GET
```js
// http://localhost:3000/src/main.js请求main.js文件：

// /src/main.js:
import { createApp } from 'vue'
import App from './App.vue'
createApp(App).mount('#app')
```

浏览器请求到了main.js文件，检测到内部含有import引入的包，又会对其内部的 import 引用发起 HTTP 请求获取模块的内容文件

如：GET http://localhost:3000/@modules/vue.js

如：GET http://localhost:3000/src/App.vue

Vite 的主要功能就是通过劫持浏览器的这些请求，并在后端进行相应的处理将项目中使用的文件通过简单的分解与整合，然后再返回给浏览器，vite整个过程中没有对文件进行打包编译，所以其运行速度比原始的webpack开发编译速度快出许多！


## webpack缺点一.缓慢的服务器启动

当冷启动开发服务器时，基于打包器的方式是在提供服务前去急切地抓取和构建你的整个应用。

### vite改进

Vite 通过在一开始将应用中的模块区分为 依赖 和 源码 两类，改进了开发服务器启动时间。

依赖 大多为纯 JavaScript 并在开发时不会变动。一些较大的依赖（例如有上百个模块的组件库）处理的代价也很高。依赖也通常会以某些方式（例如 ESM 或者 CommonJS）被拆分到大量小模块中。

Vite 将会使用 esbuild 预构建依赖。Esbuild 使用 Go 编写，并且比以 JavaScript 编写的打包器预构建依赖快 10-100 倍。

源码 通常包含一些并非直接是 JavaScript 的文件，需要转换（例如 JSX，CSS 或者 Vue/Svelte 组件），时常会被编辑。同时，并不是所有的源码都需要同时被加载。（例如基于路由拆分的代码模块）。

Vite 以 原生 ESM 方式服务源码。这实际上是让浏览器接管了打包程序的部分工作：Vite 只需要在浏览器请求源码时进行转换并按需提供源码。根据情景动态导入的代码，即只在当前屏幕上实际使用时才会被处理。

## webpack缺点2.使用的是node.js去实现

### vite改进

Vite 将会使用 esbuild 预构建依赖。Esbuild 使用 Go 编写，并且比以 Node.js 编写的打包器预构建依赖快 10-100 倍。

## webpack致命缺点3.热更新效率低下

当基于打包器启动时，编辑文件后将重新构建文件本身。显然我们不应该重新构建整个包，因为这样更新速度会随着应用体积增长而直线下降。

一些打包器的开发服务器将构建内容存入内存，这样它们只需要在文件更改时使模块图的一部分失活，但它也仍需要整个重新构建并重载页面。这样代价很高，并且重新加载页面会消除应用的当前状态，所以打包器支持了动态模块热重载（HMR）：允许一个模块 “热替换” 它自己，而对页面其余部分没有影响。这大大改进了开发体验 - 然而，在实践中我们发现，即使是 HMR 更新速度也会随着应用规模的增长而显著下降。

### vite改进

在 Vite 中，HMR 是在原生 ESM 上执行的。当编辑一个文件时，Vite 只需要精确地使已编辑的模块与其最近的 HMR 边界之间的链失效（大多数时候只需要模块本身），使 HMR 更新始终快速，无论应用的大小。

Vite 同时利用 HTTP 头来加速整个页面的重新加载（再次让浏览器为我们做更多事情）：源码模块的请求会根据 304 Not Modified 进行协商缓存，而依赖模块请求则会通过 Cache-Control: max-age=31536000,immutable 进行强缓存，因此一旦被缓存它们将不需要再次请求。

## vite缺点1.生态，生态，生态不如webpack

## vite缺点2.prod环境的构建，目前用的Rollup

原因在于esbuild对于css和代码分割不是很友好


# Vite 原理 - 版本2

在过去的 Webpack、Rollup 等构建工具的时代，我们所写的代码一般都是基于 ES Module 规范，在文件之间通过 import 和 export 形成一个很大的依赖图。

这些构建工具在本地开发调试的时候，也都会提前把你的模块先打包成浏览器可读取的 js bundle，虽然有诸如路由懒加载等优化手段，但懒加载并不代表懒构建，Webpack 还是需要把你的异步路由用到的模块提前构建好。

当你的项目越来越大的时候，启动也难免变的越来越慢，甚至可能达到分钟级别。而 HMR 热更新也会达到好几秒的耗时。

Vite 则别出心裁的利用了浏览器的原生 ES Module 支持，直接在 html 文件里写诸如这样的代码：

```html
// index.html
<div id="app"></div>
<script type="module">
  import { createApp } from 'vue'
  import Main from './Main.vue'

  createApp(Main).mount('#app')
</script>
```

Vite 会在本地帮你启动一个服务器，当浏览器读取到这个 html 文件之后，会在执行到 import 的时候才去向服务端发送 Main.vue 模块的请求，Vite 此时在利用内部的一系列黑魔法，包括 Vue 的 template 解析，代码的编译等等，解析成浏览器可以执行的 js 文件返回到浏览器端。

这就保证了只有在真正使用到这个模块的时候，浏览器才会请求并且解析这个模块，最大程度的做到了按需加载。

## 依赖预编译

依赖预编译，其实是 Vite 2.0 在为用户启动开发服务器之前，先用 esbuild 把检测到的依赖预先构建了一遍。

也许你会疑惑，不是一直说好的 no-bundle 吗，怎么还是走启动时编译这条路线了？尤老师这么做当然是有理由的，我们先以导入 lodash-es 这个包为例。

当你用 import { debounce } from 'lodash' 导入一个命名函数的时候，可能你理想中的场景就是浏览器去下载只包含这个函数的文件。但其实没那么理想，debounce 函数的模块内部又依赖了很多其他函数，形成了一个依赖图。

当浏览器请求 debounce 的模块时，又会发现内部有 2 个 import，再这样延伸下去，这个函数内部竟然带来了 600 次请求，耗时会在 1s 左右。

这当然是不可接受的，于是尤老师想了个折中的办法，正好利用 Esbuild 接近无敌的构建速度，让你在没有感知的情况下在启动的时候预先帮你把 debounce 所用到的所有内部模块全部打包成一个传统的 js bundle。

Esbuild 使用 Go 编写，并且比以 JavaScript 编写的打包器预构建依赖快 10-100 倍。

![02](02.png)


在 httpServer.listen 启动开发服务器之前，会先把这个函数劫持改写，放入依赖预构建的前置步骤，

```js
// server/index.ts
const listen = httpServer.listen.bind(httpServer)
httpServer.listen = (async (port: number, ...args: any[]) => {
  try {
    await container.buildStart({})
    // 这里会进行依赖的预构建
    await runOptimize()
  } catch (e) {
    httpServer.emit('error', e)
    return
  }
  return listen(port, ...args)
}) as any
```

runOptimize：

首先会根据本次运行的入口，来扫描其中的依赖：

```js
let deps: Record<string, string>, missing: Record<string, string>
if (!newDeps) {
  ;({ deps, missing } = await scanImports(config))
}
```
scanImports 其实就是利用 Esbuild 构建时提供的钩子去扫描文件中的依赖，收集到 deps 变量里，在扫描到入口文件（比如 index.html）中依赖的模块后，形成类似这样的依赖路径数据结构：

```json
{
  "lodash-es": "node_modules/lodash-es"
}
```

之后再根据分析出来的依赖，使用 Esbuild 把它们提前打包成单文件的 bundle。

```js
const esbuildService = await ensureService()
await esbuildService.build({
  entryPoints: Object.keys(flatIdDeps),
  bundle: true,
  format: 'esm',
  external: config.optimizeDeps?.exclude,
  logLevel: 'error',
  splitting: true,
  sourcemap: true,
  outdir: cacheDir,
  treeShaking: 'ignore-annotations',
  metafile: esbuildMetaPath,
  define,
  plugins: [esbuildDepPlugin(flatIdDeps, flatIdToExports, config)]
})
```

在浏览器请求相关模块时，返回这个预构建好的模块。这样，当浏览器请求 lodash-es 中的 debounce 模块的时候，就可以保证只发生一次接口请求了。

你可以理解为，这一步和 Webpack 所做的构建一样，只不过速度快了几十倍。

在预构建这个步骤中，还会对 CommonJS 模块进行分析，方便后面需要统一处理成浏览器可以执行的 ES Module。
