# Equity-Bank-Layer2-Hardening

# Project 9: Layer 2 Infrastructure Hardening & Multi-Vector Attack Mitigation
**Company:** Equity Bank Kenya — Upperhill Branch  
**Objective:** Remediate 4 Critical Infrastructure Vulnerabilities Within a 48-Hour Compliance Window  
**Status:** Production Hardened & Audit Verified  
**Target Audience:** Infrastructure Security Teams / Network Auditors  

---

## 1. Executive Summary & Lab Mandate
Following a comprehensive internal security audit at the Equity Bank Upperhill Branch, four critical Layer 2 vulnerabilities were identified within the local access network fabric. Left unmitigated, these vulnerabilities allow malicious actors to perform CAM table overflows, intercept sensitive financial transactions via Man-in-the-Middle (MitM) positioning, bypass VLAN segregation controls, and gain unauthorized physical access to the network domain.

This deployment project details the systematic simulation, remediation, and verification of these four attack vectors. All configurations have been hardened on a production baseline using Cisco IOS parameters, completely satisfying the compliance checklist within the mandated 48-hour deadline.

---

## 2. Hardened Branch Topology Map
The secure access architecture handles client traffic distribution using a restricted, deterministic port allocation strategy:

```text
                     [CORE-ROUTER] (Default Gateway)
                       (Fa0/0) | (Trusted Uplink Path)
                               |
                        [BRANCH-SW] (Cisco 2960 Access Switch)
                               |
       ┌───────────────────────┼───────────────────────┐
       |                       |                       |
   (Fa0/1)                  (Fa0/5)                 (Fa0/10 - Fa0/24)
  [ATTACK-PC]             [TELLER-PC]             [ BLACKHOLE ZONE ]
 (Port Security:          (Production             (VLAN 999 Isolation /
  Max 1 / Sticky)          VLAN 10)                Administrative Shutdown)

```
 VLAN Structural Design Matrix
- VLAN 10 (DATA-TELLERS): Dedicated segment isolating legitimate bank operations and host workstation traffic.

- VLAN 999 (BLACKHOLE): Unrouted dead-end isolation zone used to neutralize unassigned physical switch ports.\

  
---
## 3. Multi-Phase Technical Implementation & CLI Logs
# Attack 1: MAC Flooding (CAM Table Overflow)
- Threat Profile: An intruder utilizes tools like macof on interface Fa0/1 to flood the switch with thousands of randomized source MAC addresses. This exhausts the Content Addressable Memory (CAM) table, forcing the switch to fail-open and broadcast all data frames to every port, turning the switch into a dumb hub and allowing traffic sniffing.

- Remediation Script: Hardened the interface configuration to restrict address limits and enforce port shutdown penalties.
```bash
BRANCH-SW# configure terminal
BRANCH-SW(config)# interface Fa0/1
BRANCH-SW(config-if)# switchport mode access
BRANCH-SW(config-if)# switchport port-security
BRANCH-SW(config-if)# switchport port-security maximum 1
BRANCH-SW(config-if)# switchport port-security mac-address sticky
BRANCH-SW(config-if)# switchport port-security violation shutdown
```
- Metrics Decoded: maximum 1 limits frame ingest to a single physical device. sticky dynamically binds the legitimate host MAC address to the running configuration. violation shutdown places the port into an immediate err-disabled state if an unmapped MAC is intercepted.
---
# Operational Validation & Compliance Audit Evidence
```bash
BRANCH-SW# show port-security interface Fa0/1
Port Security              : Enabled
Port Status                : Secure-up
Violation Mode             : Shutdown
Maximum MAC Addresses      : 1
Total MAC Addresses        : 1
Configured MAC Addresses   : 0
Sticky MAC Addresses       : 1
Last Source Address:Vlan   : 00e0.b911.a214:10
Security Violation Count   : 0
```
*Audit Status: COMPLIANT ✅ (Port security engine actively monitoring; violation vector locked to shutdown).*
---

## Attack 2: VLAN Hopping (DTP Exploitation)
- Threat Profile: Switch ports operating in default dynamic negotiation states communicate via Dynamic Trunking Protocol (DTP). An attacker can issue malicious trunking negotiation frames to transition their standard access port into an 802.1Q Trunk link, bypassing inter-VLAN routing controls to hop into restricted banking subnets.

- Remediation Script: Stripped all dynamic capabilities and explicitly locked client links down to the access layer.
```bash
BRANCH-SW# configure terminal
BRANCH-SW(config)# interface range Fa0/1 - 20
BRANCH-SW(config-if-range)# switchport mode access
BRANCH-SW(config-if-range)# switchport nonegotiate
```
# Operational Validation & Compliance Audit Evidence
```bash
BRANCH-SW# show interfaces trunk

BRANCH-SW#
```
*Audit Status: COMPLIANT ✅ (The completely blank console output confirms that zero user-facing access interfaces are participating in DTP or operating as active trunking links).*
---
## Attack 3: Rogue DHCP Server Injections (Man-in-the-Middle)
- Threat Profile: A malicious laptop deployed on an access port launches a rogue DHCP server. By responding to DHCP Discover broadcasts faster than the corporate gateway, the attacker forces legitimate bank hosts to accept lease configurations where the Default Gateway points to the attacker's node, enabling complete data interception.

- Remediation Script: Configured global DHCP Snooping to treat client links as untrusted while establishing a strict trust anchor on the core backbone link.
```bash
BRANCH-SW# configure terminal
BRANCH-SW(config)# ip dhcp snooping
BRANCH-SW(config)# ip dhcp snooping vlan 10
BRANCH-SW(config-if)# interface Gi0/1
BRANCH-SW(config-if)# ip dhcp snooping trust
```
# Operational Validation & Compliance Audit Evidence
```bash
BRANCH-SW# show ip dhcp snooping
Switch DHCP snooping is enabled
DHCP snooping is configured on vlan 10
Insertion of option 82 is enabled

Interface                  Trusted       Allow trust list
-----------------------    -------       ----------------
GigabitEthernet0/1         Yes           VLAN 10
FastEthernet0/1            No            VLAN 10
FastEthernet0/5            No            VLAN 10
FastEthernet0/10           No            VLAN 10
```
*Audit Status: COMPLIANT ✅ (Only the primary core link Gi0/1 possesses trusted privileges. Rogue DHCP server messages hitting untrusted FastEthernet interfaces are automatically dropped by the kernel block).*
---
## Attack 4: Unassigned Physical Port Exploitation
- Threat Profile: Exposed physical wall jacks connected to active, unconfigured switch interfaces present an open access vector. An unauthorized intruder can simply connect to an empty desk jack and instantly infiltrate internal bank segments.

- Remediation Script: Consolidated all unassigned access interfaces (Fa0/10 through Fa0/24), moved them to a non-routable dead-end VLAN, and disabled their interface radios.
```bash
BRANCH-SW# configure terminal
BRANCH-SW(config)# interface range Fa0/10 - 24
BRANCH-SW(config-if-range)# description UNUSED-DISABLED
BRANCH-SW(config-if-range)# switchport access vlan 999
BRANCH-SW(config-if-range)# shutdown
```

# Operational Validation & Compliance Audit Evidence
```bash
BRANCH-SW# show interfaces status
Port      Name               Status       Vlan       Duplex  Speed Type
Fa0/1     ATTACK-PC          connected    10         full    100Mbps 10/100BaseTX
Fa0/5     TELLER-PC          connected    10         full    100Mbps 10/100BaseTX
Fa0/10    UNUSED-DISABLED    disabled     999        auto    auto    10/100BaseTX
Fa0/11    UNUSED-DISABLED    disabled     999        auto    auto    10/100BaseTX
Fa0/24    UNUSED-DISABLED    disabled     999        auto    auto    10/100BaseTX
Gi0/1     CORE-ROUTER-UPLINK connected    10         full    1000Mbps 1000BaseTX
```
*Audit Status: COMPLIANT ✅ (All unused interfaces are verified as administratively disabled and completely quarantined inside the Blackhole VLAN 999 block).*
----

## 4. Final Security Posture Conclusions
By successfully deploying layer-2 defense-in-depth mitigations, Equity Bank's Upperhill Branch network infrastructure is thoroughly secured against access-layer attack types. The local network switching control plane is completely isolated, preventing external manipulations and fulfilling all compliance audit requirements.


