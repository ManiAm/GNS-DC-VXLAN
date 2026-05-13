
# VXLAN + EVPN Leaf-Spine Fabric

This lab builds a realistic data center fabric using SONiC Virtual Switch (VS) in GNS3. It implements a full VXLAN + BGP EVPN deployment with distributed anycast gateway on a leaf-spine topology.

This lab mirrors production data center deployments:

- **Leaf-spine topology**: the standard data center architecture. Predictable latency, ECMP, linear scalability.

- **eBGP underlay**: used by hyperscalers (Microsoft, LinkedIn, Facebook). Each device has a unique AS. Simple, no route reflectors for underlay.

- **VXLAN**: the overlay data plane. Encapsulates Layer 2 frames inside UDP/IP tunnels, extending virtual networks across the routed underlay.

- **BGP EVPN**: the overlay control plane. No multicast in the underlay, no static VTEP lists. MAC/IP reachability is learned dynamically via BGP.

- **Symmetric IRB**: scales to thousands of VNIs. Leaves only need locally-relevant L2 VNIs plus the L3 VNI per VRF.

- **Distributed anycast gateway**: every leaf is the gateway. No traffic tromboning. Optimal path for every packet.

- **SONiC**: the same NOS running in production at Microsoft Azure, Alibaba, and many other hyperscalers. Open source, vendor-neutral.


## Documentation and Learning Path

Because data center networking spans many interdependent technologies, this project includes a structured set of guides designed to build knowledge progressively. The following documents introduce the underlying concepts step by step, beginning with physical infrastructure and how it is operated, then topology and overlays through hands-on lab configuration.

- **[Data Center](docs/01_README_DC.md):** Physical infrastructure fundamentals — facility design, rack units, power and cooling, and cabling.
- **[Infrastructure Management](docs/02_README_MGMT.md):** In-band vs. out-of-band management — how administrators monitor, configure, and maintain data center hardware.
- **[Network Topology](docs/03_README_TOPOLOGY.md):** Physical and logical topology — rack-level wiring models, the evolution from Three-Tier to Leaf-Spine (Clos), and how traffic flows through a modern fabric.
- **[VLANs](docs/04_vlan.md):** Virtual Local Area Networks — collision and broadcast domains, Layer 2 segmentation, and the scaling challenges that motivate overlay networks.
- **[VXLAN](docs/05_vxlan.md):** Virtual Extensible LAN — the overlay/underlay model, VXLAN encapsulation, VTEPs, and how Layer 2 traffic is tunneled over a Layer 3 fabric.
- **[BGP EVPN](docs/06_evpn.md):** Ethernet VPN control plane — how BGP distributes Layer 2 and Layer 3 reachability information to automate VXLAN tunnel discovery and MAC/IP learning.


## Lab Topology

```
                 +------------+     +------------+
                 |  Spine-1   |     |  Spine-2   |
                 | AS 65100   |     | AS 65200   |
                 +-----+------+     +------+-----+
                  /    |    \         /     |    \
                 /     |     \       /      |     \
                /      |      \     /       |      \
  +--------+  /  +--------+   \ /    +--------+   \  +--------+
  | Leaf-1 |/    | Leaf-2 |    X     | Leaf-3 |    \ | Leaf-4 |
  | AS 65001|    | AS 65002|  / \    | AS 65003|    | AS 65004|
  +----+---+    +----+---+  /   \   +----+---+    +----+---+
       |             |      /     \       |             |
    [Host-A]      [Host-B]        [Host-C]         [Host-D]
    VLAN 100      VLAN 100        VLAN 200         VLAN 200
    10.100.1.10   10.100.1.20     10.200.1.10      10.200.1.20
```

**Devices:**
- 2 Spine switches (SONiC VS) -- pure L3 routers, BGP route reflectors for EVPN
- 4 Leaf switches (SONiC VS) -- VTEPs, perform VXLAN encap/decap
- 4 Hosts (Linux containers or lightweight VMs)

**Design decisions:**
- eBGP underlay (each device has a unique AS)
- iBGP overlay for EVPN (spines as route reflectors, leaves as clients with AS 65000 for overlay)
- Symmetric IRB with L3 VNI for inter-subnet routing
- Distributed anycast gateway on all leaves

---

## IP Addressing Plan

### Loopbacks (VTEP source IPs)

| Device | Loopback0 IP | AS Number |
|--------|-------------|-----------|
| Spine-1 | 10.0.0.1/32 | 65100 |
| Spine-2 | 10.0.0.2/32 | 65200 |
| Leaf-1 | 10.0.0.11/32 | 65001 |
| Leaf-2 | 10.0.0.12/32 | 65002 |
| Leaf-3 | 10.0.0.13/32 | 65003 |
| Leaf-4 | 10.0.0.14/32 | 65004 |

### Point-to-Point Links (Underlay)

| Link | Subnet | Device A IP | Device B IP |
|------|--------|-------------|-------------|
| Spine-1 <-> Leaf-1 | 10.0.1.0/31 | 10.0.1.0 | 10.0.1.1 |
| Spine-1 <-> Leaf-2 | 10.0.1.2/31 | 10.0.1.2 | 10.0.1.3 |
| Spine-1 <-> Leaf-3 | 10.0.1.4/31 | 10.0.1.4 | 10.0.1.5 |
| Spine-1 <-> Leaf-4 | 10.0.1.6/31 | 10.0.1.6 | 10.0.1.7 |
| Spine-2 <-> Leaf-1 | 10.0.2.0/31 | 10.0.2.0 | 10.0.2.1 |
| Spine-2 <-> Leaf-2 | 10.0.2.2/31 | 10.0.2.2 | 10.0.2.3 |
| Spine-2 <-> Leaf-3 | 10.0.2.4/31 | 10.0.2.4 | 10.0.2.5 |
| Spine-2 <-> Leaf-4 | 10.0.2.6/31 | 10.0.2.6 | 10.0.2.7 |

### Overlay (Tenant Networks)

| VLAN | Subnet | VNI (L2) | Gateway IP (Anycast) | Purpose |
|------|--------|----------|---------------------|---------|
| 100 | 10.100.1.0/24 | 10100 | 10.100.1.1 | Tenant-A Subnet 1 |
| 200 | 10.200.1.0/24 | 10200 | 10.200.1.1 | Tenant-A Subnet 2 |

| VRF | L3 VNI | Purpose |
|-----|--------|---------|
| VrfTenantA | 50000 | Tenant-A inter-subnet routing |

---

## Step-by-Step Configuration

SONiC uses FRR (Free Range Routing) for BGP and configuration is done via:
- `config` CLI commands for interface/VLAN/VXLAN setup
- `vtysh` (FRR shell) for BGP/routing configuration

### 4.1 Configure Interfaces and MTU (All Devices)

Run on each device. Example shown for Leaf-1:

```bash
# Set hostname
sudo hostnamectl set-hostname leaf-1

# Configure interface IPs (underlay point-to-point links)
sudo config interface ip add Ethernet0 10.0.1.1/31    # To Spine-1
sudo config interface ip add Ethernet4 10.0.2.1/31    # To Spine-2

# Configure Loopback
sudo config interface ip add Loopback0 10.0.0.11/32

# Set MTU to 9216 on all fabric-facing interfaces (for VXLAN overhead)
sudo config interface mtu Ethernet0 9216
sudo config interface mtu Ethernet4 9216
```

Repeat for all devices with their respective IPs from the addressing plan.

### 4.2 Configure Underlay BGP (eBGP)

Enter the FRR shell and configure eBGP on each device.

**Leaf-1 (`vtysh`):**

```
configure terminal

router bgp 65001
  bgp router-id 10.0.0.11
  no bgp ebgp-requires-policy
  bgp bestpath as-path multipath-relax

  neighbor 10.0.1.0 remote-as 65100
  neighbor 10.0.2.0 remote-as 65200

  address-family ipv4 unicast
    network 10.0.0.11/32
    neighbor 10.0.1.0 activate
    neighbor 10.0.2.0 activate
  exit-address-family
exit
```

**Spine-1 (`vtysh`):**

```
configure terminal

router bgp 65100
  bgp router-id 10.0.0.1
  no bgp ebgp-requires-policy
  bgp bestpath as-path multipath-relax

  neighbor 10.0.1.1 remote-as 65001
  neighbor 10.0.1.3 remote-as 65002
  neighbor 10.0.1.5 remote-as 65003
  neighbor 10.0.1.7 remote-as 65004

  address-family ipv4 unicast
    network 10.0.0.1/32
    neighbor 10.0.1.1 activate
    neighbor 10.0.1.3 activate
    neighbor 10.0.1.5 activate
    neighbor 10.0.1.7 activate
  exit-address-family
exit
```

Repeat pattern for Spine-2 and Leaf-2/3/4 with their respective IPs and AS numbers.

### 4.3 Verify Underlay Connectivity

```bash
# Check BGP neighbors are established
show ip bgp summary

# Verify all loopbacks are reachable
ping 10.0.0.11   # from any device to leaf-1
ping 10.0.0.12   # to leaf-2
ping 10.0.0.13   # to leaf-3
ping 10.0.0.14   # to leaf-4
```

All loopbacks must be reachable before proceeding to overlay configuration.

### 4.4 Configure VLANs on Leaf Switches

On Leaf-1 and Leaf-2 (hosting VLAN 100):

```bash
# Create VLAN 100
sudo config vlan add 100

# Assign host-facing port to VLAN 100 (untagged/access)
sudo config vlan member add 100 Ethernet8 --untagged
```

On Leaf-3 and Leaf-4 (hosting VLAN 200):

```bash
# Create VLAN 200
sudo config vlan add 200

# Assign host-facing port to VLAN 200 (untagged/access)
sudo config vlan member add 200 Ethernet8 --untagged
```

### 4.5 Configure VXLAN Tunnel (VTEP)

On each leaf switch, create the VXLAN tunnel interface mapped to the loopback:

```bash
# Create VXLAN tunnel with source as Loopback0
sudo config vxlan add vtep 10.0.0.11    # Use leaf's own loopback IP
```

Map VLANs to VNIs:

**On Leaf-1 and Leaf-2:**

```bash
sudo config vxlan map add vtep vlan 100 vni 10100
```

**On Leaf-3 and Leaf-4:**

```bash
sudo config vxlan map add vtep vlan 200 vni 10200
```

For inter-subnet routing (all leaves need both VNIs eventually; or use L3 VNI):

```bash
# On ALL leaves -- map L3 VNI for VRF routing
sudo config vxlan map add vtep vlan 1000 vni 50000
```

VLAN 1000 is a "dummy" VLAN used internally to carry the L3 VNI (this is SONiC's convention for symmetric IRB).

### 4.6 Configure VRF for Tenant

On all leaf switches:

```bash
# Create VRF
sudo config vrf add VrfTenantA

# Bind VLAN interfaces to VRF
sudo config interface vrf bind Vlan100 VrfTenantA    # Only on Leaf-1, Leaf-2
sudo config interface vrf bind Vlan200 VrfTenantA    # Only on Leaf-3, Leaf-4
sudo config interface vrf bind Vlan1000 VrfTenantA   # All leaves (L3 VNI VLAN)
```

### 4.7 Configure SVI (Anycast Gateway)

The anycast gateway IP and MAC must be identical on all leaves for a given VLAN.

**On Leaf-1 and Leaf-2:**

```bash
# Configure SVI with anycast gateway IP
sudo config interface ip add Vlan100 10.100.1.1/24

# Set the same virtual MAC on all leaves (SONiC anycast gateway MAC)
# This is configured globally in /etc/sonic/config_db.json or via:
sudo config interface ip add Vlan1000 10.0.0.11/32   # L3 VNI loopback (unique per leaf)
```

**On Leaf-3 and Leaf-4:**

```bash
sudo config interface ip add Vlan200 10.200.1.1/24
sudo config interface ip add Vlan1000 10.0.0.13/32   # or .14 for Leaf-4
```

For the anycast gateway virtual MAC, edit `/etc/sonic/config_db.json` on each leaf:

```json
{
  "SAG_GLOBAL": {
    "IP": {
      "gateway_mac": "00:00:00:01:02:03"
    }
  }
}
```

Then apply: `sudo config reload`

This ensures all leaves use the same gateway MAC for gateway IPs -- hosts can migrate between leaves without needing to re-ARP.

### 4.8 Configure BGP EVPN Overlay

EVPN sessions run between leaves via the spines (using loopback-to-loopback iBGP with spines as route reflectors for the EVPN AF).

**Spine-1 (`vtysh`) -- add EVPN route reflector config:**

```
configure terminal

router bgp 65100
  neighbor LEAF peer-group
  neighbor LEAF remote-as external
  neighbor LEAF update-source Loopback0

  # Use the underlay-learned loopback IPs for EVPN peering
  # Alternative: configure overlay iBGP separately (shown below)
exit

# For EVPN, spines act as iBGP RR. Create a separate BGP instance or use the same:
# Simplest approach for SONiC: use eBGP for EVPN too (no RR needed)

router bgp 65100
  address-family l2vpn evpn
    neighbor 10.0.1.1 activate
    neighbor 10.0.1.3 activate
    neighbor 10.0.1.5 activate
    neighbor 10.0.1.7 activate
  exit-address-family
exit
```

**Leaf-1 (`vtysh`) -- add EVPN config:**

```
configure terminal

router bgp 65001
  address-family l2vpn evpn
    neighbor 10.0.1.0 activate
    neighbor 10.0.2.0 activate
    advertise-all-vni
  exit-address-family

  # Advertise the VRF subnets into EVPN as Type 5
  address-family ipv4 unicast vrf VrfTenantA
    redistribute connected
  exit-address-family
exit
```

The `advertise-all-vni` command tells FRR to automatically advertise all locally configured VNIs as EVPN Type 2 and Type 3 routes.

Repeat for all leaves and spines with their respective neighbor IPs.

### 4.9 Configure Hosts

Hosts are simple Linux machines (or network namespaces in GNS3):

**Host-A (connected to Leaf-1, VLAN 100):**

```bash
sudo ip addr add 10.100.1.10/24 dev eth0
sudo ip route add default via 10.100.1.1
```

**Host-B (connected to Leaf-2, VLAN 100):**

```bash
sudo ip addr add 10.100.1.20/24 dev eth0
sudo ip route add default via 10.100.1.1
```

**Host-C (connected to Leaf-3, VLAN 200):**

```bash
sudo ip addr add 10.200.1.10/24 dev eth0
sudo ip route add default via 10.200.1.1
```

**Host-D (connected to Leaf-4, VLAN 200):**

```bash
sudo ip addr add 10.200.1.20/24 dev eth0
sudo ip route add default via 10.200.1.1
```

---

## Verification and Testing

### 5.1 Verify BGP EVPN Neighbors

```bash
# On any leaf, enter vtysh:
show bgp l2vpn evpn summary
```

Expected: All spine neighbors established, routes being exchanged.

### 5.2 Verify VXLAN Tunnel

```bash
show vxlan interface
show vxlan vlanvnimap
show vxlan tunnel
show vxlan remotevtep
```

Expected: Remote VTEPs discovered, VNI mappings correct.

### 5.3 Verify EVPN Routes

```bash
# Show all EVPN routes
show bgp l2vpn evpn

# Show Type 2 (MAC/IP) routes
show bgp l2vpn evpn route type macip

# Show Type 3 (IMET) routes
show bgp l2vpn evpn route type multicast

# Show Type 5 (IP prefix) routes
show bgp l2vpn evpn route type prefix
```

### 5.4 Test L2 Connectivity (Same Subnet)

From Host-A, ping Host-B (both in VLAN 100, same subnet, different leaves):

```bash
# On Host-A:
ping 10.100.1.20
```

Expected: Ping succeeds. Traffic flows: Host-A -> Leaf-1 (encap VXLAN VNI 10100) -> Spine -> Leaf-2 (decap) -> Host-B.

### 5.5 Test L3 Connectivity (Different Subnet)

From Host-A, ping Host-C (VLAN 100 to VLAN 200, different subnets):

```bash
# On Host-A:
ping 10.200.1.10
```

Expected: Ping succeeds. Traffic flows: Host-A -> Leaf-1 (route in VrfTenantA, encap VXLAN L3 VNI 50000) -> Spine -> Leaf-3 (decap, route to VLAN 200) -> Host-C.

### 5.6 Verify MAC Learning via EVPN

```bash
# On Leaf-1:
show mac address-table

# Should show Host-A's MAC as dynamic/local
# Should show Host-B's MAC learned via EVPN (remote VTEP)
```

### 5.7 Verify ARP Suppression

```bash
# Check if the switch has cached remote host's ARP
show arp
# Or in FRR:
show evpn arp-cache vni 10100
```

---

## Extending the Lab

### Add a new tenant (VRF)

1. Create new VRF (e.g., VrfTenantB).
2. Create new VLANs, map to new L2 VNIs.
3. Create a new L3 VNI for the VRF.
4. Configure SVIs with anycast gateway.
5. Route targets keep tenants isolated automatically.

### Add border leaf (external connectivity)

1. Designate one leaf as a border leaf.
2. Connect it to an external router/firewall.
3. Advertise a default route (0.0.0.0/0) as a Type 5 EVPN route into the VRF.
4. All other leaves learn the default route via EVPN.

### Test VM mobility

1. Move Host-A from Leaf-1 to Leaf-3 (change the GNS3 connection).
2. Configure VLAN 100 on Leaf-3.
3. Observe EVPN Type 2 route update (MAC mobility sequence number increments).
4. Verify traffic continues flowing without changing Host-A's IP.

### Add MLAG / multi-homing

1. Connect a host to two leaves via a LAG.
2. Configure the same ESI on both leaves.
3. Observe Type 1 and Type 4 routes.
4. Verify active-active load balancing from remote VTEPs.

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| No BGP EVPN neighbors | Verify underlay reachability (ping loopbacks). Check `show bgp l2vpn evpn summary`. |
| VXLAN tunnel not forming | Verify VNI mapping (`show vxlan vlanvnimap`). Check that `advertise-all-vni` is set. |
| L2 ping fails (same subnet) | Check VLAN membership, check EVPN Type 2/3 routes exist. Verify MTU (must be >= 1550 on underlay). |
| L3 ping fails (different subnet) | Check VRF config, L3 VNI mapping, SVI IPs. Verify Type 5 routes or connected route redistribution. |
| ARP not resolving | Check gateway MAC consistency (SAG). Verify ARP suppression entries. |
| Asymmetric traffic (works one way) | Check that both leaves have correct VNI and VRF config. Verify route targets match. |

---

## Key SONiC Commands Reference

| Task | Command |
|------|---------|
| Show interfaces | `show interfaces status` |
| Show VLANs | `show vlan brief` |
| Show VXLAN config | `show vxlan interface` |
| Show VXLAN VNI map | `show vxlan vlanvnimap` |
| Show remote VTEPs | `show vxlan remotevtep` |
| Show MAC table | `show mac address-table` |
| Show ARP | `show arp` |
| Show BGP summary | `show ip bgp summary` (underlay) |
| Show EVPN summary | `show bgp l2vpn evpn summary` |
| Show EVPN routes | `show bgp l2vpn evpn` |
| Show VRF | `show vrf` |
| Enter FRR shell | `vtysh` |
| Save config | `sudo config save -y` |
| Reload config | `sudo config reload` |
