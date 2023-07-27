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

在curve中数据将被保存到`MetaFStream`和`rocksdb`中。

数据的保存存在三种情况：
* 只涉及到`MetaFStream`。
* 只涉及到`rocksdb`。
* 同时涉及到两者。

对于每一个`Partition`而言，我们维护：
* 一个`rocksdb applied index`，存储在rocksdb中。
* 一个`MetaFStream applied index`存储在`MetaFStream`中。

同时维护：
* 一个全局`MetaFStream applied index`，存储在`MetaFStream`中。

`rocksdb`中每一个`Partition`的`applied index`的`key`使用`NameGenerator`产生：

```cpp
enum KEY_TYPE : unsigned char {
    ...
    kTypeAppliedIndex = 9
};

class NameGenerator {
 public:
    ...
    std::string GetAppliedIndexTableName() const;
 private:
    ...
};
```

其`applied index`定义如下：

```proto
message AppliedIndex {
    required uint64 index = 1;
}
```

`MetaFStream`中每一个`Partition`的`applied index`，序列化在`PartitionInfo`中。

```proto
message PartitionInfo {
    ...
    optional uint64 appliedIndex = 14; // partition applied index
}
```

在`Partition`初始化时，读取两者的值并把其缓存到内存中：

```cpp
class Partition {
    ...
    // max applied index of current partition
    std::uint64_t appliedIndex_;
    std::string appliedIndexTableName_;

    static const char *appliedIndexKey_ = "partition";
};

Partition::Partition(PartitionInfo partition,
                     std::shared_ptr<KVStorage> kvStorage,
                     bool startCompact, bool startVolumeDeallocate) {
    ...
    // load applied index
    // for compatibility, we initialize x to 0
    this->appliedIndex_ = 0;
    std::string appliedKey = nameGen_->GetAppliedIndexTableName();
    // check applied index key
    AppliedIndex index;
    auto result = kvStorage->SGet(appliedKey,appliedIndexKey,&index);
    if(result.ok()) {
        this->appliedIndex_ = index.index();
    }
    this->appliedIndexTableName_ = std::move(appliedKey);
    // load applied index from MetaFStream
    this->metaAppliedIndex_ = partition.appliedIndex();
    ...
}
```

在`MetaStorage`初始化时，加载`MetaStorage`的`applied index`。

```cpp
bool MetaStoreImpl::Load(const std::string &pathname) {
    // Load from raft snap file to memory
    WriteLockGuard writeLockGuard(rwLock_);
    ...

    // set the applied index
    this->appliedIndex_ = fstream.GetAppliedIndex();

    ...
}
```

在执行`MetaStorage`的操作时，必须传入一个`iter.index()`。

```cpp
#define OPERATOR_ON_APPLY(TYPE)                                                \
    void TYPE##Operator::OnApply(int64_t index,                                \
        ...
        auto status = node_->GetMetaStore()->TYPE(                             \
            static_cast<const TYPE##Request *>(request_),                      \
            static_cast<TYPE##Response *>(response_),index);                   \
        ...
```

## 针对只涉及到`rocksdb`的`MetaOperation`

*以`CreateDentry`为例。*

在执行`Partition`的函数时，进行Log Entires的过滤。

```cpp
MetaStatusCode MetaStoreImpl::CreateDentry(const CreateDentryRequest *request,
                                           CreateDentryResponse *response,std::uint64_t logIndex) {
    ReadLockGuard readLockGuard(rwLock_);
    std::shared_ptr<Partition> partition;
    GET_PARTITION_OR_RETURN(partition);

    MetaStatusCode status = partition->CreateDentry(request->dentry().logIndex);
    response->set_statuscode(status);
    return status;
}
```

```cpp
MetaStatusCode Partition::CreateDentry(const Dentry& dentry,std::uint64_t logIndex) {
    PRECHECK(dentry.fsid(), dentry.parentinodeid());
    if (index <= appliedIndex_) {
        return MetaStatusCode::OK;
    }
    MetaStatusCode ret = dentryManager_->CreateDentry(dentry,appliedIndexKey_,logIndex);
    ...
}
```

```cpp
MetaStatusCode DentryManager::CreateDentry(const Dentry& dentry,const std::string &appliedIndexKey,std::uint64_t index) {
    Log4Dentry("CreateDentry", dentry);
    MetaStatusCode rc = dentryStorage_->Insert(dentry,appliedIndexKey,index);
    Log4Code("CreateDentry", rc);
    return rc;
}
```

```cpp
MetaStatusCode DentryStorage::Insert(const Dentry& dentry,const std::string &appliedIndexKey,std::uint64_t index) {
    WriteLockGuard lg(rwLock_);

    Dentry out;
    DentryVec vec;
    MetaStatusCode rc = Find(dentry, &out, &vec, true);
    if (rc == MetaStatusCode::OK) {
        if (BelongSomeOne(out, dentry)) {
            return MetaStatusCode::IDEMPOTENCE_OK;
        }
        return MetaStatusCode::DENTRY_EXIST;
    } else if (rc != MetaStatusCode::NOT_FOUND) {
        return MetaStatusCode::STORAGE_INTERNAL_ERROR;
    }
    DentryVector vector(&vec);
    vector.Insert(dentry);
    std::string skey = DentryKey(dentry);
    auto txn = kvStorage_->BeginTransaction();
    Status s = txn->SSet(table4Dentry_, skey, vec);
    if (!s.ok()) {
        txn->Rollback();
        LOG(ERROR) << "Insert dentry failed, status = " << s.ToString();
        return MetaStatusCode::STORAGE_INTERNAL_ERROR;
    }
    AppliedIndex indexVal;
    indexVal.set_index(index);
    s = txn->SSet(appliedIndexKey,"",indexVal);
    if (!s.ok()) {
        txn->Rollback();
        LOG(ERROR) << "Insert dentry failed, status = " << s.ToString();
        return MetaStatusCode::STORAGE_INTERNAL_ERROR;
    }
    s = txn->Commit();
     if (!s.ok()) {
        LOG(ERROR) << "Insert dentry failed, status = " << s.ToString();
        return MetaStatusCode::STORAGE_INTERNAL_ERROR;
    }
    vector.Confirm(&nDentry_);
    return MetaStatusCode::OK;
}
```

## 针对只涉及到`MetaFStream`的`MetaOperation`

*以`CreatePartition`为例。*

```cpp
MetaStatusCode
MetaStoreImpl::CreatePartition(const CreatePartitionRequest *request,
                               CreatePartitionResponse *response,std::uint64_t logIndex) {
    WriteLockGuard writeLockGuard(rwLock_);
    MetaStatusCode status;
    // check index
    if(this->appliedIndex >= logIndex) {
        status = MetaStatusCode::OK;
        response->set_statuscode(status);
        return status;
    }
    ...
}
```

## 针对涉及到两者的`MetaOperation`

*以`PrepareRenameTx`为例。*

```cpp
MetaStatusCode
MetaStoreImpl::PrepareRenameTx(const PrepareRenameTxRequest *request,
                               PrepareRenameTxResponse *response,std::uint64_t logIndex) {
    ReadLockGuard readLockGuard(rwLock_);
    MetaStatusCode rc;
    std::shared_ptr<Partition> partition;
    GET_PARTITION_OR_RETURN(partition);
    std::vector<Dentry> dentrys{request->dentrys().begin(),
                                request->dentrys().end()};
    rc = partition->HandleRenameTx(dentrys);
    response->set_statuscode(rc);
    return rc;
}
```

### Snapshot的保存

```cpp
bool RocksDBStorage::Checkpoint(const std::string& dir,
                                std::vector<std::string>* files) {
    rocksdb::FlushOptions options;
    options.wait = true;
    // we not allow this write stall due to flush
    options.allow_write_stall = false;
    auto status = db_->Flush(options, handles_);
    if (!status.ok()) {
        LOG(ERROR) << "Failed to flush DB, " << status.ToString();
        return false;
    }

    ...
}
```

```cpp
bool MetaStoreImpl::Save(const std::string &dir,
                         OnSnapshotSaveDoneClosure *done) {
    // lock is unnecessary
    ...
}
```

```cpp
void CopysetNode::on_snapshot_save(braft::SnapshotWriter* writer,
                                   braft::Closure* done) {
    LOG(INFO) << "Copyset " << name_ << " saving snapshot to '"
              << writer->get_path() << "'";

    brpc::ClosureGuard doneGuard(done);

    auto metricCtx = RaftSnapshotMetric::GetInstance().OnSnapshotSaveStart();
    auto cleanMetricIfFailed = absl::MakeCleanup([metricCtx]() {
        metricCtx->success = false;
        RaftSnapshotMetric::GetInstance().OnSnapshotSaveDone(metricCtx);
    });

    // flush all flying operators
    applyQueue_->Flush();

    // save conf
    std::string confFile = writer->get_path() + "/" + kConfEpochFilename;
    if (0 != SaveConfEpoch(confFile)) {
        LOG(ERROR) << "Copyset " << name_
                   << " save snapshot failed, save epoch file failed";
        done->status().set_error(errno, "Save conf file failed, error = %s",
                                 strerror(errno));
        return;
    }

    writer->add_file(kConfEpochFilename);

    // use async thread to do this
    metaStore_->Save(writer->get_path(), new OnSnapshotSaveDoneClosureImpl(
                                             this, writer, done, metricCtx));
    doneGuard.release();

    // `Cancel` only available for rvalue
    std::move(cleanMetricIfFailed).Cancel();
}
```
