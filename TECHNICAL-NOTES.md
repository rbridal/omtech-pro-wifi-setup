# Technical notes: OMTech Pro internal Wi-Fi router

*For network-savvy readers. The friendly step-by-step is in [GUIDE.md](GUIDE.md); this file explains what is actually going on, why each step matters, and the known quirks. Example addresses use `192.168.0.x`; substitute your own /24.*

## The architecture

**Hardware and firmware.** The internal unit identifies itself as **"ZL-LINK CPE"**. It's a rebranded **Padavan** (rt-n56u fork) build, firmware 3.4.2.0 (kernel built 2023), using the **Phicomm K2 / PSG1218** board profile (that's the HTTP Basic realm). It has a MediaTek **MT7620N** SoC (2.4 GHz, 2×2), 64 MB RAM, 8 MB SPI flash, and five switch ports (WAN + LAN1–4). The board has an MT7612 5 GHz radio on PCIe, but the log shows `MT7620 PPLL unlock, cannot enable PCIe!`, so **5 GHz never initializes. The uplink must be 2.4 GHz.** The web UI is HTTP only, with Basic auth, default **`admin` / `admin`**. SSH is in the build but wasn't listening. The router is powered from the laser, so a laser power-cycle reboots it.

**Port map (confirmed on a Pro 2440):**

| Switch port | Connected to |
|---|---|
| LAN4 | Ruida controller (RDC6445-series, links at 100 Mbps full duplex) |
| LAN3 | the laser's external "Ethernet" jack |
| LAN1, LAN2 | unused |
| WAN | unused |

The external jack is a **LAN** port, not WAN. That one fact explains both warnings.

**Factory state.** Operation mode is "Wireless Router Mode (Default)": NAT, firewall, UPnP and **DHCP server on**. It runs its own AP, `OMTECH PRO 2440` / `OMTECH PRO 3655`, with PSK `123456abc` (published in the manual). The Ruida ships at `192.168.1.20`. The LAN/management IP is **probably `192.168.1.1`/24**. That's the Padavan default, the firmware's own factory-reset string says so, it fits the manual's 192.168.1.x addressing, and one owner remembers it. But it hasn't been verified on an out-of-box unit.

**Target state (as running on the working unit):**

- **Operation Mode = Access Point** (`sw_mode=3`, `wan_route_x=IP_Bridged`). NAT, firewall, UPnP and the DHCP server are off (the DHCP Server tab disappears). WAN is bridged into the LAN (`hw bridge: LLLLL`).
- **LAN IP static** (`lan_proto_x` = no). Example: `192.168.0.200/24`, gateway and DNS set to your router. Gateway and DNS only matter for the router's own traffic (NTP and the like), not for forwarding.
- **2.4 GHz Bridge: Wireless Operation Mode = AP-Client (AP is disabled)**, with **AP-Client role = LAN bridge** (`rt_sta_wisp=0`). WPA2-Personal/AES, Autoseek channel on (`rt_sta_auto=1`) so it follows the upstream AP's channel. Result: `br0 = eth2.1 + apcli0`, one flat L2 segment. Setup uses AP-Client + AP temporarily, so the local AP is a fallback until the uplink is proven.
- **LAN bridge vs. WISP.** WISP ("WAN (Wireless ISP)") makes `apcli0` the WAN interface. That puts the Ruida on a separate NATed subnet. The Ruida has no gateway setting and can't reply off-subnet, so LightBurn on the main LAN can't reach it without contortions. Use LAN bridge.
- **Ruida:** static IP set on the HMI (Menu → SysCfg → Ethernet → IP Setting → OK, then power-cycle). The Ruida takes an **IP only, with no mask or gateway**. It assumes a /24, so the host has to be in the same /24 ("only the last section is different," as the panel puts it). On a /16 or other non-/24 LAN, keep the controller and LightBurn hosts within one /24. The protocol is **UDP: the controller listens on 50200, and the host side uses 40200.** No TCP services answered a probe, but ICMP echo works. Host firewalls must allow LightBurn's UDP traffic.

**MAC address translation (MAT).** An 802.11 station in 3-address mode can only send frames with its own MAC. MediaTek's AP-Client therefore does MAT (the firmware's own help text says so), rewriting the MACs of wired devices behind it to the `apcli0` MAC. So **the router's IP and the Ruida's IP show the same MAC** in your router's ARP and DHCP tables. Expect this, and don't let an IDS "duplicate MAC" alert surprise you. A MAC-keyed DHCP reservation can't tell them apart, so either exclude both IPs from the pool or reserve by IP if your router allows it. Anything else plugged into LAN1–3 shares the MAC too. Some multicast, IPv6 and DHCP-relay behavior through MAT can be odd. Static IPv4 unicast (the Ruida's whole world) works fine.

**Region code.** The working unit shipped with country code **`DB` (Debug, all channels)**. That isn't a legal regulatory domain anywhere. Set your real country on Wireless 2.4GHz → General. If your upstream AP sits on a channel your country doesn't allow (12–13 outside the US), Autoseek will fail to follow it after the change.

## Risks, quirks, and open questions

- **Rogue DHCP (factory mode).** Because the jack is LAN3, cabling it to your LAN in router mode puts a live DHCP server (and a `192.168.1.1` gateway) straight onto your network. If your LAN is also `192.168.1.0/24`, you get a duplicate-gateway IP conflict on top of that. Owners have reported exactly this ("built in router taking over my home network").
- **L2 loop (bridged mode).** After bridging, LAN3 and `apcli0` are on the same bridge. Cabling the jack to the same LAN gives you a wireless path plus a wired path into one broadcast domain. Don't count on STP being enabled on `br0`, and MAT makes upstream MAC learning flap anyway. A directly cabled laptop is just another bridged port, which is fine.
- **Single-session admin.** Padavan locks the web UI to one client IP at a time. A second client gets "please log out the other user first" (请先注销另一用户后再尝试登录) and may see the UI as unresponsive. Use Logout (注销) or wait for the session to time out. Automated scrapers and status dashboards will hold the lock.
- **Persistence.** In testing, settings survived laser power-cycles with no explicit Commit (提交). The Commit button was hidden (`display:none`, so nothing was pending) and the NVRAM integrity check passed. Clicking Commit when it's shown is still cheap insurance before cutting power.
- **Defaults to change:** the admin password (`admin`/`admin`, HTTP Basic over plain HTTP) and the local AP (disable it, or at least change the published `123456abc` PSK). Leaving AP-Client + AP running exposes a second door into your LAN.
- **Upstream security.** Tested against WPA2-Personal/AES. A WPA3-only SSID probably won't work with this firmware; WPA2/WPA3 transition mode should. (That's an expectation, not something we've tested.)
- **Unconfirmed:**
  - the factory management IP (likely `192.168.1.1`);
  - where the management address lands *immediately* after switching to AP mode. Padavan AP mode can default the LAN to DHCP client. With no upstream yet, it may stay on `192.168.1.1`, or not. If you lose it, put a host statically on `192.168.1.0/24` and scan (`arp-scan`/`nmap -sn`), or check your main router's lease table if the uplink is already up;
  - the hardware reset method. The usual Padavan behavior is to hold RESET ~10 s while running. On this hardware family, holding it at power-on may enter bootloader recovery instead. The button's exact location in the cabinet isn't documented.
  - A possibly simpler order (untested): set the LAN IP *before* switching to AP mode, while DHCP still hands the laptop an address. That might avoid the manual laptop IPs.

**What the OMTECH manual (Pro 2440, 2025 edition) gets wrong or leaves out:**
- §2.3.2 (p.18) says the Ethernet port "connects to your control computer through an Ethernet cable." §3.8.4 (p.47) says "DO NOT connect your computer… the Ethernet port is for internet connection." Both are misleading, because it's LAN3 on the internal router.
- It only describes joining the laser's hotspot and using `192.168.1.20`, which takes the laptop off the internet. Its RDWorks screenshot shows `192.168.1.100` while the text says `192.168.1.20`.
- It never mentions the router, its admin page or login, the operation modes, DHCP, client/bridge mode, or how to reset it.
- §6.7.1 (p.92) documents the panel IP setting but doesn't connect it to the host's subnet.


## Why each step matters

| Guide part | Why |
|---|---|
| Region code | Factory `DB` (debug) is not a legal regulatory domain; set your country before bridging. |
| AP mode | Kills NAT/firewall/DHCP server so the unit stops behaving like a second router (the rogue-DHCP risk). |
| Static LAN IP | In AP mode the management IP must live on your LAN so you can still reach the admin UI. |
| AP-Client, role = LAN bridge | Makes `br0 = eth2.1 + apcli0`, one flat L2 segment, so the Ruida sits directly on your LAN. WISP would NAT it away. |
| AP-Client + AP first, then disable AP | Keeps the local hotspot as a fallback until the uplink is proven, then closes the published-PSK door. |
| Change admin password | Default `admin`/`admin` over plain HTTP. |
| Panel IP in same /24 | The Ruida has no mask/gateway setting; it can only answer hosts in its own /24. |
| Exclude IPs from DHCP pool | MAT gives both IPs the same MAC, so MAC-based reservations do not work. |

See also [router-settings-reference.md](router-settings-reference.md) for a sanitized capture of a working unit.
