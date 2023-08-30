# CubeFS Nodeset/Node 分配策略

CubeFS为用户提供了四种分配策略：
* CarryWeight
* Ticket
* AvailableSpaceFirst
* RoundRobin

其中CarryWeight是Node分配的默认策略，RoundRobin是Nodeset分配的默认策略。

## CarryWeight

CarryWeight策略是所有策略之中最复杂的。

分配逻辑分为几步：
* 计算max totoal。
* 初始化节点的carry。
* 获得足够的carry nodes。
* 将节点按carry 排序，选择其中最大的几个。

### 计算`max total`

这个步骤获取节点的最大资源总量。

```go
func (s *CarryWeightNodeSelector) getTotalMax(nodes *sync.Map) (total uint64) {
	switch s.nodeType {
	case DataNodeType:
		total = s.getTotalMaxForDataNodes(nodes)
	case MetaNodeType:
		total = s.getTotalMaxForMetaNodes(nodes)
	}
	return
}

func (s *CarryWeightNodeSelector) getTotalMaxForDataNodes(nodes *sync.Map) (total uint64) {
	nodes.Range(func(key, value interface{}) bool {
		dataNode := value.(*DataNode)
		if dataNode.Total > total {
			total = dataNode.Total
		}
		return true
	})
	return
}
```

### 初始化节点的carry

在相应的selector，存在一张`carry`表。

初始化操作将检测节点是否在`carry`表中，如果不在则使用`max total`的值初始化。

计算公式为 $\frac{Available}{MaxTotal}$，其中`Available`为节点的可用资源。

```go
total := s.getTotalMax(nodes)
// prepare carry for every nodes
s.prepareCarry(nodes, total)

func (s *CarryWeightNodeSelector) prepareCarry(nodes *sync.Map, total uint64) {
	switch s.nodeType {
	case DataNodeType:
		s.prepareCarryForDataNodes(nodes, total)
	case MetaNodeType:
		s.prepareCarryForMetaNodes(nodes, total)
	default:
	}
}

func (s *CarryWeightNodeSelector) prepareCarryForDataNodes(nodes *sync.Map, total uint64) {
	nodes.Range(func(key, value interface{}) bool {
		dataNode := value.(*DataNode)
		if _, ok := s.carry[dataNode.ID]; !ok {
			// use available space to calculate initial weight
			s.carry[dataNode.ID] = float64(dataNode.AvailableSpace) / float64(total)
		}
		return true
	})
}
```

### 获取足够的carry nodes

如果一个节点的carry >= `1.0`, 那么我们称它是一个carry node。

如果没有找到足够的carry nodes，那么循环递增node的carry。

增量的计算公式为 $\frac{Available}{MaxTotal}$，其中`Available`为节点的可用资源。

由此可得，资源大的节点增长的快，资源小的节点增长的慢，一旦获得足够的carry nodes就停止增长。

```go
  // if we cannot get enough writable nodes, return error
	weightedNodes, count := s.getCarryNodes(ns, total, excludeHosts)
	if len(weightedNodes) < replicaNum {
		err = fmt.Errorf("action[%vNodeSelector::Select] no enough writable hosts,replicaNum:%v  MatchNodeCount:%v  ",
			s.GetName(), replicaNum, len(weightedNodes))
		return
	}
	// create enough carry nodes
	// we say a node is "carry node", whent its carry >= 1.0
	s.setNodeCarry(weightedNodes, count, replicaNum)

func (s *CarryWeightNodeSelector) setNodeCarry(nodes SortedWeightedNodes, availCarryCount, replicaNum int) {
	if availCarryCount >= replicaNum {
		return
	}
	for availCarryCount < replicaNum {
		availCarryCount = 0
		for _, nt := range nodes {
			carry := nt.Carry + nt.Weight
			// limit the max value of weight
			// prevent subsequent selections make node overloading
			if carry > 10.0 {
				carry = 10.0
			}
			nt.Carry = carry
			s.carry[nt.Ptr.GetID()] = carry
			if carry > 1.0 {
				availCarryCount++
			}
		}
	}
}
```

### 将节点按carry 排序，选择其中最大的几个。

```go
  // sort nodes by weight
	sort.Sort(weightedNodes)
	// pick first N nodes
	for i := 0; i < replicaNum; i++ {
		node := weightedNodes[i].Ptr
		s.selectNodeForWrite(node)
		orderHosts = append(orderHosts, node.GetAddr())
		peer := proto.Peer{ID: node.GetID(), Addr: node.GetAddr()}
		peers = append(peers, peer)
	}
	log.LogInfof("action[%vNodeSelector::Select] peers[%v]", s.GetName(), peers)
	// reshuffle for primary-backup replication
	if newHosts, err = reshuffleHosts(orderHosts); err != nil {
		err = fmt.Errorf("action[%vNodeSelector::Select] err:%v  orderHosts is nil", s.GetName(), err.Error())
		return
	}
	return
```

## Ticket

## AvailableSpaceFirst

## RoundRobin
