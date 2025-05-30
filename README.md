## What we will do?
A secure WireGuard VPN on an EC2 t2.nano can be fully automated at boot by combining systemd’s wg-quick service with persistent kernel forwarding, UFW policy tweaks, and a NAT masquerade rule. On the client side—especially Android—you must force all traffic (including DNS) into the tunnel via AllowedIPs = 0.0.0.0/0, ::/0, specify a VPN DNS, and enable “use default route” or “always‑on” in settings. Once configured, your server will route and NAT VPN traffic correctly, and your Android device will show the EC2 public IP at sites like ipinfo.io.
## Prerequisites
- An AWS EC2 t2.nano instance running Ubuntu 22.04 LTS.
- A security group allowing SSH (TCP/22) and WireGuard (UDP/51820).
- Root or sudo access to the instance.
## Process Serverside:
### Step 1: Install and Enable WireGuard
install wireguard and generate keys
```bash
sudo -i
```
then:
```bash
sudo apt update                                           
sudo apt install wireguard -y                             
sudo mkdir -p /etc/wireguard                              
cd /etc/wireguard                                         
sudo wg genkey | sudo tee private.key | wg pubkey | sudo tee public.key
sudo chmod 600 private.key                                
```
`ls -l` will show something like this-
```bash
-rw------- 1 root root 45 Apr 21 15:35 private.key
-rw-r--r-- 1 root root 45 Apr 21 15:35 public.key
```

### Step 2: Create `/etc/wireguard/wg0.conf`
Now do a 
```bash
cat private.key
```
it will show somw thing like-
```bash
PVTKEYSERVERXXXXXXX....=
```
Do a 
```bash
sudo nano /etc/wireguard/wg0.conf
```
and add this to conf
```bash
[Interface]
Address = 10.0.0.1/24
PrivateKey = PVTKEYSERVERXXXXXXX....=
ListenPort = 51820
PostUp      = iptables -t nat -A POSTROUTING -o enX0 -j MASQUERADE
PostDown    = iptables -t nat -D POSTROUTING -o enX0 -j MASQUERADE

# Later you’ll add [Peer] blocks here for each client
```
here to check the network interface do this-
```bash
ip a
```
the output will look like this-
```bash
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: enX0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc fq_codel state UP group default qlen 1000
    link/ether 0a:ff:cb:a5:e4:b3 brd ff:ff:ff:ff:ff:ff
    inet 172.31.23.147/20 metric 100 brd 172.31.31.255 scope global dynamic enX0
       valid_lft 3316sec preferred_lft 3316sec
    inet6 fe80::8ff:cbff:fea5:e4b3/64 scope link
       valid_lft forever preferred_lft forever
```
here `enX0` is the interface name

### Step 3: Enable Kernel IP Forwarding
Do a
```bash
sudo nano /etc/ufw/sysctl.conf
```
and uncomment or add:
```bash
net/ipv4/ip_forward=1
```
then apply immediately 
```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

### Step 4: Adjust UFW Forward Policy
first allow ssh and 51820 (wireguard port) in efw-
```bash
sudo ufw allow ssh
sudo ufw allow 51820/udp
```
then enable ufw -
```bash
sudo ufw enable
```
edit `/etc/default/ufw` to add forward policy
```bash
sudo nano /etc/default/ufw
```
and set
```
DEFAULT_FORWARD_POLICY="ACCEPT"
```
Reload UFW:
```bash
sudo ufw reload
```

### Optional 4.x: Persist NAT Masquerade (Alternate)
If you prefer not to use PostUp in wg0.conf, insert into `/etc/ufw/before.rules` above the COMMIT line:
should look like this:
```bash
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -o enX0 -j MASQUERADE
COMMIT
```
### Step 5: Enable WireGuard at Boot
```bash
sudo systemctl enable wg-quick@wg0.service
sudo systemctl start  wg-quick@wg0.service
```

## Process ClientSide:
Step 1: generate the keys again
 - If on desktop generate keys similarly u did on server
 - On android Wireguard app there is a option to generate there only
Step 2: In simple sence the client conf should look like this-
```bash
[Interface]
PrivateKey = <client_private_key>
Address    = 10.0.0.2/24
DNS        = 1.1.1.1

[Peer]
PublicKey  = <server_public_key>
Endpoint   = <ec2_public_ip>:51820
AllowedIPs = 0.0.0.0/0, ::/0

```
- AllowedIPs = 0.0.0.0/0, ::/0 forces all traffic into the tunnel
- Specifying DNS = 1.1.1.1 prevents DNS leaks back to your ISP

## Process ServerSide:
### Step 1: Add the peer to the wg0.conf it should look something like this-
```bash
[Interface]
Address     = 10.0.0.1/24
PrivateKey  = <server_private_key>
ListenPort  = 51820
PostUp      = iptables -t nat -A POSTROUTING -o enX0 -j MASQUERADE
PostDown    = iptables -t nat -D POSTROUTING -o enX0 -j MASQUERADE

[Peer]
PublicKey    = <client_public_key>
AllowedIPs   = 10.0.0.2/32
```
Then do a reload
```bash
sudo systemctl reload wg-quick@wg0
```










