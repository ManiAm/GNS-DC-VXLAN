# BGP EVPN (Ethernet VPN)

BGP EVPN (Ethernet Virtual Private Network) is a standards-based control plane that distributes Layer 2 and Layer 3 reachability information using the Border Gateway Protocol (BGP). It is the industry-standard control plane for VXLAN in modern data center fabrics.

This document explains EVPN step by step. Since EVPN depends on BGP and VXLAN concepts, those prerequisites are explained first.

**Prerequisites:** This document assumes familiarity with VLANs (04_vlan.md) and VXLAN (05_vxlan.md).

---

## 1. Prerequisites: BGP Fundamentals

EVPN is a BGP address family. Before understanding EVPN, you must understand these BGP concepts:

### 1.1 What Is BGP?

BGP (Border Gateway Protocol) is a path-vector routing protocol used to exchange routing information between autonomous systems (AS) on the Internet. In data centers, BGP is also used internally (within a single organization) to distribute routing information between switches.

**Two BGP modes:**
- **eBGP (External BGP):** Peering between routers in different autonomous systems. Each leaf and spine can have its own AS number.
- **iBGP (Internal BGP):** Peering between routers in the same autonomous system. Requires route reflectors to avoid full-mesh peering.

### 1.2 BGP Address Families

BGP is extensible. It can carry routing information for different protocols using "address families" (AFI/SAFI):

| AFI/SAFI | Name | Purpose |
|----------|------|---------|
| AFI 1, SAFI 1 | IPv4 Unicast | Standard IPv4 routing |
| AFI 2, SAFI 1 | IPv6 Unicast | Standard IPv6 routing |
| AFI 25, SAFI 70 | L2VPN EVPN | Layer 2 VPN using EVPN |

EVPN uses AFI 25, SAFI 70. This means BGP carries MAC/IP reachability information alongside (or instead of) traditional IP routes.

### 1.3 BGP Communities

BGP communities are tags attached to routes that convey additional information. EVPN heavily uses:
- **Route Target (RT):** Determines which VTEPs import a given route. A VTEP only imports routes with matching RTs.
- **Route Distinguisher (RD):** Makes routes unique when the same MAC/IP exists in multiple VNIs. Format: IP:VNI or ASN:VNI.

### 1.4 Route Reflectors

In iBGP, every router must peer with every other router (full mesh) -- this doesn't scale. A route reflector (RR) solves this:
- All BGP speakers peer only with the RR.
- The RR "reflects" routes from one peer to all others.
- In a spine-leaf fabric, the spines typically serve as route reflectors.

In eBGP designs, route reflectors are not needed because each eBGP peer naturally re-advertises routes.

### 1.5 BGP in Spine-Leaf Fabrics

Two common designs:

**Option A -- iBGP with Route Reflectors:**
- All leaves share one AS (e.g., AS 65000).
- Spines are route reflectors.
- Leaves peer with spines for both underlay (IPv4) and overlay (EVPN).

**Option B -- eBGP (each device has unique AS):**
- Each leaf has a unique AS (e.g., leaf-1 = AS 65001, leaf-2 = AS 65002).
- Each spine has a unique AS (e.g., spine-1 = AS 65100).
- No route reflectors needed.
- More operationally simple at scale.

---

## 2. Why EVPN? (The Problem Without It)

Without a control plane, VXLAN uses "flood and learn":

1. Host-A wants to talk to Host-B (unknown MAC).
2. VTEP-1 floods the frame to ALL remote VTEPs in that VNI.
3. All VTEPs process the frame. Only VTEP-2 (hosting Host-B) can deliver it.
4. VTEP-1 learns the mapping only when Host-B replies (data-plane learning).

**Problems with flood and learn:**
- Wasteful: Every unknown unicast and every ARP is flooded to every VTEP.
- Slow convergence: MAC moves are detected only via data traffic.
- No ARP suppression: Every ARP broadcast floods the entire fabric.
- No multi-homing support: If a host is dual-connected to two VTEPs, there's no way to coordinate.
- No inter-subnet routing optimization: No way to advertise IP routes for VTEPs to route locally.

**EVPN solves all of these.** It provides:
- Proactive MAC/IP advertisement (no flooding for known destinations).
- ARP suppression (VTEPs answer ARP locally).
- Fast convergence on MAC moves.
- Multi-homing with designated forwarder election.
- Integrated L2 and L3 (bridging + routing in one control plane).

---

## 3. How EVPN Works (High Level)

1. A host connects to a leaf switch (VTEP).
2. The VTEP detects the host's MAC and IP (via ARP snooping, ND, or data traffic).
3. The VTEP creates a BGP EVPN Type 2 route: "MAC XX:XX, IP Y.Y.Y.Y is reachable via me (VTEP IP Z.Z.Z.Z) in VNI NNNNN."
4. The VTEP advertises this route to its BGP peers (spines or route reflectors).
5. All other VTEPs in the fabric receive this route and install it in their forwarding tables.
6. When traffic destined for MAC XX:XX arrives at any VTEP, it already knows the destination VTEP and unicasts the encapsulated frame directly. No flooding.

---

## 4. EVPN Route Types

EVPN defines five main route types. Each serves a distinct purpose:

### Type 1: Ethernet Auto-Discovery (EAD)

**Purpose:** Multi-homing support and fast convergence.

**When used:**
- A host (or link aggregation group) is connected to two or more VTEPs simultaneously (e.g., MLAG/ESI-LAG).
- The EAD route advertises the Ethernet Segment (ES) membership.
- Enables "aliasing" -- remote VTEPs can load-balance traffic across both VTEPs connected to the same host.
- Enables fast withdrawal -- if one VTEP loses connectivity to the host, it withdraws its EAD route; remote VTEPs immediately stop sending to it.

**Not needed for single-homed hosts.**

### Type 2: MAC/IP Advertisement

**Purpose:** The core of EVPN. Advertises host reachability.

**What it carries:**
- MAC address of the host.
- IP address of the host (optional but strongly recommended).
- VNI (L2 VNI for bridging, L3 VNI for routing).
- Next-hop: the VTEP IP where the host resides.

**How it works:**
1. VTEP-2 detects Host-B (MAC: BB:BB, IP: 10.0.1.20) on VLAN 100 (VNI 10100).
2. VTEP-2 generates a Type 2 route:
   - RD: 10.1.1.2:10100
   - MAC: BB:BB
   - IP: 10.0.1.20
   - VNI: 10100
   - Next-Hop: 10.1.1.2
   - RT: auto-derived or manually configured
3. All other VTEPs receive this and install: "To reach BB:BB in VNI 10100, send to VTEP 10.1.1.2."

**This eliminates unknown unicast flooding.** Every known MAC has a BGP route pointing to its VTEP.

### Type 3: Inclusive Multicast Ethernet Tag (IMET)

**Purpose:** Builds the ingress replication list for BUM traffic.

**How it works:**
1. When a VTEP has local hosts in a VNI, it advertises a Type 3 route: "I (VTEP 10.1.1.2) have interest in VNI 10100."
2. All VTEPs collect Type 3 routes for each VNI.
3. When a VTEP needs to flood BUM traffic for VNI 10100, it sends a copy to every VTEP that advertised a Type 3 for that VNI.
4. This automatically builds the replication list -- no static configuration needed.

**Without Type 3:** You would have to manually configure the list of remote VTEPs for each VNI on every VTEP.

### Type 4: Ethernet Segment (ES)

**Purpose:** Multi-homing designated forwarder (DF) election.

**When used:**
- Multiple VTEPs are connected to the same host/segment (Ethernet Segment).
- Only ONE VTEP should forward BUM traffic to the host (to avoid duplicates).
- Type 4 routes are exchanged between the VTEPs sharing the ES to elect a Designated Forwarder (DF).

**Not needed for single-homed hosts.**

### Type 5: IP Prefix

**Purpose:** Advertises IP prefixes for inter-subnet routing.

**When used:**
- A VTEP knows about an IP prefix (e.g., from a connected subnet, a static route, or redistribution from another protocol).
- It advertises the prefix as a Type 5 route so other VTEPs can route to it.
- This is how external prefixes (e.g., from a firewall, WAN, or Internet) are injected into the EVPN fabric.

**Example:**
- Border leaf has a connection to the Internet and knows 0.0.0.0/0.
- It advertises a Type 5 route for 0.0.0.0/0 with its VTEP IP as next-hop.
- All other leaves install this as a default route in the tenant VRF.

---

## 5. Route Targets and Route Distinguishers

### Route Distinguisher (RD)

- Makes each EVPN route globally unique.
- Required because the same MAC/IP can exist in multiple VNIs or on multiple VTEPs.
- Typically auto-derived: `<VTEP-IP>:<VNI>` or `<VTEP-IP>:<VLAN>`.
- Example: 10.1.1.1:10100

### Route Target (RT)

- Controls route distribution (who imports which routes).
- Export RT: Attached to a route when it's advertised.
- Import RT: Configured on a VNI/VRF; only routes with matching RT are imported.

**How they work together:**
1. VTEP-1 exports a Type 2 route for VNI 10100 with RT 10100:10100.
2. VTEP-2 has VNI 10100 configured with import RT 10100:10100.
3. VTEP-2 imports the route (it matches).
4. VTEP-3 does NOT have VNI 10100 configured, so it does not import the route.

This ensures a VTEP only installs routes for VNIs it actually hosts -- saving memory and processing.

---

## 6. ARP/ND Suppression

One of EVPN's most impactful features is ARP suppression:

**Without ARP suppression:**
1. Host-A sends an ARP request ("Who has 10.0.1.20?").
2. This is a broadcast --> VTEP-1 floods it to all remote VTEPs in the VNI.
3. All VTEPs deliver the ARP to their local hosts in that VNI.
4. Only Host-B responds.

**With ARP suppression:**
1. Host-A sends an ARP request ("Who has 10.0.1.20?").
2. VTEP-1 intercepts the ARP.
3. VTEP-1 checks its EVPN table: Type 2 route says "IP 10.0.1.20 = MAC BB:BB at VTEP 10.1.1.2."
4. VTEP-1 crafts an ARP reply ("10.0.1.20 is at BB:BB") and sends it back to Host-A locally.
5. The ARP never leaves VTEP-1. Zero flooding.

**Impact:** In a fabric with 1000 hosts, ARP suppression eliminates thousands of broadcast frames per second that would otherwise be replicated to every VTEP.

---

## 7. Symmetric vs. Asymmetric IRB

IRB (Integrated Routing and Bridging) describes how a VTEP handles inter-subnet traffic. Two models exist:

### Asymmetric IRB

1. Ingress VTEP routes the packet (L3 lookup) AND bridges it into the destination VNI.
2. The encapsulated packet carries the destination L2 VNI.
3. Egress VTEP only bridges (L2 lookup) -- it does NOT route.

**Problem:** The ingress VTEP must have BOTH the source and destination VNIs configured. This means every leaf must have every VNI in the fabric -- doesn't scale.

### Symmetric IRB (Recommended)

1. Ingress VTEP routes the packet (L3 lookup) into the tenant VRF.
2. The encapsulated packet carries the L3 VNI (associated with the VRF).
3. Egress VTEP receives the packet on the L3 VNI, routes it into the destination subnet, and bridges to the host.

**Advantage:** Each leaf only needs the VNIs for locally-connected hosts, plus the L3 VNI for the tenant. The ingress leaf does NOT need the destination L2 VNI.

**This is why L3 VNI exists:** It carries inter-subnet traffic between VTEPs in a scalable way.

---

## 8. Multi-Homing with EVPN

In production networks, hosts (especially servers) connect to two leaf switches for redundancy. EVPN handles this elegantly:

### Ethernet Segment (ES)

An Ethernet Segment is the link (or LAG) connecting a host to multiple VTEPs. It has a unique identifier called the ESI (Ethernet Segment Identifier).

### How It Works

1. Both VTEP-1 and VTEP-2 detect the same ESI on their local ports.
2. Both advertise Type 1 (EAD) and Type 4 (ES) routes for this ESI.
3. Remote VTEPs see both routes and know the host is reachable via VTEP-1 AND VTEP-2.
4. Remote VTEPs can load-balance (alias) unicast traffic across both VTEPs.
5. For BUM traffic, a DF election (via Type 4) ensures only ONE VTEP forwards BUM to avoid duplicates.

### Active-Active vs. Active-Standby

- **Active-Active:** Both VTEPs forward traffic simultaneously. The host uses a LAG (LACP) across both. Most common for servers.
- **Active-Standby:** Only the DF forwards; the standby is a backup. Used for devices that cannot do multi-chassis LAG.

---

## 9. MAC Mobility

When a host (or VM) moves from one VTEP to another, EVPN detects and handles it:

1. Host-B was on VTEP-1. VTEP-1 had advertised a Type 2 route for Host-B.
2. Host-B moves to VTEP-2 (e.g., VM live migration).
3. VTEP-2 detects Host-B's MAC, generates a NEW Type 2 route with a higher sequence number (MAC Mobility extended community).
4. All VTEPs receive the new route. The higher sequence number indicates this is a MAC move.
5. VTEPs update their forwarding tables: Host-B is now at VTEP-2.
6. VTEP-1's old route is implicitly withdrawn.

**Fast convergence:** The BGP update propagates in milliseconds. Traffic is rerouted almost instantly.

---

## 10. EVPN in a Spine-Leaf Fabric

Putting it all together in a typical deployment:

```
        [Spine-1 (RR)]     [Spine-2 (RR)]
          /    |    \         /    |    \
         /     |     \       /     |     \
     [Leaf-1] [Leaf-2] [Leaf-3] [Leaf-4]
       |         |         |         |
     Host-A    Host-B    Host-C    Host-D
```

**Underlay:**
- eBGP or OSPF between leaves and spines for IPv4 reachability.
- Each leaf advertises its loopback (VTEP IP) into the underlay.
- ECMP across all spine links.

**Overlay (EVPN):**
- BGP EVPN sessions between each leaf and both spines (using loopback IPs).
- Spines act as route reflectors for the EVPN address family.
- When Host-A connects to Leaf-1:
  - Leaf-1 advertises Type 2 (MAC/IP) and Type 3 (IMET) routes.
  - Spines reflect these to all other leaves.
  - All leaves now know how to reach Host-A.

**Data plane:**
- Host-A sends to Host-C (different leaf, same VNI): Leaf-1 encapsulates with L2 VNI, sends to Leaf-3.
- Host-A sends to Host-D (different leaf, different VNI/subnet): Leaf-1 routes and encapsulates with L3 VNI, sends to Leaf-4.

---

## 11. EVPN + VXLAN vs. Other Approaches

| Approach | Control Plane | Scalability | Flooding | Multi-Homing | Inter-Subnet |
|----------|--------------|-------------|----------|--------------|--------------|
| VXLAN flood-and-learn | None (data plane) | Low | Full flooding | Not supported | External router |
| VXLAN + multicast | PIM | Medium | Multicast-based | Not supported | External router |
| VXLAN + static ingress rep | Manual config | Low | Unicast copies | Not supported | External router |
| **VXLAN + BGP EVPN** | **BGP** | **High** | **Minimized (ARP supp.)** | **Full support** | **Distributed** |

---

## 12. Summary

**EVPN provides:**
1. **MAC/IP learning via control plane** -- no flooding for known unicast.
2. **Automatic replication lists** -- Type 3 routes build BUM flood lists dynamically.
3. **ARP suppression** -- VTEPs answer ARP locally, eliminating broadcast floods.
4. **Multi-homing** -- active-active and active-standby with designated forwarder election.
5. **MAC mobility** -- fast convergence when hosts/VMs move between VTEPs.
6. **Integrated routing** -- Type 5 routes and symmetric IRB enable distributed inter-subnet routing.
7. **Multi-tenancy** -- Route Targets isolate different tenants' routes.

**EVPN is the brain of the VXLAN fabric.** VXLAN provides the data-plane tunnel; EVPN tells VTEPs where to send traffic.

For hands-on configuration using SONiC, see the companion lab document (07_lab.md).
