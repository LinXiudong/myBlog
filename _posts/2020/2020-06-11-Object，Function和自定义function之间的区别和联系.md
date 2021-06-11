---
layout: mypost
title: Object，Function和自定义function之间的区别和联系
categories: [javascript]
---

Function 被称为构造器。其实也就是一个函数。js中万物皆函数。js中所有的对象都是由Function创造出来的。包括自定义对象，内置对象，以及它本身。 

Function内部的原型关系如下图： 

![01](01.png)

函数Function与其他的函数不同，他的prototype和 _ proto _ 都指向Function的原型。 

一定要注意prototype与 _ proto _ 两者的区别。 prototype开发者可以用方法直接访问 ，而 _ proto _ 代表的是实例对象内部的内部属性。函数Function也是它本身创造出来的，所以 他的 _ proto _ 指向它自己的原型。 

怎么理解 _ proto _ 和prototype呢? 可以这样理解：

prototype: 当创建一个构造函数后，这个构造函数会自动创建一个prototype属性，这个属性是个指针，指向一个对象。这个对象称为自己创建的构造函数的原型对象。

_ proto _ :当已经创建好了构造函数。当用户创建了这个构造函数的实例后（new出来的 ），这个新实例的下也会自动添加一个属性 _ proto _。称为内部属性。 

ES5中用[[prototype]]表示， 一般用户没有办法访问。但 Firefox、Safari、Chrome、以及IE9以上可以用Object.getPrototypeOf(实例)来访问实例的这个属性。其实这个属性指向的也是这个实例的构造函数的原型对象。

```js
console.log(Function.prototype); //--> function(){}
console.log(Object.getPrototypeOf(Function); //--> function(){}
console.log(Function.prototype===Object.getPrototypeOf(Function)); //--> true
console.log(Object.getPrototype(Function.prototype)); //--> Object{};
```

其实可以从console中看出Function的原型对象也是一个空函数，但这个空函数的 _ proto _又指向Object.prototype。 

接下来看这张图： 

![02](02.png)

从这张图的虚线中可以看出 

- Object.prototype中的 _ proto _内部属性指向的是空对象Null; 
- function Object中的 _ proto _内部属性指向的是Function.prototype 

测试代码如下：

```js
console.log(Object.prototype); //-->  Object{};
console.log(Object.getPrototypeOf(Function.prototype)); //--> Object{}
console.log(Object.prototype===Object.getPrototypeOf(Function.prototype)); //-->true;
console.log(Object.getPrototypeOf(Object.prototype)); // -->null;
```

这幅图基本上已经把Function和Object之间的关系理清楚了。在这里在做一下总结，之后在进一步分析。 

不管是函数Function，Object，还是他们的原型对象。其实都是围绕Object.prototype这个原型对象展开的。

- 首先： js中先创建的是Object.prototype这个原型对象。 
- 然后：在这个原型对象的基础之上创建了 Function.prototype这个原型对象。 
- 其次：通过这个原型对象 创建出来Function这个函数。 
- 最后: 又通过Function这个函数创建出来之后，Object（）这个对象。

其实在这里有一个问题困扰我，我可以使用typeof这个属性来判断一个函数到底是object还是function类型。但是这两者究竟又什么区别，如果说objec他就是function,那么为什么还要区分这两者呢》？

接下来我在添加一个Object实例 

![03](03.png)

接下来在增加用户自定义函数 

![04](04.png)

总结

![05](05.png)