# Lab 2: Rapid Spanning Tree, PortFast and BPDU Guard

Configuration: `configs/lab02-rapid-pvst-portfast-bpduguard.txt`

## The problem

In the VLAN lab, each access switch was connected to two core switches. This **redundancy** keeps the network running if a cable or core switch fails. But multiple paths between switches let frames travel in circles, which is a **network loop**. Switches keep copying the same traffic, and the volume grows until it becomes a **broadcast storm** that exhausts bandwidth.

**Spanning Tree Protocol (STP)** prevents loops by keeping one active path and blocking the extras, then opening a blocked path if the active one fails. Classic STP can take 30 to 50 seconds to recover. **Rapid STP (RSTP)** does the same job and usually restores connectivity in about 2 to 3 seconds.

## Topology

Five switches: two core (Core Switch 01 and 02) and three access switches, with VLANs 1 to 4.

<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/ca1c1264-c46e-4af2-9820-ca0c2be3e1d3" />
 The two core switches and three access switches with their redundant links.

## Starting point: the wrong root bridge

```
show spanning-tree
```

The **root bridge** is the reference switch every other switch uses to pick its best path. Election rules: lowest bridge priority wins; with equal priorities (the default is 32768) the lowest MAC address wins.

Looking at a core switch, I found that an access switch had become the root bridge, and the access switch itself reported `This bridge is the root`. All priorities were default and that access switch happened to have the lowest MAC address. The network still worked, but traffic could take longer, less efficient paths instead of going through the core.


 <img width="1376" height="768" alt="image" src="https://github.com/user-attachments/assets/ce35cb10-3ca8-4a75-a683-955654324daf" />  
  `show spanning-tree` on the access switch showing `This bridge is the root`.

## Step 1: Rapid PVST+ on every switch

On all five switches:

```
conf t
spanning-tree mode rapid-pvst
```

Rapid PVST+ is Cisco’s RSTP with a separate spanning tree per VLAN. It recovers faster, lets each VLAN calculate its own path, and is compatible with older STP. I enabled it everywhere, because rapid convergence works best when both ends of every link run the same mode. The switches briefly recalculate the topology after the command, which is normal.

## Step 2: Choose the root bridge

```
! Core Switch 01
spanning-tree vlan 1-4 priority 4096
! Core Switch 02
spanning-tree vlan 1-4 priority 8192
```

| Switch | Priority | Role |
| --- | --- | --- |
| Core Switch 01 | 4096 | Primary root |
| Core Switch 02 | 8192 | Secondary root |
| Access switches | 32768 (default) | Never root unless both cores are down |

`show spanning-tree` then showed Core Switch 01 as root for VLANs 1, 2, 3 and 4, and the switches running RSTP. On Core Switch 02 some ports were in the **discarding** state, acting as backup paths.

I also noted that in larger networks the root role can be split between the cores per VLAN to share traffic.


<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/ae1cacb3-8910-47c1-9330-afbdcd5e571c" />
  `show spanning-tree` on Core Switch 02 with ports in the discarding (blocking) state.

## Step 3: PortFast and BPDU Guard

```
spanning-tree portfast default
spanning-tree portfast bpduguard default
```

**PortFast** lets an access port start forwarding immediately instead of waiting through the STP listening process. End devices such as PCs, printers and servers do not create loops, so the delay is unnecessary. It must only be used on ports facing end devices, never on trunks or links between switches.

**BPDU Guard** watches PortFast ports for BPDUs, which only switches send. If one arrives, the port is put into the **error-disabled** state before a loop can form. It protects against, for example, someone plugging in a small switch from home.

After enabling BPDU Guard on the access switches, one port immediately went error-disabled because another switch was connected to it and sending BPDUs.


 <img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/9aacac24-b778-4119-8538-6bbe0ebd6c44" />
  The log message or `show interfaces status` showing the port in `err-disabled`, with the port name.


## Summary

| Task | Result |
| --- | --- |
| Mode | Rapid PVST+ on all five switches |
| Root bridges | Core 01 priority 4096, Core 02 priority 8192 |
| Verification | Core Switch 01 root for VLANs 1 to 4; backup ports discarding on Core 02 |
| PortFast | Enabled on access ports |
| BPDU Guard | Enabled on PortFast ports; one port error-disabled when a switch was connected |

## Next

Continue to Lab 3: EtherChannel.
