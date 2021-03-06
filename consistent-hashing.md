# 一致性哈希

## 参考资料
[Consistent Hashing: Algorithmic Tradeoffs](https://medium.com/@dgryski/consistent-hashing-algorithmic-tradeoffs-ef6b8e2fcae8)

[jump hash](https://github.com/chuanlei/leetcode-notes/blob/master/paper/jump-consistent-hashing.pdf)

注：我们可以参考guava中`Hashing.consistentHash(long input, int buckets)`方法的使用

![guava-consistent-hashing](https://github.com/chuanlei/leetcode-notes/blob/master/pics/guava-hashing-usage.jpg)

输出结果为

![consistent-hashing output](https://github.com/chuanlei/leetcode-notes/blob/master/pics/consistent-hashing-output.jpg)

我们实验表明：在jump-hashing的语境下server的数量可增可见

[groupcache: 客户端、服务端一体的缓存系统](https://github.com/golang/groupcache)

ps: groupcache代码很简洁，值得一读

## 优秀的函数特性
![optimal functions](https://github.com/chuanlei/leetcode-notes/blob/master/pics/optimal-functions.jpg)

该方案在1997年CDN相关论文中提出，在2007年memcache和dynamo中发扬光大。

我们可以想象一个0..2^32-1的整数构成的环(ring)。

基本想法是将server映射到环上的某个点；如果需要根据key查找对应的server，那么从映射到的点开始顺序扫描。

## Ring Hash

## Jump Hash(2014)

## Multi-Probe Consistent Hashing

## Rendezvous Hashing(2015)

## Maglev Hashing(2016)
