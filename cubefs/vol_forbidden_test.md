# 测试文档

## 功能介绍

支持用户使用cli和http api去禁止某个副本卷的写入操作（包括数据写入和元数据修改）。

## 实现流程

### Master

* 在Master收到请求之后，会将卷的`Forbidden`属性设置为`true`。
* 如果卷的`Forbidden`属性为`true`，Master将禁止卷创建分区。
* 同时该属性将通过心跳发送给Datanode和Metanode。

### DataNode/MetaNode

* 在收到Master心跳后，遍历每一个分区。
* 判断分区的`VolName`是否在心跳的`ForbiddenVols`中，如果是则将分区的`Forbidden`属性设置为`true`否则设置为`false`。
* 如果分区的`Forbidden`为`true`，则对该分区的所有写入（除以下的例外操作之后）被拒绝。

**例外操作：**
* MetaPartition的底层事务操作（如`TxCommit`和`TxRollback`）不能被拒绝。

## 已有的测试

位于`master/apu_service_test.go`，函数名称为`TestForbiddenVolume`。

**对Mock的修改：**
* 修改了`MockDataServer`和`MockMetaServer`，使其能够在接收到心跳时修改对应的分区属性。
* 将所有的`MockDataServer`和`MockMetaServer`汇总到2个全局List中。

### 测试流程

* 创建测试用卷 `forbiddenVol`。
* 向Master发起请求禁用卷`forbiddenVol`。
* 等待所有`MockDataServer`的想关`MockDataPartition`的`Forbidden`被设置为`true`（超时300s）。
* 等待所有`MockMetaServer`的想关`MockMetaPartition`的`Forbidden`被设置为`true`（超时300s）。
* 向Master发起请求解除卷`forbiddenVol`的禁用状态。
* 等待所有`MockDataServer`的想关`MockDataPartition`的`Forbidden`被设置为`false`（超时300s）。
* 等待所有`MockMetaServer`的想关`MockMetaPartition`的`Forbidden`被设置为`false`（超时300s）。
* 删除测试用卷 `forbiddenVol`。
