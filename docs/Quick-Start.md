# Quick Start Guide

Essential commands and access information for homelab infrastructure.

---

## 🌐 Access URLs

### Core Infrastructure

| Service | URL | Network | Purpose |
|---------|-----|---------|---------|
| GL.iNet Admin | http://192.168.8.1 | Upstream | Router configuration |
| pfSense Web GUI | http://192.168.1.1 | Management | Firewall management |
| Proxmox Web GUI | https://192.168.1.10:8006 | Management | VM management |
| Pi-hole Admin | http://192.168.1.20/admin | Management | DNS filtering |

### Security Monitoring (Phase 3)

| Service | URL | Network | Purpose |
|---------|-----|---------|---------|
| Security Onion | https://192.168.1.30 | Management | IDS/PCAP analysis |
| Wazuh Dashboard | https://192.168.1.108 | Management | SIEM/EDR platform |

### Lab VMs

| Service | IP Address | Network | Purpose |
|---------|-----------|---------|---------|
| Kali Linux | 192.168.1.102 (mgmt)<br>10.0.0.10 (attack) | Dual-NIC | Attack platform |
| Metasploitable | 10.0.0.20 | Attack | Vulnerable target |
| Windows 11 | 10.0.0.30 | Attack | Production endpoint |

**Note:** Management network (192.168.1.0/24) accessible via GL.iNet WiFi or ethernet. Attack network (10.0.0.0/24) isolated for lab testing.

---

## 🔐 SSH Access

### Core Infrastructure
```bash
# GL.iNet Router
ssh root@192.168.8.1

# pfSense (if SSH enabled)
ssh admin@192.168.1.1

# Proxmox
ssh root@192.168.1.10

# Pi-hole
ssh sabri@192.168.1.20
```

### Security Monitoring
```bash
# Security Onion
ssh sabri@192.168.1.30

# Wazuh Server
ssh sabri@192.168.1.108
```

### Lab VMs
```bash
# Kali Linux (management interface)
ssh kali@192.168.1.102

# Windows 11 (use RDP or console)
# No SSH by default
```

---

## 🔍 Network Testing

### From MacBook on WiFi
```bash
# Check your IP (should be 192.168.8.x)
ifconfig | grep "inet 192"

# Check gateway (should be 192.168.8.1)
netstat -rn | grep default

# Test connectivity to core infrastructure
ping 192.168.1.1    # pfSense
ping 192.168.1.10   # Proxmox
ping 192.168.1.20   # Pi-hole

# Test connectivity to security monitoring
ping 192.168.1.30   # Security Onion
ping 192.168.1.108  # Wazuh

# Test internet
ping 8.8.8.8
ping google.com
```

### Verify Static Route on GL.iNet
```bash
ssh root@192.168.8.1
ip route show | grep 192.168.1
```

Expected: `192.168.1.0/24 via 192.168.8.196 dev br-lan proto static`

---

## 🛡️ Phase 3 Quick Checks

### Security Onion Status
```bash
ssh sabri@192.168.1.30

# Check all services
sudo so-status

# Check traffic capture
sudo tcpdump -i enp6s19 -c 10

# View recent alerts
sudo so-elastic-query
```

### Wazuh Status
```bash
ssh sabri@192.168.1.108

# Check Wazuh Manager
sudo systemctl status wazuh-manager

# Check Wazuh Indexer
sudo systemctl status wazuh-indexer

# Check Wazuh Dashboard
sudo systemctl status wazuh-dashboard

# List connected agents
sudo /var/ossec/bin/agent_control -l
```

### OVS Bridge Status
```bash
ssh root@192.168.1.10

# Check OVS bridge
ovs-vsctl show

# Check port mirroring
ovs-vsctl list mirror

# Verify mirrored packets
ovs-vsctl list mirror | grep tx_packets
```

### Agent Status (from Kali)
```bash
# Check Wazuh agent status
sudo systemctl status wazuh-agent

# View agent logs
sudo tail -f /var/ossec/logs/ossec.log
```

---

## 🔧 Common Tasks

### Restart GL.iNet Network
```bash
ssh root@192.168.8.1
/etc/init.d/network reload
```

### Restart Proxmox Network
```bash
ssh root@192.168.1.10
systemctl restart networking
systemctl restart pveproxy
systemctl restart pvedaemon
```

### Pi-hole Management
```bash
ssh sabri@192.168.1.20

# Check status
pihole status

# Update Pi-hole
pihole -up

# Change admin password
pihole -a -p

# Restart DNS
pihole restartdns

# Live statistics
pihole -c
```

### Security Onion Management
```bash
ssh sabri@192.168.1.30

# Restart all services
sudo so-restart

# Update Security Onion
sudo soup

# Check disk usage (important!)
df -h
```

### Wazuh Management
```bash
ssh sabri@192.168.1.108

# Restart services
sudo systemctl restart wazuh-manager
sudo systemctl restart wazuh-indexer
sudo systemctl restart wazuh-dashboard

# Check disk usage
df -h /
```

### Recreate OVS Mirror (After Proxmox Reboot)
```bash
ssh root@192.168.1.10

# Get Security Onion port UUID
PORT_UUID=$(ovs-vsctl get Port tap105i1 _uuid)

# Create mirror
ovs-vsctl -- --id=@m create mirror name=span0 select-all=true \
  output-port=$PORT_UUID -- add bridge vmbr1 mirrors @m

# Verify
ovs-vsctl list mirror
```

### pfSense Backup

1. Access http://192.168.1.1
2. Navigate to: Diagnostics → Backup & Restore
3. Click "Download configuration as XML"
4. Save as: `pfsense-backup-YYYY-MM-DD.xml`

---

## 🎯 Attack Simulation Testing

### Basic Detection Test

**From Kali (10.0.0.10) to Metasploitable (10.0.0.20):**
```bash
# Run nmap scan
nmap -A -T4 10.0.0.20
```

**Check Detection:**

1. **Security Onion:** Look for Suricata alerts (ET SCAN Nmap)
2. **Wazuh Dashboard:** Check threat hunting for Kali events

### Test Windows 11 Detection

**From Kali to Windows (10.0.0.30):**
```bash
# Port scan Windows
nmap -A 10.0.0.30
```

**Check Detection:**

1. **Security Onion:** Look for NMAP OS Detection alerts
2. **Wazuh Dashboard:** Check Windows 11 agent events

---

## 🚨 Troubleshooting

### Can't Access Lab from WiFi

**Check static route:**
```bash
ssh root@192.168.8.1
ip route show | grep 192.168.1
```

**If missing, re-add:**
```bash
ip route add 192.168.1.0/24 via 192.168.8.196
uci add network route
uci set network.@route[-1].interface='lan'
uci set network.@route[-1].target='192.168.1.0'
uci set network.@route[-1].netmask='255.255.255.0'
uci set network.@route[-1].gateway='192.168.8.196'
uci commit network
/etc/init.d/network reload
```

### Security Onion Not Capturing Traffic

**Check monitoring interface:**
```bash
ssh sabri@192.168.1.30
ip link show enp6s19
# Should show "PROMISC" flag
```

**If not promiscuous:**
```bash
sudo ip link set enp6s19 promisc on
```

**Check OVS mirror on Proxmox:**
```bash
ssh root@192.168.1.10
ovs-vsctl list mirror
# Should show span0 with tx_packets incrementing
```

### Wazuh Dashboard Not Loading

**Check service status:**
```bash
ssh sabri@192.168.1.108
sudo systemctl status wazuh-dashboard
```

**If failed, check logs:**
```bash
sudo journalctl -u wazuh-dashboard -n 50
```

**Common issue - certificates:**
```bash
# Verify certificates exist
sudo ls -la /etc/wazuh-dashboard/certs/

# Should have dashboard.pem and dashboard-key.pem
# If missing, create symlinks (see Phase 3 docs)
```

### Wazuh Agent Not Connecting

**Check agent status:**
```bash
# On Kali
sudo systemctl status wazuh-agent

# Check logs
sudo tail -f /var/ossec/logs/ossec.log
```

**Restart agent:**
```bash
sudo systemctl restart wazuh-agent
```

**Verify manager IP in config:**
```bash
sudo grep '<address>' /var/ossec/etc/ossec.conf
# Should show: <address>192.168.1.108</address>
```

### Pi-hole Not Responding

1. Check if Pi is powered on
2. Check ethernet cable connected
3. Reboot Pi (unplug power, wait 10 seconds, plug back in)
4. Check DHCP lease in pfSense: Status → DHCP Leases

### Lost pfSense Access

**Connect via ethernet:**
1. Plug MacBook directly into switch
2. Get IP from pfSense DHCP (192.168.1.x)
3. Access http://192.168.1.1

**Or use console:**
1. Connect monitor/keyboard to pfSense M910q
2. Use pfSense console menu

---

## 📊 Network Information

### Network Topology
```
Internet → GL.iNet (192.168.8.0/24)
              ↓
         pfSense (192.168.1.1)
              ↓
    ┌─────────┴─────────┐
    ↓                   ↓
vmbr0 (Management)  vmbr1 (Attack)
192.168.1.0/24     10.0.0.0/24
```

### IP Addressing

| Network | Subnet | Gateway | DHCP Server | Purpose |
|---------|--------|---------|-------------|---------|
| Upstream | 192.168.8.0/24 | 192.168.8.1 | GL.iNet | ISP connection |
| Management | 192.168.1.0/24 | 192.168.1.1 | pfSense | Admin access |
| Attack Lab | 10.0.0.0/24 | None | None | Isolated testing |

### Static Assignments (Management Network)

| Device | IP | Assignment Method |
|--------|----|--------------------|
| GL.iNet | 192.168.8.1 | Static (device default) |
| pfSense WAN | 192.168.8.196 | Static (pfSense config) |
| pfSense LAN | 192.168.1.1 | Static (pfSense config) |
| Proxmox | 192.168.1.10 | Static (OS config) |
| Pi-hole | 192.168.1.20 | DHCP Reservation |
| Security Onion | 192.168.1.30 | Static (OS config) |
| Kali (mgmt) | 192.168.1.102 | DHCP |
| Wazuh Server | 192.168.1.108 | DHCP |

### Static Assignments (Attack Network)

| Device | IP | Assignment Method |
|--------|----|--------------------|
| Kali (attack) | 10.0.0.10 | Static (OS config) |
| Metasploitable | 10.0.0.20 | Static (OS config) |
| Windows 11 | 10.0.0.30 | Static (OS config) |

---

## 🔄 After Reboot Checklist

### GL.iNet Reboot
- [ ] Verify static route: `ip route show | grep 192.168.1`
- [ ] Test lab connectivity: `ping 192.168.1.1`

### pfSense Reboot
- [ ] Verify WAN interface up (192.168.8.196)
- [ ] Verify LAN interface up (192.168.1.1)
- [ ] Check firewall logs for issues

### Proxmox Reboot
- [ ] Verify IP: `ssh root@192.168.1.10 'ip addr'`
- [ ] Access web interface: https://192.168.1.10:8006
- [ ] **Recreate OVS mirror** (see Common Tasks above)
- [ ] Verify VMs auto-start

### Security Onion Reboot
- [ ] Verify gets IP 192.168.1.30
- [ ] Test: `ping 192.168.1.30`
- [ ] SSH and check services: `sudo so-status`
- [ ] Access web interface: https://192.168.1.30

### Wazuh Reboot
- [ ] Verify gets IP 192.168.1.108
- [ ] Test: `ping 192.168.1.108`
- [ ] Check services: All three should be running
- [ ] Access dashboard: https://192.168.1.108

### Pi-hole Reboot
- [ ] Verify gets IP 192.168.1.20
- [ ] Test: `ping 192.168.1.20`
- [ ] Access web interface: http://192.168.1.20/admin

### Kali Reboot
- [ ] Verify dual NICs configured
- [ ] Management: 192.168.1.102
- [ ] Attack: 10.0.0.10
- [ ] Check Wazuh agent: `sudo systemctl status wazuh-agent`

---

## 📖 Full Documentation

For complete implementation details, troubleshooting guide, and architectural analysis:
- **[Phase 1: Network Infrastructure](Phase1-Complete-Documentation.md)** - Static routing, pfSense, Pi-hole
- **[Phase 2: Migration Analysis](Phase2-Migration-Analysis.md)** - Why dual-router is optimal
- **[Phase 3: SOC Deployment](Phase3-SOC-Deployment.md)** - Security Onion, Wazuh, detection pipeline

---

## 💡 Quick Tips

### General

**Working wirelessly from MacBook:**
- Always verify you're on GL.iNet WiFi (192.168.8.x IP)
- Lab access requires static route on GL.iNet
- Use ethernet for most reliable access during troubleshooting

**pfSense firewall logs are your friend:**
- Status → System Logs → Firewall
- Shows blocked/allowed traffic in real-time
- Essential for troubleshooting connectivity issues

**Keep backups current:**
- Backup pfSense before any major changes
- Store backups in multiple locations
- Test restore procedure when system is working

### Phase 3 Specific

**OVS mirror doesn't persist:**
- Recreate after every Proxmox reboot
- See "Recreate OVS Mirror" in Common Tasks
- Takes 30 seconds

**Monitor disk space on SIEM platforms:**
- Security Onion: `df -h` (300GB allocated)
- Wazuh: `df -h /` (60GB allocated)
- Log retention fills up fast!

**Check agent connectivity regularly:**
- Wazuh Dashboard → Agents
- Should show "Active" status
- Disconnected agents don't send events

**Test detection pipeline monthly:**
- Run nmap scan from Kali
- Verify Security Onion alerts
- Verify Wazuh agent logs
- Ensures everything still works

**Document changes:**
- Note any configuration changes
- Keep a log of issues and resolutions
- Update this guide as environment evolves

---

## 🎯 Quick Reference Commands
```bash
# Check all core services
ping 192.168.1.1 && ping 192.168.1.10 && ping 192.168.1.20 && \
ping 192.168.1.30 && ping 192.168.1.108 && echo "All services reachable"

# Check OVS mirror status
ssh root@192.168.1.10 "ovs-vsctl list mirror"

# Check Wazuh agents
ssh sabri@192.168.1.108 "sudo /var/ossec/bin/agent_control -l"

# Quick attack test
ssh kali@192.168.1.102 "nmap -A -T4 10.0.0.20"
```

---

*Last Updated: January 27, 2026*  
*For detailed documentation, see Phase 1, 2, and 3 docs*
