# SDN Packet Logger

**Real-time packet capture and analysis using an SDN controller (Ryu + OpenFlow 1.3)**

A Mininet-based simulation that demonstrates controller–switch interaction, L2 learning, firewall access control, anomaly detection, and a live web dashboard.

---

## Problem Statement

Traditional network monitoring requires dedicated hardware probes. In Software-Defined Networking (SDN), the controller has global visibility of all traffic. This project exploits that visibility to:

- Capture and parse every packet traversing the network at layer 2–4
- Identify protocol stacks (Ethernet → IP → TCP/UDP/ICMP/ARP)
- Implement L2 learning switch logic with proactive flow rule installation
- Enforce firewall policies (block specific src→dst pairs at priority 100)
- Detect network anomalies (ICMP flood, port scan) using sliding-window counters
- store data diplayed in dashbord live in sdn_traffic.txt file

---

## Architecture

```
[h1]──┐
[h2]──┤──[s1 OVSKernelSwitch]──OpenFlow 1.3──[c0 Ryu Controller]
[h3]──┘                                              │
                                              packet_logger.py
                                              (L2 switch + logger
                                               + firewall + IDS)
                                              
```

---

## Flow Rule Design

| Priority | Match | Action | Purpose |
|---|---|---|---|
| 100 | `ipv4_src=X, ipv4_dst=Y` | DROP | Firewall block rules |
| 10  | `in_port=P, eth_dst=M`   | OUTPUT specific port | L2 unicast forwarding |
| 0   | (wildcard — table-miss)  | CONTROLLER | Send new packets up for logging |

Flow rules at priority 10 include `idle_timeout=10s` and `hard_timeout=30s` to keep the table clean.

---

## Features

- **L2 Learning Switch** — MAC address table, unicast forwarding, no flooding after learning
- **Firewall** — High-priority DROP rules, manageable from the web dashboard at runtime
- **Packet Logger** — Full header parsing: Ethernet, ARP, IPv4, IPv6, TCP, UDP, ICMP
- **Anomaly Detection** — ICMP flood (>20 pkts/5s), port scan (>10 distinct ports/5s)


---

## Setup & Execution

### Requirements

```
Ubuntu 20.04+ / Linux
Python 3.8+
Mininet 2.3+
Open vSwitch
Ryu SDN framework
Flask
```

### Install dependencies

```bash
# Mininet
sudo apt-get install mininet

# Python packages
pip install ryu flask

# Open vSwitch (usually included with Mininet)
sudo apt-get install openvswitch-switch
```

### Run

**Terminal 1 — Start the Ryu controller:**
```bash
ryu-manager packet_logger.py
```




**Terminal 2 — Start the Mininet topology (requires sudo):**
```bash
sudo python3 topology.py
```



---

## Test Scenarios

### Scenario A — Normal forwarding (allowed)
```
mininet> h1 ping -c 4 h2
```
Expected: 4 ICMP packets logged, RTT ~4ms (2ms each way), flow rule installed at priority 10.

### Scenario B — Blocked flow (firewall)
Add rule in dashboard: src=10.0.0.1, dst=10.0.0.3, click Block.
```
mininet> h1 ping -c 4 h3
```
Expected: 100% packet loss, "BLOCKED" status in packet table, alert logged.



### Scenario C — View flow table
```
mininet> sh ovs-ofctl -O OpenFlow13 dump-flows s1

```
Expected: table-miss rule + learned unicast forwarding rules.

### Scenario D — Port statistics
```
mininet>sh ovs-ofctl -O OpenFlow13 dump-ports s1

```
Expected: per-port rx/tx packet counts.

---
### Scenario E — UDP Traffic Test (Deep Packet Inspection)
'''
mininet> xterm h1 h2

In h2 (receiver):

nc -u -l 5555

In h1 (sender):

echo "Testing SDN UDP Logging" | nc -u 10.0.0.2 5555
''''
Expected:
The message is displayed in the h2 terminal. Simultaneously, the Ryu controller logs the packet with protocol details (UDP) along with source and destination port numbers. If a firewall rule is configured to block port 5555, the packet will be dropped and no message will appear on h2.

Description:
This scenario validates the system’s ability to inspect and process UDP traffic, which is inherently connectionless and commonly overlooked in basic network testing. By generating custom UDP packets using netcat, the controller demonstrates accurate Layer 4 protocol detection, port extraction, and real-time logging. Additionally, this test confirms that port-based firewall rules are effectively enforced, showcasing the robustness of the Deep Packet Inspection mechanism across multiple transport protocols.

## Expected Output

**Ryu controller terminal:**
```
2026-04-15 10:32:01  [SWITCH CONNECTED] dpid=1
2026-04-15 10:32:05  [PKT #00001] ... Protocols: Ethernet / IPv4 / ICMP  Service: ICMP  IP: 10.0.0.1 → 10.0.0.2
2026-04-15 10:32:05  [PKT #00002] ... Protocols: Ethernet / ARP  Service: ARP
```

**Dashboard:** Live updating packet table, charts, alerts panel.

**logs/packets.log:** Full timestamped log file of every packet.

---

## References

- Ryu SDN Framework Documentation: https://ryu.readthedocs.io
- OpenFlow 1.3 Specification: https://opennetworking.org/wp-content/uploads/2014/10/openflow-spec-v1.3.0.pdf
- Mininet Documentation: http://mininet.org/
- Open vSwitch: https://www.openvswitch.org/
