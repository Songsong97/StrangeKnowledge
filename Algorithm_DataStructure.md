# Algorithms and Data Structures
奇怪的知识增加了！！！
## Table of contents
1. [Floyd's Turtoise and Hare(环检测算法)](#Chapter1)
2. [Josephus Problem(约瑟夫环)](#Chapter2)
3. [Fisher-Yates Shuffle(洗牌算法)](#Chapter3)
4. [Boyer-Moore Majority Vote(寻找多数元素)](#Chapter4)
5. [KMP(字符串匹配)](#Chapter5)
    1. [Maximum Repetition Factors(最大循环因子/最小循环节)](#Chapter5.1)
    2. [Is T a rotation of S?(判断旋转字符串)](#Chapter5.2)
6. [Greedy(贪心法)](#Chapter6)
    1. [Task Scheduler (LeetCode 621)](#Chapter6.1)
7. [Dynamic Programming(动态规划)](#Chapter7)
    1. [Longest Common Subsequence(最长公共子序列)](#Chapter7.1)
8. [Data Structures(数据结构)](#Chapter8)
    1. [Heap(堆)](#Chapter8.1)
    2. [Red-Black Trees(红黑树)](#Chapter8.2)
    3. [B-Trees(B树)](#Chapter8.3)
    4. [Merge-Find Set(用于不相交集合的并查集)](#Chapter8.4)
9. [Graph(图算法)](#Chapter9)
    1. [Topolocial Sort(拓扑排序)](#Chapter9.1)
    2. [Strongly Connected Component(强连通分量)](#Chapter9.2)
    3. [Dijkstra's Algorithm(单源最短路径)](#Chapter9.3)
    4. [A* Algorithm(A*启发式最短路径算法)](#Chapter9.4)

<a name="Chapter1"></a>
## Floyd's Turtoise and Hare(环检测算法)
先看一道[例题](https://leetcode.com/problems/linked-list-cycle-ii/ "LeetCode 142: Linked List Cycle II")，给定一个链表，返回环的起点；
若没有环，返回null。

这里我们介绍一种算法，可以在有限状态机、迭代函数或者链表上判断是否存在环，并求出该环的起点与长度。

初始状态下，设两个指针t（turtoise，慢指针）和h（hare，快指针），将它们均指向起点S。同时让t和h往前推进，t每次前进1步，h每次前进2步。

当h无法前进，即到达某个没有后继的节点时，就可以确定从S出发不会遇到环。

若t与h再次相遇（在节点M），则可以确定从S出发一定会进入某个环。并且**从S到环的起点的距离**等于**从M到环的起点的距离**。

**算法正确性证明：**

1. 若h推进过程中next为空，则链表无环。这种情况是trivial的。

2. 现在考虑h与t再次相遇在M(M!=S)，这说明链表存在环。我们设S=0，环的起点为C，如下图所示。
![](./FloydTurtoiseHare.jpg)
假设t推进的次数是s，h的速度是它的两倍，所以推进了2s。h在环上走过的长度为2s-C，t在环上走过的长度为s-C，在他们相遇的节点M，存在k，使得2s-C=s-C+kL，
其中L是环的长度。

于是，s=kL。M在环上相对于C的偏移为(s-C)%L，也即(kL-C)%L。
这时我们令t从M继续往前推进，令一个新的指针n从S开始推进，当推进C步之后，(kL-C)%L+C=0，也就是说t到达C，而n也到达C，所以他们相遇的地方即为环的起点。

**证毕**

另一道例题：[LeetCode 287: Find the Duplicate Number](https://leetcode.com/problems/find-the-duplicate-number/)

<a name="Chapter2"></a>
## Josephus Problem(约瑟夫环)
n个人围成一个圈，每q个人踢掉一个人，问最后留下来的人是几号？

