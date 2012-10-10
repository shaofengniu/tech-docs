# iostat信息分析

`iostat`通过采集`/proc/diskstats`中的数据并进行解析来对系统的io状况进
行展示，是我们在对系统io性能进行分析时最重要的工具之一。

可以在[](http://sebastien.godard.pagesperso-orange.fr/)下载其各个版本
的源码。

其基本用法如下:

```shell
-> % iostat -dxk 5
Device:    rrqm/s wrqm/s   r/s   w/s  rsec/s  wsec/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await  svctm  %util
sda          0.05  62.60  0.86  9.96   64.33    8.58    32.17     4.29     6.73     0.03   10.43   1.60   1.73
```

其中5为采样间隔，其余三个选项意义不详述，可参见`man iostat`，这里主要
解释一下`iostat`的各项输出的意义及可能反映出我们的机器在哪方面可能存在
问题。

### rrqm/s wrqm/s

read/write requests merged per second，即每秒钟merge的读/写请求次数。
这里解释一下merge的意思，操作系统在收到一个读写请求的时候，会根据要访
问文件的位置，在请求队列（request queue，以下简称rq）中查找访问相邻位
置的请求，如果有的话，就与其合并为一个请求，以减少io次数，提高效率。

### r/s w/s

read/write requests per second，即磁盘每秒钟完成的读/写请求次数。

### rsec/s wsec/s及rkB/s wkB/s

read/write sectors/kBs per seconds，即磁盘每秒完成的读/写的数据量的大小，单位为
扇区或者kB。该数值可以反映出磁盘的整体吞吐。

### avgqu-sz

average queue size，即平均请求队列长度。请求队列过长，说明系统的压力比
较大，每个请求的等待时间也会比较长。

### avgrq-sz

average request size，即平均请求大小，单位为sector。显然，少量的大请求
处理起来要比大量的小请求效率高得多，通过这个数值能够看出我们程序的io访问
模式是否合理。

### svctm

service time，平磁盘请求处理时间，这个数值只与底层磁盘的处理能力相关。
由于写操作能够比读操作更好的利用磁盘的cache，所以写请求的svctm通常要比
读请求的svctm小一些，大致在1ms左右。如果发现线上在大部分是写请求的情况
下svctm却比较高，那么很有可能是磁盘出了问题。

### await

average wait time，即平均等待时间，包括请求在队列中等待的时间和磁盘处
理该请求用的时间，体现了一个请求的整体处理延迟。

### %util

disk utilization，即磁盘带宽占用率。这个值最能够直观的体现出磁盘的压力，
如果发现util接近100%，那毫无疑问该机器遇到了io瓶颈。

## 实例

这里简单举几个例子，在每次操作之前都需要先使用下面的命令清空系统的page
cache，否则得到的结果没有说服力

```shell
root#sync && sysctl -w vm.drop_caches=3
```

首先，我们看一下两个读文件的例子，第一个为顺序读，第二个为随机读。
两种io模式的差异非常明显的反映在iostat的结果上。从`rrqm/s`可以看出，顺
序读的情况下，大量的连续请求会被合并成同一个请求，`qvgrq-sz`显示的平均
请求大小也是顺序读要大的多。

随机读的`svctm`的值要远大于顺序读的值，这是因为随机读的清空下，磁盘要
反复进行寻道，自然处理请求的速度要慢得多。

上述原因综合起来，就造成了两种情况下`rkB/s`的巨大差别，顺序读有270M/s，
而随机读则只有890k/s。

```shell
#sequential read
Device:         rrqm/s   wrqm/s     r/s     w/s     rkB/s    wkB/s avgrq-sz avgqu-sz   await  svctm  %util
sda             161.00     0.00 2193.00    0.00 271756.00     0.00   247.84     4.09    1.85   0.46 100.00
```


```shell
#random read
Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await  svctm  %util
sda               0.00     0.00  224.00    0.00   896.00     0.00     8.00     1.00    4.46   4.45  99.60
```

下面的这个是顺序写的例子，由于大量的相邻写请求可以被合并，所以
`wrqm/s`会相当高。我们对比一下和上面顺序读的例子的差别，可以发现顺序写
的情况下，`avgqu-sz`即请求队列的长度要大得多。这是由于从本质上来说读操
作是同步的（应用程序需要等读操作返回数据才能继续执行），而写操作则是异
步的。所以IO调度器会倾向于积攒一定数量的写操作一起执行，这样能够提高合
并请求操作的成功率，而对读请求则是会尽量早的执行。由于请求队列比较长，
请求的平均等待时间`await`也会比较大。

```shell
#dd if=/dev/zero of=out
Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await  svctm  %util
sda               0.00 17553.00    0.00  322.00     0.00 33896.00   210.53   142.18  304.78   3.11 100.00
```

下面的这个例子是复制文件的操作，即顺序读和顺序写的混合请求模式。由于是
读-写-读-写，所以平均队列长度会比较短。

```shell
#cp in_file out_file
Device:         rrqm/s   wrqm/s     r/s     w/s     rkB/s    wkB/s avgrq-sz avgqu-sz   await  svctm  %util
sda               2.00  7078.00 2983.00    7.00 381952.00  1292.00   256.35     5.03    0.70   0.32  95.30
```

## 分析方法

通常情况下，`%util`是我们首先要关注的，如果这个值很高，那么毫无疑问我
们遇到了io问题。

接下来，要根据`rkB/s`, `wkB/s`, `avgqu-sz`来判断io压力是
不是很大，分以下几种情况：

* 读或者写的量很大

没什么可说的，就是压力太大。。。

*  读写量都不大，但avgqu-sz很大：说明io压力不平均，可能某一小段时间爆
发了大量的请求，导致请求队列中积累了大量的待处理请求

* 读写量都不大，avgqu-sz也不大：首先要看看是不是读写混合型请求，如一个
程序需要频繁的读写同一个文件，这种情况下由于磁盘需要不断的寻道，吞吐会
变得极低，可能只有几M/s。
其次，如果发现`rkB/s`和`wKB/s`都不大，但是
`r/s`或`w/s`很大，这种情况下`avgrq-sz`会比较小，说明程序在进行大量的小
io操作，导致磁盘要进行大量的寻道，极大的降低处理效率。
如果上面两种情况
均不符合的话，就需要看一下`svctm`是不是正常，如果发现`svctm`异常的高，
那么很有可能是磁盘出现了硬件故障。

