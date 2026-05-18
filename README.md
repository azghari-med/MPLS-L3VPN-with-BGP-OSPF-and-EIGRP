# Lab: MPLS L3VPN — OmniCore ISP Multi-Tenant Network

# Mixed Protocols: OSPF (PE-PE) + EIGRP (CE-PE CU-BLUE) + BGP (PE-CE CU-RED) + OSPF (CE-PE CU-GREEN)

\---

## 🏢 Scenario

**Company:** OmniCore ISP
**Situation:** OmniCore provides managed WAN services to three enterprise
customers sharing the same physical PE (Provider Edge) infrastructure.
Each customer must be completely isolated from the others — they cannot
see each other's routes or traffic under any circumstance.

**Customer A — TechBlue Corp**
Uses EIGRP between their CE routers and the PE
Two sites: Headquarters and Branch

**Customer B — RedSteel Ltd**
Uses eBGP between their CE routers and the PE
Two sites: Factory and Office

**Customer C — GreenWave SA**
Uses OSPF between their CE routers and the PE
Two sites: DataCenter and Warehouse

The PE routers (PE1 and PE2) run OSPF as the provider backbone
protocol to carry VPNv4 routes between each other using MP-BGP.
VRF-Lite is used instead of full MPLS — no label switching required.

\---

## 🗺️ Topology Diagram

```
CU-BLUE (EIGRP)          CU-RED (BGP)           CU-GREEN (OSPF)
                                                  
\\\\\\\[BLUE-CE1]               \\\\\\\[RED-CE1]               \\\\\\\[GREEN-CE1]
172.16.10.1/24           172.16.20.1/24           172.16.30.1/24
(HQ LAN)                 (Factory LAN)            (DataCenter LAN)
Eth0/1                   Eth0/1                   Eth0/1
Eth0/0:10.0.1.1/30       Eth0/0:10.0.2.1/30       Eth0/0:10.0.3.1/30
     |                        |                        |
     | EIGRP AS 100           | eBGP AS 65001          | OSPF Area 0
     |                        |                        |
Eth0/0:10.0.1.2/30       Eth0/1:10.0.2.2/30       Eth0/2:10.0.3.2/30
\\\\\\\[PE1 - VRF BLUE]         \\\\\\\[PE1 - VRF RED]          \\\\\\\[PE1 - VRF GREEN]
     |                        |                        |
     +------------------------+------------------------+
                    \\\\\\\[PE1 - Backbone]
                    Eth1/0: 192.168.100.1/30
                    OSPF Area 0 (backbone)
                    MP-BGP AS 65000
                              |
                    Eth1/0: 192.168.100.2/30
                    \\\\\\\[PE2 - Backbone]
                    OSPF Area 0 (backbone)
                    MP-BGP AS 65000
     +------------------------+------------------------+
     |                        |                        |
Eth0/0:10.0.4.2/30       Eth0/1:10.0.5.2/30       Eth0/2:10.0.6.2/30
\\\\\\\[PE2 - VRF BLUE]         \\\\\\\[PE2 - VRF RED]          \\\\\\\[PE2 - VRF GREEN]
     |                        |                        |
Eth0/0:10.0.4.1/30       Eth0/0:10.0.5.1/30       Eth0/0:10.0.6.1/30
     | EIGRP AS 100           | eBGP AS 65002          | OSPF Area 0
\\\\\\\[BLUE-CE2]               \\\\\\\[RED-CE2]               \\\\\\\[GREEN-CE2]
172.16.11.1/24           172.16.21.1/24           172.16.31.1/24
(Branch LAN)             (Office LAN)             (Warehouse LAN)
```

\---

## 📋 IP Addressing Table

### Provider Backbone

|Link|PE1 IP|PE2 IP|
|-|-|-|
|PE1 ↔ PE2|192.168.100.1/30|192.168.100.2/30|

### VRF BLUE — TechBlue Corp (EIGRP AS 100)

|Link|Device|IP Address|
|-|-|-|
|PE1 ↔ BLUE-CE1|PE1|10.0.1.2/30|
|PE1 ↔ BLUE-CE1|BLUE-CE1|10.0.1.1/30|
|PE2 ↔ BLUE-CE2|PE2|10.0.4.2/30|
|PE2 ↔ BLUE-CE2|BLUE-CE2|10.0.4.1/30|
|BLUE-CE1 LAN|BLUE-CE1|172.16.10.1/24|
|BLUE-CE2 LAN|BLUE-CE2|172.16.11.1/24|

### VRF RED — RedSteel Ltd (eBGP)

|Link|Device|IP Address|ASN|
|-|-|-|-|
|PE1 ↔ RED-CE1|PE1|10.0.2.2/30|65000|
|PE1 ↔ RED-CE1|RED-CE1|10.0.2.1/30|65001|
|PE2 ↔ RED-CE2|PE2|10.0.5.2/30|65000|
|PE2 ↔ RED-CE2|RED-CE2|10.0.5.1/30|65002|
|RED-CE1 LAN|RED-CE1|172.16.20.1/24||
|RED-CE2 LAN|RED-CE2|172.16.21.1/24||

### VRF GREEN — GreenWave SA (OSPF Area 0)

|Link|Device|IP Address|
|-|-|-|
|PE1 ↔ GREEN-CE1|PE1|10.0.3.2/30|
|PE1 ↔ GREEN-CE1|GREEN-CE1|10.0.3.1/30|
|PE2 ↔ GREEN-CE2|PE2|10.0.6.2/30|
|PE2 ↔ GREEN-CE2|GREEN-CE2|10.0.6.1/30|
|GREEN-CE1 LAN|GREEN-CE1|172.16.30.1/24|
|GREEN-CE2 LAN|GREEN-CE2|172.16.31.1/24|

\---

## 🔐 VRF Parameters

|VRF|Customer|RD|RT Export|RT Import|CE Protocol|
|-|-|-|-|-|-|
|BLUE|TechBlue|65000:100|65000:100|65000:100|EIGRP 100|
|RED|RedSteel|65000:200|65000:200|65000:200|eBGP|
|GREEN|GreenWave|65000:300|65000:300|65000:300|OSPF 2|

\---

## ⚙️ PNET Build Steps

```
Step 1 — Add devices:
  2x PE Router  → rename: PE1, PE2
  6x CE Router  → rename: BLUE-CE1, BLUE-CE2,
                           RED-CE1,  RED-CE2,
                           GREEN-CE1, GREEN-CE2
  1x Switch     → for each CE LAN (or use loopbacks)

Step 2 — Connect cables:
  PE1 Eth1/0  ──► PE2 Eth1/0      (backbone link)
  PE1 Eth0/0  ──► BLUE-CE1 Eth0/0 (VRF BLUE)
  PE1 Eth0/1  ──► RED-CE1  Eth0/0 (VRF RED)
  PE1 Eth0/2  ──► GREEN-CE1 Eth0/0 (VRF GREEN)
  PE2 Eth0/0  ──► BLUE-CE2 Eth0/0 (VRF BLUE)
  PE2 Eth0/1  ──► RED-CE2  Eth0/0 (VRF RED)
  PE2 Eth0/2  ──► GREEN-CE2 Eth0/0 (VRF GREEN)

Step 3 — Configure backbone first:
  PE1 Eth1/0: 192.168.100.1/30
  PE2 Eth1/0: 192.168.100.2/30
  Verify: PE1# ping 192.168.100.2 ✅

Step 4 — Create VRFs on PE routers (Task 1)
Step 5 — Configure CE-facing interfaces in VRFs (Task 2)
Step 6 — Configure CE routers (Task 3)
Step 7 — Configure PE-CE routing per customer (Tasks 4-6)
Step 8 — Configure MP-BGP between PE routers (Task 7)
Step 9 — Redistribute into MP-BGP (Task 8)
Step 10 — Verify end-to-end (Task 9)
Step 11 — Extra challenges (Tasks 10-12)





**Loopback for BGP Router-ID** 


! On PE1

interface Loopback0

ip address 10.10.10.1 255.255.255.255

no shutdown


! On PE2

interface Loopback0

ip address 10.10.10.2 255.255.255.255

no shutdown
```

\---

## 🎯 Tasks

\---

### Task 1 — Create VRFs on PE1 and PE2

```
! On BOTH PE1 and PE2 — same config

ip vrf BLUE
 rd 65000:100
 route-target export 65000:100
 route-target import 65000:100

ip vrf RED
 rd 65000:200
 route-target export 65000:200
 route-target import 65000:200

ip vrf GREEN
 rd 65000:300
 route-target export 65000:300
 route-target import 65000:300
```

> RD (Route Distinguisher): makes routes unique in MP-BGP table
> RT (Route Target): controls which VRFs import/export which routes
> Same RT import/export = full mesh between same-customer sites

\---

### Task 2 — Assign Interfaces to VRFs on PE1

```
! VRF BLUE — facing BLUE-CE1
interface Ethernet0/0
 ip vrf forwarding BLUE
 ip address 10.0.1.2 255.255.255.252
 no shutdown

! VRF RED — facing RED-CE1
interface Ethernet0/1
 ip vrf forwarding RED
 ip address 10.0.2.2 255.255.255.252
 no shutdown

! VRF GREEN — facing GREEN-CE1
interface Ethernet0/2
 ip vrf forwarding GREEN
 ip address 10.0.3.2 255.255.255.252
 no shutdown

! Backbone — NO VRF (global routing table)
interface Ethernet1/0
 ip address 192.168.100.1 255.255.255.252
 no shutdown
```

> CRITICAL: ip vrf forwarding MUST be configured before ip address
> Adding VRF to interface REMOVES the IP address automatically
> You must re-add the IP after adding VRF

\---

### Task 3 — Configure CE Routers

```
! BLUE-CE1 — TechBlue HQ
interface Ethernet0/0
 ip address 10.0.1.1 255.255.255.252
 no shutdown
interface Ethernet0/1
 ip address 172.16.10.1 255.255.255.0
 no shutdown

! RED-CE1 — RedSteel Factory
interface Ethernet0/0
 ip address 10.0.2.1 255.255.255.252
 no shutdown
interface Ethernet0/1
 ip address 172.16.20.1 255.255.255.0
 no shutdown

! GREEN-CE1 — GreenWave DataCenter
interface Ethernet0/0
 ip address 10.0.3.1 255.255.255.252
 no shutdown
interface Ethernet0/1
 ip address 172.16.30.1 255.255.255.0
 no shutdown

! Repeat same pattern for CE2 routers on PE2 side
! BLUE-CE2:  10.0.4.1/30 + 172.16.11.1/24
! RED-CE2:   10.0.5.1/30 + 172.16.21.1/24
! GREEN-CE2: 10.0.6.1/30 + 172.16.31.1/24
```

\---

### Task 4 — VRF BLUE: EIGRP between PE and BLUE-CE routers

```
! On PE1 — EIGRP inside VRF BLUE
router eigrp BLUE
 address-family ipv4 vrf BLUE autonomous-system 100
  network 10.0.1.0 0.0.0.3
  no auto-summary
 exit-address-family

! On PE2 — EIGRP inside VRF BLUE
router eigrp BLUE
 address-family ipv4 vrf BLUE autonomous-system 100
  network 10.0.4.0 0.0.0.3
  no auto-summary
 exit-address-family

! On BLUE-CE1 — standard EIGRP
router eigrp 100
 no auto-summary
 network 10.0.1.0 0.0.0.3
 network 172.16.10.0 0.0.0.255

! On BLUE-CE2 — standard EIGRP
router eigrp 100
 no auto-summary
 network 10.0.4.0 0.0.0.3
 network 172.16.11.0 0.0.0.255
```

\---

### Task 5 — VRF RED: eBGP between PE and RED-CE routers

```
! On PE1 — eBGP inside VRF RED
router bgp 65000
 address-family ipv4 vrf RED
  neighbor 10.0.2.1 remote-as 65001
  neighbor 10.0.2.1 activate
  neighbor 10.0.2.1 as-override
  neighbor 10.0.2.1 next-hop-self
  exit-address-family

! On PE2 — eBGP inside VRF RED
router bgp 65000
 address-family ipv4 vrf RED
  neighbor 10.0.5.1 remote-as 65001
  neighbor 10.0.5.1 activate
  neighbor 10.0.5.1 as-override
  neighbor 10.0.5.1 next-hop-self
  exit-address-family

! On RED-CE1 — eBGP to PE1
router bgp 65001
 neighbor 10.0.2.2 remote-as 65000
 address-family ipv4
  neighbor 10.0.2.2 activate
  network 172.16.20.0 mask 255.255.255.0

! On RED-CE2 — eBGP to PE2
router bgp 65001
 neighbor 10.0.5.2 remote-as 65000
 address-family ipv4
  neighbor 10.0.5.2 activate
  network 172.16.21.0 mask 255.255.255.0
```

> as-override on PE: replaces CE AS number in AS-PATH
> Without it RED-CE2 drops routes from RED-CE1 because
> it sees its own AS (65001) in the path and assumes a loop

\---

### Task 6 — VRF GREEN: OSPF between PE and GREEN-CE routers

```
! On PE1 — OSPF inside VRF GREEN
router ospf 2 vrf GREEN
 router-id 1.1.1.1
 network 10.0.3.0 0.0.0.3 area 0

! On PE2 — OSPF inside VRF GREEN
router ospf 2 vrf GREEN
 router-id 2.2.2.2
 network 10.0.6.0 0.0.0.3 area 0

! On GREEN-CE1 — standard OSPF
router ospf 2
 router-id 3.3.3.3
 network 10.0.3.0 0.0.0.3 area 0
 network 172.16.30.0 0.0.0.255 area 0

! On GREEN-CE2 — standard OSPF
router ospf 2
 router-id 4.4.4.4
 network 10.0.6.0 0.0.0.3 area 0
 network 172.16.31.0 0.0.0.255 area 0
```

\---

### Task 7 — MP-BGP between PE1 and PE2 (backbone)

```
! On PE1
router bgp 65000
 bgp router-id 10.10.10.1
 neighbor 10.10.10.2 remote-as 65000
 neighbor 10.10.10.2 update-source Lo0

 address-family vpnv4 unicast
  neighbor 10.10.10.2 activate
  neighbor 10.10.10.2 send-community Both
  neighbor 10.10.10.2 next-hop-self
  exit-address-family

! On PE2
router bgp 65000
 bgp router-id 10.10.10.2
 neighbor 10.10.10.1 remote-as 65000
 neighbor 10.10.10.1 update-source Lo0

 address-family vpnv4 unicast
  neighbor 10.10.10.1 activate
  neighbor 10.10.10.1 send-community Both
  neighbor 10.10.10.1 next-hop-self
  exit-address-family


**\*\*Configure OSPF on PE1 and PE2 Backbone\*\***

! On PE1

router ospf 1

network 192.168.100.0 0.0.0.3 area 0

network 10.10.10.1 0.0.0.0 area 0


! On PE2

router ospf 1

network 192.168.100.0 0.0.0.3 area 0

network 10.10.10.2 0.0.0.0 area 0


**\*\*Configure MPLS IN PE1 PE2 \*\***



PE1# inter e1/0

mpls ip

PE2# inter e1/0

mpls ip




```

> MP-BGP carries VPNv4 routes (RD + prefix) between PE routers
> send-community extended is MANDATORY — carries RT values
> Without it routes are exchanged but RT is stripped → no import

\---

### Task 8 — Redistribute CE Routes into MP-BGP

```
! On PE1 and PE2 — redistribute each VRF protocol into BGP

router bgp 65000
 ! VRF BLUE — redistribute EIGRP into BGP
 address-family ipv4 vrf BLUE
  redistribute eigrp 100
 exit-address-family

 ! VRF RED — already using BGP, routes auto in VPNv4
 address-family ipv4 vrf RED
  redistribute connected
 exit-address-family

 ! VRF GREEN — redistribute OSPF into BGP
 address-family ipv4 vrf GREEN
  redistribute ospf 2
 exit-address-family
```

\---

### Task 9 — Redistribute MP-BGP back into CE protocols

```
! On PE1 and PE2

! VRF BLUE — redistribute BGP back into EIGRP
router eigrp BLUE
 address-family ipv4 vrf BLUE autonomous-system 100

Topologie Base
  redistribute bgp 65000 metric 1 1 1 1 1 
 exit-address-family

! VRF GREEN — redistribute BGP back into OSPF
router ospf 2 vrf GREEN
 redistribute bgp 65000 subnets
```

\---

### Task 10 — ⭐ Extra Challenge: Route Leaking Between VRFs

Real world: TechBlue (BLUE) needs access to a shared service
hosted in a separate VRF called SHARED on PE1.

```
! Create SHARED VRF
ip vrf SHARED
 rd 65000:999
 route-target export 65000:999
 route-target import 65000:999
 route-target import 65000:100



! Step 2 — Interface facing shared server

interface Ethernet0/3

ip vrf forwarding SHARED

ip address 10.0.9.2 255.255.255.0

no shutdown



! Step 3 — Advertise into BGP

router bgp 65000

address-family ipv4 vrf SHARED

redistribute connected

redistribute static

exit-address-family

! Add import of SHARED RT into BLUE VRF
ip vrf BLUE
 route-target import 65000:999





## On the Shared Server Router



! This is a CE router simulating a shared server

interface Ethernet0/0

ip address 10.0.9.1 255.255.255.0

no shutdown



! Loopback simulates the actual server/service

interface Loopback0

ip address 10.99.99.1 255.255.255.255

no shutdown



! Static route pointing to PE1

ip route 0.0.0.0 0.0.0.0 10.0.9.2


**then add in PE1**


ip route vrf SHARED 10.99.99.1 255.255.255.255 10.0.9.1


### That is All You Need on PE2

No need to create SHARED VRF on PE2 ✅

No need to configure anything else ✅



Because:

PE1 exports SHARED routes with RT 65000:999

MP-BGP carries them to PE2

PE2 VRF BLUE imports RT 65000:999 ✅

BLUE-CE2 can now reach 10.99.99.1 ✅



! Result: BLUE customers can reach SHARED services
!         RED and GREEN cannot — isolation maintained
! Verify:
  show ip route vrf BLUE   → should see SHARED routes
  show ip route vrf RED    → should NOT see SHARED routes
```

\---

### Task 11 — ⭐ Extra Challenge: BGP as-override Issue Simulation

Simulate what happens WITHOUT as-override and document it.

```
! Step 1 — Remove as-override from PE1
router bgp 65000
 address-family ipv4 vrf RED
  no neighbor 10.0.2.1 as-override

! Step 2 — Check RED-CE2 routing table
RED-CE2# show ip bgp
→ Routes from RED-CE1 (AS 65001) are MISSING
  Because RED-CE2 sees AS 65001 in path and drops it

! Step 3 — Add as-override back
router bgp 65000
 address-family ipv4 vrf RED
  neighbor 10.0.2.1 as-override

! Step 4 — Verify routes return
RED-CE2# show ip bgp
→ Routes from RED-CE1 now visible ✅

! Document: why as-override is needed in VRF BGP PE-CE design
```

\---

### Task 12 — ⭐ Extra Challenge: Verify VRF Isolation

This is the most important real-world requirement.

```
! Ping from BLUE-CE1 to RED-CE1 LAN → must FAIL
BLUE-CE1# ping 172.16.20.1
→ FAIL ✅ (isolation working)

! Ping from BLUE-CE1 to GREEN-CE1 LAN → must FAIL
BLUE-CE1# ping 172.16.30.1
→ FAIL ✅ (isolation working)

! Ping from BLUE-CE1 to BLUE-CE2 LAN → must SUCCEED
BLUE-CE1# ping 172.16.11.1
→ SUCCESS ✅ (same customer reachability working)

! On PE1 verify routing tables are separate
PE1# show ip route vrf BLUE
PE1# show ip route vrf RED
PE1# show ip route vrf GREEN
→ Each table has ONLY that customer's routes
→ No cross-contamination between VRFs
```

\---

### Task 13 — Final Verification

```
! VRF routing tables on PE1
PE1# show ip route vrf BLUE
PE1# show ip route vrf RED
PE1# show ip route vrf GREEN

! MP-BGP VPNv4 table
PE1# show bgp vpnv4 unicast all
PE1# show bgp vpnv4 unicast all summary

! Per-VRF BGP neighbors
PE1# show bgp vpnv4 unicast vrf BLUE
PE1# show bgp vpnv4 unicast vrf RED
PE1# show bgp vpnv4 unicast vrf GREEN

! CE routing tables
BLUE-CE1# show ip route eigrp
RED-CE1#  show ip bgp
GREEN-CE1# show ip route ospf

! End-to-end reachability
BLUE-CE1#  ping 172.16.11.1   ← BLUE-CE2 LAN ✅
RED-CE1#   ping 172.16.21.1   ← RED-CE2 LAN  ✅
GREEN-CE1# ping 172.16.31.1   ← GREEN-CE2 LAN ✅
```

\---

## ✅ Expected Verification Output

```
PE1# show bgp vpnv4 unicast all summary
Neighbor        V  AS    MsgRcvd  State/PfxRcd
192.168.100.2   4  65000   45     6  ← PE2 ✅

PE1# show ip route vrf BLUE
D  172.16.10.0/24 \\\\\\\[90/...] via 10.0.1.1   ← BLUE-CE1 LAN (EIGRP)
B  172.16.11.0/24 \\\\\\\[200/..] via 192.168.100.2 ← BLUE-CE2 LAN (via MP-BGP)

PE1# show ip route vrf RED
B  172.16.20.0/24 \\\\\\\[20/0] via 10.0.2.1    ← RED-CE1 LAN (eBGP)
B  172.16.21.0/24 \\\\\\\[200/0] via 192.168.100.2 ← RED-CE2 LAN (MP-BGP)

PE1# show ip route vrf GREEN
O  172.16.30.0/24 \\\\\\\[110/..] via 10.0.3.1  ← GREEN-CE1 LAN (OSPF)
B  172.16.31.0/24 \\\\\\\[200/..] via 192.168.100.2 ← GREEN-CE2 LAN (MP-BGP)

BLUE-CE1# ping 172.16.11.1
!!!!!  ← SUCCESS ✅

BLUE-CE1# ping 172.16.20.1
.....  ← FAIL ✅ (VRF isolation working)
```

\---

## 🐛 Troubleshooting Log

### Problem 1 — IP Address Removed After Adding VRF

```
Symptom : Interface has no IP after ip vrf forwarding command
Cause   : IOS removes IP automatically when VRF is added
Fix     : Always configure VRF FIRST then IP address
          ip vrf forwarding BLUE    ← first
          ip address 10.0.1.2 ...   ← then IP
```

### Problem 2 — MP-BGP Routes Not Exchanged Between PEs

```
Symptom : show bgp vpnv4 unicast all → no prefixes from PE2
Cause   : send-community extended missing on BGP neighbor
Fix     : neighbor 10.10.10.2 send-community extended
          Without this RT is stripped → routes not imported into VRF
```

### Problem 3 — EIGRP Routes Not in VRF Table

```
Symptom : show ip route vrf BLUE → no EIGRP routes
Cause   : EIGRP not configured under address-family ipv4 vrf BLUE
Fix     : router eigrp 100
           address-family ipv4 vrf BLUE autonomous-system 100            
            network ...
```

### Problem 4 — RED-CE2 Not Receiving RED-CE1 Routes

```
Symptom : show ip bgp on RED-CE2 → routes from CE1 missing
Cause   : AS-PATH loop prevention — CE2 sees CE1 AS in path
Fix     : neighbor x.x.x.x as-override on BOTH PE1 and PE2
          This replaces customer AS with provider AS in advertisement
```

### Problem 5 — OSPF Routes Not Redistributed into MP-BGP

```
Symptom : GREEN VRF routes not appearing in VPNv4 table
Cause   : redistribute ospf 2 subnets missing "subnets" keyword
Fix     : redistribute bgp 65000 subnets  ← subnets keyword required
          Without it only classful routes are redistributed
```

### Problem 6 — VRF Isolation Broken (Routes Leaking)

```
Symptom : BLUE customer can see RED customer routes
Cause   : RT import misconfigured — imported wrong RT
Fix     : Verify each VRF imports ONLY its own RT
          ip vrf BLUE → route-target import 65000:100 ONLY
          ip vrf RED  → route-target import 65000:200 ONLY
```

\---

## 📚 Key Concepts Learned

### VRF-Lite vs Full MPLS VPN

```
Full MPLS VPN:
  Uses label switching between PE routers
  Requires LDP or RSVP-TE
  Scalable to thousands of VRFs

VRF-Lite:
  No label switching — uses routing only
  PE routers connected directly (no P routers)
  Simpler but less scalable
  Perfect for enterprise without SP infrastructure
  Same VRF + RD + RT concept applies
```

### Route Distinguisher vs Route Target

```
RD (Route Distinguisher):
  Makes routes UNIQUE in the VPNv4 table
  Format: ASN:number (65000:100)
  Does NOT control which VRF gets the route
  Just a prefix for uniqueness

RT (Route Target):
  Controls which VRF IMPORTS which routes
  Export: tag routes when leaving VRF
  Import: accept routes with matching tag
  This is what actually controls isolation
```

### Why as-override in PE-CE BGP

```
Without as-override:
  RED-CE1 (AS 65001) advertises to PE1
  PE1 sends to PE2 via MP-BGP
  PE2 advertises to RED-CE2 (AS 65001)
  RED-CE2 sees AS 65001 in path
  RED-CE2 thinks it is a loop → drops route

With as-override:
  PE replaces 65001 with 65000 in AS-PATH
  RED-CE2 sees only 65000 → accepts route
```

### Redistributing into MP-BGP

```
EIGRP → BGP:  redistribute eigrp 100
OSPF  → BGP:  redistribute ospf 2 subnets  ← subnets is critical
BGP   → EIGRP: redistribute bgp 65000 metric 1 1 1 1 1
BGP   → OSPF:  redistribute bgp 65000 subnets
```

\---

## 🧠 Interview Questions This Lab Covers

* What is the difference between RD and RT in MPLS VPN?
* What is VRF-Lite and how does it differ from full MPLS VPN?
* Why is send-community extended mandatory in MP-BGP?
* What happens if you add VRF to an interface that already has an IP?
* Why do you need as-override in PE-CE BGP design?
* What does the subnets keyword do in OSPF redistribution?
* How do you verify VRF isolation is working correctly?
* How would you configure route leaking between two VRFs?
* What routing protocols can run inside a VRF?
* How does RT import/export control customer isolation?

