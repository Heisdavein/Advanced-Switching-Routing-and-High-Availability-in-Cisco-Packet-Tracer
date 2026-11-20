# Lab 6: HSRP (Hot Standby Router Protocol

Configuration: `configs/lab06-hsrp.txt`

## The problem

Every device needs a default gateway. With a single router as the gateway, a failure, power loss or restart cuts every device off from other networks and the internet. In a business that stops work, and in a hospital, bank or online store even a short outage matters.

**HSRP** gives high availability by letting two routers act as one virtual gateway. Devices use a **virtual IP address** as their gateway. One router is **Active** and forwards traffic, the other is **Standby** and watches it. If the active router fails, the standby takes over in seconds, and the devices keep using the same virtual IP.

## Concepts

| Term | Meaning |
| --- | --- |
| Active router | Forwards traffic for the virtual gateway; usually the highest priority |
| Standby router | The backup that takes over if the active router fails |
| Virtual IP | The shared gateway address, owned by the HSRP group |
| Virtual MAC | The shared MAC address linked to the virtual IP |
| Priority | Decides the active router. Default 100, highest wins |
| Preemption | Lets a higher-priority router take the active role back after it recovers |
| Hello timer | How often HSRP routers send hellos (default 3 seconds) |
| Hold timer | How long a router waits before deciding the active router has failed (default 10 seconds) |

## Topology and design

| Item | Value |
| --- | --- |
| Edge01 | Physical IP `10.1.1.1`, priority 110, Active |
| Edge02 | Physical IP `10.1.1.2`, priority 90, Standby |
| Virtual IP | `10.1.1.254`, the default gateway for all devices |
| Group | 1 |
| Hello timer | 2 seconds (customised) |
| Hold timer | 4 seconds (customised) |


 <img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/66ebb1ed-27fc-427e-a6c5-e0250bf50404" />
   Edge01, Edge02 and the PC(s) on the `10.1.1.0/24` segment with their addresses.

## Configuration

```
! Edge01 (Active)
interface gig0/0
 ip address 10.1.1.1 255.255.255.0
 standby 1 ip 10.1.1.254
 standby 1 priority 110
 standby 1 preempt
 standby 1 timers 2 4
 no shutdown

! Edge02 (Standby)
interface gig0/0
 ip address 10.1.1.2 255.255.255.0
 standby 1 ip 10.1.1.254
 standby 1 priority 90
 standby 1 preempt
 standby 1 timers 2 4
 no shutdown
```

The only difference between the two is the priority. `standby 1 timers 2 4` shortens detection, since shorter timers mean faster failover. Preemption on both means Edge02 takes over if Edge01 fails and Edge01 takes the active role back when it recovers.

## Verification

```
show standby brief
```

Edge01 showed Group 1, state Active, virtual IP `10.1.1.254`, priority 110, preemption enabled, active router Local and standby router `10.1.1.2`. Edge02 showed state Standby.

 
 <img width="1448" height="1086" alt="image" src="https://github.com/user-attachments/assets/dc124dc3-eab9-4cfa-9119-98a0d2f0dee3" />
  `show standby brief` on Edge01 with state Active.
  


 
 <img width="1448" height="1086" alt="image" src="https://github.com/user-attachments/assets/1965cd56-6e88-4ea6-8148-5e819911a0da" />
  `show standby brief` on Edge02 with state Standby.



## Failover test

From a PC I ran a continuous ping to the virtual IP:

```
ping 10.1.1.254 -t
```

While Edge01 was connected every ping succeeded. I then disconnected Edge01 to simulate a failure. For about 2 to 4 seconds, two requests timed out, then pings resumed without any change on the PC, because Edge02 stopped receiving hellos, the 4-second hold timer expired and Edge02 became Active using the same virtual IP.

| Before failure | During failure | After failover |
| --- | --- | --- |
| All pings successful | Two timeouts | Pings resumed through Edge02 |


 <img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/5fbd8de7-4fcc-437d-95b6-f5c5d2dd4793" />
   The continuous ping to `10.1.1.254` with the two timeouts followed by replies.

## Limitations of HSRP

1. **No load balancing in one group.** Only the active router forwards traffic. Multiple HSRP groups can make different routers active for different VLANs, but each group still has one active router.
2. **Mainly Cisco.** HSRP is Cisco-proprietary. **VRRP** is the open-standard alternative that provides similar functionality across vendors.
3. **Protects the gateway only.** If the active router stays powered on but loses its internet connection, HSRP may not notice. **Interface tracking** lowers the router’s priority when a monitored link fails, so the standby takes over. It was described in my notes but not configured in this lab.

## Summary

| Item | Detail |
| --- | --- |
| Protocol | HSRP group 1 |
| Virtual IP | `10.1.1.254` |
| Active / Standby | Edge01 (110) / Edge02 (90) |
| Timers | Hello 2 s, hold 4 s |
| Preemption | Enabled on both |
| Failover test | Two pings lost; Edge02 became active within the 4-second hold timer |

## Next

Continue to Troubleshooting.
