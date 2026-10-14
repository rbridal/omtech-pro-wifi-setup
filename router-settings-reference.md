# Router settings reference (sanitized capture from a working Pro 2440)

Read-only capture of a unit that was working. Addresses were changed to **example** values (`192.168.0.x`). SSIDs, keys, MACs and admin credentials were removed.

**Device:** web title "ZL-LINK CPE". HTTP Basic realm "PSG1218" (Phicomm K2 board profile). Padavan (rt-n56u) fork, firmware 3.4.2.0. The kernel was built in 2023.
**Hardware:** MT7620N SoC, 2.4 GHz 2x2, 64 MB RAM, 8 MB SPI flash. There is an MT7612 5 GHz radio, but it never comes up (`MT7620 PPLL unlock, cannot enable PCIe!`). The switch has 5 ports (WAN + LAN1-4). SSH is in the build but isn't listening.
**MAC OUI:** Shenzhen City Linwlan Technology Co. (the full MAC is omitted).

| Page | Setting | Value on working unit |
|---|---|---|
| Administration → Operation Mode | Mode | Access Point (`sw_mode=3`, `wan_route_x=IP_Bridged`). NAT, firewall and DHCP server are off. |
| LAN → LAN IP | Get LAN IP automatically | No |
| | IP / mask | `192.168.0.200` / `255.255.255.0` *(example)* |
| | Gateway | `192.168.0.1` *(example)* |
| | DNS | public resolvers or your router |
| Wireless 2.4GHz → Bridge | Wireless Operation Mode | AP-Client (AP is disabled) |
| | AP-Client role | LAN bridge (`rt_sta_wisp=0`) |
| | Upstream SSID / key | *your 2.4 GHz SSID* / *your key* |
| | Auth / cipher | WPA2-Personal / AES |
| | Channel | Autoseek (`rt_sta_auto=1`) |
| Wireless 2.4GHz → General | Country code | shipped as `DB` (debug); set it to your country |
| | Mode / width | b/g/n mixed, 20 MHz |
| Ruida panel (SysCfg → Ethernet) | IP | `192.168.0.201` *(example; same /24 as your hosts)* |

**Port status:** LAN4 is the Ruida (100 Mbps full duplex). LAN3 is the external "Ethernet" jack. WAN, LAN1 and LAN2 are not linked.
**System log:** `hw bridge: LLLLL`, `br0 = eth2.1 + apcli0`, and the NVRAM integrity check passes.
**Persistence:** the Commit button was hidden (nothing pending), and the config survives power cycles.
**MAT:** your main router sees the router IP and the Ruida IP with the **same MAC**.
**Ruida:** RDC6445-series. ICMP answers, and every TCP port probed was closed. It uses UDP 50200 (controller) and 40200 (host).
