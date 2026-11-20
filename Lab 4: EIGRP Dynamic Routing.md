# Lab 4: EIGRP Dynamic Routing

Configuration: `configs/lab04-eigrp.txt`

## Why dynamic routing

Until now I used directly connected routes (learned automatically) and static routes (typed by hand). A static route does not notice when a link fails: the router keeps sending traffic down the broken path until someone logs in, removes the route and adds another. With dynamic routing, routers share information and adapt by themselves.

**EIGRP** (Enhanced Interior Gateway Routing Protocol) is Cisco’s protocol. It reacts quickly to changes, uses bandwidth efficiently and finds good routes with little delay.

## Glasgow office topology

Three floors, each with a Layer 3 switch, all connected to one edge router that provides internet access.

| Device | Role | Transit links | LAN subnet |
| --- | --- | --- | --- |
| DMT-Edge01 | Edge router (internet gateway) | `10.0.0.1`, `10.0.0.5` | None |
| DMT-Switch01 | Basement Layer 3 switch | `10.0.0.6`, `10.0.0.9` | `192.168.1.0/24` |
| DMT-Switch02 | Ground floor Layer 3 switch | `10.0.0.10`, `10.0.0.13` | `192.168.2.0/24` |
| DMT-Switch03 | First floor Layer 3 switch | `10.0.0.14`, `10.0.0.2` | `192.168.3.0/24` |

Links between the router and switches use **/30** subnets (`255.255.255.252`), which have exactly two usable addresses, one per end. A /24 would waste 252 addresses on each point-to-point link, so /30 is common practice for these.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/fc1b4585-1b1d-4a56-88c6-b6a07f0d9af7" />
 The Glasgow topology with the Edge router and three Layer 3 switches, with link addresses labelled.


## Step 1: IP addressing (before EIGRP)

On the Layer 3 switches, ports facing other devices become routed ports with `no switchport`, and `ip routing` turns on routing:

```
! DMT-Edge01
interface gig0/1
 ip address 10.0.0.1 255.255.255.252
 no shutdown
interface gig0/2
 ip address 10.0.0.5 255.255.255.252
 no shutdown

! DMT-Switch01
ip routing
interface gig0/2
 no switchport
 ip address 10.0.0.6 255.255.255.252
 no shutdown
interface gig0/1
 no switchport
 ip address 10.0.0.9 255.255.255.252
 no shutdown
interface vlan 1
 ip address 192.168.1.1 255.255.255.0
 no shutdown
```

I repeated this on the other switches, changing the addresses. Gateways: `192.168.1.1`, `192.168.2.1`, `192.168.3.1`. At this stage devices could ping their own gateway but not devices behind other switches, because each device knew only its connected networks.

## Step 2: EIGRP

Every device uses the same Autonomous System number, here AS 1. The `network` command lists which networks are advertised, with a wildcard mask (the opposite of the subnet mask: `0.0.0.3` for /30, `0.0.0.255` for /24).

```
! DMT-Edge01
router eigrp 1
 network 10.0.0.0 0.0.0.3
 network 10.0.0.4 0.0.0.3

! DMT-Switch01
router eigrp 1
 network 10.0.0.4 0.0.0.3
 network 10.0.0.8 0.0.0.3
 network 192.168.1.0 0.0.0.255
```

The other switches follow the same pattern (see the config file). As each device was configured, `EIGRP-5-NBRCHANGE: Neighbor 10.0.0.x is up` messages showed neighbours discovering each other.

## Step 3: Verification

```
show ip eigrp neighbors
show ip eigrp topology
show ip route
```

- DMT-Edge01 listed Switch01 and Switch03 as neighbours.
- `192.168.2.0/24` was reachable by two paths with the same cost, so EIGRP **load balanced** across both.
- `show ip route` showed routes marked `D` (learned by EIGRP) on every device, with no static routes.
- Pings from a PC on Switch01 to the edge router and PCs on the other switches all succeeded.


 <img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/df446a46-3f9b-48fe-be85-d84ca3401ca1" />
 `show ip eigrp neighbors` on DMT-Edge01 with its two neighbours.



<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/32f688be-19a8-4ec7-9e0a-ac900de46ac5" />
 `show ip eigrp topology` with two equal-cost paths to `192.168.2.0/24`.



<img width="1448" height="1086" alt="image" src="https://github.com/user-attachments/assets/b1601e25-f821-4dcf-b4fe-b053ab74ebfa" />
 `show ip route` showing `D` routes.



<img width="1402" height="1122" alt="image" src="https://github.com/user-attachments/assets/12cbfa98-e4ed-47a8-be84-f48fa19ac413" />
 Pings from a Switch01 PC to the edge router and to PCs on the other floors.

## Summary

| Item | Detail |
| --- | --- |
| Protocol | EIGRP, AS 1 |
| Devices | DMT-Edge01, DMT-Switch01, 02 and 03 |
| Transit networks | /30 subnets in `10.0.0.x` |
| LANs | `192.168.1.0/24`, `192.168.2.0/24`, `192.168.3.0/24` |
| Load balancing | Two equal-cost paths to `192.168.2.0/24` |
| Verification | Neighbours, topology, routing table and pings |

## Next

Continue to Lab 5: OSPF.
