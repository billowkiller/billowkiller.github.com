---
layout: post
title: "Some Code Snippets"
date: 2014-05-01 03:18
comments: true
categories: algorithm
tags: [algorithm, snippet]
---

1. **在一个二叉树中查找值为x的结点，并打印该结点所有祖先结点的算法，也就是根节点到x的路径。**

	思路1： 利用**后序遍历**的非递归算法，在访问结点时增加一个判断，若该结点的值等于x，则打印栈中保持的路径，再中断算法。
	
	思路2： 采用**前序遍历**的递归算法，在典型的便利算法的参数表中增加了x、path[n]等参数。
	
	    template<class T>
	    int Find_Print(BinTreeNode<T*& BT, T x, T path[], int level, int &count) {
			if(BT != NULL) {
				level++;
				path[level] = BT->data;
				if (BT->data==x) { count=level; return 1; }
				if( Find_Print(BT->left, x, path, level, count) ) return 1;
				return Find_Print(BT->right, x, path, level, count);
			} else 
				return 0;
		}

<!--more-->

2. **判断二叉树是不是平衡**

		bool IsBalanced(BinaryTreeNode* pRoot)
		{
		    if(pRoot == NULL)
		        return true;
		 
		    int left = TreeDepth(pRoot->m_pLeft);
		    int right = TreeDepth(pRoot->m_pRight);
		    int diff = left - right;
		    if(diff > 1 || diff < -1)
		        return false;
		 
		    return IsBalanced(pRoot->m_pLeft) && IsBalanced(pRoot->m_pRight);
		}
	
	一个节点会被重复遍历多次，这种思路的时间效率不高。用**后序遍历**的方式遍历二叉树的每一个结点，在遍历到一个结点之前我们已经遍历了它的左右子树。只要在遍历每个结点的时候记录它的深度，我们就可以一边遍历一边判断每个结点是不是平衡的。
	
		bool IsBalanced(BinaryTreeNode* pRoot, int* pDepth)
		{
		    if(pRoot == NULL)
		    {
		        *pDepth = 0;
		        return true;
		    }
		 
		    int left, right;
		    if(IsBalanced(pRoot->m_pLeft, &left)
		        && IsBalanced(pRoot->m_pRight, &right))
		    {
		        int diff = left - right;
		        if(diff <= 1 && diff >= -1)
		        {
		            *pDepth = 1 + (left > right ? left : right);
		            return true;
		        }
		    }
		    return false;
		}
		bool IsBalanced(BinaryTreeNode* pRoot)
		{
		    int depth = 0;
		    return IsBalanced(pRoot, &depth);
		}

3. Reverse Bits

		 // special case: only works for 4 bytes (32 bits).
	
	    uint reverseMask(uint x) 
		{
	    	assert(sizeof(x) == 4);
	    	x= ((x & 0x55555555) << 1) | ((x & 0xAAAAAAAA) >> 1);
	    	x = ((x & 0x33333333) << 2) | ((x & 0xCCCCCCCC) >> 2);
	    	x = ((x & 0x0F0F0F0F) << 4) | ((x & 0xF0F0F0F0) >> 4);
	    	x = ((x & 0x00FF00FF) << 8) | ((x & 0xFF00FF00) >> 8);
	    	x= ((x & 0x0000FFFF) << 16) | ((x & 0xFFFF0000) >> 16);
	    	return x;
	    }

4. Find the k-th Smallest Element in the Union of Two Sorted Arrays

		int findKthSmallest(int A[], int m, int B[], int n, int k) {
		  assert(m >= 0); assert(n >= 0); assert(k > 0); assert(k <= m+n);
		  
		  int i = (int)((double)m / (m+n) * (k-1));
		  int j = (k-1) - i;
		 
		  assert(i >= 0); assert(j >= 0); assert(i <= m); assert(j <= n);
		  // invariant: i + j = k-1
		  // Note: A[-1] = -INF and A[m] = +INF to maintain invariant
		  int Ai_1 = ((i == 0) ? INT_MIN : A[i-1]);
		  int Bj_1 = ((j == 0) ? INT_MIN : B[j-1]);
		  int Ai   = ((i == m) ? INT_MAX : A[i]);
		  int Bj   = ((j == n) ? INT_MAX : B[j]);
		 
		  if (Bj_1 < Ai && Ai < Bj)
		    return Ai;
		  else if (Ai_1 < Bj && Bj < Ai)
		    return Bj;
		 
		  assert((Ai > Bj && Ai_1 > Bj) || 
		         (Ai < Bj && Ai < Bj_1));
		 
		  // if none of the cases above, then it is either:
		  if (Ai < Bj)
		    // exclude Ai and below portion
		    // exclude Bj and above portion
		    return findKthSmallest(A+i+1, m-i-1, B, j, k-i-1);
		  else /* Bj < Ai */
		    // exclude Ai and above portion
		    // exclude Bj and below portion
		    return findKthSmallest(A, i, B+j+1, n-j-1, k-j-1);
		}

5. 最长等差数列分析

	原题
	
	给定未排序的数组，请给出方法找到最长的等差数列。
	
	分析
	
	题目描述比较简单，但是有一个问题我们需要首先搞清楚：等差数列中的数字，是否要和原始数组中的顺序一致。题目中，并没有说明，这个就需要大家在面试的过程中和面试官进行交流。我们在这里对两种情况都进行讨论
	
	保证数字的顺序
	
	等差数列是要求相邻两个元素之间的差是相同的。那我们可以记录下来数组中任意两个数的差，并且记录下来。对于数组A，记录A[j]-A[i]，其中 i
	
		-1=>(0,1)(1,2)
		
		1=>(2,3)(4,5)
		
		3=>(3,4)

	上面已经排好序，对于第一个，找到等差数列0,1,2对应数字诶5,4,3.第二个，3和4位置没有连起来，不够成等差数列。方法平均时间复杂度为O(n^2),空间复杂度为O(n^2).
	
	无需保证数字的顺序
	
	不需要保证数字的顺序与原来数组一致，如何找到最长的等差数列呢？原来的数组是无序的，我们先对数组进行排序，最终的一定是排序之后序列的子序列。然后，我们采用动态规划的方法解决这个问题。我们假设dp[i][j]表示以A[i]A[j]开始的数列的长度，dp[i][j]如何表示呢？dp[i][j]=dp[j][k]+1，当 A[j]-A[i]=A[k]-A[j],及A[k]+A[i]=2A[j]。根据dp[i][j]的定义，我们知道dp[x][n-1]=2，也就是 最后一列是2，数列只有A[x]和A[n-1]两个元素。首先，j从n-2，开始向前遍历，对于每一个，找到i和k，满足 A[k]+A[i]=2A[j]，则有dp[i][j]=dp[j][k]+1，若没有，则dp[i][j]就为2.
	
	这里找i和k，有一个小技巧，如下：初始i=j-1,k=j+1，然后分别向两边遍历，如果A[k]+A[i]2*A[j]则i--。大家还是参考代码吧：

	![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/01YhyLG4uM7B_zps61cd517c.jpg)

6. 构造最大数分析

	[http://www.ituring.com.cn/article/50249](http://www.ituring.com.cn/article/50249)

7. Given a sorted array of integers, find the starting and ending position of a given target value.

	    int searchLow(int A[], int start, int end, int target) {
	        while(start < end) {
	            int mid = start + (end-start)/2;
	            if(A[mid] >= target) end = mid-1;
	            else start = mid+1;
	        }
	        if(A[start] == target) return start;
	        else return start+1;
	    }
	    int searchHigh(int A[], int start, int end, int target) {
	        while(start < end) {
	            int mid = start + (end-start)/2;
	            if(A[mid] <= target) start = mid+1;
	            else end = mid-1;
	        }
	        if(A[start] == target) return start;
	        else return start-1;
	    }

8. 树的最长距离

		
		struct NODE
		{
		    NODE *pLeft;
		    NODE *pRight;
		};
		 
		struct RESULT
		{
		    int nMaxDistance;
		    int nMaxDepth;
		};
		 
		RESULT GetMaximumDistance(NODE* root)
		{
		    if (!root)
		    {
		        RESULT empty = { 0, -1 };   // trick: nMaxDepth is -1 and then caller will plus 1 to balance it as zero.
		        return empty;
		    }
		 
		    RESULT lhs = GetMaximumDistance(root->pLeft);
		    RESULT rhs = GetMaximumDistance(root->pRight);
		 
		    RESULT result;
		    result.nMaxDepth = max(lhs.nMaxDepth + 1, rhs.nMaxDepth + 1);
		    result.nMaxDistance = max(max(lhs.nMaxDistance, rhs.nMaxDistance), lhs.nMaxDepth + rhs.nMaxDepth + 2);
		    return result;
		}

9. 分治法求数组最小值和最大值

		#--coding: utf8 --
		def find_max_and_min(A, p, r):
		    if r - p <= 1:
		        if A[p] < A[r]:
		            return (A[p], A[r])
		        else:
		            return (A[r], A[p])
		    # 分解
		    q = p + (r - p) / 2
		    (lmin, lmax) = find_max_and_min(A, p, q)
		    (rmin, rmax) = find_max_and_min(A, q + 1, r)
		    # 合并
		    return (min(lmin, rmin), max(lmax, rmax))
		 
		if __name__ == "__main__":
		    # A = [1]
		    # A = [1, 2]
		    A = [7, 2, 2, 4, 5, 15, 3, 5, 8, 10, 11]
		    print(find_max_and_min(A, 0, len(A) - 1))

	扩展问题，找第二大数

		#--coding: utf8 --
		def find_second_max(A, p, r):
		    if r - p == 0:
		        return (A[p], None)
		    elif r - p == 1:
		        if A[p] > A[r]:
		            return (A[p], A[r])
		        else:
		            return (A[r], A[p])
		    # 分解
		    q = p + (r - p) / 2
		    (lfmax, lsmax) = find_second_max(A, p, q)
		    (rfmax, rsmax) = find_second_max(A, q + 1, r)
		    # 合并
		    if lfmax > rfmax:
		        fmax = lfmax
		        smax = max(lsmax, rfmax)
		    else:
		        fmax = rfmax
		        smax = max(rsmax, lfmax)
		    return (fmax, smax)
		 
		if __name__ == "__main__":
		    # A = [1]
		    # A = [1, 2]
		    A = [7, 2, 2, 4, 5, 15, 3, 5, 8, 10, 11]
		    print(find_second_max(A, 0, len(A) - 1))	

10. n个色子点数和的概率。

	![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/p1_zps7700f0a2.png) 

11. 不用加减乘除做加法

		int add(int num1, int num2) {
			int sum, carry;
			do {
				sum = num1 ^ num2;           //合并
				carry = (num1 & num2) << 1;  //进位
				
				num1 = sum;
				num2 = carry;
			} while(num2);
			
			return num1;
		}

12. 丑数

	根据丑数的定义，我们可以知道丑数可以由另外一个丑数乘以2，3或者5得到。因此我们创建一个数组，里面的数字是排好序的丑数，每一个丑数都是前面的丑数乘以2，3或者5得到的。这种思路的关键在于怎样确保数组里面的数字是排序的。

		//获取第k个丑数，假定1为第一个丑数
		int getUglyNumber(int index) {
		    if(index<=0) return 0;
		
		    int *pUglyNumbers=new int[index];
		    pUglyNumbers[0]=1;
		    int nextUglyIndex=1;
		    //定义三个指向丑数数组的指针，用它们来标识从数组中的哪一个数开始计算M2，M3和M5，开始都是丑数数组的首地址。
		    int *T2=pUglyNumbers;
		    int *T3=pUglyNumbers;
		    int *T5=pUglyNumbers;
		
		    while(nextUglyIndex<index)//
		    {
		        int min=Min(*T2 * 2,*T3 * 3,*T5 * 5);//M2=*T2 * 2, M3=*T3 * 3, M5=*T5 * 5
		        pUglyNumbers[nextUglyIndex]=min;//求M2，M3，M5的最小值作为新的丑数放入丑数数组
		        //每次生成新的丑数的时候，去更新T2，T3和T5.
		        while(*T2 * 2<=pUglyNumbers[nextUglyIndex]) ++T2;
		        while(*T3 * 3<=pUglyNumbers[nextUglyIndex]) ++T3;
		        while(*T5 * 5<=pUglyNumbers[nextUglyIndex]) ++T5;
		        nextUglyIndex++;
		    }
		    int ugly=pUglyNumbers[index-1];//因为丑数有序排列，所以丑数数组中的最后一个丑数就是我们所求的第index个丑数。
		    delete[] pUglyNumbers;
		    return ugly;
		}

13. BST的中序后继，每个节点有指向父节点的指针

	- If X has a right child, then the successor must be on the right side of X (because of the order in which we visit nodes) Specifically, the left-most child must be the first node visited in that subtree
	- Else, we go to X’s parent (call it P) 
		- 	If X was a left child (P left = X), then P is the successor of X
		- 	If X was a right child (P right = X), then we have fully visited P, so we call successor(P)
	
	![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/nextnode_zpsf870c92d.png)

14. 树的公共祖先

	![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/ancester1_zpsbaf0b765.png)

	![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/ancestere_zps539db7ec.png)

15. 计算0-n中2的数量

	![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/2s1_zps37fd204d.png)

	![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/2s2_zps19a01178.png)

16. 包含空串的有序字符串数组中寻找特定的字符串

	![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/empty_string_zps15c77190.png)