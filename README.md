# Network labs (EVE-NG)

Hands-on networking labs built and tested in EVE-NG. Every lab has its own folder with the topology, the final device configs, the addressing plan and a troubleshooting log.

## Labs

| # | Lab | Topics | Status |
|---|---|---|---|
| 01 | [Campus switching](01-campus-switching/) | VLANs, RSTP, trunks, DHCP snooping, port security, HSRP, OSPF between buildings and core | Done |
| 02 | [OSPF areas](02-ospf-areas/) | Stub, totally stub, NSSA and totally NSSA areas; ABR and ASBR roles; summarization; default route origination; redistribution between two OSPF processes | Done |
| 03 | DMVPN | | Planned |
| 04 | BGP | | Planned |

## Layout of each lab

```
NN-lab-name/
├── README.md            what the lab shows, how to run it, test results, known limitations
├── topology/            topology screenshot
├── configs/             final running-config of every device
└── docs/
    ├── addressing.md    addressing plan
    └── troubleshooting.md
```

## Conventions

- EVE node labels match the device hostnames, so a config file name tells you which node it belongs to.
- Configs are the final `show running-config` output with run-specific data removed (for example learned MAC addresses).
- Problems that were hit while building a lab are documented in that lab's `troubleshooting.md`, including the ones that were not fully explained.
