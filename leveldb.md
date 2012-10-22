# Leveldb Internal (Write)

## Overview

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

## MakeRoomForWrite()

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

# BuildBatchGroup()

因为leveldb中所有的写操作都是append，所以很容易的一个优化就是在处理请
求队列中队首的请求时，可以将其之后的若干个请求一起合并为一个较大的批量请求，
这样能够减少请求的次数，从而提升写性能。

`BuildBatchGroup()`的实现还是比较简单的，就是依次向后合并。然后能合并
出的批量请求大小有个上限（1MB），不过当队首的请求比较小的时候（小于
128K），这个上限也会相应的比较小（队首请求大小+128K）。

# AddRecord()

这个函数的功能基本就是将待写入的数据按照某种格式append到log文件的尾部，
具体实现之后我们在分析log文件的格式的时候再做分析。

# InsertInto()

这个函数就是将待写入的数据依次插入memtable中，memtable的内部实现就是一
个skiplist。
