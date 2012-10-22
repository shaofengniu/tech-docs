# Leveldb Internal

## 扯在前面

leveldb的身世我就不多扯了，只提一下与其相关的两篇paper：

* Bigtable: A Distributed Storage System for Structured Data
* The Log-Structured Merge-Tree (LSM-Tree)

我们都知道，影响存储引擎性能的最大因素就是IO，其中读请求能通过将数
据缓存在访问速度更快的存储介质中来提升性能，而出于数据安全性的考虑，写
请求通常不会使用缓存，而是直接写入磁盘等持久化存储介质中，但这样也就
带来了巨大的性能问题。

由于磁盘的顺序写性能要远远大于其随机写性能，所以存储引擎对写性能的优化
通常采取的办法都是想办法将随机写转化为顺序写，只在空闲的情况下才进行随
机写。例如基于B+Tree的引擎，都是通过频繁的追加写transaction log来保证
数据安全性，而只在每间隔一段时间才进行一次随机写。

虽然通过写log可以降低随机写的频率，但是这种方式还是无法完全避免随机写。
所以近几年就出现了完全消除随机写的底层数据存储格式，如Append Only
Btree(CouchDB)，LSM-Tree(leveldb)，bitcask(Riak)。由于本文是写leveldb
的，所以我就不在此处对append only btree和bitcask的原理多做讨论，有兴趣
的可以自行搜索之。。。

leveldb中所使用的LSM-Tree的基本原理很简单，就是将要写入的数据直接
append到log文件，然后为了便于查找，同时在内存中维护一份一致的查找表
（memtable）。 但这样做有一个问题，就是即使我的实际数据量是有限的，这
个log文件也会无限增长下去。。。。

要解决这个问题的话有一个方法，就是每隔一段时间就将memtable中的数据dump
到另外一个文件（level 0 sstable），作为数据的永久存储，这样之前的log文
件就可以被删除掉了。由于memtable中的数据是有序的，所以导出的sstable中
的数据也是有序的，这就使查找也比较方便。

按照上面的方法运行了一段时间后暴露出另外一个问题，sstable越来越多了，
其中大部分还都是重复数据。。。。怎么办？合并一下呗！只需要把所有的
sstable按照时间顺序读进内存，使用新数据覆盖旧数据之后，导出来的就是
不包含重复数据的sstable(level 1 sstable)了，并且merge之前的sstable也就
可以删掉了。这样，通过每隔一段时间就对level 0中的sstable进行一次合并，
我们就能够保证level 0中的sstable数量是可控的。

这样，又平安的运行了一段时间后，发现level 1中的sstable数量也在疯狂增
长。。。怎么办？继续合并！将一部分level 1中的sstable与level 2中的
sstable合并后放入level 2即可。这样，用同样的方法我们可以保证level 2,
3,..., N中的sstable数量可控。

至此，一个LSM-Tree的雏形就算诞生了，其写操作只需要一次磁盘访问，而读
操作则可能需要log(N)次磁盘访问，即牺牲了部分读性能来换取更高的写性能。

## 写请求的处理

由于leveldb中写请求的处理最为简单，所以我们就先捡软柿子捏一捏。。。

leveldb中的写请求有三种：

* `DB::Put()`, 单个的写请求
* `DB::Delete()`, 单个的删除请求
* `DB::Write()`, 批量的写、删除请求

但其实三者背后都是由`DB::Write()`实现的，所以我们就来看一下这个函数具
体是如何实现的。

### `DBImpl::Write()` @db_impl.cc

所有的writer都首先会进入请求队列`DbImple::writers_`中，只有自己在队首
的时候才会获得执行的机会。

```c++
writers_.push_back(&w);
while (!w.done && &w != writers_.front()) {
  w.cv.Wait();
}
```

通过这个写请求队列可以保证，同时最多只可能有一个writer进行下面的流程。

接下来，writer调用`DBImple::MakeRoomForWrite()`来完成切换log文件，
memtable或者触发compaction的工作。 

然后，获得执行机会的writer会通过`DBImpl::BuildBatchGroup()`来将排在它
后面的一部分写请求合并起来，这样可以将若干个小请求合并为一个比较大的，
提高写效率。

接下来就到了真正要写入的时候了，writer会首先调用`log_->AddRecord()`来
将待数据追加到log文件的尾部，然后调用
`WriteBatchInternal::InsertInto()`将数据插入到memtable中。

由于这个写操作中合并了后面若干个writer的数据，所以为了让这些writer不会
在重新执行一遍上述过程，我们需要将其标记为已完成，并且唤醒这些writer。

```c++
while (true) {
  Writer* ready = writers_.front();
  writers_.pop_front();
  if (ready != &w) {
    ready->status = status;
    ready->done = true;
    ready->cv.Signal();
}
```

这些writer在被唤醒后，会发现自己已经处在完成状态，然后就可以直接返回操
作结果。

最后，在这个writer返回之前，需要将队列中的下一个writer唤醒。

### `DBImple::MakeRoomForWrite()` @db_imple.cc

这个函数首先会检查level 0中的文件数量是否即将达到硬性上限，如果是的话，
就让当前的writer休眠1ms，在降低写入速度的同时，腾出一些cpu资源给负责
compaction的线程使用，尽快的减少level 0中的文件数量，但是每个writer最
多只会被延迟一次。

```c++
if (allow_delay &&
  versions_->NumLevelFiles(0) >= config::kL0_SlowdownWritesTrigger) {
    ...
    env_->SleepForMicroseconds(1000);
    ...
}
```

接下来会检查当前的memtable中是否有剩余空间，如果有的话，就不需要再进行
额外的工作，直接返回即可，否则就说明我们需要切换当前的memtable及log了。

```c++
if (!force &&
  (mem_->ApproximateMemoryUsage() <= options_.write_buffer_size)) {
    break;
}
```

在切换memtable和log之前，还需要检查当前前一个memtable是不是正在进行
compaction，如果是的话，就需要等待他的完成。

```c++
if (imm_ != NULL) {
  bg_cv_.Wait();
}
```

然后，还需要再次检查level 0的文件数量，是否达到了硬性上限，如果是的话，
就需要进行等待，直到后台负责compaction的线程将level 0的文件数量降到阈
值以下。

```c++
if (versions_->NumLevelFiles(0) >= config::kL0_StopWritesTrigger) {
  bg_cv_.Wait();
}
```    

通过以上所有的检查之后，才是真正进行memtable和log切换的部分。

# `DBImple::BuildBatchGroup()` @db_imple.cc

因为leveldb中所有的写操作都是append，所以很容易的一个优化就是在处理请
求队列中队首的请求时，可以将其之后的若干个请求一起合并为一个较大的批量请求，
这样能够减少请求的次数，从而提升写性能。

`BuildBatchGroup()`的实现还是比较简单的，就是依次向后合并。然后能合并
出的批量请求大小有个上限（1MB），不过当队首的请求比较小的时候（小于
128K），这个上限也会相应的比较小（队首请求大小+128K）。

