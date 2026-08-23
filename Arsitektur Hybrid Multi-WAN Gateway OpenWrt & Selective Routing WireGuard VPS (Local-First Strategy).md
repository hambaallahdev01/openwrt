# Arsitektur Hybrid Multi-WAN Gateway OpenWrt & Selective Routing WireGuard VPS (Local-First Strategy)

Dokumentasi implementasi gateway jaringan berbasis **OpenWrt 23.05 (HG680P / ARMv8)** yang terintegrasi dengan **MikroTik Core Router**, **Dual-WAN (Fiber Optic + Modem 4G USB)**, dan **VPS Vultr Singapura (WireGuard Tunnel)** menggunakan pendekatan **Local-First Routing**, **Dynamic Selective Routing (nftset)**, **Anti-Bufferbloat (SQM CAKE)**, dan **DNS Enforcement**.

---

## 1. Ringkasan Arsitektur & Strategi Jalur

Sistem ini menerapkan prinsip **Local-First Routing**: Seluruh penelusuran umum, Google, Bing, YouTube, dan perbankan dialirkan langsung melalui ISP lokal (Fiber 35 Mbps) untuk mencegah *reCAPTCHA*, latensi tinggi, dan pembatasan wilayah (*geo-blocking*). Terowongan WireGuard VPS Singapura hanya digunakan secara spesifik untuk aplikasi *conference* (Zoom), pengujian performa (*Speedtest*), dan alat inspeksi jaringan (*Check-IP*).


```

```
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
    │  [Dynamic Set]  ──► dnsmasq-full + nftset (vps_sites)        │
    └──────────────┬───────────────┬────────────────┬──────────────┘
                   │               │                │
      [Game, Ping, │               │       [Zoom, Speedtest,
       Browsing,   │               │        Check-IP Sites]
      Google/Bing] │               │                │
                   ▼               ▼                ▼
         ┌──────────────────┐  ┌──────────┐  ┌─────────────┐
         │ WAN 1: Fiber     │  │ WAN 2: 4G│  │ VPS Vultr   │
         │ (VLAN 10 / eth0) │  │ (usb0)   │  │ (WireGuard) │
         └──────────────────┘  └──────────┘  └──────┬──────┘
                                                    │
                                                    ▼
                                              IP Singapura

```

```

### Tabel Distribusi Trafik
| Jenis Trafik | Pintu Keluar | Mekanisme |
| :--- | :--- | :--- |
| **Game Online (UDP/TCP)** | WAN Fiber (Prioritas 100%) | `mwan3` Port Matching (`policy_game`) |
| **ICMP (Ping Test)** | WAN Fiber (Prioritas 100%) | `mwan3` Proto ICMP (`policy_game`) |
| **Browsing Umum, Google & Bing** | ISP Lokal (Fiber / 4G Modem) | `mwan3` Default Rule (`policy_bypass`) |
| **Enkripsi WireGuard Endpoint** | WAN Fiber $\rightarrow$ Failover 4G | `mwan3` Rule IP VPS (`policy_game`) |
| **Zoom Meeting** | VPS Vultr Singapura | `mwan3` UDP Ports (`policy_vps`) |
| **Speedtest & Check-IP Tools** | VPS Vultr Singapura | `dnsmasq-full` $\rightarrow$ `nftset` Mark `0x300` |
| **DNS (Port 53)** | Cloudflare Families | Local DNAT Redirect ke `1.1.1.3` |

---

## 2. Konfigurasi VPS WireGuard Server (Ubuntu 24.04)

### a. Forwarding Kernel & Paket
```bash
sudo apt update && sudo apt install -y wireguard iptables
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.d/99-wireguard.conf
sudo sysctl -p /etc/sysctl.d/99-wireguard.conf

```

### b. File `/etc/wireguard/wg0.conf`

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

# VLAN 10 (WAN Fiber)
config device
	option name 'eth0.10'
	option type '8021q'
	option ifname 'eth0'
	option vid '10'

# VLAN 20 (Interkoneksi ke MikroTik Core)
config device
	option name 'eth0.20'
	option type '8021q'
	option ifname 'eth0'
	option vid '20'

# LAN Gateway untuk MikroTik
config interface 'lan'
	option device 'eth0.20'
	option proto 'static'
	option ipaddr '192.168.77.1'
	option netmask '255.255.255.0'
	list dns '1.1.1.3'
	list dns '1.0.0.3'

# WAN 1: Fiber Optic
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

# WAN 3: WireGuard Tunnel Client
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

## 4. Konfigurasi Firewall & Scoped TTL 65

### a. File: `/etc/config/firewall`

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

# Dynamic IPSet untuk Layanan Khusus VPS
config ipset
	option name 'vps_sites'
	option match 'dest_ip'
	option family 'ipv4'

# DNS Intercept ke Dnsmasq Lokal
config redirect
	option name 'Redirect-DNS-Local-UDP'
	option src 'lan'
	option proto 'udp'
	option src_dport '53'
	option dest_port '53'
	option target 'DNAT'

config redirect
	option name 'Redirect-DNS-Local-TCP'
	option src 'lan'
	option proto 'tcp'
	option src_dport '53'
	option dest_port '53'
	option target 'DNAT'

# Blokir QUIC (Memaksa Fallback ke TCP 443 Standar)
config rule
	option name 'Block-QUIC'
	option src 'lan'
	option dest 'wan'
	option proto 'udp'
	option dest_port '443'
	option target 'REJECT'

config include
	option path '/etc/firewall.user'
	option reload '1'
	option fw4_compatible '1'

```

### b. File: `/etc/firewall.user`

Aturan penandaan rute `0x300` (*WireGuard Table*) dan perbaikan TTL 65 yang diisolasi khusus ke `usb0` (mencegah error Hop Limit RA pada MikroTik):

```sh
# 1. TTL 65 Khusus Modem USB (usb0) & IPv4 Saja (Bypass Kuota Tethering)
nft add chain inet fw4 ttl_fix_out '{ type filter hook postrouting priority 300; policy accept; }' 2>/dev/null
nft flush chain inet fw4 ttl_fix_out 2>/dev/null
nft add rule inet fw4 ttl_fix_out oifname "usb0" ip ttl set 65 2>/dev/null

nft add chain inet fw4 ttl_fix_in '{ type filter hook prerouting priority 300; policy accept; }' 2>/dev/null
nft flush chain inet fw4 ttl_fix_in 2>/dev/null
nft add rule inet fw4 ttl_fix_in iifname "usb0" ip ttl set 65 2>/dev/null

# 2. Belokkan Speedtest & Check-IP ke WireGuard VPS (Mark 0x300 = wg0 table)
nft add chain inet fw4 vps_sites_prerouting '{ type filter hook prerouting priority mangle + 10; policy accept; }' 2>/dev/null
nft flush chain inet fw4 vps_sites_prerouting 2>/dev/null
nft add rule inet fw4 vps_sites_prerouting ip daddr @vps_sites meta mark set 0x300
nft add rule inet fw4 vps_sites_prerouting ip daddr @vps_sites ct mark set 0x300

```

---

## 5. Konfigurasi Dynamic DNS & Set (`dnsmasq-full`)

### a. File: `/etc/config/dhcp`

```uci
config dnsmasq
	option domainneeded '1'
	option localise_queries '1'
	option rebind_protection '1'
	option rebind_localhost '1'
	option local '/lan/'
	option domain 'lan'
	option expandhosts '1'
	option cachesize '1000'
	option authoritative '1'
	option readethers '1'
	option leasefile '/tmp/dhcp.leases'
	option resolvfile '/tmp/resolv.conf.d/resolv.conf.auto'
	option localservice '1'
	list confdir '/etc/dnsmasq.d'
	list server '1.1.1.3'
	list server '1.0.0.3'

```

### b. File: `/etc/dnsmasq.d/vps_sites.conf`

```text
# === SPEEDTEST SERVICES ===
nftset=/speedtest.net/4#inet#fw4#vps_sites
nftset=/ooklaserver.net/4#inet#fw4#vps_sites
nftset=/ookla.com/4#inet#fw4#vps_sites
nftset=/speedtestcustom.com/4#inet#fw4#vps_sites
nftset=/fast.com/4#inet#fw4#vps_sites
nftset=/netflix.com/4#inet#fw4#vps_sites
nftset=/nflxvideo.net/4#inet#fw4#vps_sites
nftset=/openspeedtest.com/4#inet#fw4#vps_sites
nftset=/meter.net/4#inet#fw4#vps_sites
nftset=/testmy.net/4#inet#fw4#vps_sites

# === CHECK IP & NETWORK TOOLS ===
nftset=/check-host.net/4#inet#fw4#vps_sites
nftset=/ipinfo.io/4#inet#fw4#vps_sites
nftset=/whoer.net/4#inet#fw4#vps_sites
nftset=/whatismyipaddress.com/4#inet#fw4#vps_sites
nftset=/whatismyip.com/4#inet#fw4#vps_sites
nftset=/icanhazip.com/4#inet#fw4#vps_sites
nftset=/ifconfig.me/4#inet#fw4#vps_sites
nftset=/ifconfig.co/4#inet#fw4#vps_sites
nftset=/myip.is/4#inet#fw4#vps_sites
nftset=/myip.com/4#inet#fw4#vps_sites
nftset=/ip.me/4#inet#fw4#vps_sites
nftset=/iplocation.net/4#inet#fw4#vps_sites
nftset=/showmyip.com/4#inet#fw4#vps_sites
nftset=/dnsleaktest.com/4#inet#fw4#vps_sites
nftset=/browserleaks.com/4#inet#fw4#vps_sites
nftset=/2ip.io/4#inet#fw4#vps_sites

```

---

## 6. Konfigurasi Routing & Failover (`mwan3`)

### File: `/etc/config/mwan3`

```uci
config globals 'globals'
	option mmx_mask '0x3F00'
	option init_done '1'

# ==========================================
# 1. TRACKING INTERFACE
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
# 2. MEMBER CONFIGURATION
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
# 0. Kunci Jalur Enkripsi WireGuard ke VPS (Prioritas Fiber -> Failover 4G)
config rule 'rule_vps_endpoint'
	option family 'ipv4'
	option dest_ip '<IP_PUBLIK_VPS_VULTR>'
	option proto 'udp'
	option dest_port '51820'
	option use_policy 'policy_game'

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

# 4. Zoom Meeting Ports (Masuk VPS Singapura)
config rule 'rule_zoom'
	option family 'ipv4'
	option proto 'udp'
	option dest_port '3478:3479,5090:5091,8801:8810'
	option use_policy 'policy_vps'

# 5. Default: Seluruh Internet Lewat ISP Lokal (Fiber 35 Mbps)
config rule 'default_rule'
	option family 'ipv4'
	option dest_ip '0.0.0.0/0'
	option use_policy 'policy_bypass'

```

---

## 7. Anti-Bufferbloat (`sqm-scripts` CAKE)

Mengamankan antrean paket game saat terjadi lonjakan unduhan di jalur Fiber.

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

## 8. Skrip Monitoring Status (`/etc/mwan3.user`)

```sh
#!/sh

LOG_FILE="/var/log/koneksi.log"

case "$ACTION" in
    disconnected)
        echo "$(date '+\%Y-\%m-\%d \%H:\%M:\%S') - [PUTUS] Interface$INTERFACE ($DEVICE) TERPUTUS" >> "$LOG_FILE"
        logger -t KONEKSI "[PUTUS] Interface $INTERFACE ($DEVICE) TERPUTUS"
        ;;
    connected)
        echo "$(date '+\%Y-\%m-\%d \%H:\%M:\%S') - [NYAMBUNG] Interface$INTERFACE ($DEVICE) TERHUBUNG KEMBALI" >> "$LOG_FILE"
        logger -t KONEKSI "[NYAMBUNG] Interface $INTERFACE ($DEVICE) TERHUBUNG"
        ;;
esac

```

```sh
chmod +x /etc/mwan3.user
touch /var/log/koneksi.log

```

---

## 9. Perintah Verifikasi Operasional

```sh
# Cek Handshake WireGuard
wg

# Cek Status Multi-WAN
mwan3 status

# Cek Daftar IP yang Terdaftar di Set Layanan VPS
nft list set inet fw4 vps_sites

# Uji Simulasi Rute Mark 0x300 ke Salah Satu IP Set
ip route get 104.21.74.214 mark 0x300

```
