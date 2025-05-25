# Konfigurasi Lengkap Proyek Jaringan GNS3

## 1. Konfigurasi IP Address di Semua Router

### Router R1
```bash
# Reset konfigurasi
/system reset-configuration no-defaults=yes skip-backup=yes

# Konfigurasi IP Address
/ip address
add address=12.12.12.1/24 interface=ether1 comment="to R2"
add address=13.13.13.1/24 interface=ether2 comment="to R3"  
add address=1.1.1.1/24 interface=ether3 comment="to R-Cabang-1"

# Set hostname
/system identity set name="R1"
```

### Router R2
```bash
# Reset konfigurasi
/system reset-configuration no-defaults=yes skip-backup=yes

# Konfigurasi IP Address
/ip address
add address=12.12.12.2/24 interface=ether1 comment="to R1"
add address=23.23.23.2/24 interface=ether2 comment="to R3"
add address=2.2.2.2/24 interface=ether3 comment="to R-Cabang-2"

# Set hostname
/system identity set name="R2"
```

### Router R3
```bash
# Reset konfigurasi
/system reset-configuration no-defaults=yes skip-backup=yes

# Konfigurasi IP Address
/ip address
add address=13.13.13.3/24 interface=ether1 comment="to R1"
add address=23.23.23.3/24 interface=ether2 comment="to R2"
add address=3.3.12.3/24 interface=ether3 comment="to R-Pusat"
add address=4.4.4.3/24 interface=ether4 comment="to PC-Publik"

# Set hostname
/system identity set name="R3"

# Konfigurasi DHCP Client untuk internet (ether5)
/ip dhcp-client add interface=ether5 disabled=no
```

### Router R-Cabang-1
```bash
# Reset konfigurasi
/system reset-configuration no-defaults=yes skip-backup=yes

# Konfigurasi IP Address
/ip address
add address=1.1.1.11/24 interface=ether1 comment="to R1"
add address=192.168.1.11/24 interface=ether2 comment="LAN PC1"

# Set hostname
/system identity set name="R-Cabang-1"
```

### Router R-Cabang-2
```bash
# Reset konfigurasi
/system reset-configuration no-defaults=yes skip-backup=yes

# Konfigurasi IP Address
/ip address
add address=2.2.2.12/24 interface=ether1 comment="to R2"
add address=192.168.2.12/24 interface=ether2 comment="LAN PC2"

# Set hostname
/system identity set name="R-Cabang-2"
```

### Router R-Pusat
```bash
# Reset konfigurasi
/system reset-configuration no-defaults=yes skip-backup=yes

# Konfigurasi IP Address
/ip address
add address=3.3.12.13/24 interface=ether1 comment="to R3"
add address=192.168.13.13/24 interface=ether2 comment="LAN Server"

# Set hostname
/system identity set name="R-Pusat"
```

## 2. Konfigurasi OSPF di R1, R2, dan R3

### Router R1 - OSPF
```bash
# Enable OSPF
/routing ospf instance
add name=default router-id=1.1.1.1

/routing ospf area
add name=backbone area-id=0.0.0.0 instance=default

/routing ospf interface-template
add area=backbone interfaces=ether1 comment="to R2"
add area=backbone interfaces=ether2 comment="to R3"
add area=backbone interfaces=ether3 comment="to R-Cabang-1"
```

### Router R2 - OSPF
```bash
# Enable OSPF
/routing ospf instance
add name=default router-id=2.2.2.2

/routing ospf area
add name=backbone area-id=0.0.0.0 instance=default

/routing ospf interface-template
add area=backbone interfaces=ether1 comment="to R1"
add area=backbone interfaces=ether2 comment="to R3"
add area=backbone interfaces=ether3 comment="to R-Cabang-2"
```

### Router R3 - OSPF
```bash
# Enable OSPF
/routing ospf instance
add name=default router-id=3.3.3.3

/routing ospf area
add name=backbone area-id=0.0.0.0 instance=default

/routing ospf interface-template
add area=backbone interfaces=ether1 comment="to R1"
add area=backbone interfaces=ether2 comment="to R2"
add area=backbone interfaces=ether3 comment="to R-Pusat"

# Redistribute default route via OSPF
/routing ospf instance
set default redistribute=connected,static
```

## 3. NAT dan Default Route

### Router R3 - Gateway Internet
```bash
# Default route ke internet (sesuaikan dengan gateway DHCP)
/ip route
add dst-address=0.0.0.0/0 gateway=ether5 comment="Default to Internet"

# NAT Masquerade untuk semua jaringan internal
/ip firewall nat
add chain=srcnat out-interface=ether5 action=masquerade comment="NAT to Internet"
```

### Router R-Cabang-1 - Default Route
```bash
# Default route menuju R1
/ip route
add dst-address=0.0.0.0/0 gateway=1.1.1.1 comment="Default via R1"

# NAT untuk LAN
/ip firewall nat
add chain=srcnat out-interface=ether1 action=masquerade comment="NAT LAN"
```

### Router R-Cabang-2 - Default Route
```bash
# Default route menuju R2
/ip route
add dst-address=0.0.0.0/0 gateway=2.2.2.2 comment="Default via R2"

# NAT untuk LAN
/ip firewall nat
add chain=srcnat out-interface=ether1 action=masquerade comment="NAT LAN"
```

### Router R-Pusat - Default Route
```bash
# Default route menuju R3
/ip route
add dst-address=0.0.0.0/0 gateway=3.3.12.3 comment="Default via R3"

# NAT untuk LAN
/ip firewall nat
add chain=srcnat out-interface=ether1 action=masquerade comment="NAT LAN"
```

## 4. Konfigurasi IP dan Route di Debian

### PC1 (Debian)
```bash
# Konfigurasi IP static
sudo nano /etc/network/interfaces

# Tambahkan:
auto eth0
iface eth0 inet static
    address 192.168.1.10
    netmask 255.255.255.0
    gateway 192.168.1.11
    dns-nameservers 3.3.12.3

# Restart networking
sudo systemctl restart networking

# Atau gunakan ip command:
sudo ip addr add 192.168.1.10/24 dev eth0
sudo ip route add default via 192.168.1.11
echo "nameserver 3.3.12.3" | sudo tee /etc/resolv.conf
```

### PC2 (Debian)
```bash
# Konfigurasi IP static
sudo nano /etc/network/interfaces

# Tambahkan:
auto eth0
iface eth0 inet static
    address 192.168.2.10
    netmask 255.255.255.0
    gateway 192.168.2.12
    dns-nameservers 3.3.12.3

# Restart networking
sudo systemctl restart networking

# Atau gunakan ip command:
sudo ip addr add 192.168.2.10/24 dev eth0
sudo ip route add default via 192.168.2.12
echo "nameserver 3.3.12.3" | sudo tee /etc/resolv.conf
```

### Server (Debian)
```bash
# Konfigurasi IP static
sudo nano /etc/network/interfaces

# Tambahkan:
auto eth0
iface eth0 inet static
    address 192.168.13.10
    netmask 255.255.255.0
    gateway 192.168.13.13
    dns-nameservers 3.3.12.3

# Restart networking
sudo systemctl restart networking

# Atau gunakan ip command:
sudo ip addr add 192.168.13.10/24 dev eth0
sudo ip route add default via 192.168.13.13
echo "nameserver 3.3.12.3" | sudo tee /etc/resolv.conf
```

### PC-Publik (Debian)
```bash
# Konfigurasi IP static
sudo nano /etc/network/interfaces

# Tambahkan:
auto eth0
iface eth0 inet static
    address 4.4.4.10
    netmask 255.255.255.0
    gateway 4.4.4.3
    dns-nameservers 8.8.8.8

# Restart networking
sudo systemctl restart networking

# Atau gunakan ip command:
sudo ip addr add 4.4.4.10/24 dev eth0
sudo ip route add default via 4.4.4.3
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

## 5. Setup Web Server di Server (Debian)

```bash
# Update sistem
sudo apt update && sudo apt upgrade -y

# Install Apache web server
sudo apt install apache2 -y

# Start dan enable Apache
sudo systemctl start apache2
sudo systemctl enable apache2

# Buat halaman web custom
sudo nano /var/www/html/index.html

# Isi dengan:
<!DOCTYPE html>
<html>
<head>
    <title>Kantor Pusat Web Server</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; margin: 50px; }
        .header { color: #2c3e50; font-size: 2em; margin-bottom: 30px; }
        .info { background: #ecf0f1; padding: 20px; border-radius: 10px; }
    </style>
</head>
<body>
    <div class="header">Selamat Datang di Kantor Pusat</div>
    <div class="info">
        <h3>Informasi Server</h3>
        <p><strong>Server IP:</strong> 192.168.13.10</p>
        <p><strong>Domain:</strong> kantorpusat.co.id</p>
        <p><strong>Status:</strong> Server Aktif</p>
        <p><strong>Waktu:</strong> <span id="datetime"></span></p>
    </div>
    <script>
        function updateTime() {
            document.getElementById('datetime').innerHTML = new Date().toString();
        }
        setInterval(updateTime, 1000);
        updateTime();
    </script>
</body>
</html>

# Set permission
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/

# Restart Apache
sudo systemctl restart apache2

# Cek status
sudo systemctl status apache2
```

## 6. Konfigurasi VPN

### A. IPsec Site-to-Site VPN (R-Cabang-1 ↔ R-Pusat)

#### Router R-Cabang-1 (Initiator)
```bash
# IPsec Policy
/ip ipsec policy
add src-address=192.168.1.0/24 dst-address=192.168.13.0/24 \
    protocol=all action=encrypt tunnel=yes \
    sa-src-address=1.1.1.11 sa-dst-address=3.3.12.13

# IPsec Peer
/ip ipsec peer
add address=3.3.12.13 secret="VPNSecret123" \
    exchange-mode=main send-initial-contact=yes \
    nat-traversal=yes proposal-check=obey

# IPsec Proposal
/ip ipsec proposal
add name="vpn-proposal" auth-algorithms=sha1 enc-algorithms=aes-256-cbc \
    pfs-group=modp1024

/ip ipsec peer
set 0 proposal="vpn-proposal"
```

#### Router R-Pusat (Responder)
```bash
# IPsec Policy
/ip ipsec policy
add src-address=192.168.13.0/24 dst-address=192.168.1.0/24 \
    protocol=all action=encrypt tunnel=yes \
    sa-src-address=3.3.12.13 sa-dst-address=1.1.1.11

# IPsec Peer
/ip ipsec peer
add address=1.1.1.11 secret="VPNSecret123" \
    exchange-mode=main send-initial-contact=yes \
    nat-traversal=yes proposal-check=obey

# IPsec Proposal
/ip ipsec proposal
add name="vpn-proposal" auth-algorithms=sha1 enc-algorithms=aes-256-cbc \
    pfs-group=modp1024

/ip ipsec peer
set 0 proposal="vpn-proposal"
```

### B. L2TP/IPSec Client VPN (PC2 → R-Pusat)

#### Router R-Pusat (L2TP Server)
```bash
# Enable L2TP Server
/interface l2tp-server server
set enabled=yes default-profile=default-encryption authentication=mschap2 \
    max-mru=1460 max-mtu=1460

# Create VPN user
/ppp secret
add name="pc2user" password="pc2pass" service=l2tp \
    local-address=10.0.0.1 remote-address=10.0.0.2

# Create L2TP profile
/ppp profile
add name="l2tp-profile" local-address=10.0.0.1 remote-address=10.0.0.2 \
    use-encryption=yes

# IPsec for L2TP
/ip ipsec policy
add src-address=0.0.0.0/0 dst-address=0.0.0.0/0 \
    protocol=udp src-port=1701 dst-port=1701 \
    action=encrypt tunnel=no

/ip ipsec peer
add address=0.0.0.0/0 secret="L2TPSecret123" \
    exchange-mode=main-l2tp generate-policy=port-strict

# Firewall untuk L2TP
/ip firewall filter
add chain=input protocol=udp dst-port=500,4500,1701 action=accept \
    comment="Allow L2TP/IPSec"
add chain=input protocol=ipsec-esp action=accept comment="Allow IPSec ESP"
```

#### PC2 (Debian - L2TP Client)
```bash
# Install L2TP client
sudo apt install xl2tpd strongswan -y

# Konfigurasi IPsec
sudo nano /etc/ipsec.conf

# Tambahkan:
config setup
    nat_traversal=yes
    virtual_private=%v4:10.0.0.0/8,%v4:192.168.0.0/16

conn L2TP-PSK
    authby=secret
    pfs=no
    auto=add
    keyingtries=3
    dpddelay=30
    dpdtimeout=120
    dpdaction=clear
    rekey=yes
    ikelifetime=8h
    keylife=1h
    type=transport
    left=%defaultroute
    leftprotoport=17/1701
    right=3.3.12.13
    rightprotoport=17/1701

# Konfigurasi secret
sudo nano /etc/ipsec.secrets

# Tambahkan:
%any %any : PSK "L2TPSecret123"

# Konfigurasi L2TP
sudo nano /etc/xl2tpd/xl2tpd.conf

# Tambahkan:
[lac vpn-connection]
lns = 3.3.12.13
ppp debug = yes
pppoptfile = /etc/ppp/options.l2tpd.client
length bit = yes

# Konfigurasi PPP
sudo nano /etc/ppp/options.l2tpd.client

# Tambahkan:
ipcp-accept-local
ipcp-accept-remote
refuse-eap
require-mschap-v2
noccp
noauth
idle 1800
mtu 1410
mru 1410
defaultroute
usepeerdns
debug
connect-delay 5000
name pc2user
password pc2pass

# Start services
sudo systemctl restart strongswan
sudo systemctl restart xl2tpd

# Connect VPN
sudo systemctl start strongswan
sudo ipsec up L2TP-PSK
sudo echo "c vpn-connection" > /var/run/xl2tpd/l2tp-control
```

## 7. DNS Server di R3

### Router R3 - DNS Server
```bash
# Enable DNS Service
/ip dns
set servers=8.8.8.8,8.8.4.4 allow-remote-requests=yes

# Add static DNS record
/ip dns static
add name="kantorpusat.co.id" address=3.3.12.13 type=A

# Verify DNS
/ip dns static print
```

## 8. Port Forwarding ke Web Server

### Router R3 - Port Forwarding
```bash
# Port forwarding HTTP ke Server
/ip firewall nat
add chain=dstnat dst-address=3.3.12.13 protocol=tcp dst-port=80 \
    action=dst-nat to-addresses=192.168.13.10 to-ports=80 \
    comment="HTTP to Web Server"

# Port forwarding HTTPS ke Server (optional)
/ip firewall nat
add chain=dstnat dst-address=3.3.12.13 protocol=tcp dst-port=443 \
    action=dst-nat to-addresses=192.168.13.10 to-ports=443 \
    comment="HTTPS to Web Server"
```

### Router R-Pusat - Allow forwarded traffic
```bash
# Allow traffic dari R3 ke Server
/ip firewall filter
add chain=forward src-address=3.3.12.3 dst-address=192.168.13.10 \
    protocol=tcp dst-port=80,443 action=accept \
    comment="Allow HTTP/HTTPS to Server"
```

## 9. Firewall Filter Rules

### Router R-Pusat - Firewall Filter
```bash
# Allow ping dari jaringan cabang
/ip firewall filter
add chain=input src-address=192.168.1.0/24 protocol=icmp action=accept \
    comment="Allow ping from Cabang-1"
add chain=input src-address=192.168.2.0/24 protocol=icmp action=accept \
    comment="Allow ping from Cabang-2"

# Block ping dari PC Publik
/ip firewall filter
add chain=input src-address=4.4.4.0/24 protocol=icmp action=drop \
    comment="Block ping from Public"

# Allow established dan related connections
/ip firewall filter
add chain=input connection-state=established,related action=accept

# Allow local management
/ip firewall filter
add chain=input src-address=192.168.13.0/24 action=accept \
    comment="Allow local management"
```

### Router R3 - General Firewall
```bash
# Basic firewall rules
/ip firewall filter
add chain=input connection-state=established,related action=accept
add chain=input protocol=icmp action=accept
add chain=input src-address=192.168.0.0/16 action=accept
add chain=input in-interface=ether5 action=drop comment="Drop from Internet"
```

## 10. Pengujian Sistem

### A. Test OSPF Connectivity
```bash
# Dari R-Cabang-1
/ping 3.3.12.13 count=5

# Dari R-Cabang-2  
/ping 3.3.12.13 count=5

# Check OSPF neighbors
/routing ospf neighbor print
```

### B. Test Internet Connectivity
```bash
# Dari PC1
ping -c 5 8.8.8.8
curl -I http://google.com

# Dari PC2
ping -c 5 8.8.8.8
curl -I http://google.com
```

### C. Test DNS Resolution
```bash
# Dari PC1
nslookup kantorpusat.co.id 3.3.12.3
dig @3.3.12.3 kantorpusat.co.id

# Dari PC-Publik
nslookup kantorpusat.co.id 3.3.12.3
```

### D. Test Web Server Access
```bash
# Dari PC-Publik (via domain)
curl -I http://kantorpusat.co.id
wget -O - http://kantorpusat.co.id

# Dari PC1 (direct IP via VPN)
curl -I http://192.168.13.10
```

### E. Test VPN Connectivity
```bash
# Test Site-to-Site VPN (dari PC1)
ping -c 5 192.168.13.10

# Check IPsec status di R-Cabang-1
/ip ipsec active-peers print
/ip ipsec policy print

# Test L2TP VPN dari PC2
# Setelah connect VPN, test akses:
curl -I http://192.168.13.10
ping -c 5 10.0.0.1
```

### F. Test Firewall Rules
```bash
# Dari PC1 (harus berhasil)
ping -c 5 3.3.12.13

# Dari PC2 (harus berhasil)  
ping -c 5 3.3.12.13

# Dari PC-Publik (harus gagal)
ping -c 5 3.3.12.13
```

## Troubleshooting Commands

### Check Interface Status
```bash
# RouterOS
/interface print
/ip address print
/ip route print

# Debian
ip addr show
ip route show
systemctl status networking
```

### Check OSPF Status
```bash
/routing ospf neighbor print
/routing ospf lsa print
/routing ospf interface print
```

### Check VPN Status
```bash
# IPsec
/ip ipsec active-peers print
/ip ipsec policy print
/ip ipsec peer print

# L2TP
/interface l2tp-server print
/ppp active print
```

### Check Firewall
```bash
/ip firewall filter print
/ip firewall nat print
/ip firewall connection print
```

### Network Monitoring
```bash
# RouterOS
/tool torch interface=ether1
/tool sniffer quick

# Debian
sudo tcpdump -i eth0
sudo netstat -tuln
ss -tuln
```

## Catatan Penting

1. **Backup Konfigurasi**: Selalu backup konfigurasi sebelum melakukan perubahan besar
   ```bash
   /export file=backup-config
   ```

2. **Security**: Ganti password default dan gunakan strong password untuk VPN

3. **Monitoring**: Monitor log secara berkala
   ```bash
   /log print
   ```

4. **Performance**: Monitor resource usage
   ```bash
   /system resource print
   ```

5. **Updates**: Pastikan RouterOS dan Debian up-to-date

Konfigurasi ini memberikan infrastruktur jaringan yang lengkap dengan routing dinamis, VPN, firewall, dan web services sesuai dengan requirement proyek Anda.
