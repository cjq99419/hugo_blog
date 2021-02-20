---
title: "Leetcode 1.两数之和"
subtitle: ""
date: 2021-01-06T20:31:01+08:00
lastmod: 2021-01-06T20:31:01+08:00
draft: false
author: "nanko"
authorLink: ""
show_in_homepage: false
description: "Leetcode 1.两数之和"

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

## leetcode 1.两数之和

使用map存储已经遍历过的值，key为target - num，即后续需要配对的值，value为值下标。

```go
func twoSum(nums []int, target int) []int {
	// key: target - num  value: index
	ano := make(map[int]int)
	for idx, v := range nums {
		if _, ok := ano[v]; ok {
			return []int{ano[v], idx}
		} else {
			ano[target-v] = idx
		}
	}
	return []int{0, 0}
}
```

