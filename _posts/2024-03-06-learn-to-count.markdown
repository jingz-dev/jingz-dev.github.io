---
layout: post
title:  "Lrn2Count: Reasoning about array boundaries properly"
date:   2024-03-06 23:00:00 -0800
categories: coding algorithms interview
---
Let's start with a fun brain teaser to delve into a common array-related concept:

Imagine you have 2 arrays, `A` and `B`.  Let's use `|A|` to denote the number of elements in `A`, a concept borrowed from set theory. Similarly, `|B|` represents the number of elements in `B`. We can access the elements of these arrays using `A[i]` and `B[j]`, where `0 <= i <= |A| - 1` and `0 <= j <= |B| - 1`. This might remind you of looping through array elements.

Now, let's introduce a new array, `C`, which is a combination of all elements from `A` and `B`. If we think in terms of concatenation, we can express this as `C = A + B`. When accessing the elements of `C` using `C[k]`, we consider:

1. `k <= |C| - 1` (following our loop logic).
2. Given `C = A + B`, logically, we would think `|C| = |A| + |B|` (the total number of elements in `C` should equal the sum of the number of elements in `A` and `B`).
3. If we try to express `k` in terms of `i` and `j`, we might assume `k = i + j`, leading to the conclusion that `k <= |A| + |B| - 2` (since `k = i + j <= |A| - 1 + |B| - 1 = |A| + |B| - 2`).

Now `k <= |C| - 1` and `k <= |C| - 2` at the same time! Where did the extra 1 go? If you know the answer already, then good for you! You know how to count.

The reason is, arithmetics between indicies in different arrays have no meaning. The only situation where a computation between 2 indices make sense, is subtraction between 2 indices within the __same__ array to __count__ the number of elements in between: number of elements in [C<sub>a</sub>, C<sub>a+1</sub>, ..., C<sub>b</sub>] is *b - a + 1*. 

Here are a few rules for index manipulation:
1. index - index = *# of elements*
    * How many elements are between C[5] and C[15]? 
2. index + *# of elements* = index
    * What's the index of 10 elements after C[5]?
3. *# of elements* + *# of elements* = *# of elements*
    * If 10 elements in A, and 5 elements in B, how many elements are there in total?
4. index + index = ??
    * Does __not__ make logical sense, unless they are a part of a bigger expression that is somehow reordered, or the index happens to mean something else.

Unfortunately it still is very confusing, say java's `subarray(C, a, b)` method, or python's array slicing operation `C[a:b]`, where if you try to count the number of elements, it will be __b - a__, because b is exclusive - the end result does not actually contain the bth index.
It is very convient to ignore the extra "1" in programming, but if you've never paid much attention to it, then the below questions may be tricky to reason properly.

Binary search
```python
def binary_search(arr: List[int], target: int):
    left: int = 0
>>  right: int = len(arr) / len(arr) - 1 ??
>>  while left < right / left <= right ?? :
        ...
    return -1
```

Sliding Window
```python
def sliding_window(arr: List[int], max_window_size: int):
    left = 0
    right = 0
    window = 0
>>  while right <= len(arr) / right < len(arr): ??
        window += arr[right]
        right += 1
>>      if right - left / right - left + 1 > max_window_size ??:
            window -= arr[left]
            left += 1
```

Select kth smallest element of two sorted arrays
```python
def select_kth(arr1: List[int], arr2: List[int], k: int):
    i = len(arr1) // 2
    j = len(arr2) // 2
    if arr1[i] > arr2[j]:
        if i + j + ?? > k: 
        # remember, it's not 2 indices that's being added, it's the count
            return select_kth(arr1[:i+??], arr2, k)
        elif i + j + ?? < k:
            return select_kth(arr1, arr2[j+??:], k - ??)
        else:
            return arr?[??]
    else:
        if i + j + ?? > k:
            return select_kth(arr1, arr2[:j+??], k)
        elif i + j + ?? < k:
            return select_kth(arr1[:i+??], arr2, k - ??)
        else:
            return arr?[??]
```
<span style="color:white"><sub><sub><sub><sub>To be honest, it's hard to imagine that a typical programmer could come up with the solution and code it properly within 45 minutes under interview pressure.</sub></sub></sub></sub></span>

Thank you for reading, and I hope you gained some insights. The key is that when you are operating on indices, you are actually counting the number of elements, or offsetting an index with a count.

Here's a final brain teaser: How many years are there between 2023 and 2022, and why doesn't __*b-a+1*__ apply?

----
### References
[1] [Fencepost error](https://en.wikipedia.org/wiki/Off-by-one_error#Fencepost_error)  
[2] [Binary Search](https://en.wikipedia.org/wiki/Binary_search_algorithm#Procedure)  
[3] [Sliding Window](https://www.geeksforgeeks.org/window-sliding-technique/)  
[4] [Median of two sorted arrays](https://leetcode.com/problems/median-of-two-sorted-arrays/description/)  
[5] [Java subarray](https://commons.apache.org/proper/commons-lang/apidocs/org/apache/commons/lang3/ArrayUtils.html#subarray-T:A-int-int-)  
[6] [Python array slicing](https://en.wikipedia.org/wiki/Array_slicing#1991:_Python)  