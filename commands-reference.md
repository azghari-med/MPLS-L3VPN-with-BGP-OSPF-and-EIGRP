# Commands Reference — OmniCore VRF-Lite Lab

## 📋 IP Addressing Quick Reference

### Backbone

|Link|PE1|PE2|
|-|-|-|
|PE1 ↔ PE2|192.168.100.1/30<br />lo0 10.10.10.1/32|192.168.100.2/30<br />lo0 10.10.10.2/32|

### VRF BLUE — TechBlue (EIGRP AS 100)

|Device|WAN IP|LAN IP|
|-|-|-|
|PE1|10.0.1.2/30|—|
|BLUE-CE1|10.0.1.1/30|172.16.10.1/24|
|PE2|10.0.4.2/30|—|
|BLUE-CE2|10.0.4.1/30|172.16.11.1/24|

### VRF RED — RedSteel (eBGP)

|Device|WAN IP|LAN IP|ASN|
|-|-|-|-|
|PE1|10.0.2.2/30|—|65000|
|RED-CE1|10.0.2.1/30|172.16.20.1/24|65001|
|PE2|10.0.5.2/30|—|65000|
|RED-CE2|10.0.5.1/30|172.16.21.1/24|65002|

### VRF GREEN — GreenWave (OSPF Area 0)

|Device|WAN IP|LAN IP|
|-|-|-|
|PE1|10.0.3.2/30|—|
|GREEN-CE1|10.0.3.1/30|172.16.30.1/24|
|PE2|10.0.6.2/30|—|
|GREEN-CE2|10.0.6.1/30|172.16.31.1/24|

\---

## 🔑 VRF Parameters

|VRF|RD|RT Export|RT Import|
|-|-|-|-|
|BLUE|65000:100|65000:100|65000:100|
|RED|65000:200|65000:200|65000:200|
|GREEN|65000:300|65000:300|65000:300|

\---

## 🔧 VRF Show Commands

```
show ip vrf
show ip vrf interfaces
show ip vrf detail BLUE
show ip vrf detail RED
show ip vrf detail GREEN
show ip route vrf BLUE
show ip route vrf RED
show ip route vrf GREEN
show ip protocols vrf BLUE
show ip protocols vrf RED
show ip protocols vrf GREEN
```

\---

## 🔀 EIGRP VRF Commands

```
show ip eigrp vrf BLUE neighbors
show ip eigrp vrf BLUE topology
show ip route vrf BLUE eigrp
ping vrf BLUE \[ip-address]
```

\---

## 📡 BGP VPNv4 Commands

```
show bgp vpnv4 unicast all
show bgp vpnv4 unicast all summary
show bgp vpnv4 unicast vrf BLUE
show bgp vpnv4 unicast vrf RED
show bgp vpnv4 unicast vrf GREEN
show bgp vpnv4 unicast all neighbors
show ip bgp vpnv4 all
```

\---

## 🌐 OSPF VRF Commands

```
show ip ospf 2 neighbor
show ip ospf 2 database
show ip route vrf GREEN ospf
show ip ospf interface
```

\---

## 🎯 Key Verification One-Liners

|What to verify|Command|
|-|-|
|VRFs created correctly|show ip vrf detail BLUE|
|Interfaces in correct VRF|show ip vrf interfaces|
|EIGRP neighbor in VRF|show ip eigrp vrf BLUE neighbors|
|OSPF neighbor in VRF|show ip ospf 2 neighbor|
|BGP PE-CE neighbor|show bgp vpnv4 unicast vrf RED summary|
|MP-BGP between PEs|show bgp vpnv4 unicast all summary|
|VPNv4 routes present|show bgp vpnv4 unicast all|
|VRF routing table|show ip route vrf BLUE|
|End-to-end BLUE|BLUE-CE1# ping 172.16.11.1|
|VRF isolation|BLUE-CE1# ping 172.16.20.1 → must FAIL|

\---

## 📝 Redistribution Summary

```
CE protocol → MP-BGP (on PE):

EIGRP → BGP:
  router bgp 65000
   address-family ipv4 vrf BLUE
    redistribute eigrp 100

OSPF → BGP:
  router bgp 65000
   address-family ipv4 vrf GREEN
    redistribute ospf 2 subnets   ← subnets required

MP-BGP → CE protocol (on PE):

BGP → EIGRP:
  router eigrp 100
   address-family ipv4 vrf BLUE
    redistribute bgp 65000 metric 1 1 1 1 1
    (bandwidth delay reliability load mtu)

BGP → OSPF:
  router ospf 2 vrf GREEN
   redistribute bgp 65000 subnets
```

\---

## 📊 Protocol Comparison in This Lab

|Customer|CE Protocol|Why Used|Key Challenge|
|-|-|-|-|
|TechBlue|EIGRP|Legacy Cisco environment|Must use address-family vrf|
|RedSteel|eBGP|Same ASN per site|as-override needed|
|GreenWave|OSPF|Open standard|Must use ospf process vrf|

\---

## 🔍 Ping with VRF

```
! Ping from PE into a specific VRF
PE1# ping vrf BLUE 10.0.1.1        ← ping CE1 from VRF context
PE1# ping vrf RED 172.16.20.1      ← ping into CE LAN
PE1# ping vrf GREEN 172.16.31.1    ← ping remote CE LAN

! Traceroute in VRF context
PE1# traceroute vrf BLUE 172.16.11.1
```

