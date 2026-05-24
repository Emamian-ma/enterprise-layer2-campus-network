```md
# Enterprise Layer 2 Campus Network (CCNA Lab)

This project is a hands-on implementation of an **enterprise-style Layer 2 campus network** using Cisco switches in **GNS3**.  
It is designed as a practical CCNA-level lab focusing on:

- Switching
- Layer 2 security
- Management & remote access (SSH)
- Enterprise campus design best practices

---

## 🏛️ Network Design Overview

The topology follows a **hierarchical campus design** with:

- **Distribution Layer**:  
  - `DSW1`, `DSW2` (distribution switches with EtherChannel between them)
- **Access Layer**:  
  - `ASW1`, `ASW2` (access switches with user PCs connected)
- **Management VLAN** for switch administration

The lab is fully virtualized in **GNS3**.

---

## 🗺️ Topology Diagram

![Enterprise Layer 2 Campus Topology](topology/topology.png)

- Each **Access Switch (ASW)** has multiple PCs connected.
- VLANs are extended from Access to Distribution over **802.1Q trunks**.
- Distribution switches are interconnected via **LACP EtherChannel**.

---

## 🧩 VLAN & Subnet Design

### VLANs

| VLAN ID | Name         | Purpose              |
|--------:|--------------|----------------------|
| 10      | ENGINEERING  | Engineering users    |
| 20      | FINANCE      | Finance users        |
| 99      | MANAGEMENT   | Switch management    |

### IP Addressing

| VLAN  | Subnet             | Usage              |
|-------|--------------------|--------------------|
| 10    | `192.168.10.0/24`  | Engineering PCs    |
| 20    | `192.168.20.0/24`  | Finance PCs        |
| 99    | `192.168.99.0/24`  | Management SVI/PCs |

> Note: This is a **pure Layer 2 lab** – there is **no inter-VLAN routing**.  
> Each VLAN is isolated by design.

---

## 🔧 Key Technologies Implemented

### Switching & Trunking

- Static VLAN assignment on access ports
- **802.1Q trunking** between:
  - Access → Distribution switches
  - Distribution ↔ Distribution (via Port-Channel)
- `switchport trunk encapsulation dot1q`
- `switchport mode trunk`
- `switchport nonegotiate` on trunk ports

### EtherChannel (LACP)

- **LACP-based EtherChannel** between `DSW1` and `DSW2`
- Multiple physical links bundled into:
  - `Port-channel1`
- Config example:

```cisco
interface range Gi0/0 - 1
  channel-group 1 mode active
!
interface Port-channel1
  switchport
  switchport trunk encapsulation dot1q
  switchport mode trunk
```

---

## 🌳 Spanning Tree Engineering

- **STP Root Bridge selection** for load balancing:
  - `DSW1` → Root Primary for VLAN 10 & 99
  - `DSW2` → Root Primary for VLAN 20
- Root Secondary configured for redundancy
- `portfast` enabled on access ports
- `bpduguard` enabled on edge/access ports

Verification commands:

```cisco
show spanning-tree root
show spanning-tree vlan 10
show spanning-tree vlan 20
show spanning-tree vlan 99
```

---

## 🔐 Layer 2 Security

### Port Security

- Applied on access ports:
  - `switchport port-security`
  - `switchport port-security maximum 1`
  - `switchport port-security mac-address sticky`
  - `switchport port-security violation shutdown`

Verification:

```cisco
show port-security
show port-security interface <int>
```

### BPDU Guard

- Enabled on all edge/access ports to protect STP:

```cisco
spanning-tree portfast
spanning-tree bpduguard enable
```

### DHCP Snooping

- Enabled globally and per VLAN (10, 20, 99)
- Trusted interfaces: uplinks/trunks towards DHCP server/Distribution

```cisco
ip dhcp snooping
ip dhcp snooping vlan 10,20,99
interface <trunk-uplink>
  ip dhcp snooping trust
```

### Dynamic ARP Inspection (DAI)

- Configured to validate ARP packets against DHCP Snooping bindings.

> During testing, DAI initially dropped ARP traffic because **VPCS used static IPs**  
> and there were no DHCP Snooping bindings.  
> For this lab, DAI was disabled on the user VLANs to restore ARP functionality.

```cisco
no ip arp inspection vlan 10,20
```

---

## 🛠️ Management & SSH Access

### Management VLAN (99)

- All switches have an SVI for VLAN 99:

```cisco
interface Vlan99
  ip address 192.168.99.X 255.255.255.0
  no shutdown
```

### Default Gateway on Access Switches

```cisco
ip default-gateway 192.168.99.1
```

(assuming `DSW1` is the management default gateway)

### SSH Configuration

- Local user defined:

```cisco
username admin privilege 15 secret <password>
```

- Domain name and RSA key:

```cisco
ip domain-name lab.local
crypto key generate rsa modulus 2048
```

- VTY lines configured for SSH:

```cisco
line vty 0 4
  transport input ssh
  login local
```

- SSH test from another switch:

```cisco
ssh -l admin 192.168.99.1
```

---

## 💻 End-Host Configuration (VPCS in GNS3)

- VPCS used to simulate end-hosts in VLAN 10, 20, and 99.
- IP addressing via:

```bash
ip 192.168.10.10 255.255.255.0
save
```

- No default gateway configured on PCs, since **no inter-VLAN routing** is present.

---

## ✅ Verification & Testing

### Connectivity Tests

- **Intra-VLAN** (expected to succeed):

  - PC in VLAN 10 ↔ PC in VLAN 10  
  - PC in VLAN 20 ↔ PC in VLAN 20  

- **Inter-VLAN** (expected to fail):

  - VLAN 10 ↔ VLAN 20  
  - Data VLANs ↔ VLAN 99

### Management Reachability

- From an access switch (e.g. `ASW1`):

```cisco
ping 192.168.99.1
ssh -l admin 192.168.99.1
```

---

## 📁 Repository Structure

```text
.
├── topology/
│   └── topology.png           # Topology diagram
│
├── configs/
│   ├── DSW1.txt               # DSW1 running-config
│   ├── DSW2.txt               # DSW2 running-config
│   ├── ASW1.txt               # ASW1 running-config
│   └── ASW2.txt               # ASW2 running-config
│
├── docs/                      # (Optional) Design & troubleshooting notes
│   ├── ip-plan.md
│   ├── vlan-design.md
│   └── troubleshooting.md
│
└── README.md
```

---

## 🧠 Learning Outcomes

This lab reinforces the following CCNA-level concepts:

- Hierarchical Campus Design
- VLANs and 802.1Q trunking
- EtherChannel (LACP)
- Spanning Tree Root Bridge engineering and port roles
- Port Security and edge port hardening
- DHCP Snooping and Dynamic ARP Inspection
- Management VLAN design
- SSH-based switch management
- Layer 2 troubleshooting (DAI, ARP, trunks, STP)

---

## 👤 Author

- **Name:** RootUser (Seyed mohammad ali Emamian)  
- **Role:** Computer Engineer / Network Enthusiast  
- **Interests:** Cisco networking, open-source, Linux, enterprise network design
