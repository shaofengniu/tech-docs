# Linux刷脏页机制与NDB queue full问题

## NDB queue full?

目前线上NDB服务存在的一个最大的问题就是每个小时会有若干分钟不能提供服
务，不断的报queue full的错误，这段时间请求的耗时长达几十秒。

据观察，拒绝服务的几分钟恰好是BDB在进行checkpoint的时间，而且从iostat
的结果来看，这段时间的写入量非常大，util一直处在100%的状态。所以很自然
的猜想，可能是因为checkpoint的时候io压力过大，导致worker thread被阻塞，


但是worker thread只是write，并没有fsync，理论上应该只是把要写入的数据
copy到kernel的Page cache中，并不会涉及到磁盘io，为什么会被阻塞这么长时
间呢？

## NDB? BDB? checkpoint? Page cache?

* BDB是指berkeley db，一个嵌入式的kv引擎库
* NDB是基于ub框架，封装了BDB，对外提供了网络层的服务接口
* checkpoint是大多数数据库都会存在的一个操作，目的是将数据库cache中的
  脏页写到磁盘上，保证数据安全
* page cache，是kernel对最近读、写过的文件的页的缓存，来减少不必要的磁
  盘io  

## 线下复现

在线下复现中，我们使用了比线上大若干倍的压力，更加随机的访问模式，试图
复现在checkpoint的时候长耗时的情况。

但结果是我们从来没有能够在线下复现这种几十秒的长耗时情况。。。。

## write到底怎么实现的？

这里讨论的write特指通常的buffered write，即将数据写入Page cache，而不
是直接同步写入磁盘。

那linux的write到底是怎么实现的？这个操作有没有可能阻塞？

linux的write操作在每次产生一个脏页之后，都会查看当前系统中的脏页是否超
过了某个阈值，如果是的话就需要采取某些手段将一部分脏页写入磁盘，来保证
系统中的脏页不会过多。

这里的阈值最主要有两个：dirty\_ratio和dirty\_background\_ratio，虽然
两者均表示脏页占总可用页数的比例，但当超过该阈值时触发的操作则有所不同。

当脏页比例超过dirty\_background\_ratio的时候，若干个专门负责写入脏页的
kernel thread（如pdflush）会被唤醒，在后台将脏页写入磁盘。

而当脏页比例超过dirty\_ratio时，任何进行write操作的进程都将被阻塞，亲
自将系统中的一部分脏页写入磁盘，之后才能继续进行write操作。

即dirty\_ratio是阻塞式的，而dirty\_background\_ratio是非阻塞式的。正因
为如此，kernel中强制dirty\_background\_ratio小于等于dirty\_ratio的1/2，
来保证后台写脏页的操作被首先触发，而只有在必要的时候才会触发进程阻塞式
的刷脏页操作。

这两个值在默认的情况下为dirty\_ratio=40, dirty\_background\_ratio=10，
粗略一看的话，是不是如果我的系统有48G内存，只有在脏页数量超过
48*0.4=19.2G的时候才会导致进程阻塞呢？那这个情况发生的概率应该非常小
吧？？

但不幸的是，这里的总可用页数(total\_available\_pages)并不是实际的物理
内存，而是大致依照`total_available_pages=Cached+Free-Mapped`这个公式来
计算，其中Cached, Free, Mapped三个数值可以通过`cat /proc/meminfo`看到。

## 理论验证

首先，我们找了一台线上一直存在问题的机器，然后监视`/proc/meminfo`里面
的数据，结果发现在进行checkpoint的时候，Dirty有800多M，而
total\_available\_pages*dirty\_ratio只有400多M，说明线上确实出现了我
们担心的阻塞式写脏页问题。

然后，在线下的NDB测试环境中，跑个小程序，将几十G的文件mmap到内存中，并
且不断的对该文件进行扫描，发现meminfo中Mapped的数值在飞涨，直到满足
Dirty>total\_available\_pages*dirty\_ratio，然后等待checkpoint的发生。

结果在checkpoint的时候，NDB终于出现了大量几十秒的长耗时和queue full的
报错，理论得到验证。

## 总结

* Linux kernel中的刷脏页机制是有全局影响的，一个写io特别大的进程可能会
  导致系统中其他所有的进程被阻塞
* mmap是有风险的，因为这个会极大的增加Mapped的数量，进而导致
  total\_available\_pages降低，而这个会导致系统更加激进的向磁盘写入脏
  页
* BDB的checkpoint机制是十分坑爹的，由于会短时间内产生大量的随机磁盘写
  入，导致磁盘占用率保持在100%，这个时候其他被迫写入脏页的进程就会被长
  时间的阻塞
  
## 解决方案

* 加大内存，调大dirty\_ratio，调小dirty\_background\_ratio，来尽量避免
  阻塞式的写脏页
* 避免mmap过大的文件，这样看来MongoDB的内存使用策略可能也会比较坑爹






