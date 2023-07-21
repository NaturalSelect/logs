# Asynchronous Snapshot

>题目： curvefs/metaserver: Can metaserver's raft snapshot implementation be asynchronous
>
>Mentor：wu-hanqing
> 
>申请人 Github ID: NaturalSelect
>
>微信号： Natural_Sharp

## 需求分析

* 使快照保存（即`on_snapshot_save`函数），不阻止其他请求执行。

## 方案

* 为`ApplyQueue`的每一个partition都持久化一个`max applied index`。

**保存snapshot时：**
* 在rocksdb的`Flush`注册callback，当`Flush`完成时，完成save snapshot。

**在apply entires时：**
* 只有log index > max applied index的log会被apply。
* 每一次apply，都同时更新partition的`max applied index`为当前 log entires的index。
