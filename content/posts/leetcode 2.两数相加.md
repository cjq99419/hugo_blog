---
title: "leetcode 2.两数相加"
subtitle: ""
date: 2021-02-01T18:00:46+08:00
lastmod: 2021-02-01T18:00:46+08:00
draft: false
author: "nanko"
authorLink: ""
show_in_homepage: false
description: "leetcode 2.两数相加"

tags: [
    "golang",
	"leetcode",
	"algorithm",
]
categories: [
    "leetcode",
]

hiddenFromHomePage: false
hiddenFromSearch: false

featuredImage: ""
featuredImagePreview: ""

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: ""
---

## leetcode 2.两数相加

![截屏2021-02-01 下午6.03.10](https://tva1.sinaimg.cn/large/008eGmZEly1gn86w349j7j315q0luwi4.jpg)

需要对两个逆序的非空数字进行求和，重点在于两个整数长度不固定。

由于是逆序存储，所以实际上链表第一位并不对其，比如32 + 234，链表的