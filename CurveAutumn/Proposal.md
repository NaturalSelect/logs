# Cluster Read-only

主要思路：
1. 在apply log entry的时候，查看`Pause`是否为`true`：
   * `true`  - 返回。
   * `false` - 继续执行。
2. 当遇到磁盘空间满时:
    * 设置Pause flag。
    * 排空ApplyQueue。
    * 将 `AppliedIndex` 设置为 `index - 1`。

```cpp
void CopysetNode::on_apply(::braft::Iterator &iter) {
    for (; iter.valid(); iter.next()) {
        // 放在bthread中异步执行，避免阻塞当前状态机的执行
        braft::AsyncClosureGuard doneGuard(iter.done());

        /**
         * 获取向braft提交任务时候传递的ChunkClosure，里面包含了
         * Op的所有上下文 ChunkOpRequest
         */
        braft::Closure *closure = iter.done();

        if (nullptr != closure) {
            /**
             * 1.closure不是null，那么说明当前节点正常，直接从内存中拿到Op
             * context进行apply
             */
            ChunkClosure
                *chunkClosure = dynamic_cast<ChunkClosure *>(iter.done());
            CHECK(nullptr != chunkClosure)
                << "ChunkClosure dynamic cast failed";
            std::shared_ptr<ChunkOpRequest>& opRequest = chunkClosure->request_;
            concurrentapply_->Push(opRequest->ChunkId(), ChunkOpRequest::Schedule(opRequest->OpType()),  // NOLINT
                                   &ChunkOpRequest::OnApply, opRequest,
                                   iter.index(), doneGuard.release());
        } else {
            // 获取log entry
            butil::IOBuf log = iter.data();
            /**
             * 2.closure是null，有两种情况：
             * 2.1. 节点重启，回放apply，这里会将Op log entry进行反序列化，
             * 然后获取Op信息进行apply
             * 2.2. follower apply
             */
            ChunkRequest request;
            butil::IOBuf data;
            auto opReq = ChunkOpRequest::Decode(log, &request, &data,
                                                iter.index(), GetLeaderId());
            auto chunkId = request.chunkid();
            concurrentapply_->Push(chunkId, ChunkOpRequest::Schedule(request.optype()),  // NOLINT
                                   &ChunkOpRequest::OnApplyFromLog, opReq,
                                   dataStore_, std::move(request), data);
        }
    }
}
```

```cpp
void WriteChunkRequest::OnApply(uint64_t index,
                                ::google::protobuf::Closure *done) {
    brpc::ClosureGuard doneGuard(done);
    uint32_t cost;

    std::string  cloneSourceLocation;
    if (existCloneInfo(request_)) {
        auto func = ::curve::common::LocationOperator::GenerateCurveLocation;
        cloneSourceLocation =  func(request_->clonefilesource(),
                            request_->clonefileoffset());
    }

    auto ret = datastore_->WriteChunk(request_->chunkid(),
                                      request_->sn(),
                                      cntl_->request_attachment(),
                                      request_->offset(),
                                      request_->size(),
                                      &cost,
                                      cloneSourceLocation);

    if (CSErrorCode::Success == ret) {
        response_->set_status(CHUNK_OP_STATUS::CHUNK_OP_STATUS_SUCCESS);
        node_->UpdateAppliedIndex(index);
    } else if (CSErrorCode::BackwardRequestError == ret) {
        // 打快照那一刻是有可能出现旧版本的请求
        // 返回错误给客户端，让客户端带新版本来重试
        LOG(WARNING) << "write failed: "
                     << " data store return: " << ret
                     << ", request: " << request_->ShortDebugString();
        response_->set_status(
            CHUNK_OP_STATUS::CHUNK_OP_STATUS_BACKWARD);
    } else if (CSErrorCode::InternalError == ret ||
               CSErrorCode::CrcCheckError == ret ||
               CSErrorCode::FileFormatError == ret) {
        /**
         * internalerror一般是磁盘错误,为了防止副本不一致,让进程退出
         * TODO(yyk): 当前遇到write错误直接fatal退出整个
         * ChunkServer后期考虑仅仅标坏这个copyset，保证较好的可用性
        */
        LOG(FATAL) << "write failed: "
                   << " data store return: " << ret
                   << ", request: " << request_->ShortDebugString();
    } else {
        LOG(ERROR) << "write failed: "
                   << " data store return: " << ret
                   << ", request: " << request_->ShortDebugString();
        response_->set_status(
            CHUNK_OP_STATUS::CHUNK_OP_STATUS_FAILURE_UNKNOWN);
    }

    response_->set_appliedindex(MaxAppliedIndex(node_, index));
    node_->ShipToSync(request_->chunkid());
}

void WriteChunkRequest::OnApplyFromLog(std::shared_ptr<CSDataStore> datastore,
                                       const ChunkRequest &request,
                                       const butil::IOBuf &data) {
    // NOTE: 处理过程中优先使用参数传入的datastore/request
    uint32_t cost;
    std::string  cloneSourceLocation;
    if (existCloneInfo(&request)) {
        auto func = ::curve::common::LocationOperator::GenerateCurveLocation;
        cloneSourceLocation =  func(request.clonefilesource(),
                            request.clonefileoffset());
    }

    auto ret = datastore->WriteChunk(request.chunkid(),
                                     request.sn(),
                                     data,
                                     request.offset(),
                                     request.size(),
                                     &cost,
                                     cloneSourceLocation);
     if (CSErrorCode::Success == ret) {
         return;
     } else if (CSErrorCode::BackwardRequestError == ret) {
        LOG(WARNING) << "write failed: "
                     << " data store return: " << ret
                     << ", request: " << request.ShortDebugString();
    } else if (CSErrorCode::InternalError == ret ||
               CSErrorCode::CrcCheckError == ret ||
               CSErrorCode::FileFormatError == ret) {
        LOG(FATAL) << "write failed: "
                   << " data store return: " << ret
                   << ", request: " << request.ShortDebugString();
    } else {
        LOG(ERROR) << "write failed: "
                   << " data store return: " << ret
                   << ", request: " << request.ShortDebugString();
    }
}
```

```cpp
void CopysetNode::on_error(const ::braft::Error &e) {
    LOG(FATAL) << "Copyset: " << GroupIdString()
               << ", peer id: " << peerId_.to_string()
               << " meet raft error: " << e;
}
```
