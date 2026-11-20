# Lab 3: EtherChannel with LACP

Configuration: `configs/lab03-etherchannel-lacp.txt`

## Scenario

The Jakarta office expanded to a second floor. The first floor had a router, Switch 01, a DHCP server and several PCs. I added Switch 02 for the second floor so those devices could share the same network, internet connection and DHCP server.

One cable between the switches would be a single point of failure. I connected four cables instead, but STP saw multiple paths and blocked three of them, so only one cable carried traffic.


<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/96c38535-b77a-4193-9a4a-4c0b31549e45" />
Switch 01 and Switch 02 joined by four links on Fa0/20 to Fa0/23, with the DHCP server and PCs.

## What EtherChannel does

EtherChannel combines several physical links into one logical **Port-Channel**. STP sees one link and no longer blocks the others, so all cables are used together. It gives more bandwidth (four 100 Mbps links give about 400 Mbps) and better reliability, because the bundle keeps working if a cable fails.

I used **LACP**, the IEEE 802.3ad standard that works across vendors. The alternative, PAgP, is Cisco-specific.

## Configuration

Identical on both switches:

```
conf t
interface range fastethernet 0/20 - 23
 channel-group 1 mode active
interface port-channel 1
 switchport mode trunk
```

- `interface range` configures all four ports at once.
- `channel-group 1` puts them in group 1, forming Port-Channel1.
- `mode active` enables LACP in active mode. Two switches negotiate when both are active, or one is active and the other passive. I used active on both.
- Trunking and other switchport settings go on the **Port-Channel interface**, not the member ports.

The interfaces went down briefly while the switches negotiated, then `Port-channel1, changed state to up` appeared.

## Verification

```
show etherchannel summary
show spanning-tree
show ip interface brief
```

`show etherchannel summary` showed Port-Channel1 with all four FastEthernet ports as active members. `show spanning-tree` treated Port-Channel1 as a single logical link, so no member was blocked.

 <img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/f99c1df9-5409-483e-bc39-a9d46d54f02d" />
  `show etherchannel summary` with Port-Channel1 in use and four member ports bundled.



<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/fda546c1-2e5f-40f3-b69f-846cb1d3818f" />
 show spanning-tree` showing `Po1` forwarding and no member port blocked.
> - **Why it matters:** Shows STP treats the bundle as one link.
> - **Capture from:** Switch CLI.

## Resilience test

From a Floor 2 PC I started a continuous ping to the DHCP server on Floor 1:

```
ping 192.168.1.2 -t
```

While it ran I disconnected the cables one at a time. The ping continued after the first, second and third cable were removed, using the last remaining link.

<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/473d8a0f-7265-4f51-b4c9-8541dc2f8fc1" />
 The continuous ping output with the topology showing links removed.

## Summary

| Task | Result |
| --- | --- |
| Protocol | LACP (802.3ad), active on both switches |
| Members | Fa0/20 to Fa0/23 on each switch |
| Logical link | Port-Channel1 as a trunk |
| STP | No member links blocked |
| Resilience | Ping continued with three of four cables removed |
| Bandwidth | Four 100 Mbps links, up to about 400 Mbps available |

## Next

Continue to Lab 4: EIGRP.
