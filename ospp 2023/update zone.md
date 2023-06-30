## Example

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
