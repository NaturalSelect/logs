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

### 为每一个partition都持久化一个`max applied index`。

```cpp
enum KEY_TYPE : unsigned char {
    kTypeInode = 1,
    kTypeS3ChunkInfo = 2,
    kTypeDentry = 3,
    kTypeVolumeExtent = 4,
    kTypeInodeAuxInfo = 5,
    kTypeBlockGroup = 6,
    kTypeDeallocatableBlockGroup = 7,
    kTypeDeallocatableInode = 8,

    // applied key type
    kTypeAppliedInde = 9
};

class NameGenerator {
 public:
    ...

    // return applied key of partition
    std::string GetAppliedIndexKeyName() const;

    static size_t GetFixedLength();
 private:
    ...
    std::string keyName4AppliedIndex_;
};
```

```proto
message AppliedIndex {
    required uint64 index = 1;
}
```

### 在每一个partition记录`applied index`和它的key。

```cpp
class Partition {
 public:
    // max applied index of current partition
    std::uint64_t appliedIndex_;
    std::string appliedIndexKey_;
};
```

同时在partition初始化时，检查这个key是否存在。

```cpp
// load applied index
// for compatibility, we initialize x to 0
this->appliedIndex_ = 0;
std::string appliedKey = nameGen_->GetAppliedIndexKeyName();
// check applied index key
AppliedIndex index;
auto result = kvStorage->SGet(appliedKey,"",&index);
if(result.ok()) {
    this->appliedIndex_ = index.index();
}
this->appliedIndexKey_ = std::move(appliedKey);
```

### 在执行`Partition`的函数时，进行Log Entires的过滤

```cpp
MetaStatusCode Partition::CreateDentry(const Dentry& dentry,std::uint64_t index) {
    if (index <= appliedIndex_) {
        return MetaStatusCode::OK;
    }
    PRECHECK(dentry.fsid(), dentry.parentinodeid());
    ...
}
```

### 使用`Transaction`在修改kvstorage的同时修改`applied index`

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

MetaStatusCode DentryManager::CreateDentry(const Dentry& dentry,const std::string &appliedIndexKey,std::uint64_t index) {
    Log4Dentry("CreateDentry", dentry);
    MetaStatusCode rc = dentryStorage_->Insert(dentry,appliedIndexKey,index);
    Log4Code("CreateDentry", rc);
    return rc;
}

MetaStatusCode Partition::CreateDentry(const Dentry& dentry,std::uint64_t logIndex) {
    // TODO(me): check applied index
    PRECHECK(dentry.fsid(), dentry.parentinodeid());
    MetaStatusCode ret = dentryManager_->CreateDentry(dentry,appliedIndexKey_,logIndex);
    ...
}

MetaStatusCode MetaStoreImpl::CreateDentry(const CreateDentryRequest *request,
                                           CreateDentryResponse *response,std::uint64_t logIndex) {
    ReadLockGuard readLockGuard(rwLock_);
    std::shared_ptr<Partition> partition;
    GET_PARTITION_OR_RETURN(partition);

    MetaStatusCode status = partition->CreateDentry(request->dentry().logIndex);
    response->set_statuscode(status);
    return status;
}

#define OPERATOR_ON_APPLY(TYPE)                                                \
    void TYPE##Operator::OnApply(int64_t index,                                \
                                 google::protobuf::Closure *done,              \
                                 uint64_t startTimeUs) {                       \
        brpc::ClosureGuard doneGuard(done);                                    \
        uint64_t timeUs = TimeUtility::GetTimeofDayUs();                       \
        node_->GetMetric()->WaitInQueueLatency(OperatorType::TYPE,             \
                                               timeUs - startTimeUs);          \
        auto status = node_->GetMetaStore()->TYPE(                             \
            static_cast<const TYPE##Request *>(request_),                      \
            static_cast<TYPE##Response *>(response_),index);                   \
        uint64_t executeTime = TimeUtility::GetTimeofDayUs() - timeUs;         \
        node_->GetMetric()->ExecuteLatency(OperatorType::TYPE, executeTime);   \
        if (status == MetaStatusCode::OK) {                                    \
            node_->UpdateAppliedIndex(index);                                  \
            static_cast<TYPE##Response *>(response_)->set_appliedindex(        \
                std::max<uint64_t>(index, node_->GetAppliedIndex()));          \
            node_->GetMetric()->OnOperatorComplete(                            \
                OperatorType::TYPE,                                            \
                TimeUtility::GetTimeofDayUs() - startTimeUs, true);            \
        } else {                                                               \
            node_->GetMetric()->OnOperatorComplete(                            \
                OperatorType::TYPE,                                            \
                TimeUtility::GetTimeofDayUs() - startTimeUs, false);           \
        }                                                                      \
    }

```

### 保存Snapshot

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
