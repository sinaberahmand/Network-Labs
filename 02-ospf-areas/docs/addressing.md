# Addressing plan and OSPF design

Generated from the final running-configs in `configs/`. All 33 routed links were checked: both ends share a /24, no IP address is used twice, the OSPF area is the same on both ends, and every router-id is unique.

## Devices

| Router | Hostname | Role | Router-ID |
|---|---|---|---|
| R-1 | RTR-1-Core | Backbone, ABR (areas 0, 5, 10) | 0.0.0.1 |
| R-2 | RTR-2-Core | Backbone, ABR (areas 0, 5, 10) | 0.0.0.2 |
| R-3 | RTR-3-Core | Backbone | 0.0.0.3 |
| R-4 | RTR-4-Core | Backbone | 0.0.0.4 |
| R-5 | RTR-5-EDGE-Internet | Internet edge, ASBR (originates default route) | 0.0.0.5 |
| R-6 | RTR-6-WAN-EDGE | WAN edge, ASBR (OSPF 1 <> OSPF 2 redistribution) | 0.0.0.6 / 0.0.1.6 |
| R-7 | RTR-7-Export-Company | WAN company (separate OSPF 2 domain) | 0.0.1.7 |
| R-8 | RTR-8-AGG1-L | Aggregation, ABR (areas 0, 16-19) | 0.0.0.8 |
| R-9 | RTR-9-AGG2-L | Aggregation, ABR (areas 0, 16-19) | 0.0.0.9 |
| R-10 | RTR-10-AGG1-R | Aggregation, ABR (areas 0, 12-15) | 0.0.0.10 |
| R-11 | RTR-11-AGG2-R | Aggregation, ABR (areas 0, 12-15) | 0.0.0.11 |
| R-12 | RTR-12-LA-1 | Stub area 12 | 0.0.0.12 |
| R-13 | RTR-13-LA-2 | Stub area 13 | 0.0.0.13 |
| R-14 | RTR-14-NY-1 | Totally stub area 14 | 0.0.0.14 |
| R-15 | RTR-15-NY-2 | Totally stub area 15 | 0.0.0.15 |
| R-16 | RTR-16-TX-1 | NSSA area 16 | 0.0.0.16 |
| R-17 | RTR-17-TX-2 | NSSA area 17, ASBR (redistribute connected) | 0.0.0.17 |
| R-18 | RTR-18-VA-1 | Totally NSSA area 18 | 0.0.0.18 |
| R-19 | RTR-19-VA-2 | Totally NSSA area 19 | 0.0.0.19 |

## Areas

| Area | Type | Routers | ABRs | Notes |
|---|---|---|---|---|
| 0 | Backbone | R-1 to R-4, loopbacks of R-8 to R-11 | | `area 0 range 172.16.0.0/16` on R-10 and R-11 |
| 5 | Normal | R-5 | R-1, R-2 | R-5 originates `default-information originate always` |
| 10 | NSSA | R-6 | R-1, R-2 | R-6 injects the WAN routes (OSPF 2) as Type 7 |
| 12 | Stub | R-12 | R-10, R-11 | range 172.18.0.0/22 |
| 13 | Stub | R-13 | R-10, R-11 | range 172.18.4.0/22 |
| 14 | Totally stub | R-14 | R-10, R-11 | `stub no-summary` on the ABRs, range 172.18.8.0/22 |
| 15 | Totally stub | R-15 | R-10, R-11 | `stub no-summary` on the ABRs, range 172.18.12.0/22 |
| 16 | NSSA | R-16 | R-8, R-9 | `nssa default-information-originate`, range 172.20.0.0/22 |
| 17 | NSSA | R-17 (ASBR) | R-8, R-9 | `nssa default-information-originate`, loopbacks redistributed |
| 18 | Totally NSSA | R-18 | R-8, R-9 | `nssa no-summary` |
| 19 | Totally NSSA | R-19 | R-8, R-9 | `nssa no-summary` |

R-6 and R-7 form a separate OSPF domain (process 2, area 0) over 192.168.67.0/24. R-6 is the only router that belongs to both processes.

## Routing policy

| Where | What |
|---|---|
| R-5 | `default-information originate always` (Type 5 default route into the domain) |
| R-1, R-2 | `ip ospf priority 255` / `254` on 192.168.12.0/24 (broadcast link) to control DR/BDR election |
| R-8, R-9 | `area 16 range 172.20.0.0 255.255.252.0` (inter-area summary) |
| R-8, R-9 | `summary-address 172.20.0.0 255.255.240.0 tag 10` (external summary of the NSSA routes) |
| R-10, R-11 | `area 12`..`15 range` (one /22 per area) and `area 0 range 172.16.0.0 255.255.0.0` |
| R-6 | `redistribute ospf 2 subnets tag 6000 route-map RED-2-1` (prefix-list 172.30.0.0/23 le 32) |
| R-6 | `redistribute ospf 1 subnets tag 4000 route-map RED-1-2` (prefix-list 172.20.4.0/22 le 32) |
| R-17 | `redistribute connected subnets` (loopbacks 172.20.4.17 to 172.20.7.17 are not in OSPF) |
| Leaf routers (R-12 to R-19) | `ip ospf cost 100` toward one ABR and `200` toward the other, so each leaf prefers one ABR for its outbound traffic |
| All point-to-point links | `ip ospf network point-to-point` and `ip ospf prefix-suppression` |

## Links (33)

All links are `/24`. Cost `-` means the default cost.

### Backbone (R-1 to R-4)

| Side A | IP | Side B | IP | Subnet | Area | Cost A / B | Note |
|---|---|---|---|---|---|---|---|
| R-1 fa0/0 | 192.168.12.1 | R-2 fa0/0 | 192.168.12.2 | 192.168.12.0/24 | 0 | - / - | broadcast, DR/BDR (prio 255 / 254) |
| R-1 fa0/1 | 192.168.13.1 | R-3 fa0/1 | 192.168.13.3 | 192.168.13.0/24 | 0 | - / - |  |
| R-2 fa0/1 | 192.168.24.2 | R-4 fa0/1 | 192.168.24.4 | 192.168.24.0/24 | 0 | - / - |  |
| R-3 fa0/0 | 192.168.34.3 | R-4 fa0/0 | 192.168.34.4 | 192.168.34.0/24 | 0 | - / - |  |

### Internet and WAN edge

| Side A | IP | Side B | IP | Subnet | Area | Cost A / B | Note |
|---|---|---|---|---|---|---|---|
| R-1 fa1/1 | 192.168.15.1 | R-5 fa1/1 | 192.168.15.5 | 192.168.15.0/24 | 5 | - / - |  |
| R-1 fa1/2 | 192.168.16.1 | R-6 fa1/2 | 192.168.16.6 | 192.168.16.0/24 | 10 | - / - |  |
| R-2 fa1/0 | 192.168.25.2 | R-5 fa1/0 | 192.168.25.5 | 192.168.25.0/24 | 5 | - / - |  |
| R-2 fa1/3 | 192.168.26.2 | R-6 fa1/3 | 192.168.26.6 | 192.168.26.0/24 | 10 | - / - |  |
| R-6 fa0/0 | 192.168.67.6 | R-7 fa0/0 | 192.168.67.7 | 192.168.67.0/24 | 0 (OSPF 2) | - / - |  |

### Backbone to left aggregation (R-8, R-9)

| Side A | IP | Side B | IP | Subnet | Area | Cost A / B | Note |
|---|---|---|---|---|---|---|---|
| R-1 fa1/0 | 192.168.18.1 | R-8 fa1/0 | 192.168.18.8 | 192.168.18.0/24 | 0 | - / - |  |
| R-1 fa1/3 | 192.168.19.1 | R-9 fa1/3 | 192.168.19.9 | 192.168.19.0/24 | 0 | - / - |  |
| R-3 fa1/3 | 192.168.38.3 | R-8 fa1/3 | 192.168.38.8 | 192.168.38.0/24 | 0 | - / - |  |
| R-3 fa1/0 | 192.168.39.3 | R-9 fa1/0 | 192.168.39.9 | 192.168.39.0/24 | 0 | - / - |  |

### Backbone to right aggregation (R-10, R-11)

| Side A | IP | Side B | IP | Subnet | Area | Cost A / B | Note |
|---|---|---|---|---|---|---|---|
| R-2 fa1/2 | 192.168.21.2 | R-10 fa1/2 | 192.168.21.10 | 192.168.21.0/24 | 0 | - / - |  |
| R-2 fa1/4 | 192.168.211.2 | R-11 fa1/4 | 192.168.211.11 | 192.168.211.0/24 | 0 | - / - |  |
| R-4 fa1/3 | 192.168.104.4 | R-10 fa1/3 | 192.168.104.10 | 192.168.104.0/24 | 0 | - / - |  |
| R-4 fa1/2 | 192.168.41.4 | R-11 fa1/2 | 192.168.41.11 | 192.168.41.0/24 | 0 | - / - |  |

### Left aggregation to NSSA / totally NSSA routers

| Side A | IP | Side B | IP | Subnet | Area | Cost A / B | Note |
|---|---|---|---|---|---|---|---|
| R-8 fa0/0 | 192.168.168.8 | R-16 fa0/0 | 192.168.168.16 | 192.168.168.0/24 | 16 | - / 200 |  |
| R-8 fa0/1 | 192.168.178.8 | R-17 fa0/1 | 192.168.178.17 | 192.168.178.0/24 | 17 | - / 200 |  |
| R-8 fa1/1 | 192.168.188.8 | R-18 fa1/1 | 192.168.188.18 | 192.168.188.0/24 | 18 | - / 200 |  |
| R-8 fa1/2 | 192.168.198.8 | R-19 fa1/2 | 192.168.198.19 | 192.168.198.0/24 | 19 | - / 200 |  |
| R-9 fa1/1 | 192.168.169.9 | R-16 fa1/1 | 192.168.169.16 | 192.168.169.0/24 | 16 | - / 100 |  |
| R-9 fa1/2 | 192.168.179.9 | R-17 fa1/2 | 192.168.179.17 | 192.168.179.0/24 | 17 | - / 100 |  |
| R-9 fa0/1 | 192.168.189.9 | R-18 fa0/1 | 192.168.189.18 | 192.168.189.0/24 | 18 | - / 100 |  |
| R-9 fa0/0 | 192.168.199.9 | R-19 fa0/0 | 192.168.199.19 | 192.168.199.0/24 | 19 | - / 100 |  |

### Right aggregation to stub / totally stub routers

| Side A | IP | Side B | IP | Subnet | Area | Cost A / B | Note |
|---|---|---|---|---|---|---|---|
| R-10 fa0/0 | 192.168.120.10 | R-12 fa0/0 | 192.168.120.12 | 192.168.120.0/24 | 12 | - / 100 |  |
| R-10 fa0/1 | 192.168.130.10 | R-13 fa0/1 | 192.168.130.13 | 192.168.130.0/24 | 13 | - / 100 |  |
| R-10 fa1/1 | 192.168.140.10 | R-14 fa1/1 | 192.168.140.14 | 192.168.140.0/24 | 14 | - / 100 |  |
| R-10 fa1/0 | 192.168.150.10 | R-15 fa1/0 | 192.168.150.15 | 192.168.150.0/24 | 15 | - / 100 |  |
| R-11 fa1/0 | 192.168.121.11 | R-12 fa1/0 | 192.168.121.12 | 192.168.121.0/24 | 12 | - / 200 |  |
| R-11 fa1/1 | 192.168.131.11 | R-13 fa1/1 | 192.168.131.13 | 192.168.131.0/24 | 13 | - / 200 |  |
| R-11 fa0/1 | 192.168.141.11 | R-14 fa0/1 | 192.168.141.14 | 192.168.141.0/24 | 14 | - / 200 |  |
| R-11 fa0/0 | 192.168.151.11 | R-15 fa0/0 | 192.168.151.15 | 192.168.151.0/24 | 15 | - / 200 |  |

## Loopbacks

| Router | Interface | Address | OSPF |
|---|---|---|---|
| R-1 | Loopback100 | 172.16.100.1/32 | area 0 |
| R-2 | Loopback100 | 172.16.100.2/32 | area 0 |
| R-3 | Loopback100 | 172.16.100.3/32 | area 0 |
| R-4 | Loopback100 | 172.16.100.4/32 | area 0 |
| R-6 | Loopback200 | 172.16.100.6/24 | area 10 |
| R-7 | Loopback100 | 172.30.0.7/24 | area 0 (OSPF 2) |
| R-7 | Loopback101 | 172.30.1.7/24 | area 0 (OSPF 2) |
| R-7 | Loopback102 | 172.30.2.7/24 | area 0 (OSPF 2) |
| R-7 | Loopback103 | 172.30.3.7/24 | area 0 (OSPF 2) |
| R-8 | Loopback100 | 172.16.100.8/32 | area 0 |
| R-9 | Loopback100 | 172.16.100.9/32 | area 0 |
| R-10 | Loopback100 | 172.16.100.10/32 | area 0 |
| R-11 | Loopback100 | 172.16.100.11/32 | area 0 |
| R-12 | Loopback100 | 172.18.0.12/24 | area 12 |
| R-12 | Loopback101 | 172.18.1.12/24 | area 12 |
| R-12 | Loopback102 | 172.18.2.12/24 | area 12 |
| R-12 | Loopback103 | 172.18.3.12/24 | area 12 |
| R-13 | Loopback100 | 172.18.4.13/24 | area 13 |
| R-13 | Loopback101 | 172.18.5.13/24 | area 13 |
| R-13 | Loopback102 | 172.18.6.13/24 | area 13 |
| R-13 | Loopback103 | 172.18.7.13/24 | area 13 |
| R-14 | Loopback100 | 172.18.8.14/24 | area 14 |
| R-14 | Loopback101 | 172.18.9.14/24 | area 14 |
| R-14 | Loopback102 | 172.18.10.14/24 | area 14 |
| R-14 | Loopback103 | 172.18.11.14/24 | area 14 |
| R-15 | Loopback100 | 172.18.12.15/24 | area 15 |
| R-15 | Loopback101 | 172.18.13.15/24 | area 15 |
| R-15 | Loopback102 | 172.18.14.15/24 | area 15 |
| R-15 | Loopback103 | 172.18.15.15/24 | area 15 |
| R-16 | Loopback100 | 172.20.0.16/24 | area 16 |
| R-16 | Loopback101 | 172.20.1.16/24 | area 16 |
| R-16 | Loopback102 | 172.20.2.16/24 | area 16 |
| R-16 | Loopback103 | 172.20.3.16/24 | area 16 |
| R-17 | Loopback100 | 172.20.4.17/24 | not in OSPF (redistributed) |
| R-17 | Loopback101 | 172.20.5.17/24 | not in OSPF (redistributed) |
| R-17 | Loopback102 | 172.20.6.17/24 | not in OSPF (redistributed) |
| R-17 | Loopback103 | 172.20.7.17/24 | not in OSPF (redistributed) |
| R-17 | Loopback200 | 172.16.100.17/32 | area 17 |
| R-18 | Loopback100 | 172.20.8.18/24 | area 18 |
| R-18 | Loopback101 | 172.20.9.18/24 | area 18 |
| R-18 | Loopback102 | 172.20.10.18/24 | area 18 |
| R-18 | Loopback103 | 172.20.11.18/24 | area 18 |
| R-19 | Loopback100 | 172.20.12.19/24 | area 19 |
| R-19 | Loopback101 | 172.20.13.19/24 | area 19 |
| R-19 | Loopback102 | 172.20.14.19/24 | area 19 |
| R-19 | Loopback103 | 172.20.15.19/24 | area 19 |
