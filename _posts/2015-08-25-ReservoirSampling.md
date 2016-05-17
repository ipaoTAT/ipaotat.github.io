---
layout: post
title: 等概率随机抽样方法
comments: true
category: 算法
tags: [算法,等概率随机抽样,蓄水池抽样]
---

###问题描述
从N的规模中等概率随机抽出K个样本(0 < K <= N), 即每个元素被抽取的概率都是K/N。

###正常方法
随机的问题一定用到随机数方法(random)，该方法能等概率产生一个随机数。倘若N比较小，且K／N较小，可以使用反复随机的方法：

1. 随机产生一个选中的数(int)(N×rand())，倘若该数未被选中则选中它；如果己经选中，则重新产生一个随机数。

2. 重复方法1，直到抽出K个数。

对应C++代码如下：

```C++

	＃include<iostream>
	using namespace std;
    boolean is_choosen(int *a, int n, int k)
    {
        for(int i = 0; i < n; i++)
        {
	        if(a[i] == k)return true;
        }
        return false;
    }

	int choose_k(int n, int *res, int k)
	{
	    for(int i = 0; i < k; i++)res[i] = -1;
		for(int i; i < k; i++)
		{
			int choice = rand() * k;
			while(is_choose(res, k, choice))choice = rand() * k;
			res[i++] = choice;
		}
		return 0;
	}

```

###未知N
在N为止或非常大时，上述方法将变得效率底下甚至失效。比如样本太大无法全部加载到内存。此时需要用其他方法。

先看k=1时，对第i个元素，以1／i的概率选中它，每个元素被不被选择相互独立。那么可以得到i被选中的概率为（第i个被选中而后面的都不被选中）：

```python

		Pi = 1/i * i/(i+1) * (i+1)/(i+2)\*...\*(n-1)/n = 1/n

```

可知每个元素被选中的概率都是1／n，满足随即抽样条件。代码如下

```C++

	＃inlcude<iostream>
	using namespace std;
	/*N已知,但可能很大*/
	int choose_1(int n)
	{
		int choice = -1;
		for(int i = 0; i < n; i++)
		{
			if(rand() * i < 1)
			{
				choice = i;
			}
		}
		return choice;
	}
	/*N未止*/
	int choose_1(istream &in)
	{
		int choice = -1;
		int tmp;
		for(int i = 0; cin >> tmp ; i++)
		{
			if(rand() * i < 1)
			{
				choice = tmp;
			}
		}
		return choice;
	}

```

<!-- more -->

一般情况下(K != 1)，上述方法经过少量修正也可以使用。K==1时，相当于后面被选中概率小，但一旦被选中，被替换掉的概率也较小。当有一个坑时，被替换的概率是1；当有多个坑时我们保证已经被选中的元素被等概率替换掉即可。

描述：
		1. 前K个元素以概率1被选中;
		2. 从第K+1原始开始，第i个元素以K／i的概率被选中，并等概率替换已经被选中的K个元素;
		3. 每个元素被选中的概率都是K／N。
		
证明：
		1. 前K个元素最终被选中的概率 = 以后的元素不被选中 + 以后的元素被选中但不被替换.

```python

			Pi = [1/(K+1) + K/(K+1) * (K-1)/K ] * [2/K+2 + K/(K+2) * (K-1)/K] * ... * [(N-K)/N + K/N * (K-1)/K] 
			   = K/(K+1) * (K+1)/(K+2) * ... * (N-1)/N
	           = K/N

```
	    2. 第K之后的元素被选中的概率 = 该元素被选中 * (以后的元素不被选中 + 以后的元素被选中但不被替换)

```python

		    Pi = K/i * {[(i+1-K)/(i+1) + K/(i+1) * (K-1)/K] * [(i+2-K)/(i+2) + K/(i+2) * (K-1)/K] * ... * [(N-K)/N + K/N * (K-1)/K]}
		       = K/i *  i/(i+1) * (i+1)/(i+2) * ... * (N-1)/N
		       = K/N

```

	得证。

C++代码如下：

```C++

	#include<iostream>
	using namespace std;
	int choose_k(istream &in, int k, int *res)
	{
	    int tmp;
	    /* init result set with first K input */
	    for(int i = 0 i < k && in >> tmp; i++)
	    {
		    res[i] = tmp;
	    }
		for(int i = k ; in >> tmp; i++)
		{
			int replace = rand() * i;
			/* replace */
			if(replace < k)
			{
				res[replace] = tmp;
			}
		}
	}

```

已知N和未知N只有结束条件不同。
