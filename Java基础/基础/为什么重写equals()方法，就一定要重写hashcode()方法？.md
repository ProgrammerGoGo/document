
[面试官：重写 equals 时为什么一定要重写 hashCode？](https://www.51cto.com/article/694975.html)

> 这是因为不同对象的 hashCode 可能相同；但 hashCode 不同的对象一定不相等，所以使用 hashCode 可以起到快速初次判断对象是否相等的作用。

hashcode 是Java虚拟机针对对象的内存地址计算出的一个整型值，可以是正整数和负整数。
