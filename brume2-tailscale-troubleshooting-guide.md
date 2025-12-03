# Brume 2 / Tailscale Troubleshooting Guide

This guide covers diagnosing and fixing issues with GL.iNet Brume 2 routers, Tailscale VPN, and VoIP ATA connectivity.

For initial setup, see: `brume2-tailscale-setup-guide.md`

---

## Quick Diagnostic Commands

Run these on the Brume to quickly identify issues:

```bash
# Is Tailscale connected?
tailscale status

# Are routes being advertised?
tailscale debug prefs | grep -A3 AdvertiseRoutes

# Can Tailscale reach the PBX?
tailscale ping <PBX-IP>

# Can the Brume ping the PBX?
ping -c 3 <PBX-IP>

# Check firewall rules exist
iptables -L FORWARD -n -v | head -10
iptables -t nat -L POSTROUTING -n -v | grep MASQ
```

---

## Common Issues and Solutions

### Issue 1: "no matching peer" from tailscale ping

**Symptom**: `tailscale ping <PBX-IP>` returns "no matching peer"

**Causes**:
1. The destination subnet isn't being advertised by any Tailscale node
2. The route isn't approved in Tailscale admin
3. This Brume isn't accepting routes (`--accept-routes` not set)

**Solutions**:

Check if the main Brume is advertising the PBX subnet:
```bash
# On main Brume
tailscale debug prefs | grep -A3 AdvertiseRoutes
```

Check Tailscale admin console:
1. Go to https://login.tailscale.com/admin/machines
2. Find the main Brume
3. Verify the PBX subnet (e.g., 192.168.1.0/24) is listed and **approved** (not "awaiting approval")

Re-apply accept-routes on the remote Brume:
```bash
tailscale up --advertise-routes=192.168.X.0/24 --accept-routes --reset
```

---

### Issue 2: Brume Can Ping PBX, But LAN Devices Cannot

**Symptom**:
- `ping <PBX-IP>` from Brume works
- ATA or other LAN devices cannot reach PBX
- SIP registration fails

**Cause**: Missing MASQUERADE rule. Tailscale requires source NAT for LAN traffic.

**Solution**:

Add MASQUERADE rule:
```bash
iptables -t nat -I POSTROUTING -o tailscale0 -s 192.168.X.0/24 -j MASQUERADE
```

Verify it's active:
```bash
iptables -t nat -L POSTROUTING -n -v | grep MASQ
```

Make it persistent by adding to `/etc/firewall.user`.

---

### Issue 3: AdvertiseRoutes is Empty

**Symptom**: `tailscale debug prefs | grep AdvertiseRoutes` shows empty brackets `[]`

**Cause**: GL.iNet firmware doesn't automatically apply `tailscale up` flags from `/etc/config/tailscale`

**Solution**:

Manually run:
```bash
tailscale up --advertise-routes=192.168.X.0/24 --accept-routes --reset
```

Make it persistent by adding to `/etc/rc.local`:
```bash
sleep 10
tailscale up --advertise-routes=192.168.X.0/24 --accept-routes --reset
```

---

### Issue 4: Settings Lost After Reboot

**Symptom**: Everything works until Brume reboots, then connectivity fails

**Cause**: Tailscale flags and iptables rules don't persist automatically

**Solution**:

Verify persistence files exist:
```bash
cat /etc/rc.local        # Should have tailscale up command
cat /etc/firewall.user   # Should have MASQUERADE and FORWARD rules
```

If missing, add them per the setup guide.

---

### Issue 5: Settings Lost After Firewall Restart

**Symptom**: After `/etc/init.d/firewall restart`, Tailscale connectivity breaks

**Cause**: Firewall restart flushes Tailscale's iptables chains (ts-forward, etc.)

**Solution**:

Restart Tailscale after firewall:
```bash
/etc/init.d/tailscale restart
```

Add to end of `/etc/firewall.user`:
```bash
/etc/init.d/tailscale restart
```

---

### Issue 6: Route Approved But Still "No Matching Peer"

**Symptom**: Route shows approved in Tailscale admin, but `tailscale ping` fails

**Cause**: The receiving Brume needs to refresh its netmap

**Solution**:

On the Brume that can't reach the subnet:
```bash
tailscale up --accept-routes --reset
```

Wait 30 seconds, then retry.

Check if the route appears in the netmap:
```bash
tailscale debug netmap 2>&1 | grep "192.168"
```

---

### Issue 7: ATA Sends SIP But PBX Doesn't Receive

**Symptom**:
- ATA shows "Registering" or "Registration Failed"
- tcpdump on PBX shows no SIP packets
- Ping from PBX to ATA works

**Cause**: Traffic from ATA isn't being routed through Tailscale properly

**Diagnostic**:

On the remote Brume, add LOG rule:
```bash
iptables -I FORWARD 1 -p udp --dport 5060 -j LOG --log-prefix "SIP: "
```

Trigger ATA registration, then check:
```bash
dmesg | grep "SIP:" | tail -10
```

If you see SIP packets with `OUT=tailscale0`, the Brume is forwarding them. Issue is further along the path.

**Solution**: Ensure MASQUERADE rule is in place and Tailscale routes are approved.

---

### Issue 8: Overlapping Subnets

**Symptom**: Remote Brume has same subnet (e.g., 192.168.1.x) on its WAN as the PBX network

**Cause**: Tailscale can't distinguish between local and remote addresses on the same subnet

**Diagnostic**:
```bash
ip route | grep 192.168.1
```

If you see both `dev tailscale0` and `dev eth0` for the same subnet, there's a conflict.

**Solution**:
- Change the remote site's WAN network if possible
- Or use explicit /32 routes for specific hosts
- The Tailscale route should have lower metric to be preferred

---

## Tracing Traffic Flow

### Method 1: iptables Packet Counters

Check counters before and after test:

```bash
# Before
iptables -L FORWARD -n -v | head -5

# Run test (ping, SIP registration, etc.)

# After - compare packet counts
iptables -L FORWARD -n -v | head -5
```

### Method 2: iptables LOG Rules

Add LOG rules to trace traffic:

```bash
# Log traffic going to tailscale0
iptables -I FORWARD 1 -o tailscale0 -j LOG --log-prefix "TO-TS: "

# Log traffic coming from tailscale0
iptables -I FORWARD 1 -i tailscale0 -j LOG --log-prefix "FROM-TS: "

# Check logs
dmesg | grep -E "TO-TS|FROM-TS" | tail -20

# Clean up when done
iptables -D FORWARD -o tailscale0 -j LOG --log-prefix "TO-TS: "
iptables -D FORWARD -i tailscale0 -j LOG --log-prefix "FROM-TS: "
```

### Method 3: tcpdump (if available)

```bash
# Watch SIP traffic
tcpdump -i any port 5060 -n

# Watch ICMP (ping)
tcpdump -i any icmp -n

# Watch specific host
tcpdump -i any host <PBX-IP> -n
```

Note: tcpdump may not be installed on GL.iNet firmware.

---

## Traffic Flow Diagram

### ATA → PBX (SIP Registration)

```
1. ATA (192.168.10.100) sends REGISTER to <PBX-IP>:5060
2. → Gateway (192.168.10.1 = Brume br-lan)
3. → Brume FORWARD chain (br-lan → tailscale0)
4. → MASQUERADE changes source to 100.x.x.x (Tailscale IP)
5. → Tailscale encrypts and sends to main Brume
6. → Main Brume receives on tailscale0
7. → Main Brume FORWARD chain (tailscale0 → eth0)
8. → PBX (<PBX-IP>)
```

### PBX → ATA (SIP Response)

```
1. PBX sends response to 100.x.x.x (MASQ'd source)
2. → Gateway reaches main Brume eth0
3. → Connection tracking routes to tailscale0
4. → Tailscale sends to remote Brume
5. → Remote Brume de-MASQUERADEs
6. → Forwards to ATA (192.168.10.100)
```

---

## Verifying Each Component

### 1. Tailscale Connection
```bash
tailscale status
# Should show "active" or "idle" for other nodes, not "offline"
```

### 2. Route Advertisement
```bash
tailscale debug prefs | grep -A3 AdvertiseRoutes
# Should show your subnet
```

### 3. Route Acceptance
```bash
tailscale debug netmap 2>&1 | grep "192.168"
# Should show the PBX subnet from main Brume
```

### 4. Kernel Routes
```bash
ip route | grep tailscale0
# Should show routes via tailscale0
```

### 5. Firewall FORWARD Rules
```bash
iptables -L FORWARD -n -v | grep -E "tailscale0|br-lan"
# Should show ACCEPT rules with packet counts
```

### 6. NAT MASQUERADE
```bash
iptables -t nat -L POSTROUTING -n -v | grep MASQ
# Should show MASQUERADE rule for your subnet
```

### 7. Tailscale Firewall Chain
```bash
iptables -L ts-forward -n -v
# Should exist and show packet counts
```

---

## Recovery Procedures

### After Power Loss

1. SSH to Brume via Tailscale IP
2. Check status: `tailscale status`
3. If routes missing: `tailscale up --advertise-routes=192.168.X.0/24 --accept-routes --reset`
4. If firewall rules missing: `/etc/init.d/firewall restart`
5. Check ATA registration on PBX

### After Firmware Update

1. Verify `/etc/rc.local` still has tailscale up command
2. Verify `/etc/firewall.user` still has custom rules
3. Verify UCI firewall zone: `uci show firewall | grep ts`
4. If missing, re-run setup steps
5. Restore from backup if needed

### Complete Reset

If nothing works, restore from backup:
1. Access Brume web UI
2. System → Backup / Restore
3. Import Configuration
4. Select backup .tar.gz file
5. Brume will reboot with restored settings

---

## Key Files Reference

| File | Purpose | Survives Reboot? |
|------|---------|------------------|
| `/etc/rc.local` | Boot commands (tailscale up) | Yes |
| `/etc/firewall.user` | Custom iptables rules | Yes |
| `/etc/config/firewall` | UCI firewall config | Yes |
| `/etc/config/tailscale` | GL.iNet Tailscale config | Yes |
| `/etc/tailscale/tailscaled.state` | Tailscale auth | Yes |
| iptables rules (in memory) | Active firewall rules | No* |
| Tailscale flags (in memory) | Active TS settings | No* |

*These are restored from config files at boot/restart
