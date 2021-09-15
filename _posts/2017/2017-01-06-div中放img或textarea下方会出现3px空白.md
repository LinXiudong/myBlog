---
layout: mypost
title: div中放img或textarea下方会出现3px空白
categories: [css]
---

在一个 div 里放一个 img，但下方会出现一小块空白。

css

```css
div {
  display: inline-block;
  border: 1px solid black;
}
img {
  width: 300px;
  height: 300px;
}
```

html

```html
<div>
  <img src="1.jpg" />
</div>
```

![此处输入图片的描述][1]

_并非 padding、margin 等边距问题，当内外边距都为 0 时，还是会有 3px 的空白。_

- 解决一：

因为 img 是内联元素，div 是块元素，浏览器解析时会在 img 下预留高度，而且火狐是 5px，谷歌是 3px。

直接把 img 变成块元素就可以了，即给其一个 **_display:block;_** 属性。

- 解释二：

通过 google 了解到，img 是一种类似 text 的元素，在结束的时候，会在末尾加上一个空白符，所以会多 3px。

因此如果 div 里面放 textarea 同样会出现这个问题。

![此处输入图片的描述][2]

- 解决方法二：**_vertical-align:middle;_**

上面两种解决方法都对 textarea 和 img 有效。

# 知乎上尤大的解释

要理解这个问题，首先要弄明白 CSS 对于 display: inline 元素的 vertical-align 各个值的含义。vertical-align 的默认值是 baseline，这是一个西文排版才有的概念：

![03](03.png)

可以看到，出现在 baseline 下面的是 p 啊，q 啊, g 啊这些字母下面的那个尾巴。

对比一下 vertical-align 的另外两个常见值，top 和 bottom:

![04](04.png)

可以看到，baseline 和 bottom 之间有一定的距离。实际上，inline 的图片下面那一道空白正是 baseline 和 bottom 之间的这段距离。即使只有图片没有文字，只要是 inline 的图片这段空白都会存在。

到这里就比较明显了，要去掉这段空白，最直接的办法是将图片的 vertical-align 设置为其他值。如果在同一行里有文字混排的话，那应该是用 bottom 或是 middle 比较好。

另外，top 和 bottom 之间的值即为 line-height。假如把 line-height 设置为 0，那么 baseline 与 bottom 之间的距离也变为 0，那道空白也就不见了。如果没有设置 line-height，line-height 的默认值是基于 font-size 的，视渲染引擎有所不同，但一般是乘以一个系数（比如 1.2）。因此，在没有设置 line-height 的情况下把 font-size 设为 0 也可以达到同样的效果。当然，这样做的后果就是不能图文混排了。

[1]: 01.png
[2]: 02.png
