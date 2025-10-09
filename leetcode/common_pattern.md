# 常见套路

## 滑动窗口

问题要求你寻找元素的一个子集

```
****#####***********
      ^ window size=5
```

*e.g.*寻找最长的子字符串，包含K个唯一的字符。
```
input: aabbcc , k = 1
outout: 2
explaination: {aa,bb,cc}

input: aabbcc , k = 2
outout: 4
explaination: {aabb,bbcc}
```

## 子集 Subset

寻找所有可能的组合，可能重复或不重复

*e.g.* 给定数组，返回所有可能的排列。

```
input: nums = [1,2,3]
output: [[1,2,3],[1,3,2],...,[3,2,1]]
```

## 改进的二分查找

下为二分查找模板

```
def biSerach(array,x):
    low,high = 0,len(array)-1
    while low<=high:
        mid = low+(high-low)//2
        if array[mid] == x:
            return mid
        elif array[mid] < x:
            low = mid+1
        else:
            high = mid-1
    return -1
```

## top-k 元素

在大数据中找到top-k个元素

*e.g.* 找到第K大的数据
使用堆

## 二叉树DFS与BFS

DFS，递归。求深度等。
BFS，需要利用队列。 *e.g.* 二叉树水平翻转

## 拓扑排序

有向无环图常见。

## 双指针

针对一个已有序的数组遍历。
*e.g.* 两数之和/三数之和