# 🖧 Proxmox 8.4 Wi-Fi Setup on Dell Optiplex 3060 MFF

This guide details how to configure **Proxmox VE 8.4** on a Dell Optiplex 3060 MFF using a **TP-Link Archer T3U Wi-Fi dongle**, enabling VM network access through a bridge and `dnsmasq`.

---

## 🖥️ Hardware

- **Model**: Dell Optiplex 3060 MFF  
- **CPU**: Intel i5-8500T  
- **RAM**: 16GB  
- **Storage**: 500GB NVMe (WD Black SN750)  
- **Wi-Fi**: TP-Link Archer T3U USB dongle  

---

## 🔌 Initial Setup

1. Install Proxmox 8.4 from the official ISO using default settings.
2. Plug in an Ethernet (ETH) cable to get internet access for the first time.
3. Check your current IP (example: `192.168.0.122`). You can display this in the login screen by editing:

   ```bash
   vim /etc/issue
   ```

   Example content:

   ```
   https://192.168.0.122:8006/
   ```

---

## 📦 Required Packages

Install networking tools:

```bash
apt update
apt install vim iwconfig wireless-tools wpasupplicant iw dnsmasq
```

Disable and stop `wpa_supplicant`:

```bash
systemctl disable wpa_supplicant
systemctl stop wpa_supplicant
```

(Optional) Move auto-start hook if it’s interfering:

```bash
mv /etc/network/if-pre-up.d/wpasupplicant ./
```

---

## 🛠️ Configuration

### Configure `dnsmasq`

Edit `/etc/dnsmasq.conf`:

```bash
vim /etc/dnsmasq.conf
```

Add:

```ini
interface=vmbr0
dhcp-range=10.10.1.1,10.10.1.255,255.255.255.0,12h
```

---

### Scan Wi-Fi and Create WPA Config

Find available networks:

```bash
iw dev [WIFI_INTERFACE_NAME] scan | grep SSID
```

Generate WPA credentials:

```bash
wpa_passphrase SSIDNAME PASSWORD >> /etc/wpa_supplicant/wpa_supplicant.conf
```

Ensure `/etc/wpa_supplicant/wpa_supplicant.conf` looks like:

```ini
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=PL

network={
    ssid="SSIDNAME"
    #psk="PASSWORD"
    psk=9************************************************************1d0
}
```

Restart networking:

```bash
systemctl restart networking
```

---

## 🌐 Networking Setup

### Test Wi-Fi Connectivity

You should have IP already

```bash
ip a
wlxd03745b0c19c: state UP, inet 192.168.0.124/24
```

Check if router is reachable:

```bash
ping [Your router IP, eg. 192.168.0.1]
```

If no IP is assigned:

```bash
dhclient [WIFI_INTERFACE_NAME]
```
it will force ip assigment from DHCP server


⚠️ You may lose Ethernet connection — physical console access may be required.

---

### Configure `/etc/network/interfaces`

```bash
vim /etc/network/interfaces
```

Replace contents with:

```ini
auto lo
iface lo inet loopback

auto [WIFI_INTERFACE_NAME]
iface [WIFI_INTERFACE_NAME] inet dhcp
    wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf

iface enp1s0 inet manual

auto vmbr0
iface vmbr0 inet static
    address 10.10.1.1/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    post-up echo 1 > /proc/sys/net/ipv4/ip_forward
    post-up   iptables -t nat -A POSTROUTING -s '10.10.1.1/24' -o [WIFI_INTERFACE_NAME] -j MASQUERADE
    post-down iptables -t nat -D POSTROUTING -s '10.10.1.1/24' -o [WIFI_INTERFACE_NAME] -j MASQUERADE
    post-up   iptables -t raw -I PREROUTING -i fwbr+ -j CT --zone 1
    post-down iptables -t raw -D PREROUTING -i fwbr+ -j CT --zone 1

source /etc/network/interfaces.d/*
```

Apply changes:

```bash
ifreload -a
```

---

### Hostname/IP Configuration

Edit `/etc/hosts` to reflect your current IP:

```ini
192.168.0.122 proxmox01.local proxmox01
```

---

## 🧪 Troubleshooting

Check service status and logs:

```bash
systemctl status networking.service
journalctl -u networking.service
dmesg -e
```

---

### Routing Example Output

```bash
ip r
```

Example:

```
default via 192.168.0.1 dev [WIFI_INTERFACE_NAME]
10.10.1.0/24 dev vmbr0 proto kernel scope link src 10.10.1.1
192.168.0.0/24 dev [WIFI_INTERFACE_NAME] proto kernel scope link src 192.168.0.124
```

```bash
ip a
```

Example:

```
1: lo: ...
2: enp1s0: state DOWN
3: wlxd03745b0c19c: state UP, inet 192.168.0.124/24
4: vmbr0: inet 10.10.1.1/24
```

---

## 💡 Notes for Proxmox GUI

1. Use **DHCP** for IPv4 when creating VM/LXC containers.
2. Default bridge is `vmbr0` (unless changed).
3. Ensure your firewall and NAT allow proper routing.

---

## 🔁 Alternative DHCP Approach

Proxmox’s official DHCP/NAT zone guide:  
https://pve.proxmox.com/wiki/Setup_Simple_Zone_With_SNAT_and_DHCP

---

## 🌐 Web Resources

- https://github.com/ThomasRives/Proxmox-over-wifi  
- https://serverfault.com/questions/1083280/configure-dnsmasq-to-give-out-addresses-in-different-ranges  
- https://www.linux.com/topic/networking/dns-and-dhcp-dnsmasq/  
- https://pve.proxmox.com/wiki/Setup_Simple_Zone_With_SNAT_and_DHCP  
- https://blog.vivekkaushik.com/guide-how-to-configure-proxmox-with-wifi  
