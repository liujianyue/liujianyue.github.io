---
layout: post_layout
title:  详解算法：求两个有序数组的中位数
time: 2016年08月03日
location: 北京
pulished: true
excerpt_separator: "~~~"
---
本题出自LeetCode 的第四题 Median of Two Sorted Arrays，下面想说一下我的想法。
为了解决这个问题，我们需要首先理解什么是中位数。在统计学中，中位数被用于将一个集合切割成两个长度相等的集合，并且其中一个集合中元素总是比另一个集合中的元素大。如果我们明白了中位数在切割中的使用，那我们已经很接近本题的答案了。

首先让我们以一个随机的位置 **i** 来切割 **A**：

``` stylus
      left_A             |        right_A
A[0], A[1], ..., A[i-1]  |  A[i], A[i+1], ..., A[m-1]
```
既然**A**有**m**个元素，那么A的切割方法就有**m+1**种（i = 0~m）,并且我们知道：**len(left_A) = i**, **len(right_A) = m - i** .
PS:当 **i = 0** 时，**left_A**为空，当**i = m** 时，**right_A** 为空。
~~~
使用相同的办法，使用随机位置 j 来切割集合B为两部分：

``` stylus
left_B             |        right_B
B[0], B[1], ..., B[j-1]  |  B[j], B[j+1], ..., B[n-1]
```
将 **left_A** 和 **left_B** 合并到一个集合中, 并且将 **right_A** 和 **right_B** 合并到另一个集合中. 我们将其命名为**left_part** 和 **right_part** :

``` stylus
left_part          |        right_part
A[0], A[1], ..., A[i-1]  |  A[i], A[i+1], ..., A[m-1]
B[0], B[1], ..., B[j-1]  |  B[j], B[j+1], ..., B[n-1]
```

如果我们能确保：
 
``` stylus
1) len(left_part) == len(right_part)
2) max(left_part) <= min(right_part)
```

那么我们就能将**{A, B}**中所有的元素分割到两个长度相等的集合中，并且其中一个集合的元素总是比另一个集合中元素大。因此我们可以得到： **median = (max(left_part) + min(right_part))/2**.
为了确保以上两个条件成立，我们只需要确保一下两点：

``` stylus
(1) i + j == m - i + n - j (or: m - i + n - j + 1)
    if n >= m, we just need to set: i = 0 ~ m, j = (m + n + 1)/2 - i
(2) B[j-1] <= A[i] and A[i-1] <= B[j]
```
（为了思路清晰，我们假设**A[i-1]****,B[j-1]**,**A[i]**,**B[j]**总是合法的，就算是 **i=0/i=m/j=0/j=n**，一会我们会谈及到如何处理这些边界元素。）
所以我们需要去做的是：

``` stylus
在 [0, m]之间搜索, 目的是去寻找第 i 个值满足：:
    B[j-1] <= A[i] and A[i-1] <= B[j], ( where j = (m + n + 1)/2 - i )
```
接下来我们可以通过二分查找法来找到这个合适的值：

``` stylus
<1> 让 imin = 0, imax = m,然后在[imin, imax]范围之间查找
<2> 让 i = (imin + imax)/2, j = (m + n + 1)/2 - i
<3>现在可以满足条件 len(left_part)==len(right_part)了，那现在只需有三种情况需要我们关注：
    <a> B[j-1] <= A[i] and A[i-1] <= B[j]
        此时说明我们已经找到了合适的 i，所以停止查找。
    <b> B[j-1] > A[i]
        此时说明  A[i] 太小，我们必须调整 i ，使得`B[j-1] <= A[i]`
        我们能增大 i 的大小吗？
            必须能，因为i 增加，j 就会减小。 所以B[j-1] 就会减小且 A[i] 就会增加，可能最终 `B[j-1] <= A[i就会成立。
        但是我们能够减小 i 吗？
            不能！因为当i减小时，j 就会增加，使得B[j-1]增加， A[i] 减小，以至于B[j-1] <= A[i]永远都不会成立！
        所以我们必须 增加 i 。也就是说，我们必须重新在 [i+1, imax]范围内搜索。所以，让 imin = i+1，且跳转到<2>继续执行。
    <c> A[i-1] > B[j]
        此时 A[i-1] 说明太大，我们必须减小 i 以使满足`A[i-1]<=B[j]`
        也就是说，我们必须调整搜索范围为 [imin, i-1]。所以让 imax = i-1,并且跳转到<2>重新执行。
```

当我们找到合适的 **i** 时，中位值也就找打了：

``` stylus
max(A[i-1], B[j-1]) (when m + n is odd)
or (max(A[i-1], B[j-1]) + min(A[i], B[j]))/2 (when m + n is even)
```

处理完以上情况，我们需要去处理 **i=0,i=m,j=0,j=n** 也就是 **A[i-1],B[j-1],A[i],B[j]** 可能不存在的情况。事实上这种情况比想象的要简单。
我们需要确保的是保证**max(left_part) <= min(right_part)**.所以如果 **i** 和  **j** 不是特殊的值（），那我们需要去确保**B[j-1] <= A[i] and A[i-1] <= B[j]**.但是如果**A[i-1],B[j-1],A[i],B[j]**其中某些值不存在，那么我们就不需要**B[j-1] <= A[i] and A[i-1] <= B[j]** 两者都确保成立了。比如说，**i =0**，那么  **A[i-1]** 就不存在了，于是我们就不需要检测**A[i-1] <= B[j]**到底成不成立了，所以，我们需要去做的是：

``` stylus
在 [0, m]之间搜索, 找到合适的 i 来确保:
    (j == 0 or i == m or B[j-1] <= A[i]) 且
    (i == 0 or j == n or A[i-1] <= B[j])
    当然需要保证  j = (m + n + 1)/2 - i
```
综上所述，字啊一个查找的循环中，我们需要关注三种情况：

``` stylus
<a> (j == 0 or i == m or B[j-1] <= A[i]) and
    (i == 0 or j = n or A[i-1] <= B[j])
   此时说明我们已经找到了合适的 i 了，停止查找。

<b> j > 0 and i < m and B[j - 1] > A[i]
    此时说明 i 太小，我们需要增加 i

<c> i > 0 and j < n and A[i - 1] > B[j]
    此时说明 i 太大，我们需要减小 i
```
下面给出Java算法：

``` stylus
    public double findMedianSortedArrays(int[] nums1, int[] nums2) {
         int m = nums1.length;
	int n = nums2.length;
	if (m > n) {
		return findMedianSortedArrays(nums2, nums1);
	}
	int i = 0, j = 0, imin = 0, imax = m, half = (m + n + 1) / 2;
	double maxLeft = 0, minRight = 0;
	while(imin <= imax){
		i = (imin + imax) / 2;
		j = half - i;
		if(j > 0 && i < m && nums2[j - 1] > nums1[i]){
			imin = i + 1;
		}else if(i > 0 && j < n && nums1[i - 1] > nums2[j]){
			imax = i - 1;
		}else{
			if(i == 0){
				maxLeft = (double)nums2[j - 1];
			}else if(j == 0){
				maxLeft = (double)nums1[i - 1];
			}else{
				maxLeft = (double)Math.max(nums1[i - 1], nums2[j - 1]);
			}
			break;
		}
	}
	if((m + n) % 2 == 1){
		return maxLeft;
	}
	if(i == m){
		minRight = (double)nums2[j];
	}else if(j == n){
		minRight = (double)nums1[i];
	}else{
		minRight = (double)Math.min(nums1[i], nums2[j]);
	}
	return (double)(maxLeft + minRight) / 2;
}
```








