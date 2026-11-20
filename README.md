# Office Network Lab: VLANs, Services, Routing, and Logging

A complete small-office network built in Cisco Packet Tracer. The lab connects two departments (Sales and IT) through VLANs, provides DHCP, DNS, HTTP, and TFTP services from one server, sends logs to a central syslog address, and routes traffic to a simulated ISP and the internet.

**Author:** [Your Name]
**Tool:** Cisco Packet Tracer (logical and physical views)
**Level:** CCNA intermediate learners

---

## Table of Contents

1. [Project Goals](#1-project-goals)
2. [Topology](#2-topology)
3. [Device Inventory](#3-device-inventory)
4. [IP Addressing Plan](#4-ip-addressing-plan)
5. [Concepts Used](#5-concepts-used)
6. [Build Steps](#6-build-steps)
7. [Verification and Testing](#7-verification-and-testing)
8. [Troubleshooting](#8-troubleshooting)
9. [Command Reference](#9-command-reference)
10. [Lessons Learned](#10-lessons-learned)
11. [Possible Improvements](#11-possible-improvements)
12. [Repository Contents](#12-repository-contents)

---

## 1. Project Goals

The lab was designed to practise the following skills in one connected network:

| Goal | What it means in this lab |
|---|---|
| LAN design | Two departments, each in its own VLAN, on one switch |
| Trunking | One cable carries both VLANs from the switch to the router |
| Inter-VLAN routing | The router moves traffic between Sales and IT |
| DHCP | PCs receive IP settings automatically |
| Servers | One server provides DNS, HTTP, and TFTP |
| Logging | Router and switch events are recorded and sent to a syslog address |
| WAN | A serial link connects the office to an ISP |
| Routing | Default and static routes, plus NAT, let the office reach the internet |
| Backup | The router configuration is saved to the TFTP server |

---

## 2. Topology

### 2.1 Physical View

```
                 [ ISP Router ]  Loopback 8.8.8.8 (simulated internet)
                       |  Serial link, 10.0.0.0/30
                       |
                 [ R1-Office ]
                  |             |
     Gi0/0 (trunk)|             | Gi0/1 (server LAN)
                  |             |
          [ SW1-Office ]   [ TFTP-Server ]
           |          |     192.168.30.10
   Fa0/1 - Fa0/12  Fa0/13 - Fa0/24
           |          |
   [ VLAN 10 Sales ]  [ VLAN 20 IT ]
     10 PCs            10 PCs
```

### 2.2 Logical View

```
 VLAN 10 Sales  192.168.10.0/27          VLAN 20 IT  192.168.20.0/27
        |                                         |
        +--------- Gi0/0 on R1 (trunk) -----------+
                          |
                 R1 sub-interfaces:
                 Gi0/0.10 = 192.168.10.1
                 Gi0/0.20 = 192.168.20.1
                          |
                 Gi0/1 = 192.168.30.1  ----  Server 192.168.30.10
                          |                 (DNS, HTTP, TFTP)
                 Serial0/0/0 = 10.0.0.1
                          |
                 ISP Serial0/0/0 = 10.0.0.2
                          |
                 ISP Loopback0 = 8.8.8.8
```

### 2.3 How a Request Travels

Example: a Sales PC opens `http://www.office.lab`.

1. The PC asks the DNS server (192.168.30.10, given by DHCP) for `www.office.lab`.
2. The DNS server answers `192.168.30.10`.
3. The PC sends the HTTP request to its gateway, 192.168.10.1, tagged for VLAN 10 on the trunk.
4. R1 receives it on sub-interface Gi0/0.10, routes it out Gi0/1, and the server replies through R1 the same way.

For an internet destination such as 8.8.8.8, R1 has no specific route, so it uses the default route to the ISP and translates the private source address with NAT.

---

## 3. Device Inventory

| Device | Name | Role | Key interfaces |
|---|---|---|---|
| Router | R1-Office | Office gateway: VLANs, DHCP, NAT, routing | Gi0/0 (trunk), Gi0/1 (server), Se0/0/0 (WAN) |
| Router | ISP | Simulated internet provider | Se0/0/0 (WAN), Lo0 (8.8.8.8) |
| Switch | SW1-Office | Connects 20 PCs in two VLANs | Fa0/1-12 (VLAN 10), Fa0/13-24 (VLAN 20), Gi0/1 (trunk to R1) |
| Server | TFTP-Server | DNS, HTTP, TFTP | Static IP 192.168.30.10 |
| PC | Sales PC ×10 | Sales department | DHCP, VLAN 10 |
| PC | IT PC ×10 | IT department | DHCP, VLAN 20 |

**Hardware note:** Routers need a serial port for the WAN link. On a 2911 router, add the HWIC-2T module while the router is powered off, then power it on again. Port names may differ in your file. Always use the names your device shows.

---

## 4. IP Addressing Plan

| Network | Subnet | Mask | Wildcard | Gateway | Used by |
|---|---|---|---|---|---|
| VLAN 10 Sales | 192.168.10.0/27 | 255.255.255.224 | 0.0.0.31 | 192.168.10.1 | 10 Sales PCs (DHCP) |
| VLAN 20 IT | 192.168.20.0/27 | 255.255.255.224 | 0.0.0.31 | 192.168.20.1 | 10 IT PCs (DHCP) |
| Server LAN | 192.168.30.0/28 | 255.255.255.240 | 0.0.0.15 | 192.168.30.1 | TFTP-Server (static 192.168.30.10) |
| WAN link | 10.0.0.0/30 | 255.255.255.252 | 0.0.0.3 | R1 = 10.0.0.1, ISP = 10.0.0.2 | Serial link |
| Internet | 8.8.8.8/32 | 255.255.255.255 | 0.0.0.0 | n/a | ISP loopback |

**Why these sizes?**
- A /27 has 32 addresses, 30 usable. That is enough for 10 PCs plus the gateway, with room to grow.
- A /28 has 16 addresses, 14 usable. That is enough for one server and the gateway.
- A /30 has 4 addresses, 2 usable. That is exactly right for a point-to-point link.

---

## 5. Concepts Used

### 5.1 VLAN (Virtual LAN)

A VLAN splits one physical switch into separate logical switches. Devices in VLAN 10 cannot reach VLAN 20 at Layer 2. Each VLAN is its own broadcast domain, so a broadcast from Sales does not reach IT.

### 5.2 Access Port and Trunk Port

| Port type | Connects to | Carries | Command |
|---|---|---|---|
| Access | One PC | One VLAN, untagged | `switchport mode access` |
| Trunk | Router or another switch | Many VLANs, tagged | `switchport mode trunk` |

Set the mode explicitly. This stops ports from negotiating on their own.

### 5.3 802.1Q Tagging and Native VLAN

When a frame crosses a trunk, the switch adds a 4-byte tag that names the VLAN (IEEE 802.1Q). The receiving device reads the tag. The **native VLAN** (default VLAN 1) is the only VLAN sent untagged on a trunk. In a real network, use an unused VLAN for it instead.

### 5.4 Router-on-a-Stick (Inter-VLAN Routing)

One physical router port carries several VLANs. The router creates a **sub-interface** for each VLAN, using `encapsulation dot1Q <vlan-id>`. The IP address on each sub-interface becomes the gateway for that VLAN. Packets between VLANs must pass through the router, which is where security rules can be applied.

### 5.5 DHCP and DORA

DHCP gives PCs their IP address, mask, gateway, and DNS server automatically. The exchange has four messages:

| Step | Message | Direction | Meaning |
|---|---|---|---|
| D | Discover | PC to all | "Is there a DHCP server?" |
| O | Offer | Server to PC | "Here is an address you can use." |
| R | Request | PC to server | "I accept that address." |
| A | Acknowledge | Server to PC | "Confirmed." |

### 5.6 DNS, HTTP, and TFTP

| Service | Transport and port | Purpose | Example in this lab |
|---|---|---|---|
| DNS | UDP 53 | Converts names to IP addresses | `www.office.lab` → 192.168.30.10 |
| HTTP | TCP 80 | Serves web pages | Browser opens the intranet page |
| TFTP | UDP 69 | Simple file transfer, no login | Router config backup |

TFTP has no authentication and no encryption, so it should be used only on internal networks.

### 5.7 Static Routing and Default Route

A static route is typed by an administrator:

```
ip route <destination network> <mask> <next-hop address>
```

A **default route** (`0.0.0.0 0.0.0.0`) matches any destination that has no more specific route. R1 uses it to send internet traffic to the ISP. The ISP has no default route to your network, so it needs three static routes back to your private subnets.

### 5.8 NAT and PAT

Private addresses (192.168.x.x) are not routable on the internet. NAT replaces the source address of outgoing packets.

| Term | Meaning in this lab |
|---|---|
| Inside | Interfaces facing the office (Gi0/0.10, Gi0/0.20, Gi0/1) |
| Outside | Interface facing the ISP (Se0/0/0) |
| Overload (PAT) | Many private addresses share the single address 10.0.0.1, distinguished by port numbers |

### 5.9 Logging and Severity Levels

Devices create log messages about events such as interface changes, logins, and errors. Each message has a severity level from 0 (most severe) to 7 (least severe).

| Level | Name | Example |
|---|---|---|
| 0 | Emergencies | System unusable |
| 1 | Alerts | Immediate action needed |
| 2 | Critical | Critical condition |
| 3 | Errors | An error occurred |
| 4 | Warnings | A warning condition |
| 5 | Notifications | Interface changed state |
| 6 | Informational | Normal events, such as configuration changes |
| 7 | Debugging | Very detailed troubleshooting output |

`logging trap informational` sends levels 0 through 6 to the syslog host. Timestamps (`service timestamps log datetime msec`) make the order of events clear.

### 5.10 Serial Link, DCE and DTE

One end of a serial cable is **DCE**, which supplies the clock signal. The other end is **DTE**, which follows the clock. Only the DCE end needs `clock rate`. Check which end you have with:

```
show controllers serial 0/0/0
```

### 5.11 Spanning Tree PortFast

Spanning Tree normally takes about 30 seconds before a port forwards traffic. `spanning-tree portfast` skips this wait. Use it only on ports that connect to end devices (PCs and servers), never on switch-to-switch links.

---

## 6. Build Steps

Build in this order and test after each step. If a test fails, fix it before moving on.

### Step 1: LAN (SW1-Office)

Create the VLANs, assign the access ports, and configure the trunk to R1.

```
enable
configure terminal
hostname SW1-Office
no ip domain-lookup

vlan 10
 name SALES
vlan 20
 name IT
exit

interface range fastEthernet 0/1 - 12
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
exit

interface range fastEthernet 0/13 - 24
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 no shutdown
exit

interface gigabitEthernet 0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 no shutdown
exit

end
write memory
```

| Command | Explanation |
|---|---|
| `vlan 10` and `name SALES` | Creates VLAN 10 with a readable name |
| `interface range fastEthernet 0/1 - 12` | Configures many ports at once. Spaces around the dash are required. |
| `switchport mode access` | The port connects one PC |
| `switchport access vlan 10` | Places the port in VLAN 10 |
| `spanning-tree portfast` | Lets the PC port forward immediately |
| `switchport mode trunk` | The port carries several VLANs |
| `switchport trunk allowed vlan 10,20` | Only these VLANs may cross the trunk |

**Verify:**

```
show vlan brief
show interfaces trunk
show interfaces status
```

Expected: VLAN 10 holds Fa0/1 to Fa0/12, VLAN 20 holds Fa0/13 to Fa0/24, and Gi0/1 is trunking with VLANs 10 and 20 allowed.

### Step 2: Office Router (R1-Office) Interfaces

Set the basic security and create the sub-interfaces.

```
enable
configure terminal
hostname R1-Office
no ip domain-lookup

enable secret Admin@123
service password-encryption
username admin privilege 15 secret Admin@456
ip domain-name office.lab
banner motd #Authorized access only#

interface gigabitEthernet 0/0
 no shutdown
exit

interface gigabitEthernet 0/0.10
 description SALES-VLAN10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.224
exit

interface gigabitEthernet 0/0.20
 description IT-VLAN20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.224
exit

interface gigabitEthernet 0/1
 description SERVER-LAN
 ip address 192.168.30.1 255.255.255.240
 no shutdown
exit

interface serial 0/0/0
 description TO-ISP
 ip address 10.0.0.1 255.255.255.252
 no shutdown
exit

end
write memory
```

| Command | Explanation |
|---|---|
| `enable secret` | Protects privileged mode with a hashed password |
| `service password-encryption` | Hides plain-text passwords in the config. This is weak protection, but it stops casual reading. |
| `username admin privilege 15 secret` | Local account with full rights, used for remote login |
| `interface gi0/0` with no IP | The physical port only carries the sub-interfaces |
| `encapsulation dot1Q 10` | This sub-interface handles VLAN 10 frames |
| `description` | A note for people. It does not change behaviour. |

**Verify:**

```
show ip interface brief
show interfaces trunk
```

Expected: Gi0/0, Gi0/0.10, Gi0/0.20, and Gi0/1 show `up / up`. Se0/0/0 shows `up / down` until the ISP is configured.

### Step 3: DHCP on R1

```
configure terminal

ip dhcp excluded-address 192.168.10.1
ip dhcp excluded-address 192.168.20.1

ip dhcp pool SALES
 network 192.168.10.0 255.255.255.224
 default-router 192.168.10.1
 dns-server 192.168.30.10
 domain-name office.lab
exit

ip dhcp pool IT
 network 192.168.20.0 255.255.255.224
 default-router 192.168.20.1
 dns-server 192.168.30.10
 domain-name office.lab
exit

end
write memory
```

| Command | Explanation |
|---|---|
| `ip dhcp excluded-address` | Reserves gateway addresses so DHCP never assigns them |
| `ip dhcp pool SALES` | Creates a named address pool for Sales |
| `network` | The range the pool hands out |
| `default-router` | Gateway given to PCs |
| `dns-server` | DNS server given to PCs (the server built in Step 4) |
| `domain-name office.lab` | Suffix added to short names, so `www` becomes `www.office.lab` |

**PC setup:** On each PC, open **Desktop**, then **IP Configuration**, and select **DHCP**.

**Verify:**

```
show ip dhcp binding
show ip dhcp pool
```

On a PC, run `ipconfig`. A Sales PC should show 192.168.10.x, mask 255.255.255.224, gateway 192.168.10.1, and DNS 192.168.30.10.

### Step 4: Server (DNS, HTTP, TFTP)

**IP configuration.** The server needs a fixed address, so set it manually in **Desktop**, then **IP Configuration**:

| Field | Value |
|---|---|
| IP address | 192.168.30.10 |
| Subnet mask | 255.255.255.240 |
| Default gateway | 192.168.30.1 |
| DNS server | 192.168.30.10 |

**DNS.** In **Services**, then **DNS**:
1. Set DNS Service to **On**.
2. Name: `www.office.lab`, Address: `192.168.30.10`.
3. Click **Add**.

**HTTP.** In **Services**, then **HTTP**:
1. Set HTTP to **On**.
2. Optional: edit `index.html` to show a welcome message, then click **Save**.

**TFTP.** In **Services**, then **TFTP**:
1. Set TFTP to **On**.
2. Keep the default settings. The router backup will be stored here.

**Verify from a Sales PC:**

```
ping 192.168.30.10
ping www.office.lab
```

Then open the Web Browser on the PC and go to `http://www.office.lab`.

### Step 5: Logging

On R1:

```
configure terminal
service timestamps log datetime msec
logging buffered 8192 informational
logging host 192.168.30.10
logging trap informational
logging on
end
write memory
```

Set the clock first so timestamps are correct:

```
clock set 10:00:00 18 November 2026
```

On SW1-Office:

```
configure terminal
service timestamps log datetime msec
logging host 192.168.30.10
logging trap informational
logging on
end
write memory
```

| Command | Explanation |
|---|---|
| `service timestamps log datetime msec` | Adds the date and time to each message |
| `logging buffered 8192 informational` | Stores up to 8192 bytes of messages in router memory |
| `logging host` | The syslog server address |
| `logging trap informational` | Sends levels 0 to 6 to the syslog host |
| `logging on` | Enables logging |

**Note on the switch:** SW1 has no management IP in this lab, so its messages may not reach the server. Use `show logging` on the switch to view its log locally.

**Test:**

```
R1-Office(config)# interface gigabitEthernet 0/1
R1-Office(config-if)# shutdown
R1-Office(config-if)# no shutdown
R1-Office# show logging
```

Expected: messages with timestamps showing the interface changed to down, then to up.

### Step 6: ISP Router

```
enable
configure terminal
hostname ISP
no ip domain-lookup

interface serial 0/0/0
 description TO-R1-OFFICE
 ip address 10.0.0.2 255.255.255.252
 clock rate 64000
 no shutdown
exit

interface loopback 0
 description INTERNET-8.8.8.8
 ip address 8.8.8.8 255.255.255.255
exit

end
write memory
```

**Important:** Check the cable end first with `show controllers serial 0/0/0`. The `clock rate` command works only on the DCE end. If this router shows DTE, remove the `clock rate` line.

**Verify:**

```
show ip interface brief
ping 10.0.0.1
```

Run `ping 10.0.0.2` on R1-Office. Both should succeed, and Se0/0/0 should show `up / up` on both routers.

### Step 7: Routing and NAT

**On R1-Office:**

```
configure terminal

ip route 0.0.0.0 0.0.0.0 10.0.0.2

access-list 1 permit 192.168.10.0 0.0.0.31
access-list 1 permit 192.168.20.0 0.0.0.31
access-list 1 permit 192.168.30.0 0.0.0.15

interface gigabitEthernet 0/0.10
 ip nat inside
exit
interface gigabitEthernet 0/0.20
 ip nat inside
exit
interface gigabitEthernet 0/1
 ip nat inside
exit
interface serial 0/0/0
 ip nat outside
exit

ip nat inside source list 1 interface serial 0/0/0 overload

end
write memory
```

| Command | Explanation |
|---|---|
| `ip route 0.0.0.0 0.0.0.0 10.0.0.2` | Default route: send unknown destinations to the ISP |
| `access-list 1 permit ...` | Chooses which private addresses NAT translates |
| `ip nat inside` | Marks the private side |
| `ip nat outside` | Marks the internet side |
| `overload` | PAT: many private addresses share 10.0.0.1 |

**Wildcard masks:** These are the reverse of subnet masks. Subtract each octet from 255. For /27, the wildcard is 0.0.0.31. For /28, it is 0.0.0.15.

**On the ISP:**

```
configure terminal
ip route 192.168.10.0 255.255.255.224 10.0.0.1
ip route 192.168.20.0 255.255.255.224 10.0.0.1
ip route 192.168.30.0 255.255.255.240 10.0.0.1
end
write memory
```

Each line means: "To reach this network, send the packet to 10.0.0.1 (R1)." Without these routes, replies from the internet would have nowhere to go.

**Verify:**

```
show ip route
show ip nat translations
```

On R1, expect an `S*` default route via 10.0.0.2 and `C` (connected) entries for 10.0.0.0/30, 192.168.10.0/27, 192.168.20.0/27, and 192.168.30.0/28. On the ISP, expect three `S` routes via 10.0.0.1.

### Step 8: Backup to TFTP

```
R1-Office# copy running-config tftp:
Address or name of remote host []? 192.168.30.10
Destination filename [R1-Office-confg]?   (press Enter)
```

Check the server: **Services**, then **TFTP**. The file `R1-Office-confg` should be listed.

To restore:

```
R1-Office# copy tftp: running-config
Address or name of remote host []? 192.168.30.10
Source filename []? R1-Office-confg
```

---

## 7. Verification and Testing

Run these tests in order.

| # | Test | Where to run | Command or action | Expected result |
|---|---|---|---|---|
| 1 | DHCP address | Sales PC | `ipconfig` | 192.168.10.x, gateway .10.1, DNS .30.10 |
| 2 | Gateway | Sales PC | `ping 192.168.10.1` | Success |
| 3 | Inter-VLAN | Sales PC | `ping 192.168.20.2` (an IT PC) | Success |
| 4 | Server | Sales PC | `ping 192.168.30.10` | Success |
| 5 | DNS | Sales PC | `ping www.office.lab` | Reply from 192.168.30.10 |
| 6 | HTTP | Sales PC | Browser: `http://www.office.lab` | Page loads |
| 7 | WAN | R1-Office | `ping 10.0.0.2` | Success |
| 8 | Internet | Sales PC | `ping 8.8.8.8` | Success |
| 9 | Internet | IT PC | `ping 8.8.8.8` | Success |
| 10 | NAT | R1-Office | `show ip nat translations` | Entries listed |
| 11 | TFTP | R1-Office | `copy running-config tftp:` | Copy complete |
| 12 | Logs | R1-Office | `show logging` | Timestamped messages |

**Path check:**

```
tracert 8.8.8.8          (from a PC)
traceroute 8.8.8.8       (from a router)
```

Expected path: 192.168.10.1, then 10.0.0.2, then 8.8.8.8.

---

## 8. Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| PC gets 169.254.x.x | No DHCP reply | Check `show ip dhcp pool` and the trunk's allowed VLANs |
| PC cannot ping its gateway | PC is on the wrong VLAN port | Run `show vlan brief` and check the access port |
| Sub-interface is down | Trunk or `encapsulation` problem | Run `show interfaces trunk` on SW1 |
| Inter-VLAN ping fails | Wrong gateway on the PC, or sub-interface IP error | Run `ipconfig` on the PC and `show ip interface brief` on R1 |
| Server ping fails | Wrong server IP, mask, or gateway | Check the server's IP Configuration |
| `www.office.lab` does not resolve | DNS record missing or DNS service off | Check Services, then DNS |
| HTTP page does not load | HTTP service off | Turn on HTTP in Services |
| TFTP copy fails | TFTP off, or wrong server IP | Check the TFTP service and the IP in the copy command |
| Serial link is down/down | Clock rate missing on DCE, or wrong cable | Run `show controllers serial 0/0/0` and add `clock rate 64000` on the DCE end |
| Ping to 8.8.8.8 fails | Default route or ISP return route missing | Run `show ip route` on R1 and on the ISP |
| No NAT translations | Interface not marked inside or outside | Check `ip nat inside` and `ip nat outside` |
| Logs missing on server | Wrong logging host, or syslog service unavailable | Run `show running-config` and confirm the server address |
| Config lost after reboot | Forgot to save | Run `write memory` |

**Troubleshooting method:** Work from the PC outward. Check the PC's IP settings, then the switch port, then the trunk, then the sub-interface, then the route. Changing one thing at a time makes the cause easy to find.

---

## 9. Command Reference

| Task | Command |
|---|---|
| Show VLANs | `show vlan brief` |
| Show trunks | `show interfaces trunk` |
| Show port status | `show interfaces status` |
| Show interface summary | `show ip interface brief` |
| Show routing table | `show ip route` |
| Show DHCP leases | `show ip dhcp binding` |
| Show NAT entries | `show ip nat translations` |
| Show logs | `show logging` |
| Show serial type (DCE or DTE) | `show controllers serial 0/0/0` |
| Set the clock | `clock set 10:00:00 18 November 2026` |
| Save configuration | `write memory` |
| Back up to TFTP | `copy running-config tftp:` |
| Restore from TFTP | `copy tftp: running-config` |
| Trace the path (router) | `traceroute 8.8.8.8` |
| Trace the path (PC) | `tracert 8.8.8.8` |

**Helpful shortcuts:**
- Type `?` to see the commands available at the current prompt.
- Press Tab to complete a command.
- Use `do <command>` inside config mode to run a show command, for example `do show ip route`.

---

## 10. Lessons Learned

| Area | What was learned |
|---|---|
| LAN | Creating VLANs, access ports, trunks, and 802.1Q tagging |
| Router | Router-on-a-stick with sub-interfaces |
| DHCP | DORA process, excluded addresses, and pools |
| Servers | Static addressing, DNS records, HTTP, and TFTP |
| Logging | Severity levels, timestamps, and syslog hosts |
| WAN | Serial links, DCE and DTE, and clock rate |
| Routing | Default routes, static return routes, and the difference between them |
| NAT | Inside and outside interfaces, and PAT with overload |
| Administration | Passwords, local users, saving configuration, and TFTP backups |

---

## 11. Possible Improvements

1. **Replace static routes with OSPF** and compare the routing table with the static version.
2. **Add an ACL** to stop Sales from reaching the TFTP service while still allowing DNS and HTTP. Use numbered extended ACLs (100 to 199) or named ACLs, and apply them in the correct direction.
3. **Add SSH** by generating an RSA key with `crypto key generate rsa` and connecting with `ssh -l admin 192.168.10.1`.
4. **Give the switch a management IP** so its logs reach the syslog server.
5. **Add a redundant WAN link** and use HSRP for the gateway.

---

## 12. Repository Contents

| File | Description |
|---|---|
| `Lab.pkt` | Packet Tracer file with the finished topology |
| `README.md` | This document |
| `screenshots/` | Optional screenshots of the topology and the verification outputs |

---

## Author

**[Your Name]**
