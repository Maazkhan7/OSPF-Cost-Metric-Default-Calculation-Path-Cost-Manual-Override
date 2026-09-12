# 🌐 OSPF Cost Metric — Default Calculation, Path Cost & Manual Override

![Cisco](https://img.shields.io/badge/Cisco-Packet%20Tracer-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)
![OSPF](https://img.shields.io/badge/Protocol-OSPF-teal?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)
![Level](https://img.shields.io/badge/Level-CCNA-blue?style=for-the-badge)
![Topic](https://img.shields.io/badge/Focus-OSPF%20Cost-purple?style=for-the-badge)

> A hands-on Cisco Packet Tracer lab breaking down OSPF's **Cost metric** — how it's calculated by default from interface bandwidth, how total path cost is built up hop by hop, and how to manually override it to engineer preferred paths.

---

## 📖 Overview

Unlike RIP's simple hop count, OSPF uses **Cost** as its metric — a value derived from interface bandwidth by default, but fully adjustable by an administrator. This lab builds a 3-router OSPF topology and walks through the entire cost picture: verifying Cisco IOS's default cost formula per interface type, confirming how total path cost accumulates across a multi-hop route, and manually overriding a link's cost to directly change which path OSPF prefers.

**Goals of this lab:**
- Verify the default OSPF cost formula on Serial, GigabitEthernet, and Loopback interfaces
- Confirm total path cost is the **sum** of every outgoing interface cost along a route
- Manually override a link's OSPF cost and observe the effect on route selection
- Reinforce that OSPF Process ID has no bearing on adjacency formation

---

## 🗺️ Network Topology

```
L1: 10.1.1.1/32           L1: 20.1.1.1/32           L1: 30.1.1.1/32

[PC0-2] 192.168.1.x ─┐                                                    ┌─ 192.168.3.x [PC6-7]
                     ├─[Switch0]──[  R1  ]══cost 64══[  R2  ]══════[  R3  ]──[Switch2]─┤
                     ┘   Gi0/0      2911   S0/0/0     2911    fg0/1  2911    Gi0/0      ┘
                     192.168.1.1/24   cost 1              cost 1  192.168.3.1/24
                        (LAN1)                                              (LAN3)
                                                    Gi0/0 192.168.2.1/24  cost 1
                                                         │
                                                    [Switch1]
                                                    [PC3-5] 192.168.2.x
                                                        (LAN2)

Routing Protocol: OSPF, single Area 0
R1: router ospf 1   |   R2: router ospf 2   (different Process IDs — by design, to prove it doesn't matter)
```

| Device | Role                                                            |
|--------|--------------------------------------------------------------------|
| R1     | Edge router – LAN1 (192.168.1.0/24), OSPF Process ID **1**       |
| R2     | Middle router – LAN2 (192.168.2.0/24), OSPF Process ID **2**     |
| R3     | Edge router – LAN3 (192.168.3.0/24)                               |

---

## 🧾 IP Addressing Table

| Device | Interface   | IP Address    | Subnet Mask       | Default OSPF Cost |
|--------|-------------|-----------------|---------------------|----------------------|
| R1     | Gi0/0       | 192.168.1.1     | 255.255.255.0        | 1                    |
| R1     | S0/0/0      | 1.1.1.1         | 255.255.255.0        | 64                   |
| R1     | Loopback1   | 10.1.1.1        | 255.255.255.255      | 1 (always, regardless of bandwidth) |
| R2     | S0/0/0      | 1.1.1.2         | 255.255.255.0        | 64                   |
| R2     | Gi0/1       | 2.1.1.1         | 255.255.255.0        | 1                    |
| R2     | Gi0/0       | 192.168.2.1     | 255.255.255.0        | 1                    |
| R2     | Loopback1   | 20.1.1.1        | 255.255.255.255      | 1                    |
| R3     | Fg0/1       | 2.1.1.2         | 255.255.255.0        | 1                    |
| R3     | Gi0/0       | 192.168.3.1     | 255.255.255.0        | 1                    |
| R3     | Loopback1   | 30.1.1.1        | 255.255.255.255      | 1                    |

---

## ⚙️ Key CLI Configuration

### 🔹 R1 — OSPF Process ID 1
```
R1(config)#router ospf 1
R1(config-router)#network 1.0.0.0 0.255.255.255 area 0
R1(config-router)#network 10.1.1.1 0.0.0.0 area 0
R1(config-router)#network 192.168.1.0 0.0.0.255 area 0
```

### 🔹 R2 — OSPF Process ID 2 (deliberately different from R1)
```
R2(config)#router ospf 2
R2(config-router)#network 1.0.0.0 0.255.255.255 area 0
R2(config-router)#network 192.168.2.0 0.0.0.255 area 0
```
```
01:00:17: %OSPF-5-ADJCHG: Process 2, Nbr 10.1.1.1 on Serial0/0/0 from LOADING to FULL, Loading Done
```
> **Confirmed:** R1 (Process ID 1) and R2 (Process ID 2) form a full OSPF adjacency despite having different Process IDs. This reinforces that the Process ID is purely a **local label** — it never needs to match between neighboring routers.

---

## 🔍 Verification & Test Sequence

### 1️⃣ Default cost per interface type — `show ip ospf interface`
```
R1#show ip ospf interface

Serial0/0/0 is up, line protocol is up
  Internet address is 1.1.1.1/24, Area 0
  Process ID 1, Router ID 10.1.1.1, Network Type POINT-TO-POINT, Cost: 64
  Adjacent with neighbor 20.1.1.1

Loopback1 is up, line protocol is up
  Internet address is 10.1.1.1/32, Area 0
  Process ID 1, Router ID 10.1.1.1, Network Type LOOPBACK, Cost: 1
  Loopback interface is treated as a stub Host

GigabitEthernet0/0 is up, line protocol is up
  Internet address is 192.168.1.1/24, Area 0
  Process ID 1, Router ID 10.1.1.1, Network Type BROADCAST, Cost: 1
  Designated Router (ID) 10.1.1.1, Interface address 192.168.1.1
```
> **Confirmed default cost formula:** `Cost = 100,000 ÷ Bandwidth (kbps)`
> - **Serial0/0/0** — default bandwidth 1544 kbps → `100,000 / 1544 ≈ 64` ✅
> - **GigabitEthernet0/0** — bandwidth 1,000,000 kbps → result rounds down below 1, so IOS enforces the **minimum cost of 1** ✅
> - **Loopback1** — always assigned cost **1** regardless of configured bandwidth, because OSPF treats loopbacks as "stub hosts," not real transit links

### 2️⃣ Total path cost — sum of every outgoing interface along the route
```
R1#show ip route
O    192.168.2.0/24 [110/65] via 1.1.1.2, 00:07:34, Serial0/0/0
```
> **[110/65]** → `110` is OSPF's Administrative Distance; `65` is the **total path cost**. This is calculated as:
> `R1's Serial0/0/0 cost (64) + R2's GigabitEthernet0/0 cost (1) = 65`
>
> **Key rule confirmed:** OSPF path cost is the sum of the cost of every **outgoing** interface a packet crosses on its way to the destination — not the incoming interface, and not a hop count.

### 3️⃣ Manually overriding cost — `ip ospf cost`
```
R1(config)#interface serial 0/0/0
R1(config-if)#ip ospf cost 100
```
```
R1#show ip route
O    192.168.2.0/24 [110/101] via 1.1.1.2, 00:00:02, Serial0/0/0
O    20.1.1.1/32   [110/101] via 1.1.1.2, 00:00:02, Serial0/0/0
```
> **Confirmed:** After manually setting the Serial0/0/0 cost to `100`, the total path cost immediately updates to `101` (`100 + 1`, R2's LAN-facing interface cost). This proves manual cost overrides take full effect in the SPF calculation instantly — no protocol restart required — and is the standard technique for **traffic engineering** preferred paths in an OSPF network with multiple routes to the same destination.

---

## 📊 Cost Summary Table

| Interface Type      | Default Bandwidth | Default OSPF Cost | Notes                                  |
|-----------------------|----------------------|-----------------------|-------------------------------------------|
| Serial (default)       | 1544 kbps              | 64                     | `100,000 / 1544 ≈ 64`                      |
| GigabitEthernet         | 1,000,000 kbps         | 1 (minimum enforced)   | Formula result < 1, rounded up to 1        |
| Loopback                | N/A                     | 1 (always)             | Treated as a stub host, not a transit link |
| Serial (after manual override) | N/A (manually set)  | 100                    | Set via `ip ospf cost 100`                 |

---

## 🎯 Key Learnings

- OSPF's default cost formula is `100,000 ÷ interface bandwidth (in kbps)` — commonly called the "reference bandwidth" method.
- **Loopback interfaces always get a default cost of 1**, regardless of any bandwidth configured on them, because OSPF classifies them as stub hosts rather than transit interfaces.
- **Total OSPF path cost is cumulative** — it's the sum of every outgoing interface's cost along the entire path to the destination, shown as the second number in `[AD/Cost]` in `show ip route`.
- `ip ospf cost <value>`, applied directly on an interface, **manually overrides** the calculated default cost — and the change is reflected immediately in the routing table without needing to clear or restart the OSPF process.
- Manually adjusting cost is the primary real-world method for **traffic engineering** in OSPF — steering traffic toward a preferred link when multiple physical paths exist to the same destination.
- OSPF Process ID has **zero effect** on adjacency formation — two routers with completely different Process IDs (as demonstrated here: 1 and 2) will still form a full neighbor relationship as long as their Area, subnet, and other OSPF parameters match.

---

## ✅ Outcomes

- Verified Cisco IOS's default OSPF cost formula across three different interface types
- Confirmed exactly how total path cost accumulates across a multi-hop OSPF route
- Successfully manually overrode interface cost and observed its immediate effect on route selection
- Reinforced that OSPF Process ID is locally significant and irrelevant to adjacency formation
- Built a metric-focused, interview-ready lab to complement the earlier OSPF Router ID project

---

## 🛠️ Tools Used

- Cisco Packet Tracer
- Cisco IOS (OSPF)
- `show ip ospf interface`, `show ip route`, `ip ospf cost`

---

## 👤 Author

**Maaz Khan**
CCNA Certified | Network & NOC Engineer
📍 Lower Dir, KPK, Pakistan
🔗 [LinkedIn](https://www.linkedin.com/in/maazkhanms) · [GitHub](https://github.com/maazkhanms)

---

⭐ If you found this lab useful, consider starring the repo — more RIP, OSPF, EIGRP, IPv6, and routing-fundamentals labs coming in this series!
