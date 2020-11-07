---
title: 
   LeetCode递归
tags: 
  [LeetCode,递归]
categories:
	[算法]
---

# LeetCode递归

> 递归是一种解决问题的有效方法，在递归过程中，函数将自身作为子例程调用

你可能想知道如何实现调用自身的函数。诀窍在于，每当递归函数调用自身时，它都会将给定的问题拆解为子问题。递归调用继续进行，直到到子问题无需进一步递归就可以解决的地步。

为了确保递归函数不会导致无限循环，它应具有以下属性：

1. 一个简单的`基本案例（basic case）`（或一些案例） —— 能够不使用递归来产生答案的终止方案。
2. 一组规则，也称作`递推关系（recurrence relation）`，可将所有其他情况拆分到基本案例。

注意，函数可能会有多个位置进行自我调用。

## [509. 斐波那契数](https://leetcode-cn.com/problems/fibonacci-number/)

**斐波那契数**，通常用 `F(n)` 表示，形成的序列称为**斐波那契数列**。该数列由 `0` 和 `1` 开始，后面的每一项数字都是前面两项数字的和。也就是：

```java
F(0) = 0,   F(1) = 1
F(N) = F(N - 1) + F(N - 2), 其中 N > 1.
```

给定 `N`，计算 `F(N)`。

**示例 1：**

```java
输入：2
输出：1
解释：F(2) = F(1) + F(0) = 1 + 0 = 1.
```

**示例 2：**

```java
输入：3
输出：2
解释：F(3) = F(2) + F(1) = 1 + 1 = 2.
```

**示例 3：**

```java
输入：4
输出：3
解释：F(4) = F(3) + F(2) = 2 + 1 = 3.
```

**提示：**

- 0 ≤ `N` ≤ 30

这题相信教材上肯定也都写过

```java
public int fib(int N) {
	if(N==0||N==1){
		return N;
	}
	return fib(N-1)+fib(N-2);
}
```

这种写法虽然代码简洁但是效率比较低并不是一个好的递归，时间复杂度比较高，因为在递归的过程中会有很多重复的计算，我们这里就可以借助一些额外的空间来cache这些可能重复的结果，如下

```java
//数组记忆化
public int fib(int N) {
	int [] cache=new int[N];
	return fib(N,cache);
}

public int fib(int N,int []cache) {
	if(N==0||N==1){
    	return N;
	}
    if(cache[N-1]!=0){
    	return cache[N];
	}
    int res=fib(N-1)+fib(N-2);
	cache[N-1]=res;
    return res;
}
```

0ms ，100% beat. 空间换时间，不过其实更好的做法是用循环，这里是个尾递归，很好改成循环，相比循环递归会更加耗费时间和空间。

## [70. 爬楼梯](https://leetcode-cn.com/problems/climbing-stairs/)

You are climbing a stair case. It takes *n* steps to reach to the top.

Each time you can either climb 1 or 2 steps. In how many distinct ways can you climb to the top?

**Note:** Given *n* will be a positive integer.

**Example 1:**

```java
Input: 2
Output: 2
Explanation: There are two ways to climb to the top.
1. 1 step + 1 step
2. 2 steps
```

**Example 2:**

```java
Input: 3
Output: 3
Explanation: There are three ways to climb to the top.
1. 1 step + 1 step + 1 step
2. 1 step + 2 steps
3. 2 steps + 1 step
```

**题解**

这题也很经典，先来个最暴力的解法

```java
public static int climbStairs(int n) {
	return climbStairs(0,n);
}

public static int climbStairs(int cur,int n) {
	if(cur>n){
		return 0;
	}
	if(n==cur){
		return 1;
	}
	return climbStairs(cur+1,n)+climbStairs(cur+2,n);
}
```

这个递归开始的时候我也没看明白，后来看了一下题解就明白了

![leetcode](https://i.loli.net/2019/07/15/5d2bf5512455c28556.jpg)

其实这里最左边的子节点还可以向下扩展，这个递归最根本的一个递推公式就是

_**climbStairs(i,n)**_=_**climbStairs(i+1,n)**_+_**climbStairs(i+2,n)**_

递归的出口就是 `i==n` 和 `i>n`的时候，`i==n`的时候刚好走完，算一条**return 1**，`i>n`的时候超过了，不算一条 **return 0**.

递归树深度差不多N层 ，所以整个节点数为 2^N个，时间复杂度 `O(2^N)`

但是仔细看这个图你会发现和上面的 `斐波拉契数`一样有很多重复的计算而这些计算会耗费很多时间所以我们可以利用上面一样的思路做**记忆化**

```java
private static int[] cache=null;

public static int climbStairs(int n) {
    cache=new int[n+1];
    return climbStairs(0,n);
}

public static int climbStairs(int cur,int n) {
    if(cur>n){
        return 0;
    }
    if(n==cur){
        return 1;
    }
    if (cache[cur]>0) {
        return cache[cur];
    }
    cache[cur]=climbStairs(cur+1,n)+climbStairs(cur+2,n);;
    return cache[cur];
}
```

1ms，43%显然不是最优解。经过记忆化之后树的规模变为N时间复杂度变为O(N)，其实这题还有DP的解法，这里是递归专题就不展开了。

## [104. 二叉树的最大深度](https://leetcode-cn.com/problems/maximum-depth-of-binary-tree/)

  Given a binary tree, find its maximum depth.

The maximum depth is the number of nodes along the longest path from the root node down to the farthest leaf node.

**Note:** A leaf is a node with no children.

**Example:**

Given binary tree `[3,9,20,null,null,15,7]`,

```java
    3
   / \
  9  20
    /  \
   15   7
```

return its depth = 3.

**题解**

递归实现，核心公式  `maxDepth(root)=1+max(maxDepth(root.left),maxDepth(root.right));`

```java
public int maxDepth(TreeNode root) {
    if(root==null){
        return 0;
    }
    int maxLeft=maxDepth(root.left);
    int maxRight=maxDepth(root.right);
    return (maxLeft>maxRight?maxLeft:maxRight)+1;
}
```

## [111. 二叉树的最小深度](https://leetcode-cn.com/problems/minimum-depth-of-binary-tree/)

给定一个二叉树，找出其最小深度。

最小深度是从根节点到最近叶子节点的最短路径上的节点数量。

**说明:** 叶子节点是指没有子节点的节点。

**示例:**

给定二叉树 `[3,9,20,null,null,15,7]`   

```java
	3
   / \
  9  20
    /  \
   15   7
```

返回它的最小深度  2.

**解法一**

```java
public int minDepth(TreeNode root) {
    if (root==null) return 0;
    if (root.left==null) {
        return minDepth(root.right)+1;
    }
    if (root.right==null) {
        return minDepth(root.left)+1;
    }
    return Math.min(minDepth(root.left),minDepth(root.right))+1;
}
```

## [50. Pow(x, n)](https://leetcode-cn.com/problems/powx-n/)

实现 pow(*x*, *n*) ，即计算 x 的 n 次幂函数。

**解法一**

这里就要介绍一种快速幂算法了

```java
    public static double fastPow(double x,int n){
        if(n==0){
            return 1;
        }
        if(n<0){
            x=1/x;
            n=-n;
        }
        double res=fastPow(x,n/2);
        if(n%2==0){
            return res*res;
        }
        return res*res*x;
    }
```

核心思想就是 `x^n=(x^2/n)^2`，常规累乘的方式计算时间复杂度是O(N)因为要遍历所有的元素，但是其实知道了`x^n/2`之后 `x^n`就可以直接平方得到了不用继续遍历，整体时间复杂度为O(logN) 

2019.8.20，又写了一遍，提交然后没过。看了下给的测试用例，最后一个给的n是 `-2^31` 也就是int整数的最小值，int类型的取值范围是 `-2^31 ~ 2^31-1` 而这个负值在这里取反之后会直接溢出最后得到的还是 `-2^31` ，所以这里这样写 if会执行两次，x就又会变回来，所以结果直接就是`Infinity`无穷大了，所以为了保证if只会执行一次可以将其封装一下

```java
public static double myPow(double x, int n) {
    if(n<0){
        x=1/x;
        n=-n;
    }
    return fastPow(x,n);
} 

public static double fastPow(double x,int n){
    if(n==0){
        return 1.0;
    }
    double  half=fastPow(x,n/2);
    if(n%2==0)
        return half*half;
    return half*half*x;
}
```

## [21. 合并两个有序链表](https://leetcode-cn.com/problems/merge-two-sorted-lists/)

将两个有序链表合并为一个新的有序链表并返回。新链表是通过拼接给定的两个链表的所有节点组成的。 

**示例：**

```java
输入：1->2->4, 1->3->4
输出：1->1->2->3->4->4
```

这题在之前的 [链表专题](http://imlgw.top/2019/02/27/leetcode-lian-biao-tag/#21-%E5%90%88%E5%B9%B6%E4%B8%A4%E4%B8%AA%E6%9C%89%E5%BA%8F%E9%93%BE%E8%A1%A8) 有写过但是没写递归的版本，这里是递归，自然要写一下递归试试

**解法一**

```java
public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
    ListNode dummyNode=new ListNode(-1);
    mergeTwoLists(l1,l2,dummyNode);
    return dummyNode.next;
}

public void mergeTwoLists(ListNode l1, ListNode l2,ListNode res) {
    if(l1==null){
        res.next=l2;
        return;
    }

    if(l2==null){
        res.next=l1;
        return;
    }

    if(l1.val>l2.val){
        res.next=l2;
        mergeTwoLists(l1,l2.next,res.next);
    }else{
        res.next=l1;
        mergeTwoLists(l1.next,l2,res.next);
    }
}
```

说实话，这个写法一点也不_递归_ 完全是按照迭代的思路来写的。。。

**解法二**

```java
public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
    if(l1==null) return l2;
    if(l2==null) return l1;
    ListNode dummyNode=new ListNode(-1);
    if(l1.val<l2.val){
        l1.next=mergeTwoLists(l1.next,l2);
        return l1;
    }else{
        l2.next=mergeTwoLists(l1,l2.next);
        return l2;
    }
}
```

## [779. 第K个语法符号](https://leetcode-cn.com/problems/k-th-symbol-in-grammar/)

On the first row, we write a `0`. Now in every subsequent row, we look at the previous row and replace each occurrence of `0` with `01`, and each occurrence of `1` with `10`.

Given row `N` and index `K`, return the `K`-th indexed symbol in row `N`. (The values of `K` are 1-indexed.) (1 indexed).

```java
Examples:
Input: N = 1, K = 1
Output: 0

Input: N = 2, K = 1
Output: 0

Input: N = 2, K = 2
Output: 1

Input: N = 4, K = 5
Output: 1

Explanation:
row 1: 0
row 2: 01
row 3: 0110
row 4: 01101001
```

**Note:**

1. `N` will be an integer in the range `[1, 30]`.
2. `K` will be an integer in the range `[1, 2^(N-1)]`.

**解法一**

找规律，前半部分和后半部分是有一定规律的，把前六行都写出来

```java
  第一行: 0
  第二行: 01
  第三行: 01|10
  第四行: 01 10|10 01
  第五行: 01 10 10 01|10 01 01 10
  第六行: 01 10 10 01 10 01 01 10 | 10 01 01 10 01 10 10 01
```

  N%2!=0 对称, 第K个等于 2^(N-1)-K+1
  N%2==0 互补对称

```java
public int kthGrammar(int N, int K) {
    if(K==1 || N==1){
        return 0;
    }
    if(K==2){
        return 1;
    }
    int len=1<<(N-1); //当前行长度
    if(K>len/2){ //大于1/2
        //结合上面的规律，找前半部分和自己等价的位置
        if(N%2!=0){ 
            K=len-K+1;
        }else{
            if(K%2==0){
                K=len-K+2;  
            }else{
                K=len-K;
            }
        }
    }
    //去上一行继续
    return kthGrammar(N-1,K);
}
```

时间复杂第O(N)，思路还算清晰，最开始没想到用`位运算`来算长度，用的`pow()`最后效率差不多，可能是底层做了优化。

**解法二**

这种解法实际上就是把整个序列看作一颗满二叉树，每个节点的值和父节点其实是有对应关系的，如果K是偶数那么就和父节点的值相反，否则就相同，所以我们可以递归的去找父节点对应的index的值。

```java
//01排列
//              0
//          /        \   
//      0                1
//    /   \            /    \
//  0       1        1       0
// / \     /  \     /  \    / \ 
//0   1   1    0   1    0  0   1


public int kthGrammar(int N, int K) {
    if (K==1) return 0;
    //(K+1)/2是对应父节点的index
    int parent=kthGrammar(N-1,(K+1)/2);
    //取反
    int f_parent=-(parent-1);
    if (K%2==0) {
        return f_parent;
    }
    return parent;
}
```
时间复杂度依然是`O(N)` 但是比上面那种要更清晰明了

**解法三**

这个解法其实和上面的思路是一样的，都是利用父节点和K的奇偶来判断，其实仔细看上面的代码你会发现N其实并没有实际的意义，具体K的值只和K本身有关，下面的解法就没有用到N.

```java
public static int kthGrammar3(int N, int K) {
    boolean r=false;
    while(K>1){
        if (K%2==0) {
            K=K/2;
            r=!r;
        }else{
            K=(K+1)/2;
        }
    }
    return r?1:0;
}
```
这题其实还有一种解法，利用二进制，对K做奇偶检验，貌似时间复杂度是O(1)。

