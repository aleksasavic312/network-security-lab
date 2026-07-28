# Implementation Overview

An enterprise network built and secured in Cisco Packet Tracer, spanning the
access, distribution and perimeter layers. Below is a breakdown of what I
implemented, grouped by area.

## Base configuration & addressing

- Assigned hostnames to every router and switch, and configured the
  FastEthernet and serial interfaces used across the topology.
- Set the clock rate to 64 kb/s on the DCE side of each serial link.
- Applied IP addressing, subnet masks and default gateways to all end hosts
  per the addressing plan in the topology.

## Routing with EIGRP

- Enabled EIGRP on every router and disabled automatic route summarization so
  that discontiguous subnets are advertised with their correct masks.

## Device access & management security

- Configured a privilege-15 local user on **Router A** with an MD5-hashed
  password.
- Hardened the VTY lines on **switch S1** to accept SSH only:
  generated a 1024-bit RSA key pair under a defined domain name, enabled
  **SSH version 2**, restricted transport to SSH, set an idle timeout and
  capped authentication retries.
- On **switch S2**, assigned a management IP to the SVI of the management VLAN
  (**VLAN 99**) using the last usable host address of its LAN and set the
  default gateway.
- Built a **role-based CLI view** (`Tech_MIXI`) on Router A that exposes only
  `show protocols`, `show ip protocols` and `show ip route`, so field
  technicians get read-only visibility without full access. Protected with an
  MD5 password.

## Layer 2 - VLANs, trunking & port roles

- Configured the GigabitEthernet links between **S3** and **S4** as trunks and
  moved the native VLAN off the default to **VLAN 90** on those trunks.
- On **switch S5**, set host-facing ports to access mode and inter-switch links
  to trunk mode; administratively shut down all remaining unused ports and
  parked them in a black-hole VLAN (**VLAN 666**).

## Port security

- **Dynamic (sticky)** port security on **S6** (Fa0/10): learns up to 3 MAC
  addresses automatically, with a `shutdown` violation action on any
  additional/invalid MAC.
- **Static** port security on **S7**: Fa0/5 is pinned to PC8's MAC and Fa0/15
  to PC7's MAC, both with a `shutdown` violation action.

## Spanning Tree & edge protection

- Set **Central** as the primary and **S4** as the secondary root bridge for
  VLAN 1, giving a deterministic, redundant STP topology.
- Enabled **PortFast** and **BPDU Guard** on the host/server-facing access
  ports of S5 and S6 to speed up edge convergence and shut a port if a rogue
  switch appears.

## Layer 2 hardening

- Applied **broadcast storm control** at a 50% threshold on the interconnects
  between Central, S3 and S4.
- Enabled **DHCP snooping** globally on S5, scoped to VLAN 1, and rate-limited
  DHCP requests to 4 per second on the host ports.
- Marked the switch-to-switch links and the uplink toward the DHCP server as
  **trusted** DHCP-snooping ports (configured on S6), leaving access ports
  untrusted.

## Access control lists

- **Standard named ACL (`SSH_MIKAN`)** - permits SSH to Router B only from
  odd-numbered hosts in the lower usable range of Router B's LAN, the last usable host of that range is
  denied and everything else is denied.
- **Extended named ACL (`MIKAN`)** - blocks SSH to Router A and ICMP to the
  CONFERENCE-ROOM and EMPLOYEE LANs from spoofed/bogon sources (RFC 1918
  class A/B/C, multicast and 127.0.0.0/8), while whitelisting host PC6
  (public 209.165.64.226) for SSH to Router A and ping to the TACACS+ server.
  Inbound EIGRP updates toward those two LANs are dropped; all other legitimate
  traffic is permitted.

## Site-to-Site IPsec VPN (Router B)

Protects traffic between LAN B and the EMPLOYEE LAN, scoped by a crypto ACL so
only that traffic is encrypted.

- **IKE Phase 1 (ISAKMP):** AES-256 encryption, SHA-1 hashing, pre-shared-key
  authentication, DH group 5, 36000-second SA lifetime.
- **IKE Phase 2 (IPsec):** a transform set (`VPN-SET`) using `esp-aes` +
  `esp-sha-hmac`, bound through a crypto map (`VPN-MAP`) to the peer.

## Perimeter firewall, Cisco ASA (`MIKAN-ASA`)

- Configured three logical interfaces with security levels: **inside (100)**,
  **outside (0)** and **DMZ (70)**, and mapped the physical ports to the right
  VLANs. Blocked traffic initiated from the DMZ toward the inside.
- Added a static default route so the internal network can reach the Internet.
- **PAT** for the inside network via an `inside-net` object; **static NAT** for
  the DMZ web server via a `web-net` object mapped to an outside address.
- **Remote management:** Telnet from the inside (10-min idle timeout) and SSH
  from the inside plus host PC6 (15-min idle timeout), with a 1024-bit RSA key
  and a configured domain name.
- **AAA:** a local user (`mixiadmin`) with the ASA's local database used to
  authenticate SSH sessions.
- **DHCP server** on the inside interface: a 32-address pool that skips the
  first 10 addresses of the subnet, hands out a DNS server, a 10000-second
  lease and a domain name.

## AAA / TACACS+ (Router B)

- Configured a local fallback user and pointed AAA authentication at a TACACS+
  server (with a shared key); if the server is unreachable, authentication
  falls back to the local database via the default method list.

## Zone-Based Firewall (Router A)

- Defined three security zones: **EMPLOYEE** (PC2 + TACACS+ server),
  **CONFERENCE-ROOM** (PC1) and **INTERNET** and assigned each interface to
  its corresponding zone.

---

*Cisco IOS · Cisco ASA · EIGRP · STP · Port Security · DHCP Snooping · IPsec ·
TACACS+ · AAA · NAT/PAT · Zone-Based Firewall*
