# Troubleshooting log

Problems hit while building this lab, with what was seen, how it was found, and the fix.

## 1. `ip ospf 1` rejected in global configuration mode

**Seen**
```
RTR-1-Core(config)#ip ospf 1
                           ^
% Invalid input detected at '^' marker.
```
`ip ospf ?` in global configuration mode only listed `name-lookup`.

**Cause:** `ip ospf 1` is not a global command. The OSPF process is created with `router ospf 1`, and `router-id` and `area ... nssa` only work inside it. `ip ospf 1 area <n>` is an interface command.

**Fix**
```
router ospf 1
 router-id 0.0.0.1
 area 10 nssa
```
After `router ospf 1` the prompt changes to `(config-router)#`. If it does not, the command was not accepted.

## 2. Wrong IP address on R-10 `fa1/3`

**Seen:** no adjacency between R-4 and R-10.

**How it was found:** a script compared the addresses of all links in the exported configs. One /24 (192.168.41.0) had three routers and another (192.168.104.0) had only one. R-4 `fa1/3` was `192.168.104.4`, but R-10 `fa1/3` was `192.168.41.10`, so the two ends of that cable were in different subnets.

**Fix:** `ip address 192.168.104.10 255.255.255.0` on R-10 `fa1/3`. After the change every link subnet has exactly two ends.

## 3. Summarization on one ABR only

**Seen:** `area 16 range` was configured on R-8 but not on R-9.

**Effect:** R-9 still advertised the four `/32` loopbacks of R-16 into the backbone, so the `/22` summary from R-8 had no effect (longest match always chose the `/32`).

**Fix:** the same `area 16 range 172.20.0.0 255.255.252.0` on R-9. R-1 then showed only the `/22` for area 16.

## 4. Exported configs did not match the running routers

**Seen:** the exported file for R-17 still had `ip ospf 1 area 17` on its four loopbacks, while the routing table on R-9 showed those prefixes as external (`O N2`), so they had already been taken out of OSPF on the router. The file for R-9 was also missing `area 16 range`, which existed on the router (the `/22 ... Null0` summary route was visible).

**Fix:** export the configs again after the last change, and compare route-table evidence with the files before publishing.

**Design note:** R-17's loopbacks (172.20.4.17 to 172.20.7.17) are redistributed with `redistribute connected subnets` and are not enabled in OSPF. That is what produces the Type 7 routes inside the NSSA.

## 5. Hostnames did not match the node labels

R-8 to R-11 had hostnames starting with `SW-` although they are routers (for example `SW-8-AGG1-L`), and the EVE node labels were `R-n`. Renamed to `RTR-8-AGG1-L` and so on, so the file name, hostname and topology agree.

## 6. Not enough memory for 19 routers

**Seen:** with 256 MB per router, `free -m` on the EVE VM (7 GB RAM) showed about 38 MB available, no swap, and the platform became unstable.

**Fix**
- RAM per router lowered to 128 MB (about 3.9 GB used with all 19 running, about 1.8 GB available).
- A 4 GB swap file as a safety net.

**CPU:** not a problem here. Load average about 2 on 4 vCPUs, about 93% idle, and each `dynamips` process about 7% CPU, which indicates the Idle PC value was effective.

## Checklist for the next time adjacencies do not come up

1. `show ip ospf neighbor` on both ends, then `show ip ospf interface brief`.
2. Compare the two ends of the link: same subnet, same area, same area type, and compatible network type.
3. For stub and NSSA areas, every router in the area must carry the same area flag.
4. `show ip protocols` to see the router-id and the networks or interfaces in each process.
5. Use `show ip route` and `show ip ospf database` to see which LSA types arrive in each area.
