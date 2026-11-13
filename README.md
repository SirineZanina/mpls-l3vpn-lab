# MPLS L3VPN Lab (GNS3)

A GNS3 lab that builds a service-provider **MPLS Layer 3 VPN** backbone on Cisco 7200 routers.
Two customer sites connect to the provider edge, and each customer lives in its own **VRF**, so their
routes and traffic stay isolated while crossing a shared MPLS core.

## Topology

```
 PC1 ── CE1 ── PE1 ──── P (R5) ──── P (R6) ──── PE2 (R4) ── CE2 (R2) ── PC2
        RIPv2   │  ◄──────── OSPF area 0 + LDP (MPLS core) ────────►  │   RIPv2
                └──────────── MP-BGP VPNv4 (iBGP, AS 700) ─────────────┘
```

| Node | Role | Image |
|---|---|---|
| CE1 (R1) | Customer edge, site 1 | Cisco 7200 (IOS 15.3) |
| PE1 (R3) | Provider edge, VRF `ce1` | Cisco 7200 |
| R5, R6 | Provider core (P) routers | Cisco 7200 |
| PE2 (R4) | Provider edge, VRF `ce2` | Cisco 7200 |
| CE2 (R2) | Customer edge, site 2 | Cisco 7200 |
| PC1, PC2 | Customer hosts | VPCS |

## Addressing

| Device | Interface | Address |
|---|---|---|
| PE1 | Loopback0 | 3.3.3.3/32 |
| PE2 | Loopback0 | 4.4.4.4/32 |
| R5 | Loopback0 | 5.5.5.5/32 |
| R6 | Loopback0 | 6.6.6.6/32 |
| PE1 – R5 | Serial2/1 – Serial2/0 | 10.0.0.0/30 |
| R5 – R6 | Serial2/1 – Serial2/1 | 11.0.0.0/30 |
| R6 – PE2 | Serial2/0 – Serial2/1 | 12.0.0.0/30 |
| CE1 – PE1 | Serial2/0 – Serial2/0 | 20.0.0.0/30 |
| PC1 – CE1 | e0 – FastEthernet0/0 | 192.168.1.0/30 |
| PC2 – CE2 | e0 – FastEthernet0/0 | 192.168.2.0/30 |

## What is configured

| Layer | Configuration |
|---|---|
| Core IGP | **OSPF area 0** on PE1, R5, R6 and PE2, advertising the core links and the loopbacks |
| Label distribution | **MPLS / LDP** (`mpls ip`) on every core-facing interface, with CEF enabled |
| VPN routing | **MP-BGP VPNv4** between the PE loopbacks (iBGP, AS 700, `update-source Loopback0`, `send-community both`) |
| Customer separation | VRF `ce1` on PE1 (RD 700:1, route-target 700:1) and VRF `ce2` on PE2 (RD 700:2, route-target 700:2) |
| PE–CE routing | **RIPv2** inside each VRF, with mutual redistribution between RIP and BGP (`redistribute bgp 700 metric transparent` / `redistribute rip`) |

Because the two VRFs use different route targets, each customer only learns its own routes: the
backbone is shared, the routing tables are not.

## Running the lab

1. Install [GNS3](https://www.gns3.com/) with Dynamips.
2. Import a Cisco 7200 IOS image (`c7200-adventerprisek9-mz.153-3.XB12`). The image is **not**
   included in this repository, as Cisco IOS is licensed software.
3. Open `mpls-l3vpn-lab.gns3`. The saved router configurations are in `project-files/dynamips/*/configs/`.
4. Start all nodes.

## Verification commands

```
show ip ospf neighbor              ! core adjacencies
show mpls ldp neighbor             ! LDP sessions between core routers
show mpls forwarding-table         ! label bindings
show ip bgp vpnv4 all summary      ! MP-BGP session between PE1 and PE2
show ip route vrf ce1              ! customer routes inside the VRF
ping vrf ce1 192.168.1.2           ! reach the customer site from the PE
```

## Status

The provider core (OSPF, LDP, MP-BGP VPNv4) and the CE1 site are fully configured. The CE2 router's
configuration was not saved in this project, so site 2 still needs its customer-edge setup.

## Repository layout

```
mpls-l3vpn-lab.gns3            GNS3 project file (topology)
project-files/dynamips/*/      Router startup configurations
project-files/vpcs/*/          PC1 / PC2 addressing (VPCS)
```
