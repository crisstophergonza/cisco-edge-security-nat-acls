# Secure Enterprise Edge Network: NAT/PAT and Extended ACL Deployment

## 📌 Project Overview
This project focuses on transitioning an internal corporate network infrastructure into a secure, production-ready edge environment. By implementing **Network Address Translation (NAT/PAT)** and **Extended Access Control Lists (ACLs)** natively on a Cisco ISR 4331 router, this deployment secures the corporate perimeter. It achieves two critical enterprise requirements: hiding internal network schemas from the public internet and enforcing strict inter-departmental security boundaries.

### Key Technical Achievements
* **Network Address Translation (NAT/PAT):** Engineered Port Address Translation (PAT) to overload a single public WAN interface, mapping internal private Class C networks to a simulated public internet space.
* **Perimeter Firewall Enforcements:** Authored and applied an Extended Access Control List (ACL) to intercept traffic at the sub-interface level, blocking specific inter-VLAN movement while safeguarding internet access.
* **Proactive Boundary Diagnostics:** Tested, debugged, and resolved live syntax configuration errors relating to Cisco IOS interface access-group binding mechanics.

---

## 🗺️ Edge Architecture Topology
The architecture simulates a corporate boundary where multi-floor local traffic aggregates at the gateway before routing securely to an external public web infrastructure:

![Edge Security Topology]([Insert your new Packet Tracer screenshot link here])

### Security & Addressing Parameters
The edge translation and traffic parameters are strictly defined as follows:

| Source Subnet | VLAN | Security Policy | NAT Status | Allowed Destination |
| :--- | :---: | :--- | :--- | :--- |
| **VLAN 10 (Reception)** | 10 | **DENY** traffic to VLAN 20 | Inside (Translated) | Public Internet (`8.8.8.8`) |
| **VLAN 20 (Admin/Finance)** | 20 | **PERMIT** traffic to all subnets | Inside (Translated) | Anywhere |
| **WAN Interface** | N/A | Gateway Public Edge Boundary | Outside | Public Internet |

---

## 🛠️ Configuration Deep-Dive

### 1. Port Address Translation (PAT) Configuration
To allow private internal subnets to communicate across the public internet boundary, an IP NAT source overload list was bound to the exterior interface:

```router-cli
! Define traffic eligible for translation
access-list 1 permit 192.168.10.0 0.0.0.255
access-list 1 permit 192.168.20.0 0.0.0.255
!
! Bind translation to public interface
ip nat inside source list 1 interface gigabitEthernet 0/0/1 overload
!
! Establish NAT boundaries on interfaces
interface gigabitEthernet 0/0/1
 ip nat outside
!
interface gigabitEthernet 0/0/0.10
 ip nat inside
!
interface gigabitEthernet 0/0/0.20
 ip nat inside

2. Named Extended Access Control List (Firewall Policy)
A granular Extended ACL was implemented to inspect packet headers. The rule drops traffic traveling from Floor 1 to Floor 2 right at the inbound gateway sub-interface, while keeping the web wide open:
ip access-list extended RECEPTION_SECURITY_POLICY
 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
 permit ip 192.168.10.0 0.0.0.255 any
!
interface gigabitEthernet 0/0/0.10
 ip access-group RECEPTION_SECURITY_POLICY in

Troubleshooting Log: The Dash Mismatch Syntax Error
Incident Description
During initial staging, endpoints on VLAN 10 could still successfully ping resources on VLAN 20, indicating a complete failure of the perimeter security policy.

Diagnostic Walkthrough & Resolution
1- Executed a verification review of the running configuration. Discovered that the router had accepted a malformed interface entry (ip access group in) without actively mapping the target string variable RECEPTION_SECURITY_POLICY.
2- Diagnosed a critical Cisco IOS syntax error: the dash delimiter was omitted from the command string (ip access group instead of ip access-group), which caused the system to drop the named policy map.
3- Remediated the interface configuration by clearing the empty group and re-applying the strict named policy with exact dash notation:
Core-Router(config)# interface gigabitEthernet 0/0/0.10
Core-Router(config-subif)# no ip access-group in
Core-Router(config-subif)# ip access-group RECEPTION_SECURITY_POLICY in

Architecture Validation and Verification:
To validate the security posture, verification testing was run from an endpoint on the isolated Reception floor (VLAN 10):

1- Inter-Subnet Verification (Blocked): Pinging an Admin PC (192.168.20.11) immediately results in Destination host unreachable, validating that the Extended ACL is inspecting and dropping unauthorized internal frames.
2- Perimeter Exit Verification (Success): Requesting the simulated external Web Server page via HTTP or pinging 8.8.8.8 returns a 100% success rate, validating that outbound NAT translation and the default permit statement are tracking properly.
