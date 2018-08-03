---
layout: post
title: ' Array.prototype.reduce'
date: 2018-07-28
author: 
# cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: reduce, Array
---


# 理解Array.prototype.reduce

## 语法

```js
arr.reduce(callback[, initialValue])
```

`Array.prototype.reduce()` 相当于一个加法器，用于把数组的所有元素合并为一个元素，合并的具体方式由传入的回调函数 `callback` 指定。

`initialValue` 是加法器的初始值。

`callback` 的函数签名有严格规定，具体如下：

```js
/*
* @param {any} accumulator - 上一次循环的处理结果或 `initialValue`。
* @param {any} currentValue - 当前循环正在处理的元素
* @param {number} currentIndex - 当前循环正在处理元素的索引值
* @param {Array} array - 正在处理的数组
* @return {any} - 回调函数返回的数值，会当作下一次循环的 accumulator
*/
function callback(accumulator, currentValue, currentIndex, array){
    // ...
}
```

其中，一般只需要传入前两个参数 `accumulator` 和 `currentValue` 即可。

## 例子

```js
var arr = [1, 2, 3, 4]
var result = arr.reduce(function(a, b){
    return a + b
})
console.log(result) // 10
```

翻译成普通 JavaScript 代码：

```js
var arr = [1, 2, 3, 4]
var result = 0
for (var i = 0; i < arr.length; i++) {
    var a = result
    var b = arr[i]
    result = a + b
}
console.log(result) // 10
```


#### REF

 ![Array-reduce](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/Reduce)
