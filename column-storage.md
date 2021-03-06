# 列存储

## 列压缩
位图编码等

经典的参考文献为： `The Design and Implementation of Modern Column-Oriented Database systems`

## 内存带宽与矢量化处理
查询引擎将一大块压缩列数据放入CPU的L1缓存中，并以紧凑循环（即没有函数调用）进行迭代。对于每个被处理的记录，CPU能够比基于很多函数调用和条件判断的代码更
快地执行这种循环。列压缩使得列中更多的行可以加载到L1缓存。这种技术被称为矢量化处理。

## 列存储中的排序

C-Store中最早引入的一种改进想法，并被商业数据库Vertica所采用：既然数据需要复制到多台机器，这样在一台机器发生故障时，不会丢失数据；那么不妨存储不
同方式排序的冗余数据，以便在处理查询时，可以选择最适合特定查询模式的排序版本。
