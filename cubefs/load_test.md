# Node负载显示 测试文档

## 功能简介

支持在CLI中查询Node的负载（CpuUtil、IoUtil）。

## 实现流程

**DataNode：**
* 在心跳中返回IoUtil和CpuUtil。

***MetaNode：**
* 在心跳中返回CpuUtil。

**Master：**
* 通过返回的心跳更新自己的`DataNode`和`MetaNode`的对应字段。
* 在用户请求时将数据返回。

## 已有的测试

位于`util/loadutil/`下的多个`_test.go`中，函数名称为：
* `TestCpuUtil`
* `TestMemoryUsed`
* `TestGetPartition`
* `TestGetIoCounter`
* `TestMultiSample`

其中`TestCpuUtil`和`TestMemoryUsed`将检测当前CPU和内存的使用率，并打印。

`TestGetPartition`、`TestGetIoCounter`和`TestMultiSample`将获取根目标（/）的磁盘IO数据，并打印。

## 使用方法

**DataNode:**

```log
$ cfs-cli datanode info
```

**MetaNode:**

```log
$ cfs-cli metanode info
```
