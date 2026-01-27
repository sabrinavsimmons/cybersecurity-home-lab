# HOMELAB PHASE 1 - COMPLETE DOCUMENTATION
## Static Routing & Network Access Configuration
### Including Phase 2 Analysis and Architectural Decision

Date: January 13, 2026 (Updated after Phase 2 analysis)

═══════════════════════════════════════════════════════════════════

## EXECUTIVE SUMMARY

This document provides complete documentation of the Phase 1 homelab implementation, including all troubleshooting steps, final configurations, lessons learned, AND the Phase 2 attempt that revealed critical architectural constraints.

Phase 1 established wireless access from a MacBook to lab resources (pfSense, Proxmox, Pi-hole) via static routing between separate networks. An attempt was made to migrate to Phase 2 (pfSense as edge router), which revealed that **Phase 1 is the architecturally correct solution given physical constraints** (WiFi-only ISP connection).

**Key Achievements:**
- ✅ Wireless access to entire homelab from MacBook
- ✅ Static routing configured between GL.iNet (192.168.8.0/24) and pfSense LAN (192.168.1.0/24)
- ✅ pfSense firewall rules properly configured
- ✅ Proxmox accessible at 192.168.1.10
- ✅ Pi-hole installed and accessible at 192.168.1.20
- ✅ Validated Phase 2 impossibility due to physical constraints
- ✅ Confirmed Phase 1 as optimal architecture for environment

**Total Time Investment:** 
- Phase 1 Implementation: ~8 hours
- Phase 2 Attempt & Analysis: ~3 hours
- Total: ~11 hours of hands-on learning

═══════════════════════════════════════════════════════════════════

## TABLE OF CONTENTS

1. Network Architecture (Phase 1 - Current)
2. Routing Configuration
3. pfSense Firewall Configuration
4. Device Configuration
5. Troubleshooting Journey (Phase 1)
6. Testing & Verification
7. Phase 2 Attempt & Analysis (CRITICAL SECTION)
8. Why Phase 1 is Correct for WiFi-Only ISP
9. Next Steps & Future Improvements
10. Key Lessons Learned
11. Appendix

═══════════════════════════════════════════════════════════════════

## 1. NETWORK ARCHITECTURE (PHASE 1 - CURRENT)

### 1.1 Final Network Topology

```
Internet (ISP WiFi)
        ↓ WiFi
   GL.iNet Router (WiFi Client + Router)
   Network: 192.168.8.0/24
   Gateway: 192.168.8.1
   Role: Connects to ISP via WiFi, routes to pfSense
        ↓ Ethernet
   pfSense M910q (Lab Firewall)
   WAN: 192.168.8.196 (static IP on GL.iNet network)
   LAN: 192.168.1.1 (lab network gateway)
   Role: Firewall and router for lab network
        ↓ Ethernet
   Netgear Switch
        ├── Proxmox M720q: 192.168.1.10
        └── Pi-hole (Raspberry Pi): 192.168.1.20
```

**Key Architectural Feature:**
Static route on GL.iNet enables MacBook (on 192.168.8.x WiFi) to access lab devices (on 192.168.1.x network) wirelessly.

### 1.2 IP Address Assignments

**GL.iNet Network (192.168.8.0/24)**
- GL.iNet Gateway: 192.168.8.1
- MacBook WiFi: 192.168.8.160 (DHCP from GL.iNet)
- pfSense WAN: 192.168.8.196 (static)

**pfSense LAN Network (192.168.1.0/24)**
- pfSense LAN Gateway: 192.168.1.1
- Proxmox: 192.168.1.10 (static - configured in OS)
- Pi-hole: 192.168.1.20 (DHCP reservation in pfSense)

### 1.3 Physical Connections

- ISP Router → (WiFi) → GL.iNet WAN (WiFi client mode)
- GL.iNet LAN port → (Ethernet) → pfSense WAN port
- pfSense LAN port → (Ethernet) → Netgear Switch
- Netgear Switch → (Ethernet) → Proxmox M720q
- Netgear Switch → (Ethernet) → Raspberry Pi (Pi-hole)
- MacBook → (WiFi) → GL.iNet

**Critical Physical Constraint:**
ISP connection point is distant from lab. Running ethernet cable to ISP is not feasible. GL.iNet MUST connect to ISP via WiFi.

═══════════════════════════════════════════════════════════════════

## 2. ROUTING CONFIGURATION

### 2.1 Static Route on GL.iNet

**Purpose:** Enable GL.iNet to route traffic destined for pfSense LAN network (192.168.1.0/24) to pfSense WAN interface (192.168.8.196).

**Configuration Method:** SSH + UCI Commands (OpenWRT)

**Commands Executed:**

```bash
ssh root@192.168.8.1

# Add temporary route
ip route add 192.168.1.0/24 via 192.168.8.196

# Make persistent across reboots
uci add network route
uci set network.@route[-1].interface='lan'
uci set network.@route[-1].target='192.168.1.0'
uci set network.@route[-1].netmask='255.255.255.0'
uci set network.@route[-1].gateway='192.168.8.196'
uci commit network
/etc/init.d/network reload
```

**Verification:**

```bash
ip route show | grep 192.168.1
```

Expected output: `192.168.1.0/24 via 192.168.8.196 dev br-lan proto static`

**Why This is Necessary:**
Without this route, GL.iNet would drop packets destined for 192.168.1.x because it doesn't know how to reach that network. The static route tells GL.iNet: "For anything going to 192.168.1.0/24, forward it to 192.168.8.196 (pfSense WAN)."

### 2.2 Traffic Flow Explanation

When MacBook (192.168.8.160) wants to access Proxmox (192.168.1.10):

1. MacBook checks: Is 192.168.1.10 on my local subnet (192.168.8.0/24)? → NO
2. MacBook sends packet to default gateway (GL.iNet at 192.168.8.1)
3. GL.iNet receives packet destined for 192.168.1.10
4. GL.iNet checks routing table: Found static route → 192.168.1.0/24 via 192.168.8.196
5. GL.iNet forwards packet to pfSense WAN (192.168.8.196)
6. pfSense checks firewall rules: WAN rule allows 192.168.8.0/24 → 192.168.1.0/24
7. pfSense forwards packet to Proxmox on LAN network
8. Proxmox receives packet and responds back through same path in reverse
9. Response delivered to MacBook

**This is classic inter-network routing - fundamental networking concept.**

═══════════════════════════════════════════════════════════════════

## 3. PFSENSE FIREWALL CONFIGURATION

### 3.1 Critical Setting: Block Private Networks

**Initial Problem:**
pfSense blocked all traffic from GL.iNet network (192.168.8.0/24) due to default 'Block private networks from WAN' rule.

**Solution:**
- Navigation: Interfaces → WAN → Reserved Networks section
- Action: Unchecked 'Block private networks and loopback addresses'
- Reason: GL.iNet uses private IP space (RFC1918), which needs to communicate with pfSense LAN

**Security Context:**
This setting is normally enabled for internet-facing WAN interfaces to block spoofed private IPs from the internet. In our architecture, pfSense WAN connects to GL.iNet (a trusted private network), so this protection is not needed and would block legitimate traffic.

### 3.2 WAN Firewall Rule

**Configuration:**
- Interface: WAN
- Action: Pass
- Protocol: Any (IPv4)
- Source Type: Network
- Source Network: 192.168.8.0/24
- Destination Type: Network
- Destination Network: 192.168.1.0/24
- Description: Allow GL.iNet network to access pfSense LAN network

**Purpose:**
Allow traffic from GL.iNet network (where MacBook WiFi connects) to access pfSense LAN network (where lab devices live).

**Future Security Improvement:**
Current rule allows ALL protocols. Should be tightened to only allow specific services:
- TCP 443/80 (pfSense web GUI)
- TCP 8006 (Proxmox web GUI)
- TCP 80 (Pi-hole web interface)
- UDP 53 (Pi-hole DNS)
- TCP 22 (SSH, if needed)

This follows the principle of least privilege.

═══════════════════════════════════════════════════════════════════

## 4. DEVICE CONFIGURATION

### 4.1 Proxmox Network Configuration

**File: /etc/network/interfaces**

```
auto lo
iface lo inet loopback

auto vmbr0
iface vmbr0 inet static
    address 192.168.1.10/24
    gateway 192.168.1.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
```

**Apply Configuration:**

```bash
systemctl restart networking
# OR
systemctl restart pveproxy
systemctl restart pvedaemon
```

**Access:** https://192.168.1.10:8006

### 4.2 Pi-hole Configuration

**Configuration Method:** DHCP Reservation in pfSense

**pfSense DHCP Reservation:**
- Location: Services → DHCP Server → LAN tab → DHCP Static Mappings
- MAC Address: 2c:cf:67:ef:51:3e
- IP Address: 192.168.1.20
- Hostname: pihole
- Description: Pi-hole DNS Server

**Why DHCP Reservation:**
Raspberry Pi OS network configuration proved inconsistent across different network managers. DHCP reservation provides:
- Centralized IP management in pfSense
- Reliability across OS updates
- Same result as static IP (consistent address) without OS-level configuration complexity

### 4.3 Pi-hole Installation

**OS:** Raspberry Pi OS Lite (64-bit)

**Installation:**
```bash
curl -sSL https://install.pi-hole.net | bash
```

**Configuration:**
- Upstream DNS: Cloudflare (DNSSEC)
- Block lists: Default
- Web interface: Enabled
- Logging: Enabled
- Privacy mode: Show everything

**Credentials:**
- Web Interface: http://192.168.1.20/admin
- Admin Password: FCHzqpr0 (change this!)

═══════════════════════════════════════════════════════════════════

## 5. TROUBLESHOOTING JOURNEY (PHASE 1)

### 5.1 Initial Routing Confusion

**Problem:** Proxmox reported IP 192.168.8.207 despite being physically connected behind pfSense.

**Root Cause:** Old network configuration from previous setup.

**Solution:** Reconfigured /etc/network/interfaces with correct IP and gateway.

**Lesson:** Always verify device configuration matches physical topology.

### 5.2 pfSense Blocking Private Networks

**Problem:** Traffic from MacBook to lab blocked. Firewall logs showed: "Block private networks from WAN block 192.168/16"

**Root Cause:** pfSense WAN default security setting blocks RFC1918 private IPs.

**Solution:** Disabled "Block private networks" on WAN interface.

**Lesson:** Security settings appropriate for internet-facing WAN must be adjusted for private upstream networks.

### 5.3 Pi-hole Network Configuration Challenges

**Problem:** Raspberry Pi refused to accept static IP through multiple methods (/etc/network/interfaces, /etc/dhcpcd.conf, NetworkManager).

**Root Cause:** Raspberry Pi OS has evolved through different network management systems. Multiple configuration files can conflict.

**Solution:** Used DHCP reservation in pfSense instead.

**Lesson:** DHCP reservations are often more reliable than OS-level static IPs for servers/services.

### 5.4 Raspberry Pi OS Selection

**Problem:** Initially flashed full Raspberry Pi OS with desktop environment.

**Impact:** Unnecessary disk space, boot time, and services.

**Solution:** Reflashed with Raspberry Pi OS Lite.

**Lesson:** Use minimal OS for headless servers.

═══════════════════════════════════════════════════════════════════

## 6. TESTING & VERIFICATION

### 6.1 Connectivity Tests (All Successful)

From MacBook on WiFi (192.168.8.160):

| Test | Command | Result |
|------|---------|--------|
| Ping pfSense | `ping 192.168.1.1` | ✅ Success |
| Ping Proxmox | `ping 192.168.1.10` | ✅ Success |
| Ping Pi-hole | `ping 192.168.1.20` | ✅ Success |
| pfSense Web | http://192.168.1.1 | ✅ Loads |
| Proxmox Web | https://192.168.1.10:8006 | ✅ Loads |
| Pi-hole Web | http://192.168.1.20/admin | ✅ Loads |

### 6.2 Routing Verification

```bash
ssh root@192.168.8.1
ip route show | grep 192.168.1
```

Output: `192.168.1.0/24 via 192.168.8.196 dev br-lan proto static` ✅

### 6.3 Network Architecture Verification

```bash
# MacBook IP (should be 192.168.8.x)
ifconfig | grep "inet 192"

# MacBook gateway (should be 192.168.8.1 - GL.iNet)
netstat -rn | grep default

# MacBook DNS (configured manually for now)
scutil --dns | grep nameserver
```

═══════════════════════════════════════════════════════════════════

## 7. PHASE 2 ATTEMPT & ANALYSIS (CRITICAL SECTION)

### 7.1 Phase 2 Goal

**Objective:** Eliminate double-router architecture by making pfSense the edge router.

**Desired Architecture:**
```
Internet (ISP WiFi)
        ↓
   GL.iNet (Bridge/AP Mode ONLY)
        ↓
   pfSense (MAIN ROUTER + FIREWALL)
   WAN: Gets IP from ISP
   LAN: 192.168.1.0/24
        ↓
   All Devices (single network)
```

**Expected Benefits:**
- Single network (no double NAT)
- Centralized management in pfSense
- Professional enterprise architecture
- Simplified DNS/DHCP configuration

### 7.2 Actions Taken During Phase 2 Attempt

**Step 1: Backup pfSense Configuration** ✅
- Created: pfsense-backup-phase1-2026-01-11.xml
- Stored safely for rollback

**Step 2: Configure GL.iNet**
- Attempted to change GL.iNet from Router mode to Access Point mode
- Changed GL.iNet LAN IP from 192.168.8.1 to 192.168.1.2
- Goal: Make GL.iNet a transparent bridge

**Step 3: Reconfigure pfSense WAN**
- Changed pfSense WAN from Static IP (192.168.8.196) to DHCP
- Set Outbound NAT to Automatic
- Renewed WAN DHCP lease

**Step 4: Testing**
- pfSense WAN successfully obtained... IPv6 address only
- No IPv4 gateway appeared
- NAT table remained empty
- IPv4 traffic (ping 8.8.8.8) failed

### 7.3 Critical Findings

**Finding 1: pfSense Never Received IPv4 Gateway**

Symptoms:
- Only WAN_DHCP6 (IPv6) gateway appeared in pfSense
- No WAN_DHCP (IPv4) gateway showed up
- NAT table empty (requires IPv4 gateway)
- IPv4 internet traffic failed

**Finding 2: This Was NOT a pfSense Misconfiguration**

Verified correct:
- ✅ Interfaces correctly assigned
- ✅ NAT mode set to automatic
- ✅ Firewall rules unchanged from working config
- ✅ WAN DHCP renewal successful
- ✅ No user error in configuration

**Finding 3: Root Cause - Physical/Hardware Limitation**

The problem is architectural, not configurational:

1. **GL.iNet connects to ISP via WiFi (Layer 3)**
   - Consumer WiFi devices cannot provide true Layer 2 bridging
   - WiFi client mode operates at Layer 3 (network layer)
   - This is a limitation of WiFi protocols and consumer hardware

2. **pfSense requires Layer 2 connectivity to ISP**
   - pfSense WAN needs to see upstream router's Layer 2 frames
   - This enables proper DHCP negotiation and gateway discovery
   - WiFi repeaters/bridges cannot provide this

3. **ISP Only Provided IPv6 to Downstream Router**
   - GL.iNet in bridge mode passed through IPv6 connectivity
   - IPv4 connectivity requires true Layer 2 bridging
   - Consumer WiFi bridges don't support this properly

4. **NAT Requires IPv4 Default Gateway**
   - pfSense cannot generate NAT rules without an IPv4 gateway
   - This is by design - NAT operates on Layer 3 IPv4
   - No gateway = no NAT = no internet for LAN devices

### 7.4 Why "Access Point" Mode Didn't Solve It

**Common Misconception:**
"Just buy a real Access Point and it will work"

**Reality:**
- True Access Points require **ethernet connection to pfSense LAN**
- Access Points do NOT connect to upstream WiFi
- Access Points provide WiFi to clients, not WAN connectivity
- Buying an AP does not eliminate the need for **wired WAN** to ISP

**What We Actually Need (But Don't Have):**
```
ISP Router → **ETHERNET CABLE** → pfSense WAN → pfSense LAN → AP
```

**What We Actually Have:**
```
ISP Router → **WIFI ONLY** → GL.iNet → pfSense
                 ↑
             Problem: Can't provide Layer 2 bridge
```

### 7.5 Phase 2 Decision: Abandoned by Design

**Status:** Phase 2 is **architecturally impossible** with current physical constraints.

**This is NOT:**
- ❌ User error
- ❌ Configuration mistake
- ❌ Missing knowledge
- ❌ Wrong approach

**This IS:**
- ✅ Physical/hardware limitation
- ✅ Correct diagnosis of problem
- ✅ Professional understanding of networking layers
- ✅ Appropriate decision to abandon infeasible approach

**Alternative Solutions Would Require:**
1. **Run ethernet cable to ISP** (not feasible - ISP is far away)
2. **Move pfSense to ISP location** (defeats purpose of homelab)
3. **Get wired ISP connection** (may not be available/affordable)
4. **Use cellular/fiber WAN** (expensive, may not be available)

**Conclusion:** Phase 1 architecture is the **correct solution** for WiFi-only ISP connectivity.

═══════════════════════════════════════════════════════════════════

## 8. WHY PHASE 1 IS CORRECT FOR WIFI-ONLY ISP

### 8.1 Phase 1 is NOT a Compromise

**Common Misunderstanding:**
"Phase 1 is a workaround until we can do Phase 2 properly."

**Reality:**
"Phase 1 is the architecturally appropriate solution for WiFi-only ISP connections."

### 8.2 Professional Network Design Principles

**Good Network Design:**
- Matches architecture to physical constraints
- Doesn't fight against hardware limitations
- Prioritizes reliability over "perfect" topology
- Makes appropriate tradeoffs

**Phase 1 Demonstrates:**
✅ Understanding of OSI layers
✅ Recognition of hardware limitations  
✅ Appropriate use of static routing
✅ Proper firewall segmentation
✅ Professional troubleshooting methodology

### 8.3 Double-Router Architecture is VALID

**When Double-Router is Appropriate:**
- Upstream connectivity is WiFi-only
- Need to segment networks with different security zones
- Transitioning between ISP providers
- Testing network configurations
- **Homelab learning environments** ← You are here

**Phase 1 Architecture Analysis:**

**Advantages:**
- ✅ Works reliably with WiFi-only ISP
- ✅ Provides network segmentation (home WiFi vs lab network)
- ✅ Enables wireless access to wired lab equipment
- ✅ pfSense protects lab with enterprise-grade firewall
- ✅ Can expand with VLANs on pfSense LAN side
- ✅ ISP router compromise doesn't affect lab directly

**Disadvantages:**
- Double NAT (minimal impact for most use cases)
- Two configuration points (GL.iNet + pfSense)
- Slightly more complex troubleshooting

**Trade-off Analysis:**
The disadvantages are minor compared to the impossibility of Phase 2. This is good engineering.

### 8.4 Career-Relevant Skills from Phase 1

**Skills Demonstrated:**
1. **Inter-network routing** (static routes between subnets)
2. **Firewall rule creation** (allowing specific traffic)
3. **Network troubleshooting** (systematic debugging)
4. **Understanding OSI layers** (recognizing Layer 2 vs Layer 3 constraints)
5. **Architecture analysis** (evaluating feasibility before implementation)
6. **Backup/restore procedures** (protecting against mistakes)
7. **Recognizing physical limitations** (knowing when to roll back)
8. **Professional decision-making** (abandoning infeasible approaches)

**Interview Talking Points:**
- "I built a homelab with static routing between networks"
- "I identified a Layer 2 bridging limitation that prevented a planned migration"
- "I validated architectural constraints before committing to changes"
- "I implemented backup procedures that enabled safe rollback"
- "I documented both successful implementation and failed attempts with root cause analysis"

This is EXACTLY what employers want to see.

═══════════════════════════════════════════════════════════════════

## 9. NEXT STEPS & FUTURE IMPROVEMENTS

### 9.1 Immediate (This Week)

**Configure Network-Wide Ad Blocking - PENDING**

Since Phase 1 is our permanent architecture, we still can't use pfSense DHCP to distribute Pi-hole DNS to all devices (MacBook gets DHCP from GL.iNet, not pfSense).

**Options:**

**Option A: Configure GL.iNet to Use Pi-hole DNS**
- Log into GL.iNet admin (http://192.168.8.1)
- Network → DNS
- Set DNS to: 192.168.1.20
- All WiFi devices get ad-blocking

**Option B: Manual DNS Configuration per Device**
- MacBook: System Settings → Network → DNS → Add 192.168.1.20
- Other devices: Configure individually
- More control, more manual work

**Recommendation:** Option A for network-wide blocking.

### 9.2 Short Term (This Month)

**1. Update pfSense**
- Current: 2.7.2
- Available: 2.8.1
- Backup configuration first!
- System → Update

**2. Change Pi-hole Admin Password**
```bash
ssh sabri@192.168.1.20
pihole -a -p
```

**3. Tighten pfSense Firewall Rules**
Replace broad "Any" rule with specific service rules:
- TCP 443 (pfSense HTTPS)
- TCP 8006 (Proxmox HTTPS)  
- TCP 80 (Pi-hole HTTP)
- UDP 53 (Pi-hole DNS)

**4. Document Network in GitHub**
- Create repository: homelab-documentation
- Include Phase 1 complete docs
- Include Phase 2 analysis
- Demonstrate professional documentation skills

### 9.3 Long Term (Next Few Months)

**1. VPN Access (WireGuard on pfSense)**
- Enable remote access to lab
- Access lab devices from anywhere
- Learn VPN configuration

**2. Network Monitoring**
- Enable detailed pfSense logging
- Set up Grafana dashboard
- Monitor Pi-hole statistics
- Alerting for anomalies

**3. Additional Services on Proxmox**
- Docker host VM
- Development environment
- Web server for projects
- Database server

**4. VLAN Segmentation on Lab Network**
Even with Phase 1 architecture, we can implement VLANs on pfSense LAN side:
- VLAN 10: Management (pfSense, Proxmox)
- VLAN 20: Services (Pi-hole, future services)
- VLAN 30: Testing/sandbox

**5. IDS/IPS (Suricata on pfSense)**
- Intrusion detection
- Threat monitoring
- Security event logging

### 9.4 Future Considerations (If Physical Setup Changes)

**If Ethernet to ISP Becomes Available:**
Then and only then would Phase 2 become feasible:
```
ISP Router → Ethernet → pfSense WAN (DHCP)
                           ↓
                     pfSense LAN → All Devices
```

At that point:
- Revisit Phase 2 migration plan
- GL.iNet becomes simple AP on pfSense LAN
- Single network architecture becomes possible

**Until then: Phase 1 is the correct architecture.**

═══════════════════════════════════════════════════════════════════

## 10. KEY LESSONS LEARNED

### 10.1 Technical Lessons

**Static Routing Fundamentals**
Static routes tell routers how to reach networks they're not directly connected to. Essential skill for network engineers.

**OSI Layer Understanding**
- Layer 2 (Data Link): Ethernet bridging, MAC addresses, switches
- Layer 3 (Network): IP routing, subnets, routers
- WiFi operates at Layer 3 in client mode (cannot provide true Layer 2 bridging)
- pfSense requires Layer 2 visibility to upstream for proper WAN function

**Firewall Context Matters**
Default security settings (like "Block Private Networks" on WAN) are correct for internet-facing deployments but must be adjusted for internal routing scenarios.

**DHCP Reservations vs Static IPs**
Centralized IP management through DHCP reservations is often more reliable than OS-level static IP configuration, especially across OS updates.

**Hardware Limitations are Real**
Not every problem is solvable with better configuration. Sometimes physics and hardware capabilities are the limiting factors.

### 10.2 Process Lessons

**Always Backup Before Major Changes**
The pfSense backup enabled clean rollback when Phase 2 proved infeasible. This saved hours of troubleshooting.

**Systematic Troubleshooting**
1. Verify physical layer (cables, power, lights)
2. Check network layer (IPs, routing, gateways)
3. Verify transport layer (firewall rules, NAT)
4. Test application layer (services, web interfaces)

**Know When to Roll Back**
Recognizing when an approach is fundamentally flawed (vs. just misconfigured) is a professional skill. We didn't waste days fighting physics.

**Document Failures, Not Just Successes**
The Phase 2 analysis is MORE valuable than Phase 1 success. It demonstrates:
- Root cause analysis
- Understanding of limitations
- Professional decision-making
- Learning from attempts

**Validate Assumptions Early**
We assumed GL.iNet could bridge WiFi for pfSense WAN. Testing revealed this assumption was wrong. Testing assumptions before full commitment saves time.

### 10.3 Career-Relevant Skills Developed

**Network Architecture & Design**
- ✅ Understanding of network segmentation
- ✅ Inter-network routing implementation
- ✅ Recognition of physical constraints
- ✅ Appropriate architecture selection

**Firewall Management**
- ✅ pfSense configuration and administration
- ✅ Firewall rule creation and testing
- ✅ Log analysis for troubleshooting
- ✅ Security policy implementation

**Systems Administration**
- ✅ Linux network configuration
- ✅ Service installation (Pi-hole, Proxmox)
- ✅ SSH administration
- ✅ DHCP/DNS management

**Problem Solving**
- ✅ Systematic troubleshooting methodology
- ✅ Log analysis and interpretation
- ✅ Root cause analysis
- ✅ Backup/restore procedures
- ✅ Rollback planning and execution

**Professional Skills**
- ✅ Technical documentation
- ✅ Architectural analysis
- ✅ Constraint identification
- ✅ Appropriate decision-making
- ✅ Learning from failures

### 10.4 Mindset Lessons

**Failure is Data**
The Phase 2 attempt was not a failure - it was successful validation of physical constraints. We learned exactly what won't work and why.

**Perfect is the Enemy of Good**
Phase 1 works reliably. Insisting on Phase 2 (which is impossible) would prevent using the lab at all.

**Professional Engineering is About Trade-offs**
We traded "single router elegance" for "actually working with WiFi ISP." This is good engineering.

**Documentation Matters**
Without documentation, Phase 2 would look like wasted time. With documentation, it's valuable learning and professional demonstration.

═══════════════════════════════════════════════════════════════════

## 11. APPENDIX

### 11.1 Quick Reference Commands

**Network Testing:**
```bash
ping <IP>                    # Test connectivity
ip addr show                 # View IP addresses
ip route show                # View routing table
netstat -rn | grep default   # Show default gateway
scutil --dns | grep nameserver  # Show DNS servers (macOS)
nc -zv <IP> <port>           # Test specific port
```

**GL.iNet Management:**
```bash
ssh root@192.168.8.1
ip route show | grep 192.168.1   # Verify static route
uci show network                 # Show all network config
/etc/init.d/network reload       # Apply network changes
```

**Proxmox:**
```bash
systemctl restart networking
systemctl restart pveproxy
systemctl restart pvedaemon
ip addr show                     # Verify IP configuration
```

**Pi-hole:**
```bash
ssh sabri@192.168.1.20
pihole status                    # Check Pi-hole status
pihole -up                       # Update Pi-hole
pihole -a -p                     # Change admin password
pihole restartdns                # Restart DNS service
pihole -c                        # Chronometer (live stats)
```

**pfSense (via web interface):**
- Status → System Logs → Firewall (troubleshooting)
- Status → Interfaces (verify WAN/LAN status)
- Status → DHCP Leases (see connected devices)
- Diagnostics → Backup & Restore (backup/restore config)
- Diagnostics → Command Prompt (run shell commands)

### 11.2 Access URLs

**Management Interfaces:**
- GL.iNet Admin: http://192.168.8.1
- pfSense Web GUI: http://192.168.1.1
- Proxmox Web GUI: https://192.168.1.10:8006
- Pi-hole Admin: http://192.168.1.20/admin

**SSH Access:**
- GL.iNet: `ssh root@192.168.8.1`
- pfSense: `ssh admin@192.168.1.1` (if enabled)
- Proxmox: `ssh root@192.168.1.10`
- Pi-hole: `ssh sabri@192.168.1.20`

### 11.3 Network Topology Diagram (Text Format)

```
PHYSICAL TOPOLOGY:
==================

Internet ←→ ISP WiFi Router
                 ↕ WiFi
          [GL.iNet Router]
             192.168.8.1
                 ↓ Ethernet (LAN port)
                 ↓
          [pfSense WAN Port]
             192.168.8.196
                 |
          [pfSense Firewall]
                 |
          [pfSense LAN Port]
             192.168.1.1
                 ↓ Ethernet
                 ↓
          [Netgear Switch]
             ↙          ↘
    [Proxmox M720q]  [Raspberry Pi]
     192.168.1.10     192.168.1.20
                       (Pi-hole)

MacBook WiFi ←→ GL.iNet WiFi (192.168.8.160)
                     ↓ Static Route
                     ↓ (via 192.168.8.196)
                     ↓
              Lab Network (192.168.1.0/24)


LOGICAL TOPOLOGY:
=================

Network A: 192.168.8.0/24 (GL.iNet)
   - Gateway: 192.168.8.1
   - DHCP: Enabled (managed by GL.iNet)
   - Devices: MacBook WiFi, pfSense WAN
   - Static Route: 192.168.1.0/24 via 192.168.8.196

Network B: 192.168.1.0/24 (Lab)
   - Gateway: 192.168.1.1 (pfSense LAN)
   - DHCP: Enabled (managed by pfSense)
   - Devices: Proxmox, Pi-hole
   - Firewall: pfSense (allows 192.168.8.0/24 → 192.168.1.0/24)
```

### 11.4 Backup Files

**Critical Backups to Maintain:**

1. **pfSense Configuration**
   - File: `pfsense-backup-phase1-YYYY-MM-DD.xml`
   - Location: Stored on MacBook + cloud backup
   - Purpose: Restore Phase 1 configuration if needed
   - How to restore: Diagnostics → Backup & Restore → Restore

2. **GL.iNet Configuration** (optional)
   - Can be backed up via GL.iNet admin interface
   - Less critical (easy to reconfigure)
   - Static route is the only custom config

3. **Documentation Files**
   - This document
   - Phase 2 migration plan (for reference)
   - Network diagrams
   - Credentials list (stored securely, not in docs)

### 11.5 Credentials Reference

**SECURITY NOTE:** Actual credentials should be stored in a password manager, not in documentation.

**Accounts that exist:**
- GL.iNet: root / [password]
- pfSense: admin / [password]
- Proxmox: root@pam / [password]
- Pi-hole (SSH): sabri / [password]
- Pi-hole (Web): Password is FCHzqpr0 (CHANGE THIS!)

### 11.6 Hardware Specifications

**pfSense Host:**
- Model: Lenovo ThinkCentre M910q
- CPU: Intel (specific model not recorded)
- RAM: (not recorded)
- Storage: (not recorded)
- NICs: 2x Ethernet (WAN + LAN)
- OS: pfSense 2.7.2 (FreeBSD-based)

**Proxmox Host:**
- Model: Lenovo ThinkCentre M720q
- CPU: (not recorded)
- RAM: (not recorded)
- Storage: (not recorded)
- NIC: 1x Ethernet
- OS: Proxmox VE

**Pi-hole Host:**
- Model: Raspberry Pi (model not specified, likely Pi 4)
- RAM: (not recorded)
- Storage: MicroSD card
- NIC: 1x Ethernet
- OS: Raspberry Pi OS Lite (64-bit)

**Network Equipment:**
- Router: GL.iNet GL-BE3600
- Switch: Netgear (model not recorded)
- Cabling: Standard Cat5e/Cat6 ethernet

### 11.7 Troubleshooting Quick Reference

**Problem: Can't access lab devices from MacBook WiFi**

Check:
1. MacBook IP: `ifconfig | grep "inet 192"` (should be 192.168.8.x)
2. Static route on GL.iNet: `ssh root@192.168.8.1` → `ip route show | grep 192.168.1`
3. pfSense WAN status: http://192.168.1.1 → Status → Interfaces
4. pfSense firewall rules: Firewall → Rules → WAN
5. pfSense firewall logs: Status → System Logs → Firewall

**Problem: Pi-hole not responding**

Check:
1. Physical: Is Pi powered on? Ethernet connected? Lights blinking?
2. Network: `ping 192.168.1.20` (from MacBook on ethernet)
3. DHCP: pfSense → Status → DHCP Leases (is Pi-hole listed?)
4. Service: `ssh sabri@192.168.1.20` → `pihole status`
5. Reboot: Power cycle the Raspberry Pi

**Problem: Proxmox not accessible**

Check:
1. Physical: Is M720q powered on?
2. Network: `ping 192.168.1.10` (from MacBook on ethernet)
3. Service: `ssh root@192.168.1.10` → check network config
4. Firewall: pfSense logs for blocks

**Problem: Lost static route on GL.iNet**

Symptoms: Could access lab before, now can't
Cause: GL.iNet reboot or firmware update
Solution: Re-add static route (see Section 2.1)

═══════════════════════════════════════════════════════════════════

## CONCLUSION

Phase 1 successfully established wireless access to the homelab environment through proper static routing, firewall configuration, and network segmentation. The subsequent Phase 2 attempt validated that this architecture is not a temporary workaround, but rather the **architecturally correct solution** for WiFi-only ISP connectivity.

### Final Architecture Assessment

**Phase 1 is OPTIMAL for current physical constraints:**
- ✅ Reliable wireless access to lab resources
- ✅ Enterprise-grade firewall protection for lab
- ✅ Network segmentation between home and lab
- ✅ Foundation for future expansion (VLANs, VPN, monitoring)
- ✅ Professional-quality implementation and documentation

**Key Achievements:**
1. **Functional homelab** with remote access capabilities
2. **Deep understanding** of routing, firewalling, and OSI layers
3. **Professional troubleshooting** methodology and backup procedures
4. **Architectural analysis** skills (evaluating feasibility vs. fighting constraints)
5. **Comprehensive documentation** suitable for portfolio/interview discussions
6. **Real-world learning** about physical networking constraints

### Value for Cybersecurity Career

This homelab demonstrates:
- Hands-on networking experience (routing, firewalling)
- Systems administration skills (Linux, pfSense, services)
- Problem-solving methodology (systematic troubleshooting)
- Professional decision-making (knowing when to roll back)
- Documentation skills (critical for security operations)
- Understanding of network architecture (essential for defensive security)

**Interview Talking Point:**
"I built a homelab with inter-network static routing to enable wireless access to wired lab equipment. During a migration attempt, I identified Layer 2 bridging constraints that made the planned architecture infeasible. I performed root cause analysis, documented findings, and made an informed decision to maintain the current architecture as it's optimal for the physical environment. This demonstrates understanding of OSI layers, recognition of hardware limitations, and professional engineering decision-making."

### Looking Forward

Phase 1 provides a solid foundation for:
- Network-wide ad blocking via Pi-hole
- VPN access for remote administration
- Additional services on Proxmox
- VLAN segmentation on lab network
- IDS/IPS implementation
- Network monitoring and logging
- Continued learning and expansion

**This is not the end - it's a strong beginning.**

═══════════════════════════════════════════════════════════════════

Document End
─────────────────────────────────────────────────────────────
Created: January 11, 2026
Updated: January 13, 2026 (Phase 2 analysis added)
Status: Phase 1 COMPLETE and VALIDATED as optimal architecture
