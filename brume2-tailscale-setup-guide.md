Phreephoneing: A Free, Private, Encrypted Phone System with Raspberry Pi and Analog Phones

What if you could pick up an old-school telephone in your house, call a friend's house across town, the country or world, and have that call travel over your existing internet connection, fully encrypted, with no monthly bill from a phone company? That's the basic idea behind this project. Phreephoneing is a free, private phone system built from a Raspberry Pis, Analog Telephone Adapters (ATAs), subnet routers, and a main router that creates an encrypted mesh between locations.

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

- GL.iNet Brume 2 (GL-MT2500) router, one for each remote line and one for the main site that will be on the same local network as the PBX https://www.gl-inet.com/products/gl-mt2500/
- Tailscale account (free tier is adequate) https://tailscale.com/
- Tailscale client installed on your admin computer** - download from https://tailscale.com/download and sign in with the same Tailscale account. This allows you to SSH into any Brume via its Tailscale IP and also into each ATA admin. You will be on the same Tailnet.
- A Raspberry Pi 3, 4, or 5 (not Zero) with RasPBX image written to the microSD card http://www.raspbx.org/downloads/
- An ATA device for each line (Cisco SPA, Linksys PAP2T, Grandstream HT802, etc.) https://www.ebay.com/sch/i.html?_nkw=analog+telephone+adapter
- An old touch tone analog telephone at each location you want to call or you want to call you.

---

## Step 1A: Main Site Brume 2 Setup

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

### Configure Main Brume (via SSH)

13. SSH to the main Brume: `ssh root@192.168.8.1` (the default LAN IP, password is same as web UI) or use the Tailscale IP once it's connected
14. Create the UCI firewall zone (see Step 3)
15. Configure `/etc/rc.local` using the **Main Site version** (see "Main Site Brume Setup" section at end of guide)
16. Configure `/etc/firewall.user` using the **Main Site version** (uses `eth0` instead of `br-lan`)
17. Apply settings: `tailscale up --advertise-routes=192.168.1.0/24 --accept-routes --reset`
18. Restart firewall: `/etc/init.d/firewall restart`

### Approve Route, Name, and Verify

19. Go to https://login.tailscale.com/admin/machines
20. Find the main Brume and **approve the subnet route** (192.168.1.0/24)
21. **Rename the device** - click the three-dot menu → "Edit machine name" → name it something like "gl-mt2500-main" or "gl-mt2500-pbx-site" to identify it easily
22. **Export a backup** - In the Brume web UI, go to **System → Backup / Restore → Export Configuration**. Save the `.tar.gz` file (e.g., `gl-mt2500-main-2024-01-15.tar.gz`)

> **Key difference from remote Brumes:** The main Brume's PBX is on its WAN side (e.g., 192.168.1.0/24), not LAN side. This means the firewall rules use `eth0` instead of `br-lan`. See the "Main Site Brume Setup" section at the end of this guide for the specific `/etc/rc.local` and `/etc/firewall.user` contents.


## Step 1B: Remote Site Brume 2 and ATA Setup

Each remote site needs its own Brume 2 in **router mode** (the default) with a unique subnet.

> **IMPORTANT: Configure before deployment!** Set up and join each remote Brume to your Tailnet **before** shipping or brining it to the remote location. This allows you to SSH into the remote Brume 2 via Tailscale (computer or smartphone clients) for any troubleshooting or configuration changes after deployment.

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

**Enable Tailscale:**

8. In Brume web UI: **Applications → Tailscale**
9. Click **Enable Tailscale**
10. Click the authentication link and log into your Tailscale account
11. Enable **"Allow Remote Access LAN"**
12. Enable **"Allow Remote Access WAN"**
13. Note the Tailscale IP assigned (100.x.x.x) - visible in Tailscale admin console

**Configure via SSH:**

14. SSH to the Brume: `ssh root@192.168.X.1` using the new LAN IP (password is same as web UI)
15. Create the UCI firewall zone (see Reference A below)
16. Configure `/etc/rc.local` for remote site (see Reference B - use the **Remote Site version**)
17. Configure `/etc/firewall.user` for remote site (see Reference C - use the **Remote Site version**)
18. Apply settings: `tailscale up --advertise-routes=192.168.X.0/24 --accept-routes --reset` (use your subnet)
19. Restart firewall: `/etc/init.d/firewall restart`

**Approve, name, verify, and backup:**

20. Go to https://login.tailscale.com/admin/machines
21. Find the new Brume and **approve the subnet route**
22. **Rename the device** - use the name or initials of the friend/family member where it will be deployed (e.g., "gl-mt2500-uncle-bob", "gl-mt2500-mom-dad", "gl-mt2500-j")
23. **Verify Tailscale connectivity** - SSH into the Brume using its Tailscale IP (`ssh root@100.x.x.x`, same password you set for web admin) WHILE your computer's Tailscale client is on AND logged in (login required for each session) and run `tailscale ping <PBX-IP>`. You should get "pong" back from the main Brume. Get the Brume 2's Tailnet IP address from the Tailscale.com admin.
24. **Export a backup** - In the Brume web UI: **System → Backup / Restore → Export Configuration**. Save the `.tar.gz` file (e.g., `gl-mt2500-uncle-bob-2024-01-15.tar.gz`)

### Deployment at Remote Site

Once pre-configured, deployment is simple:

1. Ship or carry the Brume 2, ATA, phone, and all the cables you need to the remote location
2. Connect Brume **WAN port** to the remote site's router (gets internet via DHCP)
3. Connect Brume **LAN port** to ATA (or a switch with ATA connected)
4. Connect an analog phone to the ATA
5. Power on - the Brume will automatically connect to Tailscale
6. Configure the ATA (Step 4) and verify registration (Step 5)

If anything goes wrong, you can SSH to the Brume via its Tailscale IP from the computer that you installed the Tailscale client app. Make sure Tailscale is turned on on your computer, and that you have logged in, which you must do for each session.

### Choosing a Subnet

Use a unique /24 subnet for each site. Use sequential numbers starting above 8 (since 192.168.8.x is the Brume default):

| Site | Subnet | Brume LAN IP |
|------|--------|--------------|
| Main (PBX) | 192.168.1.0/24 | (WAN side, no change needed) |
| Remote Site 1 | 192.168.9.0/24 | 192.168.9.1 |
| Remote Site 2 | 192.168.10.0/24 | 192.168.10.1 |
| Remote Site 3 | 192.168.11.0/24 | 192.168.11.1 |
| Remote Site 4 | 192.168.12.0/24 | 192.168.12.1 |

> **Why start above 8?** The Brume defaults to 192.168.8.x. Using 9, 10, 11... makes it easy to remember which site is which and avoids conflicts with the default. Also avoid 192.168.0.x and 192.168.1.x as these are common home network subnets that may conflict at remote sites.

---

## Step 1C: Wireless Option with Beryl AX

If you don't want to place the phone right next to the router or need to avoid running ethernet or phone cables to where you want the phone, you can use a wireless subnet router instead: the [GL.iNet Beryl AX (GL-MT3000)](https://www.gl-inet.com/products/gl-mt3000/).

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

Once connected to WiFi or Ethernet, configure Tailscale on the Beryl AX the same way as the Brume 2 above in section 1B:

11. Go to **Applications → Tailscale** and enable it
12. Authenticate with your Tailscale account
13. Enable **"Allow Remote Access LAN"** and **"Allow Remote Access WAN"**
14. SSH to the Beryl: `ssh root@192.168.X.1`
15. Configure the UCI firewall zone (Reference A)
16. Configure `/etc/rc.local` (Reference B - Remote Site version)
17. Configure `/etc/firewall.user` (Reference C - Remote Site version)
18. Run: `tailscale up --advertise-routes=192.168.X.0/24 --accept-routes --reset`
19. Restart firewall: `/etc/init.d/firewall restart`
20. Approve the subnet route in Tailscale admin and rename the device
21. Export a backup

### Deployment

1. Ship or bring the pre-configured Beryl AX, ATA and phone to the remote location
2. Power it on anywhere with WiFi coverage - it will automatically connect to the WiFi network you configured
3. Connect the ATA to the Beryl's LAN port via ethernet
4. Connect an analog phone to the ATA
5. The Beryl connects to WiFi → Tailscale → PBX automatically

> **Note:** The Beryl AX remembers the WiFi network credentials. If the remote site's WiFi password changes, you'll need to SSH in via Tailscale and update the Repeater settings, or have someone on-site temporarily connect to the Beryl's LAN to access the web UI.

---

# Reference Sections

The following sections contain the detailed commands referenced by Steps 1A, 1B, and 1C above. You must do each of these while ssh'ed into the Brume 2 or Beryl AX for the phone system to work.

---

## Reference A: UCI Firewall Zone

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

## Reference B: Configure /etc/rc.local

This ensures Tailscale settings persist after reboot.

> **IMPORTANT**: When typing these commands, the closing `ENDFILE` must have **no spaces before it**. If your terminal adds leading spaces when you paste, use the arrow keys and backspace to remove them before pressing Enter.

Replace `192.168.X.0/24` with your actual subnet and `192.168.1.100` with your PBX IP:

```bash
cat > /etc/rc.local << 'ENDFILE'
# Put your custom commands here that should be executed once
# the system init finished. By default this file does nothing.

. /lib/functions/gl_util.sh
remount_ubifs

# Wait for Tailscale to be ready
sleep 10

# Apply Tailscale settings - CHANGE SUBNET BELOW
tailscale up --advertise-routes=192.168.X.0/24 --accept-routes --reset

# Add explicit route to PBX - CHANGE IP BELOW
ip route add 192.168.1.100/32 dev tailscale0 2>/dev/null || true

exit 0
ENDFILE
```

Verify:
```bash
cat /etc/rc.local
```

---

## Reference C: Configure /etc/firewall.user

This adds the MASQUERADE rule required for Tailscale to forward LAN traffic.

> **IMPORTANT**: Same as above - the closing `ENDFILE` must have **no leading spaces**.

Replace `192.168.X.0/24` with your actual subnet:

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

---

# Post-Setup Steps

After completing Step 1A (main Brume) and Step 1B (remote Brumes), continue with these steps.

---

## Step 2: Verify Tailscale Routing

On the new Brume, test connectivity to the PBX:

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
2. Run `tailscale up --accept-routes --reset` again on the new Brume
3. Wait 30 seconds and retry

---

## Step 3: Create Extension in FreePBX

Before configuring the ATA, create the SIP extension in FreePBX:

1. Log into FreePBX web interface on your local network (check your router for the PBX IP)
2. Go to **Applications → Extensions**
3. Click **Add Extension** → **Add New PJSIP Extension** (or SIP if using chan_sip)
4. Enter:
   - **User Extension**: Extension number (e.g., 101, 102)
   - **Display Name**: Description (e.g., "Mom and Dad")
   - **Secret**: Note the auto-generated password or set your own
5. Click **Submit**
6. Click the **red "Apply Config" button** at the top
7. Copy the **Secret** (password) - you'll need it for the ATA
8. Repeat steps 2 through 7 for each additional remote line

---

## Step 4: Configure ATA

Access the ATA's web interface and configure:

### Network Settings
| Setting | Value |
|---------|-------|
| Connection Type | **Static IP** |
| IP Address | 192.168.X.100 (example) |
| Subnet Mask | 255.255.255.0 |
| Default Gateway | 192.168.X.1 (Brume LAN IP) |
| Primary DNS | 8.8.8.8 or Brume LAN IP |

### SIP/Line Settings
| Setting | Value |
|---------|-------|
| SIP Proxy | 192.168.1.100 (PBX IP) |
| SIP Port | 5060 |
| Register | Yes |
| User ID | Extension number (e.g., 101) |
| Auth ID | Same as User ID |
| Password | SIP secret from FreePBX |

If you will use both lines on a 2 line ATA the 2nd line should use SIP port 5061

**Click "Submit All Changes"** to save and trigger registration.

---

## Step 5: Verify SIP Registration

### Check the ATA first

Before expecting a dial tone, verify registration in the ATA's web interface:

1. Access the ATA's web admin (e.g., http://192.168.10.100)  If you are unsure of the ATA's IP you can see it under Clients in the Brume 2/Beryl AX web admin
2. Look for registration status - usually on the main status page or under Line/SIP settings
3. Should show **"Registered"** or **"Online"**
4. If it shows "Registering...", "Failed", or "Offline", there's a connectivity issue - check the Brume's Tailscale connection first

### Verify on the PBX

Once the ATA shows registered, confirm on the PBX (Raspberry Pi):

```bash
asterisk -rx "pjsip show endpoints" | grep <extension>
# or for chan_sip:
asterisk -rx "sip show peers" | grep <extension>
```
Replace <extension> with just the number, like: `asterisk -rx "sip show peers" | grep 101`

Should show the extension with status "OK" or "Avail".

If not registered, wait 1-2 minutes or reboot the ATA.

---

## Step 6: Reboot Test

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

## Step 7: Make a Test Call

The ultimate test - pick up the phone and make a call!

1. Pick up the analog phone connected to the ATA
2. Listen for dial tone (confirms ATA is working and registered.  If there is no dialtone it is not registered wit the PBX, needs more route troubleshooting)
3. Dial another extension on the system
4. Verify two-way audio works (you can hear them, they can hear you)

If you don't hear dial tone:
- Check ATA registration (Step 5)
- Verify the phone is plugged into the correct ATA port (usually "Phone 1")
- Check the ATA's line settings match the FreePBX extension

If you hear dial tone but calls don't connect:
- Verify the dial plan on the ATA allows the numbers you're dialing
- Check FreePBX outbound routes if calling external numbers

---

## Step 8: Export Backup (if not already done)

Save a backup of the Brume configuration:

1. Access Brume web UI (via LAN or Tailscale IP)
2. Go to **System → Backup / Restore**
3. Click **Export Configuration**
4. Save the `.tar.gz` file with a descriptive name:
   - Example: `brume-sitename-2024-01-15.tar.gz`

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

### Example Network Layout

| Device | Tailscale IP | LAN Subnet | Purpose |
|--------|--------------|------------|---------|
| Main Brume | 100.x.x.1 | 192.168.1.0/24 | PBX site gateway |
| Remote Brume 1 | 100.x.x.2 | 192.168.10.0/24 | Remote house 1 |
| Remote Brume 2 | 100.x.x.3 | 192.168.11.0/24 | Remote house 2 |
| Remote Brume 3 | 100.x.x.4 | 192.168.12.0/24 | Remote house 3 |

---

## Main Site Brume Setup

The main site Brume (at the PBX location) has slightly different configuration since its WAN is on the same network as the PBX.

### /etc/firewall.user (Main Brume)

```bash
# MASQUERADE traffic from WAN subnet to Tailscale
iptables -t nat -I POSTROUTING -o tailscale0 -s 192.168.1.0/24 -j MASQUERADE

# FORWARD rules for eth0 <-> tailscale0 (WAN to Tailscale)
iptables -I FORWARD -i tailscale0 -o eth0 -j ACCEPT
iptables -I FORWARD -i eth0 -o tailscale0 -j ACCEPT

# Restart Tailscale to restore its rules
/etc/init.d/tailscale restart
```

### /etc/rc.local (Main Brume)

```bash
. /lib/functions/gl_util.sh
remount_ubifs

# Wait for Tailscale to be ready
sleep 10

# Apply Tailscale settings - advertise the PBX subnet
tailscale up --advertise-routes=192.168.1.0/24 --accept-routes --reset

exit 0
```

Note: Main Brume uses `eth0` (WAN) instead of `br-lan` (LAN) because the PBX is on the WAN side.
