# Solutions 10 — Networking

A note on Netplan files: settings for the same interface can come from several files
(Vagrant's `50-vagrant.yaml` and your `60-lab.yaml`). Before applying anything that
touches `eth1`, check the **merged** result with `sudo netplan get ethernets.eth1`, and
apply with `netplan try` — if you lose the lab network, it rolls back by itself.

---

## 🟢 Warm-up

**10.1**

```bash
ip -br link
ip -br addr
```

`eth0` holds `10.0.2.15` (VirtualBox NAT, the default route); `eth1` holds `192.168.56.11`.

**10.2**

```bash
ip route
ip route get 8.8.8.8          # via 10.0.2.2 dev eth0
ip route get 192.168.56.12    # dev eth1 — directly connected, no "via"
```

**10.3**

```bash
hostnamectl
grep '^hosts:' /etc/nsswitch.conf
ls -l /etc/resolv.conf        # -> ../run/systemd/resolve/stub-resolv.conf
resolvectl status
```

**10.4**

```bash
sudo ss -tulnp
```

**10.5**

```bash
timedatectl
systemctl status systemd-timesyncd chrony 2>/dev/null | grep -E '●|Active'
```

---

## 🔵 Practical

**10.6**

```bash
sudo ip link set eth2 up
sudo ip addr add 10.10.10.11/24 dev eth2
ip -br addr show eth2
sudo reboot
# reconnect
ip -br addr show eth2         # no address
```

`ip` changes the kernel's live state only. Nothing wrote them to a configuration file, so
nothing recreates them at boot.

**10.7**

```bash
sudo tee /etc/netplan/60-lab.yaml > /dev/null <<'EOF'
network:
  version: 2
  ethernets:
    eth2:
      dhcp4: false
      addresses:
        - 10.99.0.11/24
        - "fd00:99::11/64"
EOF
sudo chmod 600 /etc/netplan/60-lab.yaml
sudo netplan generate
sudo netplan get ethernets.eth2
sudo netplan try              # press Enter to accept
ip -br addr show eth2
```

Quote IPv6 addresses in YAML — the colons can otherwise confuse the parser.

**10.8**

```bash
sudo hostnamectl set-hostname ubuntu1.lab.local
echo '127.0.1.1 ubuntu1.lab.local ubuntu1' | sudo tee -a /etc/hosts
hostnamectl
getent hosts ubuntu1.lab.local
```

Debian-family systems map the host's own name to `127.0.1.1` by convention; mapping it to
the real interface address (`192.168.56.11`) is equally valid and is what you'd do if other
software must see the machine's routable address for its own name.

**10.9**

```bash
echo '192.168.56.12 test.lab.local' | sudo tee -a /etc/hosts
getent hosts test.lab.local     # 192.168.56.12
dig +short test.lab.local       # nothing — dig asks DNS servers directly
```

**10.10**

```bash
sudo tee -a /etc/netplan/60-lab.yaml > /dev/null <<'EOF'
    eth1:
      nameservers:
        addresses: [9.9.9.9]
        search: [lab.local]
EOF
sudo netplan get ethernets.eth1     # vagrant's address must still be there
sudo netplan try
resolvectl status eth1
dig @1.1.1.1 ubuntu.com +short
dig -x 9.9.9.9 +short               # dns9.quad9.net.
```

Appending works here only because `eth1:` lands under the existing `ethernets:` key at the
right indentation — open the file and check rather than trust the heredoc.

**10.11**

Add under `eth1:` in `/etc/netplan/60-lab.yaml`:

```yaml
      routes:
        - to: 10.30.0.0/24
          via: 192.168.56.12
```

```bash
sudo netplan try
ip route get 10.30.0.13         # 10.30.0.13 via 192.168.56.12 dev eth1
```

**10.12**

```bash
ping -c2 10.30.0.13             # fails: ubuntu2 drops packets not addressed to it
# ubuntu2
sudo sysctl -w net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/90-forward.conf
sudo sysctl --system
# ubuntu1
ping -c2 10.30.0.13             # works
```

The route on ubuntu2 was added with `ip`, so it's temporary — fine for the test; a real
router would carry it in Netplan.

You may notice ubuntu2 send ICMP *redirects*: it sees that ubuntu1 and rocky1 share a
network and tells ubuntu1 it could go direct. Routers on shared segments do this; real
networks usually disable it (`net.ipv4.conf.all.send_redirects = 0`).

**10.13**

```bash
ssh ubuntu2 ip -6 addr show eth1 | grep fe80     # note ubuntu2's link-local
ping -6 -c2 fe80::…%eth1
```

Add to the `eth1` section of `60-lab.yaml` on each node (`::11` on ubuntu1, `::12` on
ubuntu2):

```yaml
      addresses:
        - "fd00:56::11/64"
```

```bash
sudo netplan get ethernets.eth1     # confirm 192.168.56.11/24 is still listed
sudo netplan try
ping -6 -c2 fd00:56::12
```

A link-local address exists on every IPv6 interface and is the same prefix on all of them,
so the kernel can't tell which interface you mean — the `%eth1` *zone* says.

**10.14**

Replace `eth2`'s block in `60-lab.yaml`:

```yaml
    eth2: {}
    eth3: {}
  bonds:
    bond0:
      interfaces: [eth2, eth3]
      addresses: [10.99.0.11/24]
      parameters:
        mode: active-backup
        mii-monitor-interval: 100
```

(`bonds:` sits at the same level as `ethernets:`.)

```bash
sudo netplan try
cat /proc/net/bonding/bond0
sudo ip link set eth2 down
grep 'Currently Active Slave' /proc/net/bonding/bond0       # now eth3
sudo ip link set eth2 up
```

**10.15**

Replace the `bonds:` section with:

```yaml
  bridges:
    br0:
      interfaces: [eth2]
      addresses: [10.98.0.11/24]
      parameters:
        stp: false
        forward-delay: 0
```

```bash
sudo netplan try
bridge link
ip -br addr show br0 eth2
```

Netplan doesn't always delete a virtual device it no longer defines; if `bond0` lingers,
`sudo ip link delete bond0`.

**10.16**

```bash
# ubuntu2
sudo apt install -y chrony
echo 'allow 192.168.56.0/24' | sudo tee /etc/chrony/conf.d/lab-server.conf
sudo systemctl restart chrony
# ubuntu1
sudo apt install -y chrony
echo 'server ubuntu2 iburst prefer' | sudo tee /etc/chrony/conf.d/lab-client.conf
sudo systemctl restart chrony
chronyc sources -v
# ubuntu2
sudo chronyc clients
```

Drop-ins in `/etc/chrony/conf.d/` are read by the Ubuntu config's `confdir` line; on RedHat
edit `/etc/chrony.conf`. If ubuntu1 shows `^?` for ubuntu2, the server isn't answering —
check the `allow` line and that UDP 123 isn't filtered.

**10.17**

```bash
sudo timedatectl set-timezone Europe/Bucharest
timedatectl; date
```

**10.18**

```bash
tracepath -n 8.8.8.8
sudo tcpdump -ni any port 53 -c 4 &
dig ubuntu.com +short
```

Through `systemd-resolved`, you'll see two conversations: `dig` → `127.0.0.53`, then the
stub → the upstream server.

**10.19** (rocky1)

```bash
nmcli connection show                 # find the lab connection's name, e.g. "System eth1"
C="System eth1"
sudo nmcli connection modify "$C" +ipv4.addresses 192.168.56.113/24
sudo nmcli connection modify "$C" +ipv4.routes "10.40.0.0/16 192.168.56.12"
sudo nmcli connection modify "$C" ipv4.dns 192.168.56.12
sudo nmcli connection up "$C"
nmcli -f ipv4.addresses,ipv4.routes,ipv4.dns connection show "$C"
```

---

## 🔴 Challenge

**10.20**

```bash
ip -br link                    # 1 link: UP, LOWER_UP — fine
ip -br addr                    # 2 address: present — fine
ping -c1 10.0.2.2              # 3 gateway reachable — fine
ip route get 8.8.8.8           # 4 routing: "Network is unreachable"  ← first failure
sudo netplan apply             # re-applies DHCP config, including the default route
# or: sudo ip route add default via 10.0.2.2 dev eth0
curl -sI https://ubuntu.com | head -1
```

The layer: **routing**. A symptom like "no internet" can live at any layer; checking from
the bottom tells you which in a minute.

**10.21**

```bash
getent hosts ubuntu.com        # 10.255.255.1 — what curl uses
dig +short ubuntu.com          # the real addresses — DNS is fine
grep ubuntu.com /etc/hosts
sudo sed -i '/ubuntu\.com/d' /etc/hosts
```

**10.22**

```bash
# ubuntu2
ss -tlnp | grep 8080           # 127.0.0.1:8080 — loopback only
curl -s localhost:8080 | head -1     # works locally
# ubuntu1
curl ubuntu2:8080              # Connection refused
# ubuntu2: restart it bound to all addresses
kill %1
python3 -m http.server 8080 --bind 0.0.0.0 &
ss -tlnp | grep 8080           # 0.0.0.0:8080
```

From the client the symptom is *refused* — the kernel on ubuntu2 has no listener on
`192.168.56.12:8080` and answers with a reset. No firewall rule would produce that.

**10.23**

```bash
# ubuntu2
nc -lk 9000 &
sudo iptables -I INPUT -p tcp --dport 9001 -j DROP
# ubuntu1
nc -zv -w3 ubuntu2 9000        # succeeded
nc -zv -w3 ubuntu2 9002        # Connection refused  — immediately
nc -zv -w3 ubuntu2 9001        # timed out           — after 3 s
# ubuntu2
sudo iptables -D INPUT -p tcp --dport 9001 -j DROP
```

| Result | Meaning |
|---|---|
| succeeded | something is listening and reachable |
| refused | the host is reachable, nothing listens on that port (it sent RST) |
| timeout | packets are being dropped — a firewall, or the host is down |

This distinction tells you whether to look at the service or the firewall.

**10.24**

```bash
sudo tcpdump -ni eth1 'host 192.168.56.12 and tcp port 22' -c 6 &
ssh ubuntu2 true
```

```text
IP 192.168.56.11.50412 > 192.168.56.12.22: Flags [S],  seq 1234…        SYN
IP 192.168.56.12.22 > 192.168.56.11.50412: Flags [S.], seq 9876…, ack … SYN-ACK
IP 192.168.56.11.50412 > 192.168.56.12.22: Flags [.],  ack …            ACK
```

`.` is ACK; `P` push (data); `F` FIN; `R` reset. A `[S]` with no `[S.]` reply means
filtered; `[S]` answered by `[R.]` means refused — the previous task, seen on the wire.

---

## Quiz answers

| Q | Answer | Why |
|---|---|---|
| 1 | It's gone | `ip` changes live state only |
| 2 | Administratively up, but no link (cable/virtual link down) | |
| 3 | `netplan try` | |
| 4 | YAML forbids tabs for indentation; netplan refuses the file | |
| 5 | `/etc/hosts` (`files` is first); `getent hosts name` | |
| 6 | The name is in `/etc/hosts` (or another NSS source); `dig` only queries DNS | |
| 7 | systemd-resolved's local stub resolver; `resolvectl status` | |
| 8 | `10.20.0.0/16` — the longest prefix | |
| 9 | `ip route get 8.8.8.8` | |
| 10 | `net.ipv4.ip_forward = 1` in a file under `/etc/sysctl.d/`, loaded with `sysctl --system` | |
| 11 | An IPv6 link-local address; every interface has one in `fe80::/64`, so you must name the interface | |
| 12 | Mode 1, `active-backup` | |
| 13 | `/proc/net/bonding/bond0` | |
| 14 | On `br0` | A bridge port carries no IP |
| 15 | Certificates and Kerberos tickets have validity windows; a wrong clock makes valid ones look expired or not yet valid | |
| 16 | `^*` the source currently synced to; `^?` unreachable / not yet usable | |
| 17 | It listens on loopback only; bind to `0.0.0.0` (or the interface address) | |
| 18 | Refused: host reachable, no listener. Timeout: packets dropped (firewall) or host unreachable | |
| 19 | link → address → routing → name resolution → port | |
| 20 | Skips reverse DNS lookups — faster, and doesn't add its own DNS traffic to the capture | |
