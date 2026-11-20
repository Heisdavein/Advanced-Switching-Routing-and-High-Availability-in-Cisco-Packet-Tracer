# Lab 5: OSPF Dynamic Routing

Configuration: `configs/lab05-ospf.txt`

## OSPF compared with EIGRP

Both find the best path through a network, but differently.

- **EIGRP** shares routing information with neighbours. Each router knows the best route to a destination but does not hold a map of the whole network.
- **OSPF** (Open Shortest Path First) routers share information about their own links. Each builds a complete map, the **Link State Database (LSDB)**, and runs Dijkstra’s SPF algorithm to calculate the best route to every destination.

OSPF is an open standard, so it works on equipment from many vendors, which is why it is preferred in many large and multi-vendor organisations.

## Areas

OSPF divides large networks into **areas** to reduce the information each router processes. Every OSPF network has **Area 0**, the backbone, and other areas must connect to it. This lab uses only Area 0 because the network is small and in one office.

| EIGRP | OSPF |
| --- | --- |
| Every device must use the same AS number | The process ID is only local and does not have to match |
| No areas | The area number must match on both ends of a link, or the routers will not become neighbours |

## Configuration

The IP addressing from Lab 4 was already in place, so only the routing protocol was configured. The `network` command is the same as in EIGRP with `area 0` added:

```
! DMT-Edge01
router ospf 1
 network 10.0.0.0 0.0.0.3 area 0
 network 10.0.0.4 0.0.0.3 area 0

! DMT-Switch01
router ospf 1
 network 10.0.0.4 0.0.0.3 area 0
 network 10.0.0.8 0.0.0.3 area 0
 network 192.168.1.0 0.0.0.255 area 0
```

DMT-Switch02 and DMT-Switch03 follow the same pattern (see the config file). As neighbours formed, the console showed `OSPF-5-ADJCHG: Neighbor changed to FULL`.

Neighbours exchange **Hello packets** to confirm each other is available, then exchange their routing information, after which the relationship reaches the **FULL** state and they can forward traffic.

## Verification

```
show ip ospf neighbor
show ip ospf database
show ip route
```

- DMT-Edge01 had two neighbours, Switch01 and Switch03, both FULL.
- Hello packets are sent every 10 seconds. If none arrive for 40 seconds the neighbour is considered down and its routes are removed.
- `show ip ospf database` displayed the LSDB with every router and network in the area.
- `show ip route` showed routes marked `O`, with two equal-cost paths to `192.168.2.0/24`, shared automatically as in the EIGRP lab. End-to-end pings succeeded.


<img width="1448" height="1086" alt="image" src="https://github.com/user-attachments/assets/2b5a257f-aff9-4caf-b6a4-245e7bd6f026" />
 `show ip ospf neighbor` on DMT-Edge01 with two FULL neighbours.


<img width="1448" height="1086" alt="image" src="https://github.com/user-attachments/assets/d515c921-f6f3-4e6a-b808-5b412c4481e5" />
 `show ip ospf database` listing the routers in Area 0.



 <img width="1448" height="1086" alt="image" src="https://github.com/user-attachments/assets/30782638-fb20-488d-9fa4-64e0f6e571a9" />
 `show ip route` with `O` routes and two equal-cost paths to `192.168.2.0/24`.

## EIGRP vs OSPF

| Feature | EIGRP | OSPF |
| --- | --- | --- |
| Standard | Mainly used on Cisco devices | Open standard, many vendors |
| Algorithm | DUAL | Dijkstra Shortest Path First |
| Updates | Sent when changes happen | Builds a full network map first, then shares updates when needed |
| Speed | Very fast | Fast, sometimes slightly slower than EIGRP |
| Best environment | Cisco-only networks | Large enterprise and multi-vendor networks |

## Summary

| Item | Detail |
| --- | --- |
| Protocol | OSPF process 1, single Area 0 |
| Devices | DMT-Edge01, DMT-Switch01, 02 and 03 |
| Adjacency | FULL, confirmed with `show ip ospf neighbor` |
| Equal-cost paths | Two routes to `192.168.2.0/24` |
| Key difference from EIGRP | Area number on every `network` statement; process ID is local only |

## Next

Continue to Lab 6: HSRP.
