# Addressing plan

Generated from the final running-configs in `configs/`. Every routed link below was checked: both ends share a subnet, no IP is used twice, and OSPF (process 1, area 0, point-to-point) runs on all of them.

## Devices

| Device | Role | Location | OSPF router-id |
|---|---|---|---|
| SW6-CORE-1 | Core (L3) | Core | 0.0.0.6 |
| SW7-CORE-2 | Core (L3) | Core | 0.0.0.7 |
| SW8-DIS-B1 | Distribution (L3, HSRP) | Building 1 | 0.0.0.8 |
| SW9-DIS-B1 | Distribution (L3, HSRP) | Building 1 | 0.0.0.9 |
| SW4-DIS-B2 | Distribution (L3, HSRP) | Building 2 | 0.0.0.4 |
| SW5-DIS-B2 | Distribution (L3, HSRP) | Building 2 | 0.0.0.5 |
| SW10-ACC-B1 | Access (L2) | Building 1 | - |
| SW11-ACC-B1 | Access (L2) | Building 1 | - |
| SW12-ACC-B1 | Access (L2) | Building 1 | - |
| SW1-ACC-B2 | Access (L2) | Building 2 | - |
| SW2-ACC-B2 | Access (L2) | Building 2 | - |
| SW3-ACC-B2 | Access (L2) | Building 2 | - |

## VLANs and HSRP

| VLAN | Subnet | HSRP virtual IP (gateway) | Active | Standby | STP root |
|---|---|---|---|---|---|
| 10 | 192.168.10.0/24 | 192.168.10.11 | SW8-DIS-B1 (192.168.10.10, prio 200) | SW9-DIS-B1 (192.168.10.20, prio 150) | SW8-DIS-B1 |
| 20 | 192.168.20.0/24 | 192.168.20.21 | SW8-DIS-B1 (192.168.20.20, prio 200) | SW9-DIS-B1 (192.168.20.25, prio 150) | SW8-DIS-B1 |
| 30 | 192.168.30.0/24 | 192.168.30.31 | SW9-DIS-B1 (192.168.30.40, prio 200) | SW8-DIS-B1 (192.168.30.30, prio 150) | SW9-DIS-B1 |
| 40 | 192.168.40.0/24 | 192.168.40.41 | SW9-DIS-B1 (192.168.40.49, prio 200) | SW8-DIS-B1 (192.168.40.40, prio 150) | SW9-DIS-B1 |
| 50 | 192.168.50.0/24 | 192.168.50.1 | SW5-DIS-B2 (192.168.50.160, prio 200) | SW4-DIS-B2 (192.168.50.150, prio 150) | SW5-DIS-B2 |
| 60 | 192.168.60.0/24 | 192.168.60.1 | SW5-DIS-B2 (192.168.60.161, prio 200) | SW4-DIS-B2 (192.168.60.151, prio 150) | SW5-DIS-B2 |
| 70 | 192.168.70.0/24 | 192.168.70.1 | SW4-DIS-B2 (192.168.70.152, prio 200) | SW5-DIS-B2 (192.168.70.162, prio 150) | SW4-DIS-B2 |
| 80 | 192.168.80.0/24 | 192.168.80.1 | SW4-DIS-B2 (192.168.80.153, prio 200) | SW5-DIS-B2 (192.168.80.163, prio 150) | SW4-DIS-B2 |

HSRP group number equals the VLAN number on every SVI. The HSRP active switch for a VLAN is also the STP root for that VLAN.

Special VLANs (Layer 2 only, no SVI):

| VLAN | Purpose |
|---|---|
| 66 | Native VLAN on all trunks |
| 666 | Parking VLAN for unused ports (all of them are also `shutdown`) |

## Routed point-to-point links (18)

All links are `/24` and use `ip ospf network point-to-point`.

### Building 1: SW8 ↔ SW9 (distribution interconnect)

| Side A | IP | Side B | IP | Subnet |
|---|---|---|---|---|
| SW8-DIS-B1 e2/0 | 192.168.55.4 | SW9-DIS-B1 e2/0 | 192.168.55.5 | 192.168.55.0/24 |
| SW8-DIS-B1 e1/3 | 192.168.75.3 | SW9-DIS-B1 e1/3 | 192.168.75.4 | 192.168.75.0/24 |
| SW8-DIS-B1 e0/0 | 192.168.89.1 | SW9-DIS-B1 e0/0 | 192.168.89.2 | 192.168.89.0/24 |
| SW8-DIS-B1 e0/1 | 192.168.98.2 | SW9-DIS-B1 e0/1 | 192.168.98.3 | 192.168.98.0/24 |

### Building 2: SW4 ↔ SW5 (distribution interconnect)

| Side A | IP | Side B | IP | Subnet |
|---|---|---|---|---|
| SW4-DIS-B2 e0/0 | 192.168.145.2 | SW5-DIS-B2 e0/0 | 192.168.145.5 | 192.168.145.0/24 |
| SW4-DIS-B2 e0/1 | 192.168.245.2 | SW5-DIS-B2 e0/1 | 192.168.245.5 | 192.168.245.0/24 |
| SW4-DIS-B2 e2/0 | 192.168.45.4 | SW5-DIS-B2 e1/3 | 192.168.45.5 | 192.168.45.0/24 |
| SW4-DIS-B2 e1/3 | 192.168.56.5 | SW5-DIS-B2 e2/0 | 192.168.56.6 | 192.168.56.0/24 |

### Distribution ↔ Core

| Side A | IP | Side B | IP | Subnet |
|---|---|---|---|---|
| SW4-DIS-B2 e1/1 | 192.168.64.4 | SW6-CORE-1 e0/0 | 192.168.64.6 | 192.168.64.0/24 |
| SW4-DIS-B2 e1/2 | 192.168.186.4 | SW7-CORE-2 e0/3 | 192.168.186.7 | 192.168.186.0/24 |
| SW5-DIS-B2 e1/1 | 192.168.195.5 | SW6-CORE-1 e0/1 | 192.168.195.6 | 192.168.195.0/24 |
| SW5-DIS-B2 e1/2 | 192.168.200.5 | SW7-CORE-2 e0/1 | 192.168.200.7 | 192.168.200.0/24 |
| SW6-CORE-1 e1/0 | 192.168.86.6 | SW8-DIS-B1 e0/2 | 192.168.86.8 | 192.168.86.0/24 |
| SW6-CORE-1 e1/1 | 192.168.96.6 | SW9-DIS-B1 e0/2 | 192.168.96.9 | 192.168.96.0/24 |
| SW7-CORE-2 e1/0 | 192.168.87.7 | SW8-DIS-B1 e0/3 | 192.168.87.8 | 192.168.87.0/24 |
| SW7-CORE-2 e1/1 | 192.168.97.7 | SW9-DIS-B1 e0/3 | 192.168.97.9 | 192.168.97.0/24 |

### Core: SW6 ↔ SW7

| Side A | IP | Side B | IP | Subnet |
|---|---|---|---|---|
| SW6-CORE-1 e0/2 | 192.168.67.6 | SW7-CORE-2 e0/0 | 192.168.67.7 | 192.168.67.0/24 |
| SW6-CORE-1 e0/3 | 192.168.76.6 | SW7-CORE-2 e0/2 | 192.168.76.7 | 192.168.76.0/24 |

## Access ports in use

| Switch | Port | Access VLAN |
|---|---|---|
| SW10-ACC-B1 | e0/2 | 10 |
| SW11-ACC-B1 | e0/2 | 20 |
| SW11-ACC-B1 | e0/3 | 30 |
| SW12-ACC-B1 | e0/2 | 20 |
| SW12-ACC-B1 | e0/3 | 30 |
| SW1-ACC-B2 | e0/2 | 50 |
| SW2-ACC-B2 | e0/2 | 50 |
| SW2-ACC-B2 | e0/3 | 60 |
| SW3-ACC-B2 | e0/2 | 50 |
| SW3-ACC-B2 | e0/3 | 60 |

All other access-switch ports are in VLAN 666 and shut down. Each access switch has two trunk uplinks (`e0/0`, `e0/1`, native VLAN 66, DHCP snooping trusted), one to each distribution switch of its building.

## Test hosts (VPCS)

| EVE node label | VPCS name | VLAN | IP/Mask | Gateway (HSRP virtual IP) | Connected to |
|---|---|---|---|---|---|
| VLAN10 | PCVLAN10 | 10 | 192.168.10.100/24 | 192.168.10.11 | SW10-ACC-B1 `e0/2` |
| VLAN20 | PCVlan20 | 20 | 192.168.20.100/24 | 192.168.20.21 | SW12-ACC-B1 `e0/2` |
| VLAN50 | PCVLAN50 | 50 | 192.168.50.100/24 | 192.168.50.1 | SW1-ACC-B2 `e0/2` |
| Vlan60 | PCVLAN60 | 60 | 192.168.60.100/24 | 192.168.60.1 | SW3-ACC-B2 `e0/3` |

VPCS addresses are set manually (`ip <address>/<mask> <gateway>`, then `save`). There is no DHCP server in this lab.
