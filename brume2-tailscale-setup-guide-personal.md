# Brume 2 / Tailscale Setup Guide for VoIP ATAs

This guide covers setting up a GL.iNet Brume 2 router with Tailscale to connect remote ATAs to a central PBX.

## Overview

```
Remote Site                          Main Site
┌─────────────────┐                  ┌─────────────────┐
│ ATA (ext 10x)   │                  │ PBX             │
│ 192.168.X.100   │                  │ 192.168.1.142   │
└────────┬────────┘                  └────────┬────────┘
         │                                    │
┌────────┴────────┐                  ┌────────┴────────┐
│ Brume 2         │    Tailscale    │ Brume 2 (main)  │
│ LAN: 192.168.X.1│◄──────────────►│ WAN: 192.168.1.x│
│ TS: 100.x.x.x   │     VPN        │ TS: 100.125.94.29│
└─────────────────┘                  └─────────────────┘
```

## Prerequisites

- GL.iNet Brume 2 (GL-MT2500) router, one for each remote line and one that will be on the same local network as the RasPBX https://www.gl-inet.com/products/gl-mt2500/
- Tailscale account (free level is adequate) https://tailscale.com/
- **Tailscale client installed on your admin computer** - download from https://tailscale.com/download and sign in with the same Tailscale account. This allows you to SSH into any Brume via its Tailscale IP from anywhere. You will be on the same Tailnet.
- A Raspberry Pi with RASPBX image written to the microSD card http://www.raspbx.org/downloads/
- An ATA device for each line and one that will be on the same local network as the RasPBX and Brume 2 (Cisco SPA, Linksys PAP2T, Grandstream GS-HT802 etc.) https://www.ebay.com/sch/i.html?_nkw=analog+telephone+adapter

---

## Step 1A: Main Site Brume 2 Setup

The main site Brume 2 sits on the same local network as the PBX and acts as the Tailscale gateway for remote sites. It stays in **Router mode** (the default) but connects differently than remote Brumes.

1. At your local site where the PBX will live, connect Brume 2 **WAN port** to your router (gets internet and local network via DHCP)
2. The PBX and local ATA connect to the **same network as the Brume's WAN** (not the LAN port)
3. Access Brume web UI (default: http://192.168.8.1)
4. Set admin password
5. Leave Network Mode as **Router** (the default)
6. Note the Brume's WAN IP (check your router's DHCP client list, e.g., 192.168.1.106)

> **Key difference from remote Brumes:** The main Brume's PBX is on its WAN side (192.168.1.0/24), not LAN side. This means the firewall rules use `eth0` instead of `br-lan`. See the "Main Site Brume Setup" section at the end of this guide for the specific configuration.


## Step 1B: Remote Site Brume 2 and ATA Setup

Each remote site needs its own Brume 2 in **router mode** (the default) with a unique subnet. Repeat this step for each remote location.

1. Connect Brume 2 **WAN port** to the remote site's local network/router (gets internet via DHCP)
2. Connect Brume **LAN port** to ATA (or a switch with ATA connected)
3. Access Brume web UI (default: http://192.168.8.1)
4. Set admin password
5. Go to **Network → LAN**
6. Configure a unique LAN subnet - **pick a subnet not used elsewhere** (see table below)
7. Note the Brume's LAN IP (e.g., 192.168.10.1) - this becomes the ATA's gateway


### Choosing a Subnet

Use a unique /24 subnet for each site. Already in use:

| Site | Subnet |
|------|--------|
| Main (PBX) | 192.168.1.0/24 |
| gl-mt2500-b | 192.168.50.0/24 |
| gl-mt2500-d | 192.168.9.0/24 |
| gl-mt2500-j | 192.168.8.0/24 |

Available examples: 192.168.10.0/24, 192.168.11.0/24, 192.168.12.0/24, etc.

---

## Step 2: Enable Tailscale on Brume

1. In Brume web UI: **VPN → Tailscale**
2. Click **Enable Tailscale**
3. Click the authentication link and log into your Tailscale account
4. Enable **"Allow Remote Access LAN"**
5. Enable **"Allow Remote Access WAN"** (if ATA needs to reach WAN-side devices)
6. Note the Tailscale IP assigned (100.x.x.x) - visible in Tailscale admin console

---

## Step 3: Configure UCI Firewall Zone

SSH to the Brume (via Tailscale IP or LAN):

```bash
ssh root@<tailscale-ip>
```

Run these commands to create a Tailscale firewall zone:

```bash
# Create Tailscale zone
uci add firewall zone
uci set firewall.@zone[-1].name='ts'
uci set firewall.@zone[-1].input='ACCEPT'
uci set firewall.@zone[-1].output='ACCEPT'
uci set firewall.@zone[-1].forward='ACCEPT'
uci set firewall.@zone[-1].device='tailscale0'

# Add forwarding ts -> lan
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='ts'
uci set firewall.@forwarding[-1].dest='lan'

# Add forwarding lan -> ts
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='lan'
uci set firewall.@forwarding[-1].dest='ts'

# Save changes
uci commit firewall
```

Verify:
```bash
uci show firewall | grep -E "zone.*ts|forwarding"
```

Should show `zone[*].name='ts'` and forwarding rules for ts↔lan.

---

## Step 4: Configure /etc/rc.local

This ensures Tailscale settings persist after reboot.

> **IMPORTANT**: When typing these commands, the closing `ENDFILE` must have **no spaces before it**. If your terminal adds leading spaces when you paste, use the arrow keys and backspace to remove them before pressing Enter.

Replace `192.168.NEW.0/24` with your actual subnet:

```bash
cat > /etc/rc.local << 'ENDFILE'
# Put your custom commands here that should be executed once
# the system init finished. By default this file does nothing.

. /lib/functions/gl_util.sh
remount_ubifs

# Wait for Tailscale to be ready
sleep 10

# Apply Tailscale settings - CHANGE SUBNET BELOW
tailscale up --advertise-routes=192.168.NEW.0/24 --accept-routes --reset

# Add explicit route to PBX
ip route add 192.168.1.142/32 dev tailscale0 2>/dev/null || true

exit 0
ENDFILE
```

Verify:
```bash
cat /etc/rc.local
```

---

## Step 5: Configure /etc/firewall.user

This adds the MASQUERADE rule required for Tailscale to forward LAN traffic.

> **IMPORTANT**: Same as above - the closing `ENDFILE` must have **no leading spaces**.

Replace `192.168.NEW.0/24` with your actual subnet:

```bash
cat >> /etc/firewall.user << 'ENDFILE'

# allow LAN <-> Tailscale
iptables -I FORWARD -i tailscale0 -o br-lan -j ACCEPT
iptables -I FORWARD -i br-lan -o tailscale0 -j ACCEPT

# MASQUERADE traffic from LAN to Tailscale - CHANGE SUBNET BELOW
iptables -t nat -I POSTROUTING -o tailscale0 -s 192.168.NEW.0/24 -j MASQUERADE

# Restart Tailscale to restore its rules
/etc/init.d/tailscale restart
ENDFILE
```

Verify:
```bash
cat /etc/firewall.user
```

---

## Step 6: Apply Settings Now

Apply the settings immediately without rebooting:

```bash
# Apply Tailscale settings (change subnet)
tailscale up --advertise-routes=192.168.NEW.0/24 --accept-routes --reset

# Apply firewall rules
/etc/init.d/firewall restart
```

---

## Step 7: Approve Route in Tailscale Admin

**This step is critical and often forgotten!**

1. Go to https://login.tailscale.com/admin/machines
2. Find the new Brume device in the list
3. Click on it to expand details
4. Look for **"Subnets"** section
5. The new subnet (192.168.NEW.0/24) may show as **"Awaiting approval"**
6. **Click to approve/enable the subnet route**

Without this step, other Tailscale nodes won't be able to route to your new subnet.

---

## Step 8: Verify Tailscale Routing

On the new Brume, test connectivity to the PBX:

```bash
# Should show route is advertised
tailscale debug prefs | grep -A3 AdvertiseRoutes

# Should return "pong from gl-mt2500" (main Brume)
tailscale ping 192.168.1.142

# Should succeed with ~20-50ms latency
ping -c 3 192.168.1.142
```

### If `tailscale ping` says "no matching peer":

1. Check that 192.168.1.0/24 route is approved for the **main Brume** in Tailscale admin
2. Run `tailscale up --accept-routes --reset` again on the new Brume
3. Wait 30 seconds and retry

---

## Step 9: Create Extension in FreePBX

Before configuring the ATA, create the SIP extension in FreePBX:

1. Log into FreePBX web interface on your local network.  You can use your router to see what the IP address is.
2. Go to **Applications → Extensions**
3. Click **Add Extension** → **Add New PJSIP Extension** (or SIP if using chan_sip)
4. Enter:
   - **User Extension**: Extension number (e.g., 101, 102)
   - **Display Name**: Description (e.g., "<y line")
   - **Secret**: Note the auto-generated password or set your own
5. Click **Submit**
6. Click the **red "Apply Config" button** at the top
7. Copy the **Secret** (password) - you'll need it for the ATA
8. Repeat steps 2 through 7 the same for each additional remote line

---

## Step 10: Configure ATA

Access the ATA's web interface and configure:

### Network Settings
| Setting | Value |
|---------|-------|
| Connection Type | **Static IP** |
| IP Address | 192.168.NEW.100 (example) |
| Subnet Mask | 255.255.255.0 |
| Default Gateway | 192.168.NEW.1 (Brume LAN IP) |
| Primary DNS | 8.8.8.8 or Brume LAN IP |

### SIP/Line Settings
| Setting | Value |
|---------|-------|
| SIP Proxy | 192.168.1.142 |
| SIP Port | 5060 |
| Register | Yes |
| User ID | Extension number (e.g., 104) |
| Auth ID | Same as User ID |
| Password | SIP secret from FreePBX |

**Click "Submit All Changes"** to save and trigger registration.

---

## Step 11: Verify SIP Registration

On the PBX (Raspberry Pi):

```bash
asterisk -rx "sip show peers" | grep <extension>
```

Should show something like:
```
104/104          192.168.9.217    D   N   A  5060     OK (45 ms)
```

If not registered, wait 1-2 minutes or reboot the ATA.

---

## Step 12: Reboot Test

Verify everything survives a power cycle:

1. **Power off** the Brume (unplug power)
2. Wait 30 seconds
3. **Power on**
4. Wait 3-5 minutes for full boot and Tailscale connection
5. Check ATA registration on PBX:
   ```bash
   asterisk -rx "sip show peers" | grep <extension>
   ```

If registration fails after reboot, check:
- `/etc/rc.local` has the tailscale up command
- `/etc/firewall.user` has the MASQUERADE rule
- Subnet route is still approved in Tailscale admin

---

## Step 13: Export Backup

Save a backup of the Brume configuration:

1. Access Brume web UI (via LAN or Tailscale IP)
2. Go to **System → Backup / Restore**
3. Click **Export Configuration**
4. Save the `.tar.gz` file with a descriptive name:
   - Example: `gl-mt2500-site-name-2024-11-30.tar.gz`

Store backups safely - they can restore the full configuration if needed.

---

## Quick Reference

### Key Files on Brume

| File | Purpose |
|------|---------|
| `/etc/rc.local` | Tailscale up command at boot |
| `/etc/firewall.user` | MASQUERADE and FORWARD rules |
| `/etc/config/firewall` | UCI firewall zones (persistent) |
| `/etc/config/tailscale` | GL.iNet Tailscale settings |
| `/etc/tailscale/tailscaled.state` | Tailscale auth state |

### Essential Commands

```bash
# Check Tailscale status
tailscale status

# Check advertised routes
tailscale debug prefs | grep -A3 AdvertiseRoutes

# Test Tailscale routing to an IP
tailscale ping <ip-address>

# Check firewall rules
iptables -L FORWARD -n -v | head -10
iptables -t nat -L POSTROUTING -n -v | grep MASQ

# Restart Tailscale
/etc/init.d/tailscale restart

# Restart firewall (also runs firewall.user)
/etc/init.d/firewall restart
```

### Current Network Assignments

| Device | Tailscale IP | LAN Subnet | Extensions |
|--------|--------------|------------|------------|
| gl-mt2500 (main) | 100.125.94.29 | 192.168.1.0/24 | PBX site |
| gl-mt2500-b | 100.77.242.75 | 192.168.50.0/24 | 105 |
| gl-mt2500-d | 100.76.158.115 | 192.168.9.0/24 | 104, 107 |
| gl-mt2500-j | 100.103.207.125 | 192.168.8.0/24 | 106 |

---

## Main Site Brume Setup

The main site Brume (at the PBX location) has slightly different configuration since its WAN is on the same network as the PBX.

### /etc/firewall.user (Main Brume)

```bash
# MASQUERADE traffic from LAN to Tailscale
iptables -t nat -I POSTROUTING -o tailscale0 -s 192.168.1.0/24 -j MASQUERADE

# FORWARD rules for eth0 <-> tailscale0 (WAN to Tailscale)
iptables -I FORWARD -i tailscale0 -o eth0 -j ACCEPT
iptables -I FORWARD -i eth0 -o tailscale0 -j ACCEPT

# Restart Tailscale to restore its rules
/etc/init.d/tailscale restart
```

### /etc/rc.local (Main Brume)

```bash
# Wait for Tailscale to be ready
sleep 10

# Apply Tailscale settings
tailscale up --advertise-routes=192.168.1.0/24 --accept-routes --reset
```

Note: Main Brume uses `eth0` (WAN) instead of `br-lan` (LAN) because the PBX is on the WAN side.
