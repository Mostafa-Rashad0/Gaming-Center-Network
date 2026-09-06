## you do NOT have to real all that, just give it to any AI and ask him about what you specifically want to know in the project 


# Gaming Center Network -- Cisco Packet Tracer

A complete enterprise-style gaming center network designed and
implemented in **Cisco Packet Tracer**.

The project was built progressively from the switching layer to
inter-VLAN routing, WAN connectivity, NAT/PAT, DHCP, wireless access,
access control, device management, and a simulated gaming web portal.

> **Current status:** Core network v1 is implemented and operational, with a Security Hardening v2 layer partially completed.
> The design can be extended later with additional security and infrastructure features.

------------------------------------------------------------------------

## 1. Project Overview

The Gaming Center network is designed to provide:

-   Dedicated gaming users network
-   Staff network
-   Guest Wi-Fi network
-   Services network
-   Management network
-   Inter-VLAN routing
-   Centralized DHCP
-   Layer 2 redundancy with STP
-   Link aggregation using LACP EtherChannels
-   Secure switch management using SSH
-   Edge routing toward an ISP
-   NAT/PAT for Internet access
-   Guest network isolation using an extended ACL
-   Public-facing network/server
-   Wireless guest access
-   A simulated gaming web portal hosted on Packet Tracer Server-PT

The project was intentionally implemented in stages so that each major
network function could be configured and verified before moving to the
next one.

------------------------------------------------------------------------

# 2. Final Topology

``` text
                         DNS NETWORK                 PUBLIC NETWORK
                          8.8.8.0/24                 198.51.100.0/24
                              |                              |
                         DNS Server                   Public Server
                         8.8.8.8                             |
                              |                            G0/0
                              +----------- ISP-R1 ------------
                                           G0/1
                                             |
                     203.0.113.0/24 WAN / ISP
                                    |
                                  G0/1
                               EDGE-R1
                                  G0/2
                                    |
                              10.0.0.0/30
                                    |
                                  G0/2
                                L3-SW1
                               /       \
                         Po1 /           \ Po2
                           /               \
                    ACCESS-SW1          ACCESS-SW2
                    /   |   \             |    \
                  PCs  AP1  ...           PCs   AP2
                       |                         |
                  Guest Wi-Fi               Guest Wi-Fi
```

### Main devices

  Device          Role
  --------------- -------------------------------------------------------------
  L3-SW1          Core multilayer switch, inter-VLAN routing, default gateway
  ACCESS-SW1      Access switch
  ACCESS-SW2      Access switch
  EDGE-R1         Enterprise edge router, NAT/PAT, WAN gateway
  ISP-R1          ISP-side router
  AP1             Guest wireless access point
  AP2             Guest wireless access point
  Public Server   Public-facing Packet Tracer server
  DNS Server      Internal DNS service simulated on the public/ISP side
  PCs/Laptops     Gaming, staff, and guest clients

------------------------------------------------------------------------

# 3. IP Addressing and VLAN Plan

The internal network uses the `192.168.0.0/16` address space.

    VLAN Purpose       Network             Default Gateway
  ------ ------------- ------------------- -----------------
      10 Gaming        `192.168.10.0/24`   `192.168.10.1`
      20 Staff         `192.168.20.0/24`   `192.168.20.1`
      30 Guest Wi-Fi   `192.168.30.0/24`   `192.168.30.1`
      40 Services      `192.168.40.0/24`   `192.168.40.1`
      99 Management    `192.168.99.0/24`   `192.168.99.1`

### Routed/WAN networks

  Network             Purpose
  ------------------- -------------------------------------
  `10.0.0.0/30`       L3-SW1 ↔ EDGE-R1 routed transit
  `203.0.113.0/24`    EDGE-R1 ↔ ISP-R1 WAN/ISP segment
  `198.51.100.0/24`   Public network / public server side
  `8.8.8.0/24`        DNS network / DNS server side

### Important addressing

``` text
L3-SW1 G0/2     10.0.0.2/30
EDGE-R1 G0/2    10.0.0.1/30

EDGE-R1 G0/1    203.0.113.2/24
ISP-R1 G0/1     203.0.113.1/24

ISP-R1 G0/0     198.51.100.1/24
DNS Server      8.8.8.8/24
```

The `198.51.100.0/24` segment is labeled **PUBLIC NETWORK** because it
is the public-server side of the ISP router. The `203.0.113.0/24`
segment is the ISP/WAN transit segment between the ISP and enterprise
edge router. A separate `8.8.8.0/24` segment was added on the ISP side
to simulate a DNS service at `8.8.8.8`.

------------------------------------------------------------------------

# 4. Implementation Sequence

## Phase 1 -- Layer 2 Switching

### 4.1 Three-Switch Topology

The initial switching topology was built as:

``` text
L3-SW1
   |
ACCESS-SW1
   |
ACCESS-SW2
```

L3-SW1 was selected as the central multilayer switch and the access
switches were placed downstream.

------------------------------------------------------------------------

## 4.2 VTP Configuration

VTP was initially configured to simplify the VLAN deployment process:

- L3-SW1 was configured as the VTP Server.
- ACCESS-SW1 was configured as a VTP Client.
- ACCESS-SW2 was configured as a VTP Client.

The VLANs were created and named on the VTP Server, and the VLAN configuration was successfully propagated to the client switches.

The VLANs deployed through VTP were:

```text
VLAN 10 → GAMING
VLAN 20 → STAFF
VLAN 30 → GUEST-WIFI
VLAN 40 → SERVICES
VLAN 99 → MANAGEMENT
```

After the VLAN creation and propagation phase was completed, VTP was changed to **Transparent mode on all three switches**.

This was an intentional security and design decision. In the final configuration, VLAN information is maintained locally on each switch rather than relying on continued VTP propagation.

### Final VTP Configuration

```text
L3-SW1       → VTP Transparent
ACCESS-SW1   → VTP Transparent
ACCESS-SW2   → VTP Transparent
```

```cisco
vtp mode transparent
```

Therefore, VTP Server/Client was used successfully for the **initial VLAN deployment**, while VTP Transparent is the **final operating mode** of the network.

------------------------------------------------------------------------

# 5. VLAN Creation and User Segmentation

The network was segmented into five VLANs:

``` text
VLAN 10 → GAMING
VLAN 20 → STAFF
VLAN 30 → GUEST-WIFI
VLAN 40 → SERVICES
VLAN 99 → MANAGEMENT
```

Gaming PCs were placed into VLAN 10.

Example access-port configuration:

``` cisco
interface fa0/x
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
```

PortFast was used only on end-device ports.

------------------------------------------------------------------------

# 6. LACP EtherChannel and Trunking

Two EtherChannels were built from L3-SW1:

``` text
L3-SW1
 ├── Port-channel 1 → ACCESS-SW1
 └── Port-channel 2 → ACCESS-SW2
```

Both EtherChannels use **LACP**.

The bundled links were verified in Packet Tracer, with member interfaces
showing the bundled state.

The port-channels were configured as 802.1Q trunks.

The required VLANs were explicitly allowed:

``` text
10,20,30,40,99
```

This provides:

-   Link aggregation
-   Additional bandwidth
-   Redundancy
-   VLAN transport between switches

------------------------------------------------------------------------

# 7. Spanning Tree Protocol

STP root priority was deliberately designed instead of relying on
default election.

### Primary root

L3-SW1:

``` cisco
spanning-tree vlan 10,20,30,40,99 root primary
```

### Secondary root

ACCESS-SW1:

``` cisco
spanning-tree vlan 10,20,30,40,99 root secondary
```

### Result

``` text
L3-SW1       → Primary STP Root
ACCESS-SW1   → Secondary STP Root
ACCESS-SW2   → Normal/non-root switch
```

The STP configuration was verified on L3-SW1, including VLAN 10 root
status.

------------------------------------------------------------------------

# 8. Phase 2 -- Layer 3 Switching

L3-SW1 was used as the central Layer 3 device.

The VLAN interfaces were configured as SVIs.

``` cisco
interface vlan 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown

interface vlan 20
 ip address 192.168.20.1 255.255.255.0
 no shutdown

interface vlan 30
 ip address 192.168.30.1 255.255.255.0
 no shutdown

interface vlan 40
 ip address 192.168.40.1 255.255.255.0
 no shutdown

interface vlan 99
 ip address 192.168.99.1 255.255.255.0
 no shutdown
```

`ip routing` was enabled on L3-SW1 as part of the Layer 3 configuration.

### Gateway design

Each VLAN uses its SVI on L3-SW1 as the default gateway:

``` text
Gaming       → 192.168.10.1
Staff        → 192.168.20.1
Guest Wi-Fi  → 192.168.30.1
Services     → 192.168.40.1
Management   → 192.168.99.1
```

------------------------------------------------------------------------

# 9. DHCP Services

DHCP was configured directly on L3-SW1.

Reserved addresses were excluded from the beginning of each user VLAN:

``` cisco
ip dhcp excluded-address 192.168.10.1 192.168.10.20
ip dhcp excluded-address 192.168.20.1 192.168.20.20
ip dhcp excluded-address 192.168.30.1 192.168.30.20
```

### Gaming DHCP pool

``` cisco
ip dhcp pool GAMING
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
```

### Staff DHCP pool

``` cisco
ip dhcp pool STAFF
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 8.8.8.8
```

### Guest Wi-Fi DHCP pool

``` cisco
ip dhcp pool GUEST-WIFI
 network 192.168.30.0 255.255.255.0
 default-router 192.168.30.1
 dns-server 8.8.8.8
```

VLAN 40 and VLAN 99 were intentionally left without DHCP pools. Services
and management devices use static addressing. In particular, the dedicated
management workstation and the switch management interfaces use static
management addressing rather than DHCP.

The DHCP pools provide `8.8.8.8` as the DNS server address. This address
corresponds to the dedicated Packet Tracer DNS server added to the ISP-side
DNS network.

------------------------------------------------------------------------

# 10. DNS Service

A dedicated Packet Tracer DNS server was added to simulate name resolution
for the gaming environment.

### DNS server

``` text
DNS Server
IP address: 8.8.8.8/24
Network:    8.8.8.0/24
```

The DNS server is connected to ISP-R1 on a separate DNS-side network.
Client DHCP pools on VLANs 10, 20, and 30 use `8.8.8.8` as their DNS
server address.

This allows the project to demonstrate DNS-based name resolution in
addition to IP connectivity.

------------------------------------------------------------------------

# 11. Management VLAN

VLAN 99 was dedicated to network-device management.

### L3-SW1

``` text
192.168.99.1/24
```

### ACCESS-SW1

``` text
192.168.99.2/24
```

``` cisco
ip default-gateway 192.168.99.1
```

### ACCESS-SW2

``` text
192.168.99.3/24
```

``` cisco
ip default-gateway 192.168.99.1
```

Connectivity was verified by successfully pinging:

``` text
ACCESS-SW1 ↔ L3-SW1
ACCESS-SW2 ↔ L3-SW1
ACCESS-SW1 ↔ ACCESS-SW2
```

This confirmed that the management VLAN was functioning end-to-end.

------------------------------------------------------------------------

# 12. SSH Device Management

SSH was configured on all three switches.

Common configuration:

``` cisco
ip domain-name gamingcenter.local
username admin privilege 15 secret mr1234
crypto key generate rsa
```

RSA key size:

``` text
1024 bits
```

VTY configuration:

``` cisco
line vty 0 15
 login local
 transport input ssh
```

Telnet was not enabled; remote VTY access was restricted to SSH.

### Management addresses

``` text
L3-SW1       192.168.99.1
ACCESS-SW1   192.168.99.2
ACCESS-SW2   192.168.99.3
```

All configurations were saved with:

``` cisco
write memory
```

> **Note:** The credentials shown above are lab/project credentials.
> They should not be reused in a real production network.

------------------------------------------------------------------------

# 13. L3-SW1 ↔ EDGE-R1 Routed Connection

A dedicated routed point-to-point connection was created between the
core multilayer switch and the edge router.

### L3-SW1 G0/2

``` cisco
interface g0/2
 description TO-EDGE-R1
 no switchport
 ip address 10.0.0.2 255.255.255.252
 no shutdown
```

### EDGE-R1 G0/2

``` cisco
interface g0/2
 description TO-L3
 ip address 10.0.0.1 255.255.255.252
 ip nat inside
 no shutdown
```

The `/30` network provides:

``` text
10.0.0.1 → EDGE-R1
10.0.0.2 → L3-SW1
10.0.0.3 → Broadcast
```

The routed link was verified as operational.

------------------------------------------------------------------------

# 14. EDGE-R1 ↔ ISP-R1

The enterprise edge router was connected to the ISP router using:

``` text
203.0.113.0/24
```

### EDGE-R1 G0/1

``` cisco
interface g0/1
 description TO-ISP
 ip address 203.0.113.2 255.255.255.0
 ip nat outside
 no shutdown
```

### ISP-R1 G0/1

``` text
203.0.113.1/24
```

The interface mapping was corrected during implementation after
initially checking the physical interface relationships.

------------------------------------------------------------------------

# 15. Static Routing

## L3-SW1

The default route points toward EDGE-R1:

``` cisco
ip route 0.0.0.0 0.0.0.0 10.0.0.1
```

## EDGE-R1

A summary route was added for the entire internal address space:

``` cisco
ip route 192.168.0.0 255.255.0.0 10.0.0.2
```

Default route toward the ISP:

``` cisco
ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

## ISP-R1

A return route toward the enterprise internal network was configured:

``` cisco
ip route 192.168.0.0 255.255.0.0 203.0.113.2
```

### Routing logic

``` text
Internal VLANs
     ↓
L3-SW1
     ↓
10.0.0.1
     ↓
EDGE-R1
     ↓
203.0.113.1
     ↓
ISP-R1
     ↓
Public Network
```

The routing tables were checked during implementation to confirm the
connected and static routes.

------------------------------------------------------------------------

# 16. NAT/PAT on EDGE-R1

PAT was configured so internal addresses from the `192.168.0.0/16`
network can share the EDGE-R1 WAN address.

### NAT ACL

``` cisco
access-list 1 permit 192.168.0.0 0.0.255.255
```

### PAT

``` cisco
ip nat inside source list 1 interface g0/1 overload
```

### NAT interface roles

``` text
G0/2 → ip nat inside
G0/1 → ip nat outside
```

This allows multiple internal hosts to be translated through:

``` text
203.0.113.2
```

NAT statistics were checked during implementation. Initially, the
translation table remained empty because no translated traffic had been
generated yet; this is expected behavior until a matching flow occurs.

------------------------------------------------------------------------

# 17. Public Network and Public Server

The ISP router provides the public-side network:

``` text
198.51.100.0/24
```

ISP-R1:

``` text
G0/0 = 198.51.100.1/24
```

The Packet Tracer public server is connected to this public network.

The network was intentionally separated into:

``` text
PUBLIC NETWORK
198.51.100.0/24
```

and:

``` text
WAN / ISP TRANSIT
203.0.113.0/24
```

This makes the topology easier to understand and document.

------------------------------------------------------------------------

# 18. Wireless Guest Network

Two access points were configured:

``` text
AP1
AP2
```

Both use the same guest wireless configuration:

``` text
SSID:       WIFI-GUEST1
Band:       2.4 GHz
Channel:    11
Security:   WPA2-PSK
Encryption: AES
PSK:        mr123456
```

The intended wireless network is:

``` text
WIFI-GUEST1
      ↓
VLAN 30
      ↓
192.168.30.0/24
      ↓
192.168.30.1
```

The access-point uplink ports must carry VLAN 30 for wireless clients to
actually enter the guest VLAN.

------------------------------------------------------------------------

# 19. Guest Network Isolation

An extended ACL was created on L3-SW1 to prevent Guest Wi-Fi users from
accessing internal networks.

``` cisco
ip access-list extended GUEST-FILTER
 permit udp any eq bootpc any eq bootps
 deny ip 192.168.30.0 0.0.0.255 192.168.0.0 0.0.255.255
 permit ip 192.168.30.0 0.0.0.255 any
```

The ACL is applied inbound on the Guest Wi-Fi SVI:

``` cisco
interface vlan 30
 ip access-group GUEST-FILTER in
```

### Policy

``` text
DHCP                    → ALLOWED
Guest → Internal LAN    → DENIED
Guest → Other networks  → ALLOWED
```

The intended security boundary is:

``` text
Guest Wi-Fi
192.168.30.0/24
       |
       | X
       |----> 192.168.0.0/16 Internal Networks
       |
       +----> Public/Internet destinations
```

The final ACL should use the `/24` wildcard:

``` text
0.0.0.255
```

for the Guest source network.

------------------------------------------------------------------------

# 20. Layer 2 Connectivity Verification

A real troubleshooting case occurred when:

``` text
PC10 = 192.168.10.31
PC11 = 192.168.10.32
```

could temporarily not ping each other even though both belonged to the
same subnet.

Investigation confirmed that both devices were assigned to VLAN 10.

On ACCESS-SW1:

``` text
VLAN 10
Fa0/2
Fa0/3
Fa0/4
Fa0/5
Fa0/6
Fa0/7
```

PC11 was connected through Fa0/7.

Packet Tracer Simulation Mode showed the frame entering ACC-SW1 through
Fa0/7 and being flooded across the VLAN, consistent with ARP/MAC
discovery.

The connectivity later started working without a configuration change.

### Conclusion

The configuration was correct and the issue appeared to be a temporary
Packet Tracer simulation/ARP/MAC-learning behavior.

The test successfully demonstrated that:

``` text
PC10 192.168.10.31
        ↕
      VLAN 10
        ↕
PC11 192.168.10.32
```

can communicate at Layer 2.

------------------------------------------------------------------------

# 21. Simulated Gaming Web Portal

Because Packet Tracer Server-PT cannot run a real game server such as
Minecraft, Counter-Strike, Node.js, Python, or another full application
stack, the project uses a lightweight web-based simulation.

A custom gaming portal was created using:

``` text
HTML
CSS
JavaScript
```

No external libraries or dependencies are required.

### Website structure

``` text
Server-PT
└── HTTP
    ├── index.html
    └── game.html
```

### Landing page

The landing page contains:

-   Gaming Center branding
-   Cyber/gaming visual design
-   Server status
-   HTTP service status
-   Game Zone section
-   "ENTER THE ARENA" section
-   Button to launch the game
-   Future-game placeholders

The game link is:

``` html
<a class="btn" href="game.html">PLAY NEON ARENA →</a>
```

### Neon Arena

The game is a simple browser-based reaction game featuring:

-   30-second timer
-   10 targets
-   Score counter
-   Gaming Center branding
-   Responsive layout
-   Inline CSS
-   Inline JavaScript

The design was chosen because Packet Tracer's HTTP server can serve
static HTML/CSS/JavaScript, while it cannot host a normal multiplayer
game backend.

------------------------------------------------------------------------

------------------------------------------------------------------------

# 22. Security Hardening v2

After completing the core network implementation, a dedicated security
hardening phase was started. The objective is to add practical Layer 2 and
Layer 3 protections without changing the core topology.

## 22.1 Port Security

Port Security was implemented on all **12 gaming-PC access ports**:

``` text
ACCESS-SW1 → 6 PC ports
ACCESS-SW2 → 6 PC ports
```

The PC-facing ports use the following policy:

``` cisco
switchport mode access
switchport port-security
switchport port-security maximum 1
switchport port-security mac-address sticky
switchport port-security violation restrict
```

Sticky MAC learning allows the switch to learn and retain the expected
endpoint MAC address. A maximum of one secure MAC address is permitted per
PC port, and violations use the `restrict` action rather than immediately
shutting the interface.

The AP ports and EtherChannel uplinks were intentionally excluded from this
PC-focused Port Security configuration.

## 22.2 PortFast and BPDU Guard

PortFast and BPDU Guard were enabled on the 12 gaming-PC access ports:

``` cisco
spanning-tree portfast
spanning-tree bpduguard enable
```

PortFast allows end-device ports to transition quickly to forwarding, while
BPDU Guard protects these edge ports from unexpected Spanning Tree BPDUs.

## 22.3 Unused-Port Isolation

Unused access ports were placed into a dedicated unused VLAN:

``` text
VLAN 999 → UNUSED-PORTS
```

The unused ports were configured as access ports in VLAN 999 and
administratively shut down.

On ACCESS-SW1, the unused ports secured this way were:

``` text
Fa0/8-Fa0/9
Fa0/11-Fa0/14
Fa0/16-Fa0/23
```

On ACCESS-SW2, the unused ports secured this way were:

``` text
Fa0/9-Fa0/15
Fa0/17-Fa0/19
Fa0/21-Fa0/23
```

The six gaming-PC ports, AP port, and EtherChannel member ports were not
shut down. Connected trunk interfaces that were identified as EtherChannel
members were explicitly preserved.

Example configuration:

``` cisco
interface range fa0/x
 switchport mode access
 switchport access vlan 999
 shutdown
```

## 22.4 Broadcast Storm Control

Packet Tracer's switch IOS provided only broadcast storm control with a
single rising threshold. Therefore, the supported configuration was used
instead of unsupported multicast/unicast syntax:

``` cisco
storm-control broadcast level 10
```

This was applied to the 12 gaming-PC access ports. The 10% value is a
broadcast-traffic threshold; exceeding it causes excessive broadcast
traffic to be suppressed rather than shutting down the PC interface.

AP ports and EtherChannel/uplink ports were intentionally excluded.

## 22.5 Guest Network Isolation

The Guest Wi-Fi ACL was reviewed and corrected during the security phase.
The final ACL is:

``` cisco
ip access-list extended GUEST-FILTER
 permit udp any eq bootpc any eq bootps
 deny ip 192.168.30.0 0.0.0.255 192.168.0.0 0.0.255.255
 permit ip 192.168.30.0 0.0.0.255 any
```

It remains applied inbound on VLAN 30:

``` cisco
interface vlan 30
 ip access-group GUEST-FILTER in
```

The final policy is:

``` text
Guest DHCP              → ALLOWED
Guest → 192.168.0.0/16  → DENIED
Guest → external        → ALLOWED
```

The final permit was corrected to use the proper `/24` source wildcard
`0.0.0.255`.

## 22.6 Management VLAN Protection

VLAN 99 is the trusted management network:

``` text
192.168.99.0/24
```

The security policy is to prevent user-facing VLANs from initiating traffic
toward the management network while allowing the management VLAN to reach
other network segments.

The implemented source-side ACL policy is:

``` text
Gaming VLAN 10 → Management VLAN 99   DENIED
Staff VLAN 20  → Management VLAN 99   DENIED
Guest VLAN 30  → Management VLAN 99   DENIED
Management VLAN 99 → other networks  ALLOWED
```

Gaming and Staff use dedicated inbound ACLs on their SVIs to deny traffic
toward `192.168.99.0/24` while permitting other traffic. Guest-to-management
isolation is already enforced by `GUEST-FILTER`, because VLAN 99 is part of
the internal `192.168.0.0/16` range denied to Guest traffic.

This creates a management-plane boundary without preventing the trusted
management network from administering the rest of the environment.

## 22.7 DHCP Snooping Evaluation

DHCP Snooping was tested during the security hardening phase but was not
retained in the final configuration. In Packet Tracer, enabling DHCP Snooping
caused DHCP behavior problems for the management/user workstation and the
expected DHCP binding table remained empty.

The feature was therefore removed and is **not considered an implemented
security control** in the final project.

## Security Hardening Status

``` text
Port Security                  Implemented
PortFast + BPDU Guard          Implemented
Unused Port Isolation          Implemented
Broadcast Storm Control        Implemented
Guest Network Isolation        Implemented
Management VLAN Isolation      Implemented
DHCP Snooping                   Tested / Removed
SSH Hardening                   Not yet implemented
```

The security phase remains intentionally incremental: each control is
configured and verified before another one is introduced.

------------------------------------------------------------------------

# 23. Configuration Verification

Throughout the project, configuration was checked using Cisco IOS
verification commands.

Important verification commands included:

``` cisco
show vlan brief
show interfaces trunk
show etherchannel summary
show spanning-tree vlan 10
show ip interface brief
show ip route
show mac address-table
show mac address-table dynamic vlan 10
show ip nat statistics
show ip nat translations
show access-lists GUEST-FILTER
show access-lists GAMING-OUT
show access-lists STAFF-OUT
show port-security
show port-security address
show storm-control
```

Client-side tests included:

``` text
ipconfig
ping <destination>
arp -a
```

------------------------------------------------------------------------

# 24. Final Network Functions

At the current v1 stage, the network provides:

  Function                        Status
  ------------------------------- ----------------------------------
  VLAN segmentation               Implemented
  Gaming VLAN                     Implemented
  Staff VLAN                      Implemented
  Guest Wi-Fi VLAN                Implemented
  Services VLAN                   Implemented
  Management VLAN                 Implemented
  VTP                             Transparent mode
  LACP EtherChannel               Implemented
  802.1Q trunking                 Implemented
  STP root hierarchy              Implemented
  Inter-VLAN routing              Implemented
  DHCP                            Implemented for VLANs 10, 20, 30
  Management addressing           Implemented
  SSH management                  Implemented
  L3-SW1 ↔ EDGE-R1 routing        Implemented
  EDGE-R1 ↔ ISP routing           Implemented
  Static routing                  Implemented
  NAT/PAT                         Implemented
  Public network                  Implemented
  DNS service                     Implemented
  Guest isolation ACL             Implemented
  Guest wireless configuration    Implemented
  Web landing page                Implemented
  Simulated browser game          Implemented
  PC-to-PC VLAN 10 connectivity   Verified

------------------------------------------------------------------------

# 25. Design Decisions

## Why use a multilayer switch?

L3-SW1 provides:

-   Inter-VLAN routing
-   SVIs
-   DHCP
-   Centralized default gateways
-   Fast internal routing

This avoids unnecessarily sending internal VLAN traffic through the edge
router.

## Why use EtherChannel?

LACP provides:

-   Increased logical link capacity
-   Link redundancy
-   Better utilization of multiple physical links
-   A single logical port-channel for STP

## Why use STP primary/secondary roots?

The root bridge election is intentionally controlled:

``` text
Primary   → L3-SW1
Secondary → ACCESS-SW1
```

This provides predictable Layer 2 topology behavior.

## Why use VLAN 99?

Management traffic is separated from user traffic.

This makes it easier to:

-   Manage switches
-   Apply security policies
-   Identify management traffic
-   Reduce unnecessary exposure

## Why use a /16 summary internally?

The entire internal network can be summarized as:

``` text
192.168.0.0/16
```

This makes routing between the internal network and EDGE-R1 simpler.

## Why use NAT/PAT?

Private RFC1918 addresses are not directly usable as public Internet
source addresses. PAT allows many internal hosts to share the EDGE-R1
WAN address.

## Why isolate Guest Wi-Fi?

Guest users should not have direct access to internal gaming, staff,
management, or service networks.

The ACL establishes this boundary while still allowing guest traffic
toward external destinations.

------------------------------------------------------------------------

# 26. Current Architecture Summary

The final architecture can be viewed as four logical layers:

``` text
                 PUBLIC / ISP
                     |
                  ISP-R1
                     |
                EDGE-R1
             NAT / Default Route
                     |
                  L3-SW1
       Inter-VLAN Routing / DHCP
              /             \
       ACCESS-SW1          ACCESS-SW2
          |                    |
     End Devices          End Devices
          |
       AP1 / AP2
          |
      Guest VLAN
```

### Traffic examples

#### Gaming PC → another Gaming PC

``` text
PC
 ↓
VLAN 10
 ↓
Layer 2 switching
 ↓
Destination PC
```

No router is required when both hosts are in the same subnet/VLAN.

#### Gaming PC → Staff PC

``` text
Gaming VLAN 10
       ↓
192.168.10.1
       ↓
L3-SW1
       ↓
192.168.20.1
       ↓
Staff VLAN 20
```

#### Internal PC → Public Network

``` text
Internal PC
    ↓
L3-SW1
    ↓
EDGE-R1
    ↓
PAT
    ↓
203.0.113.2
    ↓
ISP-R1
    ↓
198.51.100.0/24
```

#### Guest Wi-Fi → Internal Network

``` text
Guest VLAN 30
      ↓
L3-SW1
      ↓
GUEST-FILTER
      ↓
DENIED
```

------------------------------------------------------------------------

# 27. Future Security Development

The current project is considered the **baseline/v1 implementation**.

Future versions can focus on hardening rather than rebuilding the
network.

Possible improvements include:

-   Dynamic ARP Inspection
-   IP Source Guard
-   Root Guard
-   Stronger SSH configuration
-   More granular inter-VLAN ACLs
-   Better password/credential management
-   Dedicated firewall/security appliance
-   More granular inter-VLAN ACLs
-   Guest Internet-only policy
-   Management-plane protection
-   Logging and monitoring
-   Syslog
-   NTP
-   SNMP
-   AAA/RADIUS
-   Network segmentation improvements

These should be treated as subsequent security phases rather than mixed
into the completed baseline.

------------------------------------------------------------------------

# 28. Project Status

**Gaming Center Network v1 --- Core implementation completed. Security Hardening v2 partially completed.**

The project currently demonstrates practical knowledge of:

``` text
Layer 2 Switching
VLANs
802.1Q Trunking
LACP EtherChannel
STP
Layer 3 Switching
SVIs
Inter-VLAN Routing
DHCP
Management VLAN
SSH
Layer 2 Security Hardening
Management VLAN Isolation
Static Routing
WAN Connectivity
NAT/PAT
ACLs
Wireless Networking
Public Network Design
Packet Tracer Web Services
Basic Application Simulation
Network Troubleshooting
```

The network was implemented progressively and verified during
development rather than configuring the entire topology at once.

------------------------------------------------------------------------

## Author

**Mostafa Ahmed Rashad**

Cisco Networking / Network Engineering Project

**Platform:** Cisco Packet Tracer

**Project Type:** Gaming Center Enterprise Network

**Version:** v1.0
