Phreephoneing: A Free, Private, Encrypted Phone System with Raspberry Pi and Analog Phones

What if you could pick up an old-school telephone in your house, call a friend's house across town, the country or world, and have that call travel over your existing internet connection, fully encrypted, with no monthly bill from a phone company? That's the basic idea behind this project. Phreephoneing is a free, private phone system built from a Raspberry Pi, Analog Telephone Adapters (ATAs), subnet routers, and a main router that creates an encrypted mesh between locations.

# WRT Powered Subnet Router / Tailscale Setup Guide for VoIP ATAs

This guide covers setting up a GL.iNet Brume 2/Beryl AX router with Tailscale to connect remote ATAs to a central PBX for free, private and encrypted phone system. It is not connected to the larger phone system. You can only call the users you set up with the system. You don't have to pay a phone company monthly, just your ISP. Wireguard encrypts the traffic between subnet routers, just like the major data centers do.

## Overview

```
Remote Site                          Main Site
┌─────────────────┐                  ┌─────────────────┐
│ ATA (ext 10x)   │                  │ PBX             │
│ 192.168.X.100   │                  │ 192.168.1.100   │
└────────┬────────┘                  └────────┬────────┘
         │                                    │
┌────────┴────────┐                  ┌────────┴────────┐
│ Brume 2         │    Tailscale    │ Brume 2 (main)  │
│ LAN: 192.168.X.1│◄──────────────►│ WAN: 192.168.1.x│
│ TS: 100.x.x.2   │     VPN        │ TS: 100.x.x.1   │
└─────────────────┘                  └─────────────────┘
```

## Prerequisites

- A broadband internet connection at the main site and each remote site (no telephone line needed—this system is completely independent from the telecom network)
- An existing wireless router with internet access at the main site and at each remote site (the Brume 2/Beryl AX connects to these routers)
- GL.iNet Brume 2 (GL-MT2500) router, one for each remote line and one for the main site that will be on the same local network as the PBX https://www.gl-inet.com/products/gl-mt2500/ or GL.iNet Beryl AX (GL-MT3000)](https://www.gl-inet.com/products/gl-mt3000/)  which is the wireless version, great to use when one of your remote line users does not want to place the phone next to the router, but to another location without having to run ethernet cables.  When we mention the remote Brume 2 and Tailscale, the Beryl AX can be substituted here, but we'll go into the wireless details in a separate section near the end.
- Tailscale account (free tier is adequate) https://tailscale.com/
- Tailscale client installed on your admin computer** - download from https://tailscale.com/download and sign in with the same Tailscale account. This allows you to SSH into any Brume via its Tailscale IP and also into each ATA admin. You will be on the same Tailnet.
- A Raspberry Pi 3, 4, or 5 (not Zero) with RasPBX image written to the microSD card http://www.raspbx.org/downloads/ RasPBX is just Asterisk 16.13.0 & FreePBX 15.0.16.75, Raspbian Buster Lite, Apache, PHP and MySQL all pre-installed on a bootable image.
- An ATA device for each line (Cisco SPA, Linksys PAP2T, Grandstream HT802, etc.) https://www.ebay.com/sch/i.html?_nkw=analog+telephone+adapter
- An old touch tone analog telephone at each location you want to call or you want to call you.

---

## Steps

**Main Site Setup (do these first):**
- Step 1: Main Site Brume 2 Setup
- Step 2: Main Site Firewall Configuration
- Step 3: Reserve PBX and Brume IP Addresses
- Step 4: Create Extensions in FreePBX
- Step 5: Configure Main Site ATA
- Step 6: Verify Main Site SIP Registration
- Step 7: Configure and Test Remote ATA Locally

**Remote Site Setup (repeat for each remote location):**
- Step 8: Remote Site Brume 2 Setup
- Step 9: Remote Site Firewall Configuration
- Step 10: Verify Tailscale Routing
- Step 11: Reconfigure Remote ATA for Deployment
- Step 12: Verify Remote SIP Registration
- Step 13: Deploy to Remote Site

**Final Steps:**
- Step 14: Reboot Test
- Step 15: Make a Test Call
- Step 16: Export Backups
- Optional: Wireless Setup with Beryl AX (Remote Sites)

---

# Main Site Setup

Complete all main site steps before setting up any remote sites.

---

## Step 1: Main Site Brume 2 Setup

The main site Brume 2 sits on the same local network as the PBX and acts as the Tailscale gateway for remote sites. It stays in **Router mode** (the default) but connects differently than remote Brumes.

> **Set up the main Brume first!** The main Brume must be configured and advertising its subnet before remote Brumes can connect to the PBX.

### Initial Setup

1. At your local site where the PBX will live, connect Brume 2 **WAN port** to your router (gets internet and local network via DHCP)
2. The PBX and local ATA connect to the **same network as the Brume's WAN** (not the LAN port), so just connect them to your router
3. Access Brume web UI (default: http://192.168.8.1)
4. Set admin password
5. Leave Network Mode as **Router** (the default)
6. Note the Brume's WAN IP (check your router's DHCP client list)

### Enable Tailscale and Join Tailnet

7. In Brume web UI: **Applications → Tailscale**
8. Click **Enable Tailscale**
9. Click the authentication link and log into your Tailscale account
10. Enable **"Allow Remote Access LAN"**
11. Enable **"Allow Remote Access WAN"**
12. Note the Tailscale IP assigned (100.x.x.x) - visible in Tailscale admin console

### Approve Route and Name Device

13. Go to https://login.tailscale.com/admin/machines
14. Find the main Brume and **approve the subnet route** (192.168.1.0/24)
15. **Rename the device** - click the three-dot menu → "Edit machine name" → name it something like "gl-mt2500-main" or "gl-mt2500-pbx-site" to identify it easily

> **Key difference from remote Brumes:** The main Brume's PBX is on its WAN side (e.g., 192.168.1.0/24), not LAN side. This means the firewall rules use `eth0` instead of `br-lan`.

---

## Step 2: Main Site Firewall Configuration

SSH to the main Brume to configure the firewall rules. These are specific to the **main site** because the PBX is on the WAN side.

```bash
ssh root@192.168.8.1
```
(Password is the same as the web UI admin password you created)

### Make Filesystem Writable

GL.iNet routers use a read-only overlay filesystem by default. Run this command first to ensure your changes persist across reboots:

```bash
. /lib/functions/gl_util.sh && remount_ubifs
```

### 2a. Create UCI Firewall Zone

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
The output should match the content you entered above.

### 2b. Configure /etc/rc.local (Main Site)

This ensures Tailscale settings persist after reboot.

> **IMPORTANT**: When typing these commands, the closing `ENDFILE` must have **no spaces before it**. If your terminal adds leading spaces when you paste, use the arrow keys and backspace to remove them before pressing Enter.

```bash
cat > /etc/rc.local << 'ENDFILE'
# Put your custom commands here that should be executed once
# the system init finished. By default this file does nothing.

. /lib/functions/gl_util.sh
remount_ubifs

# Wait for Tailscale to be ready
sleep 10

# Apply Tailscale settings - advertise the PBX subnet
tailscale up --advertise-routes=192.168.1.0/24 --accept-routes --reset

exit 0
ENDFILE
```

Verify:
```bash
cat /etc/rc.local
```
The output should match the content you entered above.

### 2c. Configure /etc/firewall.user (Main Site)

The main site uses `eth0` (WAN interface) because the PBX is on the WAN side.

> **IMPORTANT**: Same as above - the closing `ENDFILE` must have **no leading spaces**.

```bash
cat >> /etc/firewall.user << 'ENDFILE'

# MASQUERADE traffic from WAN subnet to Tailscale
iptables -t nat -I POSTROUTING -o tailscale0 -s 192.168.1.0/24 -j MASQUERADE

# FORWARD rules for eth0 <-> tailscale0 (WAN to Tailscale)
iptables -I FORWARD -i tailscale0 -o eth0 -j ACCEPT
iptables -I FORWARD -i eth0 -o tailscale0 -j ACCEPT

# Restart Tailscale to restore its rules
/etc/init.d/tailscale restart
ENDFILE
```

Verify:
```bash
cat /etc/firewall.user
```
The output should match the content you entered above.

### 2d. Apply Settings

```bash
tailscale up --advertise-routes=192.168.1.0/24 --accept-routes --reset
/etc/init.d/firewall restart
```

---

## Step 3: Reserve PBX and Brume IP Addresses

Log into your main site router and create DHCP reservations (also called "static leases" or "address reservations") for two devices:

1. **The PBX (Raspberry Pi)** at its current IP (e.g., 192.168.1.100). Without this, the router may assign a different IP after a power outage or reboot, and every ATA's SIP Proxy setting would need updating.

2. **The Brume 2** at its current WAN IP. Tailscale routing won't break if this changes — the Tailscale IP is independent — but reserving it preserves your **local-subnet fallback access**: SSH or web admin to a known IP on the LAN if Tailscale itself ever fails. This is your only out-of-band recovery path for the Brume.

Look for DHCP reservation, static lease, or address reservation in your router's settings. You'll need each device's MAC address and current IP.

---

## Step 4: Create Extensions in FreePBX

Create SIP extensions for **all** phones in your system - both the main site ATA and all remote ATAs. Do this now while you're at the main site.

1. Log into FreePBX web interface on your local network (check your router for the PBX IP). The default username is admin, password is admin.
2. Go to **Applications → Extensions**
3. Click **Add Extension** → **Add New PJSIP Extension** (or SIP if using chan_sip)
4. Enter:
   - **User Extension**: Extension number (e.g., 100 for main site, 101-109 for remote sites)
   - **Display Name**: Description (e.g., "Main House", "Mom and Dad", "Uncle Bob")
   - **Secret**: Copy the auto-generated password or set your own
5. Click **Submit**
6. Click the **red "Apply Config" button** at the top
7. Copy the **Secret** (password) - you'll need it for the ATA
8. Repeat steps 2 through 7 for each phone in your system (main site + all remote sites)

> **Tip:** Create all extensions now so you have the passwords ready when configuring each ATA.

---

## Step 5: Configure Main Site ATA

The main site ATA connects directly to your router (same network as the PBX), so configuration is simpler than remote ATAs.

Access the ATA's web interface and configure:

### Network Settings
| Setting | Value |
|---------|-------|
| Connection Type | **DHCP** or **Static IP** |
| IP Address | (If static: 192.168.1.101 or similar) |
| Subnet Mask | 255.255.255.0 |
| Default Gateway | Your router's IP (e.g., 192.168.1.1) |
| Primary DNS | 8.8.8.8 or your router's IP |

### SIP/Line Settings
| Setting | Value |
|---------|-------|
| SIP Proxy | 192.168.1.100 (PBX IP) |
| SIP Port | 5060 |
| Register | Yes |
| User ID | Extension number (e.g., 100) |
| Auth ID | Same as User ID |
| Password | SIP secret from FreePBX |
| NAT Mapping Enable | Yes |
| NAT Keep Alive Enable | Yes |

If you will use both lines on a 2 line ATA the 2nd line should use SIP port 5061, and it might set this automatically, but verify it.

> **Why NAT Mapping / NAT Keep Alive matter:** even at the main site the ATA usually sits behind your home router's NAT. With these enabled, the ATA periodically refreshes its NAT binding so the PBX can reach it for inbound calls. With them disabled, registration may appear to succeed but the path silently breaks after the NAT entry ages out, and inbound calls fail.

**Click "Submit All Changes"** to save and trigger registration.

---

## Step 6: Verify Main Site SIP Registration

### Check the ATA

1. Access the ATA's web admin (e.g., http://192.168.1.101)
2. Look for registration status - usually on the main status page or under Line/SIP settings
3. Should show **"Registered"** or **"Online"**

### Verify on the PBX

SSH into the PBX (Raspberry Pi). The username: root, password: raspberry.

```bash
ssh root@192.168.1.100
```

Check registration:
```bash
asterisk -rx "pjsip show endpoints" | grep 100
# or for chan_sip:
asterisk -rx "sip show peers" | grep 100
```

Should show the extension with status "OK" or "Avail".

### Test Dial Tone

Pick up the phone connected to the main site ATA. You should hear a dial tone.

> **Heads up — dial tone alone doesn't always mean registered.** On Linksys and Cisco SPA-style ATAs (including the Cisco ATA 191/192), dial tone is gated on registration: no dial tone means not registered. **Grandstream ATAs (HT801/802/812/813/818) give a dial tone whether they're registered or not** — the dial tone is generated locally by the ATA, not by the PBX. Always confirm registration via the ATA's web admin Status page or `pjsip show endpoints` on the PBX rather than relying on dial tone.

---

## Step 7: Configure and Test Remote ATA Locally

Before setting up the remote Brume 2, test the remote ATA locally on your main network. This confirms the extension works before adding the complexity of the Tailscale tunnel.

### Temporary Local Setup

Connect the remote ATA directly to your main router (the same network as the PBX), **not** to the Brume 2 yet. Leave the ATA's network settings on **DHCP/Dynamic** - no network configuration is needed for this test.

### SIP/Line Settings
| Setting | Value |
|---------|-------|
| SIP Proxy | 192.168.1.100 (PBX IP) |
| SIP Port | 5060 |
| Register | Yes |
| User ID | Extension number (e.g., 101) |
| Auth ID | Same as User ID |
| Password | SIP secret from FreePBX (created in Step 4) |
| NAT Mapping Enable | Yes |
| NAT Keep Alive Enable | Yes |

> **Don't skip the NAT settings.** Once this ATA is deployed behind a remote Brume 2, it lives behind two layers of NAT (Brume LAN → remote site's WAN). Without `NAT Mapping Enable` and `NAT Keep Alive Enable` set to Yes, the ATA stops refreshing its NAT binding, the PBX loses the path, and registration silently dies — usually 30–60 seconds after a successful first registration. Set these now while you're testing locally so you don't have to revisit them after deployment.

**Click "Submit All Changes"** to save and trigger registration.

### Test Local Call

1. Verify the ATA shows **"Registered"** in its web interface
2. Pick up the phone connected to this ATA - you should hear dial tone
3. Dial the main site extension (e.g., 100)
4. The main site phone should ring - answer and verify two-way audio works
5. Hang up, then test the other direction: pick up the main site phone and dial this extension (e.g., 101)
6. Answer and verify two-way audio works in both directions

Once calls succeed in both directions, you've confirmed the extension is configured correctly. You'll reconfigure this ATA for the remote subnet after setting up the remote Brume 2.

---

# Remote Site Setup

Repeat these steps for each remote location. Complete the main site setup first!

---

## Step 8: Remote Site Brume 2 Setup

Each remote site needs its own Brume 2 in **router mode** (the default) with a unique subnet.

> **IMPORTANT: Configure before deployment!** Set up and join each remote Brume to your Tailnet **before** shipping or bringing it to the remote location. This allows you to SSH into the remote Brume 2 via Tailscale for troubleshooting after deployment.

### Pre-deployment Setup (do this at your location)

To avoid IP conflicts, configure each remote Brume while:
- The main Brume is **powered off**, OR
- On a **different network** from the main Brume (since both default to 192.168.8.1)

1. Connect Brume 2 **WAN port** to your router (needs internet for Tailscale auth)
2. Access Brume web UI (default: http://192.168.8.1)
3. Set admin password
4. Go to **Network → LAN**
5. Change the **LAN IP** to use a unique subnet:
   - Change the third octet (the "8" in 192.168.**8**.1) to a unique number
   - Example: change `192.168.8.1` to `192.168.10.1` for the first remote site
   - Use sequential numbers: 192.168.9.1, 192.168.10.1, 192.168.11.1, etc.
   - The subnet mask stays `255.255.255.0`
   - Click **Apply** - you'll be disconnected briefly as the IP changes
6. Reconnect to the Brume at its new IP (e.g., http://192.168.10.1)
7. Note this new LAN IP - it becomes the ATA's gateway

### Enable Tailscale

8. In Brume web UI: **Applications → Tailscale**
9. Click **Enable Tailscale**
10. Click the authentication link and log into your Tailscale account
11. Enable **"Allow Remote Access LAN"**
12. Enable **"Allow Remote Access WAN"**
13. Note the Tailscale IP assigned (100.x.x.x) - visible in Tailscale admin console

### Approve Route and Name Device

14. Go to https://login.tailscale.com/admin/machines
15. Find the new Brume and **approve the subnet route**
16. **Rename the device** - use the name or initials of the friend/family member where it will be deployed (e.g., "gl-mt2500-uncle-bob", "gl-mt2500-mom-dad") This is done by clicking the 3 dost on the right and selecting Edit Rout Settings.  Then you will see the route or routes to approve.

### Choosing a Subnet

Use a unique /24 subnet for each site:

| Site | Subnet | Brume LAN IP |
|------|--------|--------------|
| Main (PBX) | 192.168.1.0/24 | (WAN side, no change needed) |
| Remote Site 1 | 192.168.9.0/24 | 192.168.9.1 |
| Remote Site 2 | 192.168.10.0/24 | 192.168.10.1 |
| Remote Site 3 | 192.168.11.0/24 | 192.168.11.1 |
| Remote Site 4 | 192.168.12.0/24 | 192.168.12.1 |

> **Why start above 8?** The Brume defaults to 192.168.8.x. Using 9, 10, 11... makes it easy to remember which site is which and avoids conflicts with the default.
>
> **Two subnets to think about — yours, and theirs.** Pick a unique third octet for *your* Brume's LAN. But also be aware that the **remote site's upstream router** (the one the Brume's WAN plugs into) often hands out `192.168.0.0/24` or `192.168.1.0/24` — these are common ISP/router defaults you don't control. When the remote site's WAN happens to be on the same subnet as the PBX (`192.168.1.0/24`), the Brume's kernel sees the PBX's `/24` as directly-connected on its WAN port and tries to route PBX traffic out the WAN instead of over Tailscale. Step 9b's rc.local installs a per-host `/32` route to handle this case automatically, so you don't have to renumber the upstream router. Just be aware this is why that line is there.

---

## Step 9: Remote Site Firewall Configuration

SSH to the remote Brume to configure the firewall rules. These are specific to **remote sites** because the ATA is on the LAN side.

```bash
ssh root@192.168.X.1
```
(Replace X with your subnet number, e.g., 192.168.10.1. Password is the same as the web UI admin password you created)

### Make Filesystem Writable

GL.iNet routers use a read-only overlay filesystem by default. Run this command first to ensure your changes persist across reboots:

```bash
. /lib/functions/gl_util.sh && remount_ubifs
```

### 9a. Create UCI Firewall Zone

Run these commands to create a Tailscale firewall zone (same as main site):

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
The output should match the content you entered above.

### 9b. Configure /etc/rc.local (Remote Site)

Copy the code below to a text editor, replace `192.168.X.0/24` with your actual subnet (e.g., 192.168.10.0/24), then paste into the terminal:

> **IMPORTANT**: The closing `ENDFILE` must have **no spaces before it**.

```bash
cat > /etc/rc.local << 'ENDFILE'
# Put your custom commands here that should be executed once
# the system init finished. By default this file does nothing.

. /lib/functions/gl_util.sh
remount_ubifs

# Apply Tailscale settings - CHANGE SUBNET BELOW
tailscale up --advertise-routes=192.168.X.0/24 --accept-routes --reset

# Force PBX traffic over Tailscale, even if the remote site's WAN is
# also on 192.168.1.0/24 (common ISP default). A per-host /32 wins by
# longest-prefix-match over the connected /24 on the WAN interface.
# CHANGE THE PBX IP BELOW IF YOURS IS DIFFERENT.
( for i in 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15; do
    ip link show tailscale0 >/dev/null 2>&1 && break
    sleep 2
  done
  ip route replace 192.168.1.100/32 dev tailscale0 ) &

exit 0
ENDFILE
```

> **Why the wait loop and `ip route replace`?** `tailscale0` doesn't exist until tailscaled has started, which on these GL.iNet builds is *after* `rc.local` runs. The loop polls until the interface appears (up to 30 seconds), then installs the route. `ip route replace` is idempotent — it works whether the route already exists from a prior boot or not — so this never fails silently the way `ip route add ... 2>/dev/null || true` would.

Verify:
```bash
cat /etc/rc.local
```
The output should match the content you entered above.

### 9c. Configure /etc/firewall.user (Remote Site)

Remote sites use `br-lan` (LAN interface) because the ATA is on the LAN side.

Copy the code below to a text editor, replace `192.168.X.0/24` with your actual subnet, then paste into the terminal:

> **IMPORTANT**: The closing `ENDFILE` must have **no leading spaces**.

```bash
cat >> /etc/firewall.user << 'ENDFILE'

# allow LAN <-> Tailscale
iptables -I FORWARD -i tailscale0 -o br-lan -j ACCEPT
iptables -I FORWARD -i br-lan -o tailscale0 -j ACCEPT

# MASQUERADE traffic from LAN to Tailscale - CHANGE SUBNET BELOW
iptables -t nat -I POSTROUTING -o tailscale0 -s 192.168.X.0/24 -j MASQUERADE

# Restart Tailscale to restore its rules
/etc/init.d/tailscale restart
ENDFILE
```

Verify:
```bash
cat /etc/firewall.user
```
The output should match the content you entered above.

### 9d. Apply Settings

```bash
tailscale up --advertise-routes=192.168.X.0/24 --accept-routes --reset
/etc/init.d/firewall restart
```

---

## Step 10: Verify Tailscale Routing

On the remote Brume, test connectivity to the PBX:

```bash
# Should show route is advertised
tailscale debug prefs | grep -A3 AdvertiseRoutes

# Should return "pong from <main-brume-name>"
tailscale ping 192.168.1.100

# Should succeed with ~20-50ms latency
ping -c 3 192.168.1.100
```

### If `tailscale ping` says "no matching peer":

1. Check that the main site subnet (e.g., 192.168.1.0/24) route is approved for the **main Brume** in Tailscale admin
2. Run `tailscale up --accept-routes --reset` again on the remote Brume
3. Wait 30 seconds and retry

### If `tailscale ping` works but plain `ping` to the PBX fails:

This is the **WAN/PBX subnet collision** described in Step 8 — the remote site's upstream router is also using `192.168.1.0/24`, so the Brume's kernel routes PBX traffic out the WAN interface instead of through Tailscale. Verify the `/32` route from `/etc/rc.local` actually fired:

```bash
ip route get 192.168.1.100
# Good:  192.168.1.100 dev tailscale0 ...
# Bad:   192.168.1.100 via 192.168.1.1 dev eth0 ...
```

If you see `dev eth0`, run the route command by hand to fix it immediately:

```bash
ip route replace 192.168.1.100/32 dev tailscale0
ping -c 3 192.168.1.100
```

If that works, the rc.local pattern from Step 9b will reapply it on every boot. If `tailscale0` was missing entirely, check `tailscale status` — Tailscale itself may not have come up.

---

## Step 11: Reconfigure Remote ATA for Deployment

Connect the ATA you tested in Step 7 to the remote Brume's LAN port.

### Network Settings

No changes needed - leave the ATA on **DHCP**. The Brume will assign it an IP address in the correct subnet automatically.

To find the ATA's IP address, log into the Brume 2 web admin and check the **Clients** list.

### SIP/Line Settings

No changes needed - the SIP settings from Step 7 remain the same. The ATA will reach the PBX at 192.168.1.100 through the Tailscale tunnel.

---

## Step 12: Verify Remote SIP Registration

### Check the ATA

1. Access the ATA's web admin (e.g., http://192.168.10.100). If you are unsure of the ATA's IP you can see it under Clients in the Brume 2/Beryl AX web admin.
2. Look for registration status - usually on the main status page or under Line/SIP settings
3. Should show **"Registered"** or **"Online"**
4. If it shows "Registering...", "Failed", or "Offline", there's a connectivity issue - check the Brume's Tailscale connection first (Step 10)

### Verify on the PBX

SSH into the PBX and check registration:

```bash
asterisk -rx "pjsip show endpoints" | grep 101
# or for chan_sip:
asterisk -rx "sip show peers" | grep 101
```

Replace 101 with your extension number. Should show status "OK" or "Avail".

If not registered, wait 1-2 minutes or reboot the ATA.

---

## Step 13: Deploy to Remote Site

Once pre-configured and tested locally, deployment is simple:

1. Ship or carry the Brume 2, ATA, phone, and all the cables to the remote location
2. Connect Brume **WAN port** to the remote site's router (gets internet via DHCP)
3. Connect Brume **LAN port** to ATA (or a switch with ATA connected)
4. Connect an analog phone to the ATA
5. Power on - the Brume will automatically connect to Tailscale
6. Test by calling between the remote phone and main site phone

### Reserve the Brume's IP at the remote site (if you can)

If you have access to the remote site's home router admin, create a DHCP reservation for the Brume's WAN IP, the same way you did at the main site in Step 3. This isn't required for Tailscale to work — the Tailscale IP is stable regardless — but it preserves local-network access to the Brume if Tailscale ever fails. Without it, after a power outage the Brume may come back on a different WAN IP and the only way to find it locally is asking the homeowner to look at their router's client list.

If you don't have admin access to the remote site's router, skip this step. All your remote administration will go over Tailscale anyway.

### Remote Administration

If anything goes wrong, you can access the Brume remotely via its Tailscale IP:

1. Visit the [Tailscale admin console](https://login.tailscale.com/admin/machines)
2. Find the Brume 2 you need to access
3. Click the dropdown arrow next to the Tailscale IP address and click the copy icon
4. Make sure the Tailscale client app is running and logged in on your computer
5. Paste that IP address into a new browser tab - you're now logged into the Brume 2 web admin remotely
6. To access the ATA, go to the **Clients** tab in the Brume 2 admin to find the ATA's IP address
7. Copy that IP and paste it into a new browser tab to access the ATA's web admin

---

# Final Steps

---

## Step 14: Reboot Test

Verify everything survives a power cycle:

1. **Power off** the Brume (unplug power)
2. Wait 30 seconds
3. **Power on**
4. Wait 3-5 minutes for full boot and Tailscale connection
5. Check ATA registration on PBX:
   ```bash
   asterisk -rx "pjsip show endpoints" | grep <extension>
   ```

If registration fails after reboot, check:
- `/etc/rc.local` has the tailscale up command
- `/etc/firewall.user` has the MASQUERADE rule
- Subnet route is still approved in Tailscale admin

---

## Step 15: Make a Test Call

The ultimate test - pick up the phone and make a call!

1. Pick up the analog phone connected to the ATA
2. Listen for dial tone. **On Linksys and Cisco SPA-style ATAs**, dial tone confirms the ATA is registered — no dial tone means registration failed and you need more route troubleshooting. **On Grandstream ATAs**, you'll hear dial tone whether the ATA is registered or not, so verify registration via the ATA's web admin Status page or `pjsip show endpoints` on the PBX before assuming things are working. Try dialing an extension — fast busy or silence at that point indicates registration actually failed.
3. Dial another extension on the system
4. Verify two-way audio works (you can hear them, they can hear you)

If you don't hear dial tone:
- Check ATA registration (Step 6 for main site, Step 12 for remote)
- Verify the phone is plugged into the correct ATA port (usually "Phone 1")
- Check the ATA's line settings match the FreePBX extension

If you hear dial tone but get a fast busy signal when calling the remote extension:
- The remote extension is likely not registered with the PBX
- Check the remote ATA's registration status in its web admin
- Verify Tailscale routing (Step 10) and firewall configuration (Step 9)

If you hear dial tone but calls don't connect:
- Verify the dial plan on the ATA allows the numbers you're dialing

---

## Step 16: Export Backups

Save a backup of each Brume configuration:

1. Access Advanced Settings by logging in to the Brume 2's administration panel through your browser (use the Tailscale IP address for that location) and navigate to More Settings -> Advanced.
2. Click log into LuCi. You will be prompted to log in to the LuCi interface using your root username and password.
3. Hover over the System menu at the top nav In the LuCi interface anc click Backup/Flash Firmware.
4. Click Generate archive. This will download a .tar.gz file. This is a snapshot for all settings in the this Brume 2. Make sure to prepend the file name with the name of the location or friend/family member that this Brume 2 lives at, Example: `main-backup-GL-MT2500-2025-12-15.tar.gz`, `uncle-bob-backup-GL-MT2500-2025-12-15.tar.gz`
5. Restore Settings (if and when needed) On the same page in LuCi you can click Upload archive under the restore settings if you had to reset the Brume 2 for some reason or misconfigured it in some way. Store them in a safe place.

---

## Optional: Wireless Setup with Beryl AX (Remote Sites)

For remote sites where you don't want to place the phone right next to the router or need to avoid running cables, you can use a wireless subnet router instead: the [GL.iNet Beryl AX (GL-MT3000)](https://www.gl-inet.com/products/gl-mt3000/).

The Beryl AX connects wirelessly to the remote site's existing WiFi router, then provides a wired ethernet port for the ATA. This lets you place the phone anywhere with a power outlet and WiFi coverage.

### Setting Up Beryl AX in Repeater Mode

1. Power on the Beryl AX and connect your computer to it via ethernet or its default WiFi network (check the label on the device for the default SSID and password)
2. Access the web UI at http://192.168.8.1
3. Complete initial setup (set admin password, timezone, etc.)
4. Go to **Network → LAN** and change the LAN IP to a unique subnet (e.g., 192.168.10.1) just like with the Brume 2 - this avoids conflicts
5. Click **Apply** and reconnect to the new IP (e.g., http://192.168.10.1)
6. Go to **Internet → Repeater**
7. If you have a spare router give that router the same name and password as the one it will be connected to at your friend's or family member's home and then set up the Beryl AX to log into it, so once it is on site, it will connect directly. Confirm, if you can, if your friend or family member's existing router is 5gHz or 2.5gHz.
8. Click **Scan** to find available WiFi networks
9. Select the remote site's WiFi network and enter the password
10. Click **Join** - the Beryl will connect wirelessly to the wireless network once it is on site. For setup, just use Ethernet.
11. Reconnect and verify the connection shows as active in the Repeater section

### Configure Tailscale and Firewall

Once connected to WiFi or Ethernet, configure Tailscale on the Beryl AX the same way as the Brume 2 in Step 8, then configure the firewall as in Step 9 (Remote Site version):

12. Go to **Applications → Tailscale** and enable it
13. Authenticate with your Tailscale account
14. Enable **"Allow Remote Access LAN"** and **"Allow Remote Access WAN"**
15. SSH to the Beryl: `ssh root@192.168.X.1`
16. Configure the UCI firewall zone (Step 9a)
17. Configure `/etc/rc.local` (Step 9b - Remote Site version)
18. Configure `/etc/firewall.user` (Step 9c - Remote Site version)
19. Run: `tailscale up --advertise-routes=192.168.X.0/24 --accept-routes --reset`
20. Restart firewall: `/etc/init.d/firewall restart`
21. Approve the subnet route in Tailscale admin and rename the device
22. Export a backup

### Deployment

1. Ship or bring the pre-configured Beryl AX, ATA and phone to the remote location
2. Power it on anywhere with WiFi coverage - it will automatically connect to the WiFi network you configured
3. Connect the ATA to the Beryl's LAN port via ethernet
4. Connect an analog phone to the ATA
5. The Beryl connects to WiFi → Tailscale → PBX automatically

> **Note:** The Beryl AX remembers the WiFi network credentials. If the remote site's WiFi password changes, you'll need to SSH in via Tailscale and update the Repeater settings, or have someone on-site temporarily connect to the Beryl's LAN to access the web UI.

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

### Essential Commands (Brume 2/Beryl AX)

Check Tailscale status:
```bash
tailscale status
```

Check advertised routes:
```bash
tailscale debug prefs | grep -A3 AdvertiseRoutes
```

Test Tailscale routing to an IP:
```bash
tailscale ping <ip-address>
```

Check firewall rules:
```bash
iptables -L FORWARD -n -v | head -10
iptables -t nat -L POSTROUTING -n -v | grep MASQ
```

Restart Tailscale:
```bash
/etc/init.d/tailscale restart
```

Restart firewall (also runs firewall.user):
```bash
/etc/init.d/firewall restart
```

### Essential Commands (RasPBX)

Check registered extensions:
```bash
asterisk -rx "pjsip show endpoints"
```

Or for chan_sip:
```bash
asterisk -rx "sip show peers"
```

Monitor SIP activity in real-time (Control+C to exit):
```bash
asterisk -rx "pjsip set logger on"
```

Live console with verbosity - more v's = more detail (type "quit" to exit):
```bash
asterisk -rvvvv
```

Check active calls:
```bash
asterisk -rx "core show calls"
```

View recent call history:
```bash
asterisk -rx "core show channels verbose"
```

Restart Asterisk (if needed):
```bash
systemctl restart asterisk
```

### Firewall Configuration Differences

| Setting | Main Site | Remote Site |
|---------|-----------|-------------|
| Interface | `eth0` (WAN) | `br-lan` (LAN) |
| Reason | PBX is on WAN side | ATA is on LAN side |
| rc.local route | Not needed | Adds route to PBX |

### Example Network Layout

| Device | Tailscale IP | LAN Subnet | Purpose |
|--------|--------------|------------|---------|
| Main Brume | 100.x.x.1 | 192.168.1.0/24 | PBX site gateway |
| Remote Brume 1 | 100.x.x.2 | 192.168.10.0/24 | Remote house 1 |
| Remote Brume 2 | 100.x.x.3 | 192.168.11.0/24 | Remote house 2 |
| Remote Brume 3 | 100.x.x.4 | 192.168.12.0/24 | Remote house 3 |

---

## Revision History

| Date | Change |
|------|--------|
| 2026-04-27 | Added `NAT Mapping Enable` / `NAT Keep Alive Enable` to ATA SIP settings (Steps 5, 7). Hardened Step 9b's `/etc/rc.local` route pattern (wait loop + `ip route replace`). Clarified the WAN-vs-LAN subnet warning in Step 8. Added Step 10 diagnostic for the WAN/PBX subnet collision case. Added Grandstream dial-tone caveat (Steps 6, 15). Broadened Step 3 to also reserve the Brume's WAN IP, with matching note in Step 13. |
