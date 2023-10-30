
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
