# Dynamic Node Selector Compare

[Workload](https://github.com/NaturalSelect/cubefs/blob/745a9a10cde8df655d4de20106ed4d560e1a4a9e/master/node_selector_test.go#L425)

## CarryWeight

**DataNode:**

```log
node_selector_test.go:47: Data Node 0
        	Total Space:102400 MB
        	Avaliable Space:53939 MB
        
    node_selector_test.go:47: Data Node 1
        	Total Space:204800 MB
        	Avaliable Space:105708 MB
        
    node_selector_test.go:47: Data Node 2
        	Total Space:307200 MB
        	Avaliable Space:153998 MB
        
    node_selector_test.go:47: Data Node 3
        	Total Space:409600 MB
        	Avaliable Space:204080 MB
        
    node_selector_test.go:237: Node 3 select times 40
        Node 2 select times 30
        Node 1 select times 20
        Node 0 select times 10
        
    node_selector_test.go:47: Data Node 3
        	Total Space:409600 MB
        	Avaliable Space:204080 MB
        
    node_selector_test.go:47: Data Node 0
        	Total Space:102400 MB
        	Avaliable Space:53939 MB
        
    node_selector_test.go:47: Data Node 1
        	Total Space:204800 MB
        	Avaliable Space:105708 MB
        
    node_selector_test.go:47: Data Node 2
        	Total Space:307200 MB
        	Avaliable Space:153998 MB
```

```log
    node_selector_test.go:47: Data Node 0
        	Total Space:102400 MB
        	Avaliable Space:46165 MB
        
    node_selector_test.go:47: Data Node 1
        	Total Space:204800 MB
        	Avaliable Space:101749 MB
        
    node_selector_test.go:47: Data Node 2
        	Total Space:307200 MB
        	Avaliable Space:163236 MB
        
    node_selector_test.go:47: Data Node 3
        	Total Space:409600 MB
        	Avaliable Space:229671 MB
        
    node_selector_test.go:237: Node 1 select times 18
        Node 0 select times 10
        Node 3 select times 41
        Node 2 select times 31
        
    node_selector_test.go:47: Data Node 0
        	Total Space:102400 MB
        	Avaliable Space:46165 MB
        
    node_selector_test.go:47: Data Node 1
        	Total Space:204800 MB
        	Avaliable Space:101749 MB
        
    node_selector_test.go:47: Data Node 2
        	Total Space:307200 MB
        	Avaliable Space:163236 MB
        
    node_selector_test.go:47: Data Node 3
        	Total Space:409600 MB
        	Avaliable Space:229671 MB
```

```log
    node_selector_test.go:47: Data Node 0
        	Total Space:102400 MB
        	Avaliable Space:50991 MB
        
    node_selector_test.go:47: Data Node 1
        	Total Space:204800 MB
        	Avaliable Space:99330 MB
        
    node_selector_test.go:47: Data Node 2
        	Total Space:307200 MB
        	Avaliable Space:161466 MB
        
    node_selector_test.go:47: Data Node 3
        	Total Space:409600 MB
        	Avaliable Space:219571 MB
        
    node_selector_test.go:237: Node 2 select times 30
        Node 1 select times 19
        Node 0 select times 10
        Node 3 select times 41
        
    node_selector_test.go:47: Data Node 2
        	Total Space:307200 MB
        	Avaliable Space:161466 MB
        
    node_selector_test.go:47: Data Node 3
        	Total Space:409600 MB
        	Avaliable Space:219571 MB
        
    node_selector_test.go:47: Data Node 0
        	Total Space:102400 MB
        	Avaliable Space:50991 MB
        
    node_selector_test.go:47: Data Node 1
        	Total Space:204800 MB
        	Avaliable Space:99330 MB

```

**MetaNode:**

```log
node_selector_test.go:53: Meta Node 3
        	Total Space:409600 MB
        	Avaliable Space:204080 MB
        
    node_selector_test.go:53: Meta Node 0
        	Total Space:102400 MB
        	Avaliable Space:53939 MB
        
    node_selector_test.go:53: Meta Node 1
        	Total Space:204800 MB
        	Avaliable Space:105708 MB
        
    node_selector_test.go:53: Meta Node 2
        	Total Space:307200 MB
        	Avaliable Space:153998 MB
        
    node_selector_test.go:237: Node 3 select times 40
        Node 2 select times 30
        Node 1 select times 20
        Node 0 select times 10
        
    node_selector_test.go:53: Meta Node 3
        	Total Space:409600 MB
        	Avaliable Space:204080 MB
        
    node_selector_test.go:53: Meta Node 0
        	Total Space:102400 MB
        	Avaliable Space:53939 MB
        
    node_selector_test.go:53: Meta Node 1
        	Total Space:204800 MB
        	Avaliable Space:105708 MB
        
    node_selector_test.go:53: Meta Node 2
        	Total Space:307200 MB
        	Avaliable Space:153998 MB
```

```log
    node_selector_test.go:53: Meta Node 0
        	Total Space:102400 MB
        	Avaliable Space:46165 MB
        
    node_selector_test.go:53: Meta Node 1
        	Total Space:204800 MB
        	Avaliable Space:101749 MB
        
    node_selector_test.go:53: Meta Node 2
        	Total Space:307200 MB
        	Avaliable Space:163236 MB
        
    node_selector_test.go:53: Meta Node 3
        	Total Space:409600 MB
        	Avaliable Space:229671 MB
        
    node_selector_test.go:237: Node 3 select times 41
        Node 2 select times 31
        Node 1 select times 18
        Node 0 select times 10
        
    node_selector_test.go:53: Meta Node 0
        	Total Space:102400 MB
        	Avaliable Space:46165 MB
        
    node_selector_test.go:53: Meta Node 1
        	Total Space:204800 MB
        	Avaliable Space:101749 MB
        
    node_selector_test.go:53: Meta Node 2
        	Total Space:307200 MB
        	Avaliable Space:163236 MB
        
    node_selector_test.go:53: Meta Node 3
        	Total Space:409600 MB
        	Avaliable Space:229671 MB
```

```log
    node_selector_test.go:53: Meta Node 2
        	Total Space:307200 MB
        	Avaliable Space:161466 MB
        
    node_selector_test.go:53: Meta Node 3
        	Total Space:409600 MB
        	Avaliable Space:219571 MB
        
    node_selector_test.go:53: Meta Node 0
        	Total Space:102400 MB
        	Avaliable Space:50991 MB
        
    node_selector_test.go:53: Meta Node 1
        	Total Space:204800 MB
        	Avaliable Space:99330 MB
        
    node_selector_test.go:237: Node 3 select times 41
        Node 2 select times 30
        Node 1 select times 19
        Node 0 select times 10
        
    node_selector_test.go:53: Meta Node 0
        	Total Space:102400 MB
        	Avaliable Space:50991 MB
        
    node_selector_test.go:53: Meta Node 1
        	Total Space:204800 MB
        	Avaliable Space:99330 MB
        
    node_selector_test.go:53: Meta Node 2
        	Total Space:307200 MB
        	Avaliable Space:161466 MB
        
    node_selector_test.go:53: Meta Node 3
        	Total Space:409600 MB
        	Avaliable Space:219571 MB
```

**Full logs:**
* [Carry1](./logs/carry1.log)
* [Carry2](./logs/carry2.log)
* [Carry3](./logs/carry3.log)

## Ticket

**DataNode:**

```log
node_selector_test.go:47: Data Node 0
        	Total Space:102400 MB
        	Avaliable Space:48077 MB
        
    node_selector_test.go:47: Data Node 1
        	Total Space:204800 MB
        	Avaliable Space:76826 MB
        
    node_selector_test.go:47: Data Node 2
        	Total Space:307200 MB
        	Avaliable Space:147798 MB
        
    node_selector_test.go:47: Data Node 3
        	Total Space:409600 MB
        	Avaliable Space:208337 MB
        
    node_selector_test.go:237: Node 1 select times 20
        Node 3 select times 39
        Node 2 select times 29
        Node 0 select times 12
        
    node_selector_test.go:47: Data Node 0
        	Total Space:102400 MB
        	Avaliable Space:48077 MB
        
    node_selector_test.go:47: Data Node 1
        	Total Space:204800 MB
        	Avaliable Space:76826 MB
        
    node_selector_test.go:47: Data Node 2
        	Total Space:307200 MB
        	Avaliable Space:147798 MB
        
    node_selector_test.go:47: Data Node 3
        	Total Space:409600 MB
        	Avaliable Space:208337 MB
```

```log
    node_selector_test.go:47: Data Node 2
        	Total Space:307200 MB
        	Avaliable Space:126552 MB
        
    node_selector_test.go:47: Data Node 3
        	Total Space:409600 MB
        	Avaliable Space:227883 MB
        
    node_selector_test.go:47: Data Node 0
        	Total Space:102400 MB
        	Avaliable Space:75803 MB
        
    node_selector_test.go:47: Data Node 1
        	Total Space:204800 MB
        	Avaliable Space:74640 MB
        
    node_selector_test.go:237: Node 0 select times 3
        Node 2 select times 36
        Node 1 select times 25
        Node 3 select times 36
        
    node_selector_test.go:47: Data Node 0
        	Total Space:102400 MB
        	Avaliable Space:75803 MB
        
    node_selector_test.go:47: Data Node 1
        	Total Space:204800 MB
        	Avaliable Space:74640 MB
        
    node_selector_test.go:47: Data Node 2
        	Total Space:307200 MB
        	Avaliable Space:126552 MB
        
    node_selector_test.go:47: Data Node 3
        	Total Space:409600 MB
        	Avaliable Space:227883 MB
```

```log
    node_selector_test.go:47: Data Node 3
        	Total Space:409600 MB
        	Avaliable Space:246817 MB
        
    node_selector_test.go:47: Data Node 0
        	Total Space:102400 MB
        	Avaliable Space:52976 MB
        
    node_selector_test.go:47: Data Node 1
        	Total Space:204800 MB
        	Avaliable Space:78583 MB
        
    node_selector_test.go:47: Data Node 2
        	Total Space:307200 MB
        	Avaliable Space:139526 MB
        
    node_selector_test.go:237: Node 2 select times 35
        Node 1 select times 21
        Node 3 select times 35
        Node 0 select times 9
        
    node_selector_test.go:47: Data Node 0
        	Total Space:102400 MB
        	Avaliable Space:52976 MB
        
    node_selector_test.go:47: Data Node 1
        	Total Space:204800 MB
        	Avaliable Space:78583 MB
        
    node_selector_test.go:47: Data Node 2
        	Total Space:307200 MB
        	Avaliable Space:139526 MB
        
    node_selector_test.go:47: Data Node 3
        	Total Space:409600 MB
        	Avaliable Space:246817 MB
```

**MetaNode:**

```log
    node_selector_test.go:53: Meta Node 0
        	Total Space:102400 MB
        	Avaliable Space:48077 MB
        
    node_selector_test.go:53: Meta Node 1
        	Total Space:204800 MB
        	Avaliable Space:76826 MB
        
    node_selector_test.go:53: Meta Node 2
        	Total Space:307200 MB
        	Avaliable Space:147798 MB
        
    node_selector_test.go:53: Meta Node 3
        	Total Space:409600 MB
        	Avaliable Space:208337 MB
        
    node_selector_test.go:237: Node 0 select times 12
        Node 1 select times 20
        Node 3 select times 39
        Node 2 select times 29
        
    node_selector_test.go:53: Meta Node 2
        	Total Space:307200 MB
        	Avaliable Space:147798 MB
        
    node_selector_test.go:53: Meta Node 3
        	Total Space:409600 MB
        	Avaliable Space:208337 MB
        
    node_selector_test.go:53: Meta Node 0
        	Total Space:102400 MB
        	Avaliable Space:48077 MB
        
    node_selector_test.go:53: Meta Node 1
        	Total Space:204800 MB
        	Avaliable Space:76826 MB
```

```log
    node_selector_test.go:53: Meta Node 0
        	Total Space:102400 MB
        	Avaliable Space:75803 MB
        
    node_selector_test.go:53: Meta Node 1
        	Total Space:204800 MB
        	Avaliable Space:74640 MB
        
    node_selector_test.go:53: Meta Node 2
        	Total Space:307200 MB
        	Avaliable Space:126552 MB
        
    node_selector_test.go:53: Meta Node 3
        	Total Space:409600 MB
        	Avaliable Space:227883 MB
        
    node_selector_test.go:237: Node 1 select times 25
        Node 3 select times 36
        Node 0 select times 3
        Node 2 select times 36
        
    node_selector_test.go:53: Meta Node 1
        	Total Space:204800 MB
        	Avaliable Space:74640 MB
        
    node_selector_test.go:53: Meta Node 2
        	Total Space:307200 MB
        	Avaliable Space:126552 MB
        
    node_selector_test.go:53: Meta Node 3
        	Total Space:409600 MB
        	Avaliable Space:227883 MB
        
    node_selector_test.go:53: Meta Node 0
        	Total Space:102400 MB
        	Avaliable Space:75803 MB
```

```log
    node_selector_test.go:53: Meta Node 3
        	Total Space:409600 MB
        	Avaliable Space:246817 MB
        
    node_selector_test.go:53: Meta Node 0
        	Total Space:102400 MB
        	Avaliable Space:52976 MB
        
    node_selector_test.go:53: Meta Node 1
        	Total Space:204800 MB
        	Avaliable Space:78583 MB
        
    node_selector_test.go:53: Meta Node 2
        	Total Space:307200 MB
        	Avaliable Space:139526 MB
        
    node_selector_test.go:237: Node 2 select times 35
        Node 1 select times 21
        Node 3 select times 35
        Node 0 select times 9
        
    node_selector_test.go:53: Meta Node 0
        	Total Space:102400 MB
        	Avaliable Space:52976 MB
        
    node_selector_test.go:53: Meta Node 1
        	Total Space:204800 MB
        	Avaliable Space:78583 MB
        
    node_selector_test.go:53: Meta Node 2
        	Total Space:307200 MB
        	Avaliable Space:139526 MB
        
    node_selector_test.go:53: Meta Node 3
        	Total Space:409600 MB
        	Avaliable Space:246817 MB
```

**Full logs:**
* [Ticket1](./logs/Ticket1.log)
* [Ticket2](./logs/Ticket2.log)
* [Ticket3](./logs/Ticket3.log)
