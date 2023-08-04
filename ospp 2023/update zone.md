# Update Nodeset Selector & Node Selector

## Enable and disable zone

```log
# ./build/bin/cfs-cli zone info default
Zone Name:        default
Status:           available
Nodeset Selector:
       Data:RoundRobin
       Meta:RoundRobin

NodeSet-1:
  DataNodes[4]:
    ID        ADDRESS                                                              WRITABLE    STATUS  
    8         172.16.1.103:17310                                                   Yes         Active  
    7         172.16.1.102:17310                                                   Yes         Active  
    6         172.16.1.101:17310                                                   Yes         Active  
    9         172.16.1.104:17310                                                   Yes         Active  

  MetaNodes[4]:
    ID        ADDRESS                                                              WRITABLE    STATUS  
    3         172.16.1.102:17210                                                   Yes         Active  
    2         172.16.1.101:17210                                                   Yes         Active  
    5         172.16.1.104:17210                                                   Yes         Active  
    4         172.16.1.103:17210                                                   Yes         Active  

┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli zone update --enable=false default
Zone default has been update successfully!

┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli zone info default
Zone Name:        default
Status:           unavailable
Nodeset Selector:
       Data:RoundRobin
       Meta:RoundRobin

NodeSet-1:
  DataNodes[4]:
    ID        ADDRESS                                                              WRITABLE    STATUS  
    8         172.16.1.103:17310                                                   Yes         Active  
    7         172.16.1.102:17310                                                   Yes         Active  
    6         172.16.1.101:17310                                                   Yes         Active  
    9         172.16.1.104:17310                                                   Yes         Active  

  MetaNodes[4]:
    ID        ADDRESS                                                              WRITABLE    STATUS  
    3         172.16.1.102:17210                                                   Yes         Active  
    2         172.16.1.101:17210                                                   Yes         Active  
    5         172.16.1.104:17210                                                   Yes         Active  
    4         172.16.1.103:17210                                                   Yes         Active  
```

## Update Data Nodeset Selector

**Using Zone Update:**

```log
┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli zone update --dataNodesetSelector=CarryWeight default
Zone default has been update successfully!

┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli zone info default
Zone Name:        default
Status:           available
Nodeset Selector:
       Data:CarryWeight
       Meta:RoundRobin

NodeSet-1:
  DataNodes[4]:
    ID        ADDRESS                                                              WRITABLE    STATUS  
    8         172.16.1.103:17310                                                   Yes         Active  
    7         172.16.1.102:17310                                                   Yes         Active  
    6         172.16.1.101:17310                                                   Yes         Active  
    9         172.16.1.104:17310                                                   Yes         Active  

  MetaNodes[4]:
    ID        ADDRESS                                                              WRITABLE    STATUS  
    3         172.16.1.102:17210                                                   Yes         Active  
    5         172.16.1.104:17210                                                   Yes         Active  
    4         172.16.1.103:17210                                                   Yes         Active  
    2         172.16.1.101:17210                                                   Yes         Active  

┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli zone update --dataNodesetSelector=RoundRobin default
Zone default has been update successfully!

┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli zone info default
Zone Name:        default
Status:           available
Nodeset Selector:
       Data:RoundRobin
       Meta:RoundRobin

NodeSet-1:
  DataNodes[4]:
    ID        ADDRESS                                                              WRITABLE    STATUS  
    6         172.16.1.101:17310                                                   Yes         Active  
    9         172.16.1.104:17310                                                   Yes         Active  
    8         172.16.1.103:17310                                                   Yes         Active  
    7         172.16.1.102:17310                                                   Yes         Active  

  MetaNodes[4]:
    ID        ADDRESS                                                              WRITABLE    STATUS  
    3         172.16.1.102:17210                                                   Yes         Active  
    5         172.16.1.104:17210                                                   Yes         Active  
    4         172.16.1.103:17210                                                   Yes         Active  
    2         172.16.1.101:17210                                                   Yes         Active  

```

**Using Cluster Set:**

```log
┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli cluster set -h
Set cluster parameters

Usage:
  cfs-cli cluster set [flags]

Flags:
      --autoRepairRate string        DataNode auto repair rate
      --batchCount string            MetaNode delete batch count
      --dataNodeSelector string      Set the node select policy(datanode) for cluster
      --dataNodesetSelector string   Set the nodeset select policy(datanode) for cluster
      --deleteWorkerSleepMs string   MetaNode delete worker sleep time with millisecond. if 0 for no sleep
  -h, --help                         help for set
      --loadFactor string            Load Factor
      --markDeleteRate string        DataNode batch mark delete limit rate. if 0 for no infinity limit
      --maxDpCntLimit string         Maximum number of dp on each datanode, default 3000, 0 represents setting to default
      --metaNodeSelector string      Set the node select policy(metanode) for cluster
      --metaNodesetSelector string   Set the nodeset select policy(metanode) for cluster

┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli cluster set --dataNodesetSelector=Ticket
Cluster parameters has been set successfully. 

┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli zone info default
Zone Name:        default
Status:           available
Nodeset Selector:
       Data:Ticket
       Meta:RoundRobin

NodeSet-1:
  DataNodes[4]:
    ID        ADDRESS                                                              WRITABLE    STATUS  
    6         172.16.1.101:17310                                                   Yes         Active  
    7         172.16.1.102:17310                                                   Yes         Active  
    9         172.16.1.104:17310                                                   Yes         Active  
    8         172.16.1.103:17310                                                   Yes         Active  

  MetaNodes[4]:
    ID        ADDRESS                                                              WRITABLE    STATUS  
    5         172.16.1.104:17210                                                   Yes         Active  
    3         172.16.1.102:17210                                                   Yes         Active  
    4         172.16.1.103:17210                                                   Yes         Active  
    2         172.16.1.101:17210                                                   Yes         Active  

```

## Update Meta Nodeset Selector

**Using Zone Update:**

```log
┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli zone update --metaNodesetSelector=CarryWeight default
Zone default has been update successfully!

┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli zone info default
Zone Name:        default
Status:           available
Nodeset Selector:
       Data:RoundRobin
       Meta:CarryWeight

NodeSet-1:
  DataNodes[4]:
    ID        ADDRESS                                                              WRITABLE    STATUS  
    7         172.16.1.102:17310                                                   Yes         Active  
    6         172.16.1.101:17310                                                   Yes         Active  
    9         172.16.1.104:17310                                                   Yes         Active  
    8         172.16.1.103:17310                                                   Yes         Active  

  MetaNodes[4]:
    ID        ADDRESS                                                              WRITABLE    STATUS  
    5         172.16.1.104:17210                                                   Yes         Active  
    4         172.16.1.103:17210                                                   Yes         Active  
    2         172.16.1.101:17210                                                   Yes         Active  
    3         172.16.1.102:17210                                                   Yes         Active  

┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli zone update --metaNodesetSelector=RoundRobin default
Zone default has been update successfully!

┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli zone info default
Zone Name:        default
Status:           available
Nodeset Selector:
       Data:RoundRobin
       Meta:RoundRobin

NodeSet-1:
  DataNodes[4]:
    ID        ADDRESS                                                              WRITABLE    STATUS  
    7         172.16.1.102:17310                                                   Yes         Active  
    6         172.16.1.101:17310                                                   Yes         Active  
    9         172.16.1.104:17310                                                   Yes         Active  
    8         172.16.1.103:17310                                                   Yes         Active  

  MetaNodes[4]:
    ID        ADDRESS                                                              WRITABLE    STATUS  
    3         172.16.1.102:17210                                                   Yes         Active  
    5         172.16.1.104:17210                                                   Yes         Active  
    4         172.16.1.103:17210                                                   Yes         Active  
    2         172.16.1.101:17210                                                   Yes         Active  

```

**Using Cluster Set:**

```log
┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli cluster set --metaNodesetSelector=Ticket
Cluster parameters has been set successfully. 

┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli zone info default
Zone Name:        default
Status:           available
Nodeset Selector:
       Data:Ticket
       Meta:Ticket

NodeSet-1:
  DataNodes[4]:
    ID        ADDRESS                                                              WRITABLE    STATUS  
    7         172.16.1.102:17310                                                   Yes         Active  
    9         172.16.1.104:17310                                                   Yes         Active  
    8         172.16.1.103:17310                                                   Yes         Active  
    6         172.16.1.101:17310                                                   Yes         Active  

  MetaNodes[4]:
    ID        ADDRESS                                                              WRITABLE    STATUS  
    3         172.16.1.102:17210                                                   Yes         Active  
    4         172.16.1.103:17210                                                   Yes         Active  
    2         172.16.1.101:17210                                                   Yes         Active  
    5         172.16.1.104:17210                                                   Yes         Active  
```

### Update Data Node Selector

**Using Nodeset Update:**

```log
┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli nodeset info 1
NodeSet ID:       1
Capacity:         18
Zone:             default
DataNodeSelector: CarryWeight
MetaNodeSelector: CarryWeight
DataTotal:     3.91 TB
DataUsed:      145.26 GB
DataAvail:     3.77 TB
MetaTotal:     18.97 GB
MetaUsed:      307.78 MB
MetaAvail:     18.67 GB

DataNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  9         172.16.1.104:17310                                                   Yes         Active      1001.85 GB    36.32 GB      965.54 GB 
  8         172.16.1.103:17310                                                   Yes         Active      1001.85 GB    36.32 GB      965.54 GB 
  7         172.16.1.102:17310                                                   Yes         Active      1001.85 GB    36.32 GB      965.54 GB 
  6         172.16.1.101:17310                                                   Yes         Active      1001.85 GB    36.32 GB      965.54 GB 

MetaNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  5         172.16.1.104:17210                                                   Yes         Active      4.74 GB       63.49 MB      4.68 GB   
  4         172.16.1.103:17210                                                   Yes         Active      4.74 GB       79.41 MB      4.66 GB   
  2         172.16.1.101:17210                                                   Yes         Active      4.74 GB       87.40 MB      4.66 GB   
  3         172.16.1.102:17210                                                   Yes         Active      4.74 GB       77.48 MB      4.67 GB   

┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli nodeset update 1 --dataNodeSelector=Ticket
success
┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli nodeset info 1
NodeSet ID:       1
Capacity:         18
Zone:             default
DataNodeSelector: Ticket
MetaNodeSelector: CarryWeight
DataTotal:     3.91 TB
DataUsed:      145.26 GB
DataAvail:     3.77 TB
MetaTotal:     18.97 GB
MetaUsed:      308.14 MB
MetaAvail:     18.67 GB

DataNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  7         172.16.1.102:17310                                                   Yes         Active      1001.85 GB    36.32 GB      965.54 GB 
  6         172.16.1.101:17310                                                   Yes         Active      1001.85 GB    36.32 GB      965.54 GB 
  9         172.16.1.104:17310                                                   Yes         Active      1001.85 GB    36.32 GB      965.54 GB 
  8         172.16.1.103:17310                                                   Yes         Active      1001.85 GB    36.32 GB      965.54 GB 

MetaNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  5         172.16.1.104:17210                                                   Yes         Active      4.74 GB       63.86 MB      4.68 GB   
  4         172.16.1.103:17210                                                   Yes         Active      4.74 GB       79.41 MB      4.66 GB   
  2         172.16.1.101:17210                                                   Yes         Active      4.74 GB       87.40 MB      4.66 GB   
  3         172.16.1.102:17210                                                   Yes         Active      4.74 GB       77.48 MB      4.67 GB   

```

**Using Zone:**

```log
──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli nodeset info 1
NodeSet ID:       1
Capacity:         18
Zone:             default
DataNodeSelector: Ticket
MetaNodeSelector: Ticket
DataTotal:     3.91 TB
DataUsed:      145.32 GB
DataAvail:     3.77 TB
MetaTotal:     18.97 GB
MetaUsed:      308.44 MB
MetaAvail:     18.67 GB

DataNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  9         172.16.1.104:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  8         172.16.1.103:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  7         172.16.1.102:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  6         172.16.1.101:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 

MetaNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  5         172.16.1.104:17210                                                   Yes         Active      4.74 GB       63.86 MB      4.68 GB   
  4         172.16.1.103:17210                                                   Yes         Active      4.74 GB       79.71 MB      4.66 GB   
  2         172.16.1.101:17210                                                   Yes         Active      4.74 GB       87.40 MB      4.66 GB   
  3         172.16.1.102:17210                                                   Yes         Active      4.74 GB       77.48 MB      4.67 GB   

┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli zone update default --dataNodeSelector=CarryWeight
Zone default has been update successfully!

┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli nodeset info 1
NodeSet ID:       1
Capacity:         18
Zone:             default
DataNodeSelector: CarryWeight
MetaNodeSelector: Ticket
DataTotal:     3.91 TB
DataUsed:      145.32 GB
DataAvail:     3.77 TB
MetaTotal:     18.97 GB
MetaUsed:      308.44 MB
MetaAvail:     18.67 GB

DataNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  9         172.16.1.104:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  8         172.16.1.103:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  7         172.16.1.102:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  6         172.16.1.101:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 

MetaNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  3         172.16.1.102:17210                                                   Yes         Active      4.74 GB       77.48 MB      4.67 GB   
  5         172.16.1.104:17210                                                   Yes         Active      4.74 GB       63.86 MB      4.68 GB   
  4         172.16.1.103:17210                                                   Yes         Active      4.74 GB       79.71 MB      4.66 GB   
  2         172.16.1.101:17210                                                   Yes         Active      4.74 GB       87.40 MB      4.66 GB   

```

**Using Cluster Set:**

```log
┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli nodeset info 1
NodeSet ID:       1
Capacity:         18
Zone:             default
DataNodeSelector: CarryWeight
MetaNodeSelector: CarryWeight
DataTotal:     3.91 TB
DataUsed:      145.32 GB
DataAvail:     3.77 TB
MetaTotal:     18.97 GB
MetaUsed:      308.44 MB
MetaAvail:     18.67 GB

DataNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  7         172.16.1.102:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  6         172.16.1.101:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  9         172.16.1.104:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  8         172.16.1.103:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 

MetaNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  3         172.16.1.102:17210                                                   Yes         Active      4.74 GB       77.48 MB      4.67 GB   
  5         172.16.1.104:17210                                                   Yes         Active      4.74 GB       63.86 MB      4.68 GB   
  4         172.16.1.103:17210                                                   Yes         Active      4.74 GB       79.71 MB      4.66 GB   
  2         172.16.1.101:17210                                                   Yes         Active      4.74 GB       87.40 MB      4.66 GB   

┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli cluster set --dataNodeSelector=Ticket
Cluster parameters has been set successfully. 

┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli nodeset info 1
NodeSet ID:       1
Capacity:         18
Zone:             default
DataNodeSelector: Ticket
MetaNodeSelector: CarryWeight
DataTotal:     3.91 TB
DataUsed:      145.32 GB
DataAvail:     3.77 TB
MetaTotal:     18.97 GB
MetaUsed:      308.44 MB
MetaAvail:     18.67 GB

DataNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  9         172.16.1.104:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  8         172.16.1.103:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  7         172.16.1.102:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  6         172.16.1.101:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 

MetaNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  5         172.16.1.104:17210                                                   Yes         Active      4.74 GB       63.86 MB      4.68 GB   
  4         172.16.1.103:17210                                                   Yes         Active      4.74 GB       79.71 MB      4.66 GB   
  2         172.16.1.101:17210                                                   Yes         Active      4.74 GB       87.40 MB      4.66 GB   
  3         172.16.1.102:17210                                                   Yes         Active      4.74 GB       77.48 MB      4.67 GB   

```

### Update Meta Node Selector

**Using Nodeset Update:**

```log
┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli nodeset info 1
NodeSet ID:       1
Capacity:         18
Zone:             default
DataNodeSelector: Ticket
MetaNodeSelector: CarryWeight
DataTotal:     3.91 TB
DataUsed:      145.26 GB
DataAvail:     3.77 TB
MetaTotal:     18.97 GB
MetaUsed:      308.14 MB
MetaAvail:     18.67 GB

DataNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  7         172.16.1.102:17310                                                   Yes         Active      1001.85 GB    36.32 GB      965.54 GB 
  6         172.16.1.101:17310                                                   Yes         Active      1001.85 GB    36.32 GB      965.54 GB 
  9         172.16.1.104:17310                                                   Yes         Active      1001.85 GB    36.32 GB      965.54 GB 
  8         172.16.1.103:17310                                                   Yes         Active      1001.85 GB    36.32 GB      965.54 GB 

MetaNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  5         172.16.1.104:17210                                                   Yes         Active      4.74 GB       63.86 MB      4.68 GB   
  4         172.16.1.103:17210                                                   Yes         Active      4.74 GB       79.41 MB      4.66 GB   
  2         172.16.1.101:17210                                                   Yes         Active      4.74 GB       87.40 MB      4.66 GB   
  3         172.16.1.102:17210                                                   Yes         Active      4.74 GB       77.48 MB      4.67 GB   

┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli nodeset update 1 --metaNodeSelector=Ticket
success
┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli nodeset info 1
NodeSet ID:       1
Capacity:         18
Zone:             default
DataNodeSelector: Ticket
MetaNodeSelector: Ticket
DataTotal:     3.91 TB
DataUsed:      145.32 GB
DataAvail:     3.77 TB
MetaTotal:     18.97 GB
MetaUsed:      308.44 MB
MetaAvail:     18.67 GB

DataNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  9         172.16.1.104:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  8         172.16.1.103:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  7         172.16.1.102:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  6         172.16.1.101:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 

MetaNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  5         172.16.1.104:17210                                                   Yes         Active      4.74 GB       63.86 MB      4.68 GB   
  4         172.16.1.103:17210                                                   Yes         Active      4.74 GB       79.71 MB      4.66 GB   
  2         172.16.1.101:17210                                                   Yes         Active      4.74 GB       87.40 MB      4.66 GB   
  3         172.16.1.102:17210                                                   Yes         Active      4.74 GB       77.48 MB      4.67 GB   

```

**Using Zone:**

```log
┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli nodeset info 1
NodeSet ID:       1
Capacity:         18
Zone:             default
DataNodeSelector: CarryWeight
MetaNodeSelector: Ticket
DataTotal:     3.91 TB
DataUsed:      145.32 GB
DataAvail:     3.77 TB
MetaTotal:     18.97 GB
MetaUsed:      308.44 MB
MetaAvail:     18.67 GB

DataNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  9         172.16.1.104:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  8         172.16.1.103:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  7         172.16.1.102:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  6         172.16.1.101:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 

MetaNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  3         172.16.1.102:17210                                                   Yes         Active      4.74 GB       77.48 MB      4.67 GB   
  5         172.16.1.104:17210                                                   Yes         Active      4.74 GB       63.86 MB      4.68 GB   
  4         172.16.1.103:17210                                                   Yes         Active      4.74 GB       79.71 MB      4.66 GB   
  2         172.16.1.101:17210                                                   Yes         Active      4.74 GB       87.40 MB      4.66 GB   

┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli zone update default --metaNodeSelector=CarryWeight
Zone default has been update successfully!

┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli nodeset info 1
NodeSet ID:       1
Capacity:         18
Zone:             default
DataNodeSelector: CarryWeight
MetaNodeSelector: CarryWeight
DataTotal:     3.91 TB
DataUsed:      145.32 GB
DataAvail:     3.77 TB
MetaTotal:     18.97 GB
MetaUsed:      308.44 MB
MetaAvail:     18.67 GB

DataNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  7         172.16.1.102:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  6         172.16.1.101:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  9         172.16.1.104:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  8         172.16.1.103:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 

MetaNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  3         172.16.1.102:17210                                                   Yes         Active      4.74 GB       77.48 MB      4.67 GB   
  5         172.16.1.104:17210                                                   Yes         Active      4.74 GB       63.86 MB      4.68 GB   
  4         172.16.1.103:17210                                                   Yes         Active      4.74 GB       79.71 MB      4.66 GB   
  2         172.16.1.101:17210                                                   Yes         Active      4.74 GB       87.40 MB      4.66 GB   

```

**Using Cluster Set:**

```log
┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli nodeset info 1
NodeSet ID:       1
Capacity:         18
Zone:             default
DataNodeSelector: Ticket
MetaNodeSelector: CarryWeight
DataTotal:     3.91 TB
DataUsed:      145.32 GB
DataAvail:     3.77 TB
MetaTotal:     18.97 GB
MetaUsed:      308.44 MB
MetaAvail:     18.67 GB

DataNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  9         172.16.1.104:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  8         172.16.1.103:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  7         172.16.1.102:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  6         172.16.1.101:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 

MetaNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  5         172.16.1.104:17210                                                   Yes         Active      4.74 GB       63.86 MB      4.68 GB   
  4         172.16.1.103:17210                                                   Yes         Active      4.74 GB       79.71 MB      4.66 GB   
  2         172.16.1.101:17210                                                   Yes         Active      4.74 GB       87.40 MB      4.66 GB   
  3         172.16.1.102:17210                                                   Yes         Active      4.74 GB       77.48 MB      4.67 GB   

┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli cluster set --metaNodeSelector=Ticket
Cluster parameters has been set successfully. 

┌──(root💀LAPTOP-9TS0FG11)-[/home/nature/cubefs-master-dev]
└─# ./build/bin/cfs-cli nodeset info 1
NodeSet ID:       1
Capacity:         18
Zone:             default
DataNodeSelector: Ticket
MetaNodeSelector: Ticket
DataTotal:     3.91 TB
DataUsed:      145.32 GB
DataAvail:     3.77 TB
MetaTotal:     18.97 GB
MetaUsed:      308.44 MB
MetaAvail:     18.67 GB

DataNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  9         172.16.1.104:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  8         172.16.1.103:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  7         172.16.1.102:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 
  6         172.16.1.101:17310                                                   Yes         Active      1001.85 GB    36.33 GB      965.52 GB 

MetaNodes[4]:
  ID        ADDRESS                                                              WRITABLE    STATUS      TOTAL         USED          AVAIL     
  2         172.16.1.101:17210                                                   Yes         Active      4.74 GB       87.40 MB      4.66 GB   
  3         172.16.1.102:17210                                                   Yes         Active      4.74 GB       77.48 MB      4.67 GB   
  5         172.16.1.104:17210                                                   Yes         Active      4.74 GB       63.86 MB      4.68 GB   
  4         172.16.1.103:17210                                                   Yes         Active      4.74 GB       79.71 MB      4.66 GB   

```
