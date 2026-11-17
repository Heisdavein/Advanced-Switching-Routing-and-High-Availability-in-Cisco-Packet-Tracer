# Lab 1: Router-on-a-Stick (Inter-VLAN Routing)

Configuration: `configs/lab01-router-on-a-stick.txt`

## The problem

VLANs split one physical network into separate ones, which improves security and cuts unnecessary traffic. But VLANs work at Layer 2, so devices in different VLANs cannot talk to each other on their own. Moving traffic between networks needs a Layer 3 device: a router or a Layer 3 switch.

The traditional fix is one router interface per VLAN. That does not scale: each new VLAN needs another physical port, and routers have few. **Router-on-a-Stick** uses a single physical link instead.

## How it works

- The switch port facing the router is a **trunk**, which carries several VLANs over one cable.
- The router divides its physical interface into **sub-interfaces**, one per VLAN, each with its own IP address. That address is the default gateway for the VLAN.
- Each sub-interface uses **802.1Q** encapsulation, which adds a VLAN tag so the router knows which VLAN a frame belongs to.

Traffic from VLAN 10 to VLAN 20 goes PC to switch, across the trunk to the router, in on the VLAN 10 sub-interface, routed, out on the VLAN 20 sub-interface, back over the same trunk, and to the destination PC.

## Topology

| Item | Detail |
| --- | --- |
| Lab | Small branch topology |
| Switch | One switch connected to all devices |
| Router | Connected to the switch with one trunk |
| VLAN 10 (HR) | PC0 `192.168.10.10`, gateway `192.168.10.1` |
| VLAN 20 (Sales) | PC1 `192.168.20.10`, gateway `192.168.20.1` |
| Physical link | Switch `GigabitEthernet0/1` to Router `GigabitEthernet0/0/0` |


<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/e2b41e35-3eff-4b67-b498-9150e7d9409a" />
 Packet Tracer topology with the router, switch, PC0 and PC1 and port labels.


## Step 1: Switch VLANs and access ports

```
conf t
vlan 10
 name HR
vlan 20
 name SALES
interface fastethernet 0/1
 switchport mode access
 switchport access vlan 10
interface fastethernet 0/2
 switchport mode access
 switchport access vlan 20
```

An access port belongs to one VLAN. PC0 is now in VLAN 10 and PC1 in VLAN 20, on the same switch but unable to communicate, which is the expected result.

<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/3634953d-66ea-416e-9fe2-9df2d626b926" />
 `show vlan` on the switch with VLAN 10 (HR) on Fa0/1 and VLAN 20 (SALES) on Fa0/2.


## Step 2: Trunk to the router

```
interface gigabitethernet 0/1
 switchport mode trunk
 switchport trunk native vlan 10
 switchport trunk allowed vlan 10,20
```

| Command | Purpose |
| --- | --- |
| `switchport mode trunk` | Carries several VLANs on this link |
| `switchport trunk native vlan 10` | Sets the native VLAN, which carries untagged traffic. The Cisco default is VLAN 1, and changing it is common for security or design reasons |
| `switchport trunk allowed vlan 10,20` | Only VLANs 10 and 20 may use the trunk |

Allowing only the VLANs that are needed is good practice: unnecessary traffic is kept off the link and unauthorised VLANs cannot cross it.


 <img width="1376" height="768" alt="image" src="https://github.com/user-attachments/assets/48883d4a-615b-448e-9fe7-02a48e02533d" />
  The trunk configuration on `Gi0/1` (`show running-config interface gigabitethernet 0/1` or `show interfaces trunk`).


## Step 3: Baseline test before routing

From PC0:

```
ping 192.168.20.10
```

Every request timed out. That was expected: different VLANs, no router configured. Testing before the change gives a **baseline**, so when the ping later succeeds I know the router configuration was what fixed it.


 <img width="1376" height="768" alt="image" src="https://github.com/user-attachments/assets/65c8c5b7-e14a-4b6e-b549-35ce1910dbef" />
  PC0 Command Prompt with the failed ping, all requests timed out.

 
## Step 4: Router sub-interfaces

```
conf t
interface gigabitethernet 0/0/0
 no shutdown
interface gigabitethernet 0/0/0.10
 encapsulation dot1q 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown
interface gigabitethernet 0/0/0.20
 encapsulation dot1q 20
 ip address 192.168.20.1 255.255.255.0
 no shutdown
end
write
```

The physical interface gets no IP address, only `no shutdown`. It exists to carry the sub-interfaces. `encapsulation dot1q 10` ties a sub-interface to VLAN 10, and its address is that VLAN’s gateway.

Verification:

```
show ip interface brief
show ip route
```

The physical interface was up with no address, both sub-interfaces were up with the right addresses, and the routing table listed `192.168.10.0/24` and `192.168.20.0/24` as directly connected.


<img width="1376" height="768" alt="image" src="https://github.com/user-attachments/assets/cc2bada1-b17d-414b-ae62-5cd5d4930a2e" />
`show ip interface brief` showing `Gi0/0/0` without an address and both sub-interfaces up.


<img width="1376" height="768" alt="image" src="https://github.com/user-attachments/assets/37686963-d7f9-4460-a214-aae811b65bd3" />
 `show ip route` with the two connected networks.



## Step 5: Test inter-VLAN connectivity

From PC0, `ping 192.168.20.10` again. The first one or two requests timed out and the rest succeeded. That is normal: the devices first learn each other’s MAC addresses with ARP.

<img width="1376" height="768" alt="image" src="https://github.com/user-attachments/assets/501e735d-ec4f-4ca9-8518-9994bf09f740" />
 PC0 pinging `192.168.20.10` with the early timeouts followed by replies.


## Troubleshooting note

After the first configuration the ping still failed. I checked IP settings on both PCs, the sub-interfaces, VLAN assignments and the trunk, and pinged PC0 from the router, which worked. The cause turned out to be Packet Tracer taking a few seconds to learn MAC addresses; waiting and retrying worked. See Troubleshooting.

## Summary

| Task | Result |
| --- | --- |
| Switch | VLANs 10 and 20, Fa0/1 and Fa0/2 as access ports, Gi0/1 as a trunk with only VLANs 10 and 20 |
| Router | Physical interface up without an address; sub-interfaces `.10` (`192.168.10.1`) and `.20` (`192.168.20.1`) with 802.1Q |
| Routing | Both networks present as connected routes |
| Result | PC0 in VLAN 10 reached PC1 in VLAN 20 |
| Limitation | All inter-VLAN traffic shares one physical link, which can become a bottleneck |

## Next

Continue to Lab 2: Rapid spanning tree.
