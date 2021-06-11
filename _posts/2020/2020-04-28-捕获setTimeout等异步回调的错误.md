---
layout: mypost
title: 捕获setTimeout等异步回调的错误
categories: [javascript]
---

直接捕获无法捕获到setTimeout、ajax等异步回调中的错误

```js
function test(){
  try {
		setTimeout(function(){
			throw 'error';
		}, 1);
	} catch (e) {
    // 无法捕获到
		console.log("try catch里捕获的异常" + e);
	}
}

function testAjax(){
	try {	
		$.ajax({
		  url: "xxx",
		  cache: false,
		  success: function(data){
		    throw 'error';
		  }
		});
	} catch (e) {
		// 无法捕获到
		console.log("try catch里捕获的异常" + e);
	}
}
```

# 方法一，在最上层window.onerror捕获

```js
// 注意事项：js中对象初始化慢于方法，在方法调用时最好加个延迟，让window.onerror加载完毕。
window.onerror = function(message, source, lineno, colno, error){ 
	// message：错误消息（字符串）。在HTML οnerrοr=""处理程序event（sic！）中可用。
	// source：引发错误的脚本的URL（字符串）
	// lineno：发生错误的行号（数值）
	// colno：发生错误的行的列号（数值）
	// error：错误对象（对象）
	var message = [
		'Message: ' + msg,
		'URL: ' + url,
		'Line: ' + lineNo,
		'Column: ' + columnNo,
		'Error object: ' + JSON.stringify(error)
	].join(' - ');
	console.log(message);
	// 当函数返回true时，这可以防止触发默认事件处理程序（就是浏览器控制台打印红色的异常信息）。
	return true;
};
```

# 方法二，使用Promise

```js
const wait = ms => new Promise(resolve => setTimeout(resolve, ms));

wait(10000).then(() => saySomething("10 seconds")).catch(failureCallback);
```
