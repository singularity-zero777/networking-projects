# Layer 2 Isolation to ROAS Connectivity - Network Implementation & Troubleshooting Report

**Topology Scope:** 4 Switches (SW1–SW4), 6 End Hosts (PC1–PC6), and 1 Router (R1)
![[roas_topology.png]]
## 1. Initial Setup & Network Intent

The goal was to build a segmented network with 4 switches carrying traffic across three distinct subnets/departments:

- **VLAN 10:** PC1, PC2 (`172.16.0.0/25` subnet)
    
- **VLAN 20:** PC5, PC6 (`172.16.0.128/25` subnet)
    
- **VLAN 30:** PC3, PC4 (`172.16.1.0/24` subnet)
    

Trunk links were established between interconnected switches (`SW1`–`SW4`) using IEEE 802.1Q tagging, and host ports were assigned to their respective VLANs.

## 2. Phase 1: The Layer 2 "Brick Wall" (Why Pings Failed)

After setting up the switches, VLANs, and cables, an immediate test was conducted by pinging from **PC1 (VLAN 10)** to **PC3 (VLAN 30)**.

### The Result:

Plaintext

```
C:\> ping 172.16.1.10
Pinging 172.16.1.10 with 32 bytes of data:
Request timed out.
Request timed out.
Request timed out.
Request timed out.

Ping statistics for 172.16.1.10:
    Packets: Sent = 4, Received = 0, Lost = 4 (100% loss)
```

### Why Couldn't They Talk?

Nothing was broken at the cabling level—this was **VLAN isolation working exactly as designed**:

1. **VLANs create separate virtual switches:** Assigning PC1 to VLAN 10 and PC3 to VLAN 30 is logically identical to plugging them into two separate physical switches in different buildings with no cable connecting them.
    
2. **Layer 2 switches do not route IP traffic:** Switches only examine Layer 2 MAC addresses within the _same_ VLAN. When a switch receives a frame tagged for VLAN 10, it strictly refuses to forward that frame out of a port assigned to VLAN 30.
    
3. **Cross-subnet traffic requires Layer 3:** Because `172.16.0.10` and `172.16.1.10` sit on different IP subnets, traffic must pass through a Layer 3 routing device to cross the boundary.
    

## 3. Layer 2 Troubleshooting Log

During initial switch verification, one actual configuration error was discovered and corrected:

### SW2 Port Misconfiguration (`Fa0/4`)

- **The Problem:** PC4 (VLAN 30) lost all network connectivity and couldn't even ping PC3 (which was on the same VLAN).
    
- **Root Cause:** Interface `Fa0/4` on SW2 (connected directly to PC4) was accidentally configured as a **Trunk link** instead of an **Access link**. End-user PCs do not recognize 802.1Q VLAN tags, causing PC4 to drop incoming frames.
    
- **The Fix:**
    
    Plaintext
    
    ```
    SW2(config)# interface FastEthernet0/4
    SW2(config-if)# switchport mode access
    SW2(config-if)# switchport access vlan 30
    ```
    

## 4. The Solution: Upgrading to Router-on-a-Stick (ROAS)

To break through the Layer 2 wall and allow controlled communication between VLANs, a Layer 3 router (**R1**) was added to the topology using **Router-on-a-Stick (ROAS)**. A single physical trunk cable was connected between `SW1 Fa0/5` and `R1 Gi0/0`.

## 5. Phase 2: Router Configuration & Troubleshooting

### The IP Subnet Overlap Error

While setting up subinterfaces on the router, the following error occurred:

Plaintext

```
Router(config-subif)# ip address 172.16.0.129 255.255.255.128
% 172.16.0.128 overlaps with GigabitEthernet0/0.10
```

- **Root Cause:** Subinterface `Gi0/0.10` had been assigned a `/24` subnet mask (`255.255.255.0`) instead of a `/25` mask (`255.255.255.128`). The `/24` mask accidentally claimed the entire range from `172.16.0.0` to `172.16.0.255`, preventing VLAN 20 from using `172.16.0.129`.
    
- **The Fix:** Reconfigured `Gi0/0.10` with the correct `/25` mask to free up the upper half of the IP range for VLAN 20.
    

### Final Corrected Router Configuration (R1)

Plaintext

```
enable
configure terminal
hostname R1

! Gateway for VLAN 10
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 172.16.0.1 255.255.255.128
 no shutdown

! Gateway for VLAN 20
interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 172.16.0.129 255.255.255.128
 no shutdown

! Gateway for VLAN 30
interface GigabitEthernet0/0.30
 encapsulation dot1Q 30
 ip address 172.16.1.1 255.255.255.0
 no shutdown

! Enable physical interface
interface GigabitEthernet0/0
 no shutdown
exit
end
write memory
```

## 6. Final Verification & Breakthrough

After updating all host PCs with their respective Default Gateways (`172.16.0.1`, `172.16.0.129`, and `172.16.1.1`), the cross-VLAN ping from **PC1** to **PC3** was re-tested.

### Attempt 1: Initial Ping

Plaintext

```
C:\> ping 172.16.1.10

Request timed out.
Reply from 172.16.1.10: bytes=32 time<1ms TTL=127
Reply from 172.16.1.10: bytes=32 time=10ms TTL=127
Reply from 172.16.1.10: bytes=32 time<1ms TTL=127

Packets: Sent = 4, Received = 3, Lost = 1 (25% loss)
```

> **Note on 1st Packet Loss:** The first packet dropped due to normal **ARP Delay**. Router R1 knew PC3's IP address, but needed to issue an ARP request to resolve PC3's MAC address before completing the ICMP delivery.

### Attempt 2: Immediate Re-Test

Plaintext

```
C:\> ping 172.16.1.10

Reply from 172.16.1.10: bytes=32 time=1ms TTL=127
Reply from 172.16.1.10: bytes=32 time<1ms TTL=127
Reply from 172.16.1.10: bytes=32 time=29ms TTL=127
Reply from 172.16.1.10: bytes=32 time<1ms TTL=127

Ping statistics for 172.16.1.10:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
```

- **Result:** **100% Success.** Once R1 cached PC3's MAC address, traffic flowed seamlessly between VLAN 10 and VLAN 30.
    

## 7. Summary Comparison

| **Metric / Scenario**  | **Phase 1 (Switches Only)**         | **Phase 2 (ROAS Installed)**        |
| ---------------------- | ----------------------------------- | ----------------------------------- |
| **Same-VLAN Pings**    | Working (`PC1` $\rightarrow$ `PC2`) | Working (`PC1` $\rightarrow$ `PC2`) |
| **Cross-VLAN Pings**   | ❌ Blocked (Layer 2 Isolation)       | Routed (Layer 3 Gateway)            |
| **Switch Port Status** | Access & Trunk ports configured     | Access & Trunk ports configured     |
| **Default Gateways**   | None configured on PCs              | Configured to Router Subinterfaces  |
| **Routing Mechanism**  | None                                | 802.1Q Tagged Subinterfaces         |
