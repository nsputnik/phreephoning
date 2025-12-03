# Troubleshooting Continuation Notes
**Date:** November 30, 2024
**Session stopped at:** Debugging Tailscale subnet routing

---

## Summary of Session

We were hardening gl-mt2500-d (Brume at 100.76.158.115) which serves extensions 104 and 107 via a Cisco SPA112 ATA.

---

## What We Verified Working

| Component | Status | Evidence |
|-----------|--------|----------|
| Brume Tailscale connection | ✅ Working | Connected to gl-mt2500 (main site), direct connection |
| Route to PBX in Brume | ✅ Working | `ip route get 192.168.1.142` → via tailscale0 |
| /32 route persisted | ✅ In rc.local | `ip route add 192.168.1.142/32 dev tailscale0` |
| Firewall rules persisted | ✅ In firewall.user | iptables FORWARD rules for br-lan↔tailscale0 |
| Brume can ping PBX | ✅ Working | 24ms latency, 0% loss |
| Brume can ping ATA | ✅ Working | 1ms latency, 0% loss |
| ATA network settings | ✅ Correct | IP: 192.168.9.217, Gateway: 192.168.9.1, SIP Proxy: 192.168.1.142:5060 |
| ATA sending SIP packets | ✅ Confirmed | tcpdump on br-lan shows REGISTER packets |
| Brume forwarding to tailscale0 | ✅ Confirmed | tcpdump on tailscale0 shows same packets |

---

## The Problem We Found

**Packets enter Tailscale tunnel but never arrive at PBX.**

```
ATA (192.168.9.217)
    → Brume br-lan ✅
    → Brume tailscale0 ✅
    → [TAILSCALE TUNNEL] ❓
    → gl-mt2500 (main site) ❌
    → PBX (192.168.1.142) ❌
```

Evidence:
- `tcpdump -i any src net 192.168.9.0/24` on PBX shows **0 packets**
- ATA shows registration status: **Failed**

---

## Root Cause (Suspected)

**gl-mt2500-d is likely NOT advertising its subnet (192.168.9.0/24) to Tailscale.**

Without this advertisement, other Tailscale nodes don't know how to route traffic TO 192.168.9.x, and traffic FROM 192.168.9.x may not be properly tunneled.

The rc.local file we saw does NOT contain a `tailscale up --advertise-routes` command - it only has the static route:

```bash
# Current /etc/rc.local on gl-mt2500-d:
ip route add 192.168.1.142/32 dev tailscale0
exit 0
```

**Missing:** `tailscale up --advertise-routes=192.168.9.0/24 --accept-routes`

---

## Next Steps When Resuming

### Step 1: Check current Tailscale route advertisement

SSH to gl-mt2500-d (100.76.158.115) and run:

```bash
tailscale debug prefs | grep -i route
```

This will show if `--advertise-routes` is configured.

### Step 2: Enable subnet advertisement

If not advertising, run:

```bash
tailscale up --advertise-routes=192.168.9.0/24 --accept-routes --reset
```

### Step 3: Approve the route in Tailscale admin console

Go to: https://login.tailscale.com/admin/machines

Find gl-mt2500-d and approve the 192.168.9.0/24 subnet route (if required by your Tailscale ACL settings).

### Step 4: Verify on main site

On gl-mt2500 (100.125.94.29), check that it can see and accept the 192.168.9.0/24 route:

```bash
tailscale status
ip route | grep 192.168.9
```

### Step 5: Test again

On PBX:
```bash
sudo tcpdump -i any src net 192.168.9.0/24 -c 5
```

And check registration:
```bash
asterisk -rx "sip show peers" | grep -E "104|107"
```

### Step 6: Persist the fix

Update `/etc/rc.local` on gl-mt2500-d:

```bash
# Wait for network
sleep 10

# Bring up Tailscale with subnet advertisement
tailscale up --advertise-routes=192.168.9.0/24 --accept-routes --reset

# Add explicit route to PBX
ip route add 192.168.1.142/32 dev tailscale0 2>/dev/null || true

exit 0
```

---

## Key IPs Reference

| Device | Tailscale IP | LAN IP | Role |
|--------|--------------|--------|------|
| gl-mt2500 | 100.125.94.29 | 192.168.1.x | Main site Brume (PBX side) |
| gl-mt2500-d | 100.76.158.115 | 192.168.9.1 | Remote Brume we're fixing |
| SPA112 ATA | n/a | 192.168.9.217 | Ext 104, 107 |
| PBX (Raspberry Pi) | n/a | 192.168.1.142 | Asterisk/FreePBX |

---

## Files Modified/Verified This Session

On gl-mt2500-d:
- `/etc/rc.local` - Has /32 route, needs tailscale up command
- `/etc/firewall.user` - Correct, has iptables rules

---

## Commands to SSH In

```bash
# To gl-mt2500-d (the one we're fixing)
ssh root@100.76.158.115

# To main site Brume
ssh root@100.125.94.29

# To PBX
ssh pi@192.168.1.142  # or whatever user
```

---

## Other Files in This Project

- `overview.txt` - Original problem description and topology
- `hardening-checklist.md` - Full hardening procedure for all devices
