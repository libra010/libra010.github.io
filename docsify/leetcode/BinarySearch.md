# 二分查找



虽然二分查找是一道经典的算法题。但是还是有不少坑的。



```python
def lower_bound(array, value, lo=0, hi=None):
    if lo < 0:
        raise ValueError('lo must be non-negative')
    if hi is None:
        hi = len(a)
    while lo < hi:
        mid = lo + (hi - lo) // 2
        if array[mid] < value: 
            lo = mid + 1 
        else: hi = mid
    return lo
```



四个参数：`array`：有序数组，`value`：待查找的值，`lo`（low），`hi`（high）：待查找的上界和下界。

函数名：`lower_bound` 意思是下界搜索，返回值是第一个大于或等于value的数组元素的下标。

可以解决二分查找中有重复元素的问题，如果只需任意一个value，则可以判断一下返回值是否等于value。



维护一个左开右闭区间 `[lo, hi)` ，区间为空时（即`lo`和`hi`重合），返回`lo`即可。

求中点`mid`时从`lo`出发，以确保区间长度为`1`时，`mid = lo`仍在`[lo, lo+ 1)`区间内。
