# Verification & Testing Guide

How to confirm each part of the network is configured and working. For every
area: the command(s) to run, and what to look for in the output. Negative tests
(where something *should* fail) are included, since blocking the wrong traffic
is as much a proof as passing the right traffic.

> Commands are run in privileged EXEC mode (`enable`). ASA syntax differs from
> IOS and is noted where relevant. In Packet Tracer, a small subset of options
> may vary by device model/version.

---

## 1. Routing with EIGRP

```
show ip eigrp neighbors
show ip route eigrp
show ip protocols
```
* `neighbors` lists an adjacency for each directly connected router - the table
  should not be empty.
* `route eigrp` shows remote subnets learned as **D** entries.
* `protocols` should confirm EIGRP AS number and that **auto-summary is
  disabled**.
* **End-to-end test:** ping a host in a remote LAN from a PC, success proves
  routing + addressing.

## 2. VLANs & Trunking

```
show vlan brief
show interfaces trunk
show interfaces status
```
* `vlan brief` - host ports in the right VLANs; unused ports parked in **VLAN
  666**.
* `interfaces trunk` - S3–S4 links listed as trunks with **native VLAN 90**.
* `interfaces status` - host ports in `access`/`connected`, switch links in
  `trunk`, unused ports `disabled`.

## 3. Spanning Tree & Storm Control

```
show spanning-tree vlan 1
show spanning-tree summary
show storm-control broadcast
```
* On **Central**: the output shows `This bridge is the root` for VLAN 1.
* On **S4**: priority is the secondary value (8192 higher than root) - root when
  Central fails.
* `summary` confirms PortFast and BPDU Guard are active on edge ports.
* `storm-control broadcast` on the Central–S3–S4 links shows the **50%** level.
* **Negative test:** plug a switch into a PortFast+BPDU-Guard port - the port
  goes `err-disabled` (verify with `show interfaces status err-disabled`).

## 4. Port Security

```
show port-security
show port-security interface fastEthernet 0/10
show port-security address
```
* S6 Fa0/10: **MaxSecureAddr = 3**, sticky learning, violation mode
  **Shutdown**.
* S7 Fa0/5 / Fa0/15: exactly one secure MAC each (PC8 / PC7).
* **Negative test:** connect an extra/unauthorized host - the port moves to
  `secure-shutdown`; recover with `shutdown` / `no shutdown`.

## 5. DHCP Snooping

```
show ip dhcp snooping
show ip dhcp snooping binding
```
* Snooping enabled, operational on **VLAN 1** only.
* Trusted interfaces (uplinks / server port) listed as `trusted`; host ports
  `untrusted` with a **rate limit of 4**.
* Binding table populates as inside hosts lease addresses.

## 6. Access Control Lists

```
show access-lists
```
* `access-lists` shows `SSH_MIKAN` and `MIKAN` with incrementing hit counts as
  traffic matches.
* `ip interface` confirms each ACL is applied in the correct direction.
* **Positive/negative tests for `SSH_MIKAN`:** SSH to Router B from an
  odd-numbered host during the window succeeds; from an even host, outside the
  window, or from the last usable host - denied.
* **Tests for `MIKAN`:** a bogon-sourced ping to the KONFERENCIJSKA_SALA/ZAPOSLENI
  LANs is dropped, while **PC6 (209.165.64.226)** can still SSH to Router A and
  ping the TACACS+ server.

## 7. Site-to-Site IPsec VPN (Router B)

```
show crypto isakmp sa
show crypto ipsec sa
show crypto map
```
* First send interesting traffic (ping from LAN B to the ZAPOSLENI LAN), then:
* `isakmp sa` shows state **QM_IDLE** (Phase 1 up).
* `ipsec sa` shows **pkts encaps / decaps** counters incrementing (Phase 2 is
  encrypting real traffic).
* **Negative check:** traffic *not* matched by the crypto ACL should travel
  unencrypted (encaps counter does not move).

## 8. Perimeter Firewall, Cisco ASA (`MIKAN-ASA`)

```
show interface ip brief
show nameif
show route
show run object
show nat
show dhcpd state
show dhcpd binding
show run ssh
show run telnet
show run aaa
```
* `nameif` - inside (100), outside (0), dmz (70) with correct VLAN mapping.
* `route` - a static default route toward the outside next hop.
* `object` / `nat` - `inside-net` (PAT) and `web-net` (static NAT) present.
* `dhcpd` - pool active, first 10 addresses excluded, DNS `209.165.210.20`,
  lease `10000`, domain `ict.com`.
* `ssh` / `telnet` - inside allowed; SSH also from PC6; correct idle timeouts.
* **Flow test (ASA's built-in simulator):**
  ```
  packet-tracer input inside tcp 172.16.192.20 1025 <outside-host> 80
  ```
  Should end in **ALLOW** for inside→outside, and **DROP** for a DMZ→inside
  flow (the blocked direction).
* **Live checks:** an inside host gets an address via DHCP; the DMZ web server is
  reachable from outside via its static-NAT address; DMZ cannot initiate to
  inside.

## 9. Centralized AAA, TACACS+ (Router B)

```
show run | include aaa|tacacs
show tacacs
```
* Confirms `aaa authentication login default group tacacs+ local`.
* **Fallback test:** with the TACACS+ server reachable, log in with a
  TACACS+ account; then power off / make the server unreachable and confirm
  login still works with the **local** user `mikan2`, this proves the
  `... group tacacs+ local` fallback. `debug aaa authentication` shows which
  method answered.

## 10. Zone-Based Firewall (Router A)

```
show zone security
```
* `zone security` - three zones (ZAPOSLENI, KONFERENCIJSKA_SALA, INTERNET) with the
  right member interfaces.
* `zone-pair` / `policy-map` - inspection policies applied between zones; the
  counters confirm traffic is being matched and inspected.

## 11. Secure Management, SSH & Role-Based CLI

```
show ip ssh
show ssh
show run | section line vty
show parser view
```
* `ip ssh` on S1 - **version 2**, authentication timeout / retries, RSA modulus
  **1024**.
* `line vty` - `transport input ssh` only (Telnet refused).
* **Role test:** `enable view Tech_MIXI`, then confirm only `show protocols`,
  `show ip protocols` and `show ip route` are permitted, other commands are
  rejected.

---
