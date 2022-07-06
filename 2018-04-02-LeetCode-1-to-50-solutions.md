---
layout: post # needs to be post
title: LeetCode 1 - 50 解法 (Go, Java, JavaScript, Kotlin, Python, Swift)
summary: Let's LeetCode!
featured-img: design
categories: [LeetCode]
---



## [1. Two Sum](https://leetcode.com/problems/two-sum/description/)
Given an array of integers, return indices of the two numbers such that they add up to a specific target.

You may assume that each input would have exactly one solution, and you may not use the same element twice.

Example:

```
Given nums = [2, 7, 11, 15], target = 9,

Because nums[0] + nums[1] = 2 + 7 = 9,
return [0, 1].
```

------

## [1. 两数之和](https://leetcode-cn.com/problems/two-sum/description/)
给定一个整数数列，找出其中和为特定值的那两个数。

你可以假设每个输入都只会有一种答案，同样的元素不能被重用。

示例:

```
给定 nums = [2, 7, 11, 15], target = 9

因为 nums[0] + nums[1] = 2 + 7 = 9
所以返回 [0, 1]
```

思路:

循环遍历, 这种方法时最容易想到的. 时间复杂度为 O(n2).

解法(Go):

```go
func twoSum(nums []int, target int) []int {
	
	numsLength := len(nums)
	for i := 0; i < numsLength; i++ {
		for j := i + 1; j < numsLength; j++ {
			if target == (nums[i] + nums[j]) {
				return []int{i, j}
			}
		}
	}
	
	return []int{0, 0}
}
```

其他解法:

上面的解法还可以改进, 可以先使用快速排序(时间复杂度 O(nlogn)), 然后双指针从前后往中间扫描. 但是题目要求是返回数组下标, 所以还需要额外的空间存放下标信息.

另外一种方法则是通过 Hash 的方式. 从前往后扫描一遍, 将数及下标存放到 Map 中, 再次扫描即可. 时间复杂度为 O(n).