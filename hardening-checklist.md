# Phone System Hardening Checklist

Goal: Make all configurations survive power loss on Brume 2 routers and ATAs.

---

## PART 1: BRUME 2 HARDENING (Repeat for each remote Brume)

### Sites to harden:
- [ ] gl-mt2500-b (100.77.242.75) - serves ext 105
- [ ] gl-mt2500-d (100.76.158.115) - serves ext 104, 107
- [ ] gl-mt2500-j (100.103.207.125) - serves ext 106

---

### Step 1.1: Document Current Working State

Before changing anything, SSH to the Brume and capture what's working:

```bash
# SSH to the Brume (use appropriate Tailscale IP)
ssh root@100.77.242.75

# Capture current state to a file
echo "=== DATE ===" > /tmp/working-state.txt
date >> /tmp/working-state.txt

echo "=== TAILSCALE STATUS ===" >> /tmp/working-state.txt
tailscale status >> /tmp/working-state.txt

echo "=== IP ROUTES ===" >> /tmp/working-state.txt
ip route >> /tmp/working-state.txt

echo "=== ROUTE TO PBX ===" >> /tmp/working-state.txt
ip route get 192.168.1.142 >> /tmp/working-state.txt

echo "=== IPTABLES FORWARD ===" >> /tmp/working-state.txt
iptables -L FORWARD -n -v >> /tmp/working-state.txt

echo "=== INTERFACES ===" >> /tmp/working-state.txt
ip addr >> /tmp/working-state.txt

cat /tmp/working-state.txt
```

**Copy this output and save it locally** - this is your "known good" reference.

---

### Step 1.2: Persist Tailscale Configuration

The issue: `tailscale up` with flags like `--advertise-routes` doesn't persist after reboot.

**Check current Tailscale state:**
```bash
tailscale status
cat /etc/config/tailscale  # if it exists
```

**Fix: Create persistent Tailscale startup script**

First, identify what subnet this Brume advertises:
- gl-mt2500-b: 192.168.50.0/24
- gl-mt2500-d: 192.168.9.0/24
- gl-mt2500-j: 192.168.8.0/24

Create/edit `/etc/rc.local` (runs at boot):

```bash
vi /etc/rc.local
```

Add BEFORE the `exit 0` line (adjust subnet for each Brume):

```bash
# Wait for Tailscale to be ready
sleep 10

# Bring up Tailscale with correct flags
tailscale up --advertise-routes=192.168.50.0/24 --accept-routes --reset

# Add explicit route to PBX via Tailscale
ip route add 192.168.1.142/32 dev tailscale0 2>/dev/null || true

exit 0
```

**Make sure rc.local is executable:**
```bash
chmod +x /etc/rc.local
```

---

### Step 1.3: Persist Firewall/iptables Rules

The issue: Manual `iptables` commands for FORWARD rules don't survive reboot.

**Check if /etc/firewall.user exists:**
```bash
cat /etc/firewall.user
```

**Create or edit /etc/firewall.user:**
```bash
vi /etc/firewall.user
```

Add these rules (allows traffic between Tailscale and LAN):

```bash
# Allow forwarding between tailscale0 and br-lan (LAN bridge)
iptables -I FORWARD -i tailscale0 -o br-lan -j ACCEPT
iptables -I FORWARD -i br-lan -o tailscale0 -j ACCEPT

# Ensure related/established connections are allowed
iptables -I FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
```

**Verify firewall.user is sourced by OpenWrt:**
```bash
uci show firewall | grep include
```

You should see something like:
```
firewall.@include[0].path='/etc/firewall.user'
```

If not, add it:
```bash
uci add firewall include
uci set firewall.@include[-1].path='/etc/firewall.user'
uci commit firewall
```

**Restart firewall to test:**
```bash
/etc/init.d/firewall restart
```

---

### Step 1.4: Verify UCI Network/Firewall Config

Check that zones are correctly defined:

```bash
# Show all firewall zones
uci show firewall | grep "zone.*name"

# Check what interfaces are in each zone
uci show firewall | grep "zone.*network"
```

**Optional but recommended:** Add tailscale0 to a zone via UCI (more robust than iptables rules alone):

```bash
# Option A: Add tailscale to LAN zone (simplest)
uci add_list firewall.@zone[0].device='tailscale0'
uci commit firewall
/etc/init.d/firewall restart

# OR Option B: Create dedicated Tailscale zone
uci add firewall zone
uci set firewall.@zone[-1].name='tailscale'
uci set firewall.@zone[-1].input='ACCEPT'
uci set firewall.@zone[-1].output='ACCEPT'
uci set firewall.@zone[-1].forward='ACCEPT'
uci set firewall.@zone[-1].device='tailscale0'

# Add forwarding between tailscale and lan zones
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='tailscale'
uci set firewall.@forwarding[-1].dest='lan'

uci add firewall forwarding
uci set firewall.@forwarding[-1].src='lan'
uci set firewall.@forwarding[-1].dest='tailscale'

uci commit firewall
/etc/init.d/firewall restart
```

---

### Step 1.5: Test Persistence (CRITICAL)

**Controlled reboot test:**

1. Verify everything works now:
   ```bash
   ping 192.168.1.142
   ip route get 192.168.1.142
   ```

2. Reboot the Brume:
   ```bash
   reboot
   ```

3. Wait 2-3 minutes, then SSH back in and verify:
   ```bash
   # Check Tailscale is up with correct routes
   tailscale status

   # Check route to PBX exists
   ip route get 192.168.1.142

   # Check iptables rules are present
   iptables -L FORWARD -n | head -10

   # Test connectivity
   ping 192.168.1.142
   ```

4. Check if ATA can register (from PBX):
   ```bash
   # On the Raspberry Pi
   asterisk -rx "sip show peers" | grep -E "104|105|106|107"
   ```

**If anything failed after reboot, fix the persistence mechanism before moving on.**

---

## PART 2: ATA HARDENING (Repeat for each ATA)

### ATAs to harden:
- [ ] Linksys PAP2T at 192.168.1.106 (ext 102, 103) - local, low risk
- [ ] Cisco SPA2102 at 192.168.50.133 (ext 105)
- [ ] Cisco SPA112 at 192.168.9.217 (ext 104, 107)
- [ ] Cisco SPA2102 at 192.168.8.121 (ext 106)

---

### Step 2.1: Document Current ATA Settings

Access each ATA's web UI and record these critical settings:

**Network Settings to document:**
| Setting | Value |
|---------|-------|
| Connection Type | Static / DHCP |
| IP Address | |
| Subnet Mask | |
| Default Gateway | |
| Primary DNS | |

**SIP Settings to document (per line):**
| Setting | Value |
|---------|-------|
| Proxy | 192.168.1.142 |
| Register | yes |
| User ID | (extension number) |
| Password | (SIP secret) |
| SIP Port | 5060 |

---

### Step 2.2: Ensure Settings Are Saved to Flash

**For Cisco SPA series (SPA112, SPA2102):**

1. Log into web UI (http://[ATA-IP]/admin)
2. Make any change (or just re-enter a value)
3. Click **Submit All Changes**
4. Look for "Settings saved" or similar confirmation
5. **Important:** Some Cisco ATAs have a separate "Save to Flash" or "Backup" option - use it if available

**For Linksys PAP2T:**

1. Log into web UI
2. Go to Admin > Management
3. Look for "Save Settings to Flash" option
4. Click it explicitly

---

### Step 2.3: Export/Backup ATA Configuration

**Cisco SPA series:**
- Navigate to: Administration > Configuration Profile
- Look for "Export" or "Backup Configuration"
- Save the .xml or .cfg file locally
- Name it: `SPA112-ext104-107-backup.cfg` (descriptive)

**Linksys PAP2T:**
- Navigate to: Administration > Backup & Restore
- Export configuration
- Save locally

**Store these backups somewhere safe** - you can re-import them after a factory reset.

---

### Step 2.4: Verify Gateway is Correct

The ATA's gateway must be able to route to 192.168.1.142 via Tailscale.

| ATA Location | Correct Gateway Should Be |
|--------------|---------------------------|
| 192.168.50.x subnet | Brume gl-mt2500-b LAN IP (likely 192.168.50.1) |
| 192.168.9.x subnet | Brume gl-mt2500-d LAN IP (likely 192.168.9.1) |
| 192.168.8.x subnet | Brume gl-mt2500-j LAN IP (likely 192.168.8.1) |

**Verify from Brume:**
```bash
# Check Brume's LAN IP
ip addr show br-lan | grep inet
```

The ATA's gateway must match this IP.

---

### Step 2.5: Disable DHCP (Recommended for Stability)

If ATAs are set to DHCP, they can get different IPs or wrong gateways after reboot.

**Recommendation:** Set each ATA to Static IP with:
- IP: Current IP (e.g., 192.168.9.217)
- Subnet: 255.255.255.0
- Gateway: Brume's LAN IP
- DNS: 8.8.8.8 or Brume's LAN IP

---

## PART 3: VERIFICATION PROCEDURE

After hardening, run this verification for each site:

### Pre-Reboot Verification
```bash
# From your laptop/main machine
# 1. Check all extensions are registered
ssh pi@192.168.1.142 "asterisk -rx 'sip show peers'" | grep -E "102|103|104|105|106|107"
```

### Simulated Power Loss Test

**For each site, do this sequence:**

1. **Power off the Brume** (unplug it)
2. Wait 30 seconds
3. **Power it back on**
4. Wait 3 minutes for full boot
5. **Check from PBX:**
   ```bash
   asterisk -rx "sip show peers" | grep [extension]
   ```
6. If not registered within 5 minutes, troubleshoot using the decision tree

### Full System Test

1. Power off ALL Brumes and ATAs
2. Power on Brumes first, wait 3 minutes
3. Power on ATAs, wait 2 minutes
4. Verify all extensions register

---

## QUICK REFERENCE: Files That Must Persist

| Device | File | Contains |
|--------|------|----------|
| Each Brume | `/etc/rc.local` | Tailscale up command, route to PBX |
| Each Brume | `/etc/firewall.user` | iptables FORWARD rules |
| Each Brume | `/etc/config/firewall` | UCI zone definitions |
| Each ATA | (internal flash) | Network + SIP settings |

---

## EMERGENCY RECOVERY COMMANDS

If a site breaks and you need to fix it fast:

**SSH to the broken Brume and run:**
```bash
# Bring Tailscale up (adjust subnet)
tailscale up --advertise-routes=192.168.50.0/24 --accept-routes --reset

# Add route to PBX
ip route add 192.168.1.142/32 dev tailscale0

# Add firewall rules
iptables -I FORWARD -i tailscale0 -o br-lan -j ACCEPT
iptables -I FORWARD -i br-lan -o tailscale0 -j ACCEPT

# Test
ping 192.168.1.142
```

Then figure out why persistence failed and fix `/etc/rc.local` or `/etc/firewall.user`.
