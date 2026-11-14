# Troubleshooting log

Problems hit while building this lab, in roughly the order they appeared. Each entry says what was seen, how it was narrowed down, and what fixed it. Where the cause was never established, that is stated.

## 1. Inter-VLAN traffic caused a Layer 2 storm (HSRP virtual MAC)

**Symptoms**
- Hosts could ping their own gateway and hosts in their own VLAN, but not hosts in another VLAN, even though the distribution switch had both subnets connected and could ping both hosts itself.
- Pinging with the switch's real SVI address as gateway worked; pinging through the HSRP virtual IP did not.
- One link (distribution to access) showed about 53,000 packets/s in both directions with no ping running; other ports were near zero.
- Millions of collisions on a distribution interface, `EXCESSCOLL` log messages every 30 seconds, and the switch's own MAC (including the HSRP virtual MAC) being learned on the link toward the access switch.

**How it was narrowed down**
1. `show interfaces | include ^Ethernet|input rate` on the distribution and access switches showed which link carried the traffic. Everything else was idle.
2. `show interfaces <port> | include rate|broadcasts|multicasts` showed it was almost entirely unicast, not broadcast.
3. A packet capture on that link (EVE, right-click the link, Capture) showed one single ICMP echo reply, same IP ID and sequence number, repeated tens of thousands of times with a constant TTL of 64, destined to the HSRP virtual MAC and tagged VLAN 20.
4. A constant TTL means the frame was not being routed; it was circulating at Layer 2.
5. Shutting the access-switch port cleared the storm, and it restarted on the next inter-VLAN ping.

**Fix**
```
interface vlan 10
 standby use-bia
```
(on every SVI of both distribution switches in each building). HSRP then answers ARP with the switch's own MAC instead of the virtual MAC.

**Status: root cause not confirmed.** The evidence points at the emulated switch image mishandling frames addressed to the HSRP virtual MAC (it did not treat them as its own and sent them back out the port they arrived on). That is an interpretation, not something verified. The author's working assumption is that this is a simulator limitation that would not occur on real hardware.

**Side effect of the workaround:** see entry 2.

## 2. After a failover, hosts lost connectivity until `clear arp`

**Symptoms**
- With `standby use-bia`, the gateway MAC seen by the VPCS was the real MAC of the active switch (for example `aa:bb:cc:80:80:00` for SW8).
- When the active switch was stopped, the standby switch took over (confirmed with `show standby brief`), but the VPCS kept sending to the old MAC.
- With a continuous ping running, connectivity did not return after about 4 minutes. `clear arp` on the VPCS restored it immediately. Seen in both buildings.

**Why (partly a hypothesis)**
- Expected: the gateway MAC changes on failover, so cached ARP entries become wrong.
- Not verified: the VPCS appeared to renew the ARP entry's lifetime while traffic kept flowing (the entry showed about 109 s before a ping and about 111 s after). That would explain why a continuous ping never lets the stale entry expire.

**Notes**
- Real operating systems re-resolve the gateway when it stops answering; VPCS did not here. That was not tested with a real client.
- A core failure (SW6 stopped) did not need `clear arp`: the gateway did not change, and the path recovered through the other core node in about 38 seconds (OSPF dead timer).

## 3. HSRP showed `Standby unknown` or two Active switches

**Symptoms**
- `show standby brief` on SW9 showed `Active local` / `Standby unknown` for VLAN 10 while SW8 was also active for it; traffic from a host went to the wrong distribution switch (`trace` showed SW9's SVI as first hop).

**Context**
- The two distribution switches in a building have no Layer 2 link between them: all four links are routed. HSRP hellos therefore travel through the access switches, so they depend on STP forwarding the right ports for that VLAN.

**What changed it**
- The state became correct after the HSRP configuration was rebuilt with one group per VLAN (group number = VLAN number) and the nodes were rebooted, but the exact trigger was not isolated. After that, HSRP stayed stable through the failover tests.
- Immediately after configuring HSRP, interfaces can show `Init` or `Speak` for a short time. Wait 30 seconds before judging. `Init` that persists means the SVI is not up (check `show ip interface brief` and that the VLAN exists).

## 4. A host could not reach even its own gateway: wrong access VLAN

**Symptom:** a VPC with the correct VLAN 20 address could not ping `192.168.20.x` addresses.

**Diagnosis:** `show interfaces e0/2 switchport` showed `Access Mode VLAN: 10`. The wrong version of the configuration had been pasted on that node.

**Fix:** `switchport access vlan 20`. Always check the access VLAN of the host port first when a host cannot reach its gateway.

## 5. Trunks left in `desirable` mode

**Symptom:** `show interfaces trunk` showed Mode `desirable` instead of `on` on several access and distribution switches. The trunks were up only because the other side negotiated.

**Cause:** `switchport mode trunk` was missing from those interfaces. Typically only `switchport trunk encapsulation dot1q` and the native VLAN were present. Pasting long configuration blocks into the EVE console dropped lines; this was the likely reason, not proven.

**Fix:** apply `switchport mode trunk` on every trunk port, after the encapsulation line. Verify with `show interfaces trunk`.

## 6. `switchport nonegotiate` rejected

`switchport nonegotiate` returned `% Invalid input detected` on the images used, including inside `interface range`. The cause was not investigated. Trunks work without it (DTP frames are still sent). It is left out of the final configs.

## 7. `ip arp inspection` (DAI) not available

`ip arp inspection vlan ...` returned `% Invalid input` on the access-switch image, so DAI is not part of the design. DHCP snooping and port security were accepted and are in the configs.

## 8. Building 1 access switches were in `pvst`, not `rapid-pvst`

A configuration review showed `spanning-tree mode pvst` on SW10, SW11 and SW12 while every other switch was on `rapid-pvst`. Changed to `rapid-pvst` for consistent convergence. The reason it had not applied originally was not determined.

## 9. `%CDP-4-NATIVE_VLAN_MISMATCH`

SW5 logged a native VLAN mismatch with SW3 (SW5 side 66, SW3 side 1). The trunk on SW3 was missing `switchport trunk native vlan 66`. Setting it on both ends removed the message. Check native VLANs on both sides whenever CDP reports a mismatch.

## 10. Smaller issues

- **Pasting long configs into the console:** lines are sometimes dropped or mangled. Paste 10-20 lines at a time and confirm with `show running-config interface <x>`.
- **Node labels vs hostnames:** the EVE node names (`R11`, `R12`) did not match the hostnames (`SW11`, `SW12`), which led to configuring the wrong node more than once. Rename nodes in EVE to the hostname.
- **Config saved on the wrong switch or not at all:** a stopped node comes back with whatever was last saved. Run `wr` before stopping a node for a failover test.
- **Sticky MAC addresses end up in the saved config.** The lab MACs were removed from the published configs. On SW1 `e0/2`, a second, unknown MAC (`aeae.6b8a.7a95`) was learned in addition to the VPC's; where it came from was not determined.
- **`ip routing` does not appear in `show running-config`** on the Layer 3 nodes even though routing works. Unexplained; entering the command is harmless.
- **VPCS settings are lost on restart unless `save` is run** on each VPC.
- **VLAN 666 and unused ports:** every unused port is in VLAN 666 and `shutdown`, on all switches.

## Checklist for the next time inter-VLAN traffic fails

1. Host: `show ip` (address, mask, gateway), then `ping <gateway>`.
2. Access switch: host port VLAN, port status (err-disabled?), `show interfaces trunk` for the VLAN on both uplinks.
3. Distribution switch: `show standby brief` (Active/Standby, no `unknown`), `show ip interface brief | include Vlan`, `ping` both hosts from the switch.
4. If pings between VLANs fail but the switch reaches both hosts: check the traffic rate on every port. A storm shows as tens of thousands of packets/s on one link with nothing running.
5. Capture on the busy link. A constant TTL means a Layer 2 loop, a decreasing TTL means a routing loop.
6. After a failover test: `clear arp` on the hosts before judging whether HSRP worked.
