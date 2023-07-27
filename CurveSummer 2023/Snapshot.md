# Asynchronous Snapshot Scheme

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

具体方案：
* 维护一个`Partition`的`applied index`，存储在`rocksdb`中。
* 对于修改`rocksdb`的操作必须检查`applied index`，如果修改过就跳过。
* 对于修改in-memory 数据结构的操作（例如修改`Partition`的Table和`PendingTx`），则不需要check。

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

在`Partition`初始化时，读取`applied index`并把其缓存到内存中：

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

*以`CreatePartition`为例（目前只有对in-memory的partition map的修改是只涉及到`MetaFStream`的）。*

不需要修改任何代码，让其replay即可。

```cpp
MetaStatusCode
MetaStoreImpl::CreatePartition(const CreatePartitionRequest *request,
                               CreatePartitionResponse *response,std::uint64_t logIndex) {
    ...
}
```

## 针对涉及到两者的`MetaOperation`

*以`PrepareRenameTx`为例（目前这样`PrepareRenameTx`是涉及到两者的）。*

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

```cpp
MetaStatusCode Partition::HandleRenameTx(const std::vector<Dentry>& dentrys
                                        ,std::uint64_t logIndex) {
    for (const auto &it : dentrys) {
        PRECHECK(it.fsid(), it.parentinodeid());
    }
    return dentryManager_->HandleRenameTx(dentrys,logIndex,this->appliedIndex);
}
```

```cpp
MetaStatusCode DentryManager::HandleRenameTx(
    const std::vector<Dentry>& dentrys,std::uint64_t logIndex
    ,std::uint64_t appliedIndex) {
    for (const auto& dentry : dentrys) {
        Log4Dentry("HandleRenameTx", dentry);
    }
    auto rc = txManager_->HandleRenameTx(dentrys,logIndex,appliedIndex);
    Log4Code("HandleRenameTx", rc);
    return rc;
}
```

```cpp
MetaStatusCode TxManager::HandleRenameTx(const std::vector<Dentry>& dentrys,std::uint64_t logIndex,std::uint64_t appliedIndex) {
    ...
    // Prepare for TX
    auto renameTx = RenameTx(dentrys, storage_);

    // set the log index to renameTx
    // we will check it on dentry storage
    renameTx.SetLogIndex(logIndex);
    // set the applied index of partition
    renameTx.SetAppliedIndex(appliedIndex);

    if (!InsertPendingTx(renameTx)) {
        LOG(ERROR) << "InsertPendingTx failed, renameTx: " << renameTx;
        return MetaStatusCode::HANDLE_TX_FAILED;
    } else if (!renameTx.Prepare()) {
        LOG(ERROR) << "Prepare for RenameTx failed, renameTx: " << renameTx;
        return MetaStatusCode::HANDLE_TX_FAILED;
    }

    return MetaStatusCode::OK;
}
```

```cpp
#define FOR_EACH_DENTRY(action) \
do { \
    for (const auto& dentry : dentrys_) { \
        auto rc = storage_->HandleTx( \
            DentryStorage::TX_OP_TYPE::action, dentry,this->logIndex_,this->appliedIndex_); \
        if (rc != MetaStatusCode::OK) { \
            return false; \
        } \
    } \
} while (0)

bool RenameTx::Prepare() {
    FOR_EACH_DENTRY(PREPARE);
    return true;
}
```

当`logIndex`小于等于`applied index`时，事务的`Prepare`操作已经在`rocksdb`落盘，直接返回`OK`。

```cpp
MetaStatusCode DentryStorage::HandleTx(TX_OP_TYPE type, const Dentry& dentry
                                        ,std::uint64_t logIndex,std::uint64_t appliedIndex) {
    WriteLockGuard lg(rwLock_);
    Status s;
    Dentry out;
    DentryVec vec;
    DentryVector vector(&vec);
    std::string skey = DentryKey(dentry);
    MetaStatusCode rc = MetaStatusCode::OK;
    // if log index is older than applied index
    // we assume this tx operation is success
    if(logIndex <= appliedIndex) {
        return rc;
    }
    ...
}
```

### Snapshot的保存

```cpp
void CopysetNode::on_snapshot_save(braft::SnapshotWriter* writer,
                                   braft::Closure* done) {
    ...

    // we will start a thread or use thread pool in here
    metaStore_->Save(writer->get_path(), new OnSnapshotSaveDoneClosureImpl(
                                             this, writer, done, metricCtx));
    ...
}
```

```cpp
bool MetaStoreImpl::Save(const std::string &dir,
                         OnSnapshotSaveDoneClosure *done) {
    brpc::ClosureGuard doneGuard(done);

    // we hold the write lock in this scope
    {
        WriteLockGuard writeLockGuard(rwLock_);
        MetaStoreFStream fstream(&partitionMap_, kvStorage_,
                             copysetNode_->GetPoolId(),
                             copysetNode_->GetCopysetId());

        const std::string metadata = dir + "/" + kMetaDataFilename;
        if (!fstream.Save(metadata)) {
            done->SetError(MetaStatusCode::SAVE_META_FAIL);
            return false;
        }
    }
    ...
}
```

```cpp
bool RocksDBStorage::Checkpoint(const std::string& dir,
                                std::vector<std::string>* files) {
    rocksdb::FlushOptions options;
    options.wait = true;
    // we not allow this write stall
    options.allow_write_stall = false;
    auto status = db_->Flush(options, handles_);
    if (!status.ok()) {
        LOG(ERROR) << "Failed to flush DB, " << status.ToString();
        return false;
    }
    ...
}
```