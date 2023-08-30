# CubeFS Nodeset/Node 分配策略

CubeFS为用户提供了四种分配策略：
* CarryWeight - 使用权值算法进行分配（可用资源优先的均匀的分配算法）。
* Ticket - 使用彩票算法进行分配（同上，但随机性比较强适用于节点数量进行变更的情况）。
* AvailableSpaceFirst - 可用资源优先（可用资源优先的分配算法，只会选择可用资源最大的节点，可用资源倾斜严重的极端情况）。
* RoundRobin - 轮询算法。

其中CarryWeight是Node分配的默认策略，RoundRobin是Nodeset分配的默认策略。

## CarryWeight

CarryWeight策略是所有策略之中最复杂的。

分配逻辑分为几步：
* 计算max totoal。
* 初始化节点的carry。
* 获得足够的carry nodes。
* 将节点按carry 排序，选择其中最大的几个。
* 减少被选择的node的carry。

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

### 减少被选择的node的carry

```go
	// pick first N nodes
	for i := 0; i < replicaNum; i++ {
		node := weightedNodes[i].Ptr
		s.selectNodeForWrite(node)
		orderHosts = append(orderHosts, node.GetAddr())
		peer := proto.Peer{ID: node.GetID(), Addr: node.GetAddr()}
		peers = append(peers, peer)
	}

func (s *CarryWeightNodeSelector) selectNodeForWrite(node Node) {
	node.SelectNodeForWrite()
	// decrease node weight
	s.carry[node.GetID()] -= 1.0
}
```

## Ticket

Ticket算法与操作系统的彩票调度算法相似。

分成以下步骤：
* 将节点按id排序。
* 将节点的可用资源聚合在一起组成一条“总线”，并获取一个`[0,length]`的随机数，其中`length`是“总线”的长度。
* 根据随机数落入的节点的区域，选择对应的节点。

### 将节点按id排序

```go
	// sort nodes by id, so we can get a node list that is as stable as possible
	sort.Slice(sortedNodes, func(i, j int) bool {
		return sortedNodes[i].GetID() < sortedNodes[j].GetID()
	})
```

### 获取彩票

```go
func (s *TicketNodeSelector) GetTicket(nodes []Node, excludeHosts []string, selectedHosts []string) uint64 {
	total := uint64(0)
	for i := 0; i != len(nodes); i++ {
		if !canAllocPartition(nodes[i], s.nodeType) || contains(excludeHosts, nodes[i].GetAddr()) || contains(selectedHosts, nodes[i].GetAddr()) {
			continue
		}
		switch s.nodeType {
		case MetaNodeType:
			n := nodes[i].(*MetaNode)
			total += n.Total - n.Used
		case DataNodeType:
			n := nodes[i].(*DataNode)
			total += n.AvailableSpace
		default:
			panic("unkown node type")
		}
	}
	ticket := uint64(0)
	if total != 0 {
		ticket = s.random.Uint64() % total
	}
	return ticket
}
```

### 选择节点

```go
	for len(orderHosts) != replicaNum {
		ticket := s.GetTicket(sortedNodes, excludeHosts, orderHosts)
		node := s.GetNodeByTicket(ticket, sortedNodes, excludeHosts, orderHosts)
		if node == nil {
			break
		}
		orderHosts = append(orderHosts, node.GetAddr())
		node.SelectNodeForWrite()
		peer := proto.Peer{ID: node.GetID(), Addr: node.GetAddr()}
		peers = append(peers, peer)
	}

func (s *TicketNodeSelector) GetNodeByTicket(ticket uint64, nodes []Node, excludeHosts []string, selectedHosts []string) (node Node) {
	total := uint64(0)
	for i := 0; i != len(nodes); i++ {
		if !canAllocPartition(nodes[i], s.nodeType) || contains(excludeHosts, nodes[i].GetAddr()) || contains(selectedHosts, nodes[i].GetAddr()) {
			continue
		}
		switch s.nodeType {
		case MetaNodeType:
			n := nodes[i].(*MetaNode)
			total += n.Total - n.Used
		case DataNodeType:
			n := nodes[i].(*DataNode)
			total += n.AvailableSpace
		default:
			panic("unkown node type")
		}
		if ticket <= total {
			node = nodes[i]
			return
		}
	}
	return
}
```

## AvailableSpaceFirst

AvailableSpaceFirst算法适用于节点可用资源倾斜严重的情况。

它只会选择可用资源最大的几个节点。

步骤如下：
* 按节点可用资源排序。
* 选择前几个节点。

### 按节点可用资源排序

```go
	// sort nodes by available space
	sort.Slice(sortedNodes, func(i, j int) bool {
		return s.getNodeAvailableSpace(sortedNodes[i]) > s.getNodeAvailableSpace(sortedNodes[j])
	})
```

### 选择前几个节点

```go
	// pick first N nodes
	for i := 0; i < replicaNum && nodeIndex < len(sortedNodes); i++ {
		selectedIndex := len(sortedNodes)
		// loop until we get a writable node
		for nodeIndex < len(sortedNodes) {
			node := sortedNodes[nodeIndex]
			nodeIndex += 1
			if canAllocPartition(node, s.nodeType) {
				if excludeHosts == nil || !contains(excludeHosts, node.GetAddr()) {
					selectedIndex = nodeIndex - 1
					break
				}
			}
		}
		// if we get a writable node, append it to host list
		if selectedIndex != len(sortedNodes) {
			node := sortedNodes[selectedIndex]
			node.SelectNodeForWrite()
			orderHosts = append(orderHosts, node.GetAddr())
			peer := proto.Peer{ID: node.GetID(), Addr: node.GetAddr()}
			peers = append(peers, peer)
		}
	}
```

## RoundRobin

按轮询算法进行节点选择。

步骤如下：
* 维护一个选择指针`index`，并初始化为0。
* 将节点按id排序。
* 选择`index`后的几个节点，并将`index`前推。

```go
func (s *RoundRobinNodeSelector) Select(ns *nodeSet, excludeHosts []string, replicaNum int) (newHosts []string, peers []proto.Peer, err error) {
	newHosts = make([]string, 0)
	peers = make([]proto.Peer, 0)
	// if replica == 0, return
	if replicaNum == 0 {
		return
	}
	orderHosts := make([]string, 0)
	nodes := ns.getNodes(s.nodeType)
	sortedNodes := make([]Node, 0)
	nodes.Range(func(key, value interface{}) bool {
		sortedNodes = append(sortedNodes, asNodeWrap(value, s.nodeType))
		return true
	})
	// if we cannot get enough nodes, return error
	if len(sortedNodes) < replicaNum {
		err = fmt.Errorf("action[%vNodeSelector::Select] no enough writable hosts,replicaNum:%v  MatchNodeCount:%v  ",
			s.GetName(), replicaNum, len(sortedNodes))
		return
	}
	// sort nodes by id, so we can get a node list that is as stable as possible
	sort.Slice(sortedNodes, func(i, j int) bool {
		return sortedNodes[i].GetID() < sortedNodes[j].GetID()
	})
	nodeIndex := 0
	// pick first N nodes
	for i := 0; i < replicaNum && nodeIndex < len(sortedNodes); i++ {
		selectedIndex := len(sortedNodes)
		// loop until we get a writable node
		for nodeIndex < len(sortedNodes) {
			node := sortedNodes[(nodeIndex+s.index)%len(sortedNodes)]
			nodeIndex += 1
			if canAllocPartition(node, s.nodeType) {
				if excludeHosts == nil || !contains(excludeHosts, node.GetAddr()) {
					selectedIndex = nodeIndex - 1
					break
				}
			}
		}
		// if we get a writable node, append it to host list
		if selectedIndex != len(sortedNodes) {
			node := sortedNodes[(selectedIndex+s.index)%len(sortedNodes)]
			orderHosts = append(orderHosts, node.GetAddr())
			node.SelectNodeForWrite()
			peer := proto.Peer{ID: node.GetID(), Addr: node.GetAddr()}
			peers = append(peers, peer)
		}
	}
	// if we cannot get enough writable nodes, return error
	if len(orderHosts) < replicaNum {
		err = fmt.Errorf("action[%vNodeSelector::Select] no enough writable hosts,replicaNum:%v  MatchNodeCount:%v  ",
			s.GetName(), replicaNum, len(orderHosts))
		return
	}
	// move the index of selector
	s.index += nodeIndex
	log.LogInfof("action[%vNodeSelector::Select] peers[%v]", s.GetName(), peers)
	// reshuffle for primary-backup replication
	if newHosts, err = reshuffleHosts(orderHosts); err != nil {
		err = fmt.Errorf("action[%vNodeSelector::Select] err:%v  orderHosts is nil", s.GetName(), err.Error())
		return
	}
	return
}
```

## 关于Nodeset策略

Nodeset策略基本上是以上策略的以Nodeset为单位的版本，其中节点的资源总量被替换为nodeset的资源总量，节点的可用资源被替换为nodeset的可用资源。

## Compare CarryWeight & Ticket Nodeset Selector

```log
=== RUN   TestBenchmarkCarryWeightNodesetSelector
    nodeset_selector_test.go:170: CarryWeight Nodeset Select times:
    nodeset_selector_test.go:142: Nodeset 3 select 31 times
    nodeset_selector_test.go:142: Nodeset 2 select 20 times
    nodeset_selector_test.go:142: Nodeset 1 select 11 times
    nodeset_selector_test.go:142: Nodeset 4 select 38 times
    nodeset_selector_test.go:46: Nodeset 4
        	Total Data Space:400 GB
        	Total Meta Space:0 GB
        	Total Data Available Space:206 GB
        	Total Meta Available Space:0 GB
    nodeset_selector_test.go:46: Nodeset 2
        	Total Data Space:200 GB
        	Total Meta Space:0 GB
        	Total Data Available Space:88 GB
        	Total Meta Available Space:0 GB
    nodeset_selector_test.go:46: Nodeset 3
        	Total Data Space:300 GB
        	Total Meta Space:0 GB
        	Total Data Available Space:153 GB
        	Total Meta Available Space:0 GB
    nodeset_selector_test.go:46: Nodeset 1
        	Total Data Space:100 GB
        	Total Meta Space:0 GB
        	Total Data Available Space:56 GB
        	Total Meta Available Space:0 GB
    nodeset_selector_test.go:202: CarryWeight Nodeset Select times:
    nodeset_selector_test.go:142: Nodeset 4 select 38 times
    nodeset_selector_test.go:142: Nodeset 3 select 31 times
    nodeset_selector_test.go:142: Nodeset 2 select 20 times
    nodeset_selector_test.go:142: Nodeset 1 select 11 times
    nodeset_selector_test.go:46: Nodeset 4
        	Total Data Space:0 GB
        	Total Meta Space:400 GB
        	Total Data Available Space:0 GB
        	Total Meta Available Space:206 GB
    nodeset_selector_test.go:46: Nodeset 2
        	Total Data Space:0 GB
        	Total Meta Space:200 GB
        	Total Data Available Space:0 GB
        	Total Meta Available Space:88 GB
    nodeset_selector_test.go:46: Nodeset 3
        	Total Data Space:0 GB
        	Total Meta Space:300 GB
        	Total Data Available Space:0 GB
        	Total Meta Available Space:153 GB
    nodeset_selector_test.go:46: Nodeset 1
        	Total Data Space:0 GB
        	Total Meta Space:100 GB
        	Total Data Available Space:0 GB
        	Total Meta Available Space:56 GB
--- PASS: TestBenchmarkCarryWeightNodesetSelector (0.00s)
=== RUN   TestBenchmarkTicketNodesetSelector
    nodeset_selector_test.go:170: Ticket Nodeset Select times:
    nodeset_selector_test.go:142: Nodeset 3 select 33 times
    nodeset_selector_test.go:142: Nodeset 2 select 19 times
    nodeset_selector_test.go:142: Nodeset 4 select 42 times
    nodeset_selector_test.go:142: Nodeset 1 select 6 times
    nodeset_selector_test.go:46: Nodeset 1
        	Total Data Space:100 GB
        	Total Meta Space:0 GB
        	Total Data Available Space:68 GB
        	Total Meta Available Space:0 GB
    nodeset_selector_test.go:46: Nodeset 2
        	Total Data Space:200 GB
        	Total Meta Space:0 GB
        	Total Data Available Space:99 GB
        	Total Meta Available Space:0 GB
    nodeset_selector_test.go:46: Nodeset 3
        	Total Data Space:300 GB
        	Total Meta Space:0 GB
        	Total Data Available Space:134 GB
        	Total Meta Available Space:0 GB
    nodeset_selector_test.go:46: Nodeset 4
        	Total Data Space:400 GB
        	Total Meta Space:0 GB
        	Total Data Available Space:203 GB
        	Total Meta Available Space:0 GB
    nodeset_selector_test.go:202: Ticket Nodeset Select times:
    nodeset_selector_test.go:142: Nodeset 4 select 42 times
    nodeset_selector_test.go:142: Nodeset 1 select 6 times
    nodeset_selector_test.go:142: Nodeset 3 select 33 times
    nodeset_selector_test.go:142: Nodeset 2 select 19 times
    nodeset_selector_test.go:46: Nodeset 1
        	Total Data Space:0 GB
        	Total Meta Space:100 GB
        	Total Data Available Space:0 GB
        	Total Meta Available Space:68 GB
    nodeset_selector_test.go:46: Nodeset 2
        	Total Data Space:0 GB
        	Total Meta Space:200 GB
        	Total Data Available Space:0 GB
        	Total Meta Available Space:99 GB
    nodeset_selector_test.go:46: Nodeset 3
        	Total Data Space:0 GB
        	Total Meta Space:300 GB
        	Total Data Available Space:0 GB
        	Total Meta Available Space:134 GB
    nodeset_selector_test.go:46: Nodeset 4
        	Total Data Space:0 GB
        	Total Meta Space:400 GB
        	Total Data Available Space:0 GB
        	Total Meta Available Space:203 GB
--- PASS: TestBenchmarkTicketNodesetSelector (0.00s)
```

## Compare CarryWeight & Ticket Node Selector

```log
=== RUN   TestBenchTicketNodeSelector
    node_selector_test.go:237: Node 1 select times 23
        Node 2 select times 33
        Node 3 select times 34
        Node 0 select times 10
        
    node_selector_test.go:47: Data Node 0
        	Total Space:102400 MB
        	Avaliable Space:50686 MB
        
    node_selector_test.go:47: Data Node 1
        	Total Space:204800 MB
        	Avaliable Space:86914 MB
        
    node_selector_test.go:47: Data Node 2
        	Total Space:307200 MB
        	Avaliable Space:132137 MB
        
    node_selector_test.go:47: Data Node 3
        	Total Space:409600 MB
        	Avaliable Space:244367 MB
        
    node_selector_test.go:237: Node 2 select times 33
        Node 3 select times 34
        Node 0 select times 10
        Node 1 select times 23
        
    node_selector_test.go:53: Meta Node 0
        	Total Space:102400 MB
        	Avaliable Space:50686 MB
        
    node_selector_test.go:53: Meta Node 1
        	Total Space:204800 MB
        	Avaliable Space:86914 MB
        
    node_selector_test.go:53: Meta Node 2
        	Total Space:307200 MB
        	Avaliable Space:132137 MB
        
    node_selector_test.go:53: Meta Node 3
        	Total Space:409600 MB
        	Avaliable Space:244367 MB
        
--- PASS: TestBenchTicketNodeSelector (0.00s)
=== RUN   TestBenchCarryWeightNodeSelector
    node_selector_test.go:237: Node 0 select times 11
        Node 3 select times 38
        Node 2 select times 29
        Node 1 select times 22
        
    node_selector_test.go:47: Data Node 0
        	Total Space:102400 MB
        	Avaliable Space:57746 MB
        
    node_selector_test.go:47: Data Node 1
        	Total Space:204800 MB
        	Avaliable Space:124706 MB
        
    node_selector_test.go:47: Data Node 2
        	Total Space:307200 MB
        	Avaliable Space:152358 MB
        
    node_selector_test.go:47: Data Node 3
        	Total Space:409600 MB
        	Avaliable Space:179293 MB
        
    node_selector_test.go:237: Node 3 select times 38
        Node 2 select times 29
        Node 1 select times 22
        Node 0 select times 11
        
    node_selector_test.go:53: Meta Node 0
        	Total Space:102400 MB
        	Avaliable Space:57746 MB
        
    node_selector_test.go:53: Meta Node 1
        	Total Space:204800 MB
        	Avaliable Space:124706 MB
        
    node_selector_test.go:53: Meta Node 2
        	Total Space:307200 MB
        	Avaliable Space:152358 MB
        
    node_selector_test.go:53: Meta Node 3
        	Total Space:409600 MB
        	Avaliable Space:179293 MB
        
--- PASS: TestBenchCarryWeightNodeSelector (0.00s)
```
