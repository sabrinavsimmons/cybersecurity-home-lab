# Homelab Infrastructure Documentation

Personal homelab environment for learning defensive cybersecurity, network architecture, and systems administration.

[![Status](https://img.shields.io/badge/Status-Phase%203%20Complete-success)]()
[![Architecture](https://img.shields.io/badge/Architecture-SOC%20Homelab-blue)]()
[![License](https://img.shields.io/badge/License-MIT-green)]()

---

## 📋 Project Overview

This repository documents the design, implementation, and evolution of my homelab infrastructure. The lab demonstrates practical networking, security, and systems administration skills through hands-on implementation.

**Current State:** Fully operational **Security Operations Center (SOC)** with network monitoring, endpoint detection, and attack simulation capabilities.

**Key Learning Areas:**
- Security Operations Center (SOC) deployment and operations
- SIEM platform implementation (Wazuh)
- Network intrusion detection (Security Onion, Suricata, Zeek)
- Endpoint detection and response (EDR agents)
- Software-defined networking (Open vSwitch)
- Inter-network routing and firewall configuration
- Professional troubleshooting and capacity planning

---

## 🏗️ Current Architecture (Phase 3)
```
                    Internet (ISP WiFi)
                            ↓
                      GL.iNet Router
                      192.168.8.0/24
                            ↓ Static Route
                      pfSense Firewall
                      192.168.1.1
                            ↓
            ┌───────────────┴───────────────┐
            ↓                               ↓
        vmbr0 (Management)           vmbr1 (Monitored)
        192.168.1.0/24              10.0.0.0/24
            ↓                               ↓
    ┌───────┴────┬──────┬────────┐    ┌────┴────┬──────────┐
    │            │      │        │    │         │          │
Proxmox      Wazuh  Security    │   Kali   Metasploitable Windows
.1.10        .1.108  Onion      │  .0.10      .0.20       .0.30
                     .1.30       │
                     (mgmt)      │
                                 │
                          OVS Bridge (span0)
                          Traffic Mirroring
                                 ↓
                          Security Onion
                          (monitoring port)
```

**Status:** ✅ Fully operational SOC with validated detection capabilities

---

## 📸 Lab Setup

![Homelab Setup](images/lab-setup.jpg)

**Physical Hardware:**
- **Firewall:** Lenovo ThinkCentre M910q (pfSense)
- **Hypervisor:** Lenovo ThinkCentre M720q (Proxmox VE - 32GB RAM, 256GB NVMe + 931GB SSD)
- **Router:** GL.iNet GL-BE3600
- **Switch:** Netgear 8-port
- **DNS/Ad-Block:** Raspberry Pi 5 (Pi-hole, mounted behind rack) 
- **Rack:** Custom rack mount with cable management

**Virtual Machines:**
- **Security Onion** - Network monitoring (Suricata, Zeek, Kibana)
- **Wazuh Server** - SIEM and endpoint agent management
- **Kali Linux** - Attack platform with Wazuh agent
- **Metasploitable 2** - Vulnerable target for testing
- **Windows 11** - Production endpoint with Wazuh agent

---

## 📚 Documentation

### Phase Documentation
- **[Phase 1 Complete Documentation](docs/Phase1-Complete-Documentation.md)** - Network infrastructure, routing, and firewall implementation
- **[Phase 2 Migration Analysis](docs/Phase2-Migration-Plan.md)** - Attempted migration and architectural constraint discovery
- **[Phase 3 SOC Deployment](docs/Phase3-SOC-Deployment.md)** - Security Operations Center implementation with full detection pipeline
- **[Phase 3 Enhancements](docs/Phase3-Enhancements.md)** - ⭐ **NEW** Automation, troubleshooting, and detection validation

### Additional Resources
- **[Quick Start Guide](docs/Quick-Start.md)** - Essential commands and access information

---

## 💻 Infrastructure Components

### Physical Hardware

| Device | Model | Role | Specifications |
|--------|-------|------|----------------|
| Router | GL.iNet GL-BE3600 | ISP connectivity + WiFi | 192.168.8.1 |
| Firewall | Lenovo M910q | pfSense firewall/router | 192.168.1.1 |
| Hypervisor | Lenovo M720q | Proxmox VE virtualization | 32GB RAM, 1.2TB storage |
| DNS/Ad-Block | Raspberry Pi 5 | Pi-hole DNS filtering | 192.168.1.20 |
| Switch | Netgear 12-port | Network connectivity | Managed switch |

### Virtual Machines

| VM | vCPUs | RAM | Disk | Role | IP Addresses |
|----|-------|-----|------|------|--------------|
| Security Onion | 4 | 16 GB | 300 GB | IDS/IPS, PCAP | 192.168.1.30 + monitoring |
| Wazuh Server | 2 | 4 GB | 60 GB | SIEM, agent mgmt | 192.168.1.108 |
| Kali Linux | 2 | 4 GB | 32 GB | Attack platform | 192.168.1.102, 10.0.0.10 |
| Metasploitable | 1 | 512 MB | 8 GB | Vulnerable target | 10.0.0.20 |
| Windows 11 | 2 | 4 GB | 64 GB | Production endpoint | 10.0.0.30 |

**Total Allocated:** 11 vCPUs, 28.5 GB RAM, 464 GB disk

---

---

## 🎯 Key Accomplishments

### Phase 1: Network Infrastructure
- ✅ Static routing between separate networks (192.168.8.0/24 ↔ 192.168.1.0/24)
- ✅ pfSense firewall with custom rules for inter-network communication
- ✅ Wireless access to wired lab infrastructure
- ✅ Pi-hole DNS filtering deployment
- ✅ Proxmox virtualization platform setup

### Phase 2: Architectural Analysis
- ✅ Identified Layer 2 bridging constraints with WiFi-only ISP
- ✅ Root cause analysis of architectural limitations
- ✅ Validated Phase 1 as optimal design for physical environment
- ✅ Demonstrated professional decision-making (rollback vs. persistence)

### Phase 3: SOC Deployment ⭐
- ✅ **Security Onion deployment** with Suricata IDS and Zeek network analysis
- ✅ **Wazuh SIEM platform** with full agent management infrastructure
- ✅ **Open vSwitch bridge** for software-based traffic mirroring
- ✅ **Multi-OS endpoint monitoring** (Linux and Windows agents)
- ✅ **Attack simulation and detection validation** with confirmed visibility
- ✅ **Storage capacity planning** and enterprise troubleshooting
- ✅ **Defense-in-depth architecture** with network and endpoint layers
- ✅ **OVS mirror automation** via systemd service (auto-creates on boot)
- ✅ **Network-wide ad blocking** via pfSense → Pi-hole forwarding chain
- ✅ **Host-based command monitoring** with auditd integration
- ✅ **Proxmox stability fixes** (boot service troubleshooting)

---

## 🔧 Technologies & Skills

### Security Operations
- **SIEM:** Wazuh deployment, agent management, event correlation
- **IDS/IPS:** Suricata signature-based detection, Zeek protocol analysis
- **EDR:** Endpoint agent deployment (Linux/Windows)
- **Threat Detection:** Attack simulation and validation
- **Incident Response:** Log analysis and forensics

### Networking
- **Software-Defined Networking:** Open vSwitch configuration
- **Traffic Analysis:** Port mirroring, packet capture (tcpdump)
- **Network Segmentation:** Isolated attack network (10.0.0.0/24)
- **Static Routing:** Inter-network communication
- **Firewall Management:** pfSense rule creation and troubleshooting

### Systems Administration
- **Linux:** Ubuntu Server, Debian-based systems, systemd
- **Windows:** Windows 11, PowerShell, firewall configuration
- **Virtualization:** Proxmox VE, VM management, storage pools
- **Certificate Management:** SSL/TLS certificates for SIEM services
- **Capacity Planning:** Storage management for SIEM log retention

### Professional Practices
- **Technical Documentation:** Comprehensive project documentation (150+ pages)
- **Troubleshooting:** Root cause analysis, systematic debugging
- **Disaster Recovery:** Backup strategies, knowing when to rebuild
- **Capacity Planning:** Storage requirements for enterprise tools
- **Decision Making:** Evaluating trade-offs (software vs. hardware solutions)

---

---

## 📖 Learning Highlights

### Phase 3 Technical Discoveries

**Software-Based Traffic Mirroring:**
Physical switch port mirroring failed when attempting to mirror all VM traffic. Migrated to Open vSwitch software bridge in Proxmox, providing unlimited capacity and flexibility. Lesson: Software-defined networking can overcome hardware limitations.

**SIEM Storage Requirements:**
Initial Wazuh deployment exhausted disk space during system upgrade. Learned to calculate proper capacity for SIEM platforms: `(Base OS + Software) × 1.4 + (Daily Logs × Retention Days)`. Redeployed with 60GB on separate storage pool.

**Defense-in-Depth Detection:**
Network monitoring (Security Onion) and endpoint monitoring (Wazuh) provide complementary visibility. Network layer detects attacks in transit; endpoint layer monitors system changes. Both layers required for comprehensive security.

**Certificate Management:**
System upgrade replaced Wazuh dashboard config but deleted SSL certificates, causing service failures. Lesson: Always preserve installation backup files and understand certificate dependencies in enterprise tools.

### Phase 1-2 Discoveries

**WiFi Layer 2 Bridging Limitation:**
Consumer WiFi devices in client mode cannot provide true Layer 2 bridging required for pfSense WAN connectivity. Validated dual-router architecture as correct solution for WiFi-only ISP environments.

**Static Routing Implementation:**
Successfully implemented persistent static routes using OpenWRT's UCI system, enabling seamless wireless access without VPN overhead.

### Process Learnings

- **Backup before changes** - Critical for clean rollbacks
- **Validate assumptions early** - Test requirements before full implementation
- **Document failures and successes** - Both demonstrate professional growth
- **Recognize when to rebuild** - Sometimes starting fresh is faster than repair
- **Understand underlying architecture** - Abstractions break; know what's underneath
- **Monitor capacity proactively** - Enterprise tools have substantial resource requirements

---

## 🚀 Future Enhancements

### Completed ✅
- [x] OVS mirror automation (systemd service)
- [x] Network-wide ad blocking (pfSense + Pi-hole)
- [x] Detection pipeline validation with attack simulations
- [x] Auditd configuration for command monitoring

### Immediate (Next Phase)
- [ ] pfSense log forwarding to Security Onion
- [ ] Pi-hole DNS logging integration with SIEM
- [ ] Custom Suricata rules for lab-specific threats
- [ ] Automated attack scenarios for continuous testing
- [ ] Wazuh vulnerability detection size limits
- [ ] Custom Wazuh decoders for audit logs

### Short-term
- [ ] Add Splunk or ELK stack for SIEM comparison
- [ ] YARA rules for malware detection
- [ ] File integrity monitoring baselines
- [ ] Custom Wazuh correlation rules
- [ ] Network behavior anomaly detection (Zeek scripts)

### Long-term
- [ ] Threat intelligence feed integration
- [ ] Automated incident response playbooks
- [ ] Red team vs. blue team scenarios
- [ ] Compliance reporting (CIS benchmarks)
- [ ] Machine learning-based anomaly detection

---

## 🎓 Career Relevance

This homelab demonstrates skills directly applicable to:

**Security Operations Center (SOC) Analyst:**
- SIEM platform operation (Wazuh)
- IDS/IPS monitoring (Security Onion, Suricata)
- Log analysis and event correlation
- Incident detection and validation
- Multi-platform security monitoring

**Security Engineer:**
- Defense-in-depth architecture design
- Network segmentation for security
- EDR agent deployment and management
- Security tool integration and troubleshooting
- Capacity planning for security infrastructure

**Network Security Engineer:**
- Traffic analysis and packet capture
- Software-defined networking (OVS)
- Intrusion detection system configuration
- Network monitoring and visibility

**Systems Administrator:**
- Linux and Windows administration
- Certificate management
- Storage capacity planning
- Service troubleshooting and disaster recovery

---

## 💼 Interview Talking Points

### Technical Project Discussion

**"Tell me about a complex technical project you've worked on."**

*"I built a fully operational Security Operations Center in my homelab with enterprise-grade monitoring tools. This included deploying Security Onion for network intrusion detection using Suricata and Zeek, implementing Wazuh as a SIEM platform with endpoint agents on Linux and Windows systems, and establishing software-based traffic mirroring using Open vSwitch when physical switch capabilities proved insufficient.*

*The project required troubleshooting complex multi-system integration issues, including certificate management after system upgrades, storage capacity planning when disk space exhausted during deployment, and network connectivity problems between heterogeneous operating systems.*

*I validated the entire detection pipeline by simulating attacks and confirming that both network-layer IDS and endpoint agents properly detected and logged malicious activity, demonstrating defense-in-depth principles with complementary security controls."*

### Troubleshooting & Problem Solving

**"Describe a time you had to troubleshoot a difficult technical problem."**

*"During Wazuh SIEM deployment, a system upgrade filled the disk completely, causing the VM to crash mid-upgrade. I performed root cause analysis and discovered Proxmox had two storage pools - the NVMe pool hosting the VM was near capacity, while a 931GB SSD pool had ample space.*

*Rather than attempting to repair the corrupted system, I made the professional decision to rebuild cleanly on the appropriate storage pool. This involved understanding the hypervisor's storage architecture, calculating proper capacity requirements for SIEM platforms (60GB minimum for base installation plus log retention), and implementing monitoring to prevent recurrence.*

*The rebuilt system has been stable for ongoing operations. This demonstrated both technical troubleshooting skills and professional judgment about when to rebuild versus repair."*

### Continuous Learning

**"How do you stay current with cybersecurity trends?"**

*"I learn through hands-on implementation in my homelab SOC environment. I deploy production-grade security tools like Security Onion and Wazuh, simulate attacks using Kali Linux, and validate detection capabilities across network and endpoint layers.*

*I maintain comprehensive technical documentation (150+ pages), which serves both as a learning tool and professional deliverable. When I encounter obstacles - like incompatible hardware or exhausted storage - I use structured troubleshooting: verify configurations, check logs, test incrementally, and research systematically.*

*This practical approach means I'm not just reading about security concepts - I'm actively implementing, breaking, and fixing enterprise security infrastructure."*

---

---

## 📝 Architecture Decision Records

### Why Software-Based Traffic Mirroring?

**Context:** Need to monitor all attack traffic between VMs for Security Onion analysis.

**Decision:** Implement Open vSwitch (OVS) bridge with port mirroring instead of physical switch mirroring.

**Rationale:**
- Physical switch (Netgear GS308E) failed when mirroring all VM ports
- OVS provides unlimited mirroring capacity without hardware constraints
- Software-based approach more flexible for virtual infrastructure
- No additional hardware cost

**Trade-offs:**
- Mirror configuration doesn't persist across Proxmox reboots (acceptable for homelab)
- Slight CPU overhead on hypervisor (negligible with modern hardware)

**Status:** Validated and operational. See [Phase 3 Documentation](docs/Phase3-SOC-Deployment.md#2-ovs-bridge-traffic-mirroring) for implementation details.

---

### Why Dual-Router Architecture?

**Context:** ISP connection is WiFi-only, physically distant from lab location.

**Decision:** Maintain dual-router architecture (GL.iNet + pfSense) rather than single-router design.

**Rationale:**
- WiFi client devices cannot provide Layer 2 bridging required for pfSense WAN
- Running ethernet cable to ISP not feasible (distance, permissions)
- Dual-router provides network segmentation and firewall protection
- Appropriate trade-off: slight complexity vs. physical impossibility

**Alternatives Considered:**
- pfSense as edge router (requires wired WAN - not available)
- VPN to lab (adds latency, doesn't solve management access)
- Move equipment to ISP location (defeats purpose of dedicated lab space)

**Status:** Validated as optimal solution. See [Phase 2 Analysis](docs/Phase1-Complete-Documentation.md#7-phase-2-attempt--analysis-critical-section) for details.

---

## 🤝 Contributing

This is a personal learning project, but feedback and suggestions are welcome! Feel free to:
- Open an issue for questions or discussion
- Share your own homelab experiences
- Suggest improvements to documentation

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

Documentation and configuration examples are provided as-is for educational purposes.

---

## 🔗 Connect

- **GitHub:** https://github.com/sabrinavsimmons
- **LinkedIn:** https://www.linkedin.com/in/sabrina-simmons-830095a1


---

---

## 📊 Project Timeline

- **January 11, 2026** - Phase 1: Network infrastructure implementation (~8 hours)
- **January 13, 2026** - Phase 2: Migration attempt and analysis (~3 hours)
- **January 22-26, 2026** - Phase 3: SOC deployment and validation (~15 hours)
- **January 30, 2026** - Phase 3 Enhancements: Automation and validation (~2 hours)

**Total Investment:** ~28 hours of hands-on implementation and learning

**Documentation:** 120+ pages of comprehensive technical documentation

---

## 🙏 Acknowledgments

### Phase 1-2
- pfSense community for excellent documentation
- OpenWRT project for flexible router firmware
- Pi-hole team for DNS-based ad blocking solution

### Phase 3
- Security Onion team for enterprise security monitoring platform
- Wazuh community for SIEM and EDR tools
- Open vSwitch project for software-defined networking
- Various homelab and cybersecurity communities

---

## 🏆 Project Highlights

### What Makes This Special

**Real-World Constraints:** Not a tutorial follow-along - encountered and solved genuine problems like exhausted disk space, incompatible hardware, and certificate management issues.

**Professional Troubleshooting:** Demonstrated when to rebuild vs. repair, root cause analysis methodology, and capacity planning for enterprise tools.

**Defense-in-Depth:** Implemented layered security with network monitoring (Security Onion) and endpoint detection (Wazuh agents) providing complementary visibility.

**Comprehensive Documentation:** 120+ pages of technical documentation covering implementation, troubleshooting, lessons learned, and career relevance.

**Production-Grade Tools:** Not toy projects - deployed enterprise security platforms (Security Onion, Wazuh) used in real SOC environments.

**Validated Capabilities:** Attack simulations confirmed full detection pipeline operational across network and endpoint layers.

---

**Built with curiosity, documented with care, shared for learning.**

*This homelab represents continuous learning and professional development in defensive cybersecurity - because the best way to learn security is to build it yourself.*
