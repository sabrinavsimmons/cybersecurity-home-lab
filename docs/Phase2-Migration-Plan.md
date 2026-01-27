# Phase 2: Architecture Migration Analysis

**Status:** ❌ Attempted - Architecturally Infeasible  
**Duration:** ~3 hours analysis and testing  
**Completion Date:** January 13, 2026

---

## 📋 Overview

Phase 2 was an attempt to migrate from the dual-router architecture (Phase 1) to a single-router configuration with pfSense as the edge router. Through systematic testing and analysis, this phase revealed that the dual-router architecture is not a temporary workaround but rather the **architecturally correct solution** for WiFi-only ISP connectivity.

**Key Finding:** Consumer WiFi devices operating in client mode cannot provide the Layer 2 bridging required for pfSense WAN connectivity. This is a fundamental hardware limitation, not a configuration issue.

---

## 🎯 Objectives & Outcomes

### Original Objectives
- ❌ Eliminate dual-router architecture
- ❌ Make pfSense the edge router
- ❌ Remove double NAT
- ✅ Understand architectural constraints
- ✅ Validate Phase 1 as optimal design

### Actual Outcomes
- **Architectural Analysis:** Identified Layer 2 bridging limitation
- **Root Cause Analysis:** WiFi client mode operates at Layer 3, not Layer 2
- **Professional Decision:** Abandoned infeasible approach rather than persist
- **Documentation:** Comprehensive analysis validates Phase 1 design

---

## 🏗️ Attempted Architecture

### Target Architecture (Phase 2 Goal)
```
Internet (ISP WiFi)
        ↓
   GL.iNet (Bridge Mode Only)
        ↓
   pfSense (Edge Router)
   WAN: DHCP from ISP
   LAN: 192.168.1.0/24
        ↓
   All Devices (Single Network)
```

### Expected Benefits (If Feasible)

**Network Improvements:**
- Single network (192.168.1.0/24) for all devices
- Centralized DHCP/DNS management
- No double NAT
- Simplified configuration

**Professional Architecture:**
- Enterprise-grade firewall for all traffic
- Easier VLAN implementation
- Better scalability
- Industry-standard design

---

## 🔧 Migration Attempt Process

### Step 1: Backup pfSense Configuration ✅

**Action:** Created complete configuration backup.

**File:** `pfsense-backup-phase1-2026-01-11.xml`

**Result:** Successful backup stored for rollback.

**Lesson:** Always backup before major changes. This enabled clean rollback when Phase 2 proved infeasible.

---

### Step 2: Configure GL.iNet Bridge Mode

**Objective:** Change GL.iNet from router mode to bridge/access point mode.

**Actions Taken:**
1. Accessed GL.iNet admin interface (http://192.168.8.1)
2. Changed LAN IP from 192.168.8.1 to 192.168.1.2
3. Disabled DHCP server on GL.iNet
4. Configured to pass traffic transparently

**Expected Behavior:**
- GL.iNet provides WiFi connectivity only
- All routing/DHCP handled by pfSense
- Layer 2 bridging between WiFi and pfSense WAN

**Actual Behavior:**
- GL.iNet successfully disabled routing functions
- WiFi remained operational
- However, Layer 2 bridging requirement not met (see findings)

---

### Step 3: Reconfigure pfSense WAN

**Objective:** Change pfSense WAN from static IP to DHCP to obtain IP from ISP.

**Actions Taken:**

**Via pfSense Web Interface:**
1. Navigated to Interfaces → WAN
2. Changed IPv4 Configuration Type from "Static" to "DHCP"
3. Removed static IP (192.168.8.196)
4. Set Outbound NAT to Automatic
5. Applied changes and renewed WAN DHCP lease

**Expected Behavior:**
- pfSense WAN obtains IPv4 address from ISP
- Default gateway appears (WAN_DHCP)
- NAT table populates
- Internet connectivity works

**Actual Behavior:**
- ✅ WAN interface came up
- ❌ Only IPv6 gateway appeared (WAN_DHCP6)
- ❌ No IPv4 gateway (WAN_DHCP)
- ❌ NAT table remained empty
- ❌ IPv4 internet connectivity failed

---

### Step 4: Troubleshooting & Analysis

**Systematic Troubleshooting Performed:**

**Verified pfSense Configuration:**
- ✅ Interfaces correctly assigned
- ✅ WAN set to DHCP mode
- ✅ NAT mode set to automatic
- ✅ Firewall rules unchanged from working config
- ✅ No user configuration errors

**Checked WAN Status:**
```
Status → Interfaces → WAN
- State: UP
- IPv6: Present (WAN_DHCP6 gateway)
- IPv4: Absent (no WAN_DHCP gateway)
- NAT table: Empty (requires IPv4 gateway)
```

**Attempted Solutions:**
1. Released and renewed DHCP lease multiple times
2. Rebooted pfSense
3. Rebooted GL.iNet
4. Checked physical cable connections
5. Verified GL.iNet bridge mode settings
6. Reviewed pfSense system logs

**Result:** No IPv4 connectivity regardless of troubleshooting attempts.

---

## 🔍 Critical Findings

### Finding 1: Layer 2 vs Layer 3 Limitation

**The Core Problem:**

**pfSense Requirement:**
- pfSense WAN interface requires **Layer 2** connectivity to upstream network
- Needs to see Layer 2 frames (Ethernet frames with MAC addresses)
- Required for DHCP negotiation and gateway discovery
- Essential for proper NAT operation

**WiFi Client Mode Reality:**
- Consumer WiFi devices in client mode operate at **Layer 3** (network layer)
- WiFi protocols do not provide true Layer 2 bridging
- Cannot pass Layer 2 frames required by pfSense
- This is a fundamental protocol limitation, not hardware deficiency

**Impact:**
- GL.iNet connects to ISP via WiFi → Layer 3 connection
- pfSense WAN needs Layer 2 → Requirement not met
- Result: pfSense cannot obtain IPv4 gateway from ISP

---

### Finding 2: Not a Configuration Error

**This Was Verified Correct:**

**pfSense Configuration:**
- ✅ Interface assignments correct
- ✅ DHCP mode properly set
- ✅ NAT configuration appropriate
- ✅ Firewall rules not blocking DHCP
- ✅ No syntax or parameter errors

**GL.iNet Configuration:**
- ✅ Bridge mode properly configured
- ✅ DHCP server disabled
- ✅ Routing disabled
- ✅ Acting as transparent pass-through

**Physical Layer:**
- ✅ Cables connected properly
- ✅ Link lights showing connectivity
- ✅ No physical damage or issues

**Conclusion:** The problem is **architectural**, not configurational.

---

### Finding 3: Why IPv6 Worked But IPv4 Didn't

**IPv6 Success:**
- IPv6 uses different address assignment mechanisms
- Stateless Address Autoconfiguration (SLAAC)
- Does not require traditional DHCP negotiation
- Can work over Layer 3 connections

**IPv4 Failure:**
- Requires traditional DHCP client-server exchange
- Needs Layer 2 visibility for proper operation
- ISP provides IPv4 via DHCP only
- Layer 3 bridging insufficient for IPv4 DHCP

**Result:** pfSense obtained IPv6 connectivity but not IPv4, rendering the network non-functional for most purposes.

---

### Finding 4: Physical Constraint Analysis

**Current Physical Setup:**
```
ISP Router (distant location)
        ↓ WiFi only (Layer 3)
   GL.iNet (client mode)
        ↓ Ethernet
   pfSense WAN (needs Layer 2)
```

**Why This Doesn't Work:**
- Gap in the chain: WiFi provides Layer 3, pfSense needs Layer 2
- No consumer WiFi device can bridge this gap
- Distance prevents wired ethernet to ISP

**What Would Be Required:**

**Option 1: Wired WAN to ISP**
```
ISP Router → Ethernet Cable → pfSense WAN
```
**Constraint:** ISP location distant from lab, cable run not feasible

**Option 2: Move pfSense to ISP Location**
```
ISP Router → pfSense (at ISP location) → VPN → Lab
```
**Constraint:** Defeats purpose of homelab, no physical access to equipment

**Option 3: Different ISP Connection**
```
Fiber/Cable Modem → pfSense WAN
```
**Constraint:** May not be available, expensive, requires installation

**Conclusion:** None of these options are currently feasible.

---

## 💡 Key Technical Learnings

### 1. OSI Layer Limitations

**Layer 2 (Data Link Layer):**
- Deals with MAC addresses and Ethernet frames
- Switches and bridges operate here
- Required for proper DHCP negotiation
- True bridging requires Layer 2 visibility

**Layer 3 (Network Layer):**
- Deals with IP addresses and routing
- Routers operate here
- WiFi client mode operates at this layer
- Cannot provide true Layer 2 bridging

**Application:** Understanding which layer a device operates at is critical for network design. Attempting to use a Layer 3 device where Layer 2 is required will fail regardless of configuration.

---

### 2. WiFi Bridging Myths

**Common Misconception:**
"WiFi repeaters and bridges can connect any network wirelessly."

**Reality:**
- **WiFi Repeater:** Extends WiFi network (same SSID, Layer 2-ish)
- **WiFi Bridge (Consumer):** Converts WiFi to Ethernet (Layer 3)
- **True Bridge:** Requires Layer 2 connectivity both sides

**Why Consumer WiFi Bridges Fail Here:**
- Operate in "client mode" + "AP mode" simultaneously
- Layer 3 translation occurs
- Cannot pass raw Layer 2 frames
- Insufficient for pfSense WAN requirements

**What Would Work:**
- Enterprise WiFi bridge with WDS (Wireless Distribution System)
- Even then, not guaranteed for DHCP
- Expensive, complex, may still not solve issue

**Lesson:** WiFi "bridging" is not true bridging in most consumer scenarios.

---

### 3. DHCP Requirements

**Traditional DHCP Process:**

1. **DISCOVER** (Client broadcasts: "I need an IP")
2. **OFFER** (Server responds: "Here's an available IP")
3. **REQUEST** (Client broadcasts: "I accept that IP")
4. **ACKNOWLEDGE** (Server confirms: "IP assigned")

**Why This Requires Layer 2:**
- Steps 1 and 3 are **broadcasts** (no IP yet)
- Broadcasts only work on Layer 2
- Layer 3 routing drops broadcasts
- DHCP relay agents can help but not in this scenario

**Result:** pfSense cannot complete DHCP negotiation over Layer 3 WiFi connection.

---

### 4. Architecture vs. Configuration

**Key Professional Skill:** Recognizing when a problem is:

**Configuration Issue:**
- Wrong settings in software
- Solvable by changing parameters
- Fixed by reading documentation
- Example: Firewall rule blocking traffic

**Architecture Issue:**
- Fundamental design constraint
- Not solvable by configuration changes
- Requires different approach or hardware
- Example: WiFi not providing Layer 2

**Application:** Knowing the difference prevents wasted time fighting impossible problems. Attempting to "configure around" an architectural limitation is futile.

---

### 5. When to Roll Back

**Professional Decision-Making:**

**Indicators to Persist:**
- Problem is clearly configurational
- Documentation suggests solution exists
- Similar setups work for others
- Small changes might resolve issue

**Indicators to Roll Back:**
- Problem is architectural
- No amount of configuration will solve it
- Physical limitations exist
- Time investment not justified by potential gain

**Phase 2 Decision:** After systematic analysis, the problem was identified as architectural. Continuing would waste time. Rolling back was the professional decision.

---

## 🎓 Skills Demonstrated

### Technical Skills
- Understanding of OSI model layers
- DHCP protocol knowledge
- Network architecture design
- WiFi technology limitations
- Systematic troubleshooting methodology

### Professional Skills
- Backup/restore procedures
- Root cause analysis
- Recognizing architectural constraints
- Knowing when to abandon infeasible approaches
- Professional decision-making under uncertainty
- Comprehensive technical documentation

### Career-Relevant Competencies
- Problem diagnosis and analysis
- Understanding trade-offs in design
- Communicating technical constraints
- Documenting failures as learning opportunities

---

## 📊 Phase 1 vs Phase 2 Comparison

### Phase 1 (Current - Optimal for Environment)

**Architecture:**
```
ISP WiFi → GL.iNet Router → pfSense → Lab Devices
Two networks: 192.168.8.0/24 and 192.168.1.0/24
```

**Advantages:**
- ✅ Works reliably with WiFi-only ISP
- ✅ Provides network segmentation
- ✅ pfSense protects lab network
- ✅ Can expand with VLANs on LAN side
- ✅ Proven and stable

**Disadvantages:**
- Double NAT (minor performance impact)
- Two DHCP servers (manageable)
- Slightly more complex routing

**Verdict:** Appropriate solution for physical constraints.

---

### Phase 2 (Attempted - Infeasible)

**Architecture:**
```
ISP WiFi → GL.iNet Bridge → pfSense (Edge) → All Devices
Single network: 192.168.1.0/24
```

**Theoretical Advantages:**
- Single network
- No double NAT
- Centralized management
- Professional architecture

**Actual Result:**
- ❌ Cannot obtain IPv4 gateway
- ❌ Layer 2 bridging unavailable
- ❌ Architecturally impossible with WiFi WAN
- ❌ Not solvable by configuration

**Verdict:** Infeasible with WiFi-only ISP connection.

---

## 🚀 Path Forward

### Phase 1 is the Correct Architecture

**Professional Network Design Principles:**

**Good Design:**
- Matches architecture to physical constraints
- Doesn't fight against hardware limitations
- Prioritizes reliability over "perfect" topology
- Makes appropriate trade-offs

**Phase 1 Demonstrates:**
- Understanding of OSI layers
- Recognition of hardware limitations
- Appropriate use of static routing
- Proper firewall segmentation
- Professional troubleshooting methodology

---

### When Phase 2 Becomes Feasible

**If Physical Setup Changes:**

**Wired Ethernet to ISP Becomes Available:**
```
ISP Router → Ethernet Cable → pfSense WAN → All Devices
```

**Then Phase 2 would be feasible:**
- pfSense WAN gets Layer 2 connectivity
- Can obtain IPv4 via DHCP properly
- Single-router architecture becomes possible
- GL.iNet becomes simple AP on LAN side

**Until Then:** Phase 1 dual-router architecture is optimal.

---

### Future Enhancements (On Phase 1)

**Even with dual-router architecture, we can still implement:**

**1. VLAN Segmentation (pfSense LAN Side)**
- VLAN 10: Management (pfSense, Proxmox)
- VLAN 20: Services (Pi-hole, future)
- VLAN 30: Testing/sandbox

**2. VPN Access (WireGuard on pfSense)**
- Remote access to lab
- Secure external connectivity

**3. IDS/IPS (Suricata on pfSense)**
- Intrusion detection
- Threat monitoring
- Security event logging

**4. Advanced Monitoring**
- pfSense logging
- Grafana dashboards
- Pi-hole analytics

**Phase 1 does not limit these capabilities.**

---

## 📚 Resources & References

**Technical Documentation:**
- [OSI Model - Layers Explained](https://en.wikipedia.org/wiki/OSI_model)
- [DHCP Protocol RFC 2131](https://tools.ietf.org/html/rfc2131)
- [WiFi Bridging Technical Overview](https://www.wi-fi.org/discover-wi-fi/wi-fi-direct)
- [pfSense WAN Requirements](https://docs.netgate.com/pfsense/)

**Related Learning:**
- Layer 2 vs Layer 3 networking concepts
- DHCP relay agents and operation
- WiFi standards and limitations
- Network architecture best practices

---

## 📊 Project Statistics

**Time Investment:** ~3 hours
- Planning and preparation: 1 hour
- Migration attempt: 1 hour
- Troubleshooting and analysis: 1 hour

**Attempts:**
- Configuration changes: 5+
- DHCP renewals: 10+
- Reboots: 3 (pfSense, GL.iNet)
- Documentation reviews: Multiple

**Outcome:** Validated architectural constraint, professional rollback executed.

---

## ✅ Success Criteria (Redefined)

**Original Criteria (Not Met):**
- ❌ Single-router architecture
- ❌ No double NAT
- ❌ pfSense as edge router

**Actual Success Criteria (Met):**
- ✅ Identified architectural constraint (Layer 2 limitation)
- ✅ Performed root cause analysis
- ✅ Executed clean rollback to Phase 1
- ✅ Validated Phase 1 as optimal design
- ✅ Documented findings comprehensively
- ✅ Demonstrated professional decision-making

**Redefined Success:** Understanding why something won't work is as valuable as making something work.

---

## 🎓 Conclusion

Phase 2 was not a failure - it was a successful validation of architectural constraints. The attempt revealed that:

1. **Phase 1 dual-router architecture is not a workaround** - it's the appropriate solution for WiFi-only ISP connectivity

2. **Consumer WiFi devices have fundamental limitations** - they cannot provide the Layer 2 bridging required for pfSense WAN operation

3. **Professional engineering involves recognizing constraints** - not every problem is solvable with better configuration; some require different approaches or acceptance of limitations

4. **Documentation of "failures" demonstrates maturity** - showing why something won't work is valuable professional skill

**Career Relevance:**

This analysis demonstrates critical skills for cybersecurity and network engineering roles:
- Root cause analysis methodology
- Understanding of networking fundamentals (OSI layers)
- Recognition of architectural vs. configurational issues
- Professional decision-making (rollback vs. persist)
- Clear technical communication
- Learning from attempts, not just successes

**The homelab remains fully functional with Phase 1 architecture, validated as the optimal design for the current environment.**

---

*Last Updated: January 13, 2026*  
*Author: Sabrina Simmons*  
*Repository: [github.com/sabrinavsimmons/cybersecurity-home-lab](https://github.com/sabrinavsimmons/cybersecurity-home-lab)*
