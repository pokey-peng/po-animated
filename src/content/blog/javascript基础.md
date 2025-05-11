---
title: JavaScript基础
description: No description
pubDate: 2025-04-22T15:44:45.126Z
---
# JavaScript基础

## 字符串

标签函数

```javascript
let a = 6
let b = 9;

function simpleTag(strings, aVal, bVal, sumVal) {
	console.log(strings)
	console.log(aVal)
 	console.log(bVal)
	console.log(sumVal)
	return 'footer'
}
let tagRes = simpleTag`${a}+${b}=${a+b}`
// ["","+","=",""]
// 6,
// 9,
// 15
console.log(tagRes) // footer
```

## Symbol.species

TODO 待解决

## 位操作符

1. 按位非 ~
2. 按位与 &
3. 按位或 |
4. 按位异或 ^
5. 左移 <<
6. 有符号右移 >>
7. 无符号右移 >>>

## 布尔操作符
