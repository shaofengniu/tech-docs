# /proc/$pid/io

在新内核(2.6.32)中，`/proc/$pid/io`这个文件的数值可以帮助我们分析程序
的io行为。

```shell
arch@tc-arch-ndb02.tc:/proc/10662/ cat io 
rchar: 64844838219
wchar: 126958170323
syscr: 134154912
syscw: 52515666
read_bytes: 15813550080
write_bytes: 30828637048832
cancelled_write_bytes: 4096
```

## rchar

这个进程传给其调用的`read()`或`pread()`的数据大小的和。这个数值跟具体
从物理磁盘读取的数据量无关（读取的数据可能是从page cache中获取的）

## wchar

解释基本同`rchar`，即传给`write()`和`pwrite()`的和，与该数据是否被写入
物理磁盘无关。

## syscr

syscall read, 即该进程调用`read()`和`pread()`次数之和。

## syscw

同上，`write()`和`pwrite()`调用次数之和。

## read_bytes

实际从物理磁盘读取的数据总量

## write_bytes

实际被写入磁盘的数据总量

## cancelled_write_bytes

这个我没自己看，应该用处不大。。

## 关于socket读写是否会被计算在此处

上面的对`read`和`write`的统计都是在VFS内进行的。由于使用`send`,
`recv`等socket专有函数进行操作是不会经过VFS的，所以这种情况下的io是不
会被统计在该文件内的。但是如果使用`write`和`read`等通用的fs syscall进
行操作是会经过VFS的，也就是说socket的io也会被统计在该文件内。
