---
layout: post
title: 动态规划经典问题（一）
comments: true
category: 算法
tags: [算法,动态规划]
---

##最大上升子序列（Longest Increased Sequence）
**问题描述：**
设L=<a1,a2,…,an>是n个不同的实数的序列，L的递增子序列是这样一个子序列Lin=<aK1,ak2,…,akm>，其中k1<k2<…<km且aK1<ak2<…<akm。求最大的m值。如序列{1,4,3,5,6,9,7,8,10}，最长上升子序列为{1,3,5,6,7,8,10}或{1,4,5,6,7,8,10}，长度为7。
**DP思想：**
该题可以用DP的思想：找到DP方程。
假设序列为｛An｝，考虑DP[i]代表前i个元素中，包含ai的LIS，则：
						
		  / 1;               a[i] < a[i-1]
	DP[i]=|	
		  \ DP[i-1] + 1;     a[i] >= a[i-1]
那么,整个序列的LIS就等于max{DP[i]}, i=0,1,2....n。
**代码：**
由此可以写出代码：

```C++

	#include<iostream>
	using namespace std;
	template<int N>
	int getLIS(int &a[N])
	{
		int dp[N];
		dp[0] = 1; 
		for(int i = 1; i < N ; i++)
		{
			if(a[i] >= a[i-1])dp[i] = dp[i-1] + 1;
			else dp[i] = 1;
		}
		int res = 0;
		for(int i = 0; i < N; i++)
		{
			if(dp[i] > res)res = dp[i];
		}
		return res;
	}

```

根据递推式，在运算过程中DP[i]只依赖于DP[i-1]，因此可以优化上述代码：

```C++

	#include<iostream>
	using namespace std;
	template<int N>
	int getLIS(int &a[N])
	{
		int res = 0;
		int dp = 1;
		for(int i = 1; i < N ; i++)
		{
			if(a[i] >= a[i-1])dp = dp + 1;
			else {
				if(dp > res) res = dp;
				dp = 1;
			}
		}
		return res;
	}

```

**O(nlogn)算法**
##最长公共子序列
##最长公共子串
##背包问题

未完...
