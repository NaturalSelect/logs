# Node Selector

## CarryWeight

**DataNode:**

```log
    node_selector_test.go:43: Data Node 5
        	Total Space:2097152 MB
        	Avaliable Space:2097152 MB
        
    node_selector_test.go:43: Data Node 6
        	Total Space:1048576 MB
        	Avaliable Space:1048576 MB
        
    node_selector_test.go:43: Data Node 7
        	Total Space:1048576 MB
        	Avaliable Space:1048576 MB
        
    node_selector_test.go:43: Data Node 8
        	Total Space:1048576 MB
        	Avaliable Space:1048576 MB
    node_selector_test.go:261: CarryWeight data node select times:
    node_selector_test.go:233: Node 5 select times 40
        Node 7 select times 20
        Node 6 select times 20
        Node 8 select times 20
```

**MetaNode:**
```log
    node_selector_test.go:49: Meta Node 11
        	Total Space:20480 MB
        	Avaliable Space:21474835456 MB
        
    node_selector_test.go:49: Meta Node 12
        	Total Space:10240 MB
        	Avaliable Space:10737417216 MB
        
    node_selector_test.go:49: Meta Node 14
        	Total Space:10240 MB
        	Avaliable Space:10737417216 MB
        
    node_selector_test.go:49: Meta Node 13
        	Total Space:10240 MB
        	Avaliable Space:10737417216 MB
    node_selector_test.go:292: CarryWeight meta node select times:
    node_selector_test.go:233: Node 11 select times 25
        Node 13 select times 25
        Node 12 select times 25
        Node 14 select times 25
```

[Full Log](https://github.com/NaturalSelect/logs/blob/main/ospp%202023/carry.log)

## Ticket

**DataNode:**

```log
    node_selector_test.go:43: Data Node 5
        	Total Space:2097152 MB
        	Avaliable Space:2097152 MB
        
    node_selector_test.go:43: Data Node 6
        	Total Space:1048576 MB
        	Avaliable Space:1048576 MB
        
    node_selector_test.go:43: Data Node 7
        	Total Space:1048576 MB
        	Avaliable Space:1048576 MB
        
    node_selector_test.go:43: Data Node 8
        	Total Space:1048576 MB
        	Avaliable Space:1048576 MB
    node_selector_test.go:326: Ticket data node select times:
    node_selector_test.go:233: Node 7 select times 24
        Node 8 select times 18
        Node 6 select times 19
        Node 5 select times 39
```

[Full Log](https://github.com/NaturalSelect/logs/blob/main/ospp%202023/ticket.log)

**MetaNode:**

```log
    node_selector_test.go:49: Meta Node 11
        	Total Space:20480 MB
        	Avaliable Space:21474835456 MB

    node_selector_test.go:49: Meta Node 13
        	Total Space:10240 MB
        	Avaliable Space:10737417216 MB
        
    node_selector_test.go:49: Meta Node 12
        	Total Space:10240 MB
        	Avaliable Space:10737417216 MB
        
    node_selector_test.go:49: Meta Node 14
        	Total Space:10240 MB
        	Avaliable Space:10737417216 MB

    node_selector_test.go:353: Ticket meta node select times:
    node_selector_test.go:233: Node 14 select times 21
        Node 11 select times 41
        Node 13 select times 17
        Node 12 select times 21
```

## Fix Carry Weight

**DataNode:**

```log
    node_selector_test.go:43: Data Node 6
        	Total Space:1048576 MB
        	Avaliable Space:1048576 MB
        
    node_selector_test.go:43: Data Node 7
        	Total Space:1048576 MB
        	Avaliable Space:1048576 MB
        
    node_selector_test.go:43: Data Node 5
        	Total Space:2097152 MB
        	Avaliable Space:2097152 MB
        
    node_selector_test.go:43: Data Node 8
        	Total Space:1048576 MB
        	Avaliable Space:1048576 MB
    node_selector_test.go:261: CarryWeight data node select times:
    node_selector_test.go:233: Node 5 select times 40
        Node 6 select times 20
        Node 7 select times 20
        Node 8 select times 20
```

**MetaNode:**

```log
node_selector_test.go:49: Meta Node 11
        	Total Space:20480 MB
        	Avaliable Space:21474835456 MB
        
    node_selector_test.go:49: Meta Node 12
        	Total Space:10240 MB
        	Avaliable Space:10737417216 MB
        
    node_selector_test.go:49: Meta Node 13
        	Total Space:10240 MB
        	Avaliable Space:10737417216 MB
        
    node_selector_test.go:49: Meta Node 14
        	Total Space:10240 MB
        	Avaliable Space:10737417216 MB
    node_selector_test.go:292: CarryWeight meta node select times:
    node_selector_test.go:233: Node 11 select times 42
        Node 12 select times 20
        Node 13 select times 19
        Node 14 select times 19
```

[Full Log](https://github.com/NaturalSelect/logs/blob/main/ospp%202023/carry-fix.log)
