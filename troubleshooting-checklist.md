# Troubleshooting Checklist — OmniCore VRF-Lite Lab

## ⚡ Diagnostic Order — Always Follow This

```
Layer 1: Backbone connectivity PE1 ↔ PE2
Layer 2: VRF creation and interface assignment
Layer 3: CE-PE routing per customer (EIGRP/BGP/OSPF)
Layer 4: MP-BGP between PE routers
Layer 5: Redistribution into MP-BGP
Layer 6: Redistribution back to CE protocols
Layer 7: End-to-end reachability
Layer 8: VRF isolation verification
```

\---

## Layer 1 — Backbone PE1 ↔ PE2

```
PE1# ping 192.168.100.2
PE2# ping 192.168.100.1



PE1# ping 10.10.10.2
PE2# ping 10.10.10.1




```

Must work before anything else.
If fails: check Eth1/0 IPs, no shutdown, NO VRF on backbone interface.

\---

## Layer 2 — VRF Creation and Interface Assignment

```
PE1# show ip vrf
PE1# show ip vrf interfaces
PE1# show ip vrf detail BLUE
```

Check:
\[ ] All 3 VRFs created: BLUE, RED, GREEN
\[ ] RD correct per VRF (65000:100, 65000:200, 65000:300)
\[ ] RT export and import correct
\[ ] Each interface assigned to correct VRF
\[ ] IP address present after VRF assignment

Common mistake:
Adding VRF to interface removes IP automatically
Fix: re-add IP after ip vrf forwarding command

\---

## Layer 3a — VRF BLUE: EIGRP

```
PE1# show ip eigrp vrf BLUE neighbors
PE1# show ip route vrf BLUE eigrp
```

Expected: BLUE-CE1 appears as EIGRP neighbor
Expected: 172.16.10.0/24 in vrf BLUE routing table

If no neighbor:
\[ ] router eigrp 100 → address-family ipv4 vrf BLUE configured
\[ ] autonomous-system 100 inside address-family
\[ ] network statement covers 10.0.1.0/30
\[ ] BLUE-CE1 running EIGRP AS 100 with same network

\---

## Layer 3b — VRF RED: eBGP

```
PE1# show bgp vpnv4 unicast vrf RED summary
PE1# show ip route vrf RED bgp
```

Expected: RED-CE1 (10.0.2.1) appears as BGP neighbor
Expected: 172.16.20.0/24 in vrf RED routing table

If no neighbor:
\[ ] router bgp 65000 → address-family ipv4 vrf RED
\[ ] neighbor 10.0.2.1 remote-as 65001
\[ ] neighbor 10.0.2.1 activate inside address-family
\[ ] RED-CE1 has neighbor 10.0.2.2 remote-as 65000

If routes missing from CE2:
\[ ] as-override configured on PE neighbors
\[ ]If RED-CE2 using different ASN than RED-CE1 no Need for as-override

\---

## Layer 3c — VRF GREEN: OSPF

```
PE1# show ip ospf 2 neighbor
PE1# show ip route vrf GREEN ospf
```

Expected: GREEN-CE1 appears as OSPF neighbor FULL
Expected: 172.16.30.0/24 in vrf GREEN routing table

If no neighbor:
\[ ] router ospf 2 vrf GREEN configured (not router ospf 1)
\[ ] network statement covers 10.0.3.0/30
\[ ] GREEN-CE1 running ospf 2 with matching area
\[ ] router-id configured (avoids conflicts)

\---

## Layer 4 — MP-BGP Between PE Routers

```
PE1# show bgp vpnv4 unicast all summary
PE1# show bgp vpnv4 unicast all
```

Expected: PE2 (10.10.10.2) as neighbor, state = Established
Expected: Routes from all 3 VRFs in VPNv4 table

If neighbor not establishing:
\[ ] neighbor 10.10.10.2 remote-as 65000 (iBGP)
\[ ] address-family vpnv4 → neighbor activate
\[ ] send-community extended ← MOST COMMON MISS
\[ ] update-source correct interface (lo0)

If neighbor up but no routes:
\[ ] send-community extended missing → RT stripped → both
\[ ] Redistribution not configured (Layer 5)

\---

## Layer 5 — Redistribution into MP-BGP

```
PE1# show bgp vpnv4 unicast all
→ Look for prefixes from all 3 VRFs with correct RD
```

If VRF routes not in VPNv4 table:
\[ ] VRF BLUE: redistribute eigrp 100 under address-family ipv4 vrf BLUE
\[ ] VRF RED: routes auto if BGP — check redistribute connected
\[ ] VRF GREEN: redistribute ospf 2 subnets
↑ subnets keyword is critical — without it /24 routes missing

\---

## Layer 6 — Redistribution Back to CE Protocols

```
BLUE-CE1# show ip route eigrp   ← should see 172.16.11.0 from CE2
RED-CE1#  show ip bgp           ← should see 172.16.21.0 from CE2
GREEN-CE1# show ip route ospf   ← should see 172.16.31.0 from CE2
```

If CE not receiving remote site routes:
VRF BLUE:
\[ ] redistribute bgp 65000 metric 1 1 1 1 1
in address-family ipv4 vrf BLUE under router eigrp 100
VRF GREEN:
\[ ] redistribute bgp 65000 subnets
under router ospf 2 vrf GREEN

\---

## Layer 7 — End-to-End Reachability

```
BLUE-CE1#  ping 172.16.11.1   ← BLUE-CE2 LAN
RED-CE1#   ping 172.16.21.1   ← RED-CE2 LAN
GREEN-CE1# ping 172.16.31.1   ← GREEN-CE2 LAN
```

All must succeed.
If ping fails trace backwards through layers 1-6.

\---

## Layer 8 — VRF Isolation Verification

```
BLUE-CE1# ping 172.16.20.1   ← RED-CE1 LAN → MUST FAIL
BLUE-CE1# ping 172.16.30.1   ← GREEN-CE1 LAN → MUST FAIL
RED-CE1#  ping 172.16.10.1   ← BLUE-CE1 LAN → MUST FAIL
```

\---

## 🔑 Most Common Mistakes

|Mistake|Symptom|Fix|
|-|-|-|
|IP removed after VRF assignment|Interface has no IP|Re-add IP after ip vrf forwarding|
|send-community extended missing|MP-BGP up but no VRF routes|Add to vpnv4 neighbor|
|No autonomous-system in EIGRP VRF|EIGRP not running in VRF|Add autonomous-system 100|
|Missing subnets in OSPF redistribute|Only classful routes|redistribute ospf 2 subnets|
|No as-override|RED-CE2 drops CE1 routes|neighbor x.x.x.x as-override|
|Wrong OSPF process in VRF|OSPF not in VRF|router ospf 2 vrf GREEN|
|RT mismatch|Routes not imported|Match export RT on one side with import RT on other|
|Backbone interface in VRF|PE-PE routing broken|No VRF on backbone interface|



