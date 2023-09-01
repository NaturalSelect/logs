# Nodeset/Node选择策略 测试文档

## 功能简介

提供四种不同的Nodeset/Node选择策略（`RoundRobin`、`CarryWeight`、`Ticket`、`AvailableSpaceFirst`）。

以及不同维度的配置：
* Nodeset的Node选择策略配置。
* Zone的Nodeset选择策略配置。

同时提供粗粒度接口：
* Cluster的Nodeset/Node选择策略配置。
* Zone的Node选择策略配置。

## 实现流程

四种策略的实现流程见：[文档](/cubefs/selector.md)

**创建分区流程：**
* 在`Zone`中选择`Nodeset`（由`Zone`的`DataNodesetSelector`和`MetaNodesetSelector`负责）。
* 在上述步骤中得到的`Nodeset`选择`Node`（由`Nodeset`的`DataNodeSelector`和`MetaNodeSelector`负责）。

### Nodeset的Node选择策略配置

* Master接收配置请求。
* 在对应的`nodeset`中设置`DataNodeSelector`和`MetaNodeSelector`字段。
* 并将它们同步到raft。

### Zone的Nodeset选择策略配置

* Master接收配置请求。
* 在对应的`zone`中设置`DataNodesetSelector`和`MetaNodesetSelector`字段。
* 并将它们同步到raft。

### Cluster的Nodeset/Node选择策略配置

* Master接收配置请求。
* 遍历所有的`Zone`。
* 对于每一个`Zone`，设置它的`DataNodesetSelector`和`MetaNodesetSelector`字段，并同步到raft。
* 然后设置`Zone`的所有下层`Nodeset`的`DataNodeSelector`和`MetaNodeSelector`字段，并同步到raft。

*NOTE：此处raft同步并不是原子的，每修改一项就会同步一次。*

### Zone的Node选择策略配置

* Master接收配置请求。
* 遍历对应`Zone`的所有下层`Nodeset`的`DataNodeSelector`和`MetaNodeSelector`字段，并同步到raft。

*NOTE：此处raft同步并不是原子的，每修改一项就会同步一次。*

## 已有的测试

配置接口的测试位于`master/api_service_test.go`，函数名称为：
* `TestUpdateClusterNodeSelector`。
* `TestUpdateClusterNodesetSelector`。
* `TestUpdateZoneNodesetSelector`。
* `TestUpdateZoneNodeSelector`。
* `TestUpdateNodesetNodeSelector`。

### TestUpdateClusterNodeSelector

* 调用master的接口，将策略设置为`Ticket`。
* 选取`testZone1`进行检测。
* 遍历它的`Nodeset`的所有`NodeSelector`查看是否符合预期。
* 调用master的接口，将策略设置为`CarryWeight`（原策略）。

### TestUpdateClusterNodesetSelector

* 调用master的接口，将策略设置为`CarryWeight`。
* 选取`testZone1`进行检测。
* 查看它的`NodesetSelector`查看是否符合预期。
* 调用master的接口，将策略设置为`RoundRobin`（原策略）。

### TestUpdateZoneNodesetSelector

* 调用master的接口，将`testZone2`的策略设置为`CarryWeight`。
* 查看它的`NodesetSelector`查看是否符合预期。
* 调用master的接口，将策略设置为`RoundRobin`（原策略）。

### TestUpdateZoneNodeSelector

* 调用master的接口，将`testZone2`的策略设置为`Ticket`。
* 查看它的`NodesetSelector`查看是否符合预期。
* 调用master的接口，将策略设置为`CarryWeight`（原策略）。

### TestUpdateNodesetNodeSelector

* 调用master的接口，将`testZone2`的第一个Nodeset的策略设置为`CarryWeight`。
* 查看它的`NodeSelector`查看是否符合预期。
* 调用master的接口，将策略设置为`RoundRobin`（原策略）。

### Nodeset选择策略正确性测试

位于`master/nodeset_selector_test.go`，函数名为：

* `TestRoundRobinNodesetSelector`
* `TestCarryWeightNodesetSelector`
* `TestTicketNodesetSelector`
* `TestAvailableSpaceFirstNodesetSelector`

流程大体上一致：
* 使用`testZone2`的Nodeset进行测试。
* 判断结果是否在`testZone2`的nodeset map中。

### Nodeset选择策略效果测试

位于`master/nodeset_selector_test.go`，函数名为：
* `TestBenchmarkCarryWeightNodesetSelector`
* `TestBenchmarkTicketNodesetSelector`

流程大体上一致：
* 生成4个容量呈现阶梯状的Nodeset。
* 用对应的策略循环select 100次，并将结果记录在表中（每选择一次都随机减少0 ~ 10 GB的容量）。
* 将结果表输出。

*NOTE：目前这个测试需要人工判断。*

在正常情况下：
* `CarryWeight`的结果呈现出明显的阶梯状。
* `Ticket`的结果也呈现出阶梯状，但是随机性更大。

### Node选择策略正确性测试

位于`master/node_selector_test.go`，函数名为：

* `TestRoundRobinNodeSelector`
* `TestCarryWeightNodeSelector`
* `TestTicketNodeSelector`
* `TestAvailableSpaceFirstNodeSelector`

#### TestRoundRobinNodeSelector

* 获取`testZone2`的所有nodes。
* 遍历每一个node，同时执行select。
* 查看结果是否与当前遍历到的node一致。

#### TestCarryWeightNodeSelector

* 将`testZone2`的第一个node的资源翻倍。
* 循环select 100次，并记录数据到table中。
* 判断第一个node的选择次数是否比其他node的次数都要多。
* 还原node的资源。

#### TestTicketNodeSelector

* 将`testZone2`的第一个node的资源翻倍。
* 循环select 100次，并记录数据到table中。
* 判断第一个node的选择次数是否比其他node的次数都要多。
* 还原node的资源。

#### TestAvailableSpaceFirstNodeSelector

* 将`testZone2`的第一个node的资源翻倍。
* 判断select的结果与第一个node是否一致。
* 还原node的资源。

### Node选择策略效果测试

位于`master/node_selector_test.go`，函数名为：
* `TestBenchTicketNodeSelector`
* `TestBenchCarryWeightNodeSelector`

流程大体上一致：
* 生成4个容量呈现阶梯状的Node。
* 用对应的策略循环select 100次，并将结果记录在表中（每选择一次都随机减少0 ~ 10 GB的容量）。
* 将结果表输出。

*NOTE：目前这个测试需要人工判断。*

在正常情况下：
* `CarryWeight`的结果呈现出明显的阶梯状。
* `Ticket`的结果也呈现出阶梯状，但是随机性更大。
