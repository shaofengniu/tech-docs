# Leveldb Internal

## Write

所有的writer都首先会进入请求队列`DbImple::writers_`中，只有自己在队首
的时候才会获得执行的机会。

```c++
  writers_.push_back(&w);
  while (!w.done && &w != writers_.front()) {
    w.cv.Wait();
  }
```

接下来，调用`DBImple::MakeRoomForWrite()`来完成切换log文件，memtable或
者触发compaction的工作。

然后，或者执行机会的writer会通过`DBImpl::BuildBatchGroup()`来将排在它
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



