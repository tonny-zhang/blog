---
title: golang使用指针修改数据引起的血案
date: 2021-2-25 13:12:05
tags: blog
---
> ## 背影知识:
> golang 中的struct和slice为值赋值, map为引用赋值；range语句`for k, v := range val` 这里的v只申明了一次，每次迭代只会更新值

我们在平时应该会经常用到用指针把修改复杂对象(struct、slice、map)属性的需求，今天小编就把踩过的一个小坑跟大家分享下：

先上下代码:
```go
package main

import (
	"fmt"
)

// Node node
type Node struct {
	Val int
}

var nodes []Node

func initData() {
	for i := 0; i < 10; i++ {
		nodes = append(nodes, Node{
			Val: i,
		})
	}

	nodes[9].Val = 100
}
func main() {

	initData()

	var mapNodes map[int][]*Node = make(map[int][]*Node, 0)

	for _, v := range nodes {
		if _, ok := mapNodes[v.Val]; !ok {
			mapNodes[v.Val] = make([]*Node, 0)
		}
		mapNodes[v.Val] = append(mapNodes[v.Val], &v)
	}

	node := mapNodes[0][0]
	node.Val = 200

	fmt.Println(nodes)
}
```
## 输出
```
[{0} {1} {2} {3} {4} {5} {6} {7} {8} {100}]
```
## 期望输出
```
[{200} {1} {2} {3} {4} {5} {6} {7} {8} {100}]
```
可以明显看出我们更改的第一个元素的值没有生效，第一个原因是`range`造成的问题，我们不能直接使用`&v`去作为指向原始元素的地址，此时想得到原始元素的地址，必须明确找到元素，可以使用以下方案：
```
	for i, v := range nodes {
		if _, ok := mapNodes[v.Val]; !ok {
			mapNodes[v.Val] = make([]*Node, 0)
		}
		mapNodes[v.Val] = append(mapNodes[v.Val], &(nodes[i]))
	}
```
修改完成后，再看下输出即是我们想要的结果
```
[{200} {1} {2} {3} {4} {5} {6} {7} {8} {100}]
```

其实`range`语句里的v即为一个赋值操作，等价于`v := nodes[i]`，此时的赋值操作`v`已经是一个新的地址和原始数据`nodes[i]`已经不是同一个地址，故后续想通过`&v`是找不到原始数据的