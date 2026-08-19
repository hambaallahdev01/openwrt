# Arsitektur Hybrid Multi-WAN Gateway OpenWrt & WireGuard VPS Split-Tunneling

Dokumentasi implementasi gateway jaringan berbasis **OpenWrt 23.05 (HG680P)** yang terintegrasi dengan **MikroTik Router**, **Dual-WAN (Fiber Optic + 4G Modem USB)**, dan **VPS Vultr Singapura (WireGuard)** dengan mekanisme *Selective Routing / Split-Tunneling*, *Smart Queue Management (SQM CAKE)*, serta *DNS Enforcement*.

---

## 1. Ringkasan Arsitektur & Kebijakan Trafik

                      ┌───────────────────────────┐
                      │   Klien / Pelanggan LAN   │
                      └─────────────┬─────────────┘
                                    │
                                    ▼
                      ┌───────────────────────────┐
                      │     MikroTik Core LAN     │
                      │   (Bridge & NAT Gateway)  │
                      └─────────────┬─────────────┘
                                    │ Trunk VLAN (eth0)
                                    ▼
    ┌──────────────────────────────────────────────────────────────┐
    │                 OpenWrt Gateway (STB HG680P)                 │
    │                                                              │
    │  [DNS Hijack]   ──► Cloudflare Families (1.1.1.3 / 1.0.0.3)  │
    │  [SQM CAKE]     ──► Anti-Bufferbloat 33 Mbps di Fiber        │
    │  [Dynamic Set]  ──► dnsmasq-full + nftset (Search Engine)    │
    └──────────────┬───────────────┬────────────────┬──────────────┘
                   │               │                │
      [Traffic Game / ICMP]  [Search Engine]   [Zoom, Speedtest,
                   │               │            Browsing Umum]
                   │               │                │
                   ▼               ▼                ▼
         ┌──────────────────┐  ┌──────────┐  ┌─────────────┐
         │ WAN 1: Fiber     │  │ WAN 2: 4G│  │ VPS Vultr   │
         │ (VLAN 10 / eth0) │  │ (usb0)   │  │ (WireGuard) │
         └──────────────────┘  └──────────┘  └──────┬──────┘
                                                    │
                                                    ▼
                                              IP Singapura


### Skema Distribusi Trafik
| Jenis Trafik | Pintu Keluar | Keterangan |
| :--- | :--- | :--- |
| **Game Online (UDP/TCP)** | WAN Fiber (Prioritas 100%) | Latensi terendah, proteksi antrean SQM CAKE, failover ke 4G. |
| **ICMP (Ping Test)** | WAN Fiber (Prioritas 100%) | Menjaga stabilitas latensi uji koneksi lokal. |
| **Search Engine (Google/Bing)** | ISP Lokal (Fiber / 4G Modem) | Dynamic bypass via `nftset` untuk mencegah Captcha & lokalisasi SGD. |
| **Zoom & Video Conference** | VPS Vultr Singapura | Bypass pemblokiran/pembatasan rute dari ISP lokal. |
| **Browsing Umum & Speedtest** | VPS Vultr Singapura | Terowongan WireGuard, IP publik terdeteksi Singapura. |
| **DNS Request (Port 53)** | Cloudflare Families | Paksa port 53 ke `1.1.1.3` (Anti Konten Dewasa & Malware). |

---

## 2. Konfigurasi VPS WireGuard Server (Ubuntu 24.04)

### a. Instalasi & Konfigurasi Forwarding
```bash
# Update & Install Package
sudo apt update && sudo apt install -y wireguard iptables

# Aktifkan IP Forwarding Kernel
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.d/99-wireguard.conf
sudo sysctl -p /etc/sysctl.d/99-wireguard.conf

```

### b. Pembuatan Keypair

```bash
umask 077
wg genkey | tee server_private.key | wg pubkey > server_public.key
wg genkey | tee client_private.key | wg pubkey > client_public.key

```

### c. File Konfigurasi `/etc/wireguard/wg0.conf`

> **Catatan:** Sesuaikan `enp1s0` dengan interface internet fisik VPS Anda (cek via `ip route show default`).

```ini
[Interface]
Address = 10.10.0.1/24
ListenPort = 51820
PrivateKey = <SERVER_PRIVATE_KEY>
PostUp = iptables -t nat -A POSTROUTING -o enp1s0 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -o enp1s0 -j MASQUERADE

[Peer]
# STB OpenWrt Gateway
PublicKey = <CLIENT_PUBLIC_KEY>
AllowedIPs = 10.10.0.2/32

```

### d. Menjalankan Service

```bash
sudo systemctl enable --now wg-quick@wg0

```

---

## 3. Konfigurasi Jaringan OpenWrt

### File: `/etc/config/network`

```uci
config interface 'loopback'
	option device 'lo'
	option proto 'static'
	option ipaddr '127.0.0.1'
	option netmask '255.0.0.0'

config globals 'globals'
	option ula_prefix 'fdf1:baea:20b8::/48'

# VLAN 10 untuk WAN Fiber
config device
	option name 'eth0.10'
	option type '8021q'
	option ifname 'eth0'
	option vid '10'

# VLAN 20 untuk Interkoneksi ke MikroTik
config device
	option name 'eth0.20'
	option type '8021q'
	option ifname 'eth0'
	option vid '20'

# LAN Interface (Gateway 192.168.77.1 untuk MikroTik)
config interface 'lan'
	option device 'eth0.20'
	option proto 'static'
	option ipaddr '192.168.77.1'
	option netmask '255.255.255.0'
	list dns '1.1.1.3'
	list dns '1.0.0.3'

# WAN 1: Fiber Optic (DHCP dari ONT)
config interface 'wan'
	option device 'eth0.10'
	option proto 'dhcp'
	option delegate '0'
	option metric '10'

# WAN 2: Modem 4G LTE USB
config interface 'tethering'
	option proto 'dhcp'
	option device 'usb0'
	option metric '20'

# WAN 3: WireGuard Tunnel Client ke VPS Vultr
config interface 'wg0'
	option proto 'wireguard'
	option private_key '<CLIENT_PRIVATE_KEY>'
	list addresses '10.10.0.2/24'
	option metric '30'

config wireguard_wg0
	option public_key '<SERVER_PUBLIC_KEY>'
	option endpoint_host '<IP_PUBLIK_VPS_VULTR>'
	option endpoint_port '51820'
	option persistent_keepalive '25'
	list allowed_ips '0.0.0.0/0'
	option route_allowed_ips '1'

```

---

## 4. Konfigurasi Firewall, TTL 65 & DNS Hijacking

### a. File: `/etc/config/firewall`

Pastikan zona `wan` mencakup `tethering` dan `wg0`, serta DNS Port 53 dibelokkan ke Cloudflare Families:

```uci
config defaults
	option input 'ACCEPT'
	option output 'ACCEPT'
	option forward 'REJECT'
	option synflood_protect '1'

config zone
	option name 'lan'
	option input 'ACCEPT'
	option output 'ACCEPT'
	option forward 'ACCEPT'
	list network 'lan'

config zone
	option name 'wan'
	option input 'REJECT'
	option output 'ACCEPT'
	option forward 'REJECT'
	option masq '1'
	option mtu_fix '1'
	list network 'wan'
	list network 'wan6'
	list network 'tethering'
	list network 'wg0'

config forwarding
	option src 'lan'
	option dest 'wan'

# Dynamic IPSet untuk Bypass Search Engine
config ipset
	option name 'search_bypass'
	option match 'dest_ip'
	option family 'ipv4'
	option storage 'hash'

# Intercept & Force DNS ke Cloudflare 1.1.1.3 (Bebas Malware/Konten Dewasa)
config redirect
	option name 'Force-DNS-UDP'
	option src 'lan'
	option proto 'udp'
	option src_dport '53'
	option dest_ip '1.1.1.3'
	option dest_port '53'
	option target 'DNAT'

config redirect
	option name 'Force-DNS-TCP'
	option src 'lan'
	option proto 'tcp'
	option src_dport '53'
	option dest_ip '1.1.1.3'
	option dest_port '53'
	option target 'DNAT'

config include
	option path '/etc/nftables.d/ttl.nft'
	option type 'nftables'

```

### b. Konfigurasi TTL 65 (Bypass Kuota Tethering Operator)

Buat file `/etc/nftables.d/ttl.nft`:

```text
chain mangle_ttl_out {
    type filter hook postrouting priority mangle; policy accept;
    oifname "usb0" ip ttl set 65
}

```

Tambahkan backup persistensi di `/etc/rc.local`:

```sh
iptables -t mangle -I POSTROUTING -o usb0 -j TTL --ttl-set 65
iptables -t mangle -I PREROUTING -i usb0 -j TTL --ttl-set 65
exit 0

```

---

## 5. Konfigurasi Dynamic Bypass Search Engine

### a. Instalasi `dnsmasq-full`

```sh
opkg update
opkg remove dnsmasq
opkg install dnsmasq-full

```

### b. File: `/etc/dnsmasq.d/search_engine.conf`

```text
# === GOOGLE ===
nftset=/[google.com/4#inet#fw4#search_bypass](https://google.com/4#inet#fw4#search_bypass)
nftset=/google.co.id/4#inet#fw4#search_bypass
nftset=/[gstatic.com/4#inet#fw4#search_bypass](https://gstatic.com/4#inet#fw4#search_bypass)
nftset=/[googleapis.com/4#inet#fw4#search_bypass](https://googleapis.com/4#inet#fw4#search_bypass)
nftset=/[googleusercontent.com/4#inet#fw4#search_bypass](https://googleusercontent.com/4#inet#fw4#search_bypass)

# === BING ===
nftset=/[bing.com/4#inet#fw4#search_bypass](https://bing.com/4#inet#fw4#search_bypass)
nftset=/bing.net/4#inet#fw4#search_bypass
nftset=/[msn.com/4#inet#fw4#search_bypass](https://msn.com/4#inet#fw4#search_bypass)

# === DUCKDUCKGO & YAHOO ===
nftset=/[duckduckgo.com/4#inet#fw4#search_bypass](https://duckduckgo.com/4#inet#fw4#search_bypass)
nftset=/[yahoo.com/4#inet#fw4#search_bypass](https://yahoo.com/4#inet#fw4#search_bypass)
nftset=/[search.yahoo.com/4#inet#fw4#search_bypass](https://search.yahoo.com/4#inet#fw4#search_bypass)

```

---

## 6. Konfigurasi Routing & Failover (`mwan3`)

### File: `/etc/config/mwan3`

```uci
config globals 'globals'
	option mmx_mask '0x3F00'
	option init_done '1'

# ==========================================
# 1. TRACKING INTERFACES
# ==========================================
config interface 'wan'
	option enabled '1'
	list track_ip '1.1.1.1'
	list track_ip '8.8.8.8'
	option reliability '1'
	option count '1'
	option timeout '2'
	option interval '5'
	option down '3'
	option up '3'
	option initial_state 'online'

config interface 'tethering'
	option enabled '1'
	list track_ip '1.0.0.1'
	list track_ip '8.8.4.4'
	option reliability '1'
	option count '1'
	option timeout '2'
	option interval '5'
	option down '3'
	option up '3'
	option initial_state 'online'

config interface 'wg0'
	option enabled '1'
	list track_ip '1.1.1.1'
	list track_ip '8.8.8.8'
	option reliability '1'
	option count '1'
	option timeout '2'
	option interval '5'
	option down '3'
	option up '3'
	option initial_state 'online'

# ==========================================
# 2. MEMBER DEFINITIONS
# ==========================================
config member 'wan_local'
	option interface 'wan'
	option metric '1'
	option weight '1'

config member 'tethering_local'
	option interface 'tethering'
	option metric '2'
	option weight '1'

config member 'wan_balanced'
	option interface 'wan'
	option metric '1'
	option weight '1'

config member 'tethering_balanced'
	option interface 'tethering'
	option metric '1'
	option weight '2'

config member 'wg0_main'
	option interface 'wg0'
	option metric '1'
	option weight '1'

config member 'wan_vps_backup'
	option interface 'wan'
	option metric '2'
	option weight '1'

config member 'tethering_vps_backup'
	option interface 'tethering'
	option metric '3'
	option weight '1'

# ==========================================
# 3. POLICIES
# ==========================================
config policy 'policy_game'
	list use_member 'wan_local'
	list use_member 'tethering_local'
	option last_resort 'default'

config policy 'policy_bypass'
	list use_member 'wan_balanced'
	list use_member 'tethering_balanced'
	option last_resort 'default'

config policy 'policy_vps'
	list use_member 'wg0_main'
	list use_member 'wan_vps_backup'
	list use_member 'tethering_vps_backup'
	option last_resort 'default'

# ==========================================
# 4. TRAFFIC RULES
# ==========================================
# 1. ICMP Ping Jalur Cepat Lokal
config rule 'rule_ping'
	option family 'ipv4'
	option proto 'icmp'
	option use_policy 'policy_game'

# 2. Game UDP Ports
config rule 'rule_game_udp'
	option family 'ipv4'
	option proto 'udp'
	option dest_port '5000:5500,7000:8012,9000:9010,10001:10090,17500,27000:27050,30000:30200'
	option use_policy 'policy_game'

# 3. Game TCP Ports
config rule 'rule_game_tcp'
	option family 'ipv4'
	option proto 'tcp'
	option dest_port '7000:8000,9000:9010,27015:27050'
	option use_policy 'policy_game'

# 4. Zoom Meeting Ports (Lewat VPS)
config rule 'rule_zoom'
	option family 'ipv4'
	option proto 'udp'
	option dest_port '3478:3479,5090:5091,8801:8810'
	option use_policy 'policy_vps'

# 5. Search Engine Bypass (Google, Bing, dll. ke ISP Lokal)
config rule 'rule_bypass_search'
	option family 'ipv4'
	option ipset 'search_bypass'
	option use_policy 'policy_bypass'

# 6. Trafik Umum, Speedtest, Browsing (Masuk VPS Vultr)
config rule 'default_rule'
	option family 'ipv4'
	option dest_ip '0.0.0.0/0'
	option use_policy 'policy_vps'

```

---

## 7. Anti-Lag & QoS (`sqm-scripts` CAKE)

Mengeliminasi *bufferbloat* pada jalur Fiber agar lonjakan unduhan tidak mengganggu latensi paket game online.

### File: `/etc/config/sqm`

```uci
config queue 'eth0_10'
	option enabled '1'
	option interface 'eth0.10'
	option download '33000'
	option upload '33000'
	option qdisc 'cake'
	option script 'layer_cake.qos'
	option qdisc_advanced '1'
	option ingress_ecn 'ECN'
	option egress_ecn 'ECN'
	option qdisc_really_really_advanced '1'
	option linklayer 'none'

```

---

## 8. Event Monitoring & Logging Gateway

Skrip pencatat otomatis saat interface mengalami putus atau pulih (*event-driven* tanpa cron).

### File: `/etc/mwan3.user`

```sh
#!/bin/sh

LOG_FILE="/var/log/koneksi.log"

case "$ACTION" in
    disconnected)
        echo "$(date '+\%Y-\%m-\%d \%H:\%M:\%S') - [PUTUS] Interface$INTERFACE ($DEVICE) TERPUTUS / MATI" >> "$LOG_FILE"
        logger -t KONEKSI "[PUTUS] Interface $INTERFACE ($DEVICE) TERPUTUS"
        ;;
    connected)
        echo "$(date '+\%Y-\%m-\%d \%H:\%M:\%S') - [NYAMBUNG] Interface$INTERFACE ($DEVICE) KEMBALI NORMAL / TERHUBUNG" >> "$LOG_FILE"
        logger -t KONEKSI "[NYAMBUNG] Interface $INTERFACE ($DEVICE) KEMBALI NORMAL"
        ;;
esac

```

```sh
chmod +x /etc/mwan3.user
touch /var/log/koneksi.log

```

---

## 9. Panduan Pengujian & Operasional

### Perintah Pengecekan Status Rutin

```sh
# Cek handshake WireGuard
wg

# Cek status kesehatan multi-WAN & distribusi beban
mwan3 status

# Memantau log status koneksi fisik secara live
tail -f /var/log/koneksi.log

```

### Prosedur Verifikasi Jalur

1. **Pengecekan Identitas IP:** Akses `https://ipinfo.io` $\rightarrow$ Wajib menampilkan IP `45.45.45.45` (Vultr Singapore).
2. **Pengecekan Search Engine:** Akses `https://www.google.com` $\rightarrow$ Hasil penelusuran berbahasa Indonesia, tidak muncul *unusual traffic captcha*.
3. **Uji Failover:**
* Cabut kabel Fiber $\rightarrow$ Trafik otomatis beralih ke Modem 4G (`usb0`) dalam 5–10 detik.
* Cabut kabel USB 4G $\rightarrow$ Seluruh beban ditanggung penuh oleh Fiber (`wan`).
